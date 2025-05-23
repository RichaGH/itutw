# Вопрос
You are an IT engineer with 20 years of experience. You need to configure the nfs server on CentOS 8 Stream. You need to write configs for the server and for the client.

# Рассуждения

Okay, let's tackle configuring an NFS server on CentOS 8 Stream. I remember that NFS allows sharing directories over a network, so first, I need to install the necessary packages. The server requires nfs-utils, which I think is the main package. Let me check if that's correct. Oh right, on CentOS 8, nfs-utils is the package to install for both server and client, but the client might just need it as well. So the first step is to install nfs-utils on the server.

Next, enabling and starting the services. The main service for NFS is nfs-server.service. Also, I recall that NFS relies on RPC, so rpcbind is involved. But in newer versions, maybe that's handled by the nfs-server service? I should enable and start nfs-server, and maybe check if rpcbind is running as a dependency. Oh right, in CentOS 8, systemd handles the dependencies, so starting nfs-server should take care of the rest.

Now, configuring the exports. The main configuration file is /etc/exports. I need to decide which directories to share. Let's say I want to share /var/nfs as a common example. So I need to create that directory and set proper permissions. Maybe chown nobody:nobody or some user, but NFS uses the user's UID by default. Alternatively, using all_squash to map all users to anonymous. Hmm, security is a consideration here. If the client machines have trusted users, then maybe no squash, but for a more secure setup, squash them to nobody.

In the /etc/exports file, each line specifies the directory, allowed clients, and options. For example, /var/nfs 192.168.1.0/24(rw,sync,no_subtree_check). Wait, the syntax is space-separated, client network or IP followed by options in parentheses. Also, the sync option is recommended for data consistency. Subtree checking can be disabled if the entire directory is exported, which improves performance.

After editing the exports file, I need to export the shares using exportfs -a or restart the NFS service. Then check with exportfs -v to see the active exports.

Firewall configuration is crucial. NFS uses several ports. On CentOS 8, firewalld is the default. So I need to allow nfs, rpc-bind, and mountd services. The command would be firewall-cmd --permanent --add-service={nfs,rpc-bind,mountd} then reload. Alternatively, if the zones are specific, adjust accordingly.

On the client side, install nfs-utils as well. Then create a mount point, like /mnt/nfs. Use mount -t nfs server_ip:/var/nfs /mnt/nfs to mount manually. To make it permanent, add an entry to /etc/fstab. The line would be server_ip:/var/nfs /mnt/nfs nfs defaults 0 0. But maybe include auto, or other options like rw, sync, etc. Also, check if _netdev is needed for network filesystems to prevent boot issues.

Testing the connection is important. From the client, mount and then try creating a file. Check permissions, maybe. Also, look at the server's logs if there are issues, maybe in /var/log/messages or using journalctl.

Wait, SELinux might block NFS shares. So need to set the right contexts. The NFS export directories should have the correct SELinux type, like nfs_t. Or use semanage to adjust the context. Alternatively, temporarily set SELinux to permissive mode to test if that's the issue.

Another thing: NFS versions. CentOS 8 probably uses NFSv4 by default. So clients should mount using NFSv4. The mount command might need to specify vers=4 if there's a version mismatch. Also, in the server configuration, maybe adjust /etc/nfs.conf for NFSv4 settings, like enabling NLM (Network Lock Manager) if needed. But I think defaults are okay.

Wait, in CentOS 8, is there a separate configuration file like /etc/nfs.conf? Maybe I should check that. The nfs.conf file might have some global settings. By default, it should be set up for NFSv4. So maybe no changes needed there unless specific tuning is required.

Also, client-side after mounting, check with df -h to see if the filesystem is mounted. And test read/write operations.

So putting it all together: install packages, configure exports, set up firewall, handle SELinux, start services, then client-side install packages, mount, and test. Also, fstab entry for persistence.

# Ответ

Here's a step-by-step guide to configure NFS Server and Client on CentOS 8 Stream:

---

### **NFS Server Configuration**

1. **Install Required Packages**
```bash
sudo dnf install nfs-utils -y
```

2. **Enable and Start Services**
```bash
sudo systemctl enable --now nfs-server rpcbind
# For older implementations (not required for NFSv4+)
sudo systemctl enable --now rpc-statd.service nfs-idmapd.service
```

3. **Create Shared Directory**
```bash
sudo mkdir -p /var/nfs/shared
sudo chown nobody:nobody /var/nfs/shared  # Optional: Adjust permissions as needed
```

4. **Configure Exports (`/etc/exports`)**
```bash
sudo vi /etc/exports
```
Add the following line (adjust IP range and options):
```bash
/var/nfs/shared  192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
```
- `rw`: Read/Write access
- `sync`: Write changes to disk before confirming
- `no_subtree_check`: Improves reliability
- `no_root_squash`: Allows root access from client (use cautiously)

5. **Apply Exports**
```bash
sudo exportfs -a  # Activate shares
sudo exportfs -v  # Verify
```

6. **Configure Firewall**
```bash
sudo firewall-cmd --permanent --add-service={nfs,rpc-bind,mountd}
sudo firewall-cmd --reload
```

7. **SELinux Configuration (if enforced)**
```bash
sudo setsebool -P nfs_export_all_rw 1  # Allow NFS to export RW
sudo semanage fcontext -a -t nfs_t "/var/nfs/shared(/.*)?"
sudo restorecon -Rv /var/nfs/shared
```

---

### **NFS Client Configuration**

1. **Install NFS Utilities**
```bash
sudo dnf install nfs-utils -y
```

2. **Create Mount Point**
```bash
sudo mkdir -p /mnt/nfs/shared
```

3. **Test Manual Mount**
```bash
sudo mount -t nfs4 server_ip:/var/nfs/shared /mnt/nfs/shared
```

4. **Verify Mount**
```bash
df -hT | grep nfs4
```

5. **Persistent Mount (`/etc/fstab`)**
```bash
sudo vi /etc/fstab
```
Add:
```bash
server_ip:/var/nfs/shared  /mnt/nfs/shared  nfs4  defaults,_netdev  0 0
```
- `_netdev`: Wait for network before mounting

6. **Test Fstab Entry**
```bash
sudo mount -a
```

---

### **Troubleshooting Tips**

- **Server Logs**: Check `/var/log/messages` or `journalctl -u nfs-server`
- **Client Connectivity**: Use `showmount -e server_ip` to list exports
- **Permissions**: Ensure UID/GID consistency between server/client users
- **NFS Version**: Force NFSv4 with `mount -t nfs4` or in fstab
- **Firewall**: Confirm ports 2049 (NFS), 111 (RPC), and 20048 (mountd) are open

---

### **Security Recommendations**

1. Use `ro` (read-only) instead of `rw` if possible
2. Replace `no_root_squash` with `root_squash` for production
3. Restrict exports to specific IPs (e.g., `192.168.1.100` instead of `/24`)
4. Consider Kerberos for NFSv4 authentication (`sec=krb5`)

Let me know if you need adjustments for specific use cases!
