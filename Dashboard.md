---
Category: Design
---
#Project

```dataview
TABLE WITHOUT ID
file.link AS Planning
FROM "Projects"
WHERE Category = "Planning"
SORT file.mtime DESC
LIMIT 10
```

```dataview
TABLE WITHOUT ID
file.link AS Documentation
FROM "Projects"
WHERE Category = "Documentation"
SORT file.mtime DESC
LIMIT 10
```

```dataview
TABLE WITHOUT ID
file.link AS Research
FROM "Projects"
WHERE Category = "Research"
SORT file.mtime DESC
LIMIT 10
```

```dataview
TABLE WITHOUT ID
file.link AS Sources
FROM "Sources"
WHERE Category = "Source" AND length(file.inlinks) = 0
SORT file.mtime DESC
LIMIT 10
```

```dataview
TABLE WITHOUT ID
file.link AS Design
FROM "Projects"
WHERE Category = "Design"
SORT file.mtime DESC
LIMIT 10
```

```dataview
TABLE WITHOUT ID
file.link AS Implementation
FROM "Projects"
WHERE Category = "Implementation"
SORT file.mtime DESC
LIMIT 10
```

```dataview
TABLE WITHOUT ID
file.link AS Event
FROM "Projects"
WHERE Category = "Event"
SORT file.mtime DESC
LIMIT 10
```