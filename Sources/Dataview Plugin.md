---
fileClass: Project
Category: Source
Status: Complete
tags:
  - obsidian
  - dataview
---
# URL
- [https://obsidian.rocks/dataview-in-obsidian-a-beginners-guide/](https://obsidian.rocks/dataview-in-obsidian-a-beginners-guide/)

# Points
- SQL like query support    
- Supports four formats:
  - Calendar
  - List
  - Table
  - Task
- You can basically say:

> \`\`\`dataview
> LIST
> FROM \#tag
> \`\`\`

- You can have nested tags and query for just the prefix
- FROM supports AND, OR, and parentheses
- Here are various items you can use that are built in
  - file.name - As seen in Obsidian sidebar  
  - file.ctime - date file was created with time  
  - file.cday - date file was created without time  
  - file.mtime - date file was modified with time  
  - file.mday - date file was modified without time  
  - file.tags - list of tags where each nested tag has each prefix listed as a separate entry  
  - file.etags - list of tags where the prefixes for nested tags aren’t split out  
  - file.inlinks - list of all incoming files that link to this file  
  - file.outlinks - list of links to other notes  
  - file.tasks - list of all tasks  
  - file.lists - list of all list items in this file including tasks  
  - file.starred - uses the obsidian core starred plugin  
- Can was “TABLE attribute as “friendly_name”
- If you have created attributes, you can use “WHERE attribute = “value” “ to filter
- Can say things like “WHERE file.mtime >= date(today) dur(1 week)” to look back on things within the last week
- Can saw “SORT attribute ASC” to sort results
- Can use “LIMIT numeric_value” to limit to a certain number of results
- List files created in the last week
> \`\`\`dataview
> TABLE file.ctime AS "Created"
> WHERE file.ctime >= date(today) - dur(1 week)
> \`\`\`
- List tagged notes in order of last edits
> \`\`\`dataview
> LIST FROM #tag
> SORT file.mtime DESC
> \`\`\`
- List unlinked files
> \`\`\`dataview
> list from [[]] and !outgoing([[]])
> \`\`\`
- List Workout Logs (or any other habit!)
> \`\`\`dataview
> TABLE workout
> FROM "2 Areas/Journal/Daily Notes"
> SORT file.ctime DESC
> LIMIT 14
> \`\`\`
- List Completed Tasks
> \`\`\`dataview
> TASK
> WHERE !completed
> LIMIT 10
> GROUP BY file.link
> SORT rows.file.ctime ASC
> \`\`\`



# Need further investigation
- things that need further investigation

