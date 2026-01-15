# Вопрос

Ты администратор с 20 годами опыта. Твоя задача написать скрипт для инвентаризации версий операционной системы на всех виртуальных машинах в подсети (диапазон адресов должен быть параметром) . 
Семейство операционных систем - линукс. Доступные языки python. Результаты размещать в csv файл колонки:
(имя виртуальной машины , ip адрес, семейство операционной системы,  версия )
Ты также должен объяснить работу скрипта молодому специалисту.


You are an administrator with 20 years of experience. Your task is to write a script for an inventory of operating system versions on all virtual machines in the subnet (the address range must be a parameter) . 
The family of operating systems is Linux. Available python languages. Place the results in a csv column file:
(VM name , ip address, operating system family, version )
You should also explain the script to a young specialist.

# Рассуждения

We are going to write a script that inventories Linux VMs in a given subnet.
Steps:

We'll take the subnet as a parameter (e.g., "192.168.1.0/24").

We'll scan the subnet to find active hosts (using a method like ping sweep or checking open ports, typically SSH for Linux).

For each active host, we'll attempt to SSH and run a command to get the OS version.

We'll parse the output to get the OS family and version.

We'll write the results to a CSV file.

However, note that we are focusing on Linux, so we assume SSH is available (port 22). We'll need to handle authentication.
Since we are administrators, we might have a common SSH key or password. But in practice, we might have a variety of setups.

Assumptions:

We have a common SSH user and key (or password) that works on all VMs. Alternatively, we can use an SSH agent or expect the user to have set up keys.

We'll use paramiko for SSH in Python.

Alternatively, we might use nmap to scan and then use paramiko to connect.

However, to keep it simple and if we are in a controlled environment, we can try to connect to each IP on port 22 and then authenticate.

But note: the task says we are an administrator, so we assume we have credentials.

Let's design:

We'll take three parameters: subnet, username, and either password or key file path.

Steps:

Install required packages: paramiko, csv, ipaddress, concurrent.futures (for parallel scanning if desired).

We'll generate all IPs in the subnet.

We'll try to connect to each IP on port 22 (with a timeout) to see if it's open.

For hosts with open port 22, we'll attempt to SSH with provided credentials.

Once logged in, we can run commands like cat /etc/os-release  to get the OS info.

Parse the output for NAME and VERSION.

Write to CSV.

However, note: The task is to inventory VMs. But what if there are non-VM Linux machines? We are not distinguishing, so we'll just list all Linux machines.

Alternatively, we might use nmap's OS detection, but that might not be as accurate and requires root. So we'll stick with SSH.

Let's write the script accordingly.

We'll structure the script with functions:

scan_subnet(subnet): returns list of IPs that are up (or at least have port 22 open).

get_os_info(ip, username, password=None, key_filename=None): returns (hostname, os_family, os_version) or None if fails.

We'll use paramiko for SSH. We'll handle authentication by either password or key.

Let's code accordingly.

However, note: We might have a lot of machines, so we can use threading for the SSH connections.

We'll write to a CSV file.

Let's create the script.

Important: We must handle various Linux distributions and their way of presenting /etc/os-release.

Example of /etc/os-release:

NAME="Ubuntu"
VERSION="20.04.1 LTS (Focal Fossa)"

We'll extract NAME and VERSION.

Alternatively, we can use lsb_release -a but not all Linux have LSB. /etc/os-release is more standard.

Let's code accordingly.

# Ответ

# Linux VM Inventory Script

## Script: `linux_vm_inventory.py`

```python
#!/usr/bin/env python3
"""
Linux VM Inventory Scanner
Author: Senior Administrator
Description: Scans a subnet for Linux VMs and collects OS version information
"""

import sys
import csv
import subprocess
import ipaddress
import socket
import paramiko
import concurrent.futures
from datetime import datetime
from typing import List, Dict, Optional, Tuple
import argparse

class VMInventoryScanner:
    def __init__(self, subnet: str, ssh_user: str = None, ssh_key_path: str = None):
        """
        Initialize the scanner with subnet and SSH credentials
        
        Args:
            subnet: CIDR notation subnet (e.g., '192.168.1.0/24')
            ssh_user: SSH username (optional - will try common usernames)
            ssh_key_path: Path to SSH private key (optional)
        """
        self.subnet = subnet
        self.ssh_user = ssh_user
        self.ssh_key_path = ssh_key_path
        self.results = []
        
        # Common Linux usernames to try if none provided
        self.common_usernames = ['ubuntu', 'admin', 'root', 'ec2-user', 'centos', 'debian']
        
        # SSH connection parameters
        self.ssh_timeout = 5
        self.max_workers = 10  # Maximum concurrent SSH connections
        
        # Validate subnet
        try:
            self.network = ipaddress.ip_network(subnet, strict=False)
        except ValueError as e:
            print(f"Invalid subnet: {e}")
            sys.exit(1)
    
    def check_ssh_port(self, ip: str) -> bool:
        """
        Check if SSH port (22) is open on the target IP
        
        Args:
            ip: IP address to check
            
        Returns:
            bool: True if SSH port is open
        """
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(1)
            result = sock.connect_ex((str(ip), 22))
            sock.close()
            return result == 0
        except:
            return False
    
    def get_os_info_ssh(self, ip: str) -> Optional[Dict]:
        """
        Connect via SSH and gather OS information
        
        Args:
            ip: IP address to connect to
            
        Returns:
            Dict containing VM info or None if failed
        """
        client = paramiko.SSHClient()
        client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        
        # Try provided username or common usernames
        usernames = [self.ssh_user] if self.ssh_user else self.common_usernames
        
        for username in usernames:
            try:
                if self.ssh_key_path:
                    # Try key-based authentication
                    key = paramiko.RSAKey.from_private_key_file(self.ssh_key_path)
                    client.connect(str(ip), username=username, pkey=key, 
                                 timeout=self.ssh_timeout, banner_timeout=10)
                else:
                    # Try passwordless SSH (works if keys are configured in SSH agent)
                    client.connect(str(ip), username=username, 
                                 timeout=self.ssh_timeout, banner_timeout=10)
                
                # Get hostname
                stdin, stdout, stderr = client.exec_command('hostname')
                hostname = stdout.read().decode().strip()
                
                # Try to get OS info from /etc/os-release (standard for modern Linux)
                stdin, stdout, stderr = client.exec_command('cat /etc/os-release 2>/dev/null || cat /etc/redhat-release 2>/dev/null || echo "Unknown"')
                os_info_raw = stdout.read().decode()
                
                # Parse OS info
                os_family = "Linux"
                os_version = "Unknown"
                
                if 'PRETTY_NAME' in os_info_raw:
                    # Parse /etc/os-release format
                    for line in os_info_raw.split('\n'):
                        if line.startswith('PRETTY_NAME='):
                            os_version = line.split('=')[1].strip('"')
                            break
                        elif line.startswith('VERSION_ID='):
                            version_id = line.split('=')[1].strip('"')
                            if line.startswith('NAME='):
                                os_name = line.split('=')[1].strip('"')
                                os_version = f"{os_name} {version_id}"
                elif 'release' in os_info_raw.lower():
                    # Parse RedHat-style release file
                    os_version = os_info_raw.strip()
                
                # Alternative: try lsb_release command
                if os_version == "Unknown":
                    stdin, stdout, stderr = client.exec_command('lsb_release -d 2>/dev/null || echo "Unknown"')
                    lsb_info = stdout.read().decode()
                    if 'Description:' in lsb_info:
                        os_version = lsb_info.split('Description:')[1].strip()
                
                client.close()
                
                return {
                    'vm_name': hostname,
                    'ip_address': str(ip),
                    'os_family': os_family,
                    'os_version': os_version
                }
                
            except (paramiko.AuthenticationException, paramiko.SSHException, 
                    socket.timeout, ConnectionRefusedError):
                continue
            except Exception as e:
                print(f"Error connecting to {ip}: {e}")
                continue
        
        return None
    
    def scan_single_ip(self, ip) -> Optional[Dict]:
        """
        Scan a single IP address
        
        Args:
            ip: IP address to scan
            
        Returns:
            Dict with VM info or None
        """
        print(f"Scanning {ip}...", end='', flush=True)
        
        if not self.check_ssh_port(ip):
            print(" [SSH closed]")
            return None
        
        result = self.get_os_info_ssh(ip)
        if result:
            print(f" [Found: {result['vm_name']} - {result['os_version']}]")
            return result
        else:
            print(" [No SSH access]")
            return None
    
    def scan_subnet(self) -> List[Dict]:
        """
        Scan all IPs in the subnet
        
        Returns:
            List of dictionaries containing VM information
        """
        print(f"Starting scan of subnet: {self.subnet}")
        print(f"Total IPs to scan: {self.network.num_addresses}")
        print("-" * 50)
        
        # Use ThreadPoolExecutor for concurrent scanning
        with concurrent.futures.ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            # Submit all IPs for scanning
            future_to_ip = {
                executor.submit(self.scan_single_ip, ip): ip 
                for ip in self.network.hosts()
            }
            
            # Collect results as they complete
            for future in concurrent.futures.as_completed(future_to_ip):
                result = future.result()
                if result:
                    self.results.append(result)
        
        print("-" * 50)
        print(f"Scan completed. Found {len(self.results)} Linux VMs.")
        return self.results
    
    def export_to_csv(self, filename: str = None):
        """
        Export results to CSV file
        
        Args:
            filename: Output CSV filename (optional)
        """
        if not filename:
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            filename = f"vm_inventory_{timestamp}.csv"
        
        if not self.results:
            print("No results to export.")
            return
        
        try:
            with open(filename, 'w', newline='') as csvfile:
                fieldnames = ['vm_name', 'ip_address', 'os_family', 'os_version']
                writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
                
                writer.writeheader()
                for result in self.results:
                    writer.writerow(result)
            
            print(f"Results exported to: {filename}")
        except Exception as e:
            print(f"Error exporting to CSV: {e}")

def main():
    """Main function to parse arguments and run scanner"""
    parser = argparse.ArgumentParser(
        description='Linux VM Inventory Scanner',
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  %(prog)s 192.168.1.0/24
  %(prog)s 10.0.0.0/16 --user admin --key ~/.ssh/id_rsa
  %(prog)s 172.16.0.0/24 --output my_inventory.csv
        """
    )
    
    parser.add_argument('subnet', help='Subnet in CIDR notation (e.g., 192.168.1.0/24)')
    parser.add_argument('--user', '-u', help='SSH username (optional)')
    parser.add_argument('--key', '-k', help='Path to SSH private key (optional)')
    parser.add_argument('--output', '-o', help='Output CSV filename (optional)')
    parser.add_argument('--workers', '-w', type=int, default=10, 
                       help='Maximum concurrent workers (default: 10)')
    
    args = parser.parse_args()
    
    # Create scanner instance
    scanner = VMInventoryScanner(
        subnet=args.subnet,
        ssh_user=args.user,
        ssh_key_path=args.key
    )
    
    # Set max workers
    scanner.max_workers = args.workers
    
    # Run scan
    try:
        scanner.scan_subnet()
        
        # Export results
        output_file = args.output or f"vm_inventory_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
        scanner.export_to_csv(output_file)
        
        # Print summary
        if scanner.results:
            print("\n=== SCAN SUMMARY ===")
            for i, vm in enumerate(scanner.results, 1):
                print(f"{i:3}. {vm['vm_name']:20} {vm['ip_address']:15} {vm['os_version']}")
        
    except KeyboardInterrupt:
        print("\n\nScan interrupted by user.")
        if scanner.results:
            scanner.export_to_csv("partial_inventory.csv")
            print("Partial results saved to partial_inventory.csv")
        sys.exit(1)
    except Exception as e:
        print(f"Unexpected error: {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

## Requirements File: `requirements.txt`

```txt
paramiko>=2.11.0
```

## Installation and Usage Instructions

### 1. **Installation**

```bash
# Install required Python packages
pip install -r requirements.txt

# Or install directly
pip install paramiko
```

### 2. **Basic Usage**

```bash
# Scan a subnet (will try common usernames)
python linux_vm_inventory.py 192.168.1.0/24

# Specify SSH username
python linux_vm_inventory.py 10.0.0.0/16 --user admin

# Use SSH key authentication
python linux_vm_inventory.py 172.16.0.0/24 --user ubuntu --key ~/.ssh/id_rsa

# Specify output file
python linux_vm_inventory.py 192.168.1.0/24 --output inventory.csv

# Control concurrent connections
python linux_vm_inventory.py 192.168.1.0/24 --workers 20
```

### 3. **Using SSH Agent (Recommended)**

```bash
# Start SSH agent if not running
eval "$(ssh-agent -s)"

# Add your SSH key
ssh-add ~/.ssh/id_rsa

# Now run the script without --key parameter
python linux_vm_inventory.py 192.168.1.0/24 --user admin
```

## Explanation for Young Specialist

### **What This Script Does:**

1. **Subnet Scanning**: Takes a subnet (like `192.168.1.0/24`) and checks all IP addresses in that range.

2. **SSH Port Detection**: For each IP, it checks if port 22 (SSH) is open using a TCP connection test.

3. **OS Information Gathering**: For hosts with open SSH ports, it attempts to connect and:
   - Reads `/etc/os-release` (standard on modern Linux)
   - Falls back to `/etc/redhat-release` for older RedHat systems
   - Uses `lsb_release` command as another alternative
   - Extracts the hostname using the `hostname` command

4. **Concurrent Processing**: Uses multiple threads to scan IPs simultaneously, making it much faster than sequential scanning.

5. **CSV Export**: Saves results in a structured CSV file with columns: `vm_name`, `ip_address`, `os_family`, `os_version`.

### **Key Components Explained:**

#### **1. Subnet Handling**
```python
# Converts string like "192.168.1.0/24" to network object
network = ipaddress.ip_network("192.168.1.0/24")
for ip in network.hosts():  # Iterates through all usable IPs
    print(ip)
```

#### **2. SSH Connection**
```python
# Uses Paramiko library for SSH
client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(ip, username='admin', timeout=5)
```

#### **3. OS Detection Logic**
The script tries multiple methods to get OS information:
- **Primary**: `/etc/os-release` (contains PRETTY_NAME, VERSION_ID)
- **Fallback 1**: `/etc/redhat-release` (older RHEL/CentOS)
- **Fallback 2**: `lsb_release -d` command

#### **4. Thread Pool for Performance**
```python
# Scans multiple IPs simultaneously
with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
    for future in concurrent.futures.as_completed(future_to_ip):
        result = future.result()
```

### **Security Considerations:**

1. **SSH Keys**: The script supports SSH key authentication, which is more secure than passwords.

2. **Host Key Verification**: Uses `AutoAddPolicy()` for convenience in internal networks. For production:
   ```python
   client.load_system_host_keys()  # Use known hosts file instead
   ```

3. **Credentials**: Never hardcode credentials in the script. Use SSH agent or prompt for passwords.

### **Error Handling:**

The script includes try-except blocks for:
- Network timeouts
- Authentication failures
- Connection refused errors
- Invalid IP addresses
- File permission issues

### **Sample Output CSV:**
```csv
vm_name,ip_address,os_family,os_version
webserver01,192.168.1.10,Linux,Ubuntu 20.04.4 LTS
dbserver01,192.168.1.11,Linux,CentOS Linux 7 (Core)
appserver01,192.168.1.12,Linux,Debian GNU/Linux 10 (buster)
```

### **Best Practices for Young Specialists:**

1. **Test First**: Always test on a small subnet or specific IPs first.

2. **Network Impact**: Concurrent scanning can generate significant network traffic. Use appropriate worker counts.

3. **Authentication**: Set up SSH keys on target VMs beforehand for smooth operation.

4. **Permissions**: Ensure you have appropriate network access rights to scan the subnet.

5. **Logging**: Consider adding logging for audit trails in production environments.

6. **Rate Limiting**: For large networks, add delays between scans to avoid overwhelming devices.

7. **Validation**: Always verify a sample of results manually to ensure accuracy.

### **Common Issues and Solutions:**

1. **Permission Denied**: Ensure SSH keys are properly configured on target VMs.

2. **Firewall Blocking**: Check if firewalls allow SSH connections from your scanning host.

3. **Slow Scanning**: Reduce concurrent workers or increase timeouts.

4. **Missing OS Info**: Some minimal Linux installations might not have standard OS files.

This script provides a solid foundation for VM inventory management and can be extended with additional features like checking installed packages, disk usage, or running services.
