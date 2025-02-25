## Вопрос
how to use iftop and nload?

## Ответ

`iftop` and `nload` are powerful command-line tools for monitoring real-time network traffic. Below are detailed instructions on how to install and use these tools:

---

### **1. Installing `iftop` and `nload`**

#### **On Ubuntu/Debian**
```bash
sudo apt update
sudo apt install iftop nload
```

#### **On CentOS/RHEL**
```bash
sudo yum install epel-release
sudo yum install iftop nload
```

#### **On macOS (using Homebrew)**
```bash
brew install iftop nload
```

---

### **2. Using `iftop`**

`iftop` displays real-time network bandwidth usage on an interface.

#### **Basic Usage**
```bash
sudo iftop -i <network_interface>
```
- Replace `<network_interface>` with the name of your network interface (e.g., `eth0`, `ens33`, `enp0s3`).
- To list available interfaces, use:
  ```bash
  ip a
  ```

#### **Keybindings**
- **`h`**: Toggle help screen.
- **`n`**: Toggle DNS resolution (show/hide hostnames).
- **`s`**: Show source host.
- **`d`**: Show destination host.
- **`t`**: Toggle display mode (line, bar, or combined).
- **`j`/`k`**: Scroll through connections.
- **`q`**: Quit `iftop`.

#### **Example**
```bash
sudo iftop -i eth0
```
- This will show a live view of network traffic on `eth0`, including:
  - Source and destination IPs/hostnames.
  - Bandwidth usage (2s, 10s, and 40s averages).

---

### **3. Using `nload`**

`nload` provides a simple, real-time view of network traffic in a graphical format.

#### **Basic Usage**
```bash
nload <network_interface>
```
- Replace `<network_interface>` with the name of your network interface.

#### **Keybindings**
- **`Up`/`Down` Arrow**: Switch between interfaces (if multiple are available).
- **`q`**: Quit `nload`.

#### **Example**
```bash
nload eth0
```
- This will display:
  - Incoming traffic (Incoming).
  - Outgoing traffic (Outgoing).
  - Current, average, min, and max bandwidth usage.

---

### **4. Advanced Usage**

#### **Filtering Traffic in `iftop`**
- Filter traffic by host or port:
  ```bash
  sudo iftop -i eth0 -f "host <ip_address>"
  sudo iftop -i eth0 -f "port <port_number>"
  ```
  Example:
  ```bash
  sudo iftop -i eth0 -f "port 9200"
  ```

#### **Customizing `nload` Display**
- Use the `-u` option to specify the unit of measurement:
  ```bash
  nload -u h   # Human-readable format (auto-scale).
  nload -u m   # Megabytes.
  nload -u k   # Kilobytes.
  ```

#### **Running `iftop` or `nload` in Batch Mode**
- Use `iftop` in batch mode to capture traffic for a specific duration:
  ```bash
  sudo iftop -i eth0 -t -s 10
  ```
  - `-t`: Text mode (no interactive display).
  - `-s 10`: Run for 10 seconds.

---

### **5. Interpreting Output**

#### **`iftop` Output**
- **Top Section**:
  - Shows overall bandwidth usage (2s, 10s, and 40s averages).
- **Middle Section**:
  - Lists active connections with source and destination IPs/hostnames.
  - Displays real-time bandwidth usage for each connection.
- **Bottom Section**:
  - Shows cumulative sent and received traffic.

#### **`nload` Output**
- **Incoming Traffic**:
  - Displays a graph of incoming data rates.
- **Outgoing Traffic**:
  - Displays a graph of outgoing data rates.
- **Statistics**:
  - Shows current, average, min, and max bandwidth usage.

---

### **6. Example Use Cases**

#### **Monitoring OpenSearch Traffic**
- Use `iftop` to monitor traffic on the OpenSearch port (default: `9200`):
  ```bash
  sudo iftop -i eth0 -f "port 9200"
  ```
- Use `nload` to monitor overall network traffic while running OpenSearch:
  ```bash
  nload eth0
  ```

#### **Identifying Bandwidth Hogs**
- Use `iftop` to identify which IPs or hosts are consuming the most bandwidth:
  ```bash
  sudo iftop -i eth0
  ```

#### **Troubleshooting Network Bottlenecks**
- Use `nload` to check if network interfaces are saturated:
  ```bash
  nload eth0
  ```

---

By using `iftop` and `nload`, you can effectively monitor and analyze network traffic on your OpenSearch nodes, helping you identify and resolve performance bottlenecks.
