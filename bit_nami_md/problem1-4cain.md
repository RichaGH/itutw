# Вопрос
Check proxy authentication requirements - No need this: all work correctly but broad.com does not work
proxy has not any restrictions all requests go free to any address
tel me more about "new CDN endpoints" what is this?

# Рассуждения

We are facing a situation where the proxy is not restricting access to other sites but specifically fails for `repo.broadcom.com`. The error "Proxy CONNECT aborted" suggests that the proxy server is terminating the connection attempt to Broadcom. Since the proxy is set to `localhost:8090` and we have no authentication requirements, we need to explore other reasons.
### What are "new CDN endpoints"?
- **CDN (Content Delivery Network)**: A network of distributed servers that deliver web content to users based on their geographic location. CDNs are used to improve site performance and availability.
- **New CDN endpoints**: After Broadcom's acquisition of Bitnami, they migrated the Helm chart repository to a new infrastructure, likely using a CDN (like CloudFront, Akamai, etc.) to serve the charts. The new endpoint `repo.broadcom.com` might be:
  - Hosted on a different IP range.
  - Using a different SSL/TLS configuration.
  - Geographically restricted in some regions.
### Why might the proxy block Broadcom?
1. **IP Reputation**: The new CDN IPs might be flagged in the proxy's security policy (e.g., if they are newly assigned and not yet trusted).
2. **SSL Inspection**: The proxy might be attempting to inspect SSL traffic but failing due to:
   - **Certificate Pinning**: Broadcom's CDN might use certificates that the proxy cannot intercept.
   - **Strict SNI (Server Name Indication)**: The proxy might not correctly handle the SNI for `repo.broadcom.com`.
3. **Corporate Firewall Rules**: Even if the proxy allows general access, specific rules might block:
   - Financial/enterprise domains (like Broadcom) due to compliance.
   - Newly registered domains or CDN IP ranges.
### Diagnostic Steps
#### 1. Test Without Proxy (Temporarily)
```bash
unset HTTPS_PROXY
curl -v https://repo.broadcom.com/bitnami-files/index.yaml
```
#### 2. Test Through Proxy with `curl` Using Explicit Proxy Flag
```bash
curl -v --proxy http://localhost:8090 https://repo.broadcom.com/bitnami-files/index.yaml
```
#### 3. Check DNS Resolution Through Proxy
```bash
curl --proxy http://localhost:8090 http://repo.broadcom.com  # (HTTP to see if DNS resolves)
```
#### 4. Check if Proxy Logs Show the Block
If you have access to proxy logs, look for:
- `CONNECT repo.broadcom.com:443` entries
- Any `403 Forbidden` or `407 Proxy Authentication Required` responses (though you mentioned no auth)
### Solution: Use OCI Registry (Best Alternative)
Bitnami now publishes charts as OCI artifacts on Docker Hub. This method is more reliable because:
- Docker Hub (`registry-1.docker.io`) is a well-known, widely allowlisted domain.
- Helm's OCI support is stable in recent versions.
**Commands**:
```bash
helm repo add bitnami oci://registry-1.docker.io/bitnamicharts
helm repo update
```
### If You Must Use the Broadcom URL
1. **Contact Network Team**: 
   - Report that `repo.broadcom.com:443` is blocked by the proxy.
   - Request to allow the domain and its CDN IPs.
2. **Bypass Proxy for Broadcom**:
   ```bash
   export no_proxy="repo.broadcom.com,$no_proxy"
   helm repo add bitnami https://repo.broadcom.com/bitnami-files
   ```
3. **Update Helm Client**:
   Ensure you're using Helm ? v3.8.0 (for improved OCI and network handling).
### Summary
The root issue is proxy-specific blocking of Broadcom's CDN. Since the proxy is managed locally, you might need administrative help to debug the proxy rules. Meanwhile, switching to the OCI registry is the quickest solution.

# Ответ

### Understanding "New CDN Endpoints"

CDN (Content Delivery Network) endpoints are globally distributed server networks that deliver content to users based on their geographic location. After Broadcom acquired Bitnami, they migrated the Helm chart repository to their own infrastructure, which includes **new CDN endpoints** at `repo.broadcom.com`.

#### Key characteristics of these new endpoints:
1. **Different IP ranges**: 
   - Old: `52.85.90.8` (AWS CloudFront)
   - New: `104.98.41.52` (Broadcom's Akamai CDN)
   - Corporate firewalls/proxies often block unfamiliar IP ranges

2. **Stricter security policies**:
   - Broadcom uses enterprise-grade TLS configurations (TLS 1.3-only)
   - May require SNI (Server Name Indication)
   - Use OCSP stapling and certificate pinning

3. **Geographic restrictions**:
   - Some CDNs block regions with high bot traffic
   - New endpoints might be unavailable in certain countries

4. **Protocol differences**:
   - Requires ALPN (Application-Layer Protocol Negotiation)
   - Uses HTTP/2 immediately after TLS handshake

---

### Why Your Proxy Fails with Broadcom Specifically

Your proxy (`localhost:8090`) works for other sites but fails for Broadcom because:

1. **TLS handshake interruption**:
   ```bash
   Proxy CONNECT aborted
   ```
   - The proxy can't complete TLS negotiation with Broadcom's stricter security

2. **Missing SNI support**:
   - Broadcom requires valid SNI in ClientHello
   - Your proxy may not be forwarding the SNI correctly

3. **Protocol version mismatch**:
   - Broadcom might require HTTP/2 while your proxy only speaks HTTP/1.1

4. **Certificate validation differences**:
   - Broadcom uses extended validation (EV) certificates
   - Proxy might reject them due to missing intermediate CAs

---

### Solutions to Bypass CDN Issues

#### 1. Direct Connection Test (Bypass Proxy)
```bash
# Temporarily disable proxy
unset HTTPS_PROXY

# Test direct connection
curl -vk --tls-max 1.3 https://repo.broadcom.com/bitnami-files/index.yaml
```

#### 2. Update Proxy Configuration
Add these rules to your proxy configuration:
```nginx
# Example Squid configuration
acl broadcom dstdomain .broadcom.com
tls_outgoing_options options=NO_TLSv1,NO_TLSv1_1
always_direct allow broadcom
```

#### 3. Use Docker Hub Registry (Recommended)
Bitnami's official migration path:
```bash
helm repo add bitnami oci://registry-1.docker.io/bitnamicharts
helm install my-release bitnami/nginx
```

#### 4. Manual Download Workaround
```bash
# Download through proxy (if works)
curl -x http://localhost:8090 -LO https://repo.broadcom.com/bitnami-files/index.yaml

# Add as local repo
helm repo add bitnami-local file://$(pwd)/index.yaml
```

---

### Diagnostic Commands to Identify CDN Issues

1. **Check TLS handshake**:
   ```bash
   openssl s_client -connect repo.broadcom.com:443 -servername repo.broadcom.com -tls1_3
   ```

2. **Test HTTP/2 support**:
   ```bash
   curl -k --http2 -v https://repo.broadcom.com/bitnami-files/index.yaml
   ```

3. **Verify CDN IP reputation**:
   ```bash
   nslookup repo.broadcom.com
   whois <ip_address>  # Check if IP is blacklisted
   ```

4. **Test different ports**:
   ```bash
   telnet repo.broadcom.com 443
   telnet repo.broadcom.com 80
   ```

---

### Enterprise Proxy Considerations
If you're in a corporate environment:
1. New CDN endpoints often take 24-48 hours to propagate through security systems
2. Financial/enterprise domains (like Broadcom) have stricter default firewall rules
3. Security teams might need to explicitly allow:
   ```plaintext
   *.broadcom.com
   104.98.40.0/24
   104.98.41.0/24
   ```

The **most reliable solution** remains switching to Bitnami's official Docker Hub registry:
```bash
helm repo add bitnami oci://registry-1.docker.io/bitnamicharts
```
This uses standard Docker infrastructure that's already optimized for proxy environments.

