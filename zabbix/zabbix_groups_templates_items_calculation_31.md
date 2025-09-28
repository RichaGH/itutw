# Вопрос

You are cant see "Store value:" in Items create|edit page. Where is it in 6.0 version of zabbix?

# Рассуждения

We are using Zabbix 6.0. The "Store value" setting might be located in a different place or have a different name.
In Zabbix 6.0, the "Store value" setting for an item is found in the item configuration under the "Preprocessing" tab.
However, note that for some item types (like Zabbix agent) that are counters, we use the "Type of information" and then the "Store value" is actually set by the "Type of information" and the "Update interval".

But wait, in Zabbix 6.0, the way to handle counters (like network bytes) is to set the "Type of information" to "Numeric (unsigned)" and then use the "Store value" as "Delta (speed per second)" or "Delta (simple change)".

However, in the item configuration form in Zabbix 6.0, the "Store value" option is not directly visible in the main tab. It is now part of the "Preprocessing" tab.

Let me clarify:

For an item that is a counter (like network interface bytes), we need to set:

Type of information: Numeric (unsigned)

Then go to the Preprocessing tab and add a preprocessing step of type "Change per second" or "Simple change".

In Zabbix 6.0, the preprocessing steps are used to replace the old "Store value" setting.

So, for the item "Total data received on ens192" (key: net.if.in[ens192,bytes]), we want to store the simple change (i.e., the difference between the current and previous value). We do this by:

In the item configuration, go to the Preprocessing tab.

Click "Add" and select preprocessing type: "Simple change"

That's it.

Similarly, for the item "Total data sent on ens192" (key: net.if.out[ens192,bytes]), we do the same.

However, note that the key net.if.in[ens192,bytes] returns the cumulative counter of bytes received. By using "Simple change" preprocessing, we store the difference between two consecutive readings, which is the number of bytes received in the interval.

Now, for the calculated items, we are using these preprocessed values (the simple change) and then multiplying by the time factor to get bytes per minute, hour, etc.

But wait, the calculated items are using the keys of the base items, which now return the delta (simple change) because of the preprocessing. So the formula for bytes per minute would be:

(last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 60

This would take the last value of the delta (which is bytes per second) and multiply by 60 to get bytes per minute.

However, note that the base items (with simple change) are storing the difference in bytes per update interval. If the update interval is 60 seconds, then the delta is the bytes per 60 seconds. So we have to adjust.

Alternatively, we can use the "Change per second" preprocessing to get bytes per second, and then the calculated item would be:

(last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 60

But note: if we use "Change per second", then the value stored is the rate in bytes per second. Then multiplying by 60 gives bytes per minute.

Let's decide:

Option 1: Use "Simple change" and update interval 60s.
Then the value stored is the number of bytes in 60 seconds. So to get bytes per minute, we don't need to multiply by 60? Actually, we do because the value is for the entire interval, but we want per minute. However, if the interval is 60s, then the value is already per minute. But if the interval changes, it breaks.

Option 2: Use "Change per second" (delta per second) and then the value is in bytes per second. Then we can multiply by 60 to get bytes per minute, by 3600 for per hour, etc.

I recommend Option 2 because it is independent of the update interval.

So, for the base counter items (net.if.in[ens192,bytes] and net.if.out[ens192,bytes]):

Set the update interval to 60s (or 1m) for frequent data.

Preprocessing: Add a step of type "Change per second"

Then the calculated items:

Bytes per minute: (last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 60

Bytes per hour: (last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 3600

Bytes per day: (last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 86400

This way, even if the update interval of the base items changes, the calculated items will still be correct because the base value is in bytes per second.

Let's update the instructions accordingly.

Revised Steps for Base Counter Items:
Total Received Data (bytes per second):
Name: Network incoming bytes on ens192 (bps)

Key: net.if.in[ens192,bytes]

Type of information: Numeric (unsigned)

Update interval: 60s

Preprocessing tab: Add a preprocessing step of type Change per second

Total Sent Data (bytes per second):
Name: Network outgoing bytes on ens192 (bps)

Key: net.if.out[ens192,bytes]

Type of information: Numeric (unsigned)

Update interval: 60s

Preprocessing tab: Add a preprocessing step of type Change per second

Calculated Items:
Bytes per Minute:
Name: Network bytes per minute on ens192

Type: Calculated

Key: net.ens192.bytes.perminute

Formula: (last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 60

Units: B/min

Update interval: 1m

Bytes per Hour:
Name: Network bytes per hour on ens192

Type: Calculated

Key: net.ens192.bytes.perhour

Formula: (last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 3600

Units: B/hour

Update interval: 1h

Bytes per Day:
Name: Network bytes per day on ens192

Type: Calculated

Key: net.ens192.bytes.perday

Formula: (last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 86400

Units: B/day

Update interval: 1h

Important Note:
The base counter items (with preprocessing) now store the bytes per second. The calculated items then convert to the desired time frame.

For the Young Colleague:
In Zabbix 6.0, the concept of "Store value" has been replaced by preprocessing steps. For counter data, we use the preprocessing step "Change per second" to get the rate per second, which is more flexible.

If you have any more issues, let me know!

# Ответ

You're right! In Zabbix 6.0, the "Store value" setting has been moved and renamed. Let me show you where to find it.

## Location in Zabbix 6.0

The "Store value" setting is now called **"History storage period"** and is located in a different place:

### For the counter items (net.if.in[ens192,bytes] and net.if.out[ens192,bytes]):

1. Go to your item configuration
2. Look for the **"Preprocessing"** tab (not the main settings)
3. Click **Add** in the Preprocessing section
4. Select preprocessing step type: **"Change per second"**

### Step-by-step configuration:

**For Total Received Data:**
- **Name**: `Total data received on ens192`
- **Type**: `Zabbix agent`
- **Key**: `net.if.in[ens192,bytes]`
- **Type of information**: `Numeric (unsigned)`
- **Units**: `B`
- **Update interval**: `60s`
- **Go to Preprocessing tab**:
  - Click **Add**
  - **Type**: `Change per second`
  - **Custom multiplier**: `1` (leave as default)
  - **Parameters**: (none needed)

**For Total Sent Data:**
- **Name**: `Total data sent on ens192`
- **Type**: `Zabbix agent`
- **Key**: `net.if.out[ens192,bytes]`
- **Type of information**: `Numeric (unsigned)`
- **Units**: `B`
- **Update interval**: `60s`
- **Go to Preprocessing tab**:
  - Click **Add**
  - **Type**: `Change per second`

## What "Change per second" does:

- **Delta (simple change)** = `Change per second` in Zabbix 6.0
- It calculates the difference between the current value and previous value
- Perfect for counter metrics that continuously increase

## Alternative approach using traffic rate items:

Since you're having issues with calculated items, here's a simpler approach using only agent items:

### Create these items instead:

**1. Incoming Traffic Rate:**
- **Name**: `Network incoming rate on ens192`
- **Type**: `Zabbix agent`
- **Key**: `net.if.in[ens192]`
- **Units**: `bps`
- **Update interval**: `60s`

**2. Outgoing Traffic Rate:**
- **Name**: `Network outgoing rate on ens192`
- **Type**: `Zabbix agent`
- **Key**: `net.if.out[ens192]`
- **Units**: `bps`
- **Update interval**: `60s`

**3. Convert to bytes per minute (calculated item):**
- **Name**: `Network bytes per minute on ens192`
- **Type**: `Calculated`
- **Key**: `net.ens192.bytes.perminute`
- **Formula**: `(last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 * 60`
- **Units**: `B/min`
- **Update interval**: `60s`

## Even Simpler - Use Built-in Totals:

**Total Traffic:**
- **Name**: `Network total traffic on ens192`
- **Type**: `Zabbix agent`
- **Key**: `net.if.total[ens192]`
- **Units**: `bps`
- **Update interval**: `60s`

Then create calculated items that convert this to bytes over different time periods.

## Key Points for Zabbix 6.0:

1. **"Store value"** is now handled through **Preprocessing**
2. **"Delta (simple change)"** = **"Change per second"** preprocessing
3. **"Delta (speed per second)"** = Different preprocessing type
4. Always test preprocessing steps to ensure they work as expected

## Quick Summary for Your Template:

1. Use `net.if.in[ens192]` and `net.if.out[ens192]` for rate monitoring
2. Use preprocessing with "Change per second" for counter items
3. Keep calculated items simple and test them one by one

This should resolve your issue with the missing "Store value" setting in Zabbix 6.0!
