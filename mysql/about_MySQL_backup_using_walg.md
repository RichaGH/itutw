## Вопрос
can i backup mysql 8.0 database using wal-g to s3? Need Example of config wal-g, mysql , requiments.

## Ответ

Yes, you can back up a **MySQL 8.0** database to **Amazon S3** using **WAL-G**, a popular open-source tool designed for efficient backup and restoration of databases. WAL-G supports MySQL and is optimized for cloud storage like S3. Below, I’ll provide a step-by-step guide on how to configure WAL-G for MySQL 8.0, including requirements, setup, and an example configuration.

---

### **Requirements**
Before using WAL-G with MySQL 8.0, ensure the following:

1. **MySQL 8.0**:
   - MySQL 8.0 must be installed and running.
   - Binary logging must be enabled in your MySQL configuration (`my.cnf` or `my.ini`):
     ```ini
     [mysqld]
     log_bin = /var/log/mysql/mysql-bin.log
     server_id = 1
     ```

2. **WAL-G**:
   - Download and install WAL-G from the [official GitHub repository](https://github.com/wal-g/wal-g).

3. **Amazon S3**:
   - An S3 bucket to store backups.
   - AWS credentials (Access Key and Secret Key) with permissions to read/write to the bucket.

4. **System Requirements**:
   - Linux-based system (WAL-G is not officially supported on Windows).
   - Sufficient disk space for temporary backup files.

---

### **Install WAL-G**
1. **Download WAL-G**:
   - Download the latest release of WAL-G for Linux from the [releases page](https://github.com/wal-g/wal-g/releases).
   - Example:
     ```bash
     wget https://github.com/wal-g/wal-g/releases/download/v2.0.1/wal-g.linux-amd64.tar.gz
     ```

2. **Extract and Install**:
   - Extract the downloaded file and move the `wal-g` binary to `/usr/local/bin`:
     ```bash
     tar -xzf wal-g.linux-amd64.tar.gz
     sudo mv wal-g /usr/local/bin/
     ```

3. **Verify Installation**:
   - Check if WAL-G is installed correctly:
     ```bash
     wal-g --version
     ```

---

### **Configure WAL-G for MySQL**
1. **Set Up AWS Credentials**:
   - Export your AWS credentials as environment variables:
     ```bash
     export AWS_ACCESS_KEY_ID="your-access-key"
     export AWS_SECRET_ACCESS_KEY="your-secret-key"
     export AWS_REGION="your-region"
     ```

2. **Configure WAL-G**:
   - Create a configuration file for WAL-G (e.g., `/etc/wal-g/config.yaml`):
     ```yaml
     WALG_S3_PREFIX: s3://your-bucket-name/mysql-backups
     WALG_MYSQL_DATASOURCE_NAME: username:password@tcp(localhost:3306)/
     WALG_COMPRESSION_METHOD: brotli
     WALG_DELTA_MAX_STEPS: 5
     WALG_UPLOAD_CONCURRENCY: 10
     ```

   - Replace `your-bucket-name`, `username`, and `password` with your S3 bucket name and MySQL credentials.

3. **Set Environment Variables**:
   - Export the configuration file path:
     ```bash
     export WALG_CONFIG_FILE=/etc/wal-g/config.yaml
     ```

---

### **Backup MySQL to S3 Using WAL-G**
1. **Create a Full Backup**:
   - Run the following command to create a full backup and upload it to S3:
     ```bash
     wal-g backup-push /var/lib/mysql
     ```

   - Replace `/var/lib/mysql` with the path to your MySQL data directory.

2. **Verify the Backup**:
   - List all backups stored in S3:
     ```bash
     wal-g backup-list
     ```

---

### **Restore MySQL from S3 Using WAL-G**
1. **Download the Backup**:
   - To restore a specific backup, use the following command:
     ```bash
     wal-g backup-fetch /path/to/restore backup_name
     ```

   - Replace `backup_name` with the name of the backup (from `wal-g backup-list`).

2. **Prepare the Backup**:
   - Stop the MySQL server:
     ```bash
     sudo systemctl stop mysql
     ```

   - Replace the MySQL data directory with the restored backup:
     ```bash
     rsync -av --delete /path/to/restore/ /var/lib/mysql/
     ```

   - Start the MySQL server:
     ```bash
     sudo systemctl start mysql
     ```

---

### **Automate Backups**
You can automate backups using a cron job. For example, to back up daily at 2 AM:

1. **Create a Backup Script**:
   - Save the following script as `/usr/local/bin/mysql-backup.sh`:
     ```bash
     #!/bin/bash
     export AWS_ACCESS_KEY_ID="your-access-key"
     export AWS_SECRET_ACCESS_KEY="your-secret-key"
     export AWS_REGION="your-region"
     export WALG_CONFIG_FILE=/etc/wal-g/config.yaml
     wal-g backup-push /var/lib/mysql
     ```

2. **Make the Script Executable**:
   ```bash
   chmod +x /usr/local/bin/mysql-backup.sh
   ```

3. **Add a Cron Job**:
   - Open the crontab editor:
     ```bash
     crontab -e
     ```
   - Add the following line:
     ```bash
     0 2 * * * /usr/local/bin/mysql-backup.sh
     ```

---

### **Best Practices**
1. **Encryption**: Use WAL-G’s encryption feature to secure your backups.
2. **Retention Policy**: Configure WAL-G to delete old backups automatically:
   ```bash
   wal-g delete retain 7 --confirm
   ```
   This keeps the last 7 backups and deletes older ones.

3. **Monitoring**: Monitor backup jobs and S3 uploads to ensure they complete successfully.

---

By following these steps, you can back up your MySQL 8.0 database to S3 using WAL-G. Let me know if you need further assistance!
