#### win7 32位下修改高两G的物理地址PDE和PTE的U/S位属性

```

#include "stdafx.h"
#include <Windows.h>
//逆向MmIsAddressValid得到的公式
#define  PDE29912(x)(((x>>0x12)&0x3ff8)-0x3fa00000)	//29912
#define  PTE29912(x)(((x>>0x9)&0x7FFFF8)-0x40000000)//29912
#define  PDE(i)(((i>>0x14)&0xffc)-0x3fd00000)	//101012分页		
#define  PTE(i)(((i>>0xa)&0x3ffffc)-0x40000000) //101012分页  //只需要替换宏就可以了
#define  Ring0Begin 0x80000000
#define  Ring0End 0xffe00000
int tmppde,tmppte;
int tmpsize=0;
__declspec(naked) void test()
{
	__asm{
		pushad
		pushfd
		push 0x30
		pop fs
	}
	for (int i = Ring0Begin;i<Ring0End;i+=0x400000)//0x400000是一个PDE的大小
	{
		tmppde = PDE(i);
		tmpsize = i;
		//首先判断PDE的P位是否有效
		if (((*(int*)tmppde)&0x1) !=1 )
		{
			continue;
		}
		//PS 1 or 0  判断是否为大页
		if (((*(int*)tmppde)&0x80)==0x80)
		{
			//us 1 or 0  如果为大页，US位是否为1
			if (((*(int*)tmppde)&0x4)==4)
			{
				continue;
			}
			//如果不是1，置成1
			*(int*)tmppde = *(int*)tmppde | 0x4;
			continue;
		}
		for(int j = tmpsize;j<(tmpsize+0x400000);j+=0x1000)//这里同样套路判断修改PTE
		{
			//P
			tmppte = PTE(j);
			if (((*(int*)tmppte)&0x1) !=1 )
			{
				continue;
			}
			if (((*(int*)tmppte)&0x4)==4)
			{
				continue;
			}
			*(int*)tmppte = *(int*)tmppte | 0x4;	
		}		
	}
	__asm
	{
		popfd
		popad
		retf
	}

}
char buf[6] = {0,0,0,0,0x4b,0};
int _tmain(int argc, _TCHAR* argv[])
{

	printf("%x\n",test);
	__asm{
		push fs
		call fword ptr [buf]
		pop fs
	}
	
	system("pause");
	return 0;

}
```

