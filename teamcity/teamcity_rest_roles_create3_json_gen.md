# Вопрос
Good. now you need to write an additional script that will take the directory name from the jcat_name constant (defined at the beginning of the script)
to verify that the directory exists. 
If this is not the case, then the directory must be created. 
navigate to the specified folder.
Then the script should take the names of the groups from the GROUPS_FILE file and create a separate "$groupName" file with the json extension for each group name (file element). 
In this file, you must put all the parameters of the rest api call to create a group in teamcity in json format (except for the administrator's username and password)

# Рассуждения
We are going to create a script that:

Checks for a directory (from a constant) and creates it if it doesn't exist.

Changes to that directory.

Reads group names from a file (GROUPS_FILE).

For each group, creates a JSON file (named <groupName>.json) that contains the parameters for the REST API call to create a group in TeamCity.

Note: The JSON structure for creating a group in TeamCity via REST API is:
{
"key": "group_key",
"name": "Group Name"
}

However, note that in the previous script we used:
-d "{"key": "$group_name", "name": "$group_name"}"

We are going to output a JSON file for each group that can be used in a curl request with the -d @filename.json option.

Steps in the script:

Set the constant for the directory (jcat_name) and the GROUPS_FILE.

Check if the directory exists, if not, create it and then navigate into it.

Read the GROUPS_FILE (which is in the parent directory, so we use the original path) and for each group name, create a JSON file.

Let's write the script accordingly.

# Ответ

# TeamCity Group Configuration Generator Script

```bash
#!/bin/bash

# Configuration constants
JCAT_NAME="teamcity_groups"  # Directory name for group configurations
GROUPS_FILE="groups.txt"     # File containing group names (one per line)

# Check if groups file exists
if [[ ! -f "$GROUPS_FILE" ]]; then
    echo "Error: Groups file $GROUPS_FILE not found!"
    exit 1
fi

# Check if directory exists, create if it doesn't
if [[ ! -d "$JCAT_NAME" ]]; then
    echo "Creating directory: $JCAT_NAME"
    mkdir -p "$JCAT_NAME"
    if [[ $? -ne 0 ]]; then
        echo "Error: Failed to create directory $JCAT_NAME"
        exit 1
    fi
fi

# Navigate to the directory
cd "$JCAT_NAME" || { echo "Error: Could not navigate to directory $JCAT_NAME"; exit 1; }

# Read each group from the file and create JSON configuration
while IFS= read -r group_name; do
    # Skip empty lines
    [[ -z "$group_name" ]] && continue
    
    # Create filename (replace spaces with underscores)
    filename="${group_name// /_}.json"
    
    echo "Creating configuration for group: $group_name"
    
    # Create JSON content for TeamCity group creation API
    cat > "$filename" << EOF
{
    "key": "$group_name",
    "name": "$group_name",
    "description": "Automatically created group for $group_name",
    "roles": {
        "role": []
    }
}
EOF
    
    if [[ $? -eq 0 ]]; then
        echo "Successfully created configuration file: $filename"
    else
        echo "Error creating configuration file for group: $group_name"
    fi

done < "../$GROUPS_FILE"

echo "All group configuration files have been created in the $JCAT_NAME directory."
```

## Explanation of the Script:

1. **Configuration Constants**:
   - `JCAT_NAME`: Defines the directory where JSON configuration files will be stored
   - `GROUPS_FILE`: Specifies the file containing group names

2. **Directory Handling**:
   - Checks if the specified directory exists
   - Creates the directory if it doesn't exist
   - Navigates to the directory for file creation

3. **Group Processing**:
   - Reads each group name from the input file
   - Skips empty lines to handle malformed input files
   - Creates a filename by replacing spaces with underscores

4. **JSON Configuration Creation**:
   - Generates a JSON file for each group with the proper structure for TeamCity's REST API
   - Includes key, name, description, and an empty roles array
   - Uses a heredoc (`<< EOF`) for clean JSON formatting

5. **Error Handling**:
   - Checks for successful directory creation
   - Verifies successful navigation to the directory
   - Confirms successful file creation

## Usage Instructions:

1. Create a `groups.txt` file with group names (one per line):
   ```
   developers
   quality-assurance
   product-managers
   ```

2. Run the script:
   ```bash
   ./generate_group_configs.sh
   ```

3. The script will create a `teamcity_groups` directory with JSON files:
   - `developers.json`
   - `quality-assurance.json`
   - `product-managers.json`

4. Each JSON file contains the configuration needed to create the group via TeamCity's REST API:
   ```json
   {
       "key": "developers",
       "name": "developers",
       "description": "Automatically created group for developers",
       "roles": {
           "role": []
       }
   }
   ```

## Integration with TeamCity API:

To use these configuration files with curl, you would run commands like:
```bash
curl -X POST "https://your-teamcity-instance.example.com/app/rest/userGroups" \
  -u "username:password" \
  -H "Content-Type: application/json" \
  -d @teamcity_groups/developers.json
```

This script provides a reusable way to generate TeamCity group configurations that can be version-controlled and easily modified before applying them to your TeamCity instance.
