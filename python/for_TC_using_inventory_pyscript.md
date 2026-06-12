# Вопрос
Ok, I can run this script localy on my pc. Now we must prepare docker container for run it in teamcity. Write docker file and explain all steps for young collegue.

# Рассуждения
We need to provide a Dockerfile to containerize the Python script, and explain steps for running in TeamCity. The user wants to prepare a Docker container for running the inventory script in TeamCity. We'll write Dockerfile, perhaps also entrypoint script. Explanation for young colleague on how to build, push to registry, and use in TeamCity.

Requirements: Python 3, install paramiko, copy script, set entrypoint. Also handle SSH keys mounting as volumes or passing via environment? In TeamCity, likely need to mount SSH keys or use SSH agent forwarding? We'll explain options.

We'll produce Dockerfile, and then detailed explanation.

Let's produce final answer.

# Ответ
# Docker Container for Linux VM Inventory Script

Below is a complete Dockerfile and explanation for running your inventory scanner in TeamCity.

## Dockerfile

```dockerfile
# Use official Python 3.11 slim image as base
FROM python:3.11-slim

# Set metadata
LABEL maintainer="your-team@company.com"
LABEL description="Linux VM Inventory Scanner for TeamCity"

# Set working directory inside container
WORKDIR /app

# Install system dependencies (optional - paramiko has no native libs)
# RUN apt-get update && apt-get install -y --no-install-recommends \
#     openssh-client \
#     && rm -rf /var/lib/apt/lists/*

# Copy requirements file and install Python packages
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the inventory script
COPY linux_vm_inventory.py .

# Make script executable
RUN chmod +x linux_vm_inventory.py

# Set default command to show help
ENTRYPOINT ["python", "/app/linux_vm_inventory.py"]
CMD ["--help"]
```

## Requirements File (`requirements.txt`)

```txt
paramiko>=2.11.0
```

## Build Script (`build.sh`)

```bash
#!/bin/bash
docker build -t vm-inventory:latest .
```

## Explanation for Young Colleague

### **Why Containerize for TeamCity?**

TeamCity build agents are often ephemeral (temporary) or shared across many projects. By packaging your script in a Docker container, you get:

- **Consistent environment** – Python version and dependencies are fixed.
- **Isolation** – No interference with other builds on the same agent.
- **Portability** – Run the same container on any agent that supports Docker.
- **Easy integration** – TeamCity has native Docker support (Docker Runner).

---

### **Step-by-Step: Prepare and Use in TeamCity**

#### **Step 1: Create the Files**

On your local machine, create a folder:

```
vm-inventory/
├── Dockerfile
├── requirements.txt
├── linux_vm_inventory.py   (your script)
└── build.sh
```

Copy the content from above into each file.

#### **Step 2: Build the Docker Image Locally (Test)**

```bash
cd vm-inventory
docker build -t vm-inventory:1.0 .
```

Test it manually:

```bash
# Test with a small subnet (using SSH key from your local machine)
docker run --rm -v ~/.ssh:/root/.ssh:ro vm-inventory:1.0 192.168.1.0/24 --user admin --key /root/.ssh/id_rsa
```

> **Note:** We mount your `~/.ssh` folder into container as read-only (`:ro`) so the container can use your private key.

#### **Step 3: Push Image to a Registry Accessible by TeamCity**

TeamCity needs to pull the image. Use your company’s Docker registry (e.g., Nexus, Artifactory, AWS ECR, Docker Hub).

```bash
# Tag for your registry
docker tag vm-inventory:1.0 myregistry.company.com/devops/vm-inventory:1.0

# Push
docker push myregistry.company.com/devops/vm-inventory:1.0
```

#### **Step 4: Configure TeamCity Build Step**

In your TeamCity project:

1. **Add a build step** → Runner type: **Docker**
2. **Settings**:
   - **Image name**: `myregistry.company.com/devops/vm-inventory:1.0`
   - **Command**: `execute` (or run custom script)
   - **Command arguments**:  
     `%subnet% --user %ssh_user% --key /ssh-key/id_rsa --output /results/inventory.csv`
   - **Working directory**: (optional)
   - **Add volume mounts**:
     - SSH key folder: `%teamcity.agent.ssh.key.path%` → `/ssh-key` (read-only)
     - Output folder: `%teamcity.build.checkoutDir%/results` → `/results` (writable)
   - **Environment variables** (if needed):
     - `subnet = 192.168.10.0/24` (parameterized)

3. **Parameterization** in TeamCity:
   - Define a build parameter `subnet` (e.g., `10.0.0.0/16`)
   - Define `ssh_user` (e.g., `admin`)
   - Store SSH key as a **secured file** in TeamCity and mount it.

> **Alternative:** Use TeamCity’s **SSH Agent** feature – mount the SSH agent socket into container. This is more secure and avoids copying keys. Example mount:  
> `-v $SSH_AUTH_SOCK:/ssh-agent -e SSH_AUTH_SOCK=/ssh-agent`  
> Then your script doesn't need `--key` parameter.

#### **Step 5: Handle Output CSV in TeamCity**

After the container runs, the `inventory.csv` file will be in the mounted output folder. TeamCity can:

- **Publish as build artifact**: Add a **Artifact path** `results/inventory.csv`
- **Display in report tab**: Write a small HTML report from CSV.
- **Trigger downstream jobs**: Use the CSV for further automation.

---

### **Detailed Explanation of Dockerfile Commands**

| Command | What it does |
|---------|---------------|
| `FROM python:3.11-slim` | Starts from a small Linux image with Python 3.11 pre-installed. |
| `WORKDIR /app` | Creates and moves into `/app` directory inside container. |
| `COPY requirements.txt .` | Copies the dependency file from your host to container. |
| `RUN pip install ...` | Installs Paramiko (pure Python, no compilation needed). |
| `COPY linux_vm_inventory.py .` | Copies your script. |
| `RUN chmod +x` | Makes script executable. |
| `ENTRYPOINT` | Tells container to run your script when started. |
| `CMD` | Default arguments (show help if no args given). |

---

### **How SSH Authentication Works Inside Container**

Your script needs to connect to target VMs. You have three options in TeamCity:

#### **Option A: Mount SSH Key as File (Simplest)**

- Store private key as a **TeamCity secured file** (e.g., `id_rsa`).
- In Docker step, mount it: `-v /path/to/id_rsa:/ssh-key/id_rsa:ro`
- Call script with `--key /ssh-key/id_rsa`

#### **Option B: Use SSH Agent Forwarding (Recommended for security)**

- TeamCity agent must have SSH agent running with the key loaded.
- Mount the agent socket: `-v $SSH_AUTH_SOCK:/ssh-agent -e SSH_AUTH_SOCK=/ssh-agent`
- Script will automatically use the agent (if you remove `--key` argument). Paramiko supports SSH agent by default.

#### **Option C: Embed Key in Build (NOT RECOMMENDED)**

- Never put private key inside Docker image – it would be visible to anyone with image access.

---

### **Common Pitfalls & Solutions**

| Problem | Solution |
|---------|----------|
| Container cannot resolve hostnames of VMs | Use IP ranges instead of hostnames, or ensure DNS is correctly configured in container (`--dns` flag). |
| SSH connection timeout | Increase timeout in script or check network policies between TeamCity agent and target subnet. |
| `Permission denied (publickey)` | Verify the mounted key has correct permissions (`chmod 600`), and the target VM has the public key in `authorized_keys`. |
| No output CSV found after build | Check volume mount path: container writes to `/results`, TeamCity expects files in `%teamcity.build.checkoutDir%/results`. |
| Too many concurrent connections | Adjust `--workers` parameter (e.g., `--workers 5`). The container inherits host's network stack limits. |

---

### **Testing the Container in TeamCity (Before Full Pipeline)**

Create a simple test build configuration:

1. **Checkout** – none required.
2. **Docker step** with your image.
3. **Command arguments**: `10.0.0.0/24 --user testuser --key /ssh-key/test_key --output /results/test.csv`
4. **Volume mounts**:
   - `%teamcity.build.checkoutDir%/test_key` → `/ssh-key` (read-only)
   - `%teamcity.build.checkoutDir%/results` → `/results`
5. **Run** – then check artifacts for `test.csv`.

---

### **Final Notes for Production**

- **Tag versions**: Use semantic versioning for your image (e.g., `1.0.0`). Never use `latest` in production.
- **Scan policies**: Ensure you have permission to scan the subnet. Some security teams may require whitelisting the TeamCity agent IP.
- **Resource limits**: Set container CPU/memory limits in TeamCity to avoid overloading the build agent.
- **Logging**: The script prints to stdout – TeamCity captures that in build logs. For sensitive data (e.g., IPs), consider masking.

Now you have a fully containerized solution that can run in TeamCity on any subnet you specify, producing a clean CSV inventory of all Linux VMs.
