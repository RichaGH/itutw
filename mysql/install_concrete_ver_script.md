## Вопросы

1)you are a database engineer with 20 years of experience. Your task is to write a bash script for centos 8 stream to install a specific version of mysql, the version must be set in the parameters.
2)You are also must use second parameter using for setup non dafault mysql database dir

## Ответ

Certainly! Below is an updated version of the bash script that takes a second parameter to set up a non-default MySQL data directory. The script will configure MySQL to use the specified directory for storing its data.

```bash
#!/bin/bash

# Check if the MySQL version and data directory are provided as parameters
if [ -z "$1" ] || [ -z "$2" ]; then
  echo "Usage: $0 <mysql-version> <data-directory>"
  echo "Example: $0 8.0.26 /mnt/mysql-data"
  exit 1
fi

MYSQL_VERSION=$1
MYSQL_DATA_DIR=$2

# Add MySQL Yum Repository
echo "Adding MySQL Yum Repository..."
rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el8-1.noarch.rpm

# Enable the specific version of MySQL
echo "Enabling MySQL version $MYSQL_VERSION..."
dnf module -y disable mysql
dnf config-manager --enable mysql${MYSQL_VERSION}-community

# Install MySQL Server
echo "Installing MySQL Server version $MYSQL_VERSION..."
dnf install -y mysql-community-server-$MYSQL_VERSION

# Create the custom data directory if it doesn't exist
echo "Setting up MySQL data directory at $MYSQL_DATA_DIR..."
mkdir -p $MYSQL_DATA_DIR
chown -R mysql:mysql $MYSQL_DATA_DIR
chmod 750 $MYSQL_DATA_DIR

# Stop MySQL service to configure the data directory
echo "Stopping MySQL service..."
systemctl stop mysqld

# Backup the existing data directory (if any)
if [ -d /var/lib/mysql ]; then
  echo "Backing up existing MySQL data directory..."
  mv /var/lib/mysql /var/lib/mysql.bak
fi

# Update MySQL configuration to use the new data directory
echo "Configuring MySQL to use $MYSQL_DATA_DIR..."
cat > /etc/my.cnf.d/custom-data-dir.cnf <<EOF
[mysqld]
datadir=$MYSQL_DATA_DIR
socket=$MYSQL_DATA_DIR/mysql.sock
EOF

# Initialize MySQL with the new data directory
echo "Initializing MySQL with the new data directory..."
mysqld --initialize --user=mysql --datadir=$MYSQL_DATA_DIR

# Start and enable MySQL service
echo "Starting and enabling MySQL service..."
systemctl start mysqld
systemctl enable mysqld

# Secure MySQL installation
echo "Securing MySQL installation..."
MYSQL_TEMP_PASSWORD=$(grep 'temporary password' $MYSQL_DATA_DIR/mysqld.log | awk '{print $NF}')
mysql_secure_installation --password="$MYSQL_TEMP_PASSWORD"

echo "MySQL version $MYSQL_VERSION has been successfully installed and configured to use $MYSQL_DATA_DIR as the data directory."
```

### How to Use the Script:

1. **Save the script** to a file, e.g., `install_mysql_custom_dir.sh`.
2. **Make the script executable**:
   ```bash
   chmod +x install_mysql_custom_dir.sh
   ```
3. **Run the script** with the desired MySQL version and custom data directory as parameters:
   ```bash
   ./install_mysql_custom_dir.sh 8.0.26 /mnt/mysql-data
   ```

### Explanation of Changes:

1. **Second Parameter**: The script now takes a second parameter (`$2`) for the custom MySQL data directory.
2. **Custom Data Directory Setup**:
   - The script creates the specified directory if it doesn't exist.
   - It sets the correct ownership (`mysql:mysql`) and permissions (`750`) for the directory.
3. **MySQL Configuration**:
   - The script stops the MySQL service to reconfigure the data directory.
   - It creates a custom configuration file (`/etc/my.cnf.d/custom-data-dir.cnf`) to point MySQL to the new data directory.
   - It initializes MySQL with the new data directory using `mysqld --initialize`.
4. **Backup**: If the default MySQL data directory (`/var/lib/mysql`) exists, it is backed up to `/var/lib/mysql.bak`.

### Notes:
- Ensure that the specified data directory has sufficient disk space and is on a reliable storage device.
- The script assumes that the MySQL temporary password is stored in the log file located in the custom data directory (`$MYSQL_DATA_DIR/mysqld.log`).
- Test the script in a safe environment before using it in production.

This script should work for setting up MySQL with a custom data directory on CentOS 8 Stream. Let me know if you need further assistance!
