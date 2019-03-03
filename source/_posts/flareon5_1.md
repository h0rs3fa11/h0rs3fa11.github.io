---
title: 2018Flare-On5 Part 1
date: 2018-10-29 23:17:18
tags: [Reverse]
---
开始做的时候比赛已经结束很久了，偶然看到想做做逆向的练习。
### 0x01 MinesweeperChampionshipRegistration
> Welcome to the Fifth Annual Flare-On Challenge! The Minesweeper World Championship is coming soon and we found the registration app. You weren't *officially* invited but if you can figure out what the code is you can probably get in anyway. Good luck!

无非就是找出注册码，第一题是个jar包，使用java反编译工具jd-gui载入后可以直接看到注册码
![](/image/flareon/1_1.PNG)
```
GoldenTicket2018@flare-on.com
```
### 0x02 UltimateMinesweeper
> You hacked your way into the Minesweeper Championship, good job. Now its time to compete. Here is the Ultimate Minesweeper binary. Beat it, win the championship, and we'll move you on to greater challenges.

第二题是一个扫雷游戏，格子30x30，点错一次就弹出 FAIL!
![](/image/flareon/2_1.PNG)
查看用PEID查看导入表，只调用了一个mscoree.dll
![](/image/flareon/2_2.PNG)
判断是.NET程序，用dnSpy反编译。
![](/image/flareon/2_3.PNG)
在左侧的方法列表中可以发现GetKey，可能是关键点。GetKey在SquareRevealedCallback中被调用
```C#
private void SquareRevealedCallback(uint column, uint row)
{
	if (this.MineField.BombRevealed)
	{
		this.stopwatch.Stop();
		Application.DoEvents();
		Thread.Sleep(1000);
		new FailurePopup().ShowDialog();
		Application.Exit();
	}
	this.RevealedCells.Add(row * MainForm.VALLOC_NODE_LIMIT + column);
	if (this.MineField.TotalUnrevealedEmptySquares == 0)
	{
		this.stopwatch.Stop();
		Application.DoEvents();
		Thread.Sleep(1000);
		new SuccessPopup(this.GetKey(this.RevealedCells)).ShowDialog();
		Application.Exit();
	}
}
```
SquareRevealedCallback在方法MainForm中调用，从代码大体看来SquareRevealedCallback就是关键方法
```C#
public MainForm()
{
	this.InitializeComponent();
	this.MineField = new MineField(MainForm.VALLOC_NODE_LIMIT);
	this.AllocateMemory(this.MineField);
	this.mineFieldControl.DataSource = this.MineField;
	this.mineFieldControl.SquareRevealed += this.SquareRevealedCallback;
	this.mineFieldControl.FirstClick += this.FirstClickCallback;
	this.stopwatch = new Stopwatch();
	this.FlagsRemaining = this.MineField.TotalMines;
	this.mineFieldControl.MineFlagged += this.MineFlaggedCallback;
	this.RevealedCells = new List<uint>();
}
```
SquareRevealedCallback的两个参数名为column和row，字面意思理解应该就是当前点击的扫雷界面的位置（列，行）。代码第三行条件判断this.MineField.BombRevealed，BombRevealed根据名字应该与炸弹有关。
```C#
public bool BombRevealed
{
	get
	{
		int num = 0;
		while ((long)num < (long)((ulong)this.Size))
		{
			int num2 = 0;
			while ((long)num2 < (long)((ulong)this.Size))
			{
				if (this.MinesPresent[num2, num] && this.MinesVisible[num2, num])
				{
					return true;
				}
				num2++;
			}
			num++;
		}
		return false;
	}
}
```
上述代码遍历扫雷的格子，如果MinesPresent和MinesVisible的值同时为1，那么this.MineField.BombRevealed值就会是1，再 通过SquareRevealedCallback中，如果第一个条件判断成立，会调用FailurePopup().ShowDialog()，即Fail的窗口，那么this.MineField.BombRevealed的值显而易见的是代表当前点中的格子是不是炸弹，是为1 不是为0(bool)，在满足this.MineField.TotalUnrevealedEmptySquares == 0后，成功的窗口有个参数this.GetKey(this.RevealedCells)，应该就是计算最终Key的。那么可不可以修改程序投机取巧一下呢，试了一下，单纯的改变条件判断流程是不行的，会产生异常
```C#
this.RevealedCells.Add(row * MainForm.VALLOC_NODE_LIMIT + column);
```
RevealCells的数据类型是List<uint>,添加元素row*MainForm.VALLOC_NODE_LIMIT+column
```C#
private static uint VALLOC_NODE_LIMIT = 30u;
```
所以RevealedCells的元素是row*30+column<br>
然后是this.MineField.TotalUnrevealedEmptySquares == 0的判断，TotalUnrevealedEmptySquares字面意思：未显露的空方块总数。
```C#
public int TotalUnrevealedEmptySquares
{
	get
	{
		int num = 0;
		int num2 = 0;
		while ((long)num2 < (long)((ulong)this.Size))
		{
			int num3 = 0;
			while ((long)num3 < (long)((ulong)this.Size))
			{
				if (!this.MinesPresent[num2, num3] && !this.MinesVisible[num2, num3])
				{
					num++;
				}
				num3++;
			}
			num2++;
		}
		return num;
	}
}
```
和上面格子是否代表炸弹用的同样两个值，MinesPresent和MinesPresent，num2,num3定位，多了一个num用来计数。要理解MinesPresent和MinesPresent可能要看看前面初始化时候的代码
```C#
public MineField(uint size)
{
	this.Size = size;
	this.MinesPresent = new bool[(int)size, (int)size];
	this.MinesVisible = new bool[(int)size, (int)size];
	this.MinesFlagged = new bool[(int)size, (int)size];
}
// Token: 0x04000015 RID: 21
private bool[,] minesPresent;
// Token: 0x04000016 RID: 22
private bool[,] minesVisible;
// Token: 0x04000017 RID: 23
private bool[,] minesFlagged;
// Token: 0x04000018 RID: 24
private uint size;
```
三个二维数组，元素由bool组成，
在AllocateMemory方法中有对minesPresent的初始化
![](/image/flareon/2_4.PNG)
其中mf.GarbageCollect[(int)num2, (int)num] = flag即为对minesPresent的赋值
![](/image/flareon/2_5.PNG)
当DeriveVallocType的返回值包含在VALLOC_TYPES时，flag=false，VALLOC_TYPES的值如下
```C#
private static uint VALLOC_TYPE_HEADER_PAGE = 4294966400u;
// Token: 0x04000008 RID: 8
private static uint VALLOC_TYPE_HEADER_POOL = 4294966657u;
// Token: 0x04000009 RID: 9
private static uint VALLOC_TYPE_HEADER_RESERVED = 4294967026u;
```
写了一个脚本来辅助，结果是当r,c为(29,25),(21,8),(8,29)时，minesPresent的值为0，很大可能为0的位置就是没有炸弹的地方了，依次在界面点击
![](/image/flareon/2_6.PNG)
You have won the xxx,and nobody cares
```
Ch3aters_Alw4ys_W1n@flare-on.com
```
前面的逻辑分析有点混乱，因为不知道minesPresent就是正确值，所以直接看AllocateMemory初始化的过程就行了，后面的流程都不用看。
### 0x03 FLEGGO
> When you are finished with your media interviews and talk show appearances after that crushing victory at the Minesweeper Championship, I have another task for you. Nothing too serious, as you'll see, this one is child's play.

![](/image/flareon/3_1.PNG)
child's play？<br>
48个文件，应该都是crakeme，要求输入password<br>
第一个文件1BpnGjHOT7h5vvZsV4vISSb60Xj3pX5G.exe，会有两个比较
```
IronManSucks
ZImIT7DyCMOeF6
```
输入第一个password什么都没有，第二个会在同目录生成一个图片。并且控制台会输出一个字符
![](/image/flareon/3_2.PNG)
48个password全部输入成功，就会有48个字符。并且48张图是按照乐高的顺序排列的，就可以将48个字符排序，直接手动一个一个的分析password再输入得到结果再排序肯定是可以的，但这里是48个文件，如果是480个那得搬到哪天去~，so 接下来就分析一下这些文件的共通性。
![](/image/flareon/compare.PNG)
使用UltraCompare比较两个文件，可以看到前面的完全一致，差异性从偏移0x2ab0开始，另外比较了几个文件，差异都从0x2ab0开始，通过stu_PE查看PE结构可以知道这一段是资源段。
![](/image/flareon/3_3.PNG)
用x32dbg调试，程序通过FindResource获取资源名为"BRICK"的资源句柄。LoadResource得到password，并且最后输出的字符都在特定的偏移位置（图片名和字符通过异或加密），写了一个程序来读取每个程序的图片名和字符，没想到好的方法来读取图片的顺序并自动化，所以最后一步只有通过手动。
首先，读取资源段的C代码如下（选择用C写的原因是因为FindResource直接就可以读了，很方便）
```c
#include<stdio.h>
#include<windows.h>
#include<winuser.h>

HGLOBAL hResLoad;
HMODULE hExe;
HRSRC hRes;
char *p;
char *password, key, *filename;

int main(int argc, char *argv[])
{
    int i, j, k, len_password;
    if(argc == 2)
    {
        hExe = LoadLibrary(TEXT(argv[1]));
    }
    else if(argc == 3)
    {
        hExe = LoadLibrary(TEXT(argv[2]));
    }
    else
    {
        printf("Please input argument.\n");
    }
    if(hExe == NULL)
    {
        printf("can't load exe.\n");
        system("pause");
        return 0;
    }
    hRes = FindResourceW(hExe,101,L"BRICK");
    //printf("%d\n",GetLastError());
    if(hRes == NULL)
    {
        printf("FindResource failed.\n");
        system("pause");
        return 0;
    }
    hResLoad = LoadResource(hExe,hRes);
    if(hResLoad == NULL)
    {
        printf("LoadResource failed\n");
        printf("%d\n",GetLastError());
    }
    p = hResLoad;
    password = (char*)malloc(20);
    filename = (char*)malloc(20);
    for(i = 0, j = 0; *(p + i) != 0; i += 2, j++)
    {
        *( password + j ) = *( p + i );
    }
    len_password = j;
    for(i = 32, j = 0; i < 55; i += 2, j++)
    {
        * ( filename + j) = *( p + i ) ^ 0x85;
    }
    k = j;
    if(argc == 2)
    {
        for(j = 0; j < k; j++)
        {
            printf("%c",*(filename+j));
        }
        key = *( p + 64 ) ^ 0x1A;
        printf(",%c\n",key);
    }
    if(argc == 3)
    {
        for(j = 0; j < len_password; j++)
        {
            printf("%c",*(password + j));
        }
    }
    if (!FreeLibrary(hExe))
    {
        printf("Could not free executable.");
        return 0;
    }
    //system("pause");
    return 0;
}
```
这段代码有几个功能：
1. 读取资源段
2. 按照传入参数的不同选择输出password或者图片名+字符(key)

然后用python来实现遍历文件夹以及执行命令的功能，因为python遍历文件夹比C方便
```python
import os

path = "E:\\work\\Flare-On5\\Flare-On5_Challenges\\FlareOn5_Challenges\\03_FLEGGO"
files = os.listdir(path)
for f in files:
    if os.path.splitext(f)[-1] == '.exe':
        filename = os.path.join(path,f)
        #os.system('test.exe %s'%filename)
        os.system('test.exe -whatever %s | %s'%(filename,filename))
```
执行完毕后图片已经全部生成，并且key已经打印出来，但是要点开48张图片给字符排序也是一种重复劳动，如果要做图像的识别又感觉有点超纲了。。暂时没别的办法我只能手动了
```
mor3_awes0m3_th4n_an_awes0em_p0ssum@flare-on.com
```
弄完之后看了官方给的解答，才发现读取资源段的时候忽略了一个重要的值
![](/image/flareon/3_4.PNG)
这是图片序号为7的程序的内存
![](/image/flareon/3_5.PNG)
这是序号为35(0x23)的图片的位置。既然序号能读出来，那自动化排序也不是难事。
> Part1就这些，后续的慢慢做，毕竟还要上班，还要看书。。