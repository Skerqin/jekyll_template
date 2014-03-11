---
layout: post
title: 关于ObReferenceObjectByHandle　
---

貌似，好久没有写日志了，恩，主要是懒得写。

　　　　记录点东西。

　　　　在Hook ZwCreateProcessEx之后，希望能通过ProcessHandle参数来获取创建进程的\_EPROCESS结构，不过很悲剧的是，用PsLookupProcessByProcessId，会获得一个0xC000000D(STATUS_INVALID_PARAMETER)，ProcessHandle和ProcessId明显不是同一个东东嘛，虽然都是HANDLE型变量，那么该怎么做呢？在PspCreateProcess中可以找到答案：



	　　　　PEPROCESS Parent;
	
	　　　　......
	
	　　　　status = ObReferenceObjectByHandle(ParentProcess,
	
	　　　　　　　　　　　　PROCESS_CREATE_PROCESS,
	
	　　　　　　　　　　　　PsProcesType,PreviousMode,
	
	　　　　　　　　　　　　(PVOID*)&Parent,NULL);

　　　　也就是说，可以通过ObReferenceObjectByHandle获取进程的_EPROCESS结构。

　　　　刚刚修改好代码，还没有调试，明天再说，最后附上WDK文档中的说明：



　　　　

###ObReferenceObjectByHandle
The ObReferenceObjectByHandle routine provides access validation on the object handle, and, if access can be granted, returns the corresponding pointer to the object's body.

	NTSTATUS 
	  ObReferenceObjectByHandle(
	    IN HANDLE  Handle,
	    IN ACCESS_MASK  DesiredAccess,
	    IN POBJECT_TYPE  ObjectType  OPTIONAL,
	    IN KPROCESSOR_MODE  AccessMode,
	    OUT PVOID  *Object,
	    OUT POBJECT_HANDLE_INFORMATION  HandleInformation  OPTIONAL
	    );

Parameters
Handle
Specifies an open handle for an object.
DesiredAccess
Specifies the requested types of access to the object. The interpretation of this field is dependent on the object type. Do not use any generic access rights.
ObjectType
Pointer to the object type. ObjectType can be *ExEventObjectType, *ExSemaphoreObjectType, *IoFileObjectType, *PsProcessType, *PsThreadType, *SeTokenObjectType, *TmEnlistmentObjectType, *TmResourceManagerObjectType, *TmTransactionManagerObjectType, or *TmTransactionObjectType.
Note  The SeTokenObjectType object type is supported in Windows XP and later versions of Windows.

If ObjectType is not NULL, the operating system verifies that the supplied object type matches the object type of the object that Handle specifies.
AccessMode
Specifies the access mode to use for the access check. It must be either UserMode or KernelMode. Lower-level drivers should specify KernelMode.
Object
Pointer to a variable that receives a pointer to the object's body. The following table contains the pointer types.

ObjectType parameter	Object pointer type
	\*ExEventObjectType	PKEVENT
	\*ExSemaphoreObjectType	PKSEMAPHORE
	\*IoFileObjectType	PFILE_OBJECT
	\*PsProcessType	PEPROCESS or PKPROCESS
	\*PsThreadType	PETHREAD or PKTHREAD
	\*SeTokenObjectType	PACCESS_TOKEN
	\*TmEnlistmentObjectType	PKENLISTMENT
	\*TmResourceManagerObjectType	PKRESOURCEMANAGER
	\*TmTransactionManagerObjectType	PKTM
	\*TmTransactionObjectType	PKTRANSACTION

The structures that the pointer types reference are opaque, and drivers cannot access the structure members. Because the structures are opaque, PEPROCESS is equivalent to PKPROCESS, and PETHREAD is equivalent to PKTHREAD.

Note  The SeTokenObjectType object type is supported in Windows XP and later versions of Windows.

HandleInformation
Drivers set this to NULL.
Return Value
ObReferenceObjectByHandle returns an NTSTATUS value. The possible return values include:

	STATUS_SUCCESS
	
	STATUS_OBJECT_TYPE_MISMATCH
	
	STATUS_ACCESS_DENIED
	
	STATUS_INVALID_HANDLE