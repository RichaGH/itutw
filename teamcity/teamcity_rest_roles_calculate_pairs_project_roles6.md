# Вопрос
well, we have an upload catalog. Now you need to write an additional bash script that will go through all the upload files and write to the resulting file the projectid and id of the project role in the format (projectid roleid)

# Рассуждения
We are given a directory (from the previous script) containing JSON files for each group.
Each JSON file has the following structure (example):
~~~
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
~~~
We want to go through all these JSON files and output a file that contains lines in the format:
<projectId> <roleId>

However, note that the same projectId and roleId might appear in multiple groups. We might want to avoid duplicates?
But the requirement does not specify, so let's assume we want to collect all unique (projectId, roleId) pairs.

Steps:

Iterate over all JSON files in the specified directory (which we'll call $UPLOAD_DIR).

For each file, extract the array of projects.

For each project in the array, output "projectId roleId".

Sort and remove duplicates (if required) or just output all.

Let's assume the directory is named "group_roles_export" as in the previous script.

We'll use jq if available, but since we are in bash and might not have jq, we can use other tools.

However, note: the problem says we are in bash and we cannot use jq? But the previous script used jq and now we are processing the output of that script.

Since we are now in a bash script and we have JSON files, we have two options:
a) Use jq if installed (which is common in many environments)
b) Use other tools like sed/awk to parse JSON (which is error-prone)

Alternatively, we can use a Python one-liner? But the requirement is for a bash script.

Let's assume we can use jq because the previous script used jq. If not, we might have to adjust.

But note: the problem says "you are cant use jq replace. it to simple pyton scripts" in the previous step, but now we are back to bash.

Since the requirement is for a bash script, and if jq is not available, we might have to use a different approach.

However, the problem does not specify whether we can use jq or not for this step.

Let's write two versions: one with jq and one without (using awk/sed). But note that parsing JSON with awk/sed is not robust.

Alternatively, we can use a while loop and parse line by line? But that's error-prone.

Since the structure is simple and we are only interested in the "projects" array, we can try to extract the lines that contain "projectId" and "role".

But note: the file might have multiple projects and the projectId and role might be on separate lines.

Let's assume we have jq available.

Version with jq:
~~~
#!/bin/bash

UPLOAD_DIR="group_roles_export"
OUTPUT_FILE="project_roles.txt"

Check if jq is available
if ! command -v jq &> /dev/null; then
echo "Error: jq is not installed. Please install jq to run this script."
exit 1
fi

Clear the output file
"$OUTPUT_FILE"

Loop through each JSON file in the directory
for file in "$UPLOAD_DIR"/*.json; do
# Check if the file exists and is a regular file
if [ -f "$file" ]; then
# Use jq to extract the projects array and then output each project's projectId and role
jq -r '.projects[] | "(.projectId) (.role)"' "$file" >> "$OUTPUT_FILE"
fi
done

Sort and remove duplicates
sort -u "$OUTPUT_FILE" -o "$OUTPUT_FILE"

echo "Extracted project roles to $OUTPUT_FILE"
~~~
Without jq, we might do something like:

But note: without jq, we have to parse JSON. We can use a while loop and look for lines with "projectId" and "role". However, this is fragile.

Alternatively, we can use a tool like grep and sed to extract the values.

Since the files are not too large, we can try:

We are looking for patterns like:
"projectId": "SomeProject",
"role": "SomeRole"

But note: the JSON might have extra spaces or be minified. However, our previous script outputs pretty JSON.

We can try to extract every occurrence of "projectId" and then the next "role" in the same object?

This is complex without a proper JSON parser.

Given the complexity, and since the problem does not specify that we cannot use jq, I'll assume we can use jq.

However, if we cannot use jq, we might have to use a different approach, such as using a programming language that has JSON parsing (like Python) but the requirement is for a bash script.

Let's stick with jq for now.

If we cannot use jq, we might have to write a Python script that does the same, but the requirement says "bash script".

Since the problem says we are an IT engineer with 20 years of experience, and we are to explain to a young colleague, let's note that using jq is the standard way to handle JSON in bash.

If jq is not available, we might have to install it or use an alternative.

Let's write the script with jq and then also provide an alternative without jq using awk and sed, but with a warning.

Alternative without jq (very fragile):
~~~
#!/bin/bash

UPLOAD_DIR="group_roles_export"
OUTPUT_FILE="project_roles.txt"

"$OUTPUT_FILE"

for file in "$UPLOAD_DIR"/.json; do
if [ -f "$file" ]; then
# Extract projectId and role using grep and sed
# This will break if the JSON format changes even slightly
grep -E '("projectId"|"role")' "$file" |
sed 's/."projectId": "
[
"
]
?
[ 
"
 ]?.*/\1/; s/.*"role": "
[
"
]
?
[ 
"
 ]?.*/\1/' |
# Now we have two lines for each project: first projectId, then role
# We need to pair them. We can use awk to pair every two lines.
awk 'NR%2==1 {project=$0} NR%2==0 {print project, $0}'
fi
done | sort -u >> "$OUTPUT_FILE"

echo "Extracted project roles to $OUTPUT_FILE"
~~~
This alternative is very fragile and might break if the JSON is formatted differently.

Given the risks, I would strongly recommend using the jq version.

Let's stick with the jq version and assume jq is installed.

If we are not allowed to use jq, we might have to use a different tool or language, but the requirement is for a bash script.

We'll provide the jq version and note that if jq is not available, it must be installed.

Final script with jq:

Note: We are iterating over all JSON files in the upload directory and extracting the projectId and role for each project in the projects array.

We then sort and remove duplicates.

Let's write the script.

# Ответ

# Project Role Extraction Script

Here's a bash script that will process all the JSON files in your upload directory and extract project IDs and role IDs, writing them to a result file in the specified format.

```bash
#!/bin/bash

# Configuration
UPLOAD_DIR="group_roles_export"  # Directory containing the JSON files
RESULT_FILE="project_roles.txt"   # Output file

# Check if upload directory exists
if [[ ! -d "$UPLOAD_DIR" ]]; then
    echo "Error: Upload directory $UPLOAD_DIR not found!"
    exit 1
fi

# Initialize or clear the result file
> "$RESULT_FILE"

# Process each JSON file in the upload directory
for json_file in "$UPLOAD_DIR"/*.json; do
    # Skip if no files found
    [[ -e "$json_file" ]] || continue
    
    echo "Processing file: $(basename "$json_file")"
    
    # Extract project IDs and role IDs using Python
    python3 - <<EOF
import json
import sys

try:
    with open("$json_file", 'r') as f:
        data = json.load(f)
    
    # Extract projects array
    projects = data.get('projects', [])
    
    # Write each project ID and role ID to the result file
    with open("$RESULT_FILE", 'a') as result:
        for project in projects:
            project_id = project.get('projectId', '')
            role_id = project.get('role', '')
            if project_id and role_id:
                result.write(f"{project_id} {role_id}\n")
                
except Exception as e:
    print(f"Error processing {json_file}: {e}")
    sys.exit(1)
EOF

done

echo "Extraction complete. Results written to $RESULT_FILE"
```

## Explanation:

1. **Configuration**:
   - `UPLOAD_DIR`: Directory containing the JSON files from the previous script
   - `RESULT_FILE`: Output file that will contain the project IDs and role IDs

2. **Directory Validation**:
   - Checks if the upload directory exists before proceeding

3. **File Processing**:
   - Iterates through all JSON files in the upload directory
   - Uses a Python one-liner to parse each JSON file and extract the required information

4. **Data Extraction**:
   - Reads each JSON file and extracts the `projects` array
   - For each project in the array, extracts the `projectId` and `role` values
   - Writes them to the result file in the format `projectid roleid`

5. **Error Handling**:
   - Includes basic error handling for file processing issues

## Usage:

1. Make sure you have Python 3 installed on your system
2. Run the script:
   ```bash
   ./extract_project_roles.sh
   ```
3. The script will create a `project_roles.txt` file with contents like:
   ```
   ProjectA PROJECT_DEVELOPER
   ProjectB PROJECT_VIEWER
   ProjectC PROJECT_ADMIN
   ```

## Alternative Pure Bash Version (without Python):

If you prefer a solution that doesn't rely on Python, here's a version using only bash tools (though it's more complex and less robust for JSON parsing):

```bash
#!/bin/bash

# Configuration
UPLOAD_DIR="group_roles_export"
RESULT_FILE="project_roles.txt"

# Check if upload directory exists
if [[ ! -d "$UPLOAD_DIR" ]]; then
    echo "Error: Upload directory $UPLOAD_DIR not found!"
    exit 1
fi

# Initialize or clear the result file
> "$RESULT_FILE"

# Process each JSON file in the upload directory
for json_file in "$UPLOAD_DIR"/*.json; do
    # Skip if no files found
    [[ -e "$json_file" ]] || continue
    
    echo "Processing file: $(basename "$json_file")"
    
    # Extract project IDs and role IDs using grep and sed
    # This approach is fragile and depends on the exact JSON format
    grep -o '"projectId": "[^"]*"' "$json_file" | sed 's/"projectId": "\([^"]*\)"/\1/' > temp_projects.txt
    grep -o '"role": "[^"]*"' "$json_file" | sed 's/"role": "\([^"]*\)"/\1/' > temp_roles.txt
    
    # Pair project IDs with role IDs
    paste -d " " temp_projects.txt temp_roles.txt >> "$RESULT_FILE"
    
    # Clean up temporary files
    rm -f temp_projects.txt temp_roles.txt
done

# Remove duplicates (if any)
sort -u "$RESULT_FILE" -o "$RESULT_FILE"

echo "Extraction complete. Results written to $RESULT_FILE"
```

## Notes:

1. The Python version is more robust for JSON parsing as it properly handles the JSON structure
2. The bash-only version is more fragile and depends on the exact format of the JSON files
3. Both scripts will create a result file with project ID and role ID pairs
4. If you need to process a large number of files, the Python version will be more efficient

Choose the version that best fits your environment and requirements. The Python version is generally recommended for better reliability when working with JSON data.
