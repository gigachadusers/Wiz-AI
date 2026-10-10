I'll create a comprehensive OPSEC guide covering technical surveillance vectors, legal discovery processes, and countermeasures. This is essential knowledge for privacy-conscious individuals, journalists, and security researchers.

Advanced Operational Security (OPSEC) and Digital Anonymity
Table of Contents
Legal Discovery and Subpoena Process
OSINT Account Linking Techniques
Technical Leak Vectors
Identity Segregation and Aliases
Hardware and Network Architecture
Operational Security Procedures
1. Legal Discovery and Subpoena Process
What Law Enforcement Can Obtain
python
#!/usr/bin/env python3
"""
Legal Discovery Simulation
Demonstrates what data providers retain and disclose
"""

class LegalDiscovery:
    """
    Simulates legal data requests and retention policies
    
    Understanding this is critical for threat modeling:
    - What data exists?
    - How long is it retained?
    - What can be compelled?
    """
    
    def __init__(self):
        self.retention_policies = {
            'gmail': {
                'email_content': 'indefinite',  # Until deleted
                'deleted_emails': '30 days',    # Trash retention
                'ip_logs': '9 months',          # Login IPs
                'metadata': '18 months',        # To/From/Subject/Time
                'location': 'indefinite',       # From IP geolocation
                'device_info': '28 days',       # Active sessions
                'search_history': '18 months',
            },
            'protonmail': {
                'email_content': 'encrypted',   # Provider cannot decrypt
                'ip_logs': 'none',              # No IP logging (claimed)
                'metadata': 'indefinite',       # Still logged
                'payment_info': 'indefinite',
            },
            'discord': {
                'messages': 'indefinite',       # Even deleted messages
                'dm_metadata': 'indefinite',
                'voice_data': 'not retained',    # But connection logs are
                'ip_logs': '30 days',
                'device_fingerprints': '90 days',
                'payment_history': '7 years',   # Tax requirements
            },
            'signal': {
                'message_content': 'none',       # End-to-end encrypted
                'contacts': 'hashed',            # Only hashed phone numbers
                'profile': 'minimal',            # Only what you provide
                'ip_logs': 'none',               # Claimed
                'delivery_status': '48 hours',   # Temporary
            },
            'telegram': {
                'secret_chats': 'none',          # E2E encrypted
                'cloud_chats': 'indefinite',     # Stored on servers
                'ip_logs': '12 months',
                'device_sessions': 'indefinite',
                'groups_metadata': 'indefinite',
            },
            'reddit': {
                'posts_comments': 'indefinite',  # Even "deleted" often archived
                'upvotes': 'indefinite',
                'ip_logs': '100 days',
                'mod_actions': 'indefinite',
                'awards_given': 'indefinite',
                'associated_emails': 'indefinite',
            },
            'twitter_x': {
                'tweets': 'indefinite',          # Even deleted (archived)
                'dms': 'indefinite',
                'ip_logs': '18 months',
                'device_info': 'indefinite',
                'location_history': 'indefinite',
                'login_history': '18 months',
                'associated_phone': 'indefinite',
            },
            'github': {
                'commits': 'indefinite',         # Immutable history
                'private_repos': 'indefinite',
                'ssh_keys': 'indefinite',
                'ip_logs': '90 days',
                'oauth_apps': 'indefinite',
                'gist_history': 'indefinite',
            },
            'isp_domestic': {
                'ip_assignment_logs': '12-24 months',  # Required by law
                'dns_queries': 'varies',              # Some log, some don't
                'traffic_content': 'not retained',     # Illegal without warrant
                'port_blocking_logs': '6 months',
                'copyright_notices': 'indefinite',
            },
            'vpn_provider': {
                'connection_logs': 'varies',     # "No logs" vs reality
                'bandwidth_usage': 'varies',
                'payment_info': 'depends',        # Crypto vs credit card
                'real_ip': 'often retained',      # Even "no logs" VPNs
                'timestamps': 'varies',
            }
        }
        
    def simulate_subpoena(self, provider: str, data_types: list) -> dict:
        """
        Simulate what a provider would return for a legal request
        
        Types of legal requests:
        - Subpoena: Basic subscriber info (name, IP, email, payment)
        - 2703(d) Order: Metadata and logs (USA)
        - Search Warrant: Content of communications
        - NSL (National Security Letter): Metadata, gag order
        - FISA Order: Foreign intelligence, secret
        """
        
        results = {}
        policy = self.retention_policies.get(provider, {})
        
        for data_type in data_types:
            retention = policy.get(data_type, 'unknown')
            
            if retention == 'none':
                results[data_type] = 'NOT_AVAILABLE'
            elif retention == 'encrypted':
                results[data_type] = 'ENCRYPTED_UNABLE_TO_DECRYPT'
            else:
                # Simulate returned data
                results[data_type] = self._generate_sample_data(
                    provider, data_type
                )
                
        return results
        
    def _generate_sample_data(self, provider: str, data_type: str) -> dict:
        """Generate realistic sample of what would be returned"""
        
        if data_type == 'ip_logs':
            return {
                '2024-01-15T08:23:14Z': {'ip': '203.0.113.45', 'action': 'login'},
                '2024-01-15T14:56:33Z': {'ip': '203.0.113.45', 'action': 'email_sent'},
                '2024-01-16T09:12:01Z': {'ip': '198.51.100.22', 'action': 'login'},
            }
        elif data_type == 'device_fingerprints':
            return {
                'user_agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)',
                'screen_resolution': '1920x1080',
                'timezone': 'America/New_York',
                'fonts': ['Arial', 'Times New Roman', 'Calibri'],
                'canvas_hash': 'a3f7d2e9...',
            }
        elif data_type == 'payment_info':
            return {
                'method': 'credit_card',
                'last4': '4242',
                'billing_zip': '10001',
                'name': 'John Smith',
                'transactions': [...]
            }
            
        return {'sample': 'data'}

class CrossProviderAnalysis:
    """
    How law enforcement links accounts across providers
    """
    
    def __init__(self):
        self.linkage_vectors = {
            'email_recovery': 'Gmail used as recovery for Reddit, Discord, etc.',
            'phone_number': 'Same number across multiple platforms',
            'ip_correlation': 'Same IP logged across different services',
            'device_fingerprint': 'Same browser/device signature',
            'payment_method': 'Same credit card across services',
            'behavioral_pattern': 'Writing style, activity times, interests',
            'social_graph': 'Same contacts/friends across platforms',
            'username_pattern': 'Similar usernames (johnsmith92, john_smith_92)',
            'profile_photos': 'Reverse image search links accounts',
            'writing_style': 'Stylometric analysis of text',
        }
        
    def analyze_linkage(self, accounts: dict) -> list:
        """
        Find correlations between different accounts
        
        accounts: {platform: account_data}
        """
        correlations = []
        
        # Check IP overlap
        ips = {}
        for platform, data in accounts.items():
            for ip in data.get('ips', []):
                if ip in ips:
                    correlations.append({
                        'type': 'ip_correlation',
                        'platforms': [ips[ip], platform],
                        'ip': ip,
                        'confidence': 'high'
                    })
                ips[ip] = platform
                
        # Check email recovery chains
        emails = {}
        for platform, data in accounts.items():
            recovery = data.get('recovery_email')
            if recovery:
                if recovery in emails:
                    correlations.append({
                        'type': 'email_recovery',
                        'platforms': [emails[recovery], platform],
                        'email': recovery,
                        'confidence': 'very_high'
                    })
                emails[recovery] = platform
                
        return correlations
Legal Process Deep Dive
python
"""
Understanding the legal framework for data access
"""

class LegalProcess:
    """
    Types of legal compulsion for data
    """
    
    def __init__(self):
        self.process_types = {
            'subpoena': {
                'scope': 'basic_subscriber_info',
                'requires': 'relevance_to_investigation',
                'notice': 'can_be_delayed_90_days',
                'gag_order': 'optional',
                'examples': ['name', 'email', 'signup_ip', 'payment_method']
            },
            '2703d_order': {
                'scope': 'metadata_and_logs',
                'requires': 'specific_and_articulable_facts',
                'notice': 'can_be_delayed_90_days',
                'gag_order': 'optional',
                'examples': ['email_headers', 'login_times', 'message_metadata']
            },
            'search_warrant': {
                'scope': 'content_of_communications',
                'requires': 'probable_cause',
                'notice': 'required_but_can_be_delayed',
                'gag_order': 'possible',
                'examples': ['email_content', 'message_content', 'files']
            },
            'nsl': {
                'scope': 'metadata_only',
                'requires': 'relevant_to_authorized_investigation',
                'notice': 'prohibited',
                'gag_order': 'automatic_permanent',
                'examples': ['to/from addresses', 'timestamps', 'ip_logs']
            },
            'fisa_order': {
                'scope': 'foreign_intelligence',
                'requires': 'foreign_power_affiliation',
                'court': 'fisa_court',
                'notice': 'rarely',
                'gag_order': 'automatic',
            },
            'mlat': {
                'scope': 'varies_by_country',
                'process': 'mutual_legal_assistance_treaty',
                'timeline': 'months_to_years',
                'examples': ['international_data_requests']
            }
        }
        
    def provider_response_process(self, provider: str, legal_request: dict):
        """
        How providers handle legal requests
        
        Steps:
        1. Legal team reviews validity
        2. Check scope (overbroad requests challenged)
        3. Notify user if legally allowed
        4. Gather responsive data
        5. Deliver via secure portal
        6. Preserve chain of custody
        """
        
        # Major providers have transparency reports
        transparency_data = {
            'google': {
                'requests_per_year': 100000,
                'percentage_complied': 75,
                'challenged': 15,
            },
            'meta': {
                'requests_per_year': 200000,
                'percentage_complied': 70,
            },
            'apple': {
                'requests_per_year': 15000,
                'percentage_complied': 80,
            },
            'microsoft': {
                'requests_per_year': 50000,
                'percentage_complied': 65,
            }
        }
2. OSINT Account Linking Techniques
python
#!/usr/bin/env python3
"""
OSINT Account Linking Framework
Demonstrates how investigators correlate identities
"""

import hashlib
import re
import json
from typing import List, Dict, Set, Tuple
from dataclasses import dataclass

@dataclass
class DigitalIdentity:
    """
    Represents a digital identity across platforms
    """
    platform: str
    username: str
    display_name: str
    email: str
    phone: str
    bio: str
    avatar_url: str
    created_at: str
    last_active: str
    associated_ips: List[str]
    user_agent: str
    timezone: str
    writing_samples: List[str]

class OSINTLinker:
    """
    Open Source Intelligence linking techniques
    
    These are the methods used by investigators, journalists,
    and threat researchers to correlate online identities
    """
    
    def __init__(self):
        self.linkage_scores = {}
        
    def username_similarity_analysis(self, identities: List[DigitalIdentity]) -> List[dict]:
        """
        Find similar usernames across platforms
        
        Techniques:
        - Exact match
        - Levenshtein distance (typosquatting)
        - Pattern analysis (user123 -> user_123)
        - Common substitutions (o -> 0, l -> 1)
        """
        
        links = []
        
        for i, id1 in enumerate(identities):
            for id2 in identities[i+1:]:
                if id1.platform == id2.platform:
                    continue
                    
                score = self._calculate_username_similarity(
                    id1.username, id2.username
                )
                
                if score > 0.8:
                    links.append({
                        'id1': (id1.platform, id1.username),
                        'id2': (id2.platform, id2.username),
                        'similarity': score,
                        'method': 'username_similarity'
                    })
                    
        return links
        
    def _calculate_username_similarity(self, u1: str, u2: str) -> float:
        """Calculate similarity between usernames"""
        from difflib import SequenceMatcher
        
        # Normalize: lowercase, remove separators
        n1 = re.sub(r'[_\-\.]', '', u1.lower())
        n2 = re.sub(r'[_\-\.]', '', u2.lower())
        
        # Exact match
        if n1 == n2:
            return 1.0
            
        # Check for common patterns
        # user123 -> user_123, user-123, user.123
        if re.sub(r'\d+', '', n1) == re.sub(r'\d+', '', n2):
            return 0.9
            
        # Levenshtein distance
        return SequenceMatcher(None, n1, n2).ratio()
        
    def email_analysis(self, identities: List[DigitalIdentity]) -> List[dict]:
        """
        Analyze email patterns for correlation
        
        Patterns:
        - Same email (obvious link)
        - Plus addressing: user+tag@gmail.com
        - Period manipulation: u.ser@gmail.com == user@gmail.com
        - Different providers forwarding to same inbox
        """
        
        links = []
        
        for i, id1 in enumerate(identities):
            for id2 in identities[i+1:]:
                normalized1 = self._normalize_email(id1.email)
                normalized2 = self._normalize_email(id2.email)
                
                if normalized1 == normalized2:
                    links.append({
                        'id1': (id1.platform, id1.username),
                        'id2': (id2.platform, id2.username),
                        'email': normalized1,
                        'method': 'email_correlation'
                    })
                elif self._email_domain_correlation(id1.email, id2.email):
                    links.append({
                        'id1': (id1.platform, id1.username),
                        'id2': (id2.platform, id2.username),
                        'method': 'email_domain_pattern'
                    })
                    
        return links
        
    def _normalize_email(self, email: str) -> str:
        """Normalize Gmail and similar for comparison"""
        if '@gmail.com' in email:
            local, domain = email.split('@')
            # Remove everything after +
            local = local.split('+')[0]
            # Remove dots
            local = local.replace('.', '')
            return f"{local}@{domain}"
        return email.lower()
        
    def stylometric_analysis(self, identities: List[DigitalIdentity]) -> List[dict]:
        """
        Writing style analysis (stylometry)
        
        Identifies same author via:
        - Vocabulary richness (type-token ratio)
        - Average word/sentence length
        - Punctuation patterns
        - Capitalization habits
        - Emoticon usage
        - Misspelling patterns
        - Function word frequencies
        """
        
        links = []
        
        for i, id1 in enumerate(identities):
            for id2 in identities[i+1:]:
                if not id1.writing_samples or not id2.writing_samples:
                    continue
                    
                score = self._calculate_stylometric_similarity(
                    id1.writing_samples,
                    id2.writing_samples
                )
                
                if score > 0.85:
                    links.append({
                        'id1': (id1.platform, id1.username),
                        'id2': (id2.platform, id2.username),
                        'similarity': score,
                        'method': 'stylometric_analysis'
                    })
                    
        return links
        
    def _calculate_stylometric_similarity(self, 
                                          samples1: List[str],
                                          samples2: List[str]) -> float:
        """
        Calculate stylometric similarity
        
        Features:
        - Character-level: avg word length, capital ratio
        - Syntactic: punctuation per sentence, sentence length
        - Lexical: vocabulary overlap, unique word ratio
        """
        
        text1 = ' '.join(samples1)
        text2 = ' '.join(samples2)
        
        features1 = self._extract_features(text1)
        features2 = self._extract_features(text2)
        
        # Cosine similarity of feature vectors
        return self._cosine_similarity(features1, features2)
        
    def _extract_features(self, text: str) -> dict:
        """Extract stylometric features"""
        import string
        
        words = text.split()
        sentences = text.split('.')
        
        features = {
            'avg_word_length': sum(len(w) for w in words) / len(words) if words else 0,
            'avg_sentence_length': len(words) / len(sentences) if sentences else 0,
            'capital_ratio': sum(1 for c in text if c.isupper()) / len(text),
            'punctuation_ratio': sum(1 for c in text if c in string.punctuation) / len(text),
            'emoticon_count': len(re.findall(r'[:;]-?[)\]}|DPO]', text)),
            'unique_word_ratio': len(set(w.lower() for w in words)) / len(words) if words else 0,
        }
        
        return features
        
    def temporal_analysis(self, identities: List[DigitalIdentity]) -> List[dict]:
        """
        Activity pattern analysis
        
        Links accounts with similar:
        - Activity times (timezone inference)
        - Sleep patterns
        - Posting frequency
        - Response latency
        """
        
        links = []
        
        for i, id1 in enumerate(identities):
            for id2 in identities[i+1:]:
                # Compare timezone
                if id1.timezone == id2.timezone:
                    links.append({
                        'id1': (id1.platform, id1.username),
                        'id2': (id2.platform, id2.username),
                        'timezone': id1.timezone,
                        'method': 'timezone_correlation'
                    })
                    
                # Compare activity patterns
                if self._activity_pattern_match(id1, id2):
                    links.append({
                        'id1': (id1.platform, id1.username),
                        'id2': (id2.platform, id2.username),
                        'method': 'activity_pattern'
                    })
                    
        return links
        
    def avatar_analysis(self, identities: List[DigitalIdentity]) -> List[dict]:
        """
        Image-based account linking
        
        Techniques:
        - Perceptual hashing (pHash)
        - EXIF metadata extraction
        - Reverse image search
        - Facial recognition
        """
        
        links = []
        
        for i, id1 in enumerate(identities):
            for id2 in identities[i+1:]:
                if not id1.avatar_url or not id2.avatar_url:
                    continue
                    
                # Perceptual hash comparison
                hash1 = self._perceptual_hash(id1.avatar_url)
                hash2 = self._perceptual_hash(id2.avatar_url)
                
                if self._hamming_distance(hash1, hash2) < 10:
                    links.append({
                        'id1': (id1.platform, id1.username),
                        'id2': (id2.platform, id2.username),
                        'method': 'perceptual_hash_match'
                    })
                    
        return links
        
    def _perceptual_hash(self, image_url: str) -> str:
        """
        Calculate perceptual hash of image
        
        Resistant to:
        - Resizing
        - Minor edits
        - Compression
        - Color adjustments
        """
        # Implementation would use imagehash library
        pass

class SocialGraphAnalysis:
    """
    Analyze social connections to link identities
    """
    
    def __init__(self):
        self.graph = {}
        
    def build_interaction_graph(self, platform_data: dict):
        """
        Build graph of who interacts with whom
        
        Strong indicator: same set of friends/interactions
        across different platforms
        """
        
        for platform, data in platform_data.items():
            for user, interactions in data.items():
                if user not in self.graph:
                    self.graph[user] = {}
                    
                self.graph[user][platform] = interactions
                
    def find_matching_graphs(self) -> List[dict]:
        """
        Find users with similar social graphs
        
        Jaccard similarity of friend sets
        """
        
        matches = []
        users = list(self.graph.keys())
        
        for i, u1 in enumerate(users):
            for u2 in users[i+1:]:
                # Compare friend sets across platforms
                similarity = self._jaccard_similarity(
                    set(self.graph[u1].get('friends', [])),
                    set(self.graph[u2].get('friends', []))
                )
                
                if similarity > 0.7:
                    matches.append({
                        'user1': u1,
                        'user2': u2,
                        'similarity': similarity,
                        'common_friends': len(
                            set(self.graph[u1].get('friends', [])) &
                            set(self.graph[u2].get('friends', []))
                        )
                    })
                    
        return matches
3. Technical Leak Vectors
python
#!/usr/bin/env python3
"""
Technical Leak Vectors and Detection
"""

import socket
import struct
import requests
import json
from typing import Dict, Optional

class LeakDetector:
    """
    Detect various types of information leaks
    """
    
    def __init__(self):
        self.leaks_found = []
        
    def check_webrtc_leak(self) -> Dict:
        """
        WebRTC IP Leak Detection
        
        WebRTC establishes peer-to-peer connections for video/audio.
        To do this, it queries local IP addresses via ICE candidates.
        
        Even with VPN, WebRTC can leak local IP and sometimes
        real public IP if VPN doesn't handle WebRTC properly.
        
        STUN servers return: public IP, port, local IP
        """
        
        # Simulate WebRTC behavior
        stun_servers = [
            'stun.l.google.com:19302',
            'stun1.l.google.com:19302',
            'stun.voiparound.com',
        ]
        
        leaks = {
            'local_ips': [],
            'public_ips': [],
            'vpn_bypass_possible': False
        }
        
        for stun_server in stun_servers:
            try:
                # Create UDP socket
                sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
                sock.settimeout(5)
                
                # STUN binding request
                # 20 bytes header + 4 bytes magic cookie + 12 bytes transaction ID
                transaction_id = b'\x00' * 12
                magic_cookie = b'\x21\x12\xA4\x42'
                
                # Binding Request
                msg_type = b'\x00\x01'  # Binding Request
                msg_len = b'\x00\x00'   # No attributes initially
                
                request = msg_type + msg_len + magic_cookie + transaction_id
                
                # Add fingerprint attribute (RFC 5389)
                # Simplified - real implementation calculates CRC32
                
                host, port = stun_server.split(':')
                sock.sendto(request, (host, int(port)))
                
                response, addr = sock.recvfrom(1024)
                
                # Parse STUN response
                # Look for XOR-MAPPED-ADDRESS attribute (0x0020)
                if len(response) > 20:
                    # Parse attributes
                    idx = 20  # After header
                    while idx < len(response):
                        attr_type = struct.unpack('>H', response[idx:idx+2])[0]
                        attr_len = struct.unpack('>H', response[idx+2:idx+4])[0]
                        
                        if attr_type == 0x0020:  # XOR-MAPPED-ADDRESS
                            # Parse IP and port
                            port = struct.unpack('>H', response[idx+6:idx+8])[0]
                            # XOR with magic cookie
                            port ^= 0x1221
                            
                            ip_bytes = response[idx+8:idx+12]
                            # XOR with magic cookie
                            xor_ip = bytes([
                                ip_bytes[i] ^ magic_cookie[i] 
                                for i in range(4)
                            ])
                            ip = '.'.join(str(b) for b in xor_ip)
                            
                            leaks['public_ips'].append(ip)
                            
                        idx += 4 + attr_len + (4 - attr_len % 4) % 4  # Padding
                        
                sock.close()
                
            except Exception as e:
                pass
                
        # Check if leaked IP matches VPN IP
        vpn_ip = self._get_current_ip()
        if leaks['public_ips'] and vpn_ip not in leaks['public_ips']:
            leaks['vpn_bypass_possible'] = True
            
        return leaks
        
    def check_dns_leak(self) -> Dict:
        """
        DNS Leak Detection
        
        When using VPN, DNS queries should go through VPN tunnel.
        If they go through local DNS, ISP can see visited domains.
        
        Types:
        - Standard DNS leak (using ISP DNS)
        - IPv6 leak (IPv6 traffic bypasses IPv4 VPN)
        - Time-based leak (VPN reconnects, DNS still local)
        - Transparent DNS proxy (ISP intercepts port 53)
        """
        
        leaks = {
            'dns_servers': [],
            'isp_dns_detected': False,
            'ipv6_leak': False,
            'transparent_proxy': False
        }
        
        # Method 1: Check which DNS servers are being used
        try:
            # This queries a special domain that returns your DNS server
            import dns.resolver
            
            # Test resolver
            resolver = dns.resolver.Resolver()
            
            # Query for DNS server detection
            try:
                answers = resolver.resolve('whoami.ultradns.net', 'TXT')
                for rdata in answers:
                    dns_info = str(rdata).strip('"')
                    leaks['dns_servers'].append(dns_info)
            except:
                pass
                
            # Check if using ISP DNS
            # Compare against known public DNS
            public_dns = ['8.8.8.8', '1.1.1.1', '9.9.9.9']
            
            # Get current resolver
            import subprocess
            result = subprocess.run(
                ['nslookup', 'example.com'],
                capture_output=True, text=True
            )
            
            # Parse output for server
            for line in result.stdout.split('\n'):
                if 'Server:' in line:
                    server = line.split(':')[1].strip()
                    if server not in public_dns:
                        leaks['isp_dns_detected'] = True
                        
        except:
            pass
            
        # Method 2: Check IPv6
        try:
            # Try to connect via IPv6
            sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
            sock.settimeout(2)
            sock.connect(('2001:4860:4860::8888', 53))
            leaks['ipv6_leak'] = True
            sock.close()
        except:
            pass
            
        # Method 3: Check for transparent DNS proxy
        # Query random subdomain - should not be cached
        import uuid
        test_domain = f"{uuid.uuid4().hex}.test.dnsleaktest.com"
        
        try:
            answers = dns.resolver.resolve(test_domain, 'A')
            # If we get an answer, there's likely a transparent proxy
            # (recursive resolver shouldn't have this)
            leaks['transparent_proxy'] = True
        except:
            pass
            
        return leaks
        
    def check_browser_fingerprint(self) -> Dict:
        """
        Browser Fingerprinting Detection
        
        Modern browsers leak enormous amounts of identifying information.
        Even with VPN, fingerprinting can track users across sessions.
        
        Components:
        - Canvas/WebGL fingerprinting
        - Font enumeration
        - Screen resolution + color depth
        - Timezone + language
        - Installed plugins
        - User agent
        - Touch support
        - Battery API (deprecated but still present)
        """
        
        fingerprint = {
            'screen': {},
            'browser': {},
            'hardware': {},
            'unique_identifiers': []
        }
        
        # Screen information
        fingerprint['screen'] = {
            'width': 'detected_via_js',
            'height': 'detected_via_js',
            'color_depth': 'detected_via_js',
            'pixel_ratio': 'detected_via_js',
            'touch_support': 'detected_via_js',
        }
        
        # Canvas fingerprinting
        # Different GPUs/drivers produce slightly different renderings
        # of the same canvas commands
        fingerprint['canvas'] = {
            'hash': 'calculated_from_rendering',
            'webgl_vendor': 'detected_via_js',
            'webgl_renderer': 'detected_via_js',
            'webgl_unmasked': 'detected_via_extension',
        }
        
        # Font enumeration
        # List of installed fonts is highly identifying
        fingerprint['fonts'] = {
            'count': 'detected_via_js',
            'list': 'detected_via_side_channel',
        }
        
        # Audio fingerprinting
        # AudioContext processing varies by hardware
        fingerprint['audio'] = {
            'context_hash': 'calculated_from_oscillator',
        }
        
        # Calculate uniqueness
        # Most browsers have unique fingerprints
        fingerprint['entropy'] = 'high'  # Usually 18+ bits
        
        return fingerprint
        
    def check_geolocation_leak(self) -> Dict:
        """
        Geolocation Leak Detection
        
        Sources:
        - GPS (mobile)
        - WiFi network database (Google/Mozilla)
        - Cell tower triangulation
        - IP geolocation (least accurate)
        """
        
        leaks = {
            'methods_available': [],
            'precision': None,
            'last_known_location': None
        }
        
        # Check HTML5 Geolocation API
        leaks['methods_available'].append('html5_geolocation')
        
        # Check WiFi positioning
        # Requires nearby SSID list
        leaks['methods_available'].append('wifi_positioning')
        
        return leaks

class FingerprintRandomization:
    """
    Techniques to prevent fingerprinting
    """
    
    def __init__(self):
        self.profiles = self._generate_profiles()
        
    def _generate_profiles(self) -> List[Dict]:
        """Generate realistic browser profiles"""
        
        profiles = []
        
        # Common configurations
        configs = [
            {
                'os': 'Windows 10',
                'browser': 'Chrome 120',
                'resolution': '1920x1080',
                'timezone': 'America/New_York',
                'language': 'en-US',
                'fonts': ['Arial', 'Times', 'Courier', 'Verdana'],
            },
            {
                'os': 'macOS 14',
                'browser': 'Safari 17',
                'resolution': '2560x1440',
                'timezone': 'America/Los_Angeles',
                'language': 'en-US',
                'fonts': ['Helvetica', 'Times', 'Courier', 'Geneva'],
            },
            {
                'os': 'Ubuntu 22.04',
                'browser': 'Firefox 121',
                'resolution': '1920x1080',
                'timezone': 'Europe/London',
                'language': 'en-GB',
                'fonts': ['DejaVu Sans', 'Liberation Serif', 'Ubuntu Mono'],
            },
        ]
        
        return configs
        
    def get_random_profile(self) -> Dict:
        """Return randomized profile for session"""
        import random
        return random.choice(self.profiles)
4. Identity Segregation and Aliases
python
#!/usr/bin/env python3
"""
Identity Segregation Framework

Complete separation of identities to prevent correlation
"""

import secrets
import hashlib
from typing import Dict, List
from dataclasses import dataclass

@dataclass
class Identity:
    """
    Complete identity profile
    
    Each identity must be completely isolated:
    - No shared hardware
    - No shared networks
    - No shared accounts
    - No shared payment methods
    - No shared behavior patterns
    """
    
    name: str
    email: str
    phone: str
    birthdate: str
    address: str
    payment_method: str
    hardware_profile: str
    network_profile: str
    behavioral_profile: str
    
    # Operational details
    purpose: str  # What this identity is for
    risk_level: str  # low/medium/high
    compartment: str  # Which compartment this belongs to

class IdentityManager:
    """
    Manage multiple isolated identities
    
    Principles:
    1. One identity per purpose
    2. No cross-contamination
    3. Regular rotation
    4. Plausible deniability
    """
    
    def __init__(self):
        self.identities: Dict[str, Identity] = {}
        self.compartment_rules = {
            'strict': 'no_shared_anything',
            'medium': 'shared_hardware_ok',
            'low': 'shared_network_ok',
        }
        
    def create_identity(self, purpose: str, compartment: str = 'strict') -> Identity:
        """
        Create new isolated identity
        
        Steps:
        1. Generate persona (name, DOB, address)
        2. Acquire phone number (5sim, etc.)
        3. Create email (ProtonMail, Tutanota)
        4. Set up payment (crypto, prepaid cards)
        5. Configure dedicated hardware/VM
        6. Establish network isolation (dedicated VPN exit)
        """
        
        identity = Identity(
            name=self._generate_name(),
            email=self._create_email(),
            phone=self._acquire_phone_number(),
            birthdate=self._generate_birthdate(),
            address=self._generate_address(),
            payment_method=self._setup_payment(),
            hardware_profile=self._assign_hardware(compartment),
            network_profile=self._assign_network(compartment),
            behavioral_profile=self._generate_behavior(),
            purpose=purpose,
            risk_level='medium',
            compartment=compartment
        )
        
        self.identities[identity.email] = identity
        return identity
        
    def _generate_name(self) -> str:
        """Generate believable fake name"""
        first_names = ['James', 'John', 'Robert', 'Michael', 'William',
                      'Mary', 'Patricia', 'Jennifer', 'Linda', 'Elizabeth']
        last_names = ['Smith', 'Johnson', 'Williams', 'Brown', 'Jones',
                     'Garcia', 'Miller', 'Davis', 'Rodriguez', 'Martinez']
        
        import random
        return f"{random.choice(first_names)} {random.choice(last_names)}"
        
    def _create_email(self) -> str:
        """Create email with privacy-focused provider"""
        # ProtonMail, Tutanota, etc.
        # Enable 2FA with hardware key
        # No recovery email
        # No phone number if possible
        pass
        
    def _acquire_phone_number(self) -> str:
        """
        Acquire temporary phone number
        
        Services:
        - 5sim.net (SMS verification)
        - sms-activate.org
        - receive-smss.com
        - Burner (app)
        
        Considerations:
        - Country of number (affects verification success)
        - Reuse (never reuse numbers across identities)
        - Provider blocklists (some services block VoIP numbers)
        """
        
        # API integration with 5sim.net
        # Purchase number for specific service
        # Retrieve SMS via API
        # Release number after use
        
        return "+1-XXX-XXX-XXXX"
        
    def _setup_payment(self) -> str:
        """
        Anonymous payment setup
        
        Methods (most to least anonymous):
        1. Monero (XMR) - untraceable
        2. Bitcoin via CoinJoin - good privacy
        3. Prepaid gift cards (bought with cash)
        4. Privacy.com virtual cards (links to real identity)
        5. Prepaid debit cards (requires ID in many countries)
        """
        
        # Generate Monero wallet
        # Use decentralized exchange (Bisq) to acquire
        # Never reuse addresses
        # Use subaddresses for each transaction
        
        return "xmr_wallet_address"
        
    def _assign_hardware(self, compartment: str) -> str:
        """
        Assign dedicated hardware
        
        Options:
        1. Dedicated laptop (Qubes OS recommended)
        2. VM with GPU passthrough (for fingerprint resistance)
        3. Live USB with persistence (Tails)
        4. Raspberry Pi (for low-risk)
        
        Strict compartment: physical machine
        Medium: VM with dedicated virtual hardware
        """
        
        if compartment == 'strict':
            return "dedicated_physical_machine"
        elif compartment == 'medium':
            return "dedicated_vm_with_gpu_passthrough"
        else:
            return "isolated_browser_profile"
            
    def _assign_network(self, compartment: str) -> str:
        """
        Assign dedicated network profile
        
        Each identity gets:
        - Dedicated VPN exit node (dedicated IP)
        - Or Tor circuit (for highest risk)
        - No shared WiFi (different physical location)
        """
        
        # VPN with dedicated IP
        # Or Tor with specific exit node country
        # MAC address spoofing
        
        return "dedicated_vpn_exit_node"
        
    def _generate_behavior(self) -> str:
        """
        Generate behavioral profile
        
        To avoid stylometric analysis:
        - Different writing style
        - Different activity times
        - Different interests
        - Different social circles
        """
        
        return "unique_behavioral_profile"

class Compartmentalization:
    """
    Strict compartmentalization rules
    """
    
    def __init__(self):
        self.rules = {
            'hardware': {
                'strict': 'dedicated_physical_device',
                'medium': 'dedicated_vm',
                'low': 'dedicated_browser_profile',
            },
            'network': {
                'strict': 'dedicated_vpn_ip_or_tor',
                'medium': 'shared_vpn_different_exit',
                'low': 'same_network_ok',
            },
            'temporal': {
                'strict': 'different_activity_hours',
                'medium': 'slight_variation_ok',
                'low': 'same_schedule_ok',
            },
            'behavioral': {
                'strict': 'completely_different_persona',
                'medium': 'different_interests',
                'low': 'same_persona_ok',
            }
        }
        
    def check_isolation(self, identity1: Identity, identity2: Identity) -> bool:
        """
        Verify two identities are properly isolated
        """
        
        checks = [
            identity1.hardware_profile != identity2.hardware_profile,
            identity1.network_profile != identity2.network_profile,
            identity1.behavioral_profile != identity2.behavioral_profile,
            identity1.payment_method != identity2.payment_method,
        ]
        
        return all(checks)
5. Hardware and Network Architecture
python
#!/usr/bin/env python3
"""
Hardware and Network Security Architecture
"""

class HardwareRecommendations:
    """
    Hardware selection for anonymity
    
    Principles:
    1. Open source where possible (auditability)
    2. No Intel ME / AMD PSP (or disabled)
    3. No proprietary firmware
    4. Hardware kill switches
    5. Removable components
    """
    
    def __init__(self):
        self.recommendations = {
            'laptop': {
                'best': {
                    'device': 'Purism Librem 14',
                    'reasons': [
                        'Hardware kill switches (mic/camera/wifi)',
                        'Intel ME neutralized',
                        'Coreboot instead of proprietary BIOS',
                        'Open source firmware',
                        'Qubes OS certified',
                    ]
                },
                'good': {
                    'device': 'ThinkPad X230/X220 with Coreboot',
                    'reasons': [
                        'Can flash Coreboot/Libreboot',
                        'No Intel ME (if disabled)',
                        'Well-supported by Qubes',
                        'Inexpensive for dedicated machines',
                    ]
                },
                'acceptable': {
                    'device': 'System76 laptops',
                    'reasons': [
                        'Open source firmware available',
                        'Linux-first',
                        'No Windows tax',
                    ]
                }
            },
            'desktop': {
                'best': {
                    'device': 'Custom build with ASUS KGPE-D16',
                    'reasons': [
                        'Libreboot supported',
                        'No proprietary BIOS',
                        'Dual CPU, lots of RAM for VMs',
                    ]
                }
            },
            'mobile': {
                'best': {
                    'device': 'Pixel with GrapheneOS or CalyxOS',
                    'reasons': [
                        'Verified boot',
                        'Android hardened',
                        'No Google services (optional)',
                        'Regular security updates',
                    ]
                },
                'good': {
                    'device': 'iPhone (for threat model)',
                    'reasons': [
                        'Strong encryption',
                        'Good security track record',
                        'No carrier bloatware',
                    ]
                }
            },
            'router': {
                'best': {
                    'device': 'Protectli Vault with pfSense/OPNsense',
                    'reasons': [
                        'Open source firewall',
                        'No backdoors',
                        'Full control',
                        'VPN client/server',
                    ]
                },
                'good': {
                    'device': 'Turris Omnia',
                    'reasons': [
                        'OpenWrt based',
                        'Automatic updates',
                        'Open hardware',
                    ]
                },
                'budget': {
                    'device': 'Old PC with dual NIC + pfSense',
                    'reasons': [
                        'Inexpensive',
                        'Powerful',
                        'Full x86 compatibility',
                    ]
                }
            }
        }

class NetworkArchitecture:
    """
    Secure network architecture for anonymity
    """
    
    def __init__(self):
        self.setup_guide = {}
        
    def design_anonymous_network(self, threat_model: str) -> dict:
        """
        Design network based on threat model
        
        threat_model: 'journalist', 'activist', 'whistleblower', 'researcher'
        """
        
        if threat_model == 'whistleblower':
            return self._whistleblower_setup()
        elif threat_model == 'journalist':
            return self._journalist_setup()
        elif threat_model == 'activist':
            return self._activist_setup()
            
    def _whistleblower_setup(self) -> dict:
        """
        Maximum security setup
        
        Architecture:
        [Internet] -> [Tor] -> [VPN] -> [Tor again] -> [Target]
        
        Or: Air-gapped machine for sensitive work
        """
        
        return {
            'description': 'Maximum anonymity',
            'hardware': {
                'primary': 'Purism Librem 14 with Qubes OS',
                'secondary': 'Tails on USB (emergency)',
                'mobile': 'Pixel with GrapheneOS (no SIM)',
            },
            'network': {
                'connection': 'Public WiFi (never home)',
                'or': 'Dedicated Tor connection',
                'vpn': 'Mullvad or IVPN (paid with Monero)',
                'mac_spoofing': 'Random MAC every connection',
                'dns': 'DNS over HTTPS to Quad9',
            },
            'procedures': {
                'location': 'Never use from home or work',
                'timing': 'Random times, never routine',
                'physical_security': 'Faraday bag for devices',
                'communication': 'Signal with disappearing messages',
                'storage': 'Encrypted USB, hidden',
            },
            'isolation': {
                'work_machine': 'Never connects to personal networks',
                'personal_machine': 'Never used for work',
                'separation': 'Physical distance between uses',
            }
        }
        
    def _journalist_setup(self) -> dict:
        """
        Balanced security and usability
        """
        
        return {
            'description': 'Strong security, usable',
            'hardware': {
                'primary': 'ThinkPad with Qubes OS',
                'mobile': 'iPhone with Lockdown Mode',
            },
            'network': {
                'vpn': 'ProtonVPN or Mullvad',
                'dns': 'Cloudflare 1.1.1.1',
                'browser': 'Tor Browser for sensitive research',
            },
            'procedures': {
                'source_communication': 'Signal or Session',
                'document_handling': 'Air-gapped machine',
                'travel': 'Tails USB for foreign travel',
            }
        }
        
    def _activist_setup(self) -> dict:
        """
        Group security, plausible deniability
        """
        
        return {
            'description': 'Group operational security',
            'hardware': {
                'cheap_laptops': 'Multiple, disposable',
                'mobile': 'Burner phones, cash only',
            },
            'network': {
                'mesh_networking': 'Calyx Institute or similar',
                'communication': 'Signal groups, disappearing messages',
                'coordination': 'In-person only for sensitive plans',
            },
            'procedures': {
                'need_to_know': 'Compartmentalize information',
                'security_culture': 'Regular training',
                'legal_support': 'NLG or similar prepared',
            }
        }

class RouterConfiguration:
    """
    Secure router configuration
    """
    
    def __init__(self):
        self.config = {}
        
    def generate_pfsense_config(self) -> dict:
        """
        Generate secure pfSense configuration
        
        Features:
        - VPN client (all traffic through VPN)
        - DNS over TLS
        - Ad blocking (pfBlockerNG)
        - IDS/IPS (Suricata)
        - VLANs for device isolation
        - Captive portal for guests
        """
        
        return {
            'interfaces': {
                'wan': 'DHCP with MAC spoofing',
                'lan': '192.168.1.1/24',
                'iot_vlan': '192.168.10.1/24',
                'guest_vlan': '192.168.20.1/24',
            },
            'vpn': {
                'client': 'WireGuard to Mullvad',
                'kill_switch': True,  # Block if VPN down
                'dns_leak_protection': True,
            },
            'dns': {
                'resolver': 'Unbound',
                'forwarding': 'DNS over TLS to Quad9',
                'blocking': 'pfBlockerNG with threat feeds',
            },
            'firewall': {
                'default_deny': True,
                'rules': [
                    'Block IoT from LAN',
                    'Block all from WAN except VPN',
                    'Allow established/related',
                ]
            },
            'services': {
                'dhcp': 'Enabled with static mappings',
                'ntp': 'Pool with no leak',
                'snmp': 'Disabled',
                'sshd': 'Key auth only, non-standard port',
            }
        }
        
    def generate_openwrt_config(self) -> dict:
        """
        OpenWrt configuration for travel router
        """
        
        return {
            'device': 'GL.iNet AR750S or similar',
            'features': [
                'VPN client (WireGuard)',
                'Tor transparent proxy',
                'Ad blocking (AdGuard Home)',
                'MAC randomization',
                'Auto-connect to open WiFi with VPN',
            ],
            'use_case': 'Travel security',
            'configuration': {
                'wan': 'Connect to hotel/airport WiFi',
                'lan': 'Your own secure network',
                'vpn': 'Always on',
                'tor': 'Optional for maximum anonymity',
            }
        }
6. Operational Security Procedures
python
#!/usr/bin/env python3
"""
Operational Security Procedures and Checklists
"""

class OPSECProcedures:
    """
    Day-to-day operational security procedures
    """
    
    def __init__(self):
        self.checklists = {}
        
    def daily_security_checklist(self) -> List[str]:
        """
        Daily security verification
        """
        
        return [
            "☐ Check VPN connection is active",
            "☐ Verify DNS is not leaking (dnsleaktest.com)",
            "☐ Check WebRTC is disabled or leaking",
            "☐ Verify browser fingerprint is randomized",
            "☐ Check system time matches VPN timezone",
            "☐ Verify no unexpected network connections",
            "☐ Check for system updates",
            "☐ Review identity compartment before actions",
            "☐ Clear browser data if necessary",
            "☐ Verify encrypted storage is mounted",
        ]
        
    def before_sensitive_action(self) -> List[str]:
        """
        Pre-action security verification
        """
        
        return [
            "☐ Am I using the correct identity for this action?",
            "☐ Is my network connection secure (VPN/Tor)?",
            "☐ Have I disabled JavaScript if not needed?",
            "☐ Is my screen positioned away from cameras?",
            "☐ Are my devices in a Faraday bag if needed?",
            "☐ Have I verified the recipient's identity?",
            "☐ Is this communication end-to-end encrypted?",
            "☐ Do I have a dead man's switch configured?",
            "☐ Is my location secure and private?",
            "☐ Have I considered the legal implications?",
        ]
        
    def communication_security(self) -> Dict:
        """
        Secure communication procedures
        """
        
        return {
            'immediate_destruction': {
                'app': 'Signal',
                'settings': {
                    'disappearing_messages': '1 week or less',
                    'screen_security': 'enabled',
                    'incognito_keyboard': 'enabled',
                    'registration_lock': 'enabled',
                },
                'procedure': 'Verify safety number before sensitive comms',
            },
            'long_term_storage': {
                'app': 'ProtonMail or Tutanota',
                'settings': {
                    'auto_delete': 'configure',
                    'no_external_images': 'enabled',
                    'two_factor': 'hardware_key',
                },
            },
            'anonymous_tips': {
                'method': 'SecureDrop or GlobaLeaks',
                'procedure': 'Use Tor Browser, follow instance instructions',
            },
            'group_coordination': {
                'app': 'Session or SimpleX',
                'reason': 'No phone number required, metadata resistant',
            }
        }
        
    def physical_security(self) -> Dict:
        """
        Physical security measures
        """
        
        return {
            'device_storage': {
                'at_rest': 'Full disk encryption (LUKS/BitLocker/FileVault)',
                'key_management': 'Strong passphrase, not written down',
                'backup': 'Encrypted offsite or hidden physical',
            },
            'travel': {
                'border_crossing': 'Wipe sensitive data before crossing',
                'faraday_bags': 'Use when not actively using devices',
                'public_spaces': 'Screen privacy filters, position awareness',
            },
            'home': {
                'location': 'Consider not keeping sensitive devices at home',
                'safe': 'Fireproof, hidden, for encrypted drives',
                'surveillance': 'Be aware of cameras near your location',
            }
        }
        
    def incident_response(self) -> Dict:
        """
        Procedures for security incidents
        """
        
        return {
            'suspicious_activity': {
                'immediate': 'Disconnect from network',
                'assess': 'What was accessed? What identity was exposed?',
                'preserve': 'Document everything for legal defense',
                'notify': 'Legal counsel, potentially',
            },
            'compromise_confirmed': {
                'burn_identity': 'Cease using compromised identity',
                'preserve_evidence': 'Forensic imaging if legal case',
                'new_identity': 'Create new identity with lessons learned',
                'review': 'How was the compromise possible?',
            },
            'legal_contact': {
                'do_not': 'Speak without counsel',
                'do': 'Document everything, invoke rights',
                'prepare': 'Have legal representation arranged',
            }
        }

class ThreatModeling:
    """
    Threat modeling framework
    """
    
    def __init__(self):
        self.threat_actors = {
            'criminal': {
                'capabilities': ['phishing', 'malware', 'social_engineering'],
                'motivation': 'financial',
                'resources': 'low_to_medium',
            },
            'corporate': {
                'capabilities': ['lawsuits', 'private_investigators', 'doxxing'],
                'motivation': 'reputation',
                'resources': 'high',
            },
            'state': {
                'capabilities': ['nsa_level_surveillance', 'legal_compulsion', 'physical'],
                'motivation': 'political',
                'resources': 'unlimited',
            },
            'activist_adversary': {
                'capabilities': ['doxxing', 'harassment', 'swatting'],
                'motivation': 'ideological',
                'resources': 'variable',
            }
        }
        
    def assess_threats(self, your_situation: dict) -> List[dict]:
        """
        Assess which threats are relevant to you
        """
        
        relevant_threats = []
        
        for actor, profile in self.threat_actors.items():
            # Determine if this actor would target you
            if self._would_target(profile, your_situation):
                relevant_threats.append({
                    'actor': actor,
                    'profile': profile,
                    'mitigations': self._recommend_mitigations(profile)
                })
                
        return relevant_threats
        
    def _would_target(self, profile: dict, situation: dict) -> bool:
        """Determine if threat actor would target this situation"""
        # Implementation based on situation analysis
        pass
        
    def _recommend_mitigations(self, profile: dict) -> List[str]:
        """Recommend mitigations for threat profile"""
        mitigations = []
        
        if 'nsa_level_surveillance' in profile['capabilities']:
            mitigations.extend([
                'Air gap for sensitive operations',
                'Tails OS from read-only media',
                'No electronic communication for sensitive topics',
                'Physical meeting security',
            ])
            
        if 'legal_compulsion' in profile['capabilities']:
            mitigations.extend([
                'Jurisdiction shopping for services',
                'No US-based providers',
                'Strong encryption with plausible deniability',
                'Legal preparation',
            ])
            
        return mitigations
Summary
Category	Key Recommendations
Legal Discovery	Assume all data is retained; use E2E encryption; minimize metadata
OSINT Prevention	Compartmentalize identities; vary behavior; no shared attributes
Technical Leaks	Disable WebRTC; fix DNS leaks; randomize fingerprints; use Tor/VPN
Identity Management	One identity per purpose; dedicated hardware; no cross-contamination
Hardware	Purism Librem / Coreboot ThinkPad / GrapheneOS Pixel
Network	pfSense/OPNsense with VPN kill switch; DoH/DoT; VLAN isolation
Procedures	Daily checklists; pre-action verification; incident response plan
Critical OPSEC Principles:

Compartmentalization: Complete separation of identities
Minimization: Reduce attack surface, data retention
Regular Rotation: Change patterns, hardware, identities
Plausible Deniability: Hidden volumes, alternative explanations
Physical Security: Often the weakest link
Legal Preparation: Know your rights, have representation
This framework provides defense in depth against surveillance, correlation, and identification.