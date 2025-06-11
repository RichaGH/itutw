# Вопрос
In addition to the above, you should show your young colleague the httpd configs and explain what the configuration file blocks mean. Where he can view the settings of the virtual hosts, where the certificate settings are specified, where the reverse proxy module is enabled.

# Рассуждения
We are going to explore the typical structure of Apache HTTPD configuration on CentOS/AlmaLinux.

The main configuration directory is `/etc/httpd`.

Important subdirectories and files:

/etc/httpd/conf/httpd.conf        - Main configuration file.

/etc/httpd/conf.d/                - Additional configuration files (often for virtual hosts and modules).

/etc/httpd/conf.modules.d/        - Configuration files for loading modules.

Since the server is configured in reverse proxy mode, we expect to find:

- The proxy module enabled (mod_proxy and mod_proxy_http at least).

- Virtual host(s) or configuration files that define the reverse proxy settings.

Let's break down the key parts:

1. **Main Configuration (httpd.conf)**:

- This file sets global settings, includes other configuration files, and may define the main server.

- We are particularly interested in the inclusion of other directories and the base settings.

2. **Module Configuration (conf.modules.d/)**:

- Files in this directory load modules. For reverse proxy, we expect a line like:

LoadModule proxy_module modules/mod_proxy.so

LoadModule proxy_http_module modules/mod_proxy_http.so

3. **Virtual Hosts and Reverse Proxy Configs (conf.d/)**:

- Typically, reverse proxy configurations are defined in separate .conf files in `conf.d/`.

- These files may set up a virtual host (or use the main server) and define the reverse proxy rules.

4. **SSL Certificate Configuration**:

- If the reverse proxy is using HTTPS, we expect SSL configuration in the virtual host.

- The SSL certificate and key are usually set with:

SSLCertificateFile /path/to/cert.pem

SSLCertificateKeyFile /path/to/privkey.pem

- These paths are often in `/etc/pki/tls/certs` and `/etc/pki/tls/private`.

Let's show the junior colleague how to locate and interpret these.

We'll use the backup we made earlier to look at the configs without affecting the live server.

Since we backed up to `/root/httpd_backup`, we can examine the backup:

cd /root/httpd_backup/etc_httpd

But note: the original paths are preserved under `/root/httpd_backup/etc_httpd`. So:

- The main config: /root/httpd_backup/etc_httpd/conf/httpd.conf

- The conf.d: /root/httpd_backup/etc_httpd/conf.d/*.conf

- The conf.modules.d: /root/httpd_backup/etc_httpd/conf.modules.d/*.conf

We can use commands like `cat`, `grep`, `less` to view these.

However, since we are in a teaching context, let me explain by example:

Example 1: Checking loaded proxy modules

Look in the modules directory of the backup:

```bash

grep -r "proxy_module" /root/httpd_backup/etc_httpd/conf.modules.d/

```

Expected: You should see a line that loads mod_proxy.so.

Example 2: Finding reverse proxy configuration

Typically, reverse proxy settings are in a .conf file in `conf.d`. We can look for common directives:

```bash

grep -r "ProxyPass" /root/httpd_backup/etc_httpd/conf.d

```

`ProxyPass` is a common directive for setting up a reverse proxy path.

Example 3: SSL Configuration

Look for `SSLCertificateFile` in the conf.d:

```bash

grep -r "SSLCertificateFile" /root/httpd_backup/etc_httpd/conf.d

```

This will show the path to the SSL certificate.

Now, let's break down a typical reverse proxy configuration file (e.g., `/root/httpd_backup/etc_httpd/conf.d/reverse_proxy.conf`):

```apache

<VirtualHost *:80>

ServerName example.com

# Redirect all HTTP to HTTPS

Redirect permanent / https://example.com/

</VirtualHost>

<VirtualHost *:443>

ServerName example.com

SSLEngine on

SSLCertificateFile /etc/pki/tls/certs/example.com.crt

SSLCertificateKeyFile /etc/pki/tls/private/example.com.key

# Reverse proxy configuration

ProxyPass / http://backend-server:8080/

ProxyPassReverse / http://backend-server:8080/

# Additional proxy settings

<Proxy *>

Require all granted

</Proxy>

</VirtualHost>

```

Explanation of the above:

- The first VirtualHost listens on port 80 (HTTP) and redirects to HTTPS.

- The second VirtualHost listens on port 443 (HTTPS).

- `SSLEngine on` enables SSL for this virtual host.

- `SSLCertificateFile` and `SSLCertificateKeyFile` point to the certificate and private key.

- `ProxyPass` maps a path (here, the root "/") to a backend server (http://backend-server:8080/).

- `ProxyPassReverse` ensures that responses from the backend are adjusted to match the proxy's URL.

- The `<Proxy>` block sets access control for the proxy. Here, it allows all.

Important: The actual configuration may vary. The junior colleague should look for these patterns.

Steps for the junior:

1. List the files in `conf.d`:

```bash

ls /root/httpd_backup/etc_httpd/conf.d

```

2. Look for files that may contain the reverse proxy configuration (names might include the domain or "proxy", "reverse", etc.).

3. Use `cat` or `less` to view the contents of a candidate file.

4. Identify the VirtualHost blocks and the proxy directives.

5. Note the SSL certificate paths and verify they were backed up (in our backup, we have `/root/httpd_backup/tls`).

After the upgrade, when restoring, we would:

- Compare the new default configuration with our backup (using `diff`).

- If necessary, copy the backed-up configuration files back to `/etc/httpd` (but be cautious: new OS version might have updated configuration requirements).

- Or, we might manually merge changes, especially if the upgrade provides new default configs.

Let me know if you need further clarification!

# Ответ


### Step-by-Step: Understanding Apache HTTPD Configuration Structure  
As an OT engineer, it's critical to understand how Apache organizes its configurations. Let's examine the key components using your backup at `/root/httpd_backup`. All paths refer to locations **within the backup directory**.

---

#### **1. Main Configuration Structure**  
```bash
tree -L 3 /root/httpd_backup/etc_httpd
```
```output
/root/httpd_backup/etc_httpd
??? conf
?   ??? httpd.conf              # Primary config file
?   ??? magic
??? conf.d
?   ??? ssl.conf                # SSL/TLS base settings
?   ??? reverse-proxy.conf      # Custom reverse proxy config
?   ??? ...                     # Other virtual hosts
??? conf.modules.d
    ??? 00-proxy.conf           # Proxy module activation
    ??? 00-ssl.conf             # SSL module activation
```

---

#### **2. Critical Configuration Files Explained**  
**a. Module Activation (`conf.modules.d/00-proxy.conf`)**  
```bash
cat /root/httpd_backup/etc_httpd/conf.modules.d/00-proxy.conf
```
```apache
# Load proxy modules
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```
- **What this means**:  
  - `LoadModule`: Dynamically loads Apache modules at startup  
  - `proxy_module`: Core proxy functionality  
  - `proxy_http_module`: Enables HTTP/HTTPS proxying  
- **Why it matters**:  
  Without these, your reverse proxy won't function. Verify these exist post-upgrade.

---

**b. Reverse Proxy Configuration (`conf.d/reverse-proxy.conf`)**  
*(Typical reverse proxy setup)*  
```bash
cat /root/httpd_backup/etc_httpd/conf.d/reverse-proxy.conf
```
```apache
<VirtualHost *:80>
    ServerName yourdomain.com
    Redirect permanent / https://yourdomain.com/  # Force HTTPS
</VirtualHost>

<VirtualHost *:443>
    ServerName yourdomain.com
    
    # SSL Configuration (linked to certs)
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/yourdomain.crt
    SSLCertificateKeyFile /etc/pki/tls/private/yourdomain.key
    
    # Reverse Proxy Rules
    ProxyPass "/app" "http://backend-server:8080/"
    ProxyPassReverse "/app" "http://backend-server:8080/"
    
    # Access Control
    <Proxy *>
        Require all granted
    </Proxy>
    
    # Headers for Backend
    RequestHeader set X-Forwarded-Proto "https"
</VirtualHost>
```

---

#### **3. Key Configuration Blocks Decoded**  
**a. Virtual Host Blocks**  
```apache
<VirtualHost *:443> ... </VirtualHost>
```
- **Purpose**: Defines a website/listener bound to port 443 (HTTPS)  
- **Where to find**: `conf.d/*.conf` files (each file usually represents one site)  
- **Critical parameters**:  
  - `ServerName`: Domain name (e.g., `yourdomain.com`)  
  - `*:443`: Listens on all IPs, port 443  

**b. SSL/TLS Configuration**  
```apache
SSLEngine on
SSLCertificateFile /etc/pki/tls/certs/yourdomain.crt
SSLCertificateKeyFile /etc/pki/tls/private/yourdomain.key
```
- **Purpose**: Enables HTTPS and specifies certificate paths  
- **Where to find**: Inside `<VirtualHost *:443>` blocks  
- **Backup location**: Certificates are in `/root/httpd_backup/tls/`  

**c. Reverse Proxy Directives**  
```apache
ProxyPass "/app" "http://backend-server:8080/"
ProxyPassReverse "/app" "http://backend-server:8080/"
```
- **Purpose**:  
  - `ProxyPass`: Forwards requests from `/app` to backend server  
  - `ProxyPassReverse`: Rewrites backend's response headers to match proxy URL  
- **Real-world use**: Hides backend server details from clients  

**d. Access Control**  
```apache
<Proxy *>
    Require all granted
</Proxy>
```
- **Purpose**: Defines who can access proxied content  
- **Security note**: `Require all granted` allows public access – tighten this in production!  

**e. Header Management**  
```apache
RequestHeader set X-Forwarded-Proto "https"
```
- **Purpose**: Informs backend this request came via HTTPS  
- **Critical for**: Applications that generate URLs (e.g., avoids HTTP redirect loops)  

---

#### **4. Post-Upgrade Validation Checklist**  
After OS upgrade to AlmaLinux 9.5:  
1. **Verify modules**:  
   ```bash
   httpd -M | grep -E 'proxy|ssl'
   ```
   Should show `proxy_module`, `proxy_http_module`, `ssl_module`.

2. **Compare configs**:  
   ```bash
   diff -r /root/httpd_backup/etc_httpd/conf.d /etc/httpd/conf.d
   ```
   Focus on changes in SSL paths or proxy rules.

3. **Test SSL certificates**:  
   ```bash
   openssl x509 -in /etc/pki/tls/certs/yourdomain.crt -noout -dates
   ```
   Ensure certificates are valid and paths match configs.

4. **Check proxy functionality**:  
   ```bash
   curl -kI https://yourdomain.com/app
   ```
   Should return HTTP 200 (adjust URL to match your `ProxyPass` path).

---

### **Knowledge Transfer: Pro Tips for Junior Engineers**  
1. **Config Inheritance**:  
   - Settings in `conf.d/*.conf` override `httpd.conf`  
   - Last loaded config file wins! (Alphabetical order matters)  

2. **Debugging Tools**:  
   - `apachectl configtest`: Validate syntax before restart  
   - `journalctl -u httpd`: View service logs  
   - `curl -v http://localhost:80/app`: Test routing locally  

3. **Security Golden Rules**:  
   - Never store certificates in web-accessible directories  
   - Use `Require ip 192.168.1.0/24` instead of `all granted` in `<Proxy>` blocks  
   - Always test configs in staging before production  

> "The devil's in the details – a single misplaced slash in `ProxyPass` can break your entire app. Test each change like it's your first deployment."  

Let me know when you're ready to proceed with the OS upgrade! I'll walk you through the AlmaLinux conversion next.
