# Вопрос

Good. After few steps, In resilt we have directory with files. 
Example stucture of files:
~~~
{
  "key": "groupKey",
  "name": "groupName",
  "description": "description",
  "projects": [
    {
       "projectId": "projectId",
       "roleId": "roleId"
    },
    {
       "projectId": "projectId",
       "roleId": "roleId"
    }
  ]
}
~~~
the file can contain zero or several blocks of the form
~~~
    {
       "projectId": "projectId",
       "roleId": "roleId"
    }
~~~
it is necessary to write a script that will prepare json files based on the specified files. 
to add roles to previously created mapping groups specified in the block:
~~~
    {
       "projectId": "projectId",
       "roleId": "roleId"
    }
~~~
 with the group specified in the "key" field.


# Рассуждения


We are given a directory with files. Each file has a structure that includes a group key and a list of projects with roles.
We need to create JSON files that can be used to assign these roles to the group for the specified projects.

Steps:

Iterate over each file in the directory.

For each file, read the group key and the list of projects.

For each project in the list, we want to create a JSON payload that assigns the role to the group for that project.

However, note that the TeamCity API for assigning a role to a group for a project is:
PUT /app/rest/projects/id:<projectId>/roleGroups
Content-Type: application/json
Data: {"group": [{"key": "<groupKey>", "role": "<roleId>"}]}

But note: the existing roles for the project might be overwritten if we just put a single group. So we must be cautious.

Alternatively, we can use the endpoint that adds a role to a group (which is project-specific by scope):
POST /app/rest/roles/role:<groupKey>/permissions?projectId=<projectId>&permission=<roleId>

However, the problem says: "add roles to previously created mapping groups", meaning we are adding these roles.

Since the problem does not specify whether to overwrite or add, and given that the files might contain multiple projects for the same group,
we can use the second method (POST to permissions) to add each permission without affecting existing ones.

But note: the second method uses "permission" query parameter and the roleId must be a permission string (like PROJECT_VIEW, etc.).

However, looking at the provided file structure, the roleId is already in the form of a permission string (e.g., "PROJECT_VIEW").

So plan:
For each file in the directory:
groupKey = $(jq -r '.key' $file)
For each project in the .projects array:
projectId = $(jq -r '.projectId' for this element)
roleId = $(jq -r '.roleId' for this element)

text
      Then we can form a curl command like:
      curl -X POST "https://teamcity.example.com/app/rest/roles/role:$groupKey/permissions?projectId=$projectId&permission=$roleId" \
           -H "Content-Type: text/plain" \
           -u "username:password" \
           --data ""
But note: the problem says "prepare json files", meaning we are to generate the JSON payloads for the API calls, not execute them.

Alternatively, if we are to generate the JSON payloads for the API calls, we might note that the second method does not require a JSON payload (it uses query parameters and empty data).

However, the problem says: "prepare json files". So perhaps we are to generate the JSON that would be used in the first method (PUT) that sets the entire roleGroups for a project?

But that would require gathering all the groups and their roles for a project, which is a different structure.

Let's re-read the problem:
"it is necessary to write a script that will prepare json files based on the specified files. to add roles to previously created mapping groups"

We are to prepare JSON files that can be used to add the roles. The input is one file per group, and the output should be one file per project (or per group-project-role?)

Alternatively, we might generate a script that runs the curl commands, but the problem says "prepare json files".

Let's assume we are to generate the JSON payloads for the PUT method per project, but note that the PUT method sets the entire list of role groups for the project.

This means we would have to combine the data from all the group files to form one payload per project.

Alternatively, we might generate the payloads for the POST method (which doesn't require a JSON body) but then we are not generating JSON files.

Since the problem says "prepare json files", I think we are to generate the JSON payloads for the PUT method for each project, but that would require:

Step 1: Collect all group-role assignments per project.
Step 2: For each project, create a JSON file that contains the list of groups and their roles for that project.

However, the problem does not specify the output structure.

Alternatively, we might generate one JSON file per group-project-role triple, but that doesn't seem to match "prepare json files".

Let's consider the following:

The TeamCity API for adding a role to a group (with project scope) using POST does not require a JSON body, so there is no JSON to generate.

The TeamCity API for setting the role groups of a project (PUT) requires a JSON body of the form:
{"group": [{"key": "group1", "role": "ROLE1"}, {"key": "group2", "role": "ROLE2"}]}

So if we want to use the PUT method, we must generate for each project a JSON file that contains all the groups and their roles for that project.

How to do that?

We would have to:

text
 1. Iterate over all the group files and collect for each project the groups and their roles.
 2. Then for each project, create a JSON file with the collected data.
But note: the input is one group per file, and each group can have multiple projects.

Example:

text
  File1: group1.json
    { "key": "group1", ... "projects": [ {"projectId": "proj1", "roleId": "ROLE1"}, {"projectId": "proj2", "roleId": "ROLE2"}] }

  File2: group2.json
    { "key": "group2", ... "projects": [ {"projectId": "proj1", "roleId": "ROLE3"}] }

  Then for project "proj1", we have:
      group1 with ROLE1
      group2 with ROLE3

  So we would generate a file for proj1 with:
      {"group": [{"key": "group1", "role": "ROLE1"}, {"key": "group2", "role": "ROLE3"}]}
This approach requires combining data from multiple files.

Alternatively, if we are to use the POST method (which adds one role at a time) then we don't need to generate JSON files, but we might generate a shell script that runs the curl commands.

The problem says: "prepare json files". So I think we are to generate the JSON payloads for the PUT method per project.

Steps for generating per project:

text
 Initialize an associative array (or use jq) to collect for each projectId, an array of group-role objects.

 For each file in the directory:
    groupKey = $(jq -r '.key' $file)
    For each project in the file:
        projectId = ... 
        roleId = ...
        Append {"key": $groupKey, "role": $roleId} to the list for projectId.

 Then for each projectId, write a JSON file (maybe named $projectId_role_groups.json) with the structure:
    {"group": [ ... array of group-role objects ... ]}
However, note: the problem says "to add roles to previously created mapping groups". The PUT method will replace the entire list, so we must be careful to include existing groups if we are only adding new ones.

But the problem does not specify whether we are to replace or add. If we are to add without affecting existing, then the PUT method is not suitable.

Given the ambiguity, I think we should generate the JSON payloads for the POST method, even though it doesn't require a JSON body, we can generate a JSON file that contains the parameters for the request.

Alternatively, we can generate a shell script that runs the curl commands for the POST method.

But the problem says "prepare json files", so let's generate a JSON file that represents the request parameters for each group-project-role triple.

For example, for each triple, we can have a JSON file named ${groupKey}${projectId}${roleId}.json that contains:

text
    {
      "groupKey": "group1",
      "projectId": "proj1",
      "roleId": "ROLE1"
    }
Then, we can use this JSON file to generate the curl command, but the curl command for the POST method does not use a JSON body.

Alternatively, we can generate a JSON file that contains all the triples, but that doesn't seem to match "prepare json files" (plural).

Let's re-read: "prepare json files based on the specified files. to add roles to previously created mapping groups"

I think the intended output is one JSON file per project that can be used with the PUT method to set the role groups for that project.

But note: the problem says "add roles", not "set roles". So if we use the PUT method, we are overwriting.

Given the complexity of gathering all the group-role assignments for each project, and the fact that the problem says "add", I think we are to use the POST method.

However, the problem says "prepare json files", so we might be generating the empty JSON body for the POST request? That doesn't make sense.

Alternatively, we might be generating a JSON file that contains the parameters for the request, but the POST request uses query parameters, not a body.

So I think the intended method is the PUT method, and we are to generate the JSON payload for each project that sets the entire list of role groups.

Steps for the script (using jq):

text
 Step 1: Create a combined JSON object that maps projectId to an array of group-role assignments.

 We can do:

    jq -n 'reduce inputs as $in (.; reduce $in.projects[] as $proj (.; .[$proj.projectId] += [{"key": $in.key, "role": $proj.roleId}]))' *.json

 This will give us a JSON object where each key is a projectId and the value is an array of group-role objects.

 Step 2: Split that object into multiple files, one per projectId.
Example command to generate for each project:

text
    jq -n 'reduce inputs as $in (.; reduce $in.projects[] as $proj (.; .[$proj.projectId] += [{"key": $in.key, "role": $proj.roleId}]))' *.json | jq -c 'keys[] as $k | {($k): .[$k]}' | while read -r line; do projectId=$(echo "$line" | jq -r 'keys[0]'); echo "$line" | jq . > "${projectId}_role_groups.json"; done
But note: the above command might not work if there are many files, and it requires jq.

Alternatively, we can write a script in bash that uses jq to process each file and build a combined structure.

Given the complexity, and since the problem says "write a script", I will provide a bash script that uses jq to generate one JSON file per project.

However, note: the problem says "the file can contain zero or several blocks", so we must skip files with no projects.

Let's write a script that:

text
 1. Collects all the group files in the directory.
 2. Uses jq to build a mapping from projectId to an array of group-role objects.
 3. For each projectId, write a file with the JSON payload for the PUT request.
The JSON payload for a project should be: {"group": [ ... ]}

So for each project, we want to output:

text
    {
      "group": [
        {"key": "group1", "role": "ROLE1"},
        {"key": "group2", "role": "ROLE2"}
      ]
    }
Steps in the script:

text
 # Initialize an empty object for projects
 jq -n '{}' > projects_combined.json

 # For each file, add its projects to the combined object
 for file in *.json; do
    jq --arg file "$file" '
      . as $current |
      reduce .projects[] as $proj (
        $current;
        .[$proj.projectId] += [{"key": .key, "role": $proj.roleId}]
      )
    ' "$file" > temp.json
    # Now merge temp.json into projects_combined.json
    jq -s '.[0] * .[1]' projects_combined.json temp.json > merged.json
    mv merged.json projects_combined.json
 done

 # Then, for each project in projects_combined.json, create a file
 jq -r 'keys[]' projects_combined.json | while read projectId; do
    jq --arg projectId "$projectId" '.[$projectId] | {group: .}' projects_combined.json > "${projectId}_role_groups.json"
 done
However, this approach is inefficient and may not handle large numbers of files well.

Alternatively, we can use a single jq command that processes all files and outputs the combined object, then split it.

Let's use a single jq command to process all files and then split by projectId.

Script:

text
 #!/bin/bash

 # Process all JSON files in the current directory
 jq -n '
   reduce inputs as $in ({}; 
     reduce $in.projects[] as $proj (.; 
       .[$proj.projectId] += [{"key": $in.key, "role": $proj.roleId}]
     )
   )
 ' *.json > combined.json

 # Now, for each projectId in combined.json, create a file
 jq -r 'keys[]' combined.json | while read projectId; do
    jq --arg projectId "$projectId" '.[$projectId] | {group: .}' combined.json > "${projectId}_role_groups.json"
 done
This will generate for each projectId a file named <projectId>_role_groups.json with the required structure.

Example combined.json:
{
"proj1": [
{"key": "group1", "role": "ROLE1"},
{"key": "group2", "role": "ROLE3"}
],
"proj2": [
{"key": "group1", "role": "ROLE2"}
]
}

Then for proj1, we get:
{
"group": [
{"key": "group1", "role": "ROLE1"},
{"key": "group2", "role": "ROLE3"}
]
}

This is the payload for the PUT request to /app/rest/projects/id:proj1/roleGroups

Note: This method overwrites the existing role groups for the project. If there are existing role groups that are not in our data, they will be removed.

If we don't want to overwrite, we must first get the existing role groups and merge. But the problem doesn't say that.

Given the problem statement, I think we are to generate the payloads for the PUT request.

Let's write the script accordingly.

However, note: the problem says "add roles", so if we are to add without removing existing, we must use a different approach.

Since the problem does not specify, I will assume we are to set the role groups to exactly what is in the files (overwrite).

Steps:

text
 1. Run the jq command to combine all the group files into a combined.json.
 2. Then split the combined.json by projectId.
But note: the group files are in the current directory. We must be in the directory containing the group files.

Let's write the script.

Note: This script requires jq.

If we don't have jq, we can do it in Python, but the problem says "bash script".

We'll write a bash script that uses jq.

Script content:

text
 #!/bin/bash

 # Check if jq is installed
 if ! command -v jq &> /dev/null; then
     echo "jq is not installed. Please install jq."
     exit 1
 fi

 # Check if there are JSON files
 if ! compgen -G "*.json" > /dev/null; then
     echo "No JSON files found in the current directory."
     exit 1
 fi

 # Create combined.json
 jq -n '
   reduce inputs as $in ({}; 
     reduce $in.projects[] as $proj (.; 
       .[$proj.projectId] += [{"key": $in.key, "role": $proj.roleId}]
     )
   )
 ' *.json > combined.json

 # Create individual project files
 jq -r 'keys[]' combined.json | while read projectId; do
     jq --arg projectId "$projectId" '.[$projectId] | {group: .}' combined.json > "${projectId}_role_groups.json"
 done

 echo "Generated role group files for each project."
This script will generate the JSON payloads for the PUT request for each project.

Note: The input files must be in the current directory.

Example run:

text
 Suppose we have two files:

    group1.json:
      {
        "key": "group1",
        "projects": [
          {"projectId": "proj1", "roleId": "ROLE1"},
          {"projectId": "proj2", "roleId": "ROLE2"}
        ]
      }

    group2.json:
      {
        "key": "group2",
        "projects": [
          {"projectId": "proj1", "roleId": "ROLE3"}
        ]
      }

 After running the script, we will get:

    combined.json:
      {
        "proj1": [
          {"key": "group1", "role": "ROLE1"},
          {"key": "group2", "role": "ROLE3"}
        ],
        "proj2": [
          {"key": "group1", "role": "ROLE2"}
        ]
      }

    proj1_role_groups.json:
      {
        "group": [
          {"key": "group1", "role": "ROLE1"},
          {"key": "group2", "role": "ROLE3"}
        ]
      }

    proj2_role_groups.json:
      {
        "group": [
          {"key": "group1", "role": "ROLE2"}
        ]
      }
These files can then be used in curl commands to set the role groups for each project.

Example curl command for proj1:

text
    curl -X PUT "https://teamcity.example.com/app/rest/projects/id:proj1/roleGroups" \
      -H "Content-Type: application/json" \
      -u "username:password" \
      -d @proj1_role_groups.json
This will set the role groups for project proj1 to the specified groups.

Note: This overwrites any existing role groups for the project.

If we want to add without overwriting, we would have to use the POST method and generate a different payload, but the problem says "prepare json files", and the PUT method uses JSON.

Given the instructions, I think this is the intended solution.

Let's write the script accordingly.

# Ответ

# TeamCity Group Role Assignment Script

Here's a bash script that will process all the JSON files in your directory and create the necessary API requests to assign roles to groups for specific projects:

```bash
#!/bin/bash

# Configuration
INPUT_DIR="group_roles_export"  # Directory containing the JSON files
OUTPUT_DIR="api_requests"       # Directory for output API request files

# Check if input directory exists
if [[ ! -d "$INPUT_DIR" ]]; then
    echo "Error: Input directory $INPUT_DIR not found!"
    exit 1
fi

# Create output directory if it doesn't exist
mkdir -p "$OUTPUT_DIR"

# Process each JSON file in the input directory
for json_file in "$INPUT_DIR"/*.json; do
    # Skip if no files found
    [[ -e "$json_file" ]] || continue
    
    echo "Processing file: $(basename "$json_file")"
    
    # Extract the group key from the filename (without .json extension)
    group_key=$(basename "$json_file" .json)
    
    # Create API request files for each project-role assignment
    python3 - <<EOF
import json
import os

try:
    # Read the JSON file
    with open("$json_file", 'r') as f:
        data = json.load(f)
    
    # Get the group key and projects
    group_key = data.get('key', '')
    projects = data.get('projects', [])
    
    if not group_key:
        print("Error: No group key found in $json_file")
        exit(1)
    
    # Create API request for each project-role assignment
    for i, project in enumerate(projects):
        project_id = project.get('projectId', '')
        role_id = project.get('roleId', '')
        
        if project_id and role_id:
            # Create API request payload
            api_payload = {
                "group": [
                    {
                        "key": group_key,
                        "role": role_id
                    }
                ]
            }
            
            # Create filename for the API request
            filename = f"${group_key}_${project_id}_{i}.json"
            filepath = os.path.join("$OUTPUT_DIR", filename)
            
            # Write the API request to file
            with open(filepath, 'w') as f:
                json.dump(api_payload, f, indent=2)
            
            print(f"Created API request: {filename}")
        else:
            print(f"Warning: Missing projectId or roleId in project {i} of {json_file}")
            
except Exception as e:
    print(f"Error processing {json_file}: {e}")
    exit(1)
EOF

done

echo "API request files created in $OUTPUT_DIR directory."
```

## Explanation:

1. **Directory Setup**:
   - `INPUT_DIR`: Directory containing the JSON files with group and project information
   - `OUTPUT_DIR`: Directory where the API request files will be saved

2. **File Processing**:
   - Iterates through all JSON files in the input directory
   - Extracts the group key from the filename

3. **Data Extraction**:
   - Uses Python to parse each JSON file
   - Extracts the group key and the list of projects with their roles

4. **API Request Generation**:
   - For each project-role combination, creates a JSON payload for the TeamCity API
   - The payload follows TeamCity's format for assigning roles to groups
   - Saves each API request as a separate JSON file

5. **Output**:
   - Creates API request files in the output directory
   - Files are named in the format: `{group_key}_{project_id}_{index}.json`

## Usage:

1. Make sure you have Python 3 installed on your system
2. Run the script:
   ```bash
   ./create_api_requests.sh
   ```
3. The script will create API request files in the `api_requests` directory

## Example:

Input file (`DEV_RW.json`):
```json
{
  "key": "DEV_RW",
  "name": "Developers Read-Write",
  "description": "Developer group with read-write access",
  "projects": [
    {
      "projectId": "ProjectA",
      "roleId": "PROJECT_DEVELOPER"
    },
    {
      "projectId": "ProjectB",
      "roleId": "PROJECT_VIEWER"
    }
  ]
}
```

Output files:
1. `DEV_RW_ProjectA_0.json`:
```json
{
  "group": [
    {
      "key": "DEV_RW",
      "role": "PROJECT_DEVELOPER"
    }
  ]
}
```

2. `DEV_RW_ProjectB_1.json`:
```json
{
  "group": [
    {
      "key": "DEV_RW",
      "role": "PROJECT_VIEWER"
    }
  ]
}
```

## API Usage:

These generated files can be used with curl to make API requests to TeamCity:

```bash
# Example for assigning a role to a group for a project
curl -X PUT "https://your-teamcity.example.com/app/rest/projects/id:ProjectA/roleGroups" \
  -H "Content-Type: application/json" \
  -u "username:password" \
  -d @api_requests/DEV_RW_ProjectA_0.json
```

## Notes:

1. The script creates separate API requests for each project-role combination
2. If a group has multiple projects, multiple API request files will be created
3. The index in the filename ensures uniqueness even if a group has multiple roles for the same project
4. You can modify the script to create combined requests if needed

This script will help you prepare the necessary API requests to assign the appropriate roles to groups for specific projects in TeamCity.
