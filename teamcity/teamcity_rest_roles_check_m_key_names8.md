# вопрос

хорошо. Ты просмотрел каталог с выгрузкой и обнаружил что несколько group key были ранее созданы с нарушением стандарта  наименования ключей. имена ключей должны иметь вид один или несколько заглавных букв и символ подчеркивания за которым следуют заглавные английские буквы RW. Однако есть несколько групп в которых символ подчеркивания заменен на символ минус. Теперь необходимо написать дополнительный bash скрипт который пройдет по всем файлам в каталоге выгрузки и заменит символ минус на символ подчеркивания там где он содержится в имени ключа группы
### eng

Good. You looked through the download directory and found that several group keys were previously created in violation of the key naming standard. The key names must have the form of one or more capital letters and an underscore followed by the capital English letters RW. However, there are several groups in which the underscore character is replaced by the minus character. Now you need to write an additional bash script that will go through all the files in the upload directory and replace the minus sign with an underscore where it is contained in the group key name.

# Рассуждения

We are given that some group keys have a minus sign instead of an underscore.
The required format is: one or more capital letters, then an underscore, then capital letters RW.
We are to replace any minus sign in the group key name with an underscore.

However, note: the problem says that the underscore is replaced by a minus character.
So we are looking for keys that have the pattern: one or more capital letters, then a minus, then RW.
We will change the minus to an underscore.

But note: the key might be in the filename and also inside the JSON content (in the groupKey and possibly name fields).

Steps:

Iterate over all JSON files in the upload directory.

For each file, check if the filename (without extension) contains a minus that should be an underscore.
Specifically, we are looking for the pattern: [A-Z]+-RW.json

Also, check inside the JSON file for the groupKey and name fields that might have the same issue.

However, note: the problem says the key must be in the format: [A-Z]+_RW
So we are only concerned with keys that have a minus right before the RW part.

Let's break down the pattern:

The key should start with one or more capital letters.

Then a minus (which we want to replace with underscore).

Then the string "RW".

But note: what if there are other minuses? The requirement is very specific: the key must have an underscore before RW.
So we are only going to fix the minus that is immediately before the "RW".

Approach:
We will look for keys that match the pattern: ^[A-Z]+-RW$
And we will change the minus to an underscore.

However, note: the key might be longer? The problem says the format is one or more capital letters, then underscore, then RW.
So the entire key should be in the form [A-Z]+_RW. But if there are other minuses, that would be a different violation.

Since the problem states that the underscore is replaced by a minus, we assume only one occurrence and exactly before RW.

Steps for each file:

Extract the current key from the filename (without .json)

If the current key matches the pattern [A-Z]+-RW, then we have a candidate.

Generate the new key by replacing the last occurrence of "-" before "RW" with "_". But note: the pattern is fixed, so we can do:
new_key=${current_key/-RW/_RW}

But note: what if there are multiple minuses? We are only concerned with the one that is immediately before RW.

Actually, we can use a more specific replacement: we want to replace a minus that is followed by "RW" at the end of the string.

We can do: new_key=$(echo "$current_key" | sed 's/-RW$/_RW/')

Then we need to:

Update the JSON file: change the groupKey and name (if they are equal to the current key) to the new key.

Rename the file to the new key.

Also, what if the JSON file has the old key in the groupKey or name? We should update those fields.

However, note: the problem says the group keys were created in violation. So the JSON file's groupKey and name might also have the minus.

But caution: the name might be different from the key? We are only to fix the key and the name if it matches the key?

Since the problem is about the key, we will update the groupKey field unconditionally to the new key.
For the name field, we will update it only if it was exactly the old key.

Why? Because the name might have been set to a different string (like a human-readable name) and we don't want to change that.

However, the problem says the key naming standard, so the key must be fixed. The name might be independent.

But note: in our previous script that created the groups, we set the name to the same as the key. So it's likely the same.

Let's follow: update the groupKey to the new key. For the name, if it is exactly the old key, then update to the new key.

We'll use a temporary Python script to update the JSON.

Implementation:

We'll loop over all files in the upload directory.

For each file:
filename = example: "DEV-RW.json"
current_key = "DEV-RW"
~~~
text
Check if current_key matches the pattern: one or more capital letters, then a minus, then RW at the end.
We can use a regex: [[ $current_key =~ ^[A-Z]+-RW$ ]]

If it matches, then:
    new_key = replace the minus with underscore: e.g., "DEV_RW"

    Then update the JSON file:
        groupKey -> set to new_key
        if name == current_key, then set name to new_key

    Then rename the file to new_key.json
~~~
If the current_key does not match the pattern, skip.

Let's code accordingly

# Ответ 

# Group Key Standardization Script (Hyphen to Underscore)

Here's a bash script that will fix group key names by replacing hyphens with underscores according to the naming standard:

```bash
#!/bin/bash

# Configuration
UPLOAD_DIR="group_roles_export"  # Directory containing the JSON files

# Check if upload directory exists
if [[ ! -d "$UPLOAD_DIR" ]]; then
    echo "Error: Upload directory $UPLOAD_DIR not found!"
    exit 1
fi

# Process each JSON file in the upload directory
for json_file in "$UPLOAD_DIR"/*.json; do
    # Skip if no files found
    [[ -e "$json_file" ]] || continue
    
    echo "Processing file: $(basename "$json_file")"
    
    # Extract the current group key from the filename (without .json extension)
    current_key=$(basename "$json_file" .json)
    
    # Check if the key contains a hyphen before RW
    if [[ "$current_key" =~ ^[A-Z]+-RW$ ]]; then
        # Replace hyphen with underscore
        new_key="${current_key/-/_}"
        
        echo "Fixing key: $current_key -> $new_key"
        
        # Update the JSON file content
        python3 - <<EOF
import json
import os

try:
    # Read the JSON file
    with open("$json_file", 'r') as f:
        data = json.load(f)
    
    # Update the groupKey field
    if 'groupKey' in data:
        data['groupKey'] = "$new_key"
    
    # Update the name field if it matches the old key
    if 'name' in data and data['name'] == "$current_key":
        data['name'] = "$new_key"
    
    # Write the updated data back to the file
    with open("$json_file", 'w') as f:
        json.dump(data, f, indent=2)
        
except Exception as e:
    print(f"Error updating {json_file}: {e}")
    exit(1)
EOF
        
        # Rename the file to match the new key
        new_filename="$UPLOAD_DIR/$new_key.json"
        mv "$json_file" "$new_filename"
        
        echo "Updated file: $(basename "$new_filename")"
    else
        echo "Key $current_key follows the standard, no changes needed."
    fi
done

echo "Group key standardization complete."
```

## Explanation:

1. **Directory Validation**:
   - Checks if the upload directory exists before proceeding

2. **File Processing**:
   - Iterates through all JSON files in the upload directory
   - Extracts the current group key from the filename

3. **Pattern Matching**:
   - Uses regex to identify keys that match the pattern of one or more capital letters followed by a hyphen and then "RW"
   - Example: "DEV-RW" matches, but "DEV_RW" or "dev-RW" don't

4. **Key Transformation**:
   - For keys that need fixing, replaces the hyphen with an underscore
   - Example: "DEV-RW" becomes "DEV_RW"

5. **JSON File Updates**:
   - Uses Python to parse and update the JSON content
   - Updates both the `groupKey` field and the `name` field (if it matches the old key)

6. **File Renaming**:
   - Renames the file to match the new standardized key

## Usage:

1. Make sure you have Python 3 installed on your system
2. Run the script:
   ```bash
   ./fix_group_keys_hyphen.sh
   ```
3. The script will process all JSON files in the `group_roles_export` directory

## Example:

Before running the script:
- Files: `DEV-RW.json`, `QA-RW.json`, `PROD-RW.json`
- Content: `{"groupKey": "DEV-RW", "name": "DEV-RW", ...}`

After running the script:
- Files: `DEV_RW.json`, `QA_RW.json`, `PROD_RW.json`
- Content: `{"groupKey": "DEV_RW", "name": "DEV_RW", ...}`

## Notes:

1. The script only processes keys that exactly match the pattern of one or more capital letters followed by a hyphen and then "RW"
2. Keys that already have an underscore or don't end with "RW" are left unchanged
3. The script updates both the filename and the content of the JSON files
4. A backup of the original files is not created - you might want to add this feature for production use

This script will ensure that all group keys in your upload directory follow the required naming standard by replacing hyphens with underscores before the "RW" suffix.
