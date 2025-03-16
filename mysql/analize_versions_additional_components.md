## Вопрос
Also you must verify install part. Are you sure about version? may by exists additional related components of mysql <version> what must be installed with <version number parameter> for normal work after installation? (for example mysql-comunity-libs, mysql-comunity-client-plugins, mysql-comunity-client)

## Рассуждения
Okay, let's tackle this. The user is asking me to verify the installation part of the script, especially regarding the version and dependencies. They're concerned that just installing mysql-community-server-$MYSQL_VERSION might not be enough. They mentioned components like mysql-community-libs, mysql-community-client-plugins, and mysql-community-client. I need to check if those are necessary and if the script handles them correctly.

First, I recall that MySQL has several packages that are often required. The server package (mysql-community-server) depends on others like mysql-community-client, mysql-community-libs, and mysql-community-common. These dependencies are usually pulled in automatically when installing the server via DNF because of the package manager's dependency resolution. But maybe in some cases, like if the repository setup is incomplete, they might not be installed.

The original script uses dnf install -y mysql-community-server-$MYSQL_VERSION. By specifying the exact version, it might be forcing the installation of only that package without its dependencies. Wait, no, DNF should still resolve dependencies. But maybe in some scenarios, if the version isn't properly tracked in the repo, it could miss them. For example, if the version specified is not in the enabled repository, DNF might not find the dependencies.

Wait, the script enables the specific version community repo with dnf config-manager --enable mysql${MYSQL_VERSION}-community. But how is that line working? Let's say the version is 8.0.26. The repo name would be mysql8.0.26-community? That doesn't seem right. Oh, wait, MySQL's Yum repositories typically have release packages that contain multiple versions, and you enable the repository for a major version, not the exact point release. For example, mysql80-community would cover all 8.0.x versions. So the way the script is enabling the repo by appending the version to "mysql" might be incorrect.

Wait, the initial script adds the mysql80-community-release-el8-1.noarch.rpm, which sets up the repositories for MySQL 8.0. Then, using dnf config-manager --enable mysql${MYSQL_VERSION}-community. But if MYSQL_VERSION is 8.0.26, that would translate to mysql8.0.26-community, which doesn't exist. The repositories are usually named by major version, like mysql80-community, not including the minor or patch versions. So this is a mistake. The repo enabling part is incorrect because MySQL repositories are structured by major versions, not exact point releases. Therefore, enabling a repo for a specific minor version like 8.0.26 won't work; the repo name should be mysql80-community for all 8.0.x versions.

That's a critical error. The script as written would fail to enable the correct repository because it's trying to enable a repo that doesn't exist. For example, if the user passes 8.0.26, the command would be dnf config-manager --enable mysql8.0.26-community, but the actual repo is mysql80-community. So the repo enabling part is wrong, leading to installation failures because the correct packages aren't available.

Additionally, when installing mysql-community-server, DNF should automatically install dependencies like mysql-community-client, mysql-community-libs, etc. But if the repo setup is incorrect, those dependencies might not be found, leading to installation errors. So the primary issue is the incorrect repository configuration.

Another point: the user is asking if the script installs all necessary components. The current script only installs mysql-community-server-$MYSQL_VERSION. If the version is specified with the exact package version, that might work, but dependencies still need to be compatible. However, when using DNF, specifying the package name with version should still resolve dependencies, as long as the repositories are correctly set up.

But given that the repo enabling part is wrong, the script would fail to find the server package as well. So the main problem is the repository setup. The initial rpm -Uvh installs the mysql80-community-release package, which sets up the repos for MySQL 8.0. Then, the script tries to enable a repo named after the exact version, which doesn't exist. Instead, it should enable the major version repo, like mysql80-community.

So to fix the repo enabling, the script should parse the major version from the provided version parameter. For example, if the user provides 8.0.26, the major version is 8.0, which corresponds to mysql80-community. Therefore, the line should be dnf config-manager --enable mysql80-community, not using the full version string.

But how to extract the major version from the parameter? The user might input 8.0.26, which is 8.0.x. The major version is 8.0, which in MySQL's repo naming becomes 80 (mysql80-community). So the script needs to convert 8.0.26 to 80. That requires parsing the version string. For example, using something like MYSQL_MAJOR_VERSION=$(echo "$MYSQL_VERSION" | awk -F. '{ printf "%d%d", $1, $2 }'). Then enable mysql${MYSQL_MAJOR_VERSION}-community.

But wait, MySQL 5.7 would be 57, 8.0 becomes 80. That makes sense. So the script needs to calculate the major version correctly. If the user inputs 8.0.26, the major version is 8.0, which becomes 80. If they input 5.7.38, it becomes 57.

This is a crucial step that the original script misses. The repo enabling is incorrect, leading to failure in finding the packages.

Additionally, the package names like mysql-community-server-8.0.26 might not exist. Typically, packages are versioned with release numbers in the repo, and the exact package version might have a different naming scheme. For example, the package might be mysql-community-server-8.0.26-1.el8.x86_64. But when installing with DNF, you can specify the version as 8.0.26, and DNF will find the appropriate package. However, this depends on how the repository is structured.

But if the repo is correctly enabled (mysql80-community), then installing mysql-community-server should pull in the latest 8.0.x version. If the user wants a specific minor version, they might need to specify the full package version, which might require that the repository includes that specific version. However, MySQL's repositories typically have the latest version of each major release. So if the user wants a specific older minor version, they might need to use a different repository or download the RPMs directly, which complicates things.

In the original script, the user is supposed to specify the version as a parameter, like 8.0.26. But if the repo only has 8.0.33 (the latest), then installing mysql-community-server-8.0.26 would fail because it's not present. Therefore, the approach of specifying the exact version might not work unless the repository includes all past versions, which it doesn't. MySQL's official repos usually only have the latest version for each major release.

This is another problem. The script assumes that specifying the exact version in the package name will work, but that's only true if that exact version is available in the enabled repository. However, MySQL's repositories typically only have the latest version of each major release. Therefore, the script's approach of allowing any version parameter is flawed unless it's using a repository that contains all historical versions, which isn't standard.

So the user's concern about additional components is valid, but the deeper issue is that the script's method for enabling the repository and specifying the package version is incorrect. The repo enabling is based on the exact version, which is wrong, and the package installation assumes that the exact version exists in the repo, which it may not.

To address this, the script should either:

1. Install the latest version of MySQL from the major version's repo (e.g., 8.0.x) by enabling the correct major version repo and installing mysql-community-server without specifying the version, letting DNF get the latest.

OR

2. If the user needs a specific minor version, the script must ensure that the repository for that version is available and correctly configured, which might not be possible through the standard MySQL repos.

Given that, the script as written would not work for installing specific versions unless those versions are present in the enabled repository. Since MySQL's official repos don't keep old versions, the script's approach is flawed.

Furthermore, regarding dependencies: if the correct repository is enabled and the package is available, DNF should handle installing the required dependencies (like mysql-community-client, mysql-community-libs, etc.) automatically. So the original script's installation command (dnf install -y mysql-community-server-$MYSQL_VERSION) would work only if that exact package version is present, and dependencies are satisfied. But given the repo issues, this is unlikely.

So, to summarize the issues:

1. Incorrect repository enabling: Using the full version (e.g., 8.0.26) to enable a repo, which doesn't exist. Repos are named after major versions (e.g., mysql80-community).

2. Assuming specific package versions are available in the repo, which they aren't unless using a custom repo.

3. Potential missing dependencies if the repo isn't set up correctly, but DNF should handle dependencies if the repo is correct.

To fix this, the script should:

- Determine the major version from the input (e.g., 8.0.26 → 8.0 → mysql80-community).

- Enable the correct major version repo.

- Install the package without specifying the version, allowing DNF to install the latest available in that major version. If the user wants a specific version, this approach won't work, and the script would need to handle downloading specific RPMs or using a different repo.

Alternatively, if the user must install a specific version, the script might need to use MySQL's RPM archives, but that complicates the process and requires checking for the existence of those RPMs.

Therefore, the original script has significant flaws in repository setup and package installation that need to be addressed to ensure correct installation of MySQL and its dependencies

## Ответ

Let’s analyze and improve the script to ensure **correct version installation** and **dependency handling**. Here’s a revised script with critical fixes, followed by an explanation of the changes:

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

MYSQL_MAJOR_VERSION=$1  # e.g., "8.0" (not the full version like 8.0.26)
MYSQL_DATA_DIR=$2
MYSQL_ROOT_PASSWORD=$3

# Add MySQL Yum Repository
echo "Adding MySQL Yum Repository..."
rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el8-1.noarch.rpm

# Enable the correct major version repository (e.g., "8.0" → "mysql80-community")
MYSQL_REPO_ID="mysql$(echo "$MYSQL_MAJOR_VERSION" | tr -d .)-community"
echo "Enabling MySQL repository: $MYSQL_REPO_ID..."
dnf module -y disable mysql
dnf config-manager --enable "$MYSQL_REPO_ID"

# Install MySQL Server and dependencies
echo "Installing MySQL Server and dependencies for major version $MYSQL_MAJOR_VERSION..."
dnf install -y mysql-community-server mysql-community-client mysql-community-libs mysql-community-common

# Create custom data directory
echo "Setting up MySQL data directory at $MYSQL_DATA_DIR..."
mkdir -p "$MYSQL_DATA_DIR"
chown -R mysql:mysql "$MYSQL_DATA_DIR"
chmod 750 "$MYSQL_DATA_DIR"

# Stop MySQL service
systemctl stop mysqld

# Configure MySQL to use the custom data directory
cat > /etc/my.cnf.d/custom-data-dir.cnf <<EOF
[mysqld]
datadir=$MYSQL_DATA_DIR
socket=$MYSQL_DATA_DIR/mysql.sock
log-error=$MYSQL_DATA_DIR/mysqld.log
EOF

# Initialize MySQL
echo "Initializing MySQL..."
mysqld --initialize --user=mysql --datadir="$MYSQL_DATA_DIR"

# Start MySQL
systemctl start mysqld

# Extract temporary password (wait for log file)
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

### **Key Fixes and Improvements**

#### 1. **Correct Repository Handling**
- **Problem**: The original script used the full version (e.g., `8.0.26`) to enable a non-existent repository like `mysql8.0.26-community`.
- **Fix**: Convert the major version (e.g., `8.0`) to a valid repo ID like `mysql80-community`:
  ```bash
  MYSQL_REPO_ID="mysql$(echo "$MYSQL_MAJOR_VERSION" | tr -d .)-community"
  dnf config-manager --enable "$MYSQL_REPO_ID"
  ```
  This ensures compatibility with MySQL’s official repository structure.

#### 2. **Install All Required Dependencies**
- **Problem**: Installing only `mysql-community-server` might miss critical dependencies like `mysql-community-client` or `mysql-community-libs` if the repo is misconfigured.
- **Fix**: Explicitly install all core components:
  ```bash
  dnf install -y mysql-community-server mysql-community-client mysql-community-libs mysql-community-common
  ```
  This guarantees that essential tools (e.g., `mysql` CLI) and libraries are installed.

#### 3. **Log File Configuration**
- **Problem**: The original script assumed logs would appear in `$MYSQL_DATA_DIR/mysqld.log` without configuring it.
- **Fix**: Explicitly set the `log-error` path in `my.cnf`:
  ```ini
  [mysqld]
  log-error=$MYSQL_DATA_DIR/mysqld.log
  ```

#### 4. **Wait for Log File**
- **Problem**: The script might try to read the log file before it exists.
- **Fix**: Add a loop to wait for the log file:
  ```bash
  while [ ! -f "$MYSQL_DATA_DIR/mysqld.log" ]; do sleep 1; done
  ```

#### 5. **Security Improvements**
- Avoid passing passwords via command-line arguments (use `MYSQL_PWD` environment variable instead).
- Use `-i` (case-insensitive) in `grep` to handle variations like "Temporary Password".

---

### **Why This Works**
1. **Repository Alignment**: Enabling the correct major version repository (e.g., `mysql80-community`) ensures access to all packages for that release stream.
2. **Explicit Dependencies**: Installing `mysql-community-server`, `mysql-community-client`, etc., guarantees that all necessary components are present.
3. **Log File Reliability**: Configuring `log-error` in `my.cnf` ensures logs are written to the custom directory, and the `while` loop ensures the script waits for the log file.

---

### **Usage Notes**
- **Parameter Change**: The script now expects the **major version** (e.g., `8.0`) instead of the full version (e.g., `8.0.26`), as repositories are structured by major versions.
- **SELinux**: If SELinux is enforcing, adjust contexts for the custom data directory:
  ```bash
  semanage fcontext -a -t mysqld_db_t "$MYSQL_DATA_DIR(/.*)?"
  restorecon -Rv "$MYSQL_DATA_DIR"
  ```

---

### **Conclusion**
This revised script addresses the flaws in repository configuration, dependency installation, and log file handling. It ensures reliable installation of MySQL and its components on CentOS 8 Stream.
