中断门使用retf返回

个人理解：

retf返回的时候，弹栈顺序应该是返回地址-->cs-->esp-->ss，如果retf后有值，便会认为有参数，然后根据值去弹出相应的参数，retf会把 int指令压入的elfags当做参数弹出（因为调用门的时候它就是这么做的），因为int不能带参数没有显式的push xxx，但是弹出的时候 是retf 0x4，所以为导致栈不平衡，需要在下面显式的平衡下堆栈

```
#include "stdafx.h"
#include<windows.h>

__declspec(naked)void test()
{
__asm {	
		push 0x30
		pop fs
		int 3
		retf 0x4	//这里多return了一个值，所以会导致堆栈不平衡
	   }
}

int main()
{	
	printf("%x\n", test);
	system("pause");
	__asm
	{		
		int 0x20
		sub esp,4			//这里平衡上面的堆栈
		push 0x3b
		pop fs	
	}

return 0;

}
```

