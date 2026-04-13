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
			- Previous 90min massage had a double buffer after it
				- Require just a single buffer after this massage
			- Previous 90min massage did not have a double buffer after it
				- Require a double buffer after this massage
	- This will cause concurrent pairs of 90min massages to occupy three hours with three buffers
		- If one of the pair members is cancelled, it can only be replaced with another 90min massage, a 60min massage is not allowed since that would cause a 30min gap
		- If both are cancelled, they can be replaced with either three 60min massages OR a 120min and 60min massage (in either order)
	- The complicated case is if we have three 90min massages A, B, C booked in a row
		- This would look like A, double buffer, B, single buffer, C, double buffer
		- If B is cancelled, it can only be replaced with either a 90min massage
		- If B is cancelled and either A or B as well, either block can be replaced with:
			- Two 90min massages
			- Three 60min massages
			- A 60min and 120min massage in either order
		- The key is that that alternating double buffers for 90min massages makes them compatible with 60min and 120min massages
	- A 90min massage next to another 90min massage is compatible with 60min and 90min massage
	- A 90min massage sandwiched between 60min or 120min massages is essentially an island and can only be replaced with another 90min massage if it is cancelled
	- A 120min massage that is cancelled with no adjacent free slots can never be replaced by a 90min massage as that would create a 30min hole
	- A 90min massage can only be booked into an available window if there is either exactly 90min plus single or double buffer depending on previous massage state OR the window is at least 180min plus 3xbuffer long
	- Please note that buffer requirements after a booking don't count if this is the last booking in an availability block (meaning that there isn't 60min left after the last booking in this block)

The other thing I would like to alter is to tweak requirements at the end of an availability window. In general, if we have a window that is less than 120min but can fit a 90min massage, we require the 90min massage and don't allow the 60min massage. For the last block of time in an availability window if it is less than 120min but greater than 90min, I would like to allow either a 60min or 90min massage as 60min is the most popular and pairs well for availability.

Please copy the existing scheduling algorithm into this document with the examples and override it to support the changes listed in this document but leave all the rest unchanged. After we iterate and get it completed, we can work on an implementation plan. After we have implemented the implementation plan, we should modify Bookable Service Foundation.md to remove the scheduling algorithm from that document and replace it with a reference to this document. This should be the final step in the implementation plan.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here