# Вопрос
You are an OT engineer with 20 years of experience. You need to upgrade Centos 8 Stream to almalinux 9.5. httpd is already installed and configured on the server in reverse proxy mode. Before updating, you want to save all the configurations of the httpd service so that after updating you can check the changes and, if necessary, configure the service again. And you need to explain all your actions to a young colleague and explain to a young colleague the meaning of the commands you enter and the results of their execution.

# Рассуждения
We are going to upgrade from CentOS 8 Stream to AlmaLinux 9.5.

Since httpd is installed and configured in reverse proxy mode, we want to save all configurations so that after the upgrade we can compare and reconfigure if necessary.

Steps:

1. Backup the current httpd configuration and other important files.

2. Perform the OS upgrade.

3. After upgrade, compare the old and new configurations and restore as needed.

Let's break down the steps for the young colleague:

Step 1: Backup httpd configuration

- The main configuration directory for Apache (httpd) is typically `/etc/httpd`.

- We should also note the state of the service (enabled? running?) and the list of installed modules and packages.

Let's start by creating a backup directory and copying the current configuration.

We'll use the following commands:

mkdir /root/backup_httpd

cp -a /etc/httpd /root/backup_httpd/etc_httpd

Explanation:

- `mkdir /root/backup_httpd`: creates a directory named `backup_httpd` in the root user's home directory.

- `cp -a /etc/httpd ...`: the `-a` option preserves the attributes (like permissions, timestamps) and copies recursively.

We should also backup the list of installed httpd modules and packages.

rpm -qa | grep httpd > /root/backup_httpd/httpd_packages.list

rpm -qa | grep mod_ > /root/backup_httpd/httpd_modules.list

Explanation:

- `rpm -qa` lists all installed packages.

- We are filtering for packages with 'httpd' in their name and saving that list to a file.

- Similarly, we save the list of modules (often starting with 'mod_').

Additionally, we should note the current state of the service:

systemctl status httpd > /root/backup_httpd/httpd_service_status.txt

systemctl is-enabled httpd > /root/backup_httpd/httpd_service_enabled.txt

Explanation:

- `systemctl status httpd` shows detailed status of the httpd service.

- `systemctl is-enabled httpd` shows whether the service is enabled to start at boot.

Also, if there are any custom configurations or scripts, we might want to note the locations of configuration files we modified.

Typically, the main configuration file is `/etc/httpd/conf/httpd.conf` and additional configurations are in `/etc/httpd/conf.d/`.

Since we are in reverse proxy mode, we might have specific configurations in the conf.d directory or in the main config.

We should also check if there are any SSL certificates and keys that are being used. They are often stored in `/etc/pki/tls/` or `/etc/httpd/ssl/`. We should back those up too.

cp -a /etc/pki/tls /root/backup_httpd/etc_pki_tls   (if exists and used)

cp -a /etc/httpd/ssl /root/backup_httpd/etc_httpd_ssl   (if exists)

However, note that certificates are often managed separately and might be linked from other locations. We should also check the configuration files for paths to certificates.

Step 2: The OS upgrade process from CentOS 8 Stream to AlmaLinux 9.5.

We will follow the official migration guide from CentOS 8 to AlmaLinux 8, and then upgrade to AlmaLinux 9. However, note that we are starting from CentOS 8 Stream and going to AlmaLinux 9.5.

Important: The migration path might not be directly supported. We must check the AlmaLinux migration documentation.

But note: The CentOS 8 Stream is a rolling release and might not have a direct migration path to AlmaLinux 8. We might need to do a fresh install? However, there are community tools.

Alternatively, we can use the `elevate` project by AlmaLinux for in-place upgrades from CentOS 7 to AlmaLinux 8 and then to 9, but for CentOS 8 Stream to AlmaLinux 8, it's not officially supported.

However, since the original OS is CentOS 8 Stream, we might consider converting to AlmaLinux 8 first and then upgrading to 9.

The steps for converting CentOS 8 to AlmaLinux 8 are:

- Install the `almalinux-release` package and migrate using the `almalinux-deploy` script.

But note: CentOS 8 Stream is not exactly CentOS 8. It is a rolling preview of RHEL. So we must be cautious.

Given the complexity and risk, it is often recommended to back up the data and applications and perform a fresh install of AlmaLinux 9. However, if we must do an in-place upgrade, we proceed at our own risk.

Since the problem states we are upgrading, I will outline the steps as per the elevate project for CentOS 8 to AlmaLinux 8 and then to 9, but note that CentOS 8 Stream might have differences.

Alternatively, we can try to migrate from CentOS 8 Stream to AlmaLinux 8 using the following:

# Install the required packages

dnf install -y epel-release

dnf install -y dnf-plugins-core

# Add the AlmaLinux repository

curl -O https://raw.githubusercontent.com/AlmaLinux/almalinux-deploy/master/almalinux-deploy.sh

chmod +x almalinux-deploy.sh

./almalinux-deploy.sh

# This script should convert the system to AlmaLinux 8.

Then we can upgrade to AlmaLinux 9 using the Leapp utility as described in the AlmaLinux documentation.

However, note that the server is in production and we have a critical service (httpd). We must have a rollback plan.

Given the complexity, we might consider:

- Testing the entire process in a staging environment that mirrors production.

- Taking a full system backup (using tools like `rsync` or a snapshot if on a VM) so that we can restore if the upgrade fails.

Step 3: After the upgrade to AlmaLinux 9.5, we will have a new version of httpd. We will then compare the old configuration with the new one.

- We can use `diff` to compare the old backup with the new configuration.

For example:

diff -ur /root/backup_httpd/etc_httpd /etc/httpd

This will show the differences between the old and new configuration.

- We might have to merge the changes, especially the reverse proxy settings.

- Also, we must check if the modules we had are still available and enabled.

- We must also check the service state and enable/start it if it was enabled and running before.

- We must also check the firewall settings (firewalld) and SELinux context if any.

Considering the complexity and the fact that the young colleague is learning, we must explain that:

- Backing up is critical to avoid data loss and to have a reference.

- OS upgrades, especially major version jumps (like from EL8 to EL9), can break configurations and require adjustments.

- We must test the service after the upgrade.

Let's write the actual commands we would use for the backup:

# Create backup directory

mkdir -p /root/backup_httpd

# Backup the entire httpd configuration

cp -a /etc/httpd /root/backup_httpd/etc_httpd

# Backup the list of httpd packages and modules

rpm -qa | grep httpd > /root/backup_httpd/httpd_packages.list

rpm -qa | grep mod_ > /root/backup_httpd/httpd_modules.list

# Backup the service state

systemctl status httpd > /root/backup_httpd/httpd_service_status.txt

systemctl is-enabled httpd > /root/backup_httpd/httpd_service_enabled.txt

# If we have custom SSL certificates, back them up too.

# Check the httpd configuration for SSLCertificateFile and SSLCertificateKeyFile directives to know where the certificates are.

# For example, we can search:

grep -i SSLCertificate /etc/httpd/conf.d/*.conf /etc/httpd/conf/httpd.conf

# If we find any, we can back up those directories or files.

# Let's assume we found a directory: /etc/httpd/ssl, then:

if [ -d /etc/httpd/ssl ]; then

cp -a /etc/httpd/ssl /root/backup_httpd/etc_httpd_ssl

fi

# Similarly, for /etc/pki/tls if used

if [ -d /etc/pki/tls ]; then

cp -a /etc/pki/tls /root/backup_httpd/etc_pki_tls

fi

#### Also, back up the virtual hosts if they are in a non-standard location? Typically they are in /etc/httpd/conf.d/ or /etc/httpd/conf/httpd.conf.

#### Additionally, we should back up the document root if it contains any application code? But that is typically in /var/www. The problem says configurations, so we are focusing on configurations.

##### However, note that the reverse proxy configuration might point to backend servers. We should also back up any other configuration that is part of the reverse proxy setup (like backend server lists, etc.) but that should be in the httpd configuration.

#### Finally, we can create a tarball of the backup directory and store it off the server for safety.

tar -czf /root/backup_httpd.tar.gz -C /root backup_httpd

# Then we can copy this tarball to a safe location.

After the OS upgrade, we will:

- Compare the old and new configurations and merge as needed.

- Restore the SSL certificates to their original locations if they are not already there (the upgrade process might not touch them, but we have a backup in case).

- Check the list of modules and install any missing ones (if they are still available in AlmaLinux 9).

- Start the httpd service and check the status.

# To compare:

diff -ur /root/backup_httpd/etc_httpd /etc/httpd

# We can also use a tool like `meld` if we have a GUI, but on a server we use `diff` or `vimdiff`.

# We should also check the service state and enable it if it was enabled before.

enabled_before=$(cat /root/backup_httpd/httpd_service_enabled.txt)

if [ "$enabled_before" = "enabled" ]; then

systemctl enable httpd

fi

# And then start the service if it was running? We can check the status file we saved, but typically we start it if we have restored the configuration.

systemctl start httpd

# Then check the status and logs.

systemctl status httpd

journalctl -u httpd -b   # logs since last boot

# Also, check the web server is functioning as expected.

Important: The upgrade process might change the default configuration of httpd. We must be cautious.

Additionally, note that AlmaLinux 9.5 comes with a newer version of httpd. Some configuration directives might have been deprecated or changed.

We must review the release notes of the new httpd version.

Given the complexity of the OS upgrade, we might not go into the full OS upgrade steps in this script, but focus on the httpd backup and restore.

Let me know if you have any questions, young padawan


# Ответ

### Step-by-Step Guide: Backing Up Apache HTTPD Configuration for OS Upgrade

#### **1. Create Backup Directory**
```bash
mkdir -p /root/httpd_backup
```
- **Explanation**:  
  Creates a directory `/root/httpd_backup` to store all backups. `-p` ensures parent directories are created if missing.

---

#### **2. Backup Configuration Files**
```bash
cp -a /etc/httpd /root/httpd_backup/etc_httpd
```
- **Explanation**:  
  - Copies the entire `/etc/httpd` directory (main Apache config) to `/root/httpd_backup/etc_httpd`.  
  - `-a` preserves permissions, timestamps, and directory structure (critical for restoration).

---

#### **3. Backup Virtual Hosts and Custom Configs**
```bash
cp -a /etc/httpd/conf.d /root/httpd_backup/conf.d
```
- **Explanation**:  
  Backups `/etc/httpd/conf.d`, where reverse proxy configurations (e.g., `*.conf` files) typically reside.

---

#### **4. Backup SSL Certificates (If Used)**
```bash
cp -a /etc/pki/tls /root/httpd_backup/tls
```
- **Explanation**:  
  Copies SSL/TLS certificates (commonly stored in `/etc/pki/tls`). Needed if your reverse proxy uses HTTPS.

---

#### **5. Backup Service Status**
```bash
systemctl status httpd > /root/httpd_backup/httpd_status.txt
systemctl is-enabled httpd > /root/httpd_backup/httpd_enabled.txt
```
- **Explanation**:  
  - `httpd_status.txt`: Saves current running state (active/inactive).  
  - `httpd_enabled.txt`: Checks if Apache starts on boot (`enabled`/`disabled`).  

---

#### **6. Backup Installed Modules**
```bash
httpd -M > /root/httpd_backup/httpd_modules.txt
```
- **Explanation**:  
  Lists loaded Apache modules (e.g., `proxy_module`, `ssl_module`). Confirms critical reverse proxy modules are retained post-upgrade.

---

#### **7. Backup RPM Package List**
```bash
rpm -qa | grep httpd > /root/httpd_backup/httpd_packages.txt
```
- **Explanation**:  
  Saves installed Apache-related RPMs. After upgrade, compare with `rpm -qa | grep httpd` to verify package consistency.

---

#### **8. Verify Backup Integrity**
```bash
tree /root/httpd_backup
```
- **Expected Output**:  
  ```bash
  /root/httpd_backup
  ??? conf.d
  ??? etc_httpd
  ??? httpd_enabled.txt
  ??? httpd_modules.txt
  ??? httpd_packages.txt
  ??? httpd_status.txt
  ??? tls
  ```
- **Explanation**:  
  Confirms all backups exist. Use `cat` to view file contents (e.g., `cat /root/httpd_backup/httpd_modules.txt`).

---

### **Why This Process Matters**
1. **Risk Mitigation**:  
   OS upgrades (CentOS 8 Stream ? AlmaLinux 9.5) may overwrite configs. Backups allow:  
   - **Safe comparison** between old/new configs (`diff -r /root/httpd_backup/etc_httpd /etc/httpd`).  
   - **Restoration** of custom settings (e.g., reverse proxy rules).  

2. **Audit Trail**:  
   - `httpd_modules.txt`/`httpd_packages.txt` ensure no modules (e.g., `mod_proxy`) are missing post-upgrade.  
   - Service status files prevent unexpected boot behavior.  

3. **SSL Continuity**:  
   Certificates (`tls/` backup) avoid HTTPS breakdowns.  

---

### **Post-Upgrade Steps**
1. **Compare Configs**:  
   ```bash
   diff -r /root/httpd_backup/etc_httpd /etc/httpd
   ```
   - Investigate changes in `httpd.conf` or `conf.d/` files. Reapply custom reverse proxy logic if needed.

2. **Validate Service State**:  
   ```bash
   systemctl enable httpd  # If httpd_enabled.txt says "enabled"
   systemctl start httpd   # If httpd_status.txt showed "active"
   ```

3. **Check Modules**:  
   ```bash
   httpd -M | grep proxy   # Verify reverse proxy modules
   ```

### **Key Wisdom for Junior Engineers**
> **"Backups are your oxygen mask in production. Verify them twice before jumping."**  
> - Always check:  
>   - Permissions (e.g., `ls -l /root/httpd_backup/etc_httpd/conf`).  
>   - Critical files (e.g., reverse proxy configs in `conf.d/`).  
>   - Service dependencies (modules, SSL certs).  
> - Document *why* configs exist (e.g., "Reverse proxy for AppX"). This saves hours during recovery.  

Let me know when you’re ready to proceed with the OS upgrade! ??
