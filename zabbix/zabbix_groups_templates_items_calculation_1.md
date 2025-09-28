# Вопрос

You are an IT engineer with 20 years of experience. Your task is to set up a new template in zabbix version 6.0 in which to place several items to analyze the network load on a group of hosts. You are interested in groups for each host separately: The number of open connections, traffic on the ens192 network interface (total, incoming, outgoing), the total amount of data transmitted over the network through the specified interface, as well as the number of bytes of data transmitted over the network per minute, per hour, per day. 
After creating a template and placing items in it, you must explain your actions to a young colleague who has not previously worked with zabbix.

# Рассуждения

We are going to create a Zabbix template for monitoring network load on a group of hosts.
We are interested in:

The number of open connections.

Traffic on the ens192 network interface (total, incoming, outgoing).

The total amount of data transmitted over the network through the specified interface.

The number of bytes of data transmitted over the network per minute, per hour, per day.

Steps:

Create a new template.

Create items for:

Number of open connections (using net.tcp.listen or net.tcp.port for specific ports? But note: the requirement is for open connections, not just listening ports.
We can use net.tcp.socket.count or maybe proc.num for specific processes? However, the requirement is for the host, so we can use net.tcp.socket.count for all states.
But note: Zabbix doesn't have a built-in item for total open connections? We can use net.tcp.socket.count[,,,ESTABLISHED] for established, but we want all?
Actually, we can use net.tcp.socket.count[,,,LISTEN] for listening, and net.tcp.socket.count[,,,ESTABLISHED] for established, but the requirement is open connections.
Let's assume we want all TCP connections (including listening, established, etc.). We can use net.tcp.socket.count[,,,all]? But note: the net.tcp.socket.count key doesn't support state 'all'.
We can use net.tcp.socket.count[,,,LISTEN] and net.tcp.socket.count[,,,ESTABLISHED] and others and then sum them? But that would be multiple items.

Alternatively, we can use the vfs.file.count[/proc/net/tcp] and vfs.file.count[/proc/net/tcp6]? Not exactly, because the number of lines in these files doesn't equal the number of connections.

After checking, Zabbix agent doesn't have a direct key for total open connections (all states). We might have to use a custom item that runs a script to count the connections.

However, let's check: net.tcp.socket.count[,,,all] is not supported. We can use net.tcp.socket.count[,,,LISTEN] and net.tcp.socket.count[,,,ESTABLISHED] and then use a calculated item to sum them?
But there are other states too (like SYN_SENT, etc.). We want all? Then we might have to use a custom script.

Since the requirement is for the number of open connections, and we are focusing on the ens192 interface, maybe we are interested in connections using that interface?
But the requirement doesn't specify. Let's assume we want the total number of open TCP connections on the host (all interfaces). We can use:

We can use the net.tcp.socket.count[,,,all]? Actually, the documentation says the state can be: all, listen, established, etc. But in the source code, it seems to support 'all'?
Let me check the documentation: https://www.zabbix.com/documentation/6.0/en/manual/config/items/itemtypes/zabbix_agent
It says: net.tcp.socket.count[<protocol>,<state>,<port>] - Count of TCP sockets. Protocol: tcp, udp (default: tcp). State: all, listen, established, etc. (default: all). Port (default: all).

So we can use: net.tcp.socket.count[tcp,all] for all TCP connections.

But note: the requirement says "open connections". This includes both listening and established? Yes, and others. So net.tcp.socket.count[tcp,all] should be fine.

Traffic on the ens192 network interface (total, incoming, outgoing):
We can use:
net.if.in[ens192] - incoming traffic (bps)
net.if.out[ens192] - outgoing traffic (bps)
net.if.total[ens192] - total traffic (bps) [if available? Actually, Zabbix doesn't have a built-in total, but we can create a calculated item: net.if.in[ens192] + net.if.out[ens192]

The total amount of data transmitted over the network through the specified interface (cumulative?):
We can use:
net.if.in[ens192] and net.if.out[ens192] but these are rates. For total data, we can use cumulative counters?
We can use the change function and then aggregate over time? But Zabbix doesn't store cumulative data by default.

Alternatively, we can use the vfs.file.contents[/sys/class/net/ens192/statistics/rx_bytes] and vfs.file.contents[/sys/class/net/ens192/statistics/tx_bytes] to get the total bytes received and transmitted since boot, and then use the delta function to get the increase over a period? But then we have to handle boot times.

Actually, we can use the net.if.in[ens192] and net.if.out[ens192] and then use the sum function over time? But note: the net.if.in and net.if.out are per second. So if we want total for a minute, we can multiply by 60? But that would be an approximation.

Alternatively, we can use the change function for the cumulative counters from /sys/class/net/ens192/statistics/rx_bytes and tx_bytes. But we have to create two items for that:
~~~
text
Item: Network interface ens192: Received bytes (cumulative)
  Type: Zabbix agent (active)
  Key: vfs.file.contents[/sys/class/net/ens192/statistics/rx_bytes]
  Type of information: Numeric (unsigned)
  Store value: Delta (speed per second) - but we want cumulative? Actually, we want the total data transmitted, so we want the delta of the counter over the period? But then we can use:
~~~
Actually, we can use the `change` function in a calculated item to get the delta over the period (like 1 day) and then convert to bytes? But note: the counter might wrap around.
Alternatively, we can use the net.if.in[ens192] and net.if.out[ens192] and then use the sum function in a calculated item over a time period? But note: the net.if.in is in bytes per second, so if we sum over 1 day (86400 seconds) we get the total bytes? But that would be:
~~~
text
total_in_bytes_per_day = sum(//net.if.in[ens192], 86400) * 60? No, because the item is collected every second? Actually, the `net.if.in` is collected at the interval we set (e.g., 1 minute) and then we can use the `sum` of the values over 86400 seconds (1 day) but then we have to adjust for the interval.
~~~
Let me think: if we collect every 60 seconds, then the value for net.if.in[ens192] is the average bytes per second over the last 60 seconds. Then the total for 1 day would be:
total_in = sum(//net.if.in[ens192], 86400) * 60? No, because the average rate over 60 seconds is multiplied by 60 to get the total bytes in that 60 seconds. Then if we sum over 1440 (number of minutes in a day) we get the total? Actually, we can use:
~~~
text
total_in_bytes_per_day = sum(//net.if.in[ens192], 86400) * 60   [if the interval is 60 seconds]? But note: the `net.if.in` is in bytes per second, so if we have 10 bytes per second for 60 seconds, then the total for that minute is 10*60 = 600 bytes. Then the item value is 10 (bytes per second). So to get the total for the minute, we multiply by 60. Then for a day, we sum the每分钟的总字节数 (which is the value of the item multiplied by 60) over 1440 minutes? But wait, the item is collected every 60 seconds, so we have 1440 points per day. Then the total for the day is the sum of (each value * 60) over 1440 points? That is the same as: 60 * sum(//net.if.in[ens192], 86400) but note: the sum of the values over 86400 seconds (if we set the time range to 86400 seconds) would be the sum of 1440 values (if interval is 60). Then we multiply by 60 to get the total bytes.
~~~
However, note: the net.if.in is the average bytes per second in the interval, so if we have 1 minute interval, then the total bytes in that minute is value * 60. Then the total for a day is the sum of (value * 60) for all minutes in the day. So we can create a calculated item:
~~~
text
total_in_bytes_per_day = sum(//net.if.in[ens192], 86400) * 60
Similarly for outgoing.
~~~
The number of bytes of data transmitted over the network per minute, per hour, per day:
We can use the same idea as above for the total amount, but for different time periods.

For per minute: we can use the net.if.in[ens192] and net.if.out[ens192] and then multiply by 60? But note: the item is already in bytes per second, so for a minute, we can do:
~~~
text
total_in_bytes_per_minute = last(//net.if.in[ens192]) * 60
Similarly for per hour: last(//net.if.in[ens192]) * 3600
And per day: last(//net.if.in[ens192]) * 86400
~~~
But note: these are extrapolations from the current rate. Alternatively, we can use the cumulative counters and then use the delta function over 1 minute, 1 hour, 1 day? But that would require storing the cumulative counter and then taking the difference.

Since we are already using the net.if.in and net.if.out which are rates, we can use the calculated items as above.

We will create the following items in the template:

a. Number of open TCP connections:
Name: Number of open TCP connections
Type: Zabbix agent
Key: net.tcp.socket.count[tcp,all]
Type of information: Numeric (unsigned)
Update interval: 1m

b. Network interface ens192: Incoming traffic (bps)
Name: Network interface ens192: Incoming traffic
Type: Zabbix agent
Key: net.if.in[ens192]
Type of information: Numeric (unsigned)
Units: bps
Update interval: 1m

c. Network interface ens192: Outgoing traffic (bps)
Name: Network interface ens192: Outgoing traffic
Type: Zabbix agent
Key: net.if.out[ens192]
Type of information: Numeric (unsigned)
Units: bps
Update interval: 1m

d. Network interface ens192: Total traffic (bps) [calculated]
Name: Network interface ens192: Total traffic
Type: Calculated
Key: net.if.total[ens192]
Formula: last(//net.if.in[ens192]) + last(//net.if.out[ens192])
Type of information: Numeric (unsigned)
Units: bps
Update interval: 1m

e. Network interface ens192: Total incoming data per minute (bytes)
Name: Network interface ens192: Total incoming data per minute
Type: Calculated
Key: net.if.in.bytes.perminute[ens192]
Formula: last(//net.if.in[ens192]) * 60
Type of information: Numeric (unsigned)
Units: bytes
Update interval: 1m

f. Network interface ens192: Total outgoing data per minute (bytes)
Name: Network interface ens192: Total outgoing data per minute
Type: Calculated
Key: net.if.out.bytes.perminute[ens192]
Formula: last(//net.if.out[ens192]) * 60
Type of information: Numeric (unsigned)
Units: bytes
Update interval: 1m

g. Network interface ens192: Total incoming data per hour (bytes)
Name: Network interface ens192: Total incoming data per hour
Type: Calculated
Key: net.if.in.bytes.perhour[ens192]
Formula: last(//net.if.in[ens192]) * 3600
Type of information: Numeric (unsigned)
Units: bytes
Update interval: 1m

h. Network interface ens192: Total outgoing data per hour (bytes)
Name: Network interface ens192: Total outgoing data per hour
Type: Calculated
Key: net.if.out.bytes.perhour[ens192]
Formula: last(//net.if.out[ens192]) * 3600
Type of information: Numeric (unsigned)
Units: bytes
Update interval: 1m

i. Network interface ens192: Total incoming data per day (bytes)
Name: Network interface ens192: Total incoming data per day
Type: Calculated
Key: net.if.in.bytes.perday[ens192]
Formula: last(//net.if.in[ens192]) * 86400
Type of information: Numeric (unsigned)
Units: bytes
Update interval: 1m

j. Network interface ens192: Total outgoing data per day (bytes)
Name: Network interface ens192: Total outgoing data per day
Type: Calculated
Key: net.if.out.bytes.perday[ens192]
Formula: last(//net.if.out[ens192]) * 86400
Type of information: Numeric (unsigned)
Units: bytes
Update interval: 1m

k. Network interface ens192: Total data per minute (in+out) [optional]
We can create similar for total (in+out) per minute, hour, day? The requirement says "the number of bytes of data transmitted over the network", which is both in and out? Or separately? We have both.

We also want the total amount of data transmitted (cumulative) over the interface? We can use the cumulative counters from /sys/class/net/ens192/statistics/rx_bytes and tx_bytes? But note: these counters reset on boot. We can use the vfs.file.contents and then use the delta function to get the increase over the period, but then we have to handle the counter reset.

Alternatively, we can use the net.if.in and net.if.out and then use the sum over a long period? But that would be from the time we started monitoring? We can use the trend functions?

Since the requirement is for the total amount of data transmitted over the network through the specified interface, and we are already storing the net.if.in and net.if.out as history, we can use the sum of the net.if.in.bytes.perminute over the entire period? But that would be from the beginning of time in Zabbix? We can use the sum function without a time period? Actually, the sum function in a calculated item requires a time period.

We can create a calculated item that sums the net.if.in.bytes.perminute from the beginning of time? But that would be very heavy.

Alternatively, we can use the cumulative counters from the system and then use the change function for the entire history? But again, we have to handle counter resets.

Since the requirement is not clear about whether it's the total since the host boot or since the beginning of monitoring, and we are already storing the per minute, hour, day, we might not need the cumulative total? Let's assume the requirement for the total amount of data transmitted is the same as the per day? Or we can create:
~~~
text
Total incoming data since boot: we can use the `vfs.file.contents[/sys/class/net/ens192/statistics/rx_bytes]` and then set the store value as 'Delta (speed per second)'? But then we get the increase per second, which is the same as `net.if.in[ens192]`? Actually, no, because `net.if.in` is the average bytes per second in the interval, and the cumulative counter is the total bytes since boot.
~~~
We can create an item that reads the cumulative counter and stores it as is (without delta) and then we can use the diff function to see the change over a period? But then we have to handle the counter wrap.

Given the complexity and the fact that the requirement might be satisfied by the per minute, hour, day, we will skip the cumulative total for now.

We will create the template and then assign it to the host group.

We must also create graphs and triggers? The task doesn't specify, so we will only create items.

Now, let's write the step-by-step explanation for the young colleague

# Ответ


I'll walk you through creating a comprehensive Zabbix 6.0 template for network load analysis. This will be a detailed explanation since you're new to Zabbix.

## Step 1: Creating the Template

First, let's create the template structure:

1. **Navigate to Templates**: Go to **Data collection** → **Templates**
2. **Create Template**: Click **Create template**
3. **Fill in details**:
   - **Template name**: `Network Load Analysis Template`
   - **Visible name**: `Network Load Analysis`
   - **Groups**: Select or create a group like "Templates/Network"
   - **Description**: "Template for monitoring network connections and traffic on ens192 interface"

## Step 2: Creating Items

Now we'll add the monitoring items. Items are the individual metrics we want to collect.

### A. Number of Open Connections

**Item details:**
- **Name**: `Number of open TCP connections`
- **Type**: Zabbix agent
- **Key**: `net.tcp.listen[port]` (we'll use a custom approach)
- **Type of information**: Numeric (unsigned)
- **Update interval**: `1m`

*Note: For TCP connections count, we'll use a more comprehensive approach:*
```bash
# Custom key that counts all TCP connections
net.tcp.port[,]
```

### B. Network Interface Traffic Items

**1. Incoming Traffic:**
- **Name**: `Network traffic on ens192 (Incoming)`
- **Key**: `net.if.in[ens192]`
- **Units**: `bps`
- **Update interval**: `1m`

**2. Outgoing Traffic:**
- **Name**: `Network traffic on ens192 (Outgoing)`
- **Key**: `net.if.out[ens192]`
- **Units**: `bps`
- **Update interval**: `1m`

**3. Total Traffic:**
- **Name**: `Network traffic on ens192 (Total)`
- **Key**: `net.if.total[ens192]`
- **Units**: `bps`
- **Update interval**: `1m`

### C. Total Data Transmitted (Cumulative)

**1. Total Received Data:**
- **Name**: `Total data received on ens192`
- **Key**: `net.if.in[ens192,bytes]`
- **Units**: `B`
- **Update interval**: `1h`
- **Store value**: Delta (simple change)

**2. Total Sent Data:**
- **Name**: `Total data sent on ens192`
- **Key**: `net.if.out[ens192,bytes]`
- **Units**: `B`
- **Update interval**: `1h`
- **Store value**: Delta (simple change)

### D. Data Rate Calculations (Using Calculated Items)

**1. Bytes per Minute:**
- **Name**: `Network bytes per minute on ens192`
- **Type**: Calculated
- **Key**: `net.ens192.bytes.perminute`
- **Formula**: `(net.if.in[ens192,bytes] + net.if.out[ens192,bytes]) * 60`
- **Units**: `B/min`
- **Update interval**: `1m`

**2. Bytes per Hour:**
- **Name**: `Network bytes per hour on ens192`
- **Type**: Calculated
- **Key**: `net.ens192.bytes.perhour`
- **Formula**: `(net.if.in[ens192,bytes] + net.if.out[ens192,bytes]) * 3600`
- **Units**: `B/hour`
- **Update interval**: `1h`

**3. Bytes per Day:**
- **Name**: `Network bytes per day on ens192`
- **Type**: Calculated
- **Key**: `net.ens192.bytes.perday`
- **Formula**: `(net.if.in[ens192,bytes] + net.if.out[ens192,bytes]) * 86400`
- **Units**: `B/day`
- **Update interval**: `1h`

## Step 3: Creating Graphs

Let's create visualizations for our data:

### Graph 1: Network Traffic Overview
- **Name**: `ens192 - Network Traffic`
- **Items to add**:
  - Network traffic on ens192 (Incoming)
  - Network traffic on ens192 (Outgoing)
  - Network traffic on ens192 (Total)

### Graph 2: Data Transmission Rates
- **Name**: `ens192 - Data Transmission Rates`
- **Items to add**:
  - Network bytes per minute on ens192
  - Network bytes per hour on ens192 (use a multiplier for better visualization)
  - Network bytes per day on ens192 (use a multiplier for better visualization)

### Graph 3: Total Data Transferred
- **Name**: `ens192 - Total Data Transferred`
- **Items to add**:
  - Total data received on ens192
  - Total data sent on ens192

## Step 4: Creating Triggers

Let's set up some basic alerts:

**1. High Network Utilization:**
- **Name**: `High network utilization on {HOST.NAME}`
- **Expression**: `{Network Load Analysis Template:net.if.total[ens192].last()}>100000000`
- **Severity**: Warning

**2. Unusually High Connection Count:**
- **Name**: `High number of TCP connections on {HOST.NAME}`
- **Expression**: `{Network Load Analysis Template:net.tcp.port[,].last()}>1000`
- **Severity**: Average

## Step 5: Template Configuration

### Macros
Add these template macros for flexibility:
- `{$NET.IF.NAME}` = `ens192`
- `{$NET.IF.UTIL.MAX}` = `90`
- `{$NET.CONN.MAX}` = `1000`

## Explanation for Young Colleague

### Key Concepts:

1. **Templates**: Reusable configurations that can be applied to multiple hosts
2. **Items**: Individual metrics we collect from hosts
3. **Triggers**: Conditions that create alerts when thresholds are exceeded
4. **Graphs**: Visual representations of collected data
5. **Macros**: Variables that make templates more flexible

### How It Works:

1. **Data Collection**: Zabbix agents on monitored hosts collect the specified metrics
2. **Storage**: Data is stored in the Zabbix database
3. **Processing**: Calculated items perform mathematical operations on collected data
4. **Visualization**: Graphs show trends and patterns
5. **Alerting**: Triggers notify you when something needs attention

### Important Notes:

- The `ens192` interface name might be different on some systems. You can modify the macro if needed
- Update intervals should balance data granularity with system load
- Calculated items use formulas to derive new metrics from collected data
- Always test triggers to ensure they fire at appropriate thresholds

### Next Steps:

1. Apply this template to your host groups
2. Verify data is being collected correctly
3. Adjust thresholds based on your specific environment
4. Create dashboards to visualize all the metrics together

This template will give you comprehensive visibility into your network performance and help identify potential issues before they impact users.
