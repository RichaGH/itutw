# Вопрос
You are an IT engineer with 20 years of experience. You need to compare the two linux servers for installed packages and repositories. Find the differences. And also explain to the young colleague the meaning of the commands entered and the results of their execution.

# Рассуждения
We are going to compare two Linux servers (let's call them Server A and Server B) for installed packages and repositories.

Steps:

1. For installed packages, we can use the package manager (apt for Debian/Ubuntu, yum/dnf for RHEL/CentOS/Fedora).

Since we don't know the exact distro, I'll assume we are on Debian/Ubuntu for this example (using apt).

If it's RHEL based, we can adjust accordingly.

2. For repositories, we can look at the files in /etc/apt/sources.list and /etc/apt/sources.list.d/ for Debian/Ubuntu.

For RHEL based, we would look at /etc/yum.repos.d/.

3. We will generate a list of installed packages and repositories on each server and then compare.

Let's break it down:

Part 1: Installed Packages

On each server, we can get a list of installed packages with:

dpkg-query -l | grep '^ii' | awk '{print $2}' > packages_serverA.txt

But note: `dpkg-query` is for Debian/Ubuntu. For RHEL based, we would use:

rpm -qa > packages_serverA.txt   # or for dnf: dnf list installed

Part 2: Repositories

For Debian/Ubuntu:

We can combine the sources:

cat /etc/apt/sources.list > repos_serverA.txt

cat /etc/apt/sources.list.d/*.list >> repos_serverA.txt  2>/dev/null

For RHEL based:

ls /etc/yum.repos.d/ > repos_serverA.txt   # but this only lists the file names, not content

# Instead, we should dump the content of all .repo files:

cat /etc/yum.repos.d/*.repo > repos_serverA.txt

4. Then we can transfer these files (packages and repos) to a common machine and compare.

However, note: We are an IT engineer and we are explaining to a young colleague.

Let's assume we are working on two Debian/Ubuntu servers.

Step-by-step commands and explanation:

For Server A and Server B:

Step 1: Get installed packages

Command:

dpkg-query -l | grep '^ii' | awk '{print $2}' | sort > packages_serverA.txt

Explanation:

- `dpkg-query -l`: lists all packages installed (with status, version, etc.)

- `grep '^ii'`: filters lines that start with 'ii' (which means installed and current)

- `awk '{print $2}'`: prints the second field (the package name)

- `sort`: sorts the package names alphabetically (for easier comparison)

Step 2: Get repository list

Command:

cat /etc/apt/sources.list > repos_serverA.txt

cat /etc/apt/sources.list.d/*.list >> repos_serverA.txt 2>/dev/null

sort -u repos_serverA.txt -o repos_serverA.txt   # sort and remove duplicates

Explanation:

- The first `cat` writes the content of the main sources file.

- The second `cat` appends the content of all .list files in the sources.list.d directory.

The `2>/dev/null` suppresses any error (like if the directory is empty).

- Then we sort and remove duplicates with `sort -u` and overwrite the file.

Step 3: Transfer the files (packages_serverA.txt and repos_serverA.txt) from both servers to a common location.

Step 4: Compare the packages:

We can use `diff` or `comm` to compare the package lists.

Example using `diff`:

diff packages_serverA.txt packages_serverB.txt

But note: `diff` shows the differences line by line. Alternatively, we can use `comm` to show common and unique lines.

To see packages only on Server A:

comm -23 packages_serverA.txt packages_serverB.txt

To see packages only on Server B:

comm -13 packages_serverA.txt packages_serverB.txt

Explanation of `comm`:

- `comm` reads two sorted files and produces three columns:

Column 1: lines unique to file1

Column 2: lines unique to file2

Column 3: lines common to both.

- The options -1, -2, -3 suppress the corresponding column.

- So `comm -23 file1 file2` suppresses column 2 and 3, leaving only lines unique to file1.

Step 5: Compare the repositories:

Similarly, we can use `diff` or `comm` on the repository files.

However, note that the repository file might have comments and whitespace. We might want to clean them first?

But for simplicity, we can do:

diff repos_serverA.txt repos_serverB.txt

Or if we want to ignore whitespace and case:

diff -i -w repos_serverA.txt repos_serverB.txt

But note: the repository file might have the same repository but in a different order? We sorted them so that should help.

Alternatively, we can use a tool like `vimdiff` for an interactive view.

However, for a scripted approach, `diff` or `comm` is fine.

Let's write a script for both servers (assuming they are Debian/Ubuntu):

On Server A:

# Packages

dpkg-query -l | grep '^ii' | awk '{print $2}' | sort > /tmp/packages_serverA.txt

# Repositories

cat /etc/apt/sources.list > /tmp/repos_serverA.txt

cat /etc/apt/sources.list.d/*.list >> /tmp/repos_serverA.txt 2>/dev/null

sort -u /tmp/repos_serverA.txt -o /tmp/repos_serverA.txt

On Server B:

# Packages

dpkg-query -l | grep '^ii' | awk '{print $2}' | sort > /tmp/packages_serverB.txt

# Repositories

cat /etc/apt/sources.list > /tmp/repos_serverB.txt

cat /etc/apt/sources.list.d/*.list >> /tmp/repos_serverB.txt 2>/dev/null

sort -u /tmp/repos_serverB.txt -o /tmp/repos_serverB.txt

Then copy these files to your local machine and compare.

Example of comparing packages:

comm -23 /tmp/packages_serverA.txt /tmp/packages_serverB.txt > only_in_A.txt

comm -13 /tmp/packages_serverA.txt /tmp/packages_serverB.txt > only_in_B.txt

Similarly for repositories.

But note: the repository lines might have slight variations (like comments) but we are not removing comments.

We might want to preprocess the repository files to remove comments and blank lines.

Preprocessing for repositories (optional):

grep -v '^#' /tmp/repos_serverA.txt | grep -v '^$' | sort -u > repos_serverA_clean.txt

grep -v '^#' /tmp/repos_serverB.txt | grep -v '^$' | sort -u > repos_serverB_clean.txt

Then compare the clean versions.

Important: This approach might still have issues if the same repository is written in a different way (like with/without trailing slash, etc.).

But for a quick comparison, it's acceptable.

For the young colleague:

We are generating two lists: one for installed packages and one for repository sources.

We then compare these lists to find differences.

Why is it important?

- To ensure consistency between servers (for example, in a cluster or when troubleshooting why one server behaves differently).

- To identify missing packages or repositories that might be causing issues.

Note: If the servers are of different distributions or versions, the package names might differ. Also, the repositories will likely be different.

If the servers are RHEL based, we would use:

For packages:

rpm -qa --queryformat '%{NAME}\n' | sort > packages_serverA.txt   # or use `yum list installed` and parse

For repositories:

yum repolist   # but this doesn't give the repo URLs, so we rely on the .repo files as above.

Alternatively, on RHEL8+ we can use `dnf`:

dnf list installed > packages_serverA.txt   # but note this includes more info, so we might want to extract package names

To get just the package names with dnf:

dnf list installed | awk '{print $1}' | sed 's/\..*//' | sort > packages_serverA.txt

But note: the output of `dnf list installed` has three columns: Name, Version, Repository. And the first column might be in the format "name.arch".

We can split by dot and take the first part? However, the arch might not always be present? Alternatively, we can use:

dnf list installed -q | awk '{print $1}' | cut -d. -f1 | sort > packages_serverA.txt

But a more robust way is to use `rpm -qa` as it is always available.

For repositories on RHEL based, we can also use:

dnf repolist   # to get the enabled repositories, but again not the URLs.

So for repository sources, we stick with the .repo files.

We can also compare enabled repositories by:

dnf repolist enabled > enabled_repos_serverA.txt

Then compare the enabled repos (without the URLs) but that might be sufficient.

However, the original request was for repositories (which I interpret as the sources). So the .repo files are the source of truth.

Conclusion:

We have a method for Debian and RHEL based systems to compare installed packages and repositories.

We must be cautious about the distribution and version.

For the young colleague: always check the distribution first with:

cat /etc/os-release

Then choose the appropriate commands.

Let me know if you have any questions.

# Ответ

### Comparison of Linux Servers: Installed Packages and Repositories

#### **Step 1: Identify Installed Packages**
**Command (Debian/Ubuntu):**  
```bash
dpkg-query -f '${Package}\n' -W | sort > serverA_packages.txt
```
**Command (RHEL/CentOS):**  
```bash
rpm -qa --queryformat '%{NAME}\n' | sort > serverA_packages.txt
```

**Explanation:**  
- **`dpkg-query` (Debian/Ubuntu)** or **`rpm -qa` (RHEL/CentOS)** lists all installed packages.  
- **`-f '${Package}\n'` / `--queryformat '%{NAME}\n'`** extracts only package names.  
- **`sort`** orders the list alphabetically for clean comparison.  
- **Output saved to `serverA_packages.txt`** (repeat for Server B).  

**Why?**  
This creates a baseline list of packages to compare between servers.  

---

#### **Step 2: Compare Packages**  
**Command:**  
```bash
diff serverA_packages.txt serverB_packages.txt
```
**Alternative (show differences clearly):**  
```bash
comm -3 serverA_packages.txt serverB_packages.txt
```

**Explanation:**  
- **`diff`** highlights line-by-line differences.  
- **`comm -3`** suppresses lines common to both files, showing only unique packages.  
  - **Column 1:** Packages only on Server A.  
  - **Column 2:** Packages only on Server B.  

**Example Output:**  
```diff
5d4
< apache2   # Only on Server A
8a8
> nginx     # Only on Server B
```

---

#### **Step 3: Identify Repositories**  
**Command (Debian/Ubuntu):**  
```bash
grep -rhE '^deb ' /etc/apt/sources.list /etc/apt/sources.list.d/ | sort -u > serverA_repos.txt
```
**Command (RHEL/CentOS):**  
```bash
grep -rhE '^\[|^baseurl=' /etc/yum.repos.d/ | sort -u > serverA_repos.txt
```

**Explanation:**  
- **Debian/Ubuntu:**  
  - **`grep -rhE '^deb '`** finds all repository lines starting with `deb` (ignoring comments).  
  - Searches `/etc/apt/sources.list` and `/etc/apt/sources.list.d/`.  
- **RHEL/CentOS:**  
  - **`grep -rhE '^\[|^baseurl='`** extracts repository IDs (`[repo-name]`) and URLs.  
- **`sort -u`** removes duplicates.  
- **Output saved to `serverA_repos.txt`** (repeat for Server B).  

---

#### **Step 4: Compare Repositories**  
**Command:**  
```bash
diff serverA_repos.txt serverB_repos.txt
```
**For Detailed Insight (Debian):**  
```bash
apt-cache policy $(dpkg-query -f '${Package}\n' -W)  # Shows which repo each package came from
```

**Example Output:**  
```diff
> deb http://security.ubuntu.com focal-security main  # Only on Server B
< deb http://archive.ubuntu.com focal universe      # Only on Server A
```

---

### Key Findings Summary  
1. **Packages:**  
   - **Only on Server A:** `apache2, packageX`  
   - **Only on Server B:** `nginx, packageY`  
2. **Repositories:**  
   - **Server A:** Uses `focal universe` (Ubuntu).  
   - **Server B:** Uses `focal-security` (Security updates).  

---

### Why This Matters to a Junior Engineer  
- **Consistency:** Identifies deviations in software/configurations (e.g., why Server A has Apache but Server B runs Nginx).  
- **Security:** Unapproved repositories might introduce vulnerabilities.  
- **Troubleshooting:** Explains "works on one server but not the other" issues (e.g., missing packages).  
- **Best Practices:**  
  - Always **sort** lists before comparing (ensures alignment).  
  - Use **`grep -r`** to recursively search directories (e.g., for fragmented repo configs).  
  - **`apt-cache policy`** reveals package origins (crucial for audits).

>  **Pro Tip:** Use `meld` (GUI) or `vimdiff` for visual comparisons if allowed on your system!
