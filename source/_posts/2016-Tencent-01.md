---
title: 2016腾讯游戏安全第一题
date: 2017-05-10 21:39:54
tags: [reverse,crackme]
---
## 0x01 前言

这个题去年腾讯游戏安全挑战赛的时候做过，没有做粗来，一直留在我的电脑里，前几天逛看雪的时候看到一个分析贴，才想着把它做了。
<!--more-->
## 0x02 用户名第一次计算

这个程序未加壳。用OD载入，Ctrl+N在GetWindowTextA下断点，未断下
GetDlgItem断下，单步走发现SendMessage
```asm
mov     eax, dword ptr [eax+20]          ; |
mov     edi, dword ptr [<&USER32.SendMes>; |USER32.SendMessageA
push    eax                              ; |hWnd
call    edi                              ; \SendMessageA
```
执行到SendMessage后，看栈里面
```asm
0012F5B4 000201B6 |hWnd = 201B6
0012F5B8 0000000D |Message = WM_GETTEXT
0012F5BC 00000040 |Count = 40 (64.)
0012F5C0 0012F660 \Buffer = 0012F660
```
消息方式WM_GETTEXT
MSDN: Copies the text that corresponds to a window into a buffer provided by the caller.
把窗口对应的文本复制到调用者提供的缓冲区中
这里提供的缓冲区是0x0012F660，执行SendMessage后，0x0012F660处的内容由0变为了我输入的用户名
在0012FF60设置硬件访问断点，然后运行，断在下面
```asm
mov     cl, byte ptr [eax]
inc     eax
cmp     cl, bl
jnz     short 00401F85                            ;  用户名长度
```
接下来把用户名长度-6后与0x0E比较，如果大于0x0E就直接注册失败，说明长度不能大于20或者小于6
```asm
sub     eax, edx
mov     edi, eax
lea     eax, dword ptr [edi-6]
ja      004021E7
```
下面是对用户名username的计算
(0x1339E7E+i) x username[j] x strlen(username) 每计算一次加原先的结果
```asm
mov     eax, ecx
cdq
idiv    edi
lea     esi, dword ptr [esp+ecx+88]
inc     ecx
movsx   eax, byte ptr [esp+edx+9C]
lea     edx, dword ptr [esi+ebp]
imul    eax, edx
imul    eax, edi
add     dword ptr [esi], eax
cmp     ecx, 10                                   ;  循环16次
jl      short 00401FE0
```
## 0x03 序列号计算

序列号长度
```asm
mov     cl, byte ptr [eax]
inc     eax
cmp     cl, bl
jnz     short 00402053
```
call 401960是序列号的计算
```asm
xor     edi, edi
mov     al, byte ptr [esp+edi+18]
lea     ebx, dword ptr [esp+10]
mov     byte ptr [esp+10], al
call    00402420                       ;  索引
mov     byte ptr [esp+edi+18], al
inc     edi
cmp     edi, 4
jl      short 00401A72
```
call 00402420是对输入的序列号进行了一个在”ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789@%”中获取索引的操作(在这之前也判断了序列号是否是这个字符串当中的字符)例如字符’y’，处于上面那个字符串的第50位，返回的索引值就是0x32
然后是一个计算
![](https://attach.52pojie.cn/forum/201705/09/165655cfl8sazll9nxkmly.png)
PS:上图的按位与符号写错了，应该是&这里就是让序列号每四个字节为一组，形如[a,b,c,d],[e,f,g,h]，通过上面的运算，每四个字节会得到三个字节结果
key[0]=key[2]<<6+key[3]
key[1]=key[1]<<4^(key[2]>>2&F)
key_[2]=4key[0]+key[1]>>4&3
到下面有一个计算的值与0x14比较，这里比较的内容就是刚才计算的key值长度，这里说明计算后的key应该有20字节，说明计算前的key应该有24+3=27字节
```asm
cmp     edx, 14
je      short 004020D8
```
## 0x04 用户名第二次计算
```asm
mov     ecx, dword ptr [esp+esi+88]
mov     eax, 66666667
imul    ecx
sar     edx, 2
mov     ecx, edx
shr     ecx, 1F
add     ecx, edx        ;result_name/10
mov     edx, dword ptr [esp+24]
sub     edx, edi
mov     dword ptr [esp+esi+2C], ecx  
cmp     esi, edx
jb      short 00402131
call    0041105A
mov     edi, dword ptr [esp+20]
mov     eax, dword ptr [esi+edi]
mov     dword ptr [esp+esi+40], eax
add     esi, 4
cmp     esi, 14
jl      short 00402102
```
再往下，就到了一个比较的地方
```asm
mov     ecx, dword ptr [esp+2C]          ;  name_0
mov     eax, dword ptr [esp+50]          ;  key_4
lea     edx, dword ptr [eax+ecx]         ;  name_0+key_4
mov     ecx, dword ptr [esp+48]          ;  key_2
cmp     edx, ecx                         ;  key_2==key_4+name_2
jnz     short 004021AB
mov     edx, dword ptr [esp+30]          ;  name_1
add     edx, ecx                         ;  name_1+key_2
add     eax, eax                         ;  key4+key4
cmp     edx, eax                         ;  name_1+key_2==key_4*2
jnz     short 004021AB
mov     ecx, dword ptr [esp+4C]          ;  key_3
mov     eax, dword ptr [esp+34]          ;  name_2
mov     edx, dword ptr [esp+40]          ;  key_0
lea     esi, dword ptr [ecx+eax]         ;  key_3+name_2
cmp     esi, edx                         ;  key_0==key_3+name_2
jnz     short 004021AB
mov     esi, dword ptr [esp+38]          ;  name_3
add     esi, edx                         ;  key_0+name_3
add     ecx, ecx                         ;  key_3+key_3
cmp     esi, ecx                         ;  key_0+name_3==key_3+key_3
jnz     short 004021AB
mov     edx, dword ptr [esp+3C]          ;  name_4
mov     ecx, dword ptr [esp+44]          ;  key_1
add     ecx, edx                         ;  key_1+name_4
lea     edx, dword ptr [eax+eax*2]       ;  name_2*3
cmp     ecx, edx                         ;  name_2*3==key_1+name_4
jnz     short 004021AB
```
0x05 注册机&&CM下载
[下载链接](https://github.com/h0rs3fa11/2016Tencent-1.git)
