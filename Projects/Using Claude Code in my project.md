---
fileClass: Project
Category: Planning
Status: Active
---
# Overview
- I want to switch over to using Claude Code to help me do my website

# Tasks
- How to use Claude Code [[Using Claude Code]]
- Install Claude Code
- Create a subscription plan
- Create a new folder Claude in my vault
- Create a new template Claude in my vault
- Create Claude.md and Claude.local.md
- Add Claude.local.md to .gitignore
- Put instructions in Claude.local.md about this Vault
- Proof of concept switching over to 64bit ints as your training exercise

# What I'm working on 1/22
- Install Claude Code
	- npm install -g @anthropic-ai/claude-code
	- Installed fast and can run Claude
	- Asking for an account
- Create a subscription plan
	- https://claude.com/pricing
	- Going with Max ($100/mo)
	- Note that max is 5x pro and the $200 one is 20x so 4x resources for 2x cost. Keep tabs on how much I use but start with this.
	- Used Knotty Yoga Chase account
	- It's installed and linked to my account
- Create a new folder Claude in my vault
	- Created it
- Create a new template Claude in my vault
	- Opened up Class / Project
		- Chose Manage fields / Project - Category
		- Add a value / Claude
	- Created template as New Note / Named it Claude
	- Chose Add fileClass and chose Project
	- Right click on the tab and choose Add missing fields at section...
	- Choose Add this field at the end of the frontmatter
	- Click on Category and choose the appropriate Category
	- Click on Status and default to Active
	- In frontmatter add:
		- Authors: Mason Bendixen
		- Last Updated: {{date:M/D/YYYY}}
		- Version: 0.1
		- tags: 
	- In QuickAdd Settings, add an entry for Claude and set the template and folder both to Claude
- Create Claude.md and Claude.local.md
	- Open powershell in C:\Users\mason\source\repos\knottyyoga
	- Ran claude
	- /init
	> I'd like you to be able to access C:\Users\mason\Documents\Obsidian\Knotty Yoga. This is my Obsidian vault. In particular, I would like you to work with me in plan mode on md files in C:\Users\mason\Documents\Obsidian\Knotty Yoga\Claude to make plans to do features. How do I give you access to this directory and note that this is where I want to put the MD files for plan mode?
	- This made these changes to .claude\settings.local.json
		- "additionalDirectories": [
		  7 +      "C:\\Users\\mason\\Documents\\Obsidian\\Knotty Yoga"
		- 9 +  },
		  10 +  "plansDirectory": "C:\\Users\\mason\\Documents\\Obsidian\\Knotty Yoga\\Claude"
		- Had to restart for these to take effect
- Proof of concept switching over to 64bit ints as your training exercise
- 
