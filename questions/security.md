# Security Interview Questions

## Table of Contents
- [Security Fundamentals](#security-fundamentals)
- [Encryption](#encryption)
- [Authentication & Authorization](#authentication--authorization)
- [Network Security](#network-security)
- [Cloud Security](#cloud-security)
- [Compliance & Best Practices](#compliance--best-practices)

---

## Security Fundamentals

### Q1: Explain the CIA triad and its importance.

**Difficulty:** Junior

**Answer:**

CIA Triad is the foundation of information security:

**Confidentiality:**
- Data accessible only to authorized users
- Protection from unauthorized access
- Methods: Encryption, access controls, data classification

**Integrity:**
- Data is accurate and unmodified
- Protection from unauthorized changes
- Methods: Hashing, digital signatures, checksums

**Availability:**
- Data and systems accessible when needed
- Protection from downtime
- Methods: Redundancy, backups, DDoS protection

**Real-world Context:** 
- Confidentiality: Encrypt database, restrict access
- Integrity: Verify file hasn't changed (checksums)
- Availability: Redundant servers, backups

**Follow-up:** How do you balance CIA? (Trade-offs: More encryption = confidentiality but may impact availability. More redundancy = availability but cost)

---

### Q2: What is defense in depth and how do you implement it?

**Difficulty:** Mid

**Answer:**

Defense in depth uses multiple layers of security controls.

**Layers:**

**1. Physical Security:**
- Data center access controls
- Server room security
- Device encryption

**2. Network Security:**
- Firewalls
- Network segmentation
- Intrusion detection/prevention
- VPN

**3. Host Security:**
- OS hardening
- Antivirus/antimalware
- Host-based firewalls
- Patch management

**4. Application Security:**
- Secure coding practices
- Input validation
- Authentication/authorization
- WAF

**5. Data Security:**
- Encryption at rest and in transit
- Data classification
- Access controls
- Backup encryption

**6. Policies and Procedures:**
- Security policies
- Incident response
- Training
- Audits

**Real-world Context:** Web application: Firewall → WAF → Load balancer → Application servers (hardened) → Database (encrypted) → Backups (encrypted).

**Follow-up:** What's the difference between defense in depth and single point of failure? (Defense in depth: multiple layers, Single point: one failure breaks everything)

---

### Q3: Explain the principle of least privilege.

**Difficulty:** Mid

**Answer:**

Principle of least privilege: Grant minimum permissions necessary to perform tasks.

**Implementation:**

**1. User Accounts:**
- Regular users, not admin
- Separate accounts for different roles
- No shared accounts

**2. IAM Policies:**
- Specific permissions, not wildcards
- Scope to specific resources
- Regular access reviews

**3. Service Accounts:**
- Dedicated accounts for services
- Minimum required permissions
- Rotate credentials

**4. Network Access:**
- Restrict to necessary ports/protocols
- Use network segmentation
- VPN for remote access

**5. Application Permissions:**
- Run with minimal privileges
- Separate service accounts
- Limit file system access

**Benefits:**
- Reduces attack surface
- Limits impact of compromised account
- Easier to audit
- Compliance requirements

**Real-world Context:** Application needs S3 read access. Grant `s3:GetObject` on specific bucket, not `s3:*` on all buckets.

**Follow-up:** How do you implement least privilege in cloud? (IAM roles with specific permissions, resource-based policies, regular access reviews)

---

## Encryption

### Q4: Explain symmetric vs asymmetric encryption.

**Difficulty:** Mid

**Answer:**

**Symmetric Encryption:**
- Same key for encryption and decryption
- Fast, efficient
- Key distribution problem
- Examples: AES, DES, 3DES
- Use for: Bulk data encryption

**Asymmetric Encryption:**
- Public key encrypts, private key decrypts
- Slower, more complex
- Solves key distribution
- Examples: RSA, ECC, Elliptic Curve
- Use for: Key exchange, digital signatures

**Hybrid Approach:**
- Use asymmetric to exchange symmetric key
- Use symmetric for data encryption
- Best of both worlds

**Example:**
- TLS: Asymmetric (RSA/ECC) for key exchange, Symmetric (AES) for data

**Real-world Context:** HTTPS: Server sends public key, client encrypts symmetric key with it, both use symmetric key for data. Fast and secure.

**Follow-up:** Why not use asymmetric for everything? (Too slow for large data, symmetric is 1000x faster)

---

### Q5: Explain SSL/TLS and how it works.

**Difficulty:** Mid

**Answer:**

SSL/TLS provides encrypted communication over network.

**TLS Handshake:**

1. **Client Hello**: Client sends supported cipher suites, TLS version
2. **Server Hello**: Server chooses cipher suite, sends certificate
3. **Certificate Verification**: Client verifies server certificate
4. **Key Exchange**: Client encrypts pre-master secret with server's public key
5. **Cipher Change**: Both switch to symmetric encryption
6. **Encrypted Communication**: Data encrypted with symmetric key

**Certificate Components:**
- Public key
- Domain name
- Issuer (CA)
- Validity period
- Digital signature

**Certificate Authorities (CA):**
- Trusted third parties
- Verify domain ownership
- Sign certificates
- Root CAs in browser/OS trust store

**Types:**
- **DV (Domain Validated)**: Basic, verifies domain
- **OV (Organization Validated)**: Verifies organization
- **EV (Extended Validated)**: Highest validation

**Real-world Context:** Website uses TLS certificate. Browser verifies certificate, establishes encrypted connection. Data encrypted in transit.

**Follow-up:** What's the difference between SSL and TLS? (SSL deprecated, TLS is modern version. People say SSL but mean TLS)

---

### Q6: Explain encryption at rest vs in transit.

**Difficulty:** Mid

**Answer:**

**Encryption in Transit:**
- Data encrypted while being transmitted
- Protects data on network
- Methods: TLS/SSL, VPN, IPSec
- Examples: HTTPS, SSH, encrypted database connections

**Encryption at Rest:**
- Data encrypted when stored
- Protects data on disk/database
- Methods: Full disk encryption, database encryption, file-level encryption
- Examples: Encrypted EBS volumes, encrypted S3, database encryption

**Both Needed:**
- In transit: Protects from network interception
- At rest: Protects from physical access, data breaches

**Implementation:**

**At Rest:**
- AWS: EBS encryption, S3 server-side encryption, RDS encryption
- Database: Transparent Data Encryption (TDE)
- File system: LUKS, BitLocker

**In Transit:**
- TLS for web traffic
- SSH for remote access
- Encrypted database connections

**Key Management:**
- Use key management services (AWS KMS, HashiCorp Vault)
- Rotate keys regularly
- Separate keys for different data
- Secure key storage

**Real-world Context:** Database: Encrypt connections (in transit) and encrypt data on disk (at rest). Both required for complete protection.

**Follow-up:** What happens if you only encrypt in transit? (Data vulnerable if attacker gains disk access or database backup is stolen)

---

## Authentication & Authorization

### Q7: Explain OAuth 2.0 and how it works.

**Difficulty:** Senior

**Answer:**

OAuth 2.0 is authorization framework for delegated access.

**Roles:**
- **Resource Owner**: User who owns data
- **Client**: Application requesting access
- **Authorization Server**: Issues tokens
- **Resource Server**: Hosts protected resources

**Flow (Authorization Code):**

1. User clicks "Login with Google"
2. Redirected to authorization server
3. User authenticates and grants permission
4. Authorization server redirects back with code
5. Client exchanges code for access token
6. Client uses access token to access resources

**Grant Types:**
- **Authorization Code**: Web apps (most secure)
- **Implicit**: Public clients (deprecated)
- **Client Credentials**: Server-to-server
- **Refresh Token**: Get new access tokens

**Tokens:**
- **Access Token**: Short-lived, access resources
- **Refresh Token**: Long-lived, get new access tokens

**Real-world Context:** Mobile app wants user's Google photos. User grants permission via OAuth. App gets token, accesses photos on user's behalf.

**Follow-up:** What's the difference between OAuth and OIDC? (OAuth: authorization, OIDC: authentication + identity information)

---

### Q8: Explain JWT (JSON Web Tokens) and their use cases.

**Difficulty:** Mid

**Answer:**

JWT is compact, URL-safe token format for securely transmitting information.

**Structure:**
- **Header**: Algorithm, token type
- **Payload**: Claims (data)
- **Signature**: Verifies token integrity

**Format:**
```
header.payload.signature
```

**Characteristics:**
- Stateless (no server-side storage)
- Self-contained (includes claims)
- Signed (can verify integrity)
- Can be encrypted (JWE)

**Use Cases:**
- Authentication: After login, server issues JWT
- Authorization: Include permissions in JWT
- Information exchange: Securely transmit data

**Example Flow:**
1. User logs in
2. Server validates credentials
3. Server creates JWT with user info
4. Client stores JWT (localStorage, cookie)
5. Client sends JWT in Authorization header
6. Server validates signature and extracts claims

**Security Considerations:**
- Sign tokens (HMAC or RSA)
- Use HTTPS
- Set expiration (short-lived)
- Don't store sensitive data
- Validate signature

**Real-world Context:** API authentication: User logs in, gets JWT. Subsequent requests include JWT. Server validates and processes request.

**Follow-up:** What's the difference between JWT and session cookies? (JWT: stateless, scalable. Sessions: stateful, require server storage)

---

### Q9: Explain MFA (Multi-Factor Authentication) and its importance.

**Difficulty:** Mid

**Answer:**

MFA requires multiple authentication factors.

**Factors:**
1. **Something you know**: Password, PIN
2. **Something you have**: Phone, hardware token, smart card
3. **Something you are**: Biometric (fingerprint, face)

**Types:**
- **2FA**: Two factors (password + code)
- **MFA**: Multiple factors (password + code + biometric)

**Methods:**
- **SMS**: Code sent to phone (less secure)
- **TOTP**: Time-based one-time password (Google Authenticator, Authy)
- **Hardware tokens**: Physical device (YubiKey)
- **Push notifications**: Approve on device
- **Biometric**: Fingerprint, face recognition

**Importance:**
- Passwords can be stolen/phished
- MFA adds extra layer
- Even if password compromised, attacker needs second factor
- Reduces account takeover

**Best Practices:**
- Enable MFA for all privileged accounts
- Use TOTP or hardware tokens (more secure than SMS)
- Require MFA for sensitive operations
- Have backup codes

**Real-world Context:** AWS account: Password + MFA code from authenticator app. Even if password stolen, attacker can't access without phone.

**Follow-up:** Why is SMS less secure than TOTP? (SMS can be intercepted, SIM swapping attacks. TOTP is local, can't be intercepted)

---

## Network Security

### Q10: Explain firewalls, WAF, and their differences.

**Difficulty:** Mid

**Answer:**

**Firewall (Network Firewall):**
- Layer 3/4 (IP, TCP/UDP)
- Filters based on IP, port, protocol
- Stateful or stateless
- Examples: iptables, AWS Security Groups, network firewalls

**WAF (Web Application Firewall):**
- Layer 7 (HTTP/HTTPS)
- Protects web applications
- Inspects HTTP requests/responses
- Blocks: SQL injection, XSS, CSRF, DDoS
- Examples: AWS WAF, Cloudflare WAF, ModSecurity

**Differences:**

| Feature | Firewall | WAF |
|---------|----------|-----|
| Layer | 3/4 | 7 |
| Inspection | IP, port | HTTP content |
| Protection | Network attacks | Application attacks |
| Rules | IP/port based | URL, headers, body |

**Use Both:**
- Firewall: Network-level protection
- WAF: Application-level protection

**Real-world Context:** Web application: Network firewall allows port 443, WAF inspects HTTP requests, blocks SQL injection attempts.

**Follow-up:** Can a WAF replace a firewall? (No, different layers. WAF protects applications, firewall protects network)

---

### Q11: Explain DDoS attacks and mitigation strategies.

**Difficulty:** Senior

**Answer:**

DDoS (Distributed Denial of Service) overwhelms target with traffic.

**Types:**

**1. Volume-Based:**
- High traffic volume
- Examples: UDP flood, ICMP flood
- Mitigation: Rate limiting, filtering, CDN

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
- Cloudflare, AWS Shield, Akamai
- Filters traffic before origin
- Distributes attack

**2. Rate Limiting:**
- Limit requests per IP
- Block excessive traffic
- Multiple layers

**3. Scaling:**
- Auto-scaling
- But expensive if attack large

**4. Monitoring:**
- Detect early
- Alert on unusual patterns
- Analyze attack patterns

**5. Blacklisting:**
- Block known malicious IPs
- Geo-blocking if not needed globally

**Real-world Context:** Website under DDoS. Use Cloudflare to filter, rate limit per IP, scale infrastructure, monitor and block malicious IPs.

**Follow-up:** What's the difference between DDoS and DoS? (DoS: single source, DDoS: multiple sources, harder to block)

---

## Cloud Security

### Q12: Explain IAM best practices in cloud.

**Difficulty:** Mid

**Answer:**

**1. Principle of Least Privilege:**
- Grant minimum permissions
- Regular access reviews
- Remove unused permissions

**2. Use Roles, Not Users:**
- Roles for services (EC2, Lambda)
- Users only for humans
- Temporary credentials

**3. Enable MFA:**
- Require MFA for privileged operations
- Root account MFA
- Console access MFA

**4. Rotate Credentials:**
- Regular rotation
- Use credential rotation tools
- Monitor for old credentials

**5. Separate Accounts:**
- Development, staging, production
- Isolate environments
- Cross-account roles

**6. Audit and Monitor:**
- Enable CloudTrail (AWS)
- Monitor access logs
- Alert on suspicious activity

**7. Use Policy Conditions:**
- IP restrictions
- Time-based access
- Source restrictions

**8. Avoid Hardcoded Credentials:**
- Use IAM roles
- Use secrets management
- Never commit secrets

**Real-world Context:** EC2 instance needs S3 access. Use IAM role attached to instance, not access keys. Regular access reviews, MFA for console.

**Follow-up:** How do you audit IAM permissions? (Use IAM Access Analyzer, CloudTrail, policy simulator, regular reviews)

---

### Q13: Explain secrets management best practices.

**Difficulty:** Mid

**Answer:**

**Secrets Management:**
- Centralized storage of secrets
- Encryption at rest and in transit
- Access control and auditing
- Rotation capabilities

**Secrets Include:**
- Passwords
- API keys
- Database credentials
- TLS certificates
- SSH keys

**Best Practices:**

**1. Use Secrets Management Service:**
- AWS Secrets Manager
- HashiCorp Vault
- Azure Key Vault
- GCP Secret Manager

**2. Never Commit Secrets:**
- Use .gitignore
- Scan repositories
- Use environment variables or secrets service

**3. Rotate Regularly:**
- Automatic rotation
- Set expiration dates
- Monitor for old secrets

**4. Least Privilege Access:**
- Restrict who can access secrets
- Audit access
- Use IAM policies

**5. Encrypt at Rest:**
- Use encryption keys
- Key management service
- Separate keys for different secrets

**6. Audit and Monitor:**
- Log all access
- Alert on suspicious access
- Regular audits

**Example:**
```bash
# Bad: Hardcoded
DB_PASSWORD="secret123"

# Good: Secrets Manager
aws secretsmanager get-secret-value --secret-id db-password
```

**Real-world Context:** Application needs database password. Store in AWS Secrets Manager. Application retrieves at runtime. Rotate every 90 days.

**Follow-up:** What's the difference between Secrets Manager and Parameter Store? (Secrets Manager: automatic rotation, higher cost. Parameter Store: simpler, lower cost, manual rotation)

---

### Q14: Explain security groups and network ACLs in cloud.

**Difficulty:** Mid

**Answer:**

**Security Groups (AWS):**
- Stateful firewall at instance level
- Rules: allow only (default deny)
- Return traffic automatically allowed
- Can reference other security groups
- Applied to instances

**Network ACLs (AWS):**
- Stateless firewall at subnet level
- Rules: allow and deny
- Must allow both inbound and outbound
- Evaluated in order (first match wins)
- Applied to subnets

**Differences:**

| Feature | Security Groups | NACLs |
|---------|----------------|-------|
| Level | Instance | Subnet |
| Stateful | Yes | No |
| Rules | Allow only | Allow/Deny |
| Default | Deny all | Allow all (can change) |

**Use Cases:**
- Security Groups: Primary defense, instance-level
- NACLs: Extra layer, subnet-level, compliance

**Best Practice:**
- Use Security Groups as primary defense
- Use NACLs for additional layer or compliance

**Real-world Context:** Web server: Security Group allows 80/443 from internet, 22 from office IP. NACL adds subnet-level protection, blocks specific IPs.

**Follow-up:** What happens if you block port 80 in NACL but allow in Security Group? (Traffic blocked - NACL evaluated first)

---

## Compliance & Best Practices

### Q15: Explain security compliance: SOC 2, PCI DSS, GDPR.

**Difficulty:** Senior

**Answer:**

**SOC 2 (Service Organization Control 2):**
- Framework for security controls
- Trust Services Criteria: Security, Availability, Processing Integrity, Confidentiality, Privacy
- Type I: Point in time
- Type II: Over time period (6-12 months)
- Common for SaaS companies

**PCI DSS (Payment Card Industry Data Security Standard):**
- For organizations handling credit card data
- 12 requirements: Firewalls, encryption, access control, etc.
- Levels based on transaction volume
- Annual assessment required

**GDPR (General Data Protection Regulation):**
- EU data protection law
- Protects personal data of EU residents
- Requirements: Consent, right to access, right to deletion, data breach notification
- Applies globally if processing EU data

**Key Requirements:**

**SOC 2:**
- Access controls
- Encryption
- Monitoring
- Incident response

**PCI DSS:**
- Encrypt card data
- Restrict access
- Monitor networks
- Regular testing

**GDPR:**
- Lawful basis for processing
- Data minimization
- Right to erasure
- Breach notification (72 hours)

**Real-world Context:** E-commerce: PCI DSS for payment processing, SOC 2 for overall security, GDPR for EU customers' data protection.

**Follow-up:** How do you prepare for compliance audits? (Document controls, implement security measures, regular assessments, maintain evidence)

---

### Q16: Explain security incident response process.

**Difficulty:** Senior

**Answer:**

**Incident Response Phases:**

**1. Preparation:**
- Incident response plan
- Team roles and responsibilities
- Tools and access
- Communication plan
- Regular training

**2. Identification:**
- Detect incident
- Monitor logs, alerts
- User reports
- Security tools
- Classify severity

**3. Containment:**
- **Short-term**: Immediate actions to stop spread
  - Isolate affected systems
  - Block malicious IPs
  - Disable compromised accounts
- **Long-term**: Remove threat completely
  - Patch vulnerabilities
  - Remove malware
  - Change credentials

**4. Eradication:**
- Remove threat completely
- Patch vulnerabilities
- Remove backdoors
- Clean infected systems

**5. Recovery:**
- Restore systems
- Verify functionality
- Monitor for recurrence
- Gradual restoration

**6. Lessons Learned:**
- Post-incident review
- Document what happened
- Identify improvements
- Update procedures

**Real-world Context:** Data breach detected: Contain (isolate systems, block IPs), Eradicate (remove malware, patch), Recover (restore, monitor), Learn (review, improve).

**Follow-up:** What's the difference between incident and event? (Event: something happened, Incident: security impact, requires response)

---

### Q17: Explain security scanning and vulnerability management.

**Difficulty:** Mid

**Answer:**

**Vulnerability Management Process:**

**1. Discovery:**
- Asset inventory
- Network scanning
- Application scanning

**2. Assessment:**
- Vulnerability scanning
- Penetration testing
- Code review

**3. Prioritization:**
- Risk assessment (CVSS scores)
- Business impact
- Exploitability
- Patch availability

**4. Remediation:**
- Patch management
- Configuration changes
- Compensating controls

**5. Verification:**
- Re-scan
- Verify fixes
- Test functionality

**Scanning Types:**

**1. Network Scanning:**
- Port scanning
- Service identification
- Vulnerability detection
- Tools: Nmap, Nessus, OpenVAS

**2. Application Scanning:**
- SAST: Static code analysis
- DAST: Dynamic testing
- IAST: Interactive testing
- Tools: SonarQube, OWASP ZAP, Burp Suite

**3. Container Scanning:**
- Image vulnerabilities
- Base image issues
- Dependency vulnerabilities
- Tools: Trivy, Clair, Snyk

**4. Infrastructure Scanning:**
- IaC misconfigurations
- Cloud security issues
- Tools: Checkov, Terrascan, Scout Suite

**Real-world Context:** Regular scans: Network (monthly), Application (in CI/CD), Containers (on build), Infrastructure (on changes). Prioritize and patch.

**Follow-up:** How do you prioritize vulnerabilities? (CVSS score, exploitability, business impact, patch availability, compensating controls)

---

### Q18: Explain zero trust security model.

**Difficulty:** Senior

**Answer:**

Zero Trust: Never trust, always verify. Assume breach, verify explicitly.

**Principles:**

**1. Verify Explicitly:**
- Authenticate and authorize all access
- Use least privilege
- Verify device and user

**2. Use Least Privilege:**
- Just-in-time access
- Just-enough-access
- Risk-based policies

**3. Assume Breach:**
- Segment access
- Encrypt end-to-end
- Monitor and log
- Use analytics

**Components:**

**1. Identity:**
- Strong authentication (MFA)
- Device health checks
- Continuous verification

**2. Devices:**
- Device compliance
- Patch management
- Encryption

**3. Networks:**
- Micro-segmentation
- Encrypted connections
- No implicit trust

**4. Applications:**
- Application-level controls
- API security
- Access policies

**5. Data:**
- Classification
- Encryption
- Access controls

**Implementation:**
- Identity provider (Azure AD, Okta)
- Network segmentation
- Endpoint security
- Monitoring and analytics

**Real-world Context:** Traditional: Trust internal network. Zero Trust: Verify every access, even internal. Segment networks, encrypt everything, monitor continuously.

**Follow-up:** How is zero trust different from traditional security? (Traditional: Trust internal, protect perimeter. Zero Trust: No implicit trust, verify everything)

---

## Summary

Security is critical in DevOps. Understand encryption, authentication, network security, cloud security, and compliance. Implement defense in depth and follow security best practices.

**Next Steps:**
- Study security frameworks (OWASP, NIST)
- Practice security scanning and hardening
- Learn about compliance requirements
- Get security certifications (Security+, CISSP, AWS Security)
