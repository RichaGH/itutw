# Задача: 
Генерировать скрипт который будет писать значения в лог файл в json  формате. Одно сообщение (набор случайных данных )- одна строка. Получено запросами:
1.  Превый запрос
```
скрипт на python должен формировать лог файл в json формате. Список полей:
id - целое число (последовательный инкремент на 1 для каждого следующего значения)
idata1 целое из диапазона 0-9
sdata1 строка 4 символа
ddata1 вещественное число от -32000 до +100000000
idata2 целое из диапазона 10-99
sdata2 строка 20 символов (только буквы)
ddata2 вещественное число от -320 до +100
idata3 целое из диапазона 100-999
sdata3 строка от 3 до 30 символов (возможны буквы пробелы цыфры скобки) 
ddata3 вещественное число от -1032000 до +1000

-------------
the python script should generate a log file (mylog.log) in json format. List of fields:
id is an integer (sequential increment by 1 for each subsequent value)
idata1 is an integer from the range 0-9
sdata 1 string 4 characters
ddata1 is a real number from -32000 to +100000000
idata2 is an integer from the range 10-99
sdata2 is a string of 20 characters (letters only)
ddata2 is a real number from -320 to +100
data3 is an integer from the range 100-999
sata 3 terms from 3 to 30 characters (letters, spaces, numbers, brackets are possible) 
data3 is a real number from -1032000 to +1000
```

2. i need e result file without [ and data from {} must be in each it own line
3. i need add to log new variable  (name curDateVal ) do it ater id value. This variable must contain curent date time value
4. how can i extend curDateVal adding milliseconds to it value?
5. it is necessary to change the script so that when it is restarted, it continues to calculate the id starting from id = the last value from the previous run +1

## Результаты и описание:
To ensure that the script continues calculating the `id` starting from the last value of the previous run + 1, you need to read the existing log file (if it exists) and determine the last `id` used. If the file doesn't exist, the script should start from `id = 1`.

Here’s the updated script:

```python
import json
import random
import string
from datetime import datetime
import os

def generate_random_string(length, allow_spaces=False, allow_numbers=False, allow_brackets=False):
    characters = string.ascii_letters
    if allow_spaces:
        characters += ' '
    if allow_numbers:
        characters += string.digits
    if allow_brackets:
        characters += '()[]{}'
    return ''.join(random.choice(characters) for _ in range(length))

def generate_log_entry(entry_id):
    # Get current date and time with milliseconds
    cur_date_val = datetime.now().strftime("%Y-%m-%d %H:%M:%S.%f")[:-3]  # Truncate last 3 digits for milliseconds
    log_entry = {
        "id": entry_id,
        "curDateVal": cur_date_val,  # Add current date and time with milliseconds
        "idata1": random.randint(0, 9),
        "sdata1": generate_random_string(4),
        "ddata1": round(random.uniform(-32000, 100000000), 2),
        "idata2": random.randint(10, 99),
        "sdata2": generate_random_string(20),
        "ddata2": round(random.uniform(-320, 100), 2),
        "data3": random.randint(100, 999),
        "sdata3": generate_random_string(random.randint(3, 30), allow_spaces=True, allow_numbers=True, allow_brackets=True),
        "ddata3": round(random.uniform(-1032000, 1000), 2)
    }
    return log_entry

def get_last_id(filename):
    # Check if the file exists
    if not os.path.exists(filename):
        return 0  # If the file doesn't exist, start from id = 1

    # Read the last line of the file to get the last id
    with open(filename, 'r') as log_file:
        last_line = None
        for line in log_file:
            last_line = line
        if last_line:
            last_entry = json.loads(last_line)
            return last_entry["id"]
    return 0

def generate_log_file(num_entries, filename="mylog.log"):
    # Get the last id from the existing log file
    last_id = get_last_id(filename)

    # Open the file in append mode to add new entries
    with open(filename, 'a') as log_file:
        for i in range(last_id + 1, last_id + 1 + num_entries):
            log_entry = generate_log_entry(i)
            json.dump(log_entry, log_file)  # Write each log entry as a JSON object
            log_file.write('\n')  # Add a newline after each entry

if __name__ == "__main__":
    num_entries = 10  # Number of log entries to generate
    generate_log_file(num_entries)
    print(f"Generated {num_entries} log entries in 'mylog.log'")
```

### Key Changes:
1. **`get_last_id` Function**:
   - Checks if the log file exists.
   - If the file exists, it reads the last line to determine the last `id` used.
   - If the file doesn't exist, it returns `0` to start from `id = 1`.

2. **Appending to the Log File**:
   - The script now opens the log file in append mode (`'a'`) instead of write mode (`'w'`).
   - This ensures that new entries are added to the end of the file without overwriting existing data.

3. **Continuing `id` from the Last Value**:
   - The `generate_log_file` function calculates the starting `id` as `last_id + 1`.

### Example Workflow:
1. **First Run**:
   - If `mylog.log` doesn't exist, the script starts from `id = 1`.
   - Example output:
     ```json
     {"id": 1, "curDateVal": "2023-10-25 16:45:30.123", ...}
     {"id": 2, "curDateVal": "2023-10-25 16:45:30.456", ...}
     ...
     ```

2. **Second Run**:
   - The script reads the last `id` from the file (e.g., `10`) and starts from `id = 11`.
   - Example output:
     ```json
     {"id": 11, "curDateVal": "2023-10-25 16:46:15.789", ...}
     {"id": 12, "curDateVal": "2023-10-25 16:46:16.012", ...}
     ...
     ```

### Notes:
- The script ensures that `id` values are unique and sequential across multiple runs.
- If the log file is deleted, the script will start from `id = 1` again.
- The script appends new entries to the existing log file, preserving previous data.

