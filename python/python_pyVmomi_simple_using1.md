# Вопрос

You are an administrator with 20 years of experience. Your task is to figure out how the pyVmomi module works.  And use it to write a script that will receive VMware (vSphere) data using the pyVmomi module. Write a minimal script to get VM data (Name, ip adderss) from vSphere.

# Рассуждения

We are going to use the pyVmomi module to connect to a vSphere instance and retrieve VM data (Name and IP address).
Steps:

Install pyVmomi if not already installed (pip install pyVmomi)

Import necessary modules.

Define connection parameters (server, user, password, port).

Create a connection to the vSphere server.

Retrieve the VMs and their properties (Name and IP address).

Print the data.

Note: We are going to write a minimal script that does the following:

Connects to the vCenter or ESXi host.

Gets the list of VMs.

For each VM, get the name and the primary IP address.

Important: We are going to use the ServiceInstance to connect and then use the property collector to get the VM data.

We assume that the vSphere server is using the default port (443) and that we are using the standard vSphere API.

Let's write the script accordingly.

# Ответ

I'll help you create a minimal script to retrieve VM data from vSphere using pyVmomi. First, let me explain how pyVmomi works:

## How pyVmomi Works

pyVmomi is the Python SDK for VMware vSphere API (vSphere Web Services). It:
1. Provides Python bindings to vSphere API
2. Uses SOAP/XML for communication with vCenter/ESXi
3. Follows object-oriented patterns that map to vSphere managed objects

## Minimal Script to Get VM Data

```python
#!/usr/bin/env python3
"""
Minimal pyVmomi script to retrieve VM data from vSphere
"""

from pyVmomi import vim
from pyVim.connect import SmartConnect, Disconnect
import ssl
import sys

def get_vm_info(content, vm_name=None):
    """Retrieve VM information from vSphere"""
    
    # Get all VMs from inventory
    container = content.viewManager.CreateContainerView(
        content.rootFolder, [vim.VirtualMachine], True
    )
    
    vms = []
    for vm in container.view:
        # If vm_name is specified, filter by name
        if vm_name and vm.name != vm_name:
            continue
            
        vm_data = {
            'name': vm.name,
            'ip_address': None
        }
        
        # Get IP address if VM is powered on
        if vm.guest and vm.guest.ipAddress:
            vm_data['ip_address'] = vm.guest.ipAddress
        elif vm.guest and vm.guest.net:
            # Try to get IP from network interfaces
            for net in vm.guest.net:
                if net.ipConfig and net.ipConfig.ipAddress:
                    for ip in net.ipConfig.ipAddress:
                        if ip.ipAddress:
                            vm_data['ip_address'] = ip.ipAddress
                            break
                if vm_data['ip_address']:
                    break
        
        vms.append(vm_data)
    
    return vms

def main():
    # Connection parameters - EDIT THESE
    VCENTER_HOST = 'vcenter.example.com'
    VCENTER_USER = 'administrator@vsphere.local'
    VCENTER_PASSWORD = 'password'
    
    # Disable SSL verification (use for self-signed certs)
    context = ssl.SSLContext(ssl.PROTOCOL_TLS)
    context.verify_mode = ssl.CERT_NONE
    
    try:
        # Connect to vCenter
        print(f"Connecting to vCenter at {VCENTER_HOST}...")
        si = SmartConnect(
            host=VCENTER_HOST,
            user=VCENTER_USER,
            pwd=VCENTER_PASSWORD,
            sslContext=context
        )
        
        # Get service content
        content = si.RetrieveContent()
        print("Connected successfully!\n")
        
        # Get VM information
        vms = get_vm_info(content)
        
        # Display results
        print(f"Found {len(vms)} VMs:")
        print("-" * 50)
        print(f"{'VM Name':<30} {'IP Address':<20}")
        print("-" * 50)
        
        for vm in vms:
            print(f"{vm['name']:<30} {vm['ip_address'] or 'N/A':<20}")
        
        # Disconnect
        Disconnect(si)
        print("\nDisconnected from vCenter")
        
    except Exception as e:
        print(f"Error: {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

## Requirements File

Create a `requirements.txt` file:
```txt
pyvmomi>=7.0
```

## Installation and Usage

1. **Install dependencies:**
```bash
pip install -r requirements.txt
```

2. **Run the script:**
```bash
python vsphere_vm_info.py
```

## Enhanced Version with More Features

Here's a more robust version with additional features:

```python
#!/usr/bin/env python3
"""
Enhanced pyVmomi script with more options and error handling
"""

from pyVmomi import vim
from pyVim.connect import SmartConnect, Disconnect
import ssl
import argparse
import sys
from getpass import getpass

class VSphereClient:
    def __init__(self, host, user, password, port=443, disable_ssl_verify=True):
        self.host = host
        self.user = user
        self.password = password
        self.port = port
        self.si = None
        
        # Handle SSL context
        context = None
        if disable_ssl_verify:
            context = ssl.SSLContext(ssl.PROTOCOL_TLS)
            context.verify_mode = ssl.CERT_NONE
            
        self.ssl_context = context
    
    def connect(self):
        """Establish connection to vSphere"""
        try:
            self.si = SmartConnect(
                host=self.host,
                user=self.user,
                pwd=self.password,
                port=self.port,
                sslContext=self.ssl_context
            )
            return True
        except Exception as e:
            print(f"Connection failed: {e}")
            return False
    
    def disconnect(self):
        """Disconnect from vSphere"""
        if self.si:
            Disconnect(self.si)
            self.si = None
    
    def get_all_vms(self, filter_powered_on=False):
        """Retrieve all VMs with optional filtering"""
        if not self.si:
            raise RuntimeError("Not connected to vSphere")
        
        content = self.si.RetrieveContent()
        container = content.viewManager.CreateContainerView(
            content.rootFolder, [vim.VirtualMachine], True
        )
        
        vms_data = []
        for vm in container.view:
            # Filter by power state if requested
            if filter_powered_on and vm.runtime.powerState != vim.VirtualMachinePowerState.poweredOn:
                continue
            
            vm_info = {
                'name': vm.name,
                'power_state': vm.runtime.powerState,
                'ip_address': None,
                'host_name': vm.guest.hostName if vm.guest else None,
                'os': vm.guest.guestFullName if vm.guest else None,
                'num_cpu': vm.config.hardware.numCPU,
                'memory_mb': vm.config.hardware.memoryMB
            }
            
            # Get IP address
            if vm.guest and vm.guest.ipAddress:
                vm_info['ip_address'] = vm.guest.ipAddress
            
            vms_data.append(vm_info)
        
        return vms_data

def main():
    parser = argparse.ArgumentParser(description='Retrieve VM information from vSphere')
    parser.add_argument('--host', required=True, help='vCenter/ESXi hostname or IP')
    parser.add_argument('--user', required=True, help='Username')
    parser.add_argument('--password', help='Password (optional, will prompt if not provided)')
    parser.add_argument('--powered-on', action='store_true', help='Show only powered-on VMs')
    parser.add_argument('--output', choices=['simple', 'detailed'], default='simple', 
                       help='Output format')
    
    args = parser.parse_args()
    
    # Get password if not provided
    password = args.password
    if not password:
        password = getpass(f"Password for {args.user}: ")
    
    # Create client and connect
    client = VSphereClient(
        host=args.host,
        user=args.user,
        password=password
    )
    
    if not client.connect():
        sys.exit(1)
    
    try:
        # Get VM data
        vms = client.get_all_vms(filter_powered_on=args.powered_on)
        
        # Display results
        if args.output == 'simple':
            print(f"\n{'VM Name':<30} {'IP Address':<20} {'Power State':<15}")
            print("-" * 70)
            for vm in vms:
                power_state = str(vm['power_state']).split('.')[-1]
                print(f"{vm['name']:<30} {vm['ip_address'] or 'N/A':<20} {power_state:<15}")
        else:
            for vm in vms:
                print(f"\n{'='*50}")
                print(f"VM Name: {vm['name']}")
                print(f"IP Address: {vm['ip_address'] or 'N/A'}")
                print(f"Hostname: {vm['host_name'] or 'N/A'}")
                print(f"OS: {vm['os'] or 'N/A'}")
                print(f"Power State: {vm['power_state']}")
                print(f"vCPUs: {vm['num_cpu']}")
                print(f"Memory: {vm['memory_mb']} MB")
        
        print(f"\nTotal VMs found: {len(vms)}")
        
    finally:
        client.disconnect()

if __name__ == "__main__":
    main()
```

## Key Concepts for Administrators

1. **Connection Management**: Always disconnect after operations
2. **SSL Handling**: Self-signed certs require disabling SSL verification
3. **Object Hierarchy**: vCenter has a folder/datacenter/cluster/host/VM hierarchy
4. **Property Collector**: Efficient way to get multiple properties
5. **Guest Info**: IP addresses come from VMware Tools in the guest OS

## Usage Examples

```bash
# Simple usage
python vsphere_vm_info.py --host vcenter.company.com --user admin@vsphere.local

# Show only powered-on VMs
python vsphere_vm_info.py --host vcenter.company.com --user admin --powered-on

# Detailed output
python vsphere_vm_info.py --host vcenter.company.com --user admin --output detailed
```

This script provides a solid foundation. You can extend it to include more VM properties, handle pagination for large environments, or add CSV/JSON export functionality.
