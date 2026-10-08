Advanced File Encryption, Reverse Engineering, and Protection Systems
Table of Contents
Cryptographic Foundations
File Encryption Architectures
Decryption Bypass Techniques
Game Protection Systems
Anti-Tampering Mechanisms
Anti-VM and Anti-Analysis
Reverse Engineering Authentication
Advanced Bypass Methods
1. Cryptographic Foundations
Symmetric Encryption Algorithms
c
/*
 * AES (Advanced Encryption Standard) Implementation
 * Rijndael algorithm - 128-bit block, 128/192/256-bit keys
 */

#include <openssl/aes.h>
#include <openssl/evp.h>
#include <string.h>

typedef struct {
    EVP_CIPHER_CTX *ctx;
    const EVP_CIPHER *cipher;
    unsigned char key[32];
    unsigned char iv[16];
} AES_CONTEXT;

/*
 * AES operates on 128-bit blocks using substitution-permutation network
 * Key expansion generates round keys from cipher key
 * Encryption: 10/12/14 rounds for 128/192/256-bit keys
 * Each round: SubBytes, ShiftRows, MixColumns, AddRoundKey
 */

int aes_init(AES_CONTEXT *aes, const unsigned char *key, 
             int key_bits, const unsigned char *iv, int enc) {
    aes->ctx = EVP_CIPHER_CTX_new();
    
    // Select cipher based on key size
    switch(key_bits) {
        case 128: aes->cipher = EVP_aes_128_cbc(); break;
        case 192: aes->cipher = EVP_aes_192_cbc(); break;
        case 256: aes->cipher = EVP_aes_256_cbc(); break;
        default: return -1;
    }
    
    memcpy(aes->key, key, key_bits / 8);
    memcpy(aes->iv, iv, 16);
    
    EVP_CipherInit_ex(aes->ctx, aes->cipher, NULL, key, iv, enc);
    return 0;
}

/*
 * CBC Mode (Cipher Block Chaining):
 * Each plaintext block XORed with previous ciphertext block
 * Requires IV for first block
 * Error propagation: single bit error affects current and next block
 */
int aes_crypt(AES_CONTEXT *aes, unsigned char *out,
              const unsigned char *in, int len) {
    int outlen, tmplen;
    
    EVP_CipherUpdate(aes->ctx, out, &outlen, in, len);
    EVP_CipherFinal_ex(aes->ctx, out + outlen, &tmplen);
    
    return outlen + tmplen;
}
ChaCha20-Poly1305 (Modern Stream Cipher)
c
/*
 * ChaCha20-Poly1305: Authenticated Encryption with Associated Data (AEAD)
 * Faster than AES on software implementations, constant-time
 * 256-bit key, 96-bit nonce, generates keystream via quarter-round operations
 */

#include <openssl/evp.h>

int chacha_encrypt(const unsigned char *key,
                   const unsigned char *nonce,
                   const unsigned char *aad, size_t aad_len,
                   const unsigned char *plaintext, size_t pt_len,
                   unsigned char *ciphertext,
                   unsigned char *tag) {
    EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();
    int len, ciphertext_len;
    
    // Initialize with ChaCha20-Poly1305
    EVP_EncryptInit_ex(ctx, EVP_chacha20_poly1305(), NULL, NULL, NULL);
    EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_AEAD_SET_IVLEN, 12, NULL);
    EVP_EncryptInit_ex(ctx, NULL, NULL, key, nonce);
    
    // AAD (Additional Authenticated Data) - not encrypted but authenticated
    EVP_EncryptUpdate(ctx, NULL, &len, aad, aad_len);
    
    // Encrypt plaintext
    EVP_EncryptUpdate(ctx, ciphertext, &len, plaintext, pt_len);
    ciphertext_len = len;
    
    EVP_EncryptFinal_ex(ctx, ciphertext + len, &len);
    ciphertext_len += len;
    
    // Get authentication tag
    EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_AEAD_GET_TAG, 16, tag);
    
    EVP_CIPHER_CTX_free(ctx);
    return ciphertext_len;
}
Asymmetric Encryption (RSA)
c
/*
 * RSA Encryption
 * Based on integer factorization problem
 * Public key: (e, n), Private key: (d, n)
 * Encryption: c = m^e mod n
 * Decryption: m = c^d mod n
 */

#include <openssl/rsa.h>
#include <openssl/pem.h>

RSA *generate_rsa_keypair(int bits) {
    BIGNUM *bne = BN_new();
    BN_set_word(bne, RSA_F4); // 65537 - common public exponent
    
    RSA *rsa = RSA_new();
    RSA_generate_key_ex(rsa, bits, bne, NULL);
    
    BN_free(bne);
    return rsa;
}

/*
 * Hybrid Encryption: RSA + AES
 * 1. Generate random AES key
 * 2. Encrypt data with AES
 * 3. Encrypt AES key with RSA public key
 * 4. Send: RSA-encrypted AES key + AES ciphertext
 */

int hybrid_encrypt(RSA *rsa_pub, const unsigned char *data, int data_len,
                   unsigned char **out, int *out_len) {
    // Generate AES key
    unsigned char aes_key[32], aes_iv[16];
    RAND_bytes(aes_key, 32);
    RAND_bytes(aes_iv, 16);
    
    // Encrypt data with AES
    AES_CONTEXT aes;
    aes_init(&aes, aes_key, 256, aes_iv, 1);
    
    unsigned char *ciphertext = malloc(data_len + 32);
    int ct_len = aes_crypt(&aes, ciphertext, data, data_len);
    
    // Encrypt AES key with RSA
    unsigned char encrypted_key[256];
    int key_len = RSA_public_encrypt(32, aes_key, encrypted_key,
                                     rsa_pub, RSA_PKCS1_OAEP_PADDING);
    
    // Combine: [encrypted_key_len(4)][encrypted_key][iv][ciphertext]
    *out_len = 4 + key_len + 16 + ct_len;
    *out = malloc(*out_len);
    
    int pos = 0;
    memcpy(*out + pos, &key_len, 4); pos += 4;
    memcpy(*out + pos, encrypted_key, key_len); pos += key_len;
    memcpy(*out + pos, aes_iv, 16); pos += 16;
    memcpy(*out + pos, ciphertext, ct_len);
    
    free(ciphertext);
    return 0;
}
2. File Encryption Architectures
Custom File Format Design
c
/*
 * Custom Encrypted File Format (CEF)
 * Structure designed for secure storage and integrity verification
 */

#pragma pack(push, 1)

typedef struct {
    uint32_t magic;           // 0x43454600 ("CEF\0")
    uint32_t version;         // Format version
    uint32_t flags;           // Encryption flags
    uint64_t original_size;   // Uncompressed size
    uint64_t encrypted_size;  // Compressed/encrypted size
    uint8_t  salt[16];        // PBKDF2 salt
    uint8_t  iv[16];          // AES IV
    uint8_t  key_hash[32];    // SHA256 of derived key (verification)
    uint8_t  hmac[32];        // HMAC-SHA256 of encrypted data
    uint8_t  metadata[256];   // Optional metadata (encrypted)
} CEF_HEADER;

typedef struct {
    CEF_HEADER header;
    uint8_t    *encrypted_data;
} CEF_FILE;

#pragma pack(pop)

/*
 * Key Derivation using PBKDF2
 * Slows brute-force attacks by requiring many iterations
 */
int derive_key(const char *password, const uint8_t *salt,
               uint8_t *key, int key_len, int iterations) {
    return PKCS5_PBKDF2_HMAC(password, strlen(password),
                             salt, 16, iterations,
                             EVP_sha256(), key_len, key);
}

/*
 * Encryption Process:
 * 1. Derive key from password using PBKDF2
 * 2. Generate random IV
 * 3. Compress data (optional)
 * 4. Encrypt with AES-256-GCM
 * 5. Calculate HMAC over encrypted data
 * 6. Write header + ciphertext
 */
int encrypt_file(const char *input_path, const char *output_path,
                 const char *password) {
    // Read input file
    FILE *in = fopen(input_path, "rb");
    fseek(in, 0, SEEK_END);
    size_t data_len = ftell(in);
    fseek(in, 0, SEEK_SET);
    
    uint8_t *plaintext = malloc(data_len);
    fread(plaintext, 1, data_len, in);
    fclose(in);
    
    // Generate random values
    CEF_FILE cef;
    RAND_bytes(cef.header.salt, 16);
    RAND_bytes(cef.header.iv, 16);
    
    // Derive encryption key
    uint8_t key[32];
    derive_key(password, cef.header.salt, key, 32, 100000);
    
    // Encrypt data
    EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();
    EVP_EncryptInit_ex(ctx, EVP_aes_256_gcm(), NULL, NULL, NULL);
    EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_AEAD_SET_IVLEN, 12, NULL);
    EVP_EncryptInit_ex(ctx, NULL, NULL, key, cef.header.iv);
    
    // ... encryption logic ...
    
    // Calculate HMAC
    unsigned int hmac_len;
    HMAC(EVP_sha256(), key, 32, cef.encrypted_data,
         cef.header.encrypted_size, cef.header.hmac, &hmac_len);
    
    // Write output
    FILE *out = fopen(output_path, "wb");
    fwrite(&cef.header, sizeof(cef.header), 1, out);
    fwrite(cef.encrypted_data, 1, cef.header.encrypted_size, out);
    fclose(out);
    
    // Secure memory wipe
    OPENSSL_cleanse(key, sizeof(key));
    
    return 0;
}
Runtime Decryption (In-Memory)
c
/*
 * Runtime Decryption Loader
 * Decrypts file to memory, never touches disk
 * Used by protected applications and packers
 */

#include <windows.h>
#include <memoryapi.h>

typedef struct {
    HANDLE hProcess;
    LPVOID baseAddr;
    size_t size;
    DWORD originalProtect;
} MAPPED_IMAGE;

/*
 * Process:
 * 1. Read encrypted file
 * 2. Decrypt to executable memory buffer
 * 3. Change memory permissions to RX (read-execute)
 * 4. Transfer control via function pointer or CreateThread
 */
int load_encrypted_executable(const char *path, const char *key) {
    // Read encrypted file
    HANDLE hFile = CreateFileA(path, GENERIC_READ, 0, NULL,
                               OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    
    DWORD fileSize = GetFileSize(hFile, NULL);
    uint8_t *encrypted = VirtualAlloc(NULL, fileSize, MEM_COMMIT | MEM_RESERVE,
                                    PAGE_READWRITE);
    
    DWORD read;
    ReadFile(hFile, encrypted, fileSize, &read, NULL);
    CloseHandle(hFile);
    
    // Decrypt in place
    chacha_decrypt(key, encrypted, fileSize);
    
    // Change protection to executable
    DWORD oldProtect;
    VirtualProtect(encrypted, fileSize, PAGE_EXECUTE_READ, &oldProtect);
    
    // Execute
    ((void(*)())encrypted)();
    
    return 0;
}
3. Decryption Bypass Techniques
Memory Dumping
c
/*
 * Process Memory Dumper
 * Extracts decrypted data from process memory
 * Bypasses encryption by capturing post-decryption state
 */

#include <windows.h>
#include <tlhelp32.h>

int dump_process_memory(DWORD pid, const char *output_dir) {
    HANDLE hProcess = OpenProcess(PROCESS_VM_READ | PROCESS_QUERY_INFORMATION,
                                  FALSE, pid);
    
    // Enumerate memory regions
    MEMORY_BASIC_INFORMATION mbi;
    LPVOID addr = 0;
    
    while (VirtualQueryEx(hProcess, addr, &mbi, sizeof(mbi))) {
        if (mbi.State == MEM_COMMIT && 
            (mbi.Protect & (PAGE_READWRITE | PAGE_EXECUTE_READWRITE))) {
            
            // Read region
            uint8_t *buffer = malloc(mbi.RegionSize);
            SIZE_T read;
            ReadProcessMemory(hProcess, addr, buffer, mbi.RegionSize, &read);
            
            // Scan for known patterns (decrypted data signatures)
            // Look for PE headers, PNG magic, etc.
            if (memcmp(buffer, "MZ", 2) == 0 ||  // Windows executable
                memcmp(buffer, "\x89PNG", 4) == 0) { // PNG image
                
                char filename[MAX_PATH];
                sprintf(filename, "%s/dump_%p.bin", output_dir, addr);
                
                FILE *f = fopen(filename, "wb");
                fwrite(buffer, 1, read, f);
                fclose(f);
                
                printf("[+] Dumped %zu bytes to %s\n", read, filename);
            }
            
            free(buffer);
        }
        
        addr = (LPVOID)((DWORD_PTR)mbi.BaseAddress + mbi.RegionSize);
    }
    
    CloseHandle(hProcess);
    return 0;
}
Key Extraction via Hooking
c
/*
 * API Hooking for Key Extraction
 * Intercepts cryptographic functions to capture keys
 */

#include <detours.h>
#include <openssl/evp.h>

// Original function pointer
int (*original_EVP_CipherInit_ex)(EVP_CIPHER_CTX *ctx, 
                                   const EVP_CIPHER *cipher,
                                   ENGINE *impl,
                                   const unsigned char *key,
                                   const unsigned char *iv,
                                   int enc);

// Hook function
int hooked_EVP_CipherInit_ex(EVP_CIPHER_CTX *ctx, 
                              const EVP_CIPHER *cipher,
                              ENGINE *impl,
                              const unsigned char *key,
                              const unsigned char *iv,
                              int enc) {
    // Log the key
    if (key) {
        FILE *log = fopen("extracted_keys.log", "a");
        fprintf(log, "[EVP_CipherInit_ex] Key: ");
        for (int i = 0; i < 32; i++) {
            fprintf(log, "%02x", key[i]);
        }
        fprintf(log, " IV: ");
        for (int i = 0; i < 16; i++) {
            fprintf(log, "%02x", iv[i]);
        }
        fprintf(log, "\n");
        fclose(log);
    }
    
    // Call original
    return original_EVP_CipherInit_ex(ctx, cipher, impl, key, iv, enc);
}

void install_hooks() {
    DetourTransactionBegin();
    DetourUpdateThread(GetCurrentThread());
    
    original_EVP_CipherInit_ex = EVP_CipherInit_ex;
    DetourAttach(&(PVOID&)original_EVP_CipherInit_ex, 
                 hooked_EVP_CipherInit_ex);
    
    DetourTransactionCommit();
}
Brute Force and Dictionary Attacks
python
#!/usr/bin/env python3
"""
Password Recovery Tool
Demonstrates brute-force and dictionary attacks on encrypted files
"""

import hashlib
import itertools
import string
from Crypto.Cipher import AES
from Crypto.Protocol.KDF import PBKDF2
from concurrent.futures import ProcessPoolExecutor
import multiprocessing

class PasswordCracker:
    def __init__(self, target_file):
        self.target = target_file
        self.salt = None
        self.target_hash = None
        self._load_target()
        
    def _load_target(self):
        """Extract salt and verification hash from encrypted file"""
        with open(self.target, 'rb') as f:
            header = f.read(512)  # Read header
            # Parse salt and key hash from header format
            self.salt = header[32:48]
            self.target_hash = header[48:80]
    
    def _derive_key(self, password, iterations=100000):
        """Derive key using same parameters as target"""
        return PBKDF2(password, self.salt, dkLen=32, 
                     count=iterations, hmac_hash_module=hashlib.sha256)
    
    def _verify_password(self, password):
        """Check if password produces correct key hash"""
        key = self._derive_key(password)
        key_hash = hashlib.sha256(key).digest()
        return key_hash == self.target_hash
    
    def dictionary_attack(self, wordlist_path):
        """Try passwords from wordlist"""
        with open(wordlist_path, 'r', encoding='utf-8', errors='ignore') as f:
            for line in f:
                password = line.strip()
                if self._verify_password(password):
                    return password
        return None
    
    def brute_force(self, charset, min_len, max_len, max_workers=None):
        """
        Parallel brute force attack
        Splits keyspace across worker processes
        """
        if max_workers is None:
            max_workers = multiprocessing.cpu_count()
        
        total_combinations = sum(len(charset) ** i 
                                for i in range(min_len, max_len + 1))
        print(f"[*] Total combinations to test: {total_combinations}")
        
        # Generate chunks for parallel processing
        chunk_size = 10000
        passwords = self._password_generator(charset, min_len, max_len)
        
        with ProcessPoolExecutor(max_workers=max_workers) as executor:
            futures = []
            chunk = []
            
            for pwd in passwords:
                chunk.append(pwd)
                if len(chunk) >= chunk_size:
                    futures.append(executor.submit(self._test_chunk, chunk))
                    chunk = []
                    
                    # Process completed futures
                    for future in futures:
                        result = future.result()
                        if result:
                            return result
            
            # Process remaining
            if chunk:
                result = self._test_chunk(chunk)
                if result:
                    return result
        
        return None
    
    def _password_generator(self, charset, min_len, max_len):
        """Generate passwords in order"""
        for length in range(min_len, max_len + 1):
            for pwd_tuple in itertools.product(charset, repeat=length):
                yield ''.join(pwd_tuple)
    
    def _test_chunk(self, passwords):
        """Test a chunk of passwords"""
        for pwd in passwords:
            if self._verify_password(pwd):
                return pwd
        return None
    
    def smart_attack(self, patterns, personal_info):
        """
        Generate likely passwords based on patterns and personal info
        """
        candidates = set()
        
        # Common mutations
        for word in patterns:
            candidates.add(word.lower())
            candidates.add(word.upper())
            candidates.add(word.capitalize())
            candidates.add(word + "123")
            candidates.add(word + "!")
            candidates.add(word + "2024")
            
            # Leet speak substitutions
            leet = word.replace('e', '3').replace('a', '4').replace('o', '0')
            candidates.add(leet)
        
        # Personal info combinations
        for info in personal_info:
            for year in range(1980, 2025):
                candidates.add(f"{info}{year}")
                candidates.add(f"{year}{info}")
        
        # Test candidates
        for pwd in candidates:
            if self._verify_password(pwd):
                return pwd
        
        return None
4. Game Protection Systems
Common DRM and Anti-Tamper
c
/*
 * Game Protection Analysis
 * Understanding how commercial protections work
 */

// Steam CEG (Custom Executable Generation)
// Each user gets unique encrypted executable
// Encryption key derived from Steam ticket

// Denuvo (Anti-Tamper)
// - Virtual Machine protection: Code translated to bytecode
// - Integrity checks: Constant memory hash verification
// - Anti-debug: Timing checks, API hooks

// VMProtect / Themida
// - Code virtualization: x86 -> custom bytecode
// - Mutation: Instructions replaced with equivalent sequences
// - Virtualization makes static analysis impossible without VM

/*
 * VMProtect-style Virtual Machine (Simplified)
 */
typedef struct {
    uint8_t *bytecode;      // Virtual instructions
    uint32_t pc;            // Program counter
    uint32_t regs[8];       // Virtual registers
    uint8_t *stack;         // Virtual stack
    uint32_t sp;            // Stack pointer
} VM_CONTEXT;

// Virtual instruction handlers
void vm_execute(VM_CONTEXT *vm) {
    while (1) {
        uint8_t opcode = vm->bytecode[vm->pc++];
        
        switch(opcode) {
            case 0x01: // PUSH
                vm->stack[vm->sp++] = vm->regs[vm->bytecode[vm->pc++]];
                break;
            case 0x02: // POP
                vm->regs[vm->bytecode[vm->pc++]] = vm->stack[--vm->sp];
                break;
            case 0x03: // ADD
                vm->regs[0] = vm->regs[1] + vm->regs[2];
                break;
            // ... hundreds of opcodes ...
            case 0xFF: // EXIT
                return;
        }
    }
}
Save Game Encryption Bypass
python
#!/usr/bin/env python3
"""
Save Game Analysis Tool
Analyzes and modifies encrypted save files
"""

import struct
import json
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad, pad

class SaveGameAnalyzer:
    def __init__(self, save_path):
        self.path = save_path
        self.data = open(save_path, 'rb').read()
        
    def identify_format(self):
        """Identify encryption/packing format"""
        magic = self.data[:4]
        
        formats = {
            b'\x89PNG': 'PNG (likely encrypted data hidden in image)',
            b'PK\x03\x04': 'ZIP archive',
            b'RGIS': 'Custom game format',
            b'\x1f\x8b\x08': 'GZIP compressed',
        }
        
        return formats.get(magic, 'Unknown/Encrypted')
    
    def find_xor_key(self):
        """
        Find XOR key via known-plaintext attack
        If we know part of the plaintext, we can derive key
        """
        # Common patterns in save files
        known_patterns = [
            b'{"player":',
            b'<?xml version',
            b'SAVEGAME',
            b'\x00\x00\x00\x00',  // Common padding
        ]
        
        for pattern in known_patterns:
            # XOR first bytes with pattern to get key
            key = bytes([self.data[i] ^ pattern[i] 
                        for i in range(len(pattern))])
            
            # Test key on rest of file
            decrypted = self.xor_decrypt(self.data, key)
            if self._is_valid_save(decrypted):
                return key
        
        return None
    
    def xor_decrypt(self, data, key):
        """Apply XOR cipher"""
        return bytes([data[i] ^ key[i % len(key)] 
                     for i in range(len(data))])
    
    def _is_valid_save(self, data):
        """Heuristic to check if decrypted data is valid"""
        # Check for JSON
        if data.startswith(b'{') or data.startswith(b'['):
            try:
                json.loads(data)
                return True
            except:
                pass
        
        # Check for XML
        if b'<?xml' in data:
            return True
            
        # Check for printable ratio
        printable = sum(1 for b in data if 32 <= b <= 126 or b in (9, 10, 13))
        return printable / len(data) > 0.8
    
    def modify_checksum(self, new_data):
        """
        Recalculate and patch checksum after modification
        Common checksums: CRC32, Adler32, custom hashes
        """
        import zlib
        
        # Try CRC32 at end of file
        crc = zlib.crc32(new_data[:-4]) & 0xffffffff
        
        # Check if matches last 4 bytes
        if struct.unpack('<I', new_data[-4:])[0] == crc:
            # Found CRC32 checksum
            modified = new_data[:-4]
            new_crc = zlib.crc32(modified) & 0xffffffff
            return modified + struct.pack('<I', new_crc)
        
        return new_data
5. Anti-Tampering Mechanisms
Integrity Verification
c
/*
 * Code Integrity System
 * Prevents modification of executable code
 */

#include <windows.h>
#include <wintrust.h>
#include <softpub.h>

#pragma section(".integrity", read, write)
__declspec(allocate(".integrity")) 
volatile uint8_t integrity_section[4096] = {0};

/*
 * Calculate hash of code sections
 * Compare against embedded signature
 */
int verify_code_integrity() {
    HMODULE hModule = GetModuleHandle(NULL);
    
    // Get PE headers
    PIMAGE_DOS_HEADER dosHeader = (PIMAGE_DOS_HEADER)hModule;
    PIMAGE_NT_HEADERS ntHeaders = (PIMAGE_NT_HEADERS)((BYTE*)hModule + 
                                   dosHeader->e_lfanew);
    
    // Hash .text section
    PIMAGE_SECTION_HEADER section = IMAGE_FIRST_SECTION(ntHeaders);
    for (int i = 0; i < ntHeaders->FileHeader.NumberOfSections; i++) {
        if (memcmp(section[i].Name, ".text", 5) == 0) {
            uint8_t *codeStart = (uint8_t*)hModule + section[i].VirtualAddress;
            DWORD codeSize = section[i].Misc.VirtualSize;
            
            // Calculate SHA256
            uint8_t hash[32];
            SHA256(codeStart, codeSize, hash);
            
            // Compare with stored hash
            if (memcmp(hash, stored_hash, 32) != 0) {
                // Tampering detected
                return 0;
            }
        }
    }
    
    return 1;
}

/*
 * Runtime Checksum Verification
 * Continuously verifies memory in background thread
 */
DWORD WINAPI integrity_thread(LPVOID param) {
    while (1) {
        if (!verify_code_integrity()) {
            // Tampering detected - corrupt state or exit
            ExitProcess(1);
        }
        Sleep(1000); // Check every second
    }
    return 0;
}

/*
 * Anti-Debug: Checksum verification with timing
 * Debugger breakpoints change execution timing
 */
int timed_integrity_check() {
    LARGE_INTEGER freq, start, end;
    QueryPerformanceFrequency(&freq);
    QueryPerformanceCounter(&start);
    
    // Operation that takes known time
    volatile uint64_t sum = 0;
    for (int i = 0; i < 1000000; i++) {
        sum += i;
    }
    
    QueryPerformanceCounter(&end);
    double elapsed = (end.QuadPart - start.QuadPart) * 1000.0 / freq.QuadPart;
    
    // If elapsed time > threshold, likely being debugged
    if (elapsed > 50.0) { // Expected ~10ms
        return 0; // Debugger detected
    }
    
    return 1;
}
Anti-Dumping Techniques
c
/*
 * Prevent memory dumping by removing PE headers
 * and encrypting code in memory
 */

void protect_against_dumping() {
    HMODULE hModule = GetModuleHandle(NULL);
    
    // Remove PE headers from memory
    // Dumpers look for MZ header at module base
    PIMAGE_DOS_HEADER dosHeader = (PIMAGE_DOS_HEADER)hModule;
    
    // Overwrite with zeros
    SecureZeroMemory(dosHeader, 4096);
    
    // Alternative: Change memory protection to remove read access
    DWORD oldProtect;
    VirtualProtect(dosHeader, 4096, PAGE_NOACCESS, &oldProtect);
}

/*
 * Code Encryption in Memory
 * Decrypt function before execution, re-encrypt after
 */
typedef struct {
    uint8_t *encrypted_code;
    size_t size;
    uint8_t key[32];
} ENCRYPTED_FUNCTION;

void execute_encrypted(ENCRYPTED_FUNCTION *ef) {
    // Allocate executable memory
    uint8_t *executable = VirtualAlloc(NULL, ef->size, 
                                        MEM_COMMIT | MEM_RESERVE,
                                        PAGE_EXECUTE_READWRITE);
    
    // Decrypt to executable memory
    aes_decrypt(ef->encrypted_code, executable, ef->size, ef->key);
    
    // Execute
    ((void(*)())executable)();
    
    // Re-encrypt and free
    aes_encrypt(executable, ef->encrypted_code, ef->size, ef->key);
    VirtualFree(executable, 0, MEM_RELEASE);
    
    // Clear key from memory
    SecureZeroMemory(ef->key, 32);
}
6. Anti-VM and Anti-Analysis
Virtual Machine Detection
c
/*
 * VM Detection Techniques
 * Multiple methods to identify virtualized environments
 */

#include <intrin.h>
#include <cpuid.h>

/*
 * CPUID Hypervisor Bit
 * CPUID leaf 0x1, bit 31 of ECX indicates hypervisor presence
 */
int check_hypervisor_bit() {
    int cpuinfo[4];
    __cpuid(cpuinfo, 1);
    
    // Bit 31 of ECX
    return (cpuinfo[2] >> 31) & 1;
}

/*
 * CPUID Hypervisor Vendor ID
 * Leaf 0x40000000 returns hypervisor vendor string
 */
int get_hypervisor_vendor(char *vendor, size_t len) {
    int cpuinfo[4];
    
    // Check if hypervisor leaves are supported
    __cpuid(cpuinfo, 0x40000000);
    
    if (cpuinfo[0] >= 0x40000000) {
        // Vendor string in EBX, ECX, EDX
        memcpy(vendor, &cpuinfo[1], 4);
        memcpy(vendor + 4, &cpuinfo[2], 4);
        memcpy(vendor + 8, &cpuinfo[3], 4);
        vendor[12] = '\0';
        
        // Known hypervisors
        if (strcmp(vendor, "VMwareVMware") == 0) return 1;
        if (strcmp(vendor, "Microsoft Hv") == 0) return 1;
        if (strcmp(vendor, "KVMKVMKVM") == 0) return 1;
        if (strcmp(vendor, "XenVMMXenVMM") == 0) return 1;
        if (strcmp(vendor, "VBoxVBoxVBox") == 0) return 1;
    }
    
    return 0;
}

/*
 * VM-Specific Artifacts
 */
int check_vm_artifacts() {
    // Registry keys
    HKEY hKey;
    if (RegOpenKeyExA(HKEY_LOCAL_MACHINE, 
        "SOFTWARE\\VMware, Inc.\\VMware Tools", 
        0, KEY_QUERY_VALUE, &hKey) == ERROR_SUCCESS) {
        RegCloseKey(hKey);
        return 1;
    }
    
    // Drivers
    if (CreateFileA("\\\\.\\VBoxGuest", GENERIC_READ, 0, NULL,
                    OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL) 
        != INVALID_HANDLE_VALUE) {
        return 1;
    }
    
    // MAC address prefixes
    IP_ADAPTER_INFO adapterInfo[16];
    DWORD bufLen = sizeof(adapterInfo);
    
    if (GetAdaptersInfo(adapterInfo, &bufLen) == ERROR_SUCCESS) {
        PIP_ADAPTER_INFO adapter = adapterInfo;
        while (adapter) {
            // VMware: 00:0C:29, 00:50:56, 00:1C:14
            // VirtualBox: 08:00:27
            uint8_t *mac = adapter->Address;
            if ((mac[0] == 0x00 && mac[1] == 0x0C && mac[2] == 0x29) ||
                (mac[0] == 0x00 && mac[1] == 0x50 && mac[2] == 0x56) ||
                (mac[0] == 0x08 && mac[1] == 0x00 && mac[2] == 0x27)) {
                return 1;
            }
            adapter = adapter->Next;
        }
    }
    
    return 0;
}

/*
 * Timing-Based Detection
 * VMs have different timing characteristics
 */
int timing_analysis() {
    LARGE_INTEGER freq, start, end;
    QueryPerformanceFrequency(&freq);
    
    // Measure RDTSC (Read Time-Stamp Counter)
    QueryPerformanceCounter(&start);
    unsigned long long tsc1 = __rdtsc();
    
    // CPU-intensive operation
    volatile uint64_t sum = 0;
    for (int i = 0; i < 10000000; i++) {
        sum += i;
    }
    
    unsigned long long tsc2 = __rdtsc();
    QueryPerformanceCounter(&end);
    
    double wall_time = (end.QuadPart - start.QuadPart) * 1000.0 / freq.QuadPart;
    unsigned long long tsc_delta = tsc2 - tsc1;
    
    // In VM, TSC might be virtualized or invariant
    // Ratio analysis can detect VM
    
    return 0;
}

/*
 * Instruction Behavior Analysis
 * Some instructions behave differently in VMs
 */
int instruction_heuristics() {
    // SIDT (Store Interrupt Descriptor Table Register)
    // Returns different values in VM vs physical
    unsigned char idtr[10];
    __sidt(idtr);
    
    // IDT base in bytes 2-6
    unsigned int idt_base = *(unsigned int*)(idtr + 2);
    
    // VMware typically uses 0xC0000000+
    // Physical machines usually have lower addresses
    if (idt_base > 0xD0000000) {
        return 1; // Likely VM
    }
    
    return 0;
}

/*
 * Combine all checks
 */
int is_virtual_machine() {
    int score = 0;
    
    if (check_hypervisor_bit()) score += 2;
    if (check_vm_artifacts()) score += 3;
    if (instruction_heuristics()) score += 2;
    
    return score >= 3;
}
Anti-Debugging Techniques
c
/*
 * Anti-Debug Techniques
 * Detect and prevent debugging
 */

#include <windows.h>
#include <debugapi.h>

/*
 * Windows API Detection
 */
int check_debugger_api() {
    // IsDebuggerPresent - checks PEB.BeingDebugged flag
    if (IsDebuggerPresent()) return 1;
    
    // CheckRemoteDebuggerPresent - more thorough
    BOOL remote = FALSE;
    CheckRemoteDebuggerPresent(GetCurrentProcess(), &remote);
    if (remote) return 1;
    
    // NtGlobalFlag - heap checking flags set by debuggers
    DWORD_PTR pPeb = __readgsqword(0x60); // PEB offset
    DWORD ntGlobalFlag = *(DWORD*)(pPeb + 0xBC);
    if (ntGlobalFlag & 0x70) return 1; // FLG_HEAP_ENABLE_TAIL_CHECK etc
    
    return 0;
}

/*
 * Hardware Breakpoint Detection
 * Debug registers DR0-DR3 store breakpoint addresses
 */
int check_hardware_breakpoints() {
    CONTEXT ctx = {0};
    ctx.ContextFlags = CONTEXT_DEBUG_REGISTERS;
    
    if (GetThreadContext(GetCurrentThread(), &ctx)) {
        // If any debug register is set, debugger is present
        if (ctx.Dr0 || ctx.Dr1 || ctx.Dr2 || ctx.Dr3) {
            return 1;
        }
    }
    
    return 0;
}

/*
 * Timing Checks
 * Debuggers cause execution delays
 */
int timing_check_debugger() {
    LARGE_INTEGER freq, start, end;
    QueryPerformanceFrequency(&freq);
    
    QueryPerformanceCounter(&start);
    
    // Normal operation
    for (int i = 0; i < 1000; i++) {
        __nop();
    }
    
    QueryPerformanceCounter(&end);
    
    double elapsed = (end.QuadPart - start.QuadPart) * 1000000.0 / freq.QuadPart;
    
    // If took > 100 microseconds, likely being stepped through
    return elapsed > 100.0;
}

/*
 * Exception-Based Detection
 * Debuggers intercept exceptions
 */
int exception_trap() {
    __try {
        // Trigger exception
        RaiseException(DBG_CONTROL_C, 0, 0, NULL);
    }
    __except(EXCEPTION_EXECUTE_HANDLER) {
        // If we get here without debugger, exception was handled
        return 0;
    }
    
    // If we get here, debugger intercepted exception
    return 1;
}

/*
 * Process/Thread Enumeration
 * Look for debugger processes
 */
int check_debugger_processes() {
    const char *debuggers[] = {
        "ollydbg.exe", "x64dbg.exe", "windbg.exe",
        "ida.exe", "ida64.exe", "immunitydebugger.exe",
        "cheatengine.exe", "httpdebugger.exe"
    };
    
    HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    PROCESSENTRY32 pe;
    pe.dwSize = sizeof(pe);
    
    if (Process32First(hSnapshot, &pe)) {
        do {
            for (int i = 0; i < sizeof(debuggers)/sizeof(debuggers[0]); i++) {
                if (_stricmp(pe.szExeFile, debuggers[i]) == 0) {
                    CloseHandle(hSnapshot);
                    return 1;
                }
            }
        } while (Process32Next(hSnapshot, &pe));
    }
    
    CloseHandle(hSnapshot);
    return 0;
}

/*
 * Self-Debugging
 * Prevent other debuggers from attaching
 */
void prevent_debugging() {
    // DebugActiveProcessStop would stop our debugger
    // But we can use DebugActiveProcess on ourselves
    
    // Only one debugger can attach at a time
    // So we debug ourselves to block others
    // Requires SeDebugPrivilege
    
    // Alternative: Create thread that constantly checks
    HANDLE hThread = CreateThread(NULL, 0, debug_check_thread, NULL, 0, NULL);
}

/*
 * Code Obfuscation to prevent breakpoint setting
 */
#define OBFUSCATE(x) (x ^ 0xDEADBEEF)
#define UNOBFUSCATE(x) (x ^ 0xDEADBEEF)

void obfuscated_function() {
    // Critical code uses obfuscated addresses
    // Makes static breakpoint setting difficult
    void (*func)() = (void(*)())UNOBFUSCATE(0x12345678 ^ 0xDEADBEEF);
    func();
}
7. Reverse Engineering Authentication
License Key Verification Analysis
c
/*
 * Typical License Verification Implementation
 * Shows what reverse engineers look for
 */

typedef struct {
    uint8_t user_id[16];
    uint32_t features;
    uint64_t expiration;
    uint8_t signature[32];  // HMAC-SHA256
} LICENSE_DATA;

/*
 * Vulnerable Implementation (Easy to crack)
 */
int verify_license_weak(const char *key) {
    // Simple string comparison
    if (strcmp(key, "VALID-KEY-12345") == 0) {
        return 1;
    }
    return 0;
    // Patch: Change JE to JNE at comparison, or NOP out check
}

/*
 * Better Implementation (Harder to crack)
 */
int verify_license_strong(const char *key) {
    LICENSE_DATA license;
    uint8_t computed_sig[32];
    
    // Decode base64
    size_t decoded_len;
    uint8_t *decoded = base64_decode(key, strlen(key), &decoded_len);
    
    if (decoded_len != sizeof(LICENSE_DATA)) {
        return 0;
    }
    
    memcpy(&license, decoded, sizeof(LICENSE_DATA));
    
    // Verify signature
    HMAC(EVP_sha256(), private_key, sizeof(private_key),
         (uint8_t*)&license, sizeof(LICENSE_DATA) - 32,
         computed_sig, NULL);
    
    if (memcmp(license.signature, computed_sig, 32) != 0) {
        return 0; // Invalid signature
    }
    
    // Check expiration
    if (license.expiration < time(NULL)) {
        return 0; // Expired
    }
    
    // Check user ID against hardware fingerprint
    uint8_t hw_id[16];
    get_hardware_fingerprint(hw_id);
    if (memcmp(license.user_id, hw_id, 16) != 0) {
        return 0; // Wrong machine
    }
    
    return 1;
}

/*
 * Hardware Fingerprinting
 */
void get_hardware_fingerprint(uint8_t *out) {
    // Collect hardware info
    char cpu_id[64] = {0};
    char mac[32] = {0};
    char disk_serial[64] = {0};
    
    // CPU ID via CPUID instruction
    int cpuinfo[4];
    __cpuid(cpuinfo, 0);
    memcpy(cpu_id, &cpuinfo[1], 4);
    memcpy(cpu_id + 4, &cpuinfo[3], 4);
    memcpy(cpu_id + 8, &cpuinfo[2], 4);
    
    // MAC address
    get_mac_address(mac);
    
    // Disk serial
    get_disk_serial(disk_serial);
    
    // Hash combination
    SHA256_CTX ctx;
    SHA256_Init(&ctx);
    SHA256_Update(&ctx, cpu_id, strlen(cpu_id));
    SHA256_Update(&ctx, mac, strlen(mac));
    SHA256_Update(&ctx, disk_serial, strlen(disk_serial));
    SHA256_Final(out, &ctx);
}
Keygen Development
python
#!/usr/bin/env python3
"""
License Key Generator
Reverse-engineered from target application
"""

import hmac
import hashlib
import struct
import base64
from datetime import datetime, timedelta

class LicenseGenerator:
    def __init__(self, private_key):
        """
        private_key: Extracted from application via reverse engineering
        """
        self.private_key = private_key
        
    def generate_key(self, user_id, features=0xFFFFFFFF, days_valid=365):
        """
        Generate valid license key
        """
        # Create license structure
        expiration = int((datetime.now() + timedelta(days=days_valid)).timestamp())
        
        # Pack data (must match application's structure)
        data = struct.pack('<16sIq', 
            user_id.encode().ljust(16, b'\x00'),
            features,
            expiration
        )
        
        # Calculate HMAC
        signature = hmac.new(self.private_key, data, hashlib.sha256).digest()
        
        # Append signature
        full_license = data + signature
        
        # Base64 encode
        key = base64.b64encode(full_license).decode()
        
        return key
    
    def generate_universal_key(self):
        """
        Key that works on any machine (bypasses HW check)
        Requires patching application or using wildcard user_id
        """
        # Use all zeros for user_id (may work if app doesn't check)
        return self.generate_key(b'\x00' * 16)
    
    def generate_perpetual_key(self):
        """
        Key that never expires
        """
        # Max timestamp
        expiration = 0xFFFFFFFFFFFFFFFF
        data = struct.pack('<16sIq', b'\x00' * 16, 0xFFFFFFFF, expiration)
        signature = hmac.new(self.private_key, data, hashlib.sha256).digest()
        return base64.b64encode(data + signature).decode()

class KeyValidator:
    """
    Validates keys against extracted algorithm
    """
    def __init__(self, private_key):
        self.private_key = private_key
        
    def validate(self, key):
        try:
            decoded = base64.b64decode(key)
            data = decoded[:-32]
            sig = decoded[-32:]
            
            expected_sig = hmac.new(self.private_key, data, hashlib.sha256).digest()
            
            if hmac.compare_digest(sig, expected_sig):
                user_id, features, expiration = struct.unpack('<16sIq', data)
                return {
                    'valid': True,
                    'user_id': user_id.rstrip(b'\x00').decode(),
                    'features': hex(features),
                    'expiration': datetime.fromtimestamp(expiration),
                    'expired': datetime.fromtimestamp(expiration) < datetime.now()
                }
        except:
            pass
        
        return {'valid': False}
8. Advanced Bypass Methods
DLL Injection for Runtime Patching
c
/*
 * DLL Injection and API Hooking
 * Used to bypass checks at runtime
 */

#include <windows.h>
#include <detours.h>

// Original function pointer
int (*original_verify_license)(const char*) = NULL;

// Hook function - always returns success
int hooked_verify_license(const char *key) {
    // Log the key for analysis
    FILE *f = fopen("C:\\keys.log", "a");
    fprintf(f, "Key tried: %s\n", key);
    fclose(f);
    
    // Return success regardless of key
    return 1;
}

BOOL APIENTRY DllMain(HMODULE hModule, DWORD reason, LPVOID lpReserved) {
    if (reason == DLL_PROCESS_ATTACH) {
        // Find target function
        HMODULE hTarget = GetModuleHandleA("target_app.exe");
        original_verify_license = (void*)GetProcAddress(hTarget, 
                                                        "VerifyLicense");
        
        // Install hook
        DetourTransactionBegin();
        DetourUpdateThread(GetCurrentThread());
        DetourAttach(&(PVOID&)original_verify_license, hooked_verify_license);
        DetourTransactionCommit();
    }
    return TRUE;
}

/*
 * Manual Memory Patching
 * Directly modify code in memory
 */
void patch_license_check() {
    // Find the license check function
    // Pattern scan for signature
    uint8_t pattern[] = {0x55, 0x48, 0x89, 0xE5, 0x48, 0x83, 0xEC};
    
    HMODULE hMod = GetModuleHandle(NULL);
    uint8_t *base = (uint8_t*)hMod;
    
    // Scan for pattern
    for (size_t i = 0; i < 0x100000; i++) {
        if (memcmp(base + i, pattern, sizeof(pattern)) == 0) {
            // Found function start
            uint8_t *func = base + i;
            
            // Change memory protection
            DWORD oldProtect;
            VirtualProtect(func, 32, PAGE_EXECUTE_READWRITE, &oldProtect);
            
            // Patch: mov eax, 1 ; ret
            // Returns success immediately
            uint8_t patch[] = {0xB8, 0x01, 0x00, 0x00, 0x00, 0xC3};
            memcpy(func, patch, sizeof(patch));
            
            VirtualProtect(func, 32, oldProtect, &oldProtect);
            break;
        }
    }
}
Network Authentication Bypass
python
#!/usr/bin/env python3
"""
Network Authentication Bypass
Intercepts and modifies authentication traffic
"""

from scapy.all import *
import ssl
import socket
import json

class AuthInterceptor:
    def __init__(self):
        self.target_host = "auth.target.com"
        self.target_port = 443
        
    def setup_proxy(self):
        """
        SSL stripping and request modification
        """
        context = ssl.create_default_context()
        
        with socket.create_connection((self.target_host, self.target_port)) as sock:
            with context.wrap_socket(sock, server_hostname=self.target_host) as ssock:
                # Intercept authentication request
                request = self.modify_auth_request(original_request)
                ssock.sendall(request)
                
                response = ssock.recv(4096)
                return self.modify_auth_response(response)
    
    def modify_auth_request(self, request):
        """
        Change request parameters to bypass checks
        """
        data = json.loads(request)
        
        # Modify fields
        data['admin'] = True
        data['verified'] = True
        data['subscription'] = 'premium'
        
        return json.dumps(data).encode()
    
    def certificate_pinning_bypass(self):
        """
        Bypass SSL certificate pinning in mobile apps
        """
        # Hook SSL verification functions
        # Return success regardless of certificate
        
        # Frida script for Android:
        frida_script = """
        Java.perform(function() {
            var X509TrustManager = Java.use('javax.net.ssl.X509TrustManager');
            var SSLContext = Java.use('javax.net.ssl.SSLContext');
            
            // Create custom TrustManager that accepts all certificates
            var TrustManager = Java.registerClass({
                name: 'com.example.TrustManager',
                implements: [X509TrustManager],
                methods: {
                    checkClientTrusted: function() {},
                    checkServerTrusted: function() {},
                    getAcceptedIssuers: function() { return null; }
                }
            });
            
            // Replace SSLContext's default TrustManager
            var TrustManagers = [TrustManager.$new()];
            var SSLContext_init = SSLContext.init.overload(
                '[Ljavax.net.ssl.KeyManager;', 
                '[Ljavax.net.ssl.TrustManager;', 
                'java.security.SecureRandom'
            );
            
            SSLContext_init.implementation = function(km, tm, random) {
                SSLContext_init.call(this, km, TrustManagers, random);
            };
        });
        """
        return frida_script
Summary
This document covered:

Cryptographic Foundations: AES, ChaCha20, RSA implementations and their weaknesses
File Encryption: Custom formats, key derivation, runtime decryption
Decryption Bypass: Memory dumping, API hooking, brute force attacks
Game Protection: DRM analysis, save file manipulation, VM protection
Anti-Tampering: Integrity verification, anti-dumping, code encryption
Anti-Analysis: VM detection, anti-debugging, timing analysis
Authentication: License verification, hardware fingerprinting, keygen development
Advanced Bypass: DLL injection, memory patching, network interception
Defense Recommendations:

Use white-box cryptography to hide keys
Implement multiple overlapping protection layers
Server-side critical verification
Code virtualization for sensitive algorithms
Integrity verification with anti-tamper
Environment detection with graceful degradation
This knowledge is essential for security researchers, protection developers, and penetration testers to understand both attack and defense perspectives.