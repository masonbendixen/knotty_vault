---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 4/13/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

The Bookable Service Foundation.md file currently has the scheduling algorithm detailed but it has some issues. The core thing is that we need to think of a model based on maximizing the number of complete scheduling units that can be fit into an open block of time. Here are the constraints:

- The core unit of scheduling is an hour
- We allow booking 60min, 90min, and 120min blocks which are 1x, 1.5x, and 2x
- There is a mandatory buffer window between blocks
- The idea is that if one of these units is cancelled, we should allow it to be rebooked flexibly
- 60min and 120min are highly compatible if we require a double buffer between 120min massages. By requiring a double buffer, if a 120min massage gets cancelled, that slot can be filled with either another 120min massage OR two 60min massages with a buffer after each in the same amount of time as a 120min massage with a double buffer.
- 90min massage is complicated
	- For scheduling a 90min massage, there are two conditions:
		- Previous massage was the first massage or the block or not a 90min massage
			- Require a double buffer after this massage
		- Previous massage was a 90min massage
			- Previous 90min massage had a doubl
	- if the previous massage is not 90min or this is the first massage of the block, we should require a double buffer after the massage
	- 

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here