# Вопрос
Okay, the first script is working correctly. But now it needs to be upgraded so that it searches for an occurrence not by the string constant "proxy", but by the string that will be the first parameter in the command line. The rest of the code must be left unchanged.


# Рассуждения
We are going to modify the first script to take the search string from the command line.
The original script removed all string elements that did not contain the word "proxy".
Now, we want to remove all string elements that do not contain the string provided as the first command line argument.
Steps:

The script should accept the search string as the first argument, then the input file, and then optionally the output file.

We change the condition in the process_line function (or in the main processing) to check for the presence of the provided search string.

Let's adjust the code accordingly.

Note: The original script used a function process_line that used a hardcoded "proxy". We will change that to use the provided search string.

However, note that the original script had the following usage:
Usage: python script.py input_file [output_file]

Now we want:
Usage: python script.py search_string input_file [output_file]

So we will change the argument handling.

Also note: We must be cautious because the original script read the input file and processed it line by line.

We are going to change the main function to expect the search string as the first argument.

Let's update the code accordingly.

To solve this problem, we need to upgrade the existing script to use a search string provided as a command-line argument instead of the hardcoded string "proxy". The script will now remove all string elements from the "privileges" array that do not contain the specified search string, while keeping non-string elements and string elements that contain the search string unchanged.

# Ответ

To solve this problem, we need to upgrade the existing script to use a search string provided as a command-line argument instead of the hardcoded string "proxy". The script will now remove all string elements from the "privileges" array that do not contain the specified search string, while keeping non-string elements and string elements that contain the search string unchanged.

### Approach
1. **Command-Line Arguments**: The script will now accept the search string as the first command-line argument, followed by the input filename and an optional output filename.
2. **Processing Logic**: The core logic remains the same, but the condition for filtering string elements will now use the provided search string instead of the hardcoded "proxy".
3. **Error Handling**: The script includes error handling for missing or incorrect command-line arguments and file operations.

### Solution Code
```python
import sys
import json

def extract_array_string(s, start_idx):
    if start_idx >= len(s) or s[start_idx] != '[':
        return None
    stack = ['[']
    in_string = False
    escape = False
    end_index = None
    for i in range(start_idx + 1, len(s)):
        c = s[i]
        if not in_string:
            if c == '[':
                stack.append('[')
            elif c == ']':
                stack.pop()
                if len(stack) == 0:
                    end_index = i
                    break
            elif c == '"':
                in_string = True
        else:
            if escape:
                escape = False
            else:
                if c == '\\':
                    escape = True
                elif c == '"':
                    in_string = False
    if end_index is None:
        return None
    return s[start_idx:end_index+1]

def process_line(line, search_string):
    pattern = '"privileges" : ['
    pos = line.find(pattern)
    if pos == -1:
        return line
    start_idx = pos + len(pattern) - 1
    old_array_str = extract_array_string(line, start_idx)
    if old_array_str is None:
        return line
    try:
        arr = json.loads(old_array_str)
        new_arr = []
        for item in arr:
            if isinstance(item, str):
                if search_string in item:
                    new_arr.append(item)
            else:
                new_arr.append(item)
        new_array_str = json.dumps(new_arr)
        new_line = line[:start_idx] + new_array_str + line[start_idx + len(old_array_str):]
        return new_line
    except json.JSONDecodeError:
        return line

def main():
    if len(sys.argv) < 3:
        print("Usage: python script.py search_string input_file [output_file]")
        sys.exit(1)
    
    search_string = sys.argv[1]
    input_filename = sys.argv[2]
    output_filename = sys.argv[3] if len(sys.argv) >= 4 else None
    
    try:
        with open(input_filename, 'r') as f_in:
            lines = f_in.readlines()
    except Exception as e:
        print(f"Error reading input file: {e}")
        sys.exit(1)
    
    processed_lines = [process_line(line, search_string) for line in lines]
    
    if output_filename:
        try:
            with open(output_filename, 'w') as f_out:
                for line in processed_lines:
                    f_out.write(line)
        except Exception as e:
            print(f"Error writing output file: {e}")
            sys.exit(1)
    else:
        for line in processed_lines:
            print(line, end='')

if __name__ == "__main__":
    main()
```

### Explanation
1. **Command-Line Arguments**: The script now expects the search string as the first argument, followed by the input filename and an optional output filename. This allows users to specify any search string dynamically.
2. **Processing Logic**: The `process_line` function has been modified to accept the `search_string` parameter. When processing each line, it checks if a string element contains the `search_string` instead of the hardcoded "proxy".
3. **Error Handling**: The script includes try-except blocks to handle potential errors during file operations, providing informative error messages if issues occur.

This upgrade maintains the original functionality while making the script more flexible by allowing users to specify any search string via the command line. The rest of the code remains unchanged, ensuring consistency and reliability.
