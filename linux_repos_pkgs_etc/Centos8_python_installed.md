# Вопрос
Now you need to check the python version and installed modules. As before, it is necessary to explain the commands and the conclusion for the young specialist.

# Рассуждения
We are working on CentOS 8 Stream. We need to check the Python version and installed modules on both servers and compare them.

Since there might be multiple Python versions, we should check the default one and also note if there are others.

We'll break it down into steps: checking the Python version and then the installed modules.

Step 1: Check the default Python version and the path to the Python executable.

Step 2: List the installed Python modules for that Python version.

Step 3: Compare the results between the two servers.

However, note that the system might have both Python 2 and Python 3. Since Python 2 is deprecated, we focus on Python 3.

But we should check what is available.

Let's proceed:

Step 1: Check Python version

We can use:

python --version   # for Python 2 (if exists)

python3 --version  # for Python 3

But note: on CentOS 8, the default Python might be Python 3.6 or 3.8, and the command `python` might point to Python 3 or not.

We can check what is available by:

which python

which python3

Or more robust:

python -V  2> /dev/null || echo "python not installed"

python3 -V 2> /dev/null || echo "python3 not installed"

We are interested in the version that is used by the system or by our applications.

Step 2: Check installed modules

For the Python interpreter we are interested in (likely python3), we can list the installed packages with:

pip list

But note: we might have multiple pip versions (pip, pip3). We should use the pip corresponding to the Python version.

Alternatively, we can run:

python3 -m pip list

To avoid any ambiguity.

However, we must consider that there might be virtual environments. We are only concerned with the system-wide installation?

The task says "installed modules" on the server. So we are focusing on the system Python, not in any virtual environment.

But note: if the user is in a virtual environment, we might get the modules of the virtual environment. So we must ensure we are in the global context.

We can also use:

pip freeze   # but same as pip list, and we should specify the pip for the Python we are checking.

Step 3: Save the output and compare.

However, note that the same module might be installed in different locations (user vs system) but we are only interested in the list.

Plan:

For each server:

1. Check the available Python versions (at least the default ones).

2. For each Python version (if multiple, but we focus on the one that is used, likely Python 3), list the modules.

3. Save the module list to a file.

Then compare the files.

But note: the young colleague might not know about pip or virtual environments. So we must explain.

Let's write the commands and explanations.

Considering we are on CentOS 8 Stream, and we are in a system context (not in a virtual environment) and we are root or have sudo access?

We assume we are running as root or a user with access to system Python.

However, sometimes user-specific packages are installed. We might need to check both system and user? The task says "on the server", so we are to check all? But typically, for a server comparison, we care about the system environment and the environment of the service accounts? That might be too broad.

Since the task is about the server, we focus on the system Python and the global site-packages.

We'll do:

sudo python3 -m pip list --format=freeze   # for system-wide Python 3

But note: on CentOS, the system Python might be managed by dnf/yum and we should avoid using pip to upgrade system packages. However, for listing, it is safe.

Alternatively, we can use:

rpm -qa | grep -i '^python3'   # to list all installed python3 packages from RPM

But that will only show the packages installed via RPM, not via pip. So we need both?

The task says "installed modules", which includes both RPM-installed and pip-installed.

How to get a comprehensive list?

Option 1: `pip list` will show all packages, regardless of how they were installed (but only for the specific Python interpreter).

Option 2: `rpm -qa` for the RPM packages, but that won't show pip-installed modules.

So we need to use `pip list` for the Python interpreter we are checking.

However, note: if there are multiple Python interpreters (e.g., /usr/bin/python3.6 and /usr/bin/python3.9), we need to check each one? That complicates.

We'll focus on the one that is the default for `python3`.

Steps:

Server A:

# Check the default Python3 version

python3 --version > python_version_A.txt

# List all installed modules for Python3

python3 -m pip list --format=freeze > python_modules_A.txt

Server B:

python3 --version > python_version_B.txt

python3 -m pip list --format=freeze > python_modules_B.txt

Then compare:

diff python_version_A.txt python_version_B.txt

diff python_modules_A.txt python_modules_B.txt

But note: the pip list might have version numbers and the order might be different. We should sort the modules list.

Alternatively, we can use:

sort python_modules_A.txt > python_modules_A_sorted.txt

sort python_modules_B.txt > python_modules_B_sorted.txt

diff python_modules_A_sorted.txt python_modules_B_sorted.txt

However, the young colleague might not have pip installed? We can check.

On CentOS 8, we can install pip for Python3 if it's not there by:

sudo dnf install python3-pip

But we are just checking, so we should avoid installing anything. If pip is not installed, then we can use:

python3 -c "help('modules')"

But that is less clean and includes built-in modules. We want only the installed packages? And the output is not easy to parse.

Alternatively, we can use:

python3 -m pip list   # if pip is installed

If pip is not installed, we might have to rely on RPM.

Considering the task, we are expected to check the installed modules, which typically are managed by pip for Python packages.

But note: the system Python might have packages installed via RPM and pip. We need to see both.

Actually, `pip list` will show all packages, including those installed by RPM? Yes, because they are all in the same site-packages.

So we use `pip list` if pip is available. If not, we might have to install pip? But we are only comparing, so we can do:

if ! command -v pip3 &> /dev/null; then

echo "pip3 not installed, no external Python packages."

else

pip3 list --format=freeze | sort > python_modules.txt

fi

However, we are using `python3 -m pip` which works even if the `pip` command is not in the PATH.

But if the `pip` module is not installed, then we get an error.

So we can check:
~~~

python3 -m pip --version > /dev/null 2>&1

if [ $? -eq 0 ]; then

python3 -m pip list --format=freeze | sort > python_modules.txt

else

echo "Python3 pip module not available. Using alternative method."

# Try to list using the package manager? Or just the built-in?

# We can use the following to get a list of installed packages via the system package manager for python3

rpm -qa "python3*" | sort > python3_rpm_packages.txt

# But this won't show non-RPM packages. So we have to note that.

# And we can also try to use the Python internals?

# We can run a Python one-liner to list the modules? But it's complex and might not be complete.

# We'll leave it as the RPM list and note that non-RPM packages are not captured.

fi

~~~

Given the time, we'll assume pip is installed. If not, we have to handle it.

But for the sake of this task, we assume pip is installed.

Let's write the full explanation for the young colleague

# Ответ

### Comparing Python Environments on CentOS 8 Stream Servers

#### **Step 1: Check Default Python Version**
**Command:**
```bash
python3 --version
# OR for systems with multiple versions
ls -l /usr/bin/python* | grep -v config
```

**Explanation:**  
- `python3 --version` shows the default Python 3 version  
- The alternative command lists all installed Python binaries:  
  - `ls -l` shows detailed file info  
  - `/usr/bin/python*` finds all Python executables  
  - `grep -v config` excludes configuration tools  

**Example Output:**  
```
python3.8 (default)
python3.9
```

**Why:**  
Identifies which Python versions are available and which is default. Servers should match to ensure compatibility.

---

#### **Step 2: Verify Python Symlinks**
**Command:**  
```bash
ls -l /usr/bin/python3
```

**Output:**  
```
lrwxrwxrwx. 1 root root 9 Mar 23  2023 /usr/bin/python3 -> python3.8
```

**Explanation:**  
Shows where the `python3` command points. The arrow (`?`) indicates the actual version used when running `python3`.

**Junior Tip:**  
Symbolic links allow multiple Python versions to coexist while maintaining a default.

---

#### **Step 3: List Installed Python Packages**
**Command:**  
```bash
pip3 freeze | sort > serverA_python_packages.txt
```

**Alternative (for system-managed Python):**  
```bash
rpm -qa | grep -i '^python3' | sort
```

**Explanation:**  
- `pip3 freeze` lists all Python packages installed via pip with exact versions  
- `sort` orders alphabetically for clean comparison  
- `rpm -qa` shows packages installed through RPM (system packages)  
- `grep -i '^python3'` filters Python 3-related RPMs  

**Save Output:**  
```bash
# Server A
pip3 freeze | sort > serverA_pip_packages.txt

# Server B
pip3 freeze | sort > serverB_pip_packages.txt
```

---

#### **Step 4: Compare Python Packages**
**Command:**  
```bash
diff serverA_pip_packages.txt serverB_pip_packages.txt
```

**Alternative Visual Tool:**  
```bash
meld serverA_pip_packages.txt serverB_pip_packages.txt
```

**Sample Output:**  
```diff
< requests==2.28.1    # Only on Server A
> boto3==1.26.8       # Only on Server B
---
< numpy==1.21.0       # Different version
> numpy==1.22.3
```

---

### Key Checks for Junior Engineers  
1. **Version Consistency**  
   - Applications may break if servers run different Python versions  
   - *Fix:* Update symlinks with `sudo alternatives --config python3`  

2. **Package Differences**  
   - **Missing Packages:** `pip3 install <package>` to sync  
   - **Version Conflicts:** `pip3 install <package>==<version>` to align  
   - **Extra Packages:** Remove with `pip3 uninstall <package>`  

3. **Environment Types**  
   - System packages (RPM): Managed via `dnf`  
   - Pip packages: User-installed libraries  
   - *Never mix both* for the same package!  

---

### Critical Findings Example  
| Server A | Server B | Risk | Solution |
|----------|----------|------|----------|
| Python 3.8 | Python 3.9 | App compatibility | Standardize version |  
| `django==3.2` | `django==4.1` | Feature differences | Pin version in requirements.txt |  
| No `cryptography` | `cryptography==38.0.4` | Security vulnerability | Install/upgrade on both |  

---

### Why This Matters  
1. **Reproducibility:**  
   - Identical environments prevent "works on my machine" issues  
2. **Security:**  
   - Outdated packages (e.g., `cryptography<41.0`) may have vulnerabilities  
3. **Dependency Hell:**  
   - Version conflicts cause obscure crashes (e.g., `numpy` mismatch breaks ML models)  

---

### Pro Tips  
1. **Virtual Environments:**  
   Always use venv for projects:  
   ```bash
   python3 -m venv myapp && source myapp/bin/activate
   ```  
   Isolates project dependencies from system packages.  

2. **Requirements Files:**  
   Maintain `requirements.txt` for consistency:  
   ```bash
   # Generate
   pip3 freeze > requirements.txt
   
   # Install
   pip3 install -r requirements.txt
   ```  

3. **RPM vs Pip:**  
   - Use `dnf` for system tools (e.g., `python3-requests`)  
   - Use `pip` only for application-specific libraries  

> ?? **Golden Rule:** Never use `sudo pip3 install`! It can break system-managed Python packages.
