# hideprocess
#进程隐藏
######使用HOOK API 函数 ZwQuerySystemInformation 实现的隐藏指定进程
###函数介绍
ZwQuerySystemInformation 函数
获取指定的系统信息。

###函数声明

    `NTSTATUS WINAPI ZwQuerySystemInformation( _In_      SYSTEM_INFORMATION_CLASS SystemInformationClass, _Inout_   PVOID                    SystemInformation, _In_      ULONG                    SystemInformationLength,  _Out_opt_ PULONG         ReturnLength);`


###参数

SystemInformationClass [in]
要检索的系统信息的类型。 该参数可以是SYSTEM_INFORMATION_CLASS枚举类型中的以下值之一。

SystemInformation[in，out]
指向缓冲区的指针，用于接收请求的信息。 该信息的大小和结构取决于SystemInformationClass参数的值，如下表所示。

SystemInformationLength [in]
SystemInformation参数指向的缓冲区的大小（以字节为单位）。

ReturnLength [out]

一个可选的指针，指向函数写入所请求信息的实际大小的位置。 如果该大小小于或等于SystemInformationLength参数，则该函数将该信息复制到SystemInformation缓冲区中; 否则返回一个NTSTATUS错误代码，并以ReturnLength返回接收所请求信息所需的缓冲区大小。

返回值

返回NTSTATUS成功或错误代码。
NTSTATUS错误代码的形式和意义在DDK中提供的Ntstatus.h头文件中列出，并在DDK文档中进行了说明。
注意

ZwQuerySystemInformation函数及其返回的结构在操作系统内部，并可能从一个版本的Windows更改为另一个版本。 为了保持应用程序的兼容性，最好使用前面提到的替代功能。
如果您使用ZwQuerySystemInformation，请通过运行时动态链接访问该函数。 如果功能已被更改或从操作系统中删除，这将使您的代码有机会正常响应。 但签名变更可能无法检测。
此功能没有关联的导入库。 您必须使用LoadLibrary和GetProcAddress函数动态链接到Ntdll.dll。
实现原理
首先，先来讲解下为什么 HOOK ZwQuerySystemInformation 函数就可以实现指定进程隐藏。是因为我们遍历进程通常是调用系统 WIN32 API 函数 EnumProcess 、CreateToolhelp32Snapshot 等函数来实现，这些 WIN32 API 它们内部最终是通过调用 ZwQuerySystemInformation 这个函数实现的获取进程列表信息。所以，我们只要 HOOK ZwQuerySystemInformation 函数，对它获取的进程列表信息进行修改，把有我们要隐藏的进程信息从中去掉，那么 ZwQuerySystemInformation 就返回了我们修改后的信息，其它程序获取这个被修的信息后，自然获取不到我们隐藏的进程，这样，指定进程就被隐藏起来了。

其中，我们将HOOK ZwQuerySystemInformation 函数的代码部分写在 DLL 工程中，原因是我们要实现的是隐藏指定进程，而不是单单在自己的进程内隐藏指定进程。写成 DLL 文件，可以方便我们将 DLL 文件注入到其它进程的空间，从而 HOOK 其它进程空间中的 ZwQuerySystemInformation 函数，这样，就实现了在其它进程空间中也看不到指定进程了。

我们选取 DLL 注入的方法是设置全局钩子，这样就可以快速简单地将指定 DLL 注入到所有的进程空间里了。

其中，HOOK API 使用的是自己写的 Inline Hook，即在 32 位程序下修改函数入口前 5 个字节，跳转到我们的新的替代函数；对于 64 位程序，修改函数入口前 12 字节，跳转到我们的新的替代函数。

###编码实现见实验Test



###程序测试

我们运行将要隐藏进程的程序 520.exe，然后打开任务管理器，可以查看到 520.exe 是处于可见状态。接着，以管理员权限运行我们的程序，设置全局消息钩子，将 DLL 注入到所有的进程中，DLL 便在 DllMain 入口点函数处 HOOK ZwQuerySystemInformation 函数，成功隐藏 520.exe 的进程。所以，测试成功。

###总结
要注意 Inline Hook API 的时候，在 32 位系统和 64 位系统下的差别。在 32 位使用 jmp _NewAddress 跳转语句，机器码是 5 字节，而且要注意理解它的跳转偏移的计算方式：
跳转偏移 = 跳转地址 - 下一跳指令的地址
在 64 位使用的是的汇编指令是：
mov rax, _NewAddress
jmp rax
机器码是 12 字节。

