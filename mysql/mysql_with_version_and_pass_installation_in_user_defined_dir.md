## Вопросы 
1) you are a database engineer with 20 years of experience. Your task is to write a bash script for centos 8 stream to install a specific version of mysql, the version must be set in the parameters.
2) yoy must install is to userdefined directory
3)Also you must set up new root password

## Ответ

Got it! Below is the updated bash script that includes a **third parameter** for setting up a new root password for MySQL. The script will configure MySQL to use the specified data directory and set the new root password during the installation process.

```bash
#!/bin/bash

# Check if the MySQL version, data directory, and root password are provided as parameters
if [ -z "$1" ] || [ -z "$2" ] || [ -z "$3" ]; then
  echo "Usage: $0 <mysql-version> <data-directory> <root-password>"
  echo "Example: $0 8.0.26 /mnt/mysql-data MyNewSecurePassword123"
  exit 1
fi

MYSQL_VERSION=$1
MYSQL_DATA_DIR=$2
MYSQL_ROOT_PASSWORD=$3

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

# Start MySQL service
echo "Starting MySQL service..."
systemctl start mysqld

# Extract the temporary root password from the log file
MYSQL_TEMP_PASSWORD=$(grep 'temporary password' $MYSQL_DATA_DIR/mysqld.log | awk '{print $NF}')

# Change the root password
echo "Changing MySQL root password..."
mysql --user=root --password="$MYSQL_TEMP_PASSWORD" --connect-expired-password --execute="ALTER USER 'root'@'localhost' IDENTIFIED BY '$MYSQL_ROOT_PASSWORD';"

# Secure MySQL installation
echo "Securing MySQL installation..."
mysql_secure_installation --password="$MYSQL_ROOT_PASSWORD" <<EOF
n
y
y
y
y
EOF

# Enable MySQL service to start on boot
echo "Enabling MySQL service to start on boot..."
systemctl enable mysqld

echo "MySQL version $MYSQL_VERSION has been successfully installed and configured:"
echo "- Data directory: $MYSQL_DATA_DIR"
echo "- Root password: $MYSQL_ROOT_PASSWORD"
```

### How to Use the Script:

1. **Save the script** to a file, e.g., `install_mysql_custom_dir_password.sh`.
2. **Make the script executable**:
   ```bash
   chmod +x install_mysql_custom_dir_password.sh
   ```
3. **Run the script** with the desired MySQL version, custom data directory, and root password as parameters:
   ```bash
   ./install_mysql_custom_dir_password.sh 8.0.26 /mnt/mysql-data MyNewSecurePassword123
   ```

### Explanation of Changes:

1. **Third Parameter**: The script now takes a third parameter (`$3`) for the new MySQL root password.
2. **Change Root Password**:
   - After initializing MySQL, the script extracts the temporary root password from the log file.
   - It uses the `mysql` command to change the root password to the one provided in the parameter.
3. **Secure Installation**:
   - The script runs `mysql_secure_installation` non-interactively using a heredoc (`EOF`) to automate the process.
   - It ensures that anonymous users, remote root login, and test databases are removed.

### Notes:
- Ensure that the root password provided is strong and secure.
- The script assumes that the MySQL temporary password is stored in the log file located in the custom data directory (`$MYSQL_DATA_DIR/mysqld.log`).
- Test the script in a safe environment before using it in production.

This script should now fully automate the installation of MySQL with a custom data directory and a new root password on CentOS 8 Stream. Let me know if you need further assistance!
