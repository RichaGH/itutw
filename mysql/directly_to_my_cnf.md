## Вопрос
Ok, validate-config say what file nether will be used because it not exists in list of used files. In that list exist /etc/my.cnf , Rewrite script to apply chacges in /etc/my.cnf directly without using /etc/my.cnf.d/custom-data-dir.cnf

## Рассуждения
Okay, the user is having trouble with MySQL not using the custom configuration file they created in /etc/my.cnf.d. They mentioned that when they ran mysqld --validate-config, it didn't include their custom file because it wasn't in the list of used files. Instead, only /etc/my.cnf was listed. They want the script to modify /etc/my.cnf directly instead of adding a new file in /etc/my.cnf.d.

First, I need to understand why the custom file wasn't being used. MySQL's configuration loading order might prioritize /etc/my.cnf over the .d directory files, or maybe there's a syntax issue. But the user's main request is to rewrite the script to edit /etc/my.cnf directly.

So the original script added a new file in /etc/my.cnf.d, but if that's not being loaded, perhaps the main my.cnf is taking precedence. The solution is to modify the main my.cnf file instead. However, I should make sure that the script doesn't just overwrite the existing my.cnf but appends the necessary configurations to it. But wait, the user might have an existing my.cnf with other settings. Overwriting it could cause issues. However, the user's instruction says to apply changes directly to /etc/my.cnf, so maybe they want to replace it or append to it.

But the safest approach is to check if the [mysqld] section exists in /etc/my.cnf and modify it, or create the file if it doesn't exist. However, the user's problem might be that the .d directory isn't being read at all, which could be due to how MySQL is compiled or configured. So modifying the main my.cnf directly ensures the settings are applied.

In the previous script, the custom-data-dir.cnf was created in /etc/my.cnf.d, but the user says it's not being used. So the revised script should edit /etc/my.cnf instead.

But how to do that in the script? The script needs to add the [mysqld] section with datadir, socket, and log-error to /etc/my.cnf. However, there's a risk of overwriting existing configurations. To handle this, the script should check if the [mysqld] section already exists and append the settings if it does, or create the section if it doesn't.

Alternatively, the script could use a here document to append the configurations to /etc/my.cnf. But appending might not be the right approach if the file already has a [mysqld] section. The correct way is to modify the existing [mysqld] section or add it if not present.

This can be done using sed or awk. For example, using sed to replace or add the lines under the [mysqld] section. However, handling this in a bash script can be error-prone, especially if the file has complex configurations.

Another approach is to back up the original /etc/my.cnf and then create a new one with the necessary settings. But the user might have other important settings in their existing my.cnf. So maybe the script should append the configurations to the end of the file, assuming that the last occurrence of a setting takes precedence.

Wait, in MySQL, the last occurrence of a parameter in the configuration files takes precedence. So if the script appends the [mysqld] section with the new settings at the end of /etc/my.cnf, those settings should override any previous ones.

But the original /etc/my.cnf might not have a [mysqld] section. So the script should check if the section exists and append the settings accordingly.

Alternatively, the script can use a here document to add the [mysqld] section with the necessary parameters to /etc/my.cnf. If the file already has a [mysqld] section, this would create a duplicate, but MySQL will use the last occurrence, which would be the one the script added. However, this might not be ideal, but given the user's requirement to avoid using the .d directory, this might be the way to go.

So in the script, instead of creating a file in /etc/my.cnf.d, the script will modify /etc/my.cnf directly. The revised script should:

1. Backup the original /etc/my.cnf if it exists.

2. Append the [mysqld] section with datadir, socket, and log-error to /etc/my.cnf.

3. Ensure that the parameters are set correctly.

But there's a risk of breaking existing configurations. To mitigate this, the script should check if the parameters are already set and update them, or add them if not present. However, doing this reliably in a bash script requires more complex text processing.

Alternatively, the script can overwrite /etc/my.cnf with the necessary configuration, but that's risky. The user might have other configurations. However, given the user's request, perhaps they are okay with overwriting or are setting up a new system where this isn't an issue.

In the previous script, the user was using a custom data directory and setting the root password. The key is to ensure that after modifying /etc/my.cnf, MySQL uses the new settings.

Another consideration is that when MySQL is installed, it might generate a default my.cnf file. The script should handle that.

Putting this all together, the revised script will modify /etc/my.cnf directly by appending the [mysqld] section with the necessary settings. This approach ensures that the settings are applied without relying on the .d directory.

Now, to implement this in the script:

- Remove the part where the custom file in /etc/my.cnf.d is created.

- Instead, append the configuration to /etc/my.cnf.

- Use a heredoc to add the [mysqld] section with the parameters.

But to avoid duplicates, the script should check if the section already exists and update it. However, that's complex. Alternatively, the script can comment out existing [mysqld] sections and add the new one. But that's also risky.

Given time constraints, perhaps the simplest way is to append the configuration to /etc/my.cnf, which will override any previous settings for the same parameters.

Thus, in the script:

echo "Configuring MySQL to use $MYSQL_DATA_DIR in /etc/my.cnf..."

cat >> /etc/my.cnf <<EOF

[mysqld]

datadir=$MYSQL_DATA_DIR

socket=$MYSQL_DATA_DIR/mysql.sock

log-error=$MYSQL_DATA_DIR/mysqld.log

EOF

This appends the settings to the end of /etc/my.cnf. MySQL will process this last, so the parameters here will take precedence.

But if there's an existing [mysqld] section in /etc/my.cnf, this will create a second [mysqld] section, which is allowed in MySQL, and the last values will be used.

This should work, but the user should be aware that any existing [mysqld] settings in /etc/my.cnf may be overridden by the appended ones.

Additionally, the script should handle the case where /etc/my.cnf doesn't exist. The cat >> will create it if it doesn't.

Another thing to consider is that the existing /etc/my.cnf might include other files from /etc/my.cnf.d, which could reintroduce the original problem. But the user's issue is that the custom file in /etc/my.cnf.d wasn't being loaded, so modifying /etc/my.cnf directly should take precedence.

In conclusion, the revised script will modify /etc/my.cnf directly by appending the [mysqld] section with the necessary parameters, ensuring that MySQL uses the custom data directory and log file locations. This approach avoids relying on the .d directory and ensures the settings are applied correctly

## Ответ

Below is the revised script that **directly modifies `/etc/my.cnf`** (instead of using `/etc/my.cnf.d/custom-data-dir.cnf`) to ensure MySQL uses the custom data directory. The script includes critical fixes for configuration precedence and dependency handling.

---

### **Revised Script**
```bash
#!/bin/bash

# Check parameters
if [ -z "$1" ] || [ -z "$2" ] || [ -z "$3" ]; then
  echo "Usage: $0 <mysql-major-version> <data-directory> <root-password>"
  echo "Example: $0 8.0 /mnt/mysql-data MyNewSecurePassword123"
  exit 1
fi

MYSQL_MAJOR_VERSION=$1  # e.g., "8.0"
MYSQL_DATA_DIR=$2
MYSQL_ROOT_PASSWORD=$3

# Add MySQL Yum Repository
echo "Adding MySQL Yum Repository..."
rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el8-1.noarch.rpm

# Enable the correct major version repository
MYSQL_REPO_ID="mysql$(echo "$MYSQL_MAJOR_VERSION" | tr -d .)-community"
echo "Enabling MySQL repository: $MYSQL_REPO_ID..."
dnf module -y disable mysql
dnf config-manager --enable "$MYSQL_REPO_ID"

# Install MySQL Server and dependencies
echo "Installing MySQL Server and dependencies..."
dnf install -y mysql-community-server mysql-community-client mysql-community-libs mysql-community-common

# Create custom data directory
echo "Setting up MySQL data directory at $MYSQL_DATA_DIR..."
mkdir -p "$MYSQL_DATA_DIR"
chown -R mysql:mysql "$MYSQL_DATA_DIR"
chmod 750 "$MYSQL_DATA_DIR"

# Stop MySQL service
systemctl stop mysqld

# Backup existing /etc/my.cnf (if it exists)
if [ -f /etc/my.cnf ]; then
  echo "Backing up /etc/my.cnf to /etc/my.cnf.bak..."
  cp /etc/my.cnf /etc/my.cnf.bak
fi

# Configure MySQL to use the custom data directory DIRECTLY in /etc/my.cnf
echo "Updating /etc/my.cnf to use $MYSQL_DATA_DIR..."
cat > /etc/my.cnf <<EOF
[mysqld]
datadir=$MYSQL_DATA_DIR
socket=$MYSQL_DATA_DIR/mysql.sock
log-error=$MYSQL_DATA_DIR/mysqld.log

[client]
socket=$MYSQL_DATA_DIR/mysql.sock

# Include all files from /etc/my.cnf.d (optional)
!includedir /etc/my.cnf.d
EOF

# Initialize MySQL
echo "Initializing MySQL..."
mysqld --initialize --user=mysql --datadir="$MYSQL_DATA_DIR"

# Start MySQL
systemctl start mysqld

# Extract temporary password (wait for log file)
echo "Waiting for MySQL log file..."
while [ ! -f "$MYSQL_DATA_DIR/mysqld.log" ]; do sleep 1; done
MYSQL_TEMP_PASSWORD=$(grep -i 'temporary.*password' "$MYSQL_DATA_DIR/mysqld.log" | awk '{print $NF}')

# Change root password securely
export MYSQL_PWD="$MYSQL_TEMP_PASSWORD"
mysql --user=root --connect-expired-password <<EOF
ALTER USER 'root'@'localhost' IDENTIFIED BY '$MYSQL_ROOT_PASSWORD';
FLUSH PRIVILEGES;
EOF
unset MYSQL_PWD

# Secure MySQL installation
mysql_secure_installation --password="$MYSQL_ROOT_PASSWORD" <<EOF
n
y
y
y
y
EOF

systemctl enable mysqld

echo "MySQL installed successfully:"
echo "- Major version: $MYSQL_MAJOR_VERSION"
echo "- Data directory: $MYSQL_DATA_DIR"
```

---

### **Key Changes Explained**

#### 1. **Direct Modification of `/etc/my.cnf`**
   - The script replaces `/etc/my.cnf` entirely to ensure the `datadir`, `socket`, and `log-error` settings take precedence.
   - A backup of the original `/etc/my.cnf` is created (if it exists) to avoid data loss.
   - The new `/etc/my.cnf` explicitly defines:
     ```ini
     [mysqld]
     datadir=/your/custom/data/dir
     socket=/your/custom/data/dir/mysql.sock
     log-error=/your/custom/data/dir/mysqld.log

     [client]
     socket=/your/custom/data/dir/mysql.sock
     ```
   - The `!includedir /etc/my.cnf.d` line is preserved (optional) to retain compatibility with other configurations.

#### 2. **Force Configuration Precedence**
   - By defining settings directly in `/etc/my.cnf`, you override any conflicting values in other files (e.g., `/etc/my.cnf.d/*.cnf`), as MySQL loads `/etc/my.cnf` first.

#### 3. **Full Path for Socket and Logs**
   - The `socket` and `log-error` paths are explicitly set to the custom data directory to avoid conflicts with the default `/var/lib/mysql`.

#### 4. **SELinux Context Fixes**
   - If SELinux is enabled, run these commands after creating the data directory:
     ```bash
     semanage fcontext -a -t mysqld_db_t "$MYSQL_DATA_DIR(/.*)?"
     restorecon -Rv "$MYSQL_DATA_DIR"
     ```

---

### **Verification Steps**
1. **Check MySQL Configuration**:
   ```bash
   mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW VARIABLES LIKE 'datadir';"
   ```
   Output should show the custom directory:
   ```
   +---------------+---------------------+
   | Variable_name | Value               |
   +---------------+---------------------+
   | datadir       | /mnt/mysql-data/    |
   +---------------+---------------------+
   ```

2. **Validate Log File Location**:
   ```bash
   ls -l "$MYSQL_DATA_DIR/mysqld.log"
   ```

3. **Test Socket Connection**:
   ```bash
   mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -S "$MYSQL_DATA_DIR/mysql.sock" -e "STATUS;"
   ```

---

### **Why This Works**
- **Centralized Configuration**: By defining everything in `/etc/my.cnf`, you eliminate ambiguity in configuration precedence.
- **Atomic Replacement**: Overwriting `/etc/my.cnf` ensures no leftover settings interfere with the new configuration.
- **Dependency Handling**: Explicit installation of `mysql-community-*` packages guarantees all components use the same version.

---

### **Notes**
- **Backup**: The script backs up `/etc/my.cnf` to avoid losing existing configurations.
- **Major Version**: The script uses the major version (e.g., `8.0`) to enable the correct repository (e.g., `mysql80-community`), ensuring compatibility with MySQL’s package structure.

This script should resolve issues where MySQL ignores the custom data directory due to configuration precedence. Let me know if you need further adjustments!
