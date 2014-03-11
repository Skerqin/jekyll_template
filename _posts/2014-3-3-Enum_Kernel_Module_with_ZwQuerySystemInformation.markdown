---
layout: post
title: ZwQuerySystemInformation枚举内核模块
---

函数原型:

    :::C
     NTSTAUS ZwQuerySystemInformation(
    
    __inSYSTEM_INFORMATION_CLASS SystemInformationClass,
    
    __inout  PVOID SystemInformation,
    
    __inULONG SystemInformationLength,
    
    __out_optPULONG ReturnLength
    
       );
    

MSDN中说该函数在后续的Windows版本中可能会被改变或取消，建议使用其他替代函数～～额，无视，这里只说该函数的一个用法。

当SystemInformationClass的值为SystemModuleInformation（对应值为0x0b）时，可以用来枚举内核中已加载的模块。

使用时，可以先将SystemInformation设为NULL，来获取信息所需内存的大小:
	
	ZwQuerySystemInformation(SystemModuleInformation,NULL,0,&NeedLen);

然后根据返回NeedLen的大小来申请内存空间，最后在调用ZwQuerySystemInformation获取信息。

返回SYSTEM_MODULE_INFORMATION结构，其中的Module是一个数组，元素个数由Count决定。

这里需要说的是，在360的Hookport.sys文件中采取了另外一种方法。及先申请一定大小的空间，然后调用ZwQuerySystemInformation获取信息，判断返回值，如果是STATUS_INFO_LENGTH_MISMATCH，则释放已经申请的空间，重新申请更大的空间，再次调用ZwQuerySystemInformation，直到返回成功或者其他错误信息。目前并不是很清楚为什么要用这种方法，猜测是为了防止使用ZwQuerySystemInformation获取所需空间之后，系统触发时钟中断，切换到其他进程之后，又加载新的模块进入内核空间，当切换回本进程时，再调用ZwQuerySystemInformation则会失败。

   额～～好吧，MJ0011在debugman上给了答复，在win2000中ZwQuerySystemInformation并不支持返回NeedLen。

然后～～上代码：

	#include <ntifs.h>
	
	 
	
	NTKERNELAPI
	
	NTSTATUS ZwQuerySystemInformation(
	
	        IN  ULONG SystemInformationClass,
	
	        IN  OUT PVOID SystemInformation,
	
	        IN  ULONG SystemInformationLength,
	
	        OUT PULONG ReturnLength OPTIONAL
	
	);
	
	 
	
	typedef struct _SYSTEM_MODULE_INFORMATION_ENTRY {
	
	    HANDLE Section;
	
	    PVOID MappedBase;
	
	    PVOID Base;
	
	    ULONG Size;
	
	    ULONG Flags;
	
	    USHORT LoadOrderIndex;
	
	    USHORT InitOrderIndex;
	
	    USHORT LoadCount;
	
	    USHORT PathLength;
	
	    CHAR ImageName[256];
	
	} SYSTEM_MODULE_INFORMATION_ENTRY, *PSYSTEM_MODULE_INFORMATION_ENTRY;
	
	 
	
	 
	
	typedef struct _SYSTEM_MODULE_INFORMATION {
	
	    ULONG Count;
	
	    SYSTEM_MODULE_INFORMATION_ENTRY Module[1];
	
	} SYSTEM_MODULE_INFORMATION, *PSYSTEM_MODULE_INFORMATION;
	
	 
	
	 
	
	 
	
	VOID MyUnload(PDRIVER_OBJECT DriverObject)
	
	{
	
	}
	
	 
	
	NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
	
	{
	
	    ULONG ReturnLength;
	
	    PSYSTEM_MODULE_INFORMATION pBuf;
	
	 
	
	#if DBG
	
	_asm int 3
	
	#endif
	
	  
	
	    DriverObject->DriverUnload = MyUnload;
	
	   
	
	    ZwQuerySystemInformation(0x0b, NULL, NULL, &ReturnLength);
	
	    pBuf = ExAllocatePoolWithTag(NonPagedPool, ReturnLength, 'Ddk ');
	
	    ZwQuerySystemInformation(0x0b, pBuf, ReturnLength, &ReturnLength);
	
	    ExFreePool(pBuf);
	
	   
	
	    return STATUS_SUCCESS;
	
	}
	
	 

ZwQuerySystemInformation需要声明为导入函数，在ntifs.h中有#define NTKERNELAPI DECLSPEC_IMPORT   

 

另外一种方法就不写了，最后总结一下该功能涉及到的声明：

 
	
	NTKERNELAPI
	
	NTSTATUS ZwQuerySystemInformation(
	
	        IN  ULONG SystemInformationClass,
	
	        IN  OUT PVOID SystemInformation,
	
	        IN  ULONG SystemInformationLength,
	
	        OUT PULONG ReturnLength OPTIONAL
	
	);
	
	 
	
	 
	
	typedef struct _SYSTEM_MODULE_INFORMATION_ENTRY {
	
	    HANDLE Section;
	
	    PVOID MappedBase;
	
	    PVOID Base;
	
	    ULONG Size;
	
	    ULONG Flags;
	
	    USHORT LoadOrderIndex;
	
	    USHORT InitOrderIndex;
	
	    USHORT LoadCount;
	
	    USHORT PathLength;
	
	    CHAR ImageName[256];
	
	} SYSTEM_MODULE_INFORMATION_ENTRY, *PSYSTEM_MODULE_INFORMATION_ENTRY;
	
	 
	
	 
	
	typedef struct _SYSTEM_MODULE_INFORMATION {
	
	    ULONG Count;
	
	    SYSTEM_MODULE_INFORMATION_ENTRY Module[1];
	
	} SYSTEM_MODULE_INFORMATION, *PSYSTEM_MODULE_INFORMATION;
	
	 
	
	typedef enum _SYSTEM_INFORMATION_CLASS { 
	SystemBasicInformation, // 0 Y N 
	SystemProcessorInformation, // 1 Y N 
	SystemPerformanceInformation, // 2 Y N 
	SystemTimeOfDayInformation, // 3 Y N 
	SystemNotImplemented1, // 4 Y N 
	SystemProcessesAndThreadsInformation, // 5 Y N 
	SystemCallCounts, // 6 Y N 
	SystemConfigurationInformation, // 7 Y N 
	SystemProcessorTimes, // 8 Y N 
	SystemGlobalFlag, // 9 Y Y 
	SystemNotImplemented2, // 10 Y N 
	SystemModuleInformation, // 11 Y N 
	SystemLockInformation, // 12 Y N 
	SystemNotImplemented3, // 13 Y N 
	SystemNotImplemented4, // 14 Y N 
	SystemNotImplemented5, // 15 Y N 
	SystemHandleInformation, // 16 Y N 
	SystemObjectInformation, // 17 Y N 
	SystemPagefileInformation, // 18 Y N 
	SystemInstructionEmulationCounts, // 19 Y N 
	SystemInvalidInfoClass1, // 20 
	SystemCacheInformation, // 21 Y Y 
	SystemPoolTagInformation, // 22 Y N 
	SystemProcessorStatistics, // 23 Y N 
	SystemDpcInformation, // 24 Y Y 
	SystemNotImplemented6, // 25 Y N 
	SystemLoadImage, // 26 N Y 
	SystemUnloadImage, // 27 N Y 
	SystemTimeAdjustment, // 28 Y Y 
	SystemNotImplemented7, // 29 Y N 
	SystemNotImplemented8, // 30 Y N 
	SystemNotImplemented9, // 31 Y N 
	SystemCrashDumpInformation, // 32 Y N 
	SystemExceptionInformation, // 33 Y N 
	SystemCrashDumpStateInformation, // 34 Y Y/N 
	SystemKernelDebuggerInformation, // 35 Y N 
	SystemContextSwitchInformation, // 36 Y N 
	SystemRegistryQuotaInformation, // 37 Y Y 
	SystemLoadAndCallImage, // 38 N Y 
	SystemPrioritySeparation, // 39 N Y 
	SystemNotImplemented10, // 40 Y N 
	SystemNotImplemented11, // 41 Y N 
	SystemInvalidInfoClass2, // 42 
	SystemInvalidInfoClass3, // 43 
	SystemTimeZoneInformation, // 44 Y N 
	SystemLookasideInformation, // 45 Y N 
	SystemSetTimeSlipEvent, // 46 N Y 
	SystemCreateSession, // 47 N Y 
	SystemDeleteSession, // 48 N Y 
	SystemInvalidInfoClass4, // 49 
	SystemRangeStartInformation, // 50 Y N 
	SystemVerifierInformation, // 51 Y Y 
	SystemAddVerifier, // 52 N Y 
	SystemSessionProcessesInformation // 53 Y N 
	} SYSTEM_INFORMATION_CLASS; 

　　　　