# Networking Interview Questions

## Table of Contents
- [TCP/IP Fundamentals](#tcpip-fundamentals)
- [DNS](#dns)
- [Load Balancing](#load-balancing)
- [CDN & Caching](#cdn--caching)
- [Network Security](#network-security)
- [Cloud Networking](#cloud-networking)

---

## TCP/IP Fundamentals

### Q1: Explain the OSI model and TCP/IP model.

**Difficulty:** Junior

**Answer:**

**OSI Model (7 Layers):**
1. **Physical**: Electrical signals, cables
2. **Data Link**: Frames, MAC addresses, switches
3. **Network**: IP addresses, routing, routers
4. **Transport**: TCP/UDP, ports, reliability
5. **Session**: Session management
6. **Presentation**: Data encryption, compression
7. **Application**: HTTP, FTP, SMTP

**TCP/IP Model (4 Layers):**
1. **Link**: Physical + Data Link (Ethernet, WiFi)
2. **Internet**: Network layer (IP, ICMP)
3. **Transport**: TCP, UDP
4. **Application**: Application + Presentation + Session (HTTP, DNS)

**Key Differences:**
- OSI: Theoretical, 7 layers
- TCP/IP: Practical, 4 layers, what internet uses

**Real-world Context:** HTTP request: Application layer → TCP segments (Transport) → IP packets (Internet) → Ethernet frames (Link).

**Follow-up:** What layer does a router operate at? (Network layer - Layer 3, routes IP packets)

---

### Q2: What is the difference between TCP and UDP?

**Difficulty:** Junior

**Answer:**

**TCP (Transmission Control Protocol):**
- **Connection-oriented**: Establishes connection before data transfer
- **Reliable**: Guarantees delivery, retransmits lost packets
- **Ordered**: Packets arrive in order
- **Flow control**: Prevents overwhelming receiver
- **Congestion control**: Adjusts sending rate
- **Overhead**: Higher (headers, acknowledgments)
- **Use cases**: HTTP, HTTPS, FTP, SSH, email

**UDP (User Datagram Protocol):**
- **Connectionless**: No connection establishment
- **Unreliable**: No guarantee of delivery
- **Unordered**: Packets may arrive out of order
- **No flow/congestion control**: Sends at full speed
- **Overhead**: Lower (smaller headers)
- **Use cases**: DNS, DHCP, video streaming, gaming, VoIP

**Key Differences:**
- TCP: Reliable but slower
- UDP: Fast but unreliable

**Real-world Context:** Web browsing uses TCP (need reliable delivery). Video streaming uses UDP (speed more important than perfect delivery).

**Follow-up:** When would you choose UDP over TCP? (Real-time applications where speed > reliability: gaming, live video, DNS queries)

---

### Q3: Explain the TCP three-way handshake.

**Difficulty:** Mid

**Answer:**

Three-way handshake establishes TCP connection before data transfer.

**Process:**
1. **SYN**: Client sends SYN (synchronize) packet with initial sequence number
2. **SYN-ACK**: Server responds with SYN-ACK (acknowledges client's SYN, sends own SYN)
3. **ACK**: Client sends ACK (acknowledges server's SYN)

**State Transitions:**
- Client: CLOSED → SYN_SENT → ESTABLISHED
- Server: CLOSED → LISTEN → SYN_RECEIVED → ESTABLISHED

**Why Three-Way?**
- Ensures both sides can send and receive
- Prevents old duplicate connections
- Synchronizes sequence numbers

**Connection Termination (Four-Way Handshake):**
1. FIN from one side
2. ACK
3. FIN from other side
4. ACK

**Real-world Context:** Browser connects to web server. Three-way handshake establishes connection, then HTTP request sent over TCP connection.

**Follow-up:** What is SYN flood attack? (Attacker sends many SYN packets, doesn't complete handshake, exhausts server resources)

---

### Q4: What are ports and how do they work?

**Difficulty:** Junior

**Answer:**

Ports are 16-bit numbers (0-65535) that identify specific processes/services on a host.

**Port Ranges:**
- **Well-known ports (0-1023)**: Reserved for system services
  - 22: SSH
  - 80: HTTP
  - 443: HTTPS
  - 53: DNS
- **Registered ports (1024-49151)**: Assigned to applications
- **Dynamic/Private ports (49152-65535)**: Ephemeral ports for client connections

**How it Works:**
- IP address identifies host
- Port identifies service on host
- Socket = IP address + Port (e.g., 192.168.1.1:80)

**Example:**
- Web server listens on port 80
- Client connects to server:80
- Server responds from port 80 to client's ephemeral port (e.g., 54321)

**Real-world Context:** Server runs web (80), SSH (22), database (5432). Client connects to specific port to reach specific service.

**Follow-up:** What's the difference between source and destination port? (Destination: service port, Source: client's ephemeral port)

---

### Q5: Explain subnetting and CIDR notation.

**Difficulty:** Mid

**Answer:**

**Subnetting:**
- Dividing network into smaller subnetworks
- Improves organization, security, performance
- Reduces broadcast domains

**CIDR (Classless Inter-Domain Routing):**
- Notation: IP address/prefix length
- Prefix length: Number of network bits
- Example: 192.168.1.0/24
  - 24 bits = network (192.168.1)
  - 8 bits = host (0-255, 254 usable)
  - Subnet mask: 255.255.255.0

**Common CIDR Blocks:**
- /32: Single host (255.255.255.255)
- /24: 254 hosts (255.255.255.0)
- /16: 65,534 hosts (255.255.0.0)
- /8: 16,777,214 hosts (255.0.0.0)

**Subnetting Example:**
- Network: 192.168.1.0/24
- Subnet 1: 192.168.1.0/25 (128 hosts)
- Subnet 2: 192.168.1.128/25 (128 hosts)

**Real-world Context:** VPC with 10.0.0.0/16. Create subnets: 10.0.1.0/24 (public), 10.0.2.0/24 (private). Each subnet has 254 usable IPs.

**Follow-up:** How do you calculate number of hosts in a subnet? (2^(32-prefix) - 2, subtract 2 for network and broadcast addresses)

---

## DNS

### Q6: How does DNS resolution work?

**Difficulty:** Mid

**Answer:**

DNS (Domain Name System) translates domain names to IP addresses.

**Resolution Process:**

1. **Check Local Cache**: Browser/OS cache
2. **Check Hosts File**: Local file override
3. **Query Recursive Resolver**: ISP's DNS server
4. **Query Root Nameservers**: "." (13 root servers)
5. **Query TLD Nameservers**: ".com" nameservers
6. **Query Authoritative Nameservers**: Domain's nameservers
7. **Return IP Address**: Cached and returned

**DNS Record Types:**
- **A**: IPv4 address
- **AAAA**: IPv6 address
- **CNAME**: Canonical name (alias)
- **MX**: Mail exchange
- **TXT**: Text records (SPF, DKIM)
- **NS**: Nameserver

**Example:**
```
Query: www.example.com
1. Recursive resolver → Root: "Where is .com?"
2. Root → ".com nameservers"
3. Recursive → .com: "Where is example.com?"
4. .com → "example.com nameservers"
5. Recursive → example.com: "Where is www.example.com?"
6. example.com → "192.0.2.1"
7. Return to client
```

**Real-world Context:** Browser requests www.google.com. DNS resolves to 142.250.191.14. Browser connects to that IP.

**Follow-up:** What's the difference between recursive and authoritative nameservers? (Recursive: queries on behalf of clients, Authoritative: owns domain records)

---

### Q7: What is DNS caching and TTL?

**Difficulty:** Mid

**Answer:**

**DNS Caching:**
- Store DNS responses to reduce queries
- Cached at multiple levels: browser, OS, resolver, authoritative

**TTL (Time To Live):**
- Time in seconds record can be cached
- Set by authoritative nameserver
- After TTL expires, cache invalidated

**TTL Values:**
- **Low (300-600s)**: Fast changes, load balancing
- **Medium (3600s)**: Normal websites
- **High (86400s)**: Stable services

**Caching Layers:**
1. Browser cache (minutes)
2. OS cache (hours)
3. Recursive resolver cache (respects TTL)
4. Authoritative server (no cache)

**Example:**
```
A record: www.example.com → 192.0.2.1 (TTL: 3600)
- Cached for 1 hour
- After 1 hour, new query made
- If IP changes, takes up to TTL to propagate
```

**Real-world Context:** Change website IP. Set low TTL (300s) before change. After change, old IP cached for max 5 minutes. Then new IP used.

**Follow-up:** What happens if you set TTL too low? (More DNS queries, higher load on nameservers, slower resolution)

---

### Q8: Explain DNS record types: A, AAAA, CNAME, MX, TXT.

**Difficulty:** Mid

**Answer:**

**A Record:**
- Maps domain to IPv4 address
- Example: `example.com → 192.0.2.1`

**AAAA Record:**
- Maps domain to IPv6 address
- Example: `example.com → 2001:db8::1`

**CNAME Record:**
- Creates alias pointing to another domain
- Can't use CNAME on root domain (use A record)
- Example: `www.example.com → example.com`

**MX Record:**
- Mail exchange server for domain
- Has priority (lower = higher priority)
- Example: `example.com MX 10 mail.example.com`

**TXT Record:**
- Text data (SPF, DKIM, verification)
- Example: `example.com TXT "v=spf1 include:_spf.google.com ~all"`

**Common Use Cases:**
- A/AAAA: Website IP addresses
- CNAME: Subdomains, CDN aliases
- MX: Email routing
- TXT: Email authentication, domain verification

**Real-world Context:** Website: A record. Email: MX record. Subdomain: CNAME to main domain. SPF: TXT record.

**Follow-up:** Why can't you use CNAME on root domain? (Root domain needs other records like MX, NS. CNAME conflicts with them)

---

## Load Balancing

### Q9: What is load balancing and what are the different types?

**Difficulty:** Mid

**Answer:**

Load balancing distributes incoming traffic across multiple servers to improve performance and availability.

**Types:**

**1. Layer 4 (Transport Layer) Load Balancing:**
- Based on IP and port
- Fast, simple
- No content inspection
- Examples: AWS Network Load Balancer, HAProxy (TCP mode)

**2. Layer 7 (Application Layer) Load Balancing:**
- Based on HTTP content (URL, headers, cookies)
- More intelligent routing
- SSL termination
- Examples: AWS Application Load Balancer, NGINX, HAProxy (HTTP mode)

**Algorithms:**
- **Round Robin**: Distribute sequentially
- **Least Connections**: Send to server with fewest connections
- **IP Hash**: Consistent hashing based on client IP
- **Weighted**: Assign weights to servers

**Real-world Context:** 3 web servers behind load balancer. L7 LB routes `/api` to API servers, `/static` to static servers. L4 LB just distributes by IP/port.

**Follow-up:** What's the difference between L4 and L7 load balancing? (L4: IP/port based, L7: HTTP content based, more features)

---

### Q10: Explain health checks in load balancing.

**Difficulty:** Mid

**Answer:**

Health checks determine if backend servers are healthy and can receive traffic.

**Types:**

**1. Active Health Checks:**
- Load balancer proactively checks servers
- Sends requests periodically
- Removes unhealthy servers from pool
- Examples: HTTP GET, TCP connect, custom script

**2. Passive Health Checks:**
- Monitor actual requests
- Track response times, errors
- Remove servers based on failure rate

**Health Check Parameters:**
- **Interval**: How often to check (e.g., 30s)
- **Timeout**: Max wait for response (e.g., 5s)
- **Healthy Threshold**: Successes needed to mark healthy (e.g., 2)
- **Unhealthy Threshold**: Failures needed to mark unhealthy (e.g., 3)
- **Path**: Endpoint to check (e.g., `/health`)

**Example:**
```
Health check: GET /health
Interval: 30s
Timeout: 5s
Healthy: 2 consecutive successes
Unhealthy: 3 consecutive failures
```

**Real-world Context:** Load balancer checks `/health` every 30s. If server fails 3 times, removed from pool. When healthy again, added back.

**Follow-up:** What happens if health check endpoint is slow? (May cause false negatives, use separate lightweight endpoint, adjust timeout)

---

### Q11: What is session affinity (sticky sessions) and when to use it?

**Difficulty:** Mid

**Answer:**

Session affinity (sticky sessions) ensures client requests go to same backend server.

**How it Works:**
- Load balancer uses cookie or source IP
- First request → assign server → set cookie
- Subsequent requests → route to same server

**Use Cases:**
- Applications with server-side sessions
- Stateful applications
- File uploads (large files)

**Implementation:**
- **Cookie-based**: Set cookie with server identifier
- **IP-based**: Hash client IP (less reliable with NAT)

**Drawbacks:**
- Uneven load distribution
- Server failure loses sessions
- Doesn't work well with mobile clients (IP changes)

**Alternatives:**
- Use shared session store (Redis, database)
- Stateless applications (JWT tokens)
- Client-side sessions

**Real-world Context:** E-commerce site with shopping cart stored in server memory. Use sticky sessions so cart persists. Better: Store cart in Redis.

**Follow-up:** What are the problems with sticky sessions? (Uneven load, lost sessions on server failure, doesn't scale well)

---

## CDN & Caching

### Q12: What is a CDN and how does it work?

**Difficulty:** Mid

**Answer:**

CDN (Content Delivery Network) is a distributed network of servers that cache content closer to users.

**How it Works:**
1. User requests content (e.g., image)
2. Request routed to nearest CDN edge server
3. If cached (cache hit), serve immediately
4. If not cached (cache miss), fetch from origin, cache, serve
5. Subsequent requests served from cache

**Benefits:**
- **Lower Latency**: Content closer to users
- **Reduced Bandwidth**: Offloads origin server
- **Better Performance**: Faster page loads
- **DDoS Protection**: Distributes traffic
- **Global Scale**: Serves users worldwide

**CDN Providers:**
- CloudFront (AWS)
- Cloud CDN (GCP)
- Azure CDN
- Cloudflare
- Fastly

**Use Cases:**
- Static assets (images, CSS, JS)
- Video streaming
- Software downloads
- API acceleration

**Real-world Context:** Website serves images from S3. Use CloudFront CDN. Images cached at edge locations worldwide. Users get faster loads.

**Follow-up:** What content should you put on CDN? (Static assets, public content. Not: dynamic content, private content, frequently changing data)

---

### Q13: Explain cache invalidation strategies.

**Difficulty:** Mid

**Answer:**

Cache invalidation removes or updates cached content when it changes.

**Strategies:**

**1. TTL (Time To Live):**
- Cache expires after time period
- Simple, but may serve stale content
- Example: Cache for 1 hour

**2. Cache Invalidation:**
- Manually purge cache when content changes
- Immediate update, but requires action
- Example: Purge cache after deployment

**3. Versioning:**
- Include version in URL (query string, path)
- New version = new URL = cache miss
- Example: `style.css?v=2.0`

**4. ETags:**
- Server sends ETag (content hash)
- Client sends If-None-Match header
- If unchanged, server returns 304 Not Modified

**5. Cache-Control Headers:**
- `max-age`: How long to cache
- `no-cache`: Revalidate before serving
- `no-store`: Don't cache
- `must-revalidate`: Revalidate when expired

**Real-world Context:** Update website CSS. Use versioning: `style.css?v=2`. CDN caches new version. Old version expires naturally.

**Follow-up:** What's the difference between TTL and manual invalidation? (TTL: automatic but may be stale, Manual: immediate but requires action)

---

## Network Security

### Q14: What is a firewall and how does it work?

**Difficulty:** Mid

**Answer:**

Firewall is network security device that filters traffic based on rules.

**Types:**

**1. Packet Filtering Firewall:**
- Inspects packets (source/dest IP, port, protocol)
- Fast, simple
- Stateless (no connection tracking)

**2. Stateful Firewall:**
- Tracks connections (stateful inspection)
- Allows return traffic automatically
- More secure, understands context

**3. Application Firewall (WAF):**
- Inspects application layer (HTTP)
- Protects against application attacks
- Examples: AWS WAF, Cloudflare WAF

**Firewall Rules:**
- **Allow**: Permit traffic
- **Deny**: Block traffic
- **Default policy**: What to do if no rule matches (deny all or allow all)

**Example Rules:**
```
Allow: TCP port 80 (HTTP) from any
Allow: TCP port 443 (HTTPS) from any
Allow: TCP port 22 (SSH) from 10.0.0.0/8
Deny: All other traffic
```

**Real-world Context:** Web server firewall: Allow 80/443 from internet, Allow 22 from office IP, Deny everything else.

**Follow-up:** What's the difference between stateful and stateless firewall? (Stateful: tracks connections, allows return traffic. Stateless: inspects each packet independently)

---

### Q15: Explain VPN and how it works.

**Difficulty:** Mid

**Answer:**

VPN (Virtual Private Network) creates encrypted tunnel over public network.

**How it Works:**
1. Client connects to VPN server
2. Encrypted tunnel established
3. All traffic routed through tunnel
4. VPN server decrypts and forwards to destination
5. Response encrypted and sent back

**Types:**

**1. Site-to-Site VPN:**
- Connects two networks
- Example: Office network to AWS VPC

**2. Remote Access VPN:**
- Connects individual users
- Example: Employee working from home

**3. SSL/TLS VPN:**
- Uses HTTPS for VPN
- Works through firewalls (port 443)

**4. IPsec VPN:**
- Network layer encryption
- More complex setup

**Benefits:**
- **Security**: Encrypted traffic
- **Privacy**: Hide IP address
- **Access**: Access private networks remotely
- **Bypass**: Bypass geo-restrictions (not recommended for work)

**Real-world Context:** Employee connects to company VPN. All traffic encrypted. Can access internal servers as if on office network.

**Follow-up:** What's the difference between VPN and proxy? (VPN: encrypts all traffic, routes all traffic. Proxy: only specific traffic, may not encrypt)

---

### Q16: What is DDoS and how do you mitigate it?

**Difficulty:** Senior

**Answer:**

DDoS (Distributed Denial of Service) overwhelms target with traffic from multiple sources.

**Types:**

**1. Volume-Based:**
- Overwhelm with high traffic volume
- Examples: UDP flood, ICMP flood
- Mitigation: Rate limiting, filtering

**2. Protocol-Based:**
- Exploit protocol weaknesses
- Examples: SYN flood, ping of death
- Mitigation: Firewall rules, protocol validation

**3. Application-Based:**
- Target application layer
- Examples: HTTP flood, slowloris
- Mitigation: WAF, rate limiting, CAPTCHA

**Mitigation Strategies:**

**1. CDN/DDoS Protection:**
- Use Cloudflare, AWS Shield, etc.
- Filters traffic before reaching origin
- Distributes attack across edge locations

**2. Rate Limiting:**
- Limit requests per IP
- Block excessive traffic
- Use at multiple layers

**3. Firewall Rules:**
- Block known malicious IPs
- Geo-blocking if not needed globally
- Filter by protocol/port

**4. Scaling:**
- Auto-scaling to handle traffic
- But expensive if attack is large

**5. Monitoring:**
- Detect attacks early
- Alert on unusual traffic patterns
- Analyze attack patterns

**Real-world Context:** Website under DDoS attack. Use Cloudflare to filter traffic. Rate limit per IP. Scale infrastructure. Block malicious IPs.

**Follow-up:** What's the difference between DDoS and DoS? (DoS: single source, DDoS: multiple sources, harder to block)

---

## Cloud Networking

### Q17: Explain VPC, subnets, and routing in cloud.

**Difficulty:** Mid

**Answer:**

**VPC (Virtual Private Cloud):**
- Isolated network in cloud
- Define IP ranges (CIDR blocks)
- Complete control over networking

**Subnets:**
- Subdivision of VPC IP range
- **Public Subnet**: Route to Internet Gateway (IGW)
- **Private Subnet**: Route to NAT Gateway (no direct internet)

**Routing:**
- Route tables define traffic flow
- Each subnet has route table
- Routes: destination → target (IGW, NAT, VPC peering)

**Example Architecture:**
```
VPC: 10.0.0.0/16
  ├── Public Subnet: 10.0.1.0/24
  │   └── Route: 0.0.0.0/0 → IGW
  └── Private Subnet: 10.0.2.0/24
      └── Route: 0.0.0.0/0 → NAT Gateway
```

**Security:**
- Security Groups: Instance-level firewall
- NACLs: Subnet-level firewall
- Private subnets: No direct internet access

**Real-world Context:** AWS VPC with public subnets for load balancers, private subnets for application servers and databases.

**Follow-up:** What's the difference between public and private subnet? (Public: route to IGW, Private: route to NAT, no direct internet)

---

### Q18: What is VPC peering and when would you use it?

**Difficulty:** Mid

**Answer:**

VPC peering connects two VPCs using private IP addresses.

**Characteristics:**
- One-to-one connection (not transitive)
- Can peer VPCs in same or different accounts/regions
- CIDR blocks must not overlap
- No single point of failure
- No bandwidth bottleneck

**Use Cases:**
- Connect VPCs in same region
- Share resources between accounts
- Hub-and-spoke architecture

**Setup:**
1. Request peering connection
2. Accept peering connection
3. Update route tables in both VPCs
4. Update security groups

**Limitations:**
- Not transitive (A→B, B→C, but A cannot reach C)
- CIDR blocks cannot overlap
- Need multiple peerings for many VPCs

**Alternative:**
- Transit Gateway: Central hub, transitive routing

**Real-world Context:** Development VPC needs access to shared services VPC (databases, monitoring). Peer VPCs, update routes.

**Follow-up:** How do you connect 3 VPCs? (Need 3 peerings: A-B, A-C, B-C. Or use Transit Gateway with one attachment per VPC)

---

### Q19: Explain load balancing in cloud (ALB, NLB, GLB).

**Difficulty:** Mid

**Answer:**

**AWS Application Load Balancer (ALB):**
- Layer 7 (HTTP/HTTPS)
- Content-based routing (path, host, headers)
- SSL termination
- WebSocket support
- Use for: Web applications, microservices

**AWS Network Load Balancer (NLB):**
- Layer 4 (TCP/UDP)
- Ultra-low latency
- Static IP addresses
- Use for: High-performance applications, gaming

**AWS Gateway Load Balancer (GLB):**
- Transparent network gateway
- Integrates with third-party security appliances
- Use for: Security inspection, firewalls

**GCP Load Balancer:**
- Global or regional
- HTTP(S), TCP/UDP, SSL proxy
- Global: Anycast IP, routes to nearest region

**Azure Load Balancer:**
- Basic or Standard tier
- Public or internal
- Layer 4 load balancing

**Real-world Context:** Web app → ALB (path-based routing, SSL termination). High-performance API → NLB (low latency). Security inspection → GLB.

**Follow-up:** When would you use ALB vs NLB? (ALB: HTTP routing, SSL termination. NLB: TCP/UDP, low latency, static IPs)

---

### Q20: What is service mesh and how does it relate to networking?

**Difficulty:** Senior

**Answer:**

Service mesh is infrastructure layer for microservices communication, handling service-to-service communication.

**Components:**
- **Data Plane**: Sidecar proxies (Envoy) handle traffic
- **Control Plane**: Manages proxies, policies, configuration

**Features:**
- **Service Discovery**: Find services automatically
- **Load Balancing**: Distribute traffic
- **Security**: mTLS encryption, authentication
- **Observability**: Metrics, tracing, logging
- **Traffic Management**: Canary, A/B testing, circuit breakers

**Service Mesh Solutions:**
- **Istio**: Most popular, complex
- **Linkerd**: Simpler, lighter
- **Consul Connect**: HashiCorp
- **AWS App Mesh**: AWS-native

**How it Works:**
- Sidecar proxy injected into each pod
- All traffic goes through proxy
- Proxy handles: routing, security, observability
- Control plane configures proxies

**Real-world Context:** Microservices architecture. Service mesh handles: service discovery, load balancing, mTLS, metrics, without changing application code.

**Follow-up:** What's the difference between service mesh and API gateway? (Service mesh: service-to-service, API gateway: external-to-service, north-south traffic)

---

## Summary

Networking is fundamental to DevOps. Understand TCP/IP, DNS, load balancing, security, and cloud networking. Practice troubleshooting network issues and designing network architectures.

**Next Steps:**
- Practice subnetting and CIDR calculations
- Set up load balancers in cloud
- Learn DNS management
- Study network security best practices
