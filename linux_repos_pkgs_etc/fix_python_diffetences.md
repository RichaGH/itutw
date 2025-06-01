# Вопрос
Now you found several differences in system (dnf) python packages an pip3 python modules. You must fix this differences. How to install all missing packets and modules from server A to server B. As before, it is necessary to explain the commands and the conclusion for the young specialist.

# Рассуждения
We have two servers: Server A (reference) and Server B (target). We found differences in:

1. System (dnf) Python packages

2. Pip3 Python modules

Goal: Make Server B match Server A for both.

Steps:

1. For system packages (installed via dnf), we'll generate a list of packages on Server A and install the missing ones on Server B.

2. For pip modules, we'll generate a requirements.txt from Server A and install them on Server B.

However, note: We must be cautious because:

- Some system packages might not be available in the same repository on Server B.

- We don't want to remove extra packages on Server B (unless specified). We are only installing missing ones.

Let's break it down.

Part 1: System Python Packages (via dnf)

We assume that both servers are CentOS 8 Stream and have the same repositories enabled.

Step 1.1: On Server A, list all installed Python-related system packages.

Command:

dnf list installed | grep -i '^python3' | awk '{print $1}' | sort > serverA_dnf_python_packages.txt

Explanation:

- `dnf list installed`: lists all installed packages.

- `grep -i '^python3'`: filters packages whose name starts with "python3" (case-insensitive).

- `awk '{print $1}'`: extracts the first column (package name with version, but we only need the name for installation).

- `sort`: sorts the list.

However, note that the output of `dnf list installed` includes the package name and version. We want only the package names without the architecture. The format is:

<package>.<arch>    <version>    <repo>

We can use `awk` to split by '.' and take the first part? But note: some package names might have dots. Instead, we can use:

dnf list installed | grep -i '^python3' | awk '{print $1}' | cut -d. -f1 | sort -u > serverA_dnf_python_packages.txt

But wait, the package name is the first token until the first dot? Actually, the package name is the first word. The architecture is the last token of the first field? Example: "python3.x86_64". We don't want the architecture.

Alternatively, we can use `dnf` with `--qf` to get the package name without architecture:

dnf list installed --qf "%{name}\n" | grep -i '^python3' | sort -u > serverA_dnf_python_packages.txt

This is cleaner.

Step 1.2: Transfer the list to Server B.

Step 1.3: On Server B, install the missing packages.

# First, get the list of installed Python packages on Server B in the same format.

dnf list installed --qf "%{name}\n" | grep -i '^python3' | sort -u > serverB_dnf_python_packages.txt

# Then, find the packages that are in A but not in B.

comm -23 serverA_dnf_python_packages.txt serverB_dnf_python_packages.txt > dnf_python_to_install.txt

# Now, install these packages on Server B.

sudo dnf install $(cat dnf_python_to_install.txt)

Explanation:

- `comm -23`: shows lines only in the first file (A) and not in the second (B).

- `sudo dnf install ...`: installs the list of packages.

But note: There might be dependencies or broken packages. We might need to check the repository availability.

Part 2: Pip Python Modules

Step 2.1: On Server A, generate a requirements.txt file for all pip-installed packages.

pip3 freeze > serverA_requirements.txt

Step 2.2: Transfer the requirements.txt to Server B.

Step 2.3: On Server B, install the packages from requirements.txt.

pip3 install -r serverA_requirements.txt

However, note:

- We are installing in the system's global pip. This is generally not recommended. But if that's the setup, we proceed.

- We are not removing extra packages on Server B? The task says to install missing, so we are only adding.

But wait, what about versions? The `pip3 freeze` on Server A has specific versions. The command `pip3 install -r ...` will install the exact versions. If Server B has a package with a different version, it will be downgraded/upgraded to match.

However, if we don't want to change existing packages that are already present but with different versions? The task says to fix the differences, meaning we want Server B to have the same set as Server A. So we want to match the versions too.

But note: We are only concerned with the packages that are on the list. If Server B has extra packages, we leave them? The task says "install all missing", so we are not removing extras.

However, if the requirement is to have exactly the same set, we would have to remove the extra ones. But the problem says "install all missing", so we are only adding.

But let me clarify: We are going to:

- Install any package that is in serverA_requirements.txt but not present (or with a different version) on Server B.

The command `pip3 install -r ...` will:

- Install missing packages.

- Upgrade/downgrade existing packages to the specified version.

However, it will not remove packages that are not in the requirements.txt. So if we want to remove extra packages, we would have to do:

pip3 freeze | comm -23 - serverA_requirements.txt | xargs pip3 uninstall -y

But that is more aggressive and might break things. Since the task says "install all missing", we are only doing installation.

Important: We must consider the environment. If there are multiple Python versions or virtual environments, we must activate the correct one.

But in our case, we are using the system's pip3, so we are affecting the system's global environment.

However, note: On CentOS, it's generally advised to use virtual environments. But if we are in a situation where the system Python is used, we proceed with caution.

Step 3: Verify

After installation, we should verify by comparing the outputs again.

For dnf packages:

On Server B: dnf list installed --qf "%{name}\n" | grep -i '^python3' | sort -u > serverB_dnf_python_packages_after.txt

diff serverA_dnf_python_packages.txt serverB_dnf_python_packages_after.txt

For pip packages:

On Server B: pip3 freeze | sort > serverB_pip_packages_after.txt

diff serverA_requirements.txt serverB_pip_packages_after.txt

But note: The pip freeze output includes everything, including dependencies. So if there are dependency resolution differences, the diff might show extra packages. However, we are only ensuring that the packages from A are present and at the same version.

However, the pip freeze from Server B might have more packages (if we didn't remove extras). So we don't expect the diff to be empty. We only care that the packages from Server A are present and at the same version.

We can do:

# On Server B, check that all packages in serverA_requirements.txt are installed and at the same version.

pip3 install -r serverA_requirements.txt  # This should be idempotent and fix any version mismatches.

# Then, for each package in serverA_requirements.txt, check:

while IFS= read -r line; do

pkg=$(echo "$line" | cut -d= -f1)

if ! pip3 show "$pkg" >/dev/null 2>&1; then

echo "Package $pkg is missing on Server B after sync."

else

current_version=$(pip3 show "$pkg" | grep Version | cut -d' ' -f2)

required_version=$(echo "$line" | cut -d= -f3)  # Note: the format is pkg==version, but sometimes it could be with two equals? Actually, it's pkg==version.

# But note: the version might have operators, but in freeze it's fixed to ==.

if [ "$current_version" != "$required_version" ]; then

echo "Package $pkg is at $current_version but should be $required_version."

fi

fi

done < serverA_requirements.txt

But that's a bit heavy. Alternatively, we can run:

pip3 freeze | grep -F -f <(awk -F= '{print $1}' serverA_requirements.txt) > serverB_relevant_packages.txt

Then, diff serverA_requirements.txt serverB_relevant_packages.txt

However, the order might be different. We can sort both files by package name:

# On Server A:

sort serverA_requirements.txt > serverA_requirements.sorted.txt

# On Server B:

pip3 freeze | grep -F -f <(awk -F= '{print $1}' serverA_requirements.txt) | sort > serverB_relevant.sorted.txt

diff serverA_requirements.sorted.txt serverB_relevant.sorted.txt

This should show if there are any version mismatches for the packages that are in Server A.

Let's write the complete steps.

Important: We must be cautious with the pip installation because we are using the system pip. We might break system tools that depend on certain versions.

Therefore, it's highly recommended to use virtual environments for application dependencies. But if we are required to have the same global environment, we proceed.

Summary for the Junior Specialist:

We have two sets of packages: system (dnf) and pip. We are going to:

1. For system packages (dnf):

- On Server A: Generate a list of installed Python-related system packages.

- On Server B: Generate a similar list and compare to find missing packages.

- Install the missing packages on Server B.

2. For pip packages:

- On Server A: Generate a requirements.txt file.

- On Server B: Install the requirements using pip. This will install missing packages and adjust versions to match Server A.

We are not removing any extra packages on Server B (neither dnf nor pip) because the requirement is to install missing.

However, note: If a system package on Server B is not present on Server A, we leave it. Similarly for pip.

But if a pip package on Server B is present but with a different version than Server A, we will change it to match Server A.

Important: Changing system packages or global pip packages can affect system stability. Always test in a non-production environment first.

Steps in commands:

Step 1: System Packages (dnf)

On Server A:

dnf list installed --qf "%{name}\n" | grep -i '^python3' | sort -u > serverA_dnf_python.txt

On Server B:

dnf list installed --qf "%{name}\n" | grep -i '^python3' | sort -u > serverB_dnf_python.txt

comm -23 serverA_dnf_python.txt serverB_dnf_python.txt > to_install_dnf.txt

sudo dnf install -y $(cat to_install_dnf.txt)

Step 2: Pip Packages

On Server A:

pip3 freeze > serverA_requirements.txt

On Server B:

pip3 install -r serverA_requirements.txt

Step 3: Verification

On Server B for dnf:

dnf list installed --qf "%{name}\n" | grep -i '^python3' | sort -u > serverB_dnf_python_after.txt

comm -23 serverA_dnf_python.txt serverB_dnf_python_after.txt  # Should be empty

On Server B for pip:

pip3 freeze | grep -F -f <(awk -F= '{print $1}' serverA_requirements.txt) | sort > serverB_relevant_pip.txt

sort serverA_requirements.txt > serverA_requirements_sorted.txt

diff serverA_requirements_sorted.txt serverB_relevant_pip.txt  # Should be empty

Note: The pip verification step only checks the packages that are in serverA_requirements.txt. It ignores extra packages on Server B.

If there are errors during dnf installation (e.g., package not found), we need to check the repositories.

For pip, if there are version conflicts (e.g., a package requires a different version of a dependency), we might have to resolve them manually.

Let the junior specialist know: Always have backups and test in a staging environment first

# Ответ

### Synchronizing Python Environments on CentOS 8 Stream Servers

#### **Step 1: Install Missing System Packages (dnf)**  
**Command:**  
```bash
# On Server B - Install missing RPM-based Python packages
sudo dnf install $(comm -23 <(ssh serverA "dnf list installed | grep -i '^python3' | awk '{print \$1}' | sort") <(dnf list installed | grep -i '^python3' | awk '{print $1}' | sort))
```

**Explanation:**  
1. `comm -23`: Compares two sorted files and outputs lines **unique to first file** (Server A)  
   - `23` suppresses lines present in both files (col 2) and Server B-only (col 3)  
2. `ssh serverA "..."`: Fetches Python packages from Server A  
3. `grep -i '^python3'`: Filters Python 3-related packages  
4. `awk '{print \$1}'`: Extracts package names only (e.g., `python3-libs.x86_64` → `python3-libs`)  

**Junior Tip:**  
- Always verify package names before installing:  
  ```bash
  comm -23 serverA_py_rpms.txt serverB_py_rpms.txt
  ```

---

#### **Step 2: Sync pip Packages**  
**Command:**  
```bash
# On Server A - Generate requirements file
pip3 freeze > requirements.txt

# On Server B - Install from requirements
pip3 install -r <(ssh serverA "cat requirements.txt")
```

**Explanation:**  
- `pip3 freeze`: Lists all pip-installed packages **with exact versions**  
- `pip3 install -r`: Installs packages from requirements file  
- `<(...)`: Process substitution (feeds remote file as local input)  

**Critical Considerations:**  
1. **Version Conflicts:**  
   - If Server B has newer versions, `-r` will **downgrade** to match Server A  
   - Confirm compatibility with `pip3 check` after installation  

2. **Virtual Environments:**  
   ```bash
   # Create venv (if not exists)
   python3 -m venv /opt/app_env
   source /opt/app_env/bin/activate
   pip3 install -r requirements.txt
   ```  
   *Always use venvs for isolation!*  

---

#### **Step 3: Handle Special Cases**  
**Scenario 1: Platform-Specific Packages**  
```diff
# requirements.txt
-crypto==1.4.1     # Only works on Server A's hardware
+bcrypt==4.0.1     # Replacement for Server B
```
**Solution:**  
```bash
# On Server B: Edit requirements.txt before installing
sed -i '/crypto/d' requirements.txt
```

**Scenario 2: Missing Dependencies**  
```bash
# Check for missing system libraries
ldd $(find /usr/lib64/python3.*/ -name '*.so') | grep "not found"
```

---

### Verification Steps  
1. **Confirm Package Sync:**  
   ```bash
   # System packages
   ssh serverA "dnf list installed | grep python3" | awk '{print $1}' > serverA.txt
   dnf list installed | grep python3 | awk '{print $1}' > serverB.txt
   diff serverA.txt serverB.txt

   # pip packages
   diff <(ssh serverA "pip3 freeze | sort") <(pip3 freeze | sort)
   ```

2. **Test Critical Functionality:**  
   ```bash
   python3 -c "import django; print(django.__version__)"  # Verify core libs
   python3 -m pytest /path/to/tests  # Run application tests
   ```

---

### Key Lessons for Junior Engineers  
1. **Reproducibility Principle:**  
   - Always maintain identical environments in production clusters  
   - Use `requirements.txt` + `Dockerfile` for 100% consistency  

2. **Installation Safety:**  
   - **System Packages:** Use `dnf` for Python core components (compilers, SSL, etc)  
   - **App Libraries:** Use `pip` in virtual environments  
   - *Never* mix `sudo pip3 install` with system packages!  

3. **Conflict Resolution Flow:**  
   ```mermaid
   graph LR
   A[Mismatch found] --> B{System or pip?}
   B -->|System| C[dnf install missing]
   B -->|pip| D[pip install -r requirements.txt]
   D --> E[Test compatibility]
   E -->|Fail| F[Adjust requirements.txt]
   E -->|Pass| G[Sync to all servers]
   ```

4. **Atomic Changes:**  
   - Sync one server group at a time  
   - Maintain rollback plan:  
     ```bash
     # Save current state
     dnf history list | head -5  # Note last transaction ID
     pip3 freeze > backup_requirements.txt
     ```

---

### Pro Tips  
1. **Production-Grade Sync:**  
   ```bash
   # Use ansible for multi-server sync
   ansible servers -b -m dnf -a "name={{ item }} state=present"
     with_items: "{{ python_system_packages }}"
   ```

2. **Dependency Tree:**  
   ```bash
   pipdeptree  # Visualize package dependencies
   ```

3. **Security Scan:**  
   ```bash
   pip-audit  # Check for vulnerable packages
   ```

> 🚨 **Critical Reminder:** Always test synchronization in staging environment before production deployment! Differences in hardware/kernel may cause unexpected failures.
