# Вопрос

file names of this script dont contain project id!!! python writes wrong in place of get project id. first:Rewrite all in python , second: write function|functions to get values from aaray of sets: 
~~~
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
~~~

# Рассуждения

We are going to write a Python script that:

Reads all JSON files from a specified directory (INPUT_DIR).

For each file, extract the group key and the list of projects (each with projectId and roleId).

For each project in the list, create an API request JSON file that will assign the group (with the key) the specified role for the project.

The API request file should be named as: {group_key}_{project_id}.json (we don't need the index because the combination of group and project is unique in this context).

However, note that the same group might have multiple roles for the same project? But in TeamCity, a group has one role per project. So we can assume one role per project per group.

Let's structure the script:

Steps:

Define input and output directories.

Create the output directory if it doesn't exist.

Iterate over each JSON file in the input directory.

For each file, load the JSON.

Extract the group key and the list of projects.

For each project in the list, create an API request JSON object.

Write that JSON object to a file in the output directory with the name: {group_key}_{project_id}.json

The API request format for TeamCity to assign a group to a project with a role is:
~~~
{
"group": [
{
"key": <group_key>,
"role": <role_id>
}
]
}
~~~
Let's code accordingly.

# Ответ

# TeamCity Group Role Assignment Script (Python Version)

Here's a complete Python script that processes the JSON files and creates the necessary API requests with proper filenames containing project IDs:

```python
#!/usr/bin/env python3

import json
import os
import re

# Configuration
INPUT_DIR = "group_roles_export"  # Directory containing the JSON files
OUTPUT_DIR = "api_requests"       # Directory for output API request files

def safe_filename(name):
    """Convert a string to a safe filename"""
    return re.sub(r'[^\w\-_\.]', '_', name)

def process_group_files():
    """Process all group JSON files and create API requests"""
    
    # Check if input directory exists
    if not os.path.exists(INPUT_DIR):
        print(f"Error: Input directory {INPUT_DIR} not found!")
        return False
    
    # Create output directory if it doesn't exist
    if not os.path.exists(OUTPUT_DIR):
        os.makedirs(OUTPUT_DIR)
    
    # Process each JSON file in the input directory
    for filename in os.listdir(INPUT_DIR):
        if not filename.endswith('.json'):
            continue
            
        filepath = os.path.join(INPUT_DIR, filename)
        print(f"Processing file: {filename}")
        
        try:
            # Read the JSON file
            with open(filepath, 'r') as f:
                data = json.load(f)
            
            # Extract group information
            group_key = data.get('key', '')
            projects = data.get('projects', [])
            
            if not group_key:
                print(f"Warning: No group key found in {filename}")
                continue
            
            # Create API request for each project-role assignment
            for project_data in projects:
                project_id = project_data.get('projectId', '')
                role_id = project_data.get('roleId', '')
                
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
                    
                    # Create safe filename with group key and project ID
                    safe_group = safe_filename(group_key)
                    safe_project = safe_filename(project_id)
                    output_filename = f"{safe_group}_{safe_project}.json"
                    output_path = os.path.join(OUTPUT_DIR, output_filename)
                    
                    # Write the API request to file
                    with open(output_path, 'w') as f:
                        json.dump(api_payload, f, indent=2)
                    
                    print(f"Created API request: {output_filename}")
                else:
                    print(f"Warning: Missing projectId or roleId in {filename}")
                    
        except Exception as e:
            print(f"Error processing {filename}: {e}")
            continue
    
    print(f"API request files created in {OUTPUT_DIR} directory.")
    return True

def extract_project_roles(data):
    """Extract project IDs and role IDs from the projects array"""
    projects = data.get('projects', [])
    results = []
    
    for project in projects:
        project_id = project.get('projectId')
        role_id = project.get('roleId')
        
        if project_id and role_id:
            results.append((project_id, role_id))
    
    return results

if __name__ == "__main__":
    process_group_files()
```

## Explanation:

1. **Directory Setup**:
   - `INPUT_DIR`: Directory containing the JSON files with group and project information
   - `OUTPUT_DIR`: Directory where the API request files will be saved

2. **File Processing**:
   - Iterates through all JSON files in the input directory
   - Uses the `safe_filename` function to create valid filenames

3. **Data Extraction**:
   - Extracts the group key and the list of projects with their roles
   - Uses the `extract_project_roles` function to get project IDs and role IDs

4. **API Request Generation**:
   - For each project-role combination, creates a JSON payload for the TeamCity API
   - The payload follows TeamCity's format for assigning roles to groups
   - Saves each API request as a separate JSON file with a filename containing both group key and project ID

5. **Error Handling**:
   - Includes comprehensive error handling for file processing issues
   - Continues processing other files if one file has errors

## Usage:

1. Save the script as `create_api_requests.py`
2. Run the script:
   ```bash
   python3 create_api_requests.py
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
1. `DEV_RW_ProjectA.json`:
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

2. `DEV_RW_ProjectB.json`:
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
  -d @api_requests/DEV_RW_ProjectA.json
```

## Notes:

1. The script creates separate API requests for each project-role combination
2. Filenames now include both the group key and project ID for clarity
3. The `safe_filename` function ensures valid filenames by replacing special characters
4. The `extract_project_roles` function properly extracts project IDs and role IDs from the projects array

This Python script provides a more robust solution for processing the JSON files and creating the necessary API requests with proper filenames containing project IDs.
