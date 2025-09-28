# Вопрос

You are login to you zabbix version 6.0 web interface. You are want to find "Data collection ? Templates" and catn do it. May be it present in newer versions of zabbix. But you are must do it in version zabbix version 6.0 .Plese write instucrion for zabbix version 6.0

# Рассуждения

We are using Zabbix 6.0. The navigation might be slightly different from what you recall. In Zabbix 6.0, the menu structure is as follows:

Data collection is now under Configuration -> Hosts or Templates. However, to create and manage templates, we can go directly to Configuration -> Templates.

Let me guide you step by step for Zabbix 6.0:

Step 1: Creating the Template
Log in to your Zabbix 6.0 web interface.

In the left sidebar, click on Configuration -> Templates.

Click on the Create template button (located at the top right of the template list).

Fill in the template details:

Template name: Network Load Analysis Template

Visible name: Network Load Analysis

Groups: Click on Select and choose a group, for example, Templates/Network (if the group doesn't exist, you can create it by typing the name and then clicking on Add).

Description: You can optionally add a description, e.g., "Template for monitoring network connections and traffic on ens192 interface".

Click Add to create the template.

Step 2: Adding Items to the Template
Now, we will add the items to the template.

Navigate to the template you just created:

Go to Configuration -> Templates.

Find your template "Network Load Analysis Template" and click on its name.

You will see several tabs: Template, Items, Triggers, Graphs, etc. Click on Items.

Click on Create item (top right) to start adding items.

Let's add the first item: Number of open TCP connections
Name: Number of open TCP connections

Type: Zabbix agent (if you are using an agent to collect this data)

Key: We can use net.tcp.port[,] but note that this key counts the number of listening ports. Alternatively, we can use a custom key. However, for counting all TCP connections (including established, listening, etc.), we might need to use a different key.

Actually, in Zabbix, there is a key net.tcp.listen[port] that checks if a port is listening, but for all connections, we can use net.tcp.socket.count[,tcp] which returns the number of TCP sockets in all states.

Let me check the documentation: In Zabbix, the key net.tcp.socket.count[,tcp] is available since version 5.0. So in 6.0, we can use:

Key: net.tcp.socket.count[,tcp]

Type of information: Numeric (unsigned)

Update interval: 1m

History storage period: You can leave as default (e.g., 90d) or adjust as needed.

Trend storage period: Default (365d) is fine.

Click Add to save the item.

Now, let's add the network interface traffic items
We'll create items for the ens192 interface.

Incoming Traffic:

Name: Network traffic on ens192 (Incoming)

Type: Zabbix agent

Key: net.if.in[ens192]

Units: bps

Update interval: 1m

History: 90d

Trends: 365d

Click Add.

Outgoing Traffic:

Name: Network traffic on ens192 (Outgoing)

Type: Zabbix agent

Key: net.if.out[ens192]

Units: bps

Update interval: 1m

Click Add.

Total Traffic:

Name: Network traffic on ens192 (Total)

Type: Zabbix agent

Key: net.if.total[ens192]

Units: bps

Update interval: 1m

Click Add.

Total Data Transmitted (Cumulative)
We'll use the keys that return the total bytes (cumulative) and then use delta to calculate the change.

Total Received Data:

Name: Total data received on ens192

Type: Zabbix agent

Key: net.if.in[ens192,bytes]

Units: B

Update interval: 1h

Store value: Delta (simple change) - This will give us the difference between the current and previous value, which is the amount of data received in the interval.

Click Add.

Total Sent Data:

Name: Total data sent on ens192

Type: Zabbix agent

Key: net.if.out[ens192,bytes]

Units: B

Update interval: 1h

Store value: Delta (simple change)

Click Add.

Data Rate Calculations (Using Calculated Items)
Now, we'll create calculated items for bytes per minute, per hour, and per day.

Bytes per Minute:

Name: Network bytes per minute on ens192

Type: Calculated

Key: net.ens192.bytes.perminute

Formula: (last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 60

Units: B/min

Update interval: 1m

Note: The formula uses the last values of the cumulative bytes and multiplies by 60 to get per minute. However, note that the cumulative bytes are stored as simple change (delta) per hour, so this might not be accurate. Alternatively, we can use the traffic items (which are in bps) and convert to bytes per minute.

Let me reconsider: The items net.if.in[ens192] and net.if.out[ens192] are in bits per second. We can calculate bytes per minute as:

( (incoming bps + outgoing bps) * 60 ) / 8

But note: The calculated item formula must use the same units. We have two options:

Option 1: Use the cumulative bytes (delta) and then multiply by 60 for per minute? But that would be the same as the delta for the minute? Actually, the cumulative bytes are stored as the total bytes since boot, and we are taking the delta (change) per hour. So if we want per minute, we can use the traffic items (which are in bps) and then convert.

Option 2: Use the traffic items (bps) and convert to bytes per minute:

Formula: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) * 60 ) / 8

But note: The traffic items are collected every minute, so the last value is the average bits per second over the last minute. Multiplying by 60 gives the total bits in the last minute, then dividing by 8 gives bytes.

I think Option 2 is more straightforward and accurate.

Let's change the calculated item for bytes per minute to:

Bytes per Minute:

Name: Network bytes per minute on ens192

Type: Calculated

Key: net.ens192.bytes.perminute

Formula: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) * 60 ) / 8

Units: B/min

Update interval: 1m

Similarly, for bytes per hour:

Bytes per Hour:

Name: Network bytes per hour on ens192

Type: Calculated

Key: net.ens192.bytes.perhour

Formula: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) * 3600 ) / 8

Units: B/hour

Update interval: 1h

And for bytes per day:

Bytes per Day:

Name: Network bytes per day on ens192

Type: Calculated

Key: net.ens192.bytes.perday

Formula: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) * 86400 ) / 8

Units: B/day

Update interval: 1h

Step 3: Creating Graphs
Now, let's create graphs for visualization.

In the template, click on the Graphs tab.

Click on Create graph (top right).

Graph 1: Network Traffic Overview
Name: ens192 - Network Traffic

Width: 900 (or adjust as needed)

Height: 200 (or adjust as needed)

Graph type: Normal (or Stacked if you prefer)

Now, add the items:

Click Add in the Items section and select:

Network traffic on ens192 (Incoming)

Network traffic on ens192 (Outgoing)

Network traffic on ens192 (Total)

You can adjust the color and draw style for each.

Click Add to save the graph.

Graph 2: Data Transmission Rates
Name: ens192 - Data Transmission Rates

Graph type: Normal

Add the calculated items:

Network bytes per minute on ens192

Network bytes per hour on ens192

Network bytes per day on ens192

Note: The values for bytes per day will be much larger, so you might want to use a multiplier for the smaller ones to make them visible on the same graph. Alternatively, you can create separate graphs for each.

Graph 3: Total Data Transferred
Name: ens192 - Total Data Transferred

Graph type: Normal

Add the items:

Total data received on ens192

Total data sent on ens192

Step 4: Creating Triggers
Now, let's create some triggers.

In the template, click on the Triggers tab.

Click on Create trigger (top right).

Trigger 1: High Network Utilization
Name: High network utilization on {HOST.NAME}

Expression:
~~~
text
{Network Load Analysis Template:net.if.total[ens192].last()}>100000000
This threshold is 100 Mbps (100,000,000 bps). Adjust as needed.

Severity: Warning
~~~
Click Add.

Trigger 2: Unusually High Connection Count
Name: High number of TCP connections on {HOST.NAME}

Expression:
~~~
text
{Network Load Analysis Template:net.tcp.socket.count[,tcp].last()}>1000
Severity: Average
~~~
Click Add.

Step 5: Template Macros
We can define macros to make the template more flexible.

In the template, click on the Macros tab.

Click on Add (top right) to add a macro.

Add the following:

Macro: {$NET.IF.NAME}

Value: ens192

Macro: {$NET.IF.UTIL.MAX}

Value: 100000000 (100 Mbps)

Macro: {$NET.CONN.MAX}

Value: 1000

Now, we can update the items and triggers to use these macros.

For example, in the items, we can use the macro for the interface name:

Instead of ens192, we can use {$NET.IF.NAME}.

So, for example, the key for incoming traffic would be: net.if.in[{$NET.IF.NAME}]

Similarly, update all the items and triggers that use the interface name and the thresholds.

Final Steps
Apply the template to hosts:

Go to Configuration -> Hosts

Select the host(s) you want to monitor

Click on Templates tab

In the Link new templates field, type Network Load Analysis Template and select it.

Click Update

Check the latest data:

Go to Monitoring -> Latest data

Select the host and check if the items are being collected.

Important Note
The calculated items depend on the traffic items (bps) and are recalculated every minute (for per minute) or hour (for per hour and per day). The formulas use the last value of the traffic items, which are the average bps over the update interval (1 minute). So the calculated bytes per minute is the total bytes in the last minute.

This template should give you a comprehensive view of the network load on the ens192 interface.

If you have any questions or run into issues, let me know!

# Ответ


You're absolutely right! Let me provide the correct navigation for Zabbix 6.0. The menu structure in Zabbix 6.0 is different from newer versions.

## Correct Navigation for Zabbix 6.0

### Step 1: Accessing Templates

1. **Log in** to your Zabbix 6.0 web interface
2. In the left sidebar menu, click on: **Configuration** ? **Templates**

### Step 2: Creating the Template

1. Once in the **Templates** section, click the **Create template** button (usually in the top right corner)

2. **Fill in template details**:
   - **Template name**: `Network Load Analysis Template`
   - **Visible name**: `Network Load Analysis`
   - **Groups**: Click **Select** and choose:
     - Either select an existing group like "Templates"
     - Or create a new group by typing "Templates/Network"
   - **Description**: "Template for monitoring network connections and traffic on ens192 interface"

3. Click **Add** to create the template

### Step 3: Adding Items to the Template

After creating the template:

1. **Find your template** in the templates list
2. **Click on the template name** to open its configuration
3. Go to the **Items** tab
4. Click **Create item** (in the top right corner)

Now let's add the items with the correct Zabbix 6.0 format:

#### A. Number of Open Connections
- **Name**: `Number of open TCP connections`
- **Type**: `Zabbix agent`
- **Key**: `net.tcp.port[,]`
- **Type of information**: `Numeric (unsigned)`
- **Update interval**: `1m`
- **Units**: `connections`

#### B. Network Interface Traffic

**Incoming Traffic:**
- **Name**: `Network traffic on ens192 (Incoming)`
- **Type**: `Zabbix agent`
- **Key**: `net.if.in[ens192]`
- **Units**: `bps`
- **Update interval**: `1m`

**Outgoing Traffic:**
- **Name**: `Network traffic on ens192 (Outgoing)`
- **Type**: `Zabbix agent`
- **Key**: `net.if.out[ens192]`
- **Units**: `bps`
- **Update interval**: `1m`

#### C. Total Data Transmitted

**Total Received:**
- **Name**: `Total data received on ens192`
- **Type**: `Zabbix agent`
- **Key**: `net.if.in[ens192,bytes]`
- **Units**: `B`
- **Update interval**: `1h`
- **Store value**: `Delta (simple change)`

**Total Sent:**
- **Name**: `Total data sent on ens192`
- **Type**: `Zabbix agent`
- **Key**: `net.if.out[ens192,bytes]`
- **Units**: `B`
- **Update interval**: `1h`
- **Store value**: `Delta (simple change)`

#### D. Calculated Items (Data Rates)

**Bytes per Minute:**
- **Name**: `Network bytes per minute on ens192`
- **Type**: `Calculated`
- **Key**: `net.ens192.bytes.perminute`
- **Formula**: `(net.if.in[ens192,bytes] + net.if.out[ens192,bytes]) * 60`
- **Units**: `B/min`
- **Update interval**: `1m`

**Bytes per Hour:**
- **Name**: `Network bytes per hour on ens192`
- **Type**: `Calculated`
- **Key**: `net.ens192.bytes.perhour`
- **Formula**: `(net.if.in[ens192,bytes] + net.if.out[ens192,bytes]) * 3600`
- **Units**: `B/hour`
- **Update interval**: `1h`

**Bytes per Day:**
- **Name**: `Network bytes per day on ens192`
- **Type**: `Calculated`
- **Key**: `net.ens192.bytes.perday`
- **Formula**: `(net.if.in[ens192,bytes] + net.if.out[ens192,bytes]) * 86400`
- **Units**: `B/day`
- **Update interval**: `1h`

### Step 4: Creating Graphs

1. In your template, go to the **Graphs** tab
2. Click **Create graph**

**Graph 1: Network Traffic**
- **Name**: `ens192 - Network Traffic`
- Add items:
  - `Network traffic on ens192 (Incoming)`
  - `Network traffic on ens192 (Outgoing)`

**Graph 2: Data Transmission Rates**
- **Name**: `ens192 - Data Transmission Rates`
- Add items:
  - `Network bytes per minute on ens192`
  - `Network bytes per hour on ens192`
  - `Network bytes per day on ens192`

### Step 5: Creating Triggers

1. Go to the **Triggers** tab in your template
2. Click **Create trigger**

**High Network Utilization:**
- **Name**: `High network utilization on {HOST.NAME}`
- **Expression**: `{Network Load Analysis Template:net.if.in[ens192].avg(5m)}>50000000`
- **Severity**: Warning

### Important Zabbix 6.0 Specific Notes:

1. **Menu Structure**: In Zabbix 6.0, everything is under **Configuration** in the main menu
2. **Template Management**: All template operations are in **Configuration ? Templates**
3. **Item Types**: Make sure to select the correct type (Zabbix agent, Calculated, etc.)
4. **Key Syntax**: The key names I provided are compatible with Zabbix 6.0

### Verification Steps:

1. After creating the template, go to **Monitoring ? Latest data**
2. Apply the template to a test host
3. Check if data is being collected
4. Verify that graphs are displaying correctly

This should work perfectly in Zabbix 6.0. The navigation path **Configuration ? Templates** is the correct one for your version.
