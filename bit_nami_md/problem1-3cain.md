# Вопрос

and 
curl --tlsv1.2 -v https://repo.broadcom.com/bitnami-files/index.yaml

* Uses proxy env variable HTTPS_PROXY == 'http://localhost:8090'

*   Trying 127.0.0.1:8090...

* Connected to (nil) (127.0.0.1) port 8090 (#0)

* allocate connect buffer!

* Establish HTTP proxy tunnel to repo.broadcom.com:443

> CONNECT repo.broadcom.com:443 HTTP/1.1

> Host: repo.broadcom.com:443

> User-Agent: curl/7.81.0

> Proxy-Connection: Keep-Alive

> 

* Proxy CONNECT aborted

* CONNECT phase completed!

* Closing connection 0

curl: (56) Proxy CONNECT aborted


# Рассуждения

We are seeing that the curl command is using a proxy (HTTPS_PROXY=http://localhost:8090) and the proxy connection is being aborted.
 This indicates that the proxy is either not working correctly or is blocking the connection to repo.broadcom.com.
 Steps to resolve:
 1. **Verify the proxy settings**:
    - Ensure that the proxy server at `http://localhost:8090` is running and accepting connections.
    - Test the proxy with a known working site (e.g., `curl --proxy http://localhost:8090 https://google.com`).
 2. **Check proxy configuration for Helm**:
    - Helm uses the environment variables `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY`.
    - If the proxy is not intended to be used for Broadcom, add `repo.broadcom.com` to `NO_PROXY`:
        ```bash
        export NO_PROXY=$NO_PROXY,repo.broadcom.com
        ```
    - Alternatively, if the proxy is required but failing, troubleshoot the proxy server.
 3. **Temporarily bypass the proxy** for testing:
    - Unset the proxy variables:
        ```bash
        unset HTTPS_PROXY
        unset HTTP_PROXY
        ```
    - Then retry the curl command and Helm.
 4. **If the proxy is necessary and you cannot bypass it**:
    - Investigate why the proxy is aborting the connection. Check proxy logs and configuration.
    - The error "Proxy CONNECT aborted" might indicate:
        - The proxy server requires authentication.
        - The proxy server is misconfigured.
        - The proxy server blocks the destination.
 5. **For Helm with proxy authentication**:
    - If your proxy requires authentication, include credentials in the proxy URL:
        ```bash
        export HTTPS_PROXY="http://username:password@localhost:8090"
        ```
 6. **Use the OCI registry as a workaround** (if the proxy issue persists and you cannot resolve it immediately):
    ```bash
    helm repo add bitnami oci://registry-1.docker.io/bitnamicharts
    ```
 Let's proceed step by step:
 Step 1: Test the proxy with a different site
    ```bash
    curl -v --proxy http://localhost:8090 https://google.com
    ```
 Step 2: If the above fails, fix the proxy or bypass it for Broadcom.
 Step 3: If you are in a corporate environment, contact your network administrator to ensure that `repo.broadcom.com` is allowed through the proxy.
 Given the above, the immediate solution might be to use the OCI registry if you are blocked by proxy issues.
 However, if you are required to use the proxy and it is aborted, you must fix the proxy configuration.
 Let me know the outcome of testing the proxy with a different site (Step 1). 

# Ответ


The error indicates that **your proxy server at `localhost:8090` is blocking the connection** to Broadcom's repository. Let's resolve this:

### Immediate Solution: Bypass the Proxy for Broadcom
1. **Add Broadcom to `NO_PROXY`**:
   ```bash
   export NO_PROXY="repo.broadcom.com,$NO_PROXY"
   helm repo add bitnami https://repo.broadcom.com/bitnami-files
   ```

2. **If still blocked, temporarily disable proxy**:
   ```bash
   unset HTTPS_PROXY
   helm repo add bitnami https://repo.broadcom.com/bitnami-files
   ```

### If You Must Use the Proxy
1. **Check proxy authentication requirements**:
   ```bash
   export HTTPS_PROXY="http://username:password@localhost:8090"
   ```

2. **Verify proxy functionality**:
   ```bash
   curl -v --proxy http://localhost:8090 https://google.com
   ```

### Recommended Permanent Solution: Use Bitnami's OCI Registry
This bypasses the problematic Broadcom URL entirely:
```bash
helm repo add bitnami oci://registry-1.docker.io/bitnamicharts
helm repo update
helm search repo bitnami
```

### Proxy Troubleshooting Checklist
1. **Check proxy logs** (if accessible):
   ```bash
   tail -f /var/log/proxy.log  # Common log locations vary
   ```

2. **Test direct connection** (temporary):
   ```bash
   curl --proxy "" https://repo.broadcom.com/bitnami-files/index.yaml
   ```

3. **Inspect proxy configuration**:
   - Is `localhost:8090` really your proxy? Check:
     ```bash
     env | grep -i proxy
     ```
   - Does your organization require authentication?
   - Is Broadcom's domain allowlisted in proxy rules?

### Why This Happened
Broadcom's acquisition of VMware/Bitnami has caused infrastructure changes. Many corporate proxies:
1. Block new domains like `repo.broadcom.com` by default
2. Have stricter rules for financial/enterprise domains
3. May require manual updates to allow new CDN endpoints

> The OCI registry method (`oci://registry-1.docker.io/bitnamicharts`) is the most reliable solution as it uses standard Docker Hub infrastructure which is typically allowlisted in proxies.
