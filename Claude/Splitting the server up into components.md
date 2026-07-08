---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/8/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

This web server has grown into a product with a lot of interesting code. I have some other websites I'm interested in creating. I was thinking I could factor a lot of this code into components that could be shared by a set of webservers. I'm trying to figure out how best to do this.

I currently am using CMake and conan. I have my code in Gitlab. I have my Obsidian vaults in Github. I wasn't sure if it would be a good idea to package up components and place them in a repository like Conan. I think I read Gitlab has a repository as well. Not sure about Github. Some friends are interested in working on this with me and would most likely want to work in Github so they can point potential employers to their work. I am open to leaving my server and the components in Gitlab or moving them to Github. If you can help me think through this process, that would be great.

I'm also trying to figure out what code to factor out into component(s). I feel like all the authentication stuff (business_logic/auth), images (business_logic/images), endpoint helper stuff (endpoints/ enpoint_auth_helper.* endpoint_test_helper.\*), scheduler, most of the stuff under sql_util minus some of the table_helpers stuff (which should probably be grouped), util (factoring out the server specific secret keys and values), and a good chunk of the stuff under test. I'm thinking some of these could be separate components that are

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here