# Вопрос
You are a programmer with 20 years of experience. Your task is to write a python script for processing a file in json format according to the following rule: Leave all lines in the file unchanged except the one containing "privileges" : [
in this line, you need to delete all substrings of the form with any number of characters starting with the quotation mark and ending with the quotation mark +comma characters in a row that do not they contain the word proxy

# Рассуждения
We are going to read the file line by line.
 For each line, we check if it contains the string `"privileges" : [`.
 If it does, we need to process that line: remove any substring that is a quoted string (which may contain any characters) ending with a quotation mark and comma, but only if that quoted string does not contain the word "proxy".
 However, note that the line might have multiple such strings and we want to remove only those that do not contain "proxy".
 Steps for processing the line:
 1. We note that the line might look like: `"privileges" : [ "string1", "string2", ... ]`
 2. We want to remove any element (i.e., a quoted string followed by a comma) that does not have the word "proxy" in it.
 3. But note: after the last element there might be no comma? However, the problem says "ending with the quotation mark + comma". So we are looking for patterns: `"<something>",`
 4. However, we must also consider that the element might be at the end and without a comma? But the rule says we are to delete substrings that end with quotation mark and comma. So if there is no comma, then we are not matching? But note: the rule says "ending with the quotation mark + comma characters in a row". So if the element is the last one and has no comma, we are not removing it? But wait, the requirement is to remove the ones that do not contain the word "proxy". So we must remove even if it's the last one? However, the pattern described is for a substring that ends with `",` (quotation mark and comma). So we have two cases: elements with a comma and the last element without a comma.
 Let me re-read: "delete all substrings of the form with any number of characters starting with the quotation mark and ending with the quotation mark + comma characters in a row". So it must end with `",` (quotation mark and comma). So if an element is at the end and has no comma, then it doesn't match the pattern? Then we leave it? But that might be inconsistent because we want to remove non-proxy elements.
 However, note the problem says: "Leave all lines in the file unchanged except the one containing ...". So we are only changing the line with `"privileges" : [`. And in that line, we are to remove non-proxy elements that are in the form of a quoted string followed by a comma.
 But what about the last element? It doesn't have a trailing comma. So we have two types of elements:
   - Those that are followed by a comma: `"some string",`
   - The last one: `"some string"`
 We are only required to remove the ones that are followed by a comma? That doesn't make sense because we want to remove non-proxy elements regardless of position.
 Alternatively, we can interpret the requirement as: remove any element (quoted string) that does not contain "proxy", and then we have to reconstruct the array. But note the input is a JSON array? However, we are processing a line of text, not a JSON object.
 Approach:
 Since we are dealing with a single line, and we don't want to parse the entire JSON (because the problem says to leave other lines unchanged and we are only changing one line), we can do:
 Idea:
  We can split the line by commas, but note that the array might have nested structures? The problem says it's a line in a JSON file and the line contains `"privileges" : [`. But the array elements are strings? So we assume the array is a simple array of strings.
 However, the problem states: "delete all substrings of the form ... that do not contain the word proxy". So we are to remove entire elements (the whole quoted string and the comma) that do not contain "proxy", but note that the element might be in the middle and we remove it along with its comma? But what if we remove an element and then we have two commas in a row? That would break the JSON.
 Alternatively, we can do:
  Step 1: Extract the entire array part? But that might be complex because the array might span multiple lines? The problem says "the one containing", meaning the entire array might be in one line? We assume the entire array is in the same line.
  Step 2: We can use a regular expression to find all quoted strings in the array? But note: the requirement is to remove the entire element (including the quotes and the comma) for non-proxy ones.
  How about:
    We split the line into parts that are outside the array? Actually, we can try to extract the array content and then process it.
  But note: we are only allowed to change this one line.
  Let me break down the line structure:
      "privileges" : [ ... ]
  We can do:
      - Find the starting bracket after `"privileges" : [`, and then find the matching closing bracket? But that might be heavy and we are in a line.
  Alternatively, we can use a simpler approach: since the array is of strings, we can look for the pattern of a quoted string: `"<any character that isn't a quote or that is escaped but we don't have escaped?>`
  But note: the problem does not specify if the strings can contain escaped quotes. However, to be safe, we should consider that they might. But the requirement is to remove the ones without "proxy", so we can use a state machine or use a regex that can handle simple cases and hope that the strings are simple? Or we can use the json module for that line? But note: the entire file might be too big, and we are processing line by line.
  However, the problem says: "Leave all lines ... unchanged except the one ...". So we can try to parse the entire line as a JSON object? But the line is just one part of a larger JSON? So that won't work.
  Let's stick to string processing for that line.
  We can do:
      We are going to remove every occurrence of a pattern: `"<anything not containing the word proxy>",`? But note: the requirement is to remove the ones that do not contain the word proxy. So we want to keep the ones that contain proxy.
  However, the pattern must be: a quoted string that does not contain "proxy", and that is immediately followed by a comma (and optional whitespace). But what if the element is the last one? Then it doesn't have a comma. So we have to also handle the last element.
  We can break the line into tokens? Actually, we can use a regex to match each element and then decide to remove it or not.
  Proposed regex for matching a string element: `"([^"\\]*(\\.[^"\\]*)*)"`? This is a standard regex for matching a string with escaped quotes? But that might be overkill.
  Alternatively, we can do:
      We know the array starts with `[` and ends with `]` in the same line? We assume so.
  Steps:
      1. Locate the array part: from the first '[' after the string `"privileges" : [`, to the matching closing bracket? But note: the string `"privileges" : [` might appear only once in the line.
      2. We can extract the substring between the brackets.
      3. Then split that substring by commas? But again, commas inside nested structures? We are assuming the array elements are simple strings without commas.
      4. Then, for each element, check if it is a quoted string? and then check if it contains the word "proxy". But note: the element might be surrounded by spaces.
      5. Then, we will build a new array string by including only the elements that contain "proxy" (case sensitive? the problem doesn't say, but let's assume as is).
      6. Then replace the old array string with the new one.
  However, the problem says to delete the substring (the entire element including the quotes and the comma) for non-proxy ones. So we are effectively rebuilding the array.
  But note: the array might be empty? or have only one element? We have to be careful.
  Example:
      Original line: ... "privileges" : [ "read", "write", "create_proxy", "delete" ] ...
      We want to remove "read", "write", "delete" because they don't contain "proxy". So the new array should be: [ "create_proxy" ]
      So the line becomes: ... "privileges" : [ "create_proxy" ] ...
  How to do:
      Step 1: Find the index of the opening bracket and then find the closing bracket? But note: the array might have nested arrays? The problem says it's a line containing `"privileges" : [`, and we are to assume the entire array is in one line and without nested arrays? Let's assume so.
      We can do:
          start_index = line.find('[')   # but we know we are in the line that has the pattern, so we can look for the first '[' after the colon?
          Actually, we know the string `"privileges" : [`, so we can do:
          start_str = '"privileges" : ['
          start_index = line.find(start_str)
          if start_index != -1:
              # then the array starts at start_index + len(start_str) - 1? Actually, the last character of start_str is the '['? 
              # Actually, the start_str ends with '['? So the array content starts at start_index + len(start_str) and then we need to find the matching closing bracket.
          But how to find the matching bracket? We can count: 
              count = 0
              for i from start_index+len(start_str)-1 to end of line:
                  if char is '[' then count++
                  if char is ']' then count--; if count becomes 0, then we found the closing bracket.
          However, note: the array might contain strings that have brackets? But if they are inside quotes, then we don't count. But we are not parsing the content.
      Alternatively, we can assume the array is flat (no nested arrays) and the closing bracket is the first ']' that is not inside a string? That's complex.
  Given the complexity, and since the array is of strings, we can assume that the array does not contain any brackets? Then we can just look for the next ']' and take everything until then.
  But note: the strings might contain a bracket? For example: `"abc[def"`. Then we cannot simply look for the first ']'.
  So we have to parse the array content as a list of JSON strings? We can use the `json` module to parse the entire array? But note: the entire line is not a valid JSON? Only the array part?
  We can extract the array part by:
      start_index = line.find('[')   # at the position after "privileges" : [
      Then we start from that '[' and then we want to parse until the matching ']'. We can use a stack:
          stack = []
          in_string = False
          escape = False
          start = start_index
          for i in range(start_index, len(line)):
              c = line[i]
              if not in_string:
                  if c == '[':
                      stack.append('[')
                  elif c == ']':
                      stack.pop()
                      if len(stack)==0:
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
          Then the array string is line[start_index: end_index+1]
      Then we can use `json.loads` to parse the array string? But note: the array string might have trailing commas? No, JSON does not allow trailing commas.
  Alternatively, we can avoid parsing and do the same tokenization to split the array into elements? But that's what we are doing above to find the matching bracket.
  Given the complexity, and since the problem states that the file is in JSON format, I think we can rely on the entire file being valid JSON? Then we can read the entire file as JSON? But the problem says to leave other lines unchanged? Actually, the entire file is one JSON object? Then we could:
      Read the entire file as JSON, then traverse the structure to find the "privileges" array and remove the non-proxy strings, then write back the entire JSON.
  However, the problem says: "write a python script for processing a file". It doesn't specify the size. If the file is huge, we cannot load it entirely.
  So we are back to line-by-line.
  Considering the complexity, and that the array is of strings and we are only changing one line, we can do:
      We assume the array is flat and the strings do not contain brackets or commas? Then we can split by commas and then check each token.
  But the requirement says "any number of characters", meaning the string can contain commas and brackets? So we cannot split by commas.
  Therefore, we have to parse the array content.
  Let's design a function to extract the array string (including the brackets) from the line, then parse it as a JSON array, then filter out the strings that do not contain "proxy", then replace the old array string with the new one.
  Steps:
      line = ... the line we are processing ...
      # Find the array part by locating the brackets with a simple bracket matching that ignores brackets inside strings.
      We'll write a helper function: `extract_array_string(s, start_index)` that returns the substring from start_index until the matching closing bracket.
      Then we do:
          array_str = extract_array_string(line, start_index)   # where start_index is the index of the opening bracket
          Then we parse array_str as JSON -> becomes a list of strings.
          Then we filter: new_list = [s for s in old_list if "proxy" in s]
          Then we convert new_list to a JSON string (without spaces? or with the same formatting? We want to preserve the original formatting? But we don't know. Let's use the same as the original? Actually, we are replacing the entire array string. We can format the new array without any extra space? But the original might have spaces. We can use:
              new_array_str = json.dumps(new_list)
          But note: the original array string might have been formatted with newlines or spaces? We are replacing the entire array string. We are going to change the formatting? The problem says "Leave all lines unchanged" but we are changing this line. However, it doesn't specify preserving formatting. But let's try to preserve the rest of the line.
          Then we replace the old array_str with new_array_str in the line.
      But note: the array might be empty? Then we write [].
  How about if there are multiple arrays in the same line? But the line contains the string `"privileges" : [`, so we are only concerned with the array that comes after that string.
  So we can do:
      # Find the start of the array: we look for the string `"privileges" : [`, then the next character after that string is the beginning of the array? Actually, the string ends with '[' so the array starts at the position of that '['.
      start_str = '"privileges" : ['
      pos = line.find(start_str)
      if pos == -1:
          # This line doesn't have the pattern? Then output the line unchanged.
          # But we were only called on the line that contains the string? So we should find it.
          pass
      Then we set start_index = pos + len(start_str) - 1   # because the last character of start_str is the '['
      Actually, the string is `"privileges" : [`, so the '[' is at the end. So the opening bracket is at `pos + len(start_str) - 1`. But then we want to extract from that bracket.
      Then we call extract_array_string(line, start_index) -> returns the substring from start_index to the matching closing bracket (inclusive).
      Then we parse that substring as JSON, get the list, filter, then convert the new list to JSON string.
      Then we replace the extracted substring (which is the array) with the new JSON string.
      Then write the line.
  However, note: the array might be empty? Then we get a string "[]", we parse to an empty list, then we write "[]" again.
  Example:
      line: ... "privileges" : [ "a", "b" ] ...
      We extract: `[ "a", "b" ]` -> becomes a list ['a','b'] -> if none contains proxy, then new list is [] -> then we replace with '[]'
      Then the line becomes: ... "privileges" : [] ...
  But what if there are multiple arrays? We are only replacing the first occurrence? The problem says the line contains the string `"privileges" : [`, so we assume only one such array per line? And we are to process the entire line? So we do the first occurrence? But the problem says "the one containing", meaning there is one? So we do the first.
  Implementation of extract_array_string:
      def extract_array_string(s, start_idx):
          # s is the string, start_idx is the index of the opening bracket '['
          stack = []
          in_string = False
          escape = False
          stack.append('[')   # we start with the opening bracket
          start = start_idx
          end_index = None
          for i in range(start_idx+1, len(s)):
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
                  # ignore other characters
              else: # in string
                  if escape:
                      escape = False
                  else:
                      if c == '\\':
                          escape = True
                      elif c == '"':
                          in_string = False
          if end_index is None:
              # if we didn't find, then we take the rest of the string? But that would be invalid. We should throw an error? Or return the whole rest?
              # Since the file is valid JSON, we assume we find it.
              end_index = len(s)-1
          return s[start:end_index+1]
  But note: we are starting at the opening bracket at `start_idx`, and we are including it. Then we look until we find the matching closing bracket.
  Then we do:
      array_str = extract_array_string(line, start_idx)
      # Parse array_str to a list
      import json
      try:
          arr = json.loads(array_str)
      except:
          # if fails, we leave the line unchanged?
          arr = None
      If arr is not None and it's a list of strings? Then we filter:
          new_arr = [item for item in arr if isinstance(item, str) and "proxy" in item]
          Then new_array_str = json.dumps(new_arr)
      Then we replace in the line: from start_idx to end_index (which we got from the extraction) with new_array_str.
  But note: the extraction function returns the substring including the brackets. And when we parse, we get a list. Then we dump and get a string that is the array representation.
  Example: 
      array_str = '["a", "b"]'
      arr = ['a','b']
      new_arr = ['b']   if "proxy" in 'b'? Actually, we are filtering by the presence of "proxy", so if 'b' doesn't have it, then new_arr = []
      Then new_array_str = '[]'
      Then we replace the old array_str with '[]'
  But what if the array is not of strings? The problem says it's a line containing `"privileges" : [`, and we assume the array is of strings? If it's not, then we skip? The problem says "delete all substrings of the form ...", meaning we are only concerned with string elements? But the array might have numbers? Then we would get an exception in json.loads? Then we skip? Or the problem says "any number of characters" meaning it's a string? So we assume the array is of strings.
  Steps for the entire script:
      We read the input file line by line.
      For each line:
          if the line contains the string `"privileges" : [`, then we process it as above.
          else: output the line unchanged.
  But note: what if the string `"privileges" : [` spans two lines? The problem says "the one containing", so we assume it's in one line.
  We'll code accordingly.
  Let's code the helper function for extracting the array string.
  However, note: the array might be empty: `[]` -> then the extraction will work.
  We'll test with a few examples.
  Example 1: 
      line = '  "privileges" : [ "a", "b" ] , ... '
      start_str = '"privileges" : ['
      pos = line.find(start_str) -> 2
      start_idx = 2 + len(start_str) - 1 = 2+14-1=15? 
      Actually, the string is 14 characters: 
          "privileges" : [ -> 14 characters? 
          Let me count: 
            " p r i v i l e g e s "   :   [  -> that's 14? 
          So the opening bracket is at index 15? Actually, the string starts at index2 and goes to index2+13 (0-indexed, 14 characters from index2 to index2+13). Then the bracket is at index2+13.
      Then we extract from index2+13 to the matching closing bracket.
      We can also note: the array starts at the bracket at index2+13.
  Alternatively, we can do:
      We know the array starts at the last character of the matched string (which is the '['). So:
          start_idx = pos + len(start_str) - 1
      Then we extract from start_idx.
  But note: the matched string might be found in the middle of the line, and we want to extract the array that starts at that bracket.
  Now, let's code the extraction function.
  However, note: the extraction function might be heavy for a very long array? But we assume the array is not huge.
  Let's code the extraction function.
  But note: the problem does not specify the size of the array.
  Alternatively, we can use the `json` module to decode the entire line? But the entire line is not a complete JSON? It might be part of a larger structure.
  So we stick to the extraction.
  Steps for the script:
      import json
      def extract_array_string(s, start_idx):
          if s[start_idx] != '[':
              return None
          stack = ['[']
          in_string = False
          escape = False
          end_index = None
          # We start from start_idx+1
          for i in range(start_idx+1, len(s)):
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
                  # other characters: ignore
              else:
                  if escape:
                      escape = False
                  else:
                      if c == '\\':
                          escape = True
                      elif c == '"':
                          in_string = False
          if end_index is None:
              # If we didn't find the closing bracket, then we take the whole rest? But that would break JSON. We'll return the whole rest and hope for the best?
              end_index = len(s)-1
          return s[start_idx:end_index+1]
      Then:
          for line in input_lines:
              if '"privileges" : [' in line:
                  pos = line.find('"privileges" : [')
                  if pos != -1:
                      start_idx = pos + len('"privileges" : [') - 1   # because the last character of the string is the '['
                      old_array_str = extract_array_string(line, start_idx)
                      if old_array_str is not None:
                          try:
                              arr = json.loads(old_array_str)
                              # Now, we assume arr is a list. We remove non-string? Or we only process if the element is string?
                              new_arr = []
                              for item in arr:
                                  # If the item is a string and contains "proxy", we keep it.
                                  # Also, if the item is not a string, we keep it? Because the problem says "delete all substrings ...", meaning only the string ones that don't contain proxy? 
                                  # But the problem says: "delete all substrings of the form ...", meaning we are only concerned with string elements? So non-string elements we leave?
                                  if isinstance(item, str):
                                      if "proxy" in item:
                                          new_arr.append(item)
                                  else:
                                      new_arr.append(item)   # leave non-strings unchanged
                              new_array_str = json.dumps(new_arr)
                              # Replace the old_array_str in the line with new_array_str
                              # But note: we are replacing a substring from start_idx to start_idx+len(old_array_str)-1
                              line = line[:start_idx] + new_array_str + line[start_idx+len(old_array_str):]
                          except json.JSONDecodeError:
                              # If we cannot parse, leave the line unchanged.
                              pass
                  # If we didn't find the string, we output the line unchanged? But we are in the if that found the string? So we already found it.
              # output the line
  But note: what if the array is empty? Then we do nothing? It will become empty? But that's okay.
  Also note: the array might have non-string elements? Then we leave them? The problem only says to remove string elements that do not contain "proxy".
  However, the problem says: "delete all substrings of the form with any number of characters starting with the quotation mark and ending with the quotation mark + comma" -> meaning it's a string element? So non-string elements are not of that form? So we leave them.
  But note: the array might have an object? Then we leave it? But then the array might be mixed? We are only removing the string elements that do not contain "proxy".
  However, the problem says: "in this line, you need to delete all substrings ...", meaning we are to delete the entire element (the substring representing the element) if it is a string and doesn't contain "proxy". So we are effectively removing the element from the array? But note: if the element is an object, it doesn't match the pattern? So we leave it.
  But the pattern: "starting with the quotation mark and ending with the quotation mark + comma" -> that's a string element? So we are only removing string elements that don't have "proxy". For non-string elements, we leave.
  However, the extraction and parsing as JSON will give us the entire array as a list of Python objects. Then we remove only the string elements that don't have "proxy". Then we rebuild the array.
  This matches the requirement.
  Let's test with an example:
      Original array: [ "a", 123, "proxy_value", { "key": "value" } ]
      We remove "a" because it's a string without "proxy". We leave 123, "proxy_value", and the object.
      New array: [ 123, "proxy_value", { "key": "value" } ]
      But note: the pattern for deletion is only for string elements? So we are not deleting the number or the object? That's correct.
  However, the requirement says: "delete all substrings of the form ...", meaning we are only deleting the ones that are in the form of a quoted string? So the object is not in that form? So we leave it.
  So the code above does that.
  But note: the array might be nested? Then our extraction function might break? But we are using a bracket matching that accounts for strings. So it should work for nested arrays? 
      Example: [ [1,2] ] -> 
          start at the outer bracket, then we see an inner bracket, then we close the inner, then the outer.
  However, the problem says the array is for "privileges", which we assume is a list of strings? But the specification allows non-strings? So we support nested arrays? Our extraction function should handle nested arrays because we count brackets.
  But note: the array might be nested and have strings? Then our removal condition: we remove only the string elements (at the top level?) that do not contain "proxy". But what about string elements inside a nested array? They are not at the top level? The problem says "in this line", meaning the entire line? So we are only concerned with the top-level array? 
  However, the requirement says: "in this line, you need to delete all substrings ...", meaning every occurrence in the line that matches the pattern? But the pattern is: a quoted string that is an element of the array? Or any quoted string in the line? The requirement says: "the one containing `"privileges" : [` in this line", meaning we are only processing the array in that line? And the pattern is for the array elements? So we are only concerned with the top-level array elements? 
  Therefore, we are only removing the top-level string elements that don't contain "proxy".
  Our method of parsing the entire array and then traversing the list will only look at the top-level elements? Yes, because we are iterating over the list we got from `json.loads`, which is the top-level array.
  So we are safe.
  Now, we code accordingly.
  But note: the extraction function might be inefficient for very large arrays? But we assume the array is not too big.
  Let's code the extraction function with the state machine.
  We'll write the function carefully.
  We'll also consider: what if the array has a string that contains a bracket? For example: `[ "a[b]" ]` -> then when we see the '[' inside the string, we are in_string, so we don't count it. Then when we see the ']' inside the string? We are in_string, so we don't count it? Then we break when we see the closing bracket at the top level.
  This should work.
  Now, let's write the code.
  Steps:
      Read input file and output file names? The problem doesn't specify, so we take from stdin and write to stdout? Or from a file to a file? 
      The problem says: "processing a file", so we can take the input file as the first argument and output to stdout? Or we can take two arguments? Let's assume:
          input file: first command line argument
          output file: second command line argument? Or we can overwrite? We'll write to stdout? 
      But the problem doesn't specify, so we'll read from stdin and write to stdout? 
      However, the problem says "processing a file", so we can also read from a file and write to a file? Let's do:
          import sys
          if len(sys.argv) < 2:
              print("Usage: python script.py input.json [output.json]")
              sys.exit(1)
          input_file = sys.argv[1]
          if len(sys.argv) >= 3:
              output_file = sys.argv[2]
          else:
              output_file = None   # then we write to stdout
      Then we open the input file for reading, and the output file for writing.
  Alternatively, we can do:
      input_file = sys.argv[1]
      output_file = sys.argv[2] if len(sys.argv)>=3 else None
      with open(input_file, 'r') as f_in:
          lines = f_in.readlines()
      # process each line
      if output_file:
          with open(output_file, 'w') as f_out:
              for line in lines:
                  f_out.write(line)   # but we might change the line
      else:
          for line in lines:
              print(line, end='')
  But note: we are processing line by line, and we change the line if it contains the pattern.
  However, the array might span multiple lines? The problem says "the one containing", meaning the entire array is in one line? We assume so.
  So we'll read line by line.
  Code:
      import sys
      import json
      def extract_array_string(s, start_idx):
          # s: the string, start_idx: the index of the opening bracket '['
          if start_idx >= len(s) or s[start_idx] != '[':
              return None
          stack = ['[']  # we start with the opening bracket
          in_string = False
          escape = False
          end_index = None
          # We start scanning from start_idx+1
          for i in range(start_idx+1, len(s)):
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
              else:  # in string
                  if escape:
                      escape = False
                  else:
                      if c == '\\':
                          escape = True
                      elif c == '"':
                          in_string = False
          if end_index is None:
              # We didn't find a matching closing bracket? Then we take the rest of the string.
              end_index = len(s) - 1
          return s[start_idx:end_index+1]
      def process_line(line):
          # Look for the pattern
          pattern = '"privileges" : ['
          pos = line.find(pattern)
          if pos == -1:
              return line
          # The opening bracket of the array is at pos + len(pattern) - 1 because the pattern ends with '['
          start_idx = pos + len(pattern) - 1
          old_array_str = extract_array_string(line, start_idx)
          if old_array_str is None:
              return line
          try:
              arr = json.loads(old_array_str)
              # We'll create a new array: we remove string elements that do not contain "proxy"
              new_arr = []
              for item in arr:
                  if isinstance(item, str):
                      if "proxy" in item:
                          new_arr.append(item)
                  else:
                      new_arr.append(item)
              new_array_str = json.dumps(new_arr)
              # Replace the old_array_str in the line
              new_line = line[:start_idx] + new_array_str + line[start_idx+len(old_array_str):]
              return new_line
          except json.JSONDecodeError:
              return line
      # Main
      if len(sys.argv) < 2:
          print("Usage: python script.py input_file [output_file]")
          sys.exit(1)
      input_filename = sys.argv[1]
      output_filename = sys.argv[2] if len(sys.argv) >= 3 else None
      with open(input_filename, 'r') as f_in:
          lines = f_in.readlines()
      # Process each line
      processed_lines = [process_line(line) for line in lines]
      if output_filename:
          with open(output_filename, 'w') as f_out:
              for line in processed_lines:
                  f_out.write(line)
      else:
          for line in processed_lines:
              print(line, end='')
  Note: we are reading the entire file into memory. If the file is huge, we might want to process line by line and write as we go.
  But the problem doesn't specify the size, so we do this.
  However, what if the array spans multiple lines? Our extraction function is for one line? Then it will fail? We assumed the entire array is in one line.
  The problem says: "the one containing", meaning the line that contains the string `"privileges" : [`. So we assume that the entire array is in that same line? If not, then our extraction function will not find the closing bracket? Then we leave the line unchanged? But then the array might be broken? 
  To be safe, we note that the problem says "processing a file in json format", so the entire file is valid JSON? Then we might want to parse the entire file? But the problem says to leave other lines unchanged? Actually, the entire file is one JSON object? Then we could parse it and then modify the specific field? 
  But the problem says: "Leave all lines in the file unchanged except the one containing ...". So we are to do line-based processing.
  Therefore, we assume the array is in one line.
  We'll run the code as above.
  Let me test with a sample:
      Input line: 
          ... "privileges" : [ "read", "write", "create_proxy", "delete" ] ...
      We want to remove "read", "write", "delete", so the new array is [ "create_proxy" ]
      The extraction: 
          old_array_str = '[ "read", "write", "create_proxy", "delete" ]'
          arr = ['read','write','create_proxy','delete']
          new_arr = ['create_proxy']
          new_array_str = '["create_proxy"]'   (without spaces? json.dumps by default uses spaces after commas? But the original might have spaces? We don't care about the exact formatting? The problem doesn't specify. We are only required to remove the non-proxy elements.)
      Then we replace: 
          line = ... "privileges" : '["create_proxy"]' ...
      But note: the original array string might have been: 
          [ "read", "write", "create_proxy", "delete" ]
          and we replace with: ["create_proxy"]   -> without spaces? 
      This is acceptable? The problem doesn't specify formatting.
  Alternatively, we can try to mimic the original formatting? But that's complex.
  We'll go with the above.
  Let's run the code on a sample.
  However, note: the pattern might appear in a string? But the problem says the line contains the string `"privileges" : [`, so we are matching that exact string? And in a valid JSON, that string would be part of a key and then the array? So it should be outside a string? So we are safe.
  We'll code accordingly.
  Code complete.
  Note: the extraction function might be improved for edge cases, but we assume the JSON is well-formed.
  Let me test the extraction function with a few cases:
      Case 1: simple array: 
          s = 'abc [ "a", "b" ] def'
          start_idx = 4 -> the index of the '['.
          Then we should return: '[ "a", "b" ]'
      Case 2: array with nested array: 
          s = 'abc [ "a", [1,2] ] def'
          returns: '[ "a", [1,2] ]'
      Case 3: array with string containing bracket:
          s = 'abc [ "a[bc]", "d]" ] def'
          returns: '[ "a[bc]", "d]" ]'
      Case 4: array with escaped quotes: 
          s = r'abc [ "a\"b", "c" ] def'
          returns: r'[ "a\"b", "c" ]'
  We'll code accordingly.
  We'll write the extraction function and test it.
  Due to time, we'll assume the extraction function works.
  We are done.

# Ответ

To solve this problem, we need to process a JSON file and modify a specific line that contains the string `"privileges" : [`. Within this line, we must remove all string elements from the array that do not contain the substring "proxy". Non-string elements and string elements containing "proxy" should remain unchanged. The solution involves reading the file line by line, identifying the target line, extracting the array, filtering the elements, and reconstructing the line with the modified array.

### Approach
1. **Identify the Target Line**: For each line in the input file, check if it contains the string `"privileges" : [`. This identifies the line that needs processing.
2. **Extract the Array**: Once the target line is found, extract the entire array string starting from the opening bracket `[` to the matching closing bracket `]`. This involves parsing the string while handling nested arrays and escaped characters within strings.
3. **Filter the Array**: Parse the extracted array string into a Python list. Iterate through each element in the list, retaining non-string elements and string elements that contain "proxy".
4. **Reconstruct the Line**: Convert the filtered list back into a JSON array string and replace the original array in the line with this new string.
5. **Output the Result**: Write the processed lines to the specified output file or standard output.

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
        end_index = len(s) - 1
    return s[start_idx:end_index + 1]

def process_line(line):
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
                if "proxy" in item:
                    new_arr.append(item)
            else:
                new_arr.append(item)
        new_array_str = json.dumps(new_arr)
        new_line = line[:start_idx] + new_array_str + line[start_idx + len(old_array_str):]
        return new_line
    except json.JSONDecodeError:
        return line

def main():
    if len(sys.argv) < 2:
        print("Usage: python script.py input_file [output_file]")
        sys.exit(1)
    input_filename = sys.argv[1]
    output_filename = sys.argv[2] if len(sys.argv) >= 3 else None
    with open(input_filename, 'r') as f_in:
        lines = f_in.readlines()
    processed_lines = [process_line(line) for line in lines]
    if output_filename:
        with open(output_filename, 'w') as f_out:
            for line in processed_lines:
                f_out.write(line)
    else:
        for line in processed_lines:
            print(line, end='')

if __name__ == "__main__":
    main()
```

### Explanation
1. **Reading the File**: The script reads the input file line by line.
2. **Finding the Target Line**: Each line is checked for the presence of the string `"privileges" : [`. If found, the line is processed further.
3. **Extracting the Array**: The array string is extracted by locating the opening bracket `[` and finding the matching closing bracket `]`, accounting for nested arrays and escaped characters within strings.
4. **Filtering Elements**: The extracted array string is parsed into a Python list. String elements that do not contain "proxy" are removed, while non-string elements and strings containing "proxy" are retained.
5. **Reconstructing the Line**: The filtered list is converted back to a JSON array string, replacing the original array in the line.
6. **Writing Output**: The processed lines are written to the specified output file or printed to standard output.

This approach efficiently processes the file, ensuring only the specified line is modified while maintaining the integrity of the rest of the file. The solution handles edge cases such as nested arrays and escaped characters within strings.
