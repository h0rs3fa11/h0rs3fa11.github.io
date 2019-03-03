---
title: API HOOK(1)
date: 2017-05-15 21:53:24
tags: [hook api,笔记,dll注入]
---
## 前言

还是windows核心编程的期末答辩，上次那个键盘记录器只完成了一部分，这次要实现的是采用HOOK API技术，
拦截记事本的保存操作，<!--more-->调用MessageBox函数输出”记事本被监控”。HOOK API是对API进行拦截，而全局钩子是对消息进行拦截。
在写程序之前先想一下具体要做些什么，还是要写一个dll文件完成HOOK API的具体操作。然后这个dll还需要被注入到目标进程，相当于还需要一把枪，来发射这个dll，所以还要写一把”枪”。首先来看dll注入的具体实现。

## dll注入

主要有三种方法

使用注册表
消息钩取(SetWindowsHookEx())
创建远程线程(CreateRemoteThread())
我使用CreateRemote()，就当顺便学习dll注入的姿势了。
这里先解决dll注入，所以dll的代码随便写一点
```c
//APIHook.cpp
#include<Windows.h>
HMODULE g_hMod = NULL;
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
	HANDLE hThread = NULL;
	g_hMod = (HMODULE)hinstDLL;
	switch (fdwReason)
	{
	case DLL_PROCESS_ATTACH:
		OutputDebugString("myhack.dll Injection!!!");
		break;
	}
	return TRUE;
}
```
关于CreateRemoteThread注入的代码网上有很多，就不贴上来了。记录一下主要思路和我程序出的错。

>OpenProcess()获得目标进程的句柄，第一个参数要设置为PROCESS_ALL_ACCESS，最高权限以便后面的申请内存空间和写内存。
VitualAllocEx()申请内存空间。CreateRemoteThread创建新线程的时候传递的函数参数必须在目标进程空间内，这里的参数就是dll的路径。
WriteProcessMemory() 写入内存
GetModuleHandle(“kernel32.dll”)，为了获得LoadLibrary。
GetProcAddress(“hModule,”LoadLibraryA”)，获得目标进程LoadLibrary的地址
CreateRemoteThread(),这个函数的第四个参数是创建的新线程要执行的函数，相当于CreateThread的ThreadProc()。这里心机的把它设为LoadLibrary的地址，好让新线程来执行LoadLibrary，参数自然就是dll的路径。

我的一些错误

>关于64位系统运行的notepad.exe的问题。我的主程序是32位的，dll也是，然后机器是64位的notepad，我也没多想，直接Win+R运行notepad.exe，然而CreateRemoteThread执行都出错。错误码是5，突然发现GetLastError挺好用的。上网搜了一下，Win+R运行的是C:\Windows\System32下的notepad.exe，而C:\Windows\System32保存的是64位的文件。这里运行C:\Windows\SysWOW64下的notepad.exe就行了，真的坑。。

经过上一个问题，CreateRemoteThread()执行成功了以后，可是什么都没有。。Process explorer来看notepad加载的dll也没有，这就很奇怪了，我又搞了很久，后面想到了OD，附加到notepad.exe上，调试选项-事件勾选上中断于新线程。然后notepad断下了，说明CreateRemoteThread()确实已经成功创建了线程，然后我发现是LoadLibrary的调用出了错，原来如此–
在后面添加代码来输出dll是否成功加载
```c++
::WaitForSingleObject(hRemoteThread, INFINITE);
	DWORD exitCode = 0;
	GetExitCodeThread(hRemoteThread, &exitCode);
	if (exitCode != 0)
		std::cout << "DLL loaded successfully." << std::endl;
	else
		std::cout << "DLL load failed." << std::endl;
```
LoadLibrary失败有很多原因，我这里是目录没对，以及调用LoadLibraryW，我在OD里手动把参数改成了Unicode，就成了。然后我改成调用LoadLibraryA。mdzz
细节和玄学，折腾了这么久
![](\image\dll注入.PNG)