第一种办法：构造一个带参数的调用门

 

```
eq 80b95048 0040ec01`000814f0

declspec(naked)void test()
{
	__asm {		
		int 3
		iretd	//这里会把那个参数当做eflags当做参数弹出去
	}
}

char buf[6] = { 0,0,0,0,0x4b,0 };

__asm
 	{
		pushfd					//这里因为要用到iretd返回所以故意压一个参数进去，可以是随意值
		push fs 				//为了程序正常执行，先保存fs，我这里环境不这样做会报错
		call fword ptr [buf];
		pop fs
		add esp,4				//恢复上面pushfd造成的堆栈不平衡
	｝
```

第二种方法：

```
#include "stdafx.h"
#include<windows.h>

__declspec(naked)void test()
{
	__asm {
		int 3
		sub esp,4
		mov eax,[esp+0x4]
		mov [esp],eax
		mov eax,[esp+0x8]
		mov [esp+0x4],eax
		iretd
}

}
char buf[6] = { 0,0,0,0,0x4b,0 };
int main()
{	
	printf("%x\n", test);
	system("pause");
__asm
	{
		push fs
		call fword ptr [buf];
		pop fs	
	}
	return 0;

}


```

//提权调用DbgPrint

```
#include "stdafx.h"
#include<windows.h>
//83e6141f  //DbgPrint地址
typedef int(_cdecl *DbgPrintfProc)(char const* const _Format, ...);
DbgPrintfProc DbgPrint = (DbgPrintfProc)0x83e6141f;
char* str1 = { "1111111111111111111" };
__declspec(naked)void test()
{
	__asm {
		push 0x30
		pop fs
		mov eax,[str1]
		push eax
		call DbgPrint
		add esp,4
		retf
	}
}
char buf[6] = { 0,0,0,0,0x4b,0 };
int main()
{
	printf("%x\n", test);
	system("pause");
	__asm
		{
			push fs
			call fword ptr [buf];
			pop fs
	
	}
	return 0;
}
```



3.从TSS来，白皮书说的



