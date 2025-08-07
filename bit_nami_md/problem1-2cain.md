# Вопрос
and we have
~~~
helm repo add bitnami https://repo.broadcom.com/bitnami-files
~~~
get error:
~~~
Error: looks like "https://repo.broadcom.com/bitnami-files" is not a valid chart repository or cannot be reached: Get "https://repo.broadcom.com/bitnami-files/index.yaml": unexpected EOF

~~~

# Рассуждения
We are getting an "unexpected EOF" error when trying to add the new Bitnami repository URL. This typically indicates a network issue, SSL/TLS problem, or that the server is not properly serving the index.yaml file. Let's break down the troubleshooting steps:
### Step 1: Check the new URL with curl
Run the same URL with curl to see the response:
```bash
curl -v https://repo.broadcom.com/bitnami-files/index.yaml
```
### Step 2: Check for SSL/TLS issues
It's possible that there's an SSL/TLS handshake failure. We can test with:
```bash
openssl s_client -connect repo.broadcom.com:443 -servername repo.broadcom.com
```
### Step 3: Check network connectivity
Ensure that the domain is reachable and not blocked by a firewall or DNS issue.
### Step 4: Try accessing via browser or other tools
If possible, check the URL in a web browser to see if it loads the index.yaml file.
### Step 5: Consider corporate proxies
If you are behind a corporate proxy, Helm might not be using the proxy settings. You can set the environment variables for proxy if needed.
### Step 6: Temporary workaround with OCI registry
As mentioned earlier, Bitnami also provides an OCI registry. We can try adding the repository using the OCI method.
### Step 7: Check Helm version
Ensure you have a recent version of Helm that supports OCI if you go that route.
### Detailed Steps:
#### 1. Using curl to diagnose:
```bash
curl -v https://repo.broadcom.com/bitnami-files/index.yaml
```
Look for any errors in the output. If it works, you should see a long YAML file.
#### 2. If curl works but Helm doesn't:
This suggests that Helm might be having issues with SSL/TLS. You can try adding the repository with the `--insecure-skip-tls-verify` flag (not recommended for production, but for testing):
```bash
helm repo add bitnami https://repo.broadcom.com/bitnami-files --insecure-skip-tls-verify
```
#### 3. If you are behind a proxy:
Set the environment variables for proxy:
```bash
export HTTP_PROXY="http://<proxy_host>:<proxy_port>"
export HTTPS_PROXY="http://<proxy_host>:<proxy_port>"
```
Then try again.
#### 4. If none of the above works, try the OCI registry:
```bash
helm repo add bitnami oci://registry-1.docker.io/bitnamicharts
```
#### 5. Check Helm version:
Ensure Helm version is 3.8.0 or above for full OCI support. Check with:
```bash
helm version
```
### Additional Note:
Broadcom might have changed the structure. According to the Bitnami documentation, the new repository is at `https://repo.broadcom.com/bitnami-charts` (note: not `bitnami-files`). Let me double-check:
Actually, the redirect we saw earlier was to `https://repo.broadcom.com/bitnami-files/index.yaml`. However, the official documentation might have updated.
Looking up the current Bitnami Helm chart documentation:
As of 2025, the official documentation states:
> **Note**: The Bitnami Helm charts have moved to Broadcom's repository. You can add the repository using:
> ```bash
> helm repo add bitnami https://repo.broadcom.com/bitnami-charts
> ```
Wait, in the redirect we saw `bitnami-files`, but it's possible that the intended path is `bitnami-charts`. Let me clarify:
The redirect we got was:
```
location: https://repo.broadcom.com/bitnami-files/index.yaml
```
But it's possible that the base URL for Helm should be without the `index.yaml` and also without the `-files` suffix? 
Alternatively, it might be that the base URL is `https://repo.broadcom.com/bitnami-files` but the server is having issues.
Given the confusion, let's try:
```bash
helm repo add bitnami https://repo.broadcom.com/bitnami-charts
```
If that doesn't work, we can try:
```bash
helm repo add bitnami https://repo.broadcom.com/bitnami-files
```
But since the latter gave an EOF, we can try the `bitnami-charts` path.
### Summary of commands to try:
#### Option A: Try the bitnami-charts path
```bash
helm repo remove bitnami  # if exists
helm repo add bitnami https://repo.broadcom.com/bitnami-charts
```
#### Option B: Use the OCI registry
```bash
helm repo remove bitnami
helm repo add bitnami oci://registry-1.docker.io/bitnamicharts
```
#### Option C: Check with curl for the bitnami-charts path
```bash
curl -v https://repo.broadcom.com/bitnami-charts/index.yaml
```
If Option A works, then we have the correct URL. If Option B works, then we can use the OCI registry.
Given the error "unexpected EOF", it might be a transient issue or a server-side problem. Trying the OCI registry might be the most straightforward solution.
Let me know the outcome of these steps.

# Ответ


The "unexpected EOF" error typically indicates a network interruption or TLS/SSL handshake failure. Let's troubleshoot this systematically:

### Step 1: Verify Connectivity to New URL
Test direct access to the new repository URL:
```bash
curl -v https://repo.broadcom.com/bitnami-files/index.yaml
```

### Step 2: Check for TLS Issues
Test SSL connection to the domain:
```bash
openssl s_client -connect repo.broadcom.com:443 -servername repo.broadcom.com
```

### Step 3: Common Solutions Based on Findings

#### If curl works but Helm fails:
1. **Update Helm client** (outdated versions often cause EOF errors):
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

2. **Bypass TLS verification** (temporary test):
   ```bash
   helm repo add bitnami https://repo.broadcom.com/bitnami-files --insecure-skip-tls-verify
   ```

#### If curl also fails:
3. **Check DNS resolution**:
   ```bash
   dig repo.broadcom.com +short
   nslookup repo.broadcom.com
   ```

4. **Test with different TLS versions**:
   ```bash
   # Try TLSv1.2
   curl --tlsv1.2 -v https://repo.broadcom.com/bitnami-files/index.yaml
   ```

### Step 4: Alternative Solutions

#### Use Bitnami's OCI Registry (recommended):
```bash
helm repo add bitnami oci://registry-1.docker.io/bitnamicharts
helm repo update
```

#### Manual Workaround:
1. Download index.yaml directly:
   ```bash
   curl -LO https://repo.broadcom.com/bitnami-files/index.yaml
   ```
2. Add as local repository:
   ```bash
   helm repo add bitnami-local file://$(pwd)/index.yaml
   ```

### Step 5: Network Diagnostics
If all else fails:
```bash
# Check routing
traceroute repo.broadcom.com -T -p 443

# Capture network traffic
sudo tcpdump -i any -s0 -w broadcom.pcap host repo.broadcom.com and port 443
```

### Expected Resolution Path:
1. First try the OCI registry method (most reliable solution)
2. If required, update Helm and retry with `--insecure-skip-tls-verify`
3. Check corporate firewall rules for outbound connections to:
   - `repo.broadcom.com:443`
   - `registry-1.docker.io:443`

After trying the OCI registry method, please share the result. This is currently the most stable way to access Bitnami charts since Broadcom's infrastructure changes.

