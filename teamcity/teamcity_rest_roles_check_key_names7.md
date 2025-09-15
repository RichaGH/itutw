# Вопрос
хорошо. Ты просмотрел каталог с выгрузкой и обнаружил что несколько group key были ранее созданы с нарушением стандарта  наименования ключей. имена ключей должны иметь вид один или несколько заглавных букв и символ подчеркивания за которым следуют заглавные английские буквы RW. Однако есть несколько групп в которых пропущен символ подчеркивания.Теперь необходимо написать дополнительный bash скрипт который пройдет по всем файлам в каталоге выгрузки добавит символ подчеркивания там где он был пропущен в имени ключа группы.
# Вопрос
Good. You looked through the download directory and found that several group keys were previously created in violation of the key naming standard. The key names must have the form of one or more capital letters and an underscore followed by the capital English letters RW. However, there are several groups in which the underscore character is omitted.Now you need to write an additional bash script that will go through all the files in the upload directory and add an underscore where it was omitted in the group key name.

# Рассуждения

We are going to assume that the group key naming standard is: one or more capital letters, then an underscore, then the letters "RW" (also capital).
However, there are some keys that have the letters "RW" without the preceding underscore.

Steps:

We'll iterate over each JSON file in the upload directory.

For each file, we'll read the current group key.

We'll check if the key ends with "RW" but does not have an underscore before "RW".

If it doesn't have the underscore, we'll add an underscore right before "RW".

We'll update the JSON file with the new key and also update the filename accordingly.

Example:
Old key: "DEVRW" -> should become "DEV_RW"
But note: the standard says the part after underscore is "RW", so we are just adding an underscore.

However, note that the standard requires the key to end with "RW" (after underscore). So we are only concerned with keys that have "RW" at the end without an underscore.

Let's break down the key pattern:
The key should be in the form: [A-Z]+_RW
But we have some keys that are [A-Z]+RW (without underscore).

We'll check for keys that match:

They consist of one or more capital letters and then "RW" at the end.

They do not have an underscore.

We'll use a regular expression: ^[A-Z]+RW$

But note: what if there are keys that have "RW" in the middle? We are only concerned with the end.

So we want to change:
"DEVRW" -> "DEV_RW"
"QARW" -> "QA_RW"

However, note that the key might have other patterns. We only want to fix those that are missing the underscore before the trailing "RW".

We'll use sed or awk to process the JSON file and update the key.

Also, we must update the filename if the key changes.

Steps in script:

For each file in the upload directory with .json extension:

Extract the current key from the JSON file (we can use jq or grep, but since we are in bash and might not have jq, we'll use grep and sed)

Alternatively, we can use a Python one-liner for more robust JSON parsing, but the requirement is bash.

Since we are not sure if jq is available, and to keep it pure bash, we'll use grep and sed, but note that this might be fragile.

Alternatively, we can use a while loop that reads the file and looks for the "key" field.

Let's assume the JSON structure is:
{
"groupKey": "DEVRW",
... other fields ...
}

We can extract the current key with:
current_key=$(grep '"groupKey":' "$file" | sed -n 's/."groupKey": "
[
"
]
?
[ 
"
 ]?"./\1/p')

Then check if the key matches the pattern of having one or more capital letters and then "RW" at the end without underscore.

We can use a regex in bash: if [[ $current_key =~ ^[A-Z]+RW$ ]]

Then new_key will be: ${current_key%RW}_RW

Example: current_key="DEVRW", then remove "RW" from the end -> "DEV", then add "_RW" -> "DEV_RW"

Then we need to:

Replace the old key with the new key in the JSON file.

Rename the file from old_key.json to new_key.json.

However, note that the file name might not exactly be the key (because we used safe_filename function). But in our previous script, we used the groupKey as the base for the filename.

We must be cautious: the file might have been named with the safe_filename function, which replaced some characters. But in our case, the keys are only capital letters and we are adding an underscore, which is safe.

Let's assume the filename is the same as the groupKey with .json extension.

But note: the safe_filename function replaced spaces and slashes. Our keys don't have those, so we assume the filename is exactly the groupKey with .json.

However, to be safe, we'll get the current key from the JSON content, not from the filename.


Steps for each file:

file="path/to/file.json"

current_key=$(grep '"groupKey":' "$file" | sed -n 's/."groupKey": "
[
"
]
?
[ 
"
 ]?"./\1/p')
~~~
if [[ $current_key =~ ^[A-Z]+RW$ ]]; then
new_key="${current_key%RW}_RW"

text
    # Update the JSON file: replace the old key with the new key
    sed -i "s/\"groupKey\": \"$current_key\"/\"groupKey\": \"$new_key\"/g" "$file"

    # Also update the "name" field if it exists and is the same as the old key?
    # According to the standard, the name might be the same as the key. We'll update the name as well if it matches the old key.
    sed -i "s/\"name\": \"$current_key\"/\"name\": \"$new_key\"/g" "$file"

    # Now rename the file
    old_filename="$file"
    new_filename="$(dirname "$file")/${new_key}.json"

    mv "$old_filename" "$new_filename"
fi
~~~
But note: what if the "name" field is different? We are only updating if it exactly matches the old key.

Alternatively, we might have to update the name field only if it was the same as the key? Or maybe the name field should be updated to the new key as well?

Let's assume that the name field should follow the same standard and we want to update it if it was exactly the old key.

However, the requirement only says the key must be fixed. But typically the name might be the same.

We'll update the name field only if it is exactly the old key.

But note: the JSON file might have the name field as a different value. We don't want to change that if it's not the same as the key.

Alternatively, we might not update the name field at all? The requirement is about the key.

Let's stick to updating only the key in the JSON, and then if the name field is the same as the key, we update it too.

We can do two separate sed commands: one for the key and one for the name.

However, if the name field is not the same as the key, we leave it.

But wait: the standard might require the name to be the same as the key? We don't know. Let's assume we only fix the key and then set the name to the new key if it was previously the old key.

Alternatively, we might not change the name field at all. The requirement is about the key.

Let's clarify: the requirement says "key names must have the form...". So we are only required to fix the key.

However, in the JSON structure, we have both "key" and "name". We'll update the key and then if the "name" field is exactly the old key, we update it to the new key.

Steps for updating the JSON:

We'll do:
sed -i "s/"groupKey": "$current_key"/"groupKey": "$new_key"/g" "$file"
# Now, if the name field is the same as the old key, we change it too.
sed -i "s/"name": "$current_key"/"name": "$new_key"/g" "$file"

But note: the sed command might match other fields if they contain the same string. We are relying on the exact format of the line.

Alternatively, we can use a JSON-aware tool like jq, but we are trying to avoid dependencies.

Since we are in a controlled environment (the files were generated by our script) and the JSON is pretty-printed, we can assume that the lines we are changing are exactly the ones we want.

Let's write the script accordingly.

However, note that the JSON might have multiple occurrences of the old key? We are only changing the groupKey and name fields.

We'll use the exact match for the line containing "groupKey" and then the old key.

Let's code accordingly.

We'll loop over each JSON file in the upload directory.

Important: We must be careful with sed and special characters in the key. Since the key is only capital letters and we are adding an underscore, we don't have to worry about special regex characters in the key.

Let's write the script.

# Ответ

# Group Key Standardization Script

Here's a bash script that will fix group key names by adding missing underscores according to the naming standard:

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
    
    # Check if the key follows the pattern: one or more capital letters followed by RW without underscore
    if [[ "$current_key" =~ ^[A-Z]+RW$ ]]; then
        # Extract the prefix (everything before RW)
        prefix="${current_key%RW}"
        
        # Create the new key with underscore
        new_key="${prefix}_RW"
        
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
   - Uses regex to identify keys that match the pattern of one or more capital letters followed by "RW" without an underscore
   - Example: "DEVRW" matches, but "DEV_RW" or "devRW" don't

4. **Key Transformation**:
   - For keys that need fixing, extracts the prefix (everything before "RW")
   - Adds an underscore between the prefix and "RW"
   - Example: "DEVRW" becomes "DEV_RW"

5. **JSON File Updates**:
   - Uses Python to parse and update the JSON content
   - Updates both the `groupKey` field and the `name` field (if it matches the old key)

6. **File Renaming**:
   - Renames the file to match the new standardized key

## Usage:

1. Make sure you have Python 3 installed on your system
2. Run the script:
   ```bash
   ./fix_group_keys.sh
   ```
3. The script will process all JSON files in the `group_roles_export` directory

## Example:

Before running the script:
- Files: `DEVRW.json`, `QARW.json`, `PRODRW.json`
- Content: `{"groupKey": "DEVRW", "name": "DEVRW", ...}`

After running the script:
- Files: `DEV_RW.json`, `QA_RW.json`, `PROD_RW.json`
- Content: `{"groupKey": "DEV_RW", "name": "DEV_RW", ...}`

## Notes:

1. The script only processes keys that exactly match the pattern of one or more capital letters followed by "RW" without an underscore
2. Keys that already have an underscore or don't end with "RW" are left unchanged
3. The script updates both the filename and the content of the JSON files
4. A backup of the original files is not created - you might want to add this feature for production use

This script will ensure that all group keys in your upload directory follow the required naming standard of having an underscore before the "RW" suffix.
