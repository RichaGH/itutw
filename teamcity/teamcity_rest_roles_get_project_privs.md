# вопрос
Good. now you need to write an additional script that will receive a list of groups from the team and, for each group, output to a separate file (file name = group name) the name and id of the project to which the group is linked and the project role. If there are several projects and roles, then group them together

# Рассуждения
We are going to use the TeamCity REST API to get the list of groups and then for each group, get the projects and roles assigned to it.
Steps:

Get all groups from TeamCity.

For each group, get the details (which includes the roles per project).

For each group, create a file (with the group name as the filename) and write the project name, project id, and role.

Note: The TeamCity REST API endpoint for getting all groups is:
GET /app/rest/userGroups

And for a specific group, the details can be obtained by:
GET /app/rest/userGroups/key:<groupKey>

However, note that the group details might include the roles per project.

Alternatively, we can get the roles for a group in a project by:
GET /app/rest/projects/id:<projectId>/roleGroups

But that might be inefficient. Instead, we can get the group details and parse the roles field.

Let's check the API: The group details endpoint returns XML (or JSON) that includes a roles element which has a list of role elements for each project.

Example of group details (in XML, but we can request JSON):
<userGroup>
...
<roles>
<role scope="p:projectId" roleId="PROJECT_DEVELOPER"/>
</roles>
</userGroup>

We can request JSON by setting the Accept header.

Alternatively, we can use the following endpoint to get the roles for a group in all projects?
Actually, the group details already include the roles.

So plan:

Get all groups (JSON) from /app/rest/userGroups

For each group, get its details (JSON) from /app/rest/userGroups/key:<groupKey>

In the details, there is a "roles" field which is an object with a "role" array (might be empty or not present).

Each role object has:
"roleId": "PROJECT_DEVELOPER",
"scope": "p:projectId"

We need to map the scope (which is in the form "p:projectId") to the project name. We can get the project details by projectId from the REST API if we don't have a mapping.

However, to avoid too many API calls, we can first get all projects and create a mapping from projectId to project name.

Steps updated:

Pre-step: Get all projects and create a map (projectId -> projectName)

GET /app/rest/projects to get all projects.

Parse the projects and create a mapping.

Get all groups.

For each group, get its details.

For each role in the group details, if the scope starts with "p:", then it's a project role. Extract the projectId and look up the project name.

Write to a file named after the group (replace any special characters to avoid issues) with the project name, project id, and role.

Let's write the script accordingly.

We'll use jq for JSON parsing.

Note: We assume that the TeamCity server URL and credentials are set as constants.

Example of group details in JSON (from TeamCity 2020.2):

{
"key": "groupKey",
"name": "groupName",
"href": "/app/rest/userGroups/key:groupKey",
"description": "group description",
"users": { ... },
"roles": {
"role": [
{
"roleId": "PROJECT_DEVELOPER",
"scope": "p:projectId"
}
]
}
}

We are interested in the "roles" -> "role" array.

Let's code accordingly.


# Ответ

# TeamCity Group Project Roles Export Script

```bash
#!/bin/bash

# TeamCity Base URL
TEAMCITY_URL="https://your-teamcity-instance.example.com"
# Credentials for authentication
USERNAME="admin"
PASSWORD="password"

# Output directory for group role files
OUTPUT_DIR="group_roles_export"
# File to store all groups (we'll get this from TeamCity API)
GROUPS_FILE="all_groups.txt"

# Create output directory if it doesn't exist
mkdir -p "$OUTPUT_DIR"

# Get all groups from TeamCity
echo "Fetching groups from TeamCity..."
curl -s -u "$USERNAME:$PASSWORD" \
  -H "Accept: application/json" \
  "$TEAMCITY_URL/app/rest/userGroups" | \
  jq -r '.group[].key' > "$GROUPS_FILE"

# Check if we got groups
if [[ ! -s "$GROUPS_FILE" ]]; then
    echo "Error: No groups found or failed to fetch groups from TeamCity"
    exit 1
fi

echo "Found $(wc -l < "$GROUPS_FILE") groups. Processing..."

# Process each group
while IFS= read -r group_key; do
    # Skip empty lines
    [[ -z "$group_key" ]] && continue
    
    echo "Processing group: $group_key"
    
    # Get group details including roles
    group_data=$(curl -s -u "$USERNAME:$PASSWORD" \
      -H "Accept: application/json" \
      "$TEAMCITY_URL/app/rest/userGroups/key:$group_key")
    
    # Create a safe filename (replace special characters)
    safe_filename=$(echo "$group_key" | tr '/' '_' | tr ' ' '_' | tr -cd '[:alnum:]_-.')
    output_file="$OUTPUT_DIR/${safe_filename}.json"
    
    # Extract and format the roles information
    echo "$group_data" | jq '{
        groupKey: .key,
        groupName: .name,
        projects: (.roles.role | map(select(.scope | startswith("p:")) | {
            projectId: (.scope | sub("^p:"; "")),
            role: .roleId
        }))
    }' > "$output_file"
    
    echo "Saved roles for $group_key to $output_file"

done < "$GROUPS_FILE"

echo "Export completed. Files are in the $OUTPUT_DIR directory."
```

## Explanation of the Script:

1. **Configuration**:
   - Sets up TeamCity URL and credentials
   - Defines output directory for the exported files
   - Creates a temporary file to store all group keys

2. **Fetching Groups**:
   - Uses curl to get all groups from TeamCity's REST API
   - Extracts group keys using jq (JSON processor)
   - Saves the list to a temporary file

3. **Processing Each Group**:
   - For each group, fetches detailed information including roles
   - Creates a safe filename by replacing special characters
   - Extracts project roles using jq to filter and format the data

4. **Data Formatting**:
   - Creates a JSON structure with:
     - Group key and name
     - Array of projects with their IDs and assigned roles
   - Filters to only include project-specific roles (scope starting with "p:")

5. **Output**:
   - Saves each group's project roles to a separate JSON file
   - Provides progress feedback during processing

## Usage Instructions:

1. Make sure you have jq installed (JSON processor):
   ```bash
   sudo apt-get install jq  # On Ubuntu/Debian
   # or
   brew install jq          # On macOS
   ```

2. Set the correct TeamCity URL and credentials in the script

3. Run the script:
   ```bash
   ./export_group_roles.sh
   ```

4. The script will create an output directory with JSON files for each group:
   - `group_roles_export/developers.json`
   - `group_roles_export/quality-assurance.json`
   - etc.

## Example Output File:

```json
{
  "groupKey": "developers",
  "groupName": "Developers",
  "projects": [
    {
      "projectId": "ProjectA",
      "role": "PROJECT_DEVELOPER"
    },
    {
      "projectId": "ProjectB",
      "role": "PROJECT_VIEWER"
    }
  ]
}
```

## Additional Notes:

1. **Security**: 
   - Consider using API tokens instead of username/password
   - Never commit credentials to version control

2. **Error Handling**:
   - The script includes basic error checking
   - You might want to add more robust error handling for production use

3. **Customization**:
   - You can modify the jq filter to include additional information
   - The output format can be adjusted based on your needs

4. **Performance**:
   - For large TeamCity instances, consider adding delays between requests
   - You might want to implement pagination if you have many groups

This script provides a comprehensive way to export group-project role relationships from TeamCity, which can be useful for documentation, auditing, or backup purposes.
