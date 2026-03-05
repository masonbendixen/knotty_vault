---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/5/2026
Version: 0.1
tags: 
---
# Overview

Please go into plan mode. Please use this document for your planning document. Do not create anything in .claude/plans. Use this file for planning and do not ask me for permission to modify this file. It is your plan file. Please leave this Overview section in tact and do your work in the sections below.

We want to add classes to the system with photo support. The classes table already exists on the server side with photo support. You should be able to fetch classes with the existing /api/get_table_rows/ endpoint and fetch scaled photos using the existing support. 

Let's modify the existing Our Classes menu so that it keeps the all classes first item but then populates the rest of the menu with the names of all of the classes. Let's have a page for all classes that has cards for each class with a photo to the left and the name and description to the right. Clicking on a class should take you to the page entry for that class in the same way that clicking the same item in the menu does. On the page for a given class, you should show the photo on top with the name and then the description below. Please be sure to add tests for any new code.

Please use the codebase and this document to generate your plan. Please create phases of implementation with check boxes next to them. Please do the layered architecture with database schema changes, CRUD table helpers, business logic changes (this could probably be added to PersonHelper), the endpoint. There shouldn't be server side work items for this change. Then the client stuff with the types, network access layer, components, and then wiring into the system.

# Add plan here