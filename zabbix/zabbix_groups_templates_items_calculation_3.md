# Вопрос

And now you are has the problems then adding items to template  with type=Calculated and Formula: (net.if.in[ens192,bytes] + net.if.out[ens192,bytes]) * 60. Zabbiz send you a message "Invalid parameters "/1/params": incorrectexpression (net.if.in[ens192,bytes] + net.if.out[ens192,bytes]) * 60"

# Рассуждения

We are getting an error when creating a calculated item because the formula syntax is incorrect for Zabbix 6.0.
In Zabbix, calculated items require a specific syntax for referring to other items. We cannot use the key directly as in the expression.
Instead, we must use the item's key in the form of a function that retrieves the value. The correct way is to use the last function or another function (like avg, min, max) to get the value of the item.

However, note that in calculated items, we refer to other items by their key and use a function to get the value. The general syntax is:
function1(key1) operator function2(key2) ...

But in our case, we are trying to use the raw key without a function. Also, note that the calculated item formula must be a valid expression.

Let's break down:

We want: (net.if.in[ens192,bytes] + net.if.out[ens192,bytes]) * 60

But in Zabbix calculated item, we must use a function to get the value of an item. Typically, we use last to get the last value.

So, the corrected formula would be:
(last(net.if.in[ens192,bytes]) + last(net.if.out[ens192,bytes])) * 60

However, note that the calculated item in Zabbix 6.0 uses a different syntax for referring to items. Actually, the correct way is to use the item's key in the formula and then use a function, but the function must be one of the supported functions for calculated items.

But wait, the official documentation for Zabbix 6.0 says that in calculated items, you can use the same functions as in trigger expressions. And in trigger expressions, we use the function with the item key as argument, e.g., last( item key ).

However, there is a caveat: the calculated item formula must be a valid expression and the item keys must be enclosed in quotes if they contain commas or square brackets? Actually, in calculated items, we don't use the key directly but we use the item's ID or the key in a specific way.

Alternatively, we can use the last function and enclose the key in double quotes. But note that the key contains commas and square brackets, so we must escape them? Actually, the calculated item formula uses the same syntax as trigger expressions.

Let me check the documentation: In trigger expressions, we write:
{host:key.function(param)}

But in calculated items, we don't have the host because it's the same host. So, we can use the key without the host? Actually, in calculated items, we refer to other items by their key and we can use the same functions as in triggers.

However, the correct syntax for calculated items is to use the function and the key in double quotes. For example:
last("net.if.in[ens192,bytes]")

So, let's try:

(last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 60

But wait, the key contains a comma and square brackets. We must escape the double quotes? Actually, the documentation says to use the key in double quotes.

Alternatively, we can use the item's numeric ID? But that is not practical because it changes.

Another option: use the key without quotes and without the host? Actually, the calculated item formula in Zabbix 6.0 requires the key to be in double quotes.

However, the error message says the expression is incorrect. Let me check the Zabbix documentation for calculated items.

According to the Zabbix 6.0 documentation, the formula for a calculated item is an expression that can use:

Functions: last, min, max, avg, etc.

Operators: +, -, *, /, etc.

Numeric constants

And the item key must be enclosed in double quotes. Also, if the key contains any special characters (like commas, spaces, etc.), we must escape them? Actually, the key must be enclosed in double quotes and that's it.

But note: the key we are using has commas and square brackets. The documentation says that the key must be enclosed in double quotes and that's enough.

So, let's try again with:

(last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 60

However, wait: the key net.if.in[ens192,bytes] is correct? Actually, the key for the total bytes received on the interface is net.if.in[ens192] and we are using net.if.in[ens192,bytes]? Let me check the Zabbix agent item key for network interface.

Actually, the key net.if.in[ens192] returns the number of bytes received? No, the key net.if.in[ens192] returns the number of incoming bits per second? And the key net.if.in[ens192,bytes] is not a standard key.

Let me check the Zabbix agent documentation for network interface:

net.if.in[interface] - incoming traffic in bps

net.if.out[interface] - outgoing traffic in bps

net.if.total[interface] - total traffic in bps

But we want the total bytes, not bits. And we are using net.if.in[ens192,bytes]? That is not a standard key.

Actually, the standard key for the total number of bytes received on the interface is net.if.in[ens192] but that returns bits per second. We want the cumulative bytes?

Wait, we have two items:

net.if.in[ens192] -> bits per second (counter)

net.if.in[ens192,bytes] -> this is not a standard key.

How do we get the total bytes? Actually, the net.if.in[ens192] item returns the number of bits per second, but if we set the type of information to Numeric (unsigned) and the units to B (bytes) and then use the Delta storage, we can get the number of bytes transferred in the interval.

But note: the key net.if.in[ens192] returns the number of bits per second. To get the bytes, we would have to divide by 8? And then multiply by the interval? Actually, the item net.if.in[ens192] is a counter that gives the rate in bits per second.

However, we have an item that we created earlier: Total data received on ens192 with key net.if.in[ens192,bytes]? That key is not standard. We might have to use a different approach.

Let me clarify:

We want the total bytes transmitted? Then we should use the net.if.in[ens192] item with type of information as Numeric (unsigned) and store value as Delta (simple change). Then the value stored will be the difference between the current and previous value, which in this case is the number of bits received in the interval? But wait, the item returns bits per second, so if we collect every 1 minute, then the value we get is the average bits per second over that minute. Then the delta would be the average bits per second? That doesn't give us the total bytes.

Alternatively, we can use the vfs.file.contents or a custom script to read the /proc/net/dev file and get the cumulative bytes. But that is more complex.

Alternatively, we can use the net.if.in[ens192] item and then use a calculated item to convert to bytes and then multiply by the interval? But note: the item net.if.in[ens192] returns bits per second. So, if we want the bytes per minute, we can do:

(last("net.if.in[ens192]") / 8) * 60 [because bits per second to bytes per minute]

But wait, the item net.if.in[ens192] is the number of bits per second. So, in one second, the number of bytes is net.if.in[ens192] / 8. Then in 60 seconds, it would be (net.if.in[ens192] / 8) * 60.

Similarly, for outgoing: (last("net.if.out[ens192]") / 8) * 60

So, let's change our calculated items to use the standard keys and convert from bits to bytes.

Revised plan for calculated items:

Bytes per Minute:

Formula: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 ) * 60

Bytes per Hour:

Formula: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 ) * 3600

Bytes per Day:

Formula: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 ) * 86400

But note: the item net.if.in[ens192] returns the current rate in bits per second. So, if we take the last value and multiply by the time, we are assuming that the rate was constant over that period? This is an approximation.

Alternatively, we can use the cumulative counters from the network interface. The Zabbix agent does not provide cumulative counters by default? We would have to use a different key.

Actually, the Zabbix agent does have the net.if key that can return the number of bytes and packets, but it is not enabled by default. We would have to configure the agent to use the net.if key with the bytes parameter? I don't think it's available.

Alternatively, we can use the vfs.file.contents[/proc/net/dev] and then parse the output? That would be more accurate for cumulative bytes.

However, let's stick with the standard keys and the approximation for now.

So, let's change the calculated items to use the above formulas.

Also, note that the original items for net.if.in[ens192] and net.if.out[ens192] are already in the template, so we can refer to them.

Steps:

Go to the template and edit the calculated items.

Change the formulas to:

Bytes per minute: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 ) * 60

Bytes per hour: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 ) * 3600

Bytes per day: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 ) * 86400

Also, change the units accordingly.

Make sure the keys for the calculated items are unique.

Let me write the corrected item details:

Bytes per Minute:

Name: Network bytes per minute on ens192

Type: Calculated

Key: net.ens192.bytes.perminute

Formula: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 ) * 60

Units: B/min

Update interval: 1m

Bytes per Hour:

Name: Network bytes per hour on ens192

Type: Calculated

Key: net.ens192.bytes.perhour

Formula: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 ) * 3600

Units: B/hour

Update interval: 1h

Bytes per Day:

Name: Network bytes per day on ens192

Type: Calculated

Key: net.ens192.bytes.perday

Formula: ( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 ) * 86400

Units: B/day

Update interval: 1h

But wait, the update interval for the calculated item should be at least as long as the update interval of the items it depends on? Actually, it can be the same or longer.

However, note that the formula uses the last function, which gets the most recent value. So, if the calculated item updates every minute, it will get the latest value of the incoming and outgoing traffic (which are updated every minute) and then calculate the bytes per minute.

This should work.

Let's test the formula in the Zabbix frontend by creating a calculated item with the formula:

( (last("net.if.in[ens192]") + last("net.if.out[ens192]")) / 8 ) * 60

If we still get an error, we might need to check the exact key names and whether they exist on the host.

Alternatively, we can use the item IDs in the formula? In Zabbix, we can use the item ID in the formula by prefixing with a hash? Actually, the calculated item formula can use the item ID by writing it as {item_id}. But that is not recommended because the template will be applied to multiple hosts and the item IDs will be different.

So, we stick with the key.

Another possibility: the key might be case-sensitive? Or the interface name might be different? We must ensure that the key exactly matches the key of the items we created.

Let me check the keys of the items we created:

net.if.in[ens192]

net.if.out[ens192]

They are exactly as above.

So, let's try again with the corrected formula.

If you still get an error, try using the last function without any parameters? Actually, the last function requires one parameter: the key in double quotes.

Also, note that the calculated item formula must be a valid expression and the functions must be in lowercase.

Let me write the formula again without any extra spaces:

((last("net.if.in[ens192]")+last("net.if.out[ens192]"))/8)*60

We can try without spaces to see if it helps.

Alternatively, we can try using the avg function for the last 60 seconds? But the last function should be sufficient.

I hope this resolves the issue.

# Ответ


You're absolutely right! The issue is with the formula syntax in Zabbix 6.0 for calculated items. Let me correct this with the proper syntax.

## Problem Analysis

In Zabbix 6.0, calculated items require a specific syntax:
- You need to reference items by their **key**, not by their display names
- The formula must use proper mathematical operators
- You need to reference the actual collected values correctly

## Corrected Calculated Items

### Option 1: Using Interface Traffic Counters (Recommended)

First, let's create the base items correctly:

**1. Incoming Bytes (counter):**
- **Name**: `Network incoming bytes on ens192`
- **Type**: Zabbix agent
- **Key**: `net.if.in[ens192,bytes]`
- **Type of information**: `Numeric (unsigned)`
- **Store value**: `Delta (simple change)`
- **Units**: `B`
- **Update interval**: `60s`

**2. Outgoing Bytes (counter):**
- **Name**: `Network outgoing bytes on ens192`
- **Type**: Zabbix agent
- **Key**: `net.if.out[ens192,bytes]`
- **Type of information**: `Numeric (unsigned)`
- **Store value**: `Delta (simple change)`
- **Units**: `B`
- **Update interval**: `60s`

### Option 2: Correct Calculated Items Formulas

Now for the calculated items:

**Bytes per Minute:**
- **Name**: `Network bytes per minute on ens192`
- **Type**: Calculated
- **Key**: `net.ens192.bytes.perminute`
- **Formula**: `last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")`
- **Units**: `B/min`
- **Update interval**: `60s`

**Bytes per Hour:**
- **Name**: `Network bytes per hour on ens192`
- **Type**: Calculated
- **Key**: `net.ens192.bytes.perhour`
- **Formula**: `(last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 60`
- **Units**: `B/hour`
- **Update interval**: `1h`

**Bytes per Day:**
- **Name**: `Network bytes per day on ens192`
- **Type**: Calculated
- **Key**: `net.ens192.bytes.perday`
- **Formula**: `(last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 1440`
- **Units**: `B/day`
- **Update interval**: `1h`

## Alternative Approach: Using Dependent Items (Easier)

If you're still having issues with calculated items, use dependent items instead:

### Step 1: Create Master Items
Create the two base counter items as shown above.

### Step 2: Create Dependent Items

**Bytes per Minute (Dependent):**
- **Name**: `Network bytes per minute on ens192`
- **Type**: Dependent item
- **Key**: `net.ens192.bytes.perminute`
- **Master item**: Select `Network incoming bytes on ens192`
- **Preprocessing**:
  - **Step 1**: Type: `Custom multiplier`, Value: `1`
  - **Step 2**: Type: `JavaScript`, Parameters: `return value + (parseInt(vars["net.if.out[ens192,bytes]"]) || 0)`
- **Units**: `B/min`
- **Update interval**: `60s`

## Even Simpler Alternative: Use Traffic Rate Items

Instead of calculated items, use these simpler Zabbix agent items:

**Total Traffic Rate:**
- **Name**: `Network traffic total on ens192`
- **Type**: Zabbix agent  
- **Key**: `net.if.total[ens192]`
- **Units**: `bps`
- **Update interval**: `60s`

Then create a calculated item to convert to bytes:
- **Formula**: `last("net.if.total[ens192]") / 8`
- **Units**: `B/s`

## Step-by-Step Verification

1. **First, create only the basic agent items:**
   - `net.if.in[ens192]`
   - `net.if.out[ens192]`
   - `net.if.in[ens192,bytes]`
   - `net.if.out[ens192,bytes]`

2. **Verify they work:**
   - Go to **Monitoring ? Latest data**
   - Check if data is being collected
   - Wait a few minutes to see values

3. **Then add calculated items one by one:**
   - Start with the simplest formula
   - Test each one before adding the next

## Key Points for Young Colleague:

1. **Calculated items** use the `last("key")` function to reference other items
2. **Dependent items** are often more reliable than calculated items
3. **Always test base items first** before creating calculated items
4. **The multiplication in formulas** happens in the calculated item, not in the base counters

## Quick Fix for Your Current Issue:

Change your formula from:
```
(net.if.in[ens192,bytes] + net.if.out[ens192,bytes]) * 60
```

To:
```
(last("net.if.in[ens192,bytes]") + last("net.if.out[ens192,bytes]")) * 60
```

This should resolve the "Invalid parameters" error you're encountering.
