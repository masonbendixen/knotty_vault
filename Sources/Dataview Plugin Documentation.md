---
fileClass: Project
Category: Source
Status: Complete
tags:
  - obsidian
  - dataview
---
# URL
- [https://blacksmithgu.github.io/obsidian-dataview/](https://blacksmithgu.github.io/obsidian-dataview/)

# Points
- There is an inline Data Query Language (DQL) that you say ‘= {DQL}’
- Access the current page via this.
- Access another page via \[\[{page_link}\]\].
- You can say this.file.{any_of_the_file_properties}
- Date objects have properties for year, month, weekyear, week, weekday, day, hour, minute, second, millisecond
- There are durations that are of the form “\<time\> \<unit\>” like “6 hours” or “6hr7min”. Can say dur(\<time\> \<unit\>) and things like dur(5 days 12 hours) are also supported
- Supported values in fields can also be links \[\[\<link\>\]\]
- There are two ways to do lists:
> \-\-\-
> key3: [one, two, three]
> key4:
> - four
> - five
> - six
> \-\-\-
- You can also do them inline like: [key: value1, value2, … valueN]
- You can nest fields to create objects and access fields of objects like obj.field
- There are tasks that have the following fields:
  - due- date when the task is due
  - completion - date completed
  - created - date when the task was created
  - start - presumably when the task was started. Not sure how this is determined.
  - scheduled - again, not sure how this is determined.
  - There are also a large number of implicit fields for tasks. See the documentation:
https://blacksmithgu.github.io/obsidian-dataview/annotation/metadata-tasks/
- Queries support the following things:
  - Choosing an output format / query type (TABLE, LIST, TASK, CALENDAR)
  - Can say WITHOUT ID to not have the link be shown as an implicit column. file.link gets you the same result as the implicit value but let’s you choose the name of the column.
  - LIST creates a bullet point list of the items. Can optionally have a single field that is concatenated with the link (which is separated by a colon)
    - Can create nested lists by using a GROUP BY (the GROUP BY field will be the top level list items and the listed item will be sub lists under each heading)
  - TABLE creates a table with the File being an implicit column that can be removed via WITHOUT ID.
    - Use AS on each field to give a custom column name
  - TASK queries tasks
    - Normally, the dataview stuff is readonly but if you check a task through the dataview query, it checks it at the source!
    - Maintains nesting of tasks by default.
  - FROM - Optional source. Can only be one and it must immediately collow the query type if present. A source like a tag, folder, single file, links, or a combination of these
    - Folders are specified by putting the folder name in quotes
    - Tags are specified by #{tag_name_optionally_nested}
    - Specific files can be indicated with quotes like a folder. In general, you don’t need to specify the .md but if a folder and file both exist with the same name minus the extension, you must use the extension to differentiate.
    - You can specify a link with \[\[{link_name}\]\]. This could optionally be routed through a function like outgoing \[\[{my_link}\]\]
    - Can combine things like (\#tag1 OR \#tag2) AND (\#tag3 OR \#tag4)
  - WHERE - Optionally filter results
  - SORT field ASC/DESC
  - GROUP BY - aggregates groups matching a trait into single rows (usually using aggregation functions to process them). Can use AS to name things here which is useful for computed fields.
  - LIMIT - limits the result count
  - FLATTEN - splits up one result into multiple results based on a field or calculation. This will cause the result set indicated to cause one row per item to be generated in the results so something like links can be processed.
  - DQL is different in that it is executed top to bottom, line by line. The result of the previous line is passed to the next. So you can have multiple WHERE, GROUP BY, and FLATTEN sections.
  - Expressions are anything that isn’t a query type or data command
    - Fields
    - Arithmetic
    - Comparisons
    - List / Object indexing - value[index_or_quoted_field_name]. Can also use object.{field_name}
    - Function calls
    - Lambdas - (arg1, arg2,... argN) => {expression using args}
    - Links- can access properties of a link like [[{linked_page}]].{page_field}
    - Literals
      - String, bool, numbers, links
      - \[\[\]\] - link to current file
      - \[1, 2, 3\] list that doesn’t need to be numbers
      - {key1: value1, key2: value2, … keyN: valueN}
      - date({date})
        - Can be a literal date value
        - today - current date without time
        - now - current date with time
        - yesterday
        - tomorrow
        - so{w/m/y} - date that is start of week/month/year
        - eo{w/m/y} - date that is end of week/month/year
      - dur({dur})
    - Functions
      - object(key1, value1, key2, value2, … keyN, valueN)
      - list(val1, val2, … valN)
      - date(text, format)
      - dur(any)
      - number(string) - pulls the first number out of the string
      - string(value) - attempts to force the value into a string representation
      - link(page, \[display\]) - creates a link to the given page with an optional different text to display for the link.
      - embed(link) - generally used to embed an image
      - elink(url, display) - created an external link with optional text to display instead of the link
      - typeof(any) - shows the type of the item like number, string, array, object, date, duration
      - Numeric - round(number, \[digits\]), trunc(number), floor(number), ceil(number), min(...), max(...), sum(array), product(array), reduce(array, operand), average(array), minby(array, function), maxby(array, function)
      - Objects / Arrays / Strings
        - contains
          - icontains is case insensitive
          - For objects, this returns true if the object contains the given key
          - For arrays, this returns true if the given item is present in the array
          - For strings, this does a substring match
        - extract(object, key1, key2, … keyN)
          - Returns a new object with just the keys specified (with the values)
        - sort(list)
        - reverse(list)
        - length(object|array)
        - nonnull(array) - returns a new array with all the nulls removed
        - firstvalue(array) - returns the first non null value from the array
        - any(array) - returns true if any of the values in the array are truthy
        - all(array) - returns true if all the values in the array are truthy
        - none(array) - returns true if none of the values in the array are truthy
        - join(array, \[delimiter\]) - returns a string concatenating all the values in the array with an optional delimiter. Default delimiter is “, “
        - filter(array, predicate) - returns a copy of the array with just the items for which the predicate returned true.
        - unique(array) - returns a copy of the array without duplicate values
        - map(array, func) - returns a copy of the array with func applied to each item
        - flat(array) - takes subarrays and flattens them to a single array. flat(list(list(1, 2, 3), list(4, 5, 6))) would be the same as list(1, 2, 3, 4, 5, 6)
        - slice(array, \[start, \[end\]\]) - returns a new array containing the items from start index to end index not including the end index item. End is optional. Can specify minus values that go from the end.
      - String operations
        - regextest(pattern, string) - returns true if any part of the string matches the regex pattern
        - regexmatch(pattern, string) - returns true if the full string matches the regex pattern
        - regexreplace(string, pattern, replacement) - replaces all instances in string of the pattern matching with the replacement string. Can use $1 and so forth to refer to capture groups.
        - replace(string, pattern, replacement) - string replaces pattern with replacement.
        - lower(string)
        - upper(string)
        - split(string, delimiter, \[limit\]) - where delimiter is a regex. Normally, the regex match is omitted but if there are capture groups, each group is included in the result.
        - startswith(string, prefix)
        - endswith(string, prefix)
        - pad\[left|right\](string, length, \[padding\]) for strings that aren’t the given length, padding will be inserted righ or left. Default padding is space.
        - substring(string, start, \[end\]) - like slice but with strings
        - truncate(string, length, \[suffix\]) - makes the string at most length characters long including the given suffix that defaults to “...”
      - Utility
        - default(field, value) - if field is null, return value. This is vectorized to go through lists and replace null values with default. If you want non vectorized, use ldefault.
        - display(string) - converts a string as DQL into the just text representation so something like display(link(“path/to/file.md”)) would turn into “file”
        - choice(bool, left, right) - returns left if true or right if false
        - hash(see, \[text\], \[variant\])
        - striptime(date) - removes time component of date
        - dateformat(date|datetime, string)
        - durationformat(duration, string)
          - S - Milliseconds
          - s - seconds
          - m - minutes
          - h - hours
          - d - days
          - w - weeks
          - M - months
          - y - years
          - ‘{text}’ - stuff you want to pass through
        - currencyformat(number, [currency])
        - localtime(date)
        - meta(link) - returns an object with metadata about a link with the following fields
          - meta(link).display - display text of link
          - meta(link).embed - whether or not this is an embedded link (created via !\[\[{my_link}\]\])
          - meta(link).path
  - Tasks with Dataview
    - https://blacksmithgu.github.io/obsidian-dataview/annotation/metadata-tasks/
    - You can use the following fields: due, completion, created, start, scheduled 
    - completed is a bool property on a task that indicates if the task has been completed
    - fullyCompleted is a bool property on a task that indicates if the task and all of its children have been completed
    - path is the path to the file containing this task
    - Each note has a tasks property that can be used to query all the tasks of that page
    - There are a lot of properties associated with tasks

# Need further investigation
- things that need further investigation

