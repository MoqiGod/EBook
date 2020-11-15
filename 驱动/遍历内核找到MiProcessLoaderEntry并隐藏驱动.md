```
/*
win7 32 sp1遍历内核找到 MiProcessLoaderEntry的地址，然后隐藏驱动
*/

#include<ntddk.h>
//隐藏驱动的函数指针
typedef VOID (*MyMiProcessLoaderEntry)(ULONG DataTableEntry,LOGICAL Insert);
CHAR bufcode[] = { 0xb1, 0x1b, 0x88, 0x45, 0x0b, 0x3a, 0xc1 };//第一段特征码  
CHAR bufcode2[] = { 0x33, 0xc0, 0xff, 0x87, 0x60, 0x35, 0x00, 0x00, 0x8b, 0xce, 0xf0, 0x0f, 0xba, 0x29, 0x1f };//+27字节的第二段
VOID MyDriverUnload(PDRIVER_OBJECT pdriver)
{
	KdPrint(("Leave DriverEntry\n"));
}
//这里偷懒了  写死了，应该多定义个模块参数，调用时根据传进来的字符串找模块
NTSTATUS FindNtAddr(PDRIVER_OBJECT pDriverObject,ULONG* addr)
{
	NTSTATUS status = -1;
	PLIST_ENTRY CurList, NextList;
	UNICODE_STRING DllName;
	PUNICODE_STRING tmpDllName;
	RtlInitUnicodeString(&DllName, L"ntoskrnl.exe");
	CurList = (PLIST_ENTRY)(pDriverObject->DriverSection);
	NextList = CurList->Flink;
	KdPrint(("%X", NextList));
	while (CurList != NextList)
	{
		tmpDllName = (PUNICODE_STRING)((ULONG)NextList + 0x2c);
		if (RtlCompareUnicodeString(tmpDllName, &DllName, TRUE) == 0)
		{
			KdPrint(("Find Ok"));
			*addr = (ULONG)NextList;
			status = STATUS_SUCCESS;
			break;
		}
		NextList = NextList->Flink;
	}
	return status;
}
//通过_LDR_DATA_TABLE_ENTRY的首地址+偏移找到内核模块的基址和大小
//参数1，2分别为驱动基址和大小，3是out参数 返回找到的函数地址
NTSTATUS FinMiProcessFun(ULONG Moudlestart, ULONG moudlesize,ULONG* retfun)
{
	NTSTATUS status = -1;
	for (int i = Moudlestart; i < moudlesize; i++)
	{
		if (memcmp(Moudlestart, bufcode, sizeof(bufcode)) == 0)
		{
			if (memcmp(((char*)Moudlestart + 27), bufcode2, sizeof(bufcode2))==0)
			{
				KdPrint(("找到了"));
				__asm int 3;
				*retfun = Moudlestart;
				status = STATUS_SUCCESS;
				return status;
			}
		}
		(char*)Moudlestart++;
	}
	KdPrint(("没找到"));
	return status;
}
NTSTATUS DriverEntry(IN PDRIVER_OBJECT pDriverObject, IN PUNICODE_STRING pRegPath)
{
	NTSTATUS status = STATUS_SUCCESS;
	ULONG addrs;
	ULONG MoudleAddr, MoudleSize;
	ULONG funaddr;
	MyMiProcessLoaderEntry myfun;
	pDriverObject->DriverUnload = MyDriverUnload;
	status = FindNtAddr(pDriverObject, &addrs);
	if (status!=0)
	{
		KdPrint(("模块没找到"));
		return status;
	}
	MoudleAddr = *(ULONG*)(addrs + 0x18);
	MoudleSize = *(ULONG*)(addrs + 0x20);
	MoudleSize += MoudleAddr;
	FinMiProcessFun(MoudleAddr, MoudleSize, &funaddr);
	funaddr = funaddr - 0x1e;
	myfun = funaddr;
	myfun(addrs,FALSE);
	KdPrint(("%X", funaddr));
	KdPrint(("Enter DriverEntry\n"));
	return status;
}
```

