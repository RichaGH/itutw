## Вопрос
why mysql dont use /etc/my.cnf.d/custom-data-dir.cnf  after restart service?

## Рассуждения
Alright, the user is asking why MySQL isn't using the custom configuration file `/etc/my.cnf.d/custom-data-dir.cnf` after restarting the service. Let me break this down step by step.

First, I remember that MySQL reads configuration files from specific directories in a certain order. The main file is `/etc/my.cnf`, then it looks into `/etc/my.cnf.d/` for additional `.cnf` files. So, if the custom file is there, MySQL should load it. But the user is having issues, so maybe there's a problem in how the configuration is set up.

One possibility is that there's a conflict with other configuration files. MySQL processes files in alphabetical order, so if another file in `/etc/my.cnf.d/` has settings that override the custom-data-dir.cnf, that could be the issue. For example, if another file sets `datadir` again later, it would override the custom one.

Another angle is syntax errors in the custom config file. If there's a typo or incorrect parameter, MySQL might ignore that file or the specific settings. The user should check the MySQL error logs to see if there are any warnings or errors about the configuration.

Permissions on the custom config file could also be a problem. If the file isn't readable by the MySQL user (usually `mysql`), MySQL might skip it. The user should verify the file permissions and ownership.

SELinux might be blocking MySQL from accessing the new data directory. Even if the directory permissions are correct, SELinux contexts need to be set properly. Using `semanage` and `restorecon` to adjust the context could resolve this.

Also, after changing the data directory, the user needs to ensure that the `mysql` systemd service is pointing to the correct configuration. If the service file specifies a different `--defaults-file`, it might ignore the custom config. Checking the service unit file for any overrides is important.

AppArmor on some systems could similarly restrict access to the new data directory. The user should check AppArmor logs and adjust the profile if necessary.

Another thought: maybe the custom config file isn't being loaded because MySQL is configured to ignore certain files. For example, if the file has an incorrect extension or is in the wrong directory. The user should confirm that the file ends with `.cnf` and is placed in `/etc/my.cnf.d/`.

Testing the configuration without restarting the service could help. Using `mysqld --verbose --help` would show the effective settings. If the custom `datadir` isn't listed there, the config isn't being applied.

If the user moved the existing data directory without properly initializing the new one, MySQL might fail to start. The initialization step with `mysqld --initialize` is crucial for setting up the new directory with the correct system tables.

Lastly, checking the systemd service status and journal logs would provide specific error messages. Commands like `systemctl status mysqld` and `journalctl -xe` can reveal why the service isn't starting with the new config.

In summary, the key areas to check are configuration file order and conflicts, syntax errors, permissions, SELinux/AppArmor policies, service unit overrides, proper initialization of the data directory, and thorough logging to pinpoint the exact issue.

## Ответ

MySQL may not use the custom configuration file `/etc/my.cnf.d/custom-data-dir.cnf` after restarting the service for several reasons. Let’s analyze the potential issues and solutions:

---

### **1. Configuration File Priority**
MySQL loads configuration files in a specific order. If another configuration file overrides your settings, your custom `datadir` might be ignored. The default load order is:
1. `/etc/my.cnf`
2. `/etc/my.cnf.d/*.cnf` (in alphabetical order)
3. `~/.my.cnf` (user-specific)

**Common Conflicts**:
- If another file (e.g., `/etc/my.cnf.d/server.cnf`) defines `datadir`, it might override your custom file.
- The file name `custom-data-dir.cnf` might load **too early** (e.g., if another file loads later and redefines `datadir`).

**Solution**:
- Rename your custom file to load **last**, e.g., `zz-custom-data-dir.cnf` (alphabetically last):
  ```bash
  mv /etc/my.cnf.d/custom-data-dir.cnf /etc/my.cnf.d/zz-custom-data-dir.cnf
  ```
- Verify configuration precedence with:
  ```bash
  mysqld --verbose --help | grep -A1 "Default options"
  ```

---

### **2. Incorrect Syntax in the Custom File**
If the custom file has syntax errors, MySQL will ignore it. For example:
- Missing sections (e.g., `[mysqld]`).
- Typos in the `datadir` or `socket` paths.

**Example Valid File**:
```ini
[mysqld]
datadir=/mnt/mysql-data
socket=/mnt/mysql-data/mysql.sock
log-error=/mnt/mysql-data/mysqld.log
```

**Diagnose Syntax Errors**:
- Check MySQL error logs:
  ```bash
  journalctl -u mysqld
  ```
- Validate syntax with:
  ```bash
  mysqld --defaults-file=/etc/my.cnf.d/zz-custom-data-dir.cnf --validate-config
  ```

---

### **3. Permissions on the Data Directory**
If MySQL cannot access the custom data directory, it will fail to start:
- **Ownership**: The directory must be owned by the `mysql` user.
- **Permissions**: Use `750` for the directory.

**Fix**:
```bash
chown -R mysql:mysql /mnt/mysql-data
chmod 750 /mnt/mysql-data
```

---

### **4. SELinux Context Issues**
On CentOS 8 Stream, SELinux might block MySQL from accessing the custom directory.

**Solution**:
1. Set the correct SELinux context:
   ```bash
   semanage fcontext -a -t mysqld_db_t "/mnt/mysql-data(/.*)?"
   restorecon -Rv /mnt/mysql-data
   ```
2. Verify with:
   ```bash
   ls -lZ /mnt/mysql-data
   ```

---

### **5. Systemd Service Overrides**
The `mysqld` service might override configuration paths. Check for customizations:
```bash
systemctl cat mysqld
```

If the service defines `--datadir` or `--defaults-file`, it will override your config.  
**Example Fix**:
```bash
# Create a systemd override
systemctl edit mysqld

# Add these lines to override nothing (use default config):
[Service]
ExecStart=
ExecStart=/usr/sbin/mysqld --defaults-file=/etc/my.cnf
```

---

### **6. Missing Data Directory Initialization**
If the data directory was not properly initialized, MySQL will fail to start.  
**Fix**:
1. Stop MySQL:
   ```bash
   systemctl stop mysqld
   ```
2. Initialize the data directory:
   ```bash
   mysqld --initialize --user=mysql --datadir=/mnt/mysql-data
   ```
3. Restart MySQL:
   ```bash
   systemctl start mysqld
   ```

---

### **7. Verify Configuration**
To confirm MySQL is using the correct `datadir`:
```bash
mysql -uroot -p -e "SHOW VARIABLES LIKE 'datadir';"
```

The output should show:
```
+---------------+-------------------+
| Variable_name | Value             |
+---------------+-------------------+
| datadir       | /mnt/mysql-data/  |
+---------------+-------------------+
```

---

### **Summary of Fixes**
1. Ensure the custom config file is loaded last (e.g., `zz-custom-data-dir.cnf`).
2. Validate syntax and permissions.
3. Check for SELinux/AppArmor restrictions.
4. Confirm no systemd service overrides.
5. Reinitialize the data directory if needed.

Let me know if you need further troubleshooting!


