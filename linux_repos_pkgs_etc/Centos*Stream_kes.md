# Вопрос
so, we found out that our servers are on Centos 8 Stream. We need to check and compare the key database on both servers. As before, we have to explain all our actions to a young colleague.

# Рассуждения
We are working with CentOS 8 Stream servers. The key database in CentOS is managed by the RPM package manager and the DNF package manager (which is the successor to YUM). The "key database" here refers to the GPG keys used to verify the authenticity and integrity of the packages. These keys are stored in the directory `/etc/pki/rpm-gpg/`.

### Steps to Compare GPG Keys on Two CentOS 8 Stream Servers

#### Step 1: List the RPM GPG Keys on Each Server

We need to list the GPG keys that are imported into the RPM database on each server.

**Command:**

```bash

rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n'

```

**Explanation:**

- `rpm -q gpg-pubkey`: This queries all the installed GPG public keys.

- `--qf` or `--queryformat`: This allows us to format the output.

- `'%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n'`: This formats each key as:

- `NAME`: The package name (which is always `gpg-pubkey`).

- `VERSION`: The version of the key (which is the last 8 hex characters of the key ID).

- `RELEASE`: The release, which is the installation date (in seconds since the epoch).

- `SUMMARY`: The key's owner (usually the name of the organization that issued the key).

We can redirect the output to a file for each server.

**Example:**

On Server A:

```bash

rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n' | sort > serverA_gpg_keys.txt

```

On Server B:

```bash

rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n' | sort > serverB_gpg_keys.txt

```

#### Step 2: Compare the Key Lists

We can then compare the two files using the `diff` command.

**Command:**

```bash

diff serverA_gpg_keys.txt serverB_gpg_keys.txt

```

Alternatively, to see which keys are unique to each server, we can use `comm`:

**Command:**

```bash

comm -3 serverA_gpg_keys.txt serverB_gpg_keys.txt

```

**Explanation:**

- `comm` compares two sorted files and produces three columns: lines only in the first file, lines only in the second file, and lines in both.

- `-3` suppresses the lines that appear in both files, so it will show lines unique to serverA (column 1) and lines unique to serverB (column 2).

#### Step 3: Verify the Key Files in `/etc/pki/rpm-gpg/`

The actual key files are stored in `/etc/pki/rpm-gpg/`. We can compare the contents of this directory between the two servers.

**Command to list key files:**

```bash

ls -1 /etc/pki/rpm-gpg/ | sort > serverA_rpm_gpg_files.txt

```

Similarly for Server B.

Then compare:

```bash

diff serverA_rpm_gpg_files.txt serverB_rpm_gpg_files.txt

```

**Note:** The presence of a key file in this directory does not necessarily mean it is imported. We are only listing the files for a complete picture.

#### Step 4: Check the Key Fingerprints (Optional but Recommended)

To be thorough, we can check the fingerprints of the keys. This step is more advanced and might be needed if there are keys with the same name but different content.

On each server, we can list the fingerprints of the imported keys:

**Command:**

```bash

rpm -qa gpg-pubkey* | while read key; do echo; echo "Key: $key"; rpm -qi "$key" | grep -E 'Summary|Fingerprint'; done

```

This will list each key and its fingerprint. We can redirect to a file and compare.

However, note that the `gpg-pubkey` package name includes the version and release (which is the key ID and the install date). So if the key ID (the version part) is the same, then the key is the same.

### Explanation to the Young Colleague

#### Why Check GPG Keys?

- **Security:** GPG keys ensure that the packages you install are from a trusted source and have not been tampered with. If the keys are different, one server might be trusting a different set of repositories, which could lead to security risks or inconsistencies.

- **Consistency:** In a managed environment, all servers should have the same set of trusted keys to ensure consistent package installations.

#### What We Did:

1. **Listed Imported Keys:** We used `rpm` to list all the GPG keys that are currently trusted by the package manager. We formatted the output to show the key ID and the owner, and sorted it for comparison.

2. **Compared the Lists:** We used `diff` or `comm` to see if there are any keys on one server that are not on the other.

3. **Checked Key Files:** We also compared the list of key files in the `/etc/pki/rpm-gpg/` directory to see if there are any extra or missing key files. This is because sometimes a key might be present as a file but not imported, or vice versa.

4. **Fingerprints (Optional):** If we suspect that two keys with the same name might be different, we check the fingerprint. However, the key ID (the last 8 characters of the fingerprint) is included in the `gpg-pubkey` package name, so if the key ID matches, the key is the same.

#### Example Output Interpretation:

If we see in the `diff` output:

```

< gpg-pubkey-8483c65d-5ccc5b19    CentOS Stream 8 Official Signing Key

---

> gpg-pubkey-abcdef12-12345678    Some Third-Party Key

```

This means:

- Server A has the CentOS Stream 8 Official Signing Key (with key ID ending in 5ccc5b19).

- Server B has an additional third-party key (with key ID ending in 12345678) that Server A does not have.

#### Pro Tip:

Always ensure that your production servers have the same set of trusted keys to maintain consistency and security. If you find a key on one server that is not on the other, investigate why and decide whether to import or remove it.

By following these steps, we can ensure that both servers are using the same set of trusted repositories and that the package verification process is consistent.

# Ответ

### Comparing GPG Key Databases on CentOS 8 Stream Servers

As CentOS 8 Stream uses RPM packages, the "key database" refers to **GPG keys** used to verify package authenticity. Here's how to compare them between servers:

---

#### **Step 1: List Imported GPG Keys**
**Command:**  
```bash
rpm -qa gpg-pubkey* --queryformat '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n' | sort
```

**Example Output:**  
```
gpg-pubkey-8483c65d-5ccc5b19    CentOS Stream 8 Official Signing Key
gpg-pubkey-8d5a09dc-616f11f5    EPEL 8 Signing Key
```

**Explanation to Junior Colleague:**  
- `rpm -qa gpg-pubkey*`: Lists all installed GPG keys  
- `--queryformat`: Customizes output to show:  
  - `%{NAME}`: Key name (always `gpg-pubkey`)  
  - `%{VERSION}`: Key ID (last 8 chars of fingerprint)  
  - `%{RELEASE}`: Installation date (Unix timestamp)  
  - `%{SUMMARY}`: Key description  
- `| sort`: Alphabetical sorting for clean comparison  

**Action:**  
Run on both servers and save output:  
```bash
# Server A
rpm -qa ... > serverA_keys.txt

# Server B
rpm -qa ... > serverB_keys.txt
```

---

#### **Step 2: Compare Key Lists**
**Command:**  
```bash
diff serverA_keys.txt serverB_keys.txt
```

**Alternative (visual):**  
```bash
vimdiff serverA_keys.txt serverB_keys.txt
```

**Expected Differences:**  
```diff
> gpg-pubkey-9c0d193d-5a34f32a    Custom Company Signing Key
< gpg-pubkey-4ebfc273-5b7431a2    Deprecated Security Key
```

---

#### **Step 3: Verify Key Files (Advanced Check)**
**Command:**  
```bash
rpm -ql $(rpm -qa gpg-pubkey*) | grep '\.pub$'
```

**Output:**  
```
/etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
/etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-8
```

**Explanation:**  
- `rpm -ql <key-package>`: Lists files installed by a package  
- `grep '\.pub$'`: Filters for actual public key files  
- Shows physical key locations in `/etc/pki/rpm-gpg/`

**Compare with:**  
```bash
# List key files on both servers
ls -1 /etc/pki/rpm-gpg/*.pub > serverA_keyfiles.txt
ls -1 /etc/pki/rpm-gpg/*.pub > serverB_keyfiles.txt
diff serverA_keyfiles.txt serverB_keyfiles.txt
```

---

### Key Findings Interpretation  
1. **Missing Keys:**  
   - If Server B lacks a key present on Server A (`<` symbol in diff), it won't verify packages from that source  
   - *Example:* `Deprecated Security Key` missing on Server B  

2. **Extra Keys:**  
   - If Server B has additional keys (`>` symbol), it trusts packages from extra sources  
   - *Example:* `Custom Company Signing Key` only on Server B  

3. **Security Implications:**  
   - Mismatched keys could cause package verification failures  
   - Unauthorized keys might indicate security policy violations  

---

### Why This Matters  
1. **Consistency:**  
   - Production servers should have identical trusted keys to ensure uniform package verification  

2. **Troubleshooting:**  
   - If one server rejects valid packages (`Public key not found` errors), check missing keys  

3. **Security:**  
   - Unapproved keys could allow installation of untrusted software  
   - Always verify the origin of new keys (e.g., `Custom Company Signing Key` should come from internal IT)  

---

### Pro Tips for Junior Engineer  
1. **Key Fingerprints:**  
   Verify suspicious keys with:  
   ```bash
   rpm -qi gpg-pubkey-9c0d193d-5a34f32a | grep 'Fingerprint'
   # Compare with official sources
   ```

2. **Add Missing Keys:**  
   ```bash
   sudo rpm --import /path/to/missing-key.pub
   ```

3. **Remove Invalid Keys:**  
   ```bash
   sudo rpm -e gpg-pubkey-4ebfc273-5b7431a2
   ```

> 🔑 **Rule of Thumb:** Never add keys from unverified sources! Always validate fingerprints with official documentation.
