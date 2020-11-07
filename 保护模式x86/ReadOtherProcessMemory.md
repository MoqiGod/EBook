### 切换CR3读取另一个进程的内存

```


// ReadOtherProcessMemory.cpp :切换cr3读取另一个进程的数据实验
#include "stdafx.h"
#include <Windows.h>
#define  PDE29912(x)(((x>>0x12)&0x3ff8)-0x3fa00000)	//29912
#define  PTE29912(x)(((x>>0x9)&0x7FFFF8)-0x40000000)//29912

int oldcr3 = 0;
int newcr3 = 0;
int tagvaule = 0;
int tmppde,tmppte;
int lineaddr = 0x80b95100;
int funaddr = 0;
//shellcode代码
__declspec(naked) void ReadMem()
{
	__asm
	{
	//切换新的CR3，并读取405000处的值，这个地址是我在另一个进程里打印出来的 肯定有物理页
		mov eax,newcr3
		mov cr3,eax
		mov ecx,0x00405000
		mov ecx,dword ptr ds:[ecx]
		mov dword ptr ds:[0x80b95140],ecx
		//从公共区域恢复保存的本进程的CR3
		mov eax,dword ptr [0x80b95080]
		mov eax,[eax]
		mov cr3,eax
		ret	
	}
}
__declspec(naked) void test()
{
	__asm
	{
		pushad
		pushfd
		push 0x30
		pop fs		
	}
	funaddr = (int)ReadMem;
	//复制shellcode到指定的位置，我这里是0x80b95100
	for (int i = 0;i<200;i++)
	{
		*(char*)lineaddr = *(char*)funaddr;
		(char*)lineaddr++;
		(char*)funaddr++;
	}
	__asm{	
    //保存本进程的cr3到公共区域，下面切换cr3以后放在变量里会找不到，因为用的是目标进程的cr3
		mov eax,cr3
		mov dword ptr ds:[0x80b95080],eax	
		//调用shellcode
		mov eax,0x80b95100
		call eax
        //从目标进程读到的数据 放到变量里
		mov eax,dword ptr ds:[0x80b95140]
		mov tagvaule,eax
		popfd
		popad
		retf
	}
}
char buf[6] = {0,0,0,0,0x48,0};
int _tmain(int argc, _TCHAR* argv[])
{
	printf("%x\n",test);
	scanf("%x",&newcr3);
	//先打印下本进程的405000
	printf("%x\n",*(int*)0x405000);
	system("pause");
	__asm 
	{	push fs
		call fword ptr buf
		pop fs
	}
	printf("%x\n",tagvaule);
	system("pause");
	return 0;

}
```

#### 封装了一下，最垃圾的那种

```
// ReadOtherProcessMemory.cpp :切换cr3读取另一个进程的数据实验
#include "stdafx.h"
#include <Windows.h>
#define  PDE29912(x)(((x>>0x12)&0x3ff8)-0x3fa00000)        //29912
#define  PTE29912(x)(((x>>0x9)&0x7FFFF8)-0x40000000)//29912

int parvaule = 0;
int newcr3 = 0;
int tagvaule = 0;
int tmppde,tmppte;
int lineaddr = 0x80b95100;
int funaddr = 0;

//shellcode代码
__declspec(naked) void ReadMem()
{
	__asm
	{
		
		//切换新的CR3，并读取405000处的值，这个地址是我在另一个进程里打印出来的 肯定有物理页
		mov eax,newcr3
		mov cr3,eax
		mov ebx,dword ptr ds:[0x80b95088]	
		mov ecx,dword ptr ds:[ebx]
		mov dword ptr ds:[0x80b95140],ecx
		//从公共区域恢复保存的本进程的CR3
		mov eax,dword ptr [0x80b95080]
		mov eax,[eax]
		mov cr3,eax
		ret       
	}
}
__declspec(naked) void test()
{
	__asm
	{
		pushad
		pushfd
		push 0x30
		pop fs               
	}
	funaddr = (int)ReadMem;
	//复制shellcode到指定的位置，我这里是0x80b95100
	for (int i = 0;i<200;i++)
	{
		*(char*)lineaddr = *(char*)funaddr;
		(char*)lineaddr++;
		(char*)funaddr++;
	}
	__asm{       
		//保存参数为全局变量,这里可以使用有参的调用门
		mov eax,parvaule
		mov dword ptr ds:[0x80b95088],eax
		//保存本进程的cr3到公共区域，下面切换cr3以后放在变量里会找不到，因为用的是目标进程的cr3
		mov eax,cr3
		mov dword ptr ds:[0x80b95080],eax       
		//调用shellcode
		mov eax,0x80b95100
		call eax
		//从目标进程读到的数据 放到变量里
		mov eax,dword ptr ds:[0x80b95140]
		mov tagvaule,eax
		popfd
		popad
		retf
	}
}
char buf[6] = {0,0,0,0,0x48,0};
//参数为想读取的线性地址
int testfun()
{
	//保存参数到全局变量，这里可以使用有参的调用门

	printf("%x\n",test);
	printf("请挂上调用门");
	system("pause");
	__asm
	{   push fs
		call fword ptr buf
		pop fs
	}

	return tagvaule;
}

int _tmain(int argc, _TCHAR* argv[])
{
	printf("请输入cr3:");
	scanf("%x",&newcr3);
	printf("请输入线形地址:");
	scanf("%x",&parvaule);
	int y = testfun();
	printf("%x\n",y);
	system("pause");
	return 0;

}
```

