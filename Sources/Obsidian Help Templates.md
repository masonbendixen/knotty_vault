---
fileClass: Project
Category: Source
Status: Complete
tags:
  - obsidian
  - templates
---
# URL
- [https://help.obsidian.md/plugins/templates](https://help.obsidian.md/plugins/templates)

# Points
- Can have the following template variables that get replaced at template instantiation time
  - {{title}} - Title of the active note.
  - {{date}} - Today's date. Default format: YYYY-MM-DD.
  - {{time}} - Current time. Default format: HH:mm.
- Default format for time is YYYY-MM-DD
  - You can change this with a Moment.js format string
    - https://momentjs.com/docs/#/displaying/format/
  - You do this via something like {{date:YYYY-MM-DD}}
  - Note that M gives you a single or two digit month, MM always gives you two characters (0 prefix if needed), MMM gives you Jan/Feb/Mar…, and MMMM gives you January/February/March…, d gives you a zero based day of the week (0-6), dd gives you Su/Mo/Tu…, ddd gives you Sun/Mon/Tue…, dddd gives you Sunday/Monday/Tuesday…, A gives you AM/PM, a gives you am/pm, H gives military time hours, HH gives you military time hours with a 0 padded two digit value always, h gives standard time, hh gives standard time 0 padded to always do a two digit value, X gives Unix timestamp in seconds, x gives a Unix timestamp in milliseconds.


# Need further investigation
- things that need further investigation

