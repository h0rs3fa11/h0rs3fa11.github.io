---
title: API HOOK(2)
date: 2017-05-15 22:03:45
tags: [hook api,笔记]
---
## 0x01 前言

接着上一篇的dll注入写，现在是来写API HOOK的主要代码。
<!--more-->
## 0x02 API HOOK 介绍

## 0x03 导入表

## 0x04 定位导入表

PEID来看看notepad.exe调用了哪些API，然后找到了WriteFile，应该会在保存文件的时候调用到。所以我就用OD载入，在WriteFile上下个断点，运行之后保存的时候就断在了WriteFile，所以现在目的是Hook WriteFile。
定位导入表就先来分析PE结构，首先是DOS头（也可以使用一些PE View来查看，不过我想再熟悉一下PE结构）
```c
typedef struct _IMAGE_DOS_HEADER{
    ...//略
    LONG e_lfanew;
}IMAGE_DOS_HEADER,*PIMAGE_DOS_HEADER
```
DOS头的最后一个成员e_lfanew（偏移0x3F）指向NT头，如图
![](\image\NTh.png)
接下来的结构是DOS存根，暂时不用管。根据NT头的偏移找到NT头。

NT头
```c
typedef struct _IMAGE_NT_HEADERS{
    DWORD Signature;
    IMAGE_FILE_HEADER FileHeader;
    IMAGE_OPTIONAL_HEADER32 OptionalHeader;
}IMAGE_NT_HEADERS32,*PIMAGE_NT_HEADERS32;
```
第一个成员是签名，第二、三个成员是文件头和可选头。导入表也是在可选头中。可选头在NT头中的偏移为0x18。
```c
typedef struct _IMAGE_OPTIONAL_HEADER{
    ...//略
    IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];
}IMAGE_OPTIONAL_HEADER32,*PIMAGE_OPTIONAL_HEADER32;
```
DataDirectory的类型IMAGE_DATA_DIRECTORY定义如下：
```c
typedef struct _IMAGE_DATA_DIRECTORY{
    DWORD VitualAddress;
    DWORD Size;
}IMAGE_DATA_DIRECTORY,*PIMAGE_DATA_DIRECTORY;
```
最后一个成员DataDirectory（偏移0x60）数组的DataDirectory[1].VirtualAddress保存了指向导入表IMAGE_IMPORT_DESCRIPTOR的值。
![](\image\IAT.png)
其中204A4是RVA地址，转换成RAW
RAW=RVA-VirtualAddress+PointerToRawData
OD中看一下，RVA=204A4位于第三个节区，节区起始20000
用Winhex来观察一下节区头的结构
```c
#define IMAGE_SIZEOF_SHORT_NAME 8
typedef struct _IMAGE_SECtION_HEADER{
    BYTE Name[IMAGE_SIZEOF_SHORT_NAME]; 
    union{
        DWORD PhysicalAddress;
        DWORD VirtualSize;
    }Misc;
    DWORD VirtualAddress;
    DWORD SizeOfRawData;
    DWORD PointerToRawData;
    ...//略
}
```
![](\image\IAT1.png)
从图中可以看出来，VirtualAddress是20000，PointerToRawData是1BE00
RAW=204A4-20000+1BE00=1C2A4
![](\image\import_.png)
图中红框部分就是全部的IMAGE_IMPORT_DESCRIPTOR结构体数组
```c
typedef struct _IMAGE_IMPORT_DESCRIPTOR{
    union{
        DWORD Characteristics;
        DWORD OriginalFirstThunk;  //Import name table(INT) address(RVA)
    };
    DWORD TimeDateStamp;
    DWORD ForwarderChain;
    DWORD Name;      //library name string address(RVA)
    DWORD FirstThunk;   //IAT address(RVA)
}IMAGE_IMPORT_DESCRIPTOR;
```
从图中可以知道对应的一些值分别为
OriginalFirstThunk=20698
TimeDateStamp=00000000
ForwarderChain=00000000
Name=20C32
FirstThunk=20000
然后把OriginalFirstThunk、Name、FirstThunk分别转化成RAW
通过上面的计算
OriginalFirstThunk(RAW)=20698-20000+1BE00=1C498
Name(RAW)=20c32-20000+1BE00=1CA32
FirstThunk(RAW)=1BE00
![](\image\dllname.png)
可以看到已经定位到了库名称Name
然后是OriginalFirstThunk(INT)，包含导入函数信息的结构体数组。
![](\image\INT.png)
跟随第一个成员20B38->1C938
![](\image\intname.png)
然后是FristThunk，就是IAT
![](\image\IAT(1).png)
在OD中跟随一下上面的地址
![](\image\iat_.png)

## 0x05 HOOK API的实现

先构造一个自定义的WriteFile来替换本来的WriteFile(结构要和WriteFile一样)，下面是WriteFile的MSDN定义
```c
BOOL WINAPI WriteFile(
  _In_        HANDLE       hFile,
  _In_        LPCVOID      lpBuffer,
  _In_        DWORD        nNumberOfBytesToWrite,
  _Out_opt_   LPDWORD      lpNumberOfBytesWritten,
  _Inout_opt_ LPOVERLAPPED lpOverlapped
);
```
所以，就定义一个MyWriteFile函数如下
```
BOOL WINAPI MyWriteFile(HANDLE hFile,LPCVOID lpBuffer,DWORD nNumberOfBytesToWrite,LPDWORD lpNumberOfBytesWritten,LPOVERLAPPED lpOverlapped)
{
    MessageBox(0, "你的记事本正在被监控", "Hacked", MB_OK);
	return TRUE;
}
```
dll中(在这个dll已经被注入目标进程的前提下)的DllMain代码如下
```c
HANDLE hThread = NULL;
	g_hMod = (HMODULE)hinstDLL;
	BOOL bOk = TRUE;
	switch (fdwReason)
	{
	case DLL_PROCESS_ATTACH:
		OutputDebugString("myhack.dll Injection!!! IAT");
		bOk = SetHook(::GetModuleHandle(NULL));
		if (bOk)
		{
			OutputDebugString("IAT success");
		}
		else
		{
			OutputDebugString("IAT failed");
		}
		break;
	}
	return TRUE;
```
这些代码会让dll被加载时执行SetHook()函数，SetHook()里就是定位导入表+修改导入表，通过上一节PE结构的分析，可以得出定位导入表的代码。然后就是将我们新构造的函数地址覆盖原有地址，这里涉及到内存权限的问题，我本来以为自身的进程都可以读写。我们教材上的代码也是直接就调用WriteProcessMemory()了，直接调用的话GetLastError()会得到错误代码998。
这里我们先用VirtualQuery()查询一下你要写入的内存的Protect属性，我查询后得到的值是0x02，也就是PAGE_READONLY，怪不得不能写入。
接下来就是用VirtualProtect()修改Protect属性，修改为PAGE_READWRITE。然后再调用WriteProcessMemory()就可以了。
SetHook()完整代码如下：
```c
BOOL SetHook(HMODULE hMod)
{
	MEMORY_BASIC_INFORMATION infor;
	PDWORD oldProtect;
	IMAGE_DOS_HEADER* pDosHeader = (IMAGE_DOS_HEADER*)hMod;
	IMAGE_OPTIONAL_HEADER* pOptHeader = 
		(IMAGE_OPTIONAL_HEADER*)((BYTE*)hMod + pDosHeader->e_lfanew + 24);
	IMAGE_IMPORT_DESCRIPTOR* pImportDesc = (IMAGE_IMPORT_DESCRIPTOR*)
		((BYTE*)hMod + pOptHeader->DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress);
	//查找kernel32.dll
	while (pImportDesc->FirstThunk)
	{
		char* pszDllName = (char*)((BYTE*)hMod + pImportDesc->Name);
		if (lstrcmpA(pszDllName, "KERNEL32.dll") == 0)
		{
			break;
		}
		pImportDesc++;
	}
	if (pImportDesc->FirstThunk)
	{
		IMAGE_THUNK_DATA* pThunk = (IMAGE_THUNK_DATA*)((BYTE*)hMod + pImportDesc->FirstThunk);
		while (pThunk->u1.Function)
		{
			DWORD* lpAddr = (DWORD*)&(pThunk->u1.Function);
			if (*lpAddr == (DWORD)g_orgProc)
			{
				//修改IAT
				DWORD *lpNewProc = (DWORD*)MyWriteFile;
				::VirtualQuery(lpAddr, &infor, sizeof(infor));
				oldProtect = (PDWORD)malloc(sizeof(DWORD));
				::VirtualProtect(lpAddr, sizeof(DWORD), PAGE_READWRITE, oldProtect);
				if(!WriteProcessMemory(GetCurrentProcess(), lpAddr, &lpNewProc, sizeof(DWORD),NULL))
				{
					return FALSE;
				}
				return TRUE;
			}
			pThunk++;
		}
	}
	
	return FALSE;
}
```