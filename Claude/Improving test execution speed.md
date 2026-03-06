---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/6/2026
Version: 0.1
tags: 
---
# Overview

Please go into plan mode. Please use this document for your planning document. Do not create anything in .claude/plans. Use this file for planning and do not ask me for permission to modify this file. It is your plan file. Please leave this Overview section in tact and do your work in the sections below.

Going back to improving the execution speed of my tests, I was curious if there is anyway to try and speed this up. The changes to change the ACID type setting on a transaction really didn't speed things up and even made things a bit slower. I was curious if we can change the settings on the test database creation in code (not postgres conf changes) that might speed things up. For the test execution database, I really won't have concurrent access and I don't really need things even to be durable. Things could just operate in memory and not even be committed to disk. We do need to support reads and writes but the tests are all serialized and don't run concurrently. Short of that, we could explore using a single transaction for tests and use something like checkpoints that we can revert to. I'm open to any ideas you have for speeding up test execution speed knowing that we don't need most of the ACID things we care about in production and that test execution is not in parallel. 

Please use the codebase and this document to generate your plan. Please create phases of implementation with check boxes next to them.

# Steps
- List of steps to accomplish this task.