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

We need to add instructors to the system with photo support. We need to create a new role called instructor and a permission called instructor. We need to add a table for instructors called instructors. This should have an auto incrementing 64bit serial counter for the primary key called id. We should have a foreign key reference to a person since all instructors are people. We need another column called bio that is a varchar. This should be a table that shows up in the admin console as a nested table under people. It should also have photo support. The primary key should be readonly and the bio should map a long text html control.

We need a public endpoint to enumerate the instructors with first name, last name, and an id for the photo to be able to request a scaled photo. This should be callable to all users and called get_instructors.

I want to add Instructors to the About dropdown in the main menu. It should bring up a page that says Instructors and then has a set of cards for each instructor with a decent sized photo to the left (maybe )

# Add plan here