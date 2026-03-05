---
fileClass: Project
Category: Implementation
Status: Active
Author: Mason Bendixen
Reviewers: 
Date: 10/8/2025
Version: 0.1
tags: 
---
# Overview
In database_helper.cpp, RunInTransaction does all the Postgres calls inside a try/catch and essentially swallows all exceptions. This is bad news. The try/catch should be at the top level and should turn into an HTTP error or things that can handle the errors should do local handlers. This is a comprehensive work item though since a LOT of code has been written to assume that this does not percolate exceptions. We also need to add a lot of new tests all throughout the system. Ideally, this document will note all the callers and go and enumerate all the places that we need new test cases.

# Background Research
- Links to research notes with background

# Requirements
- Functional and non-functional requirements go here

# High-Level Architecture
Description of high level design goes here

# Detailed Design
APIs, classes, data structures, algorithms, database design, security, performance

# Alternatives Considered
- List other possible alternative solutions and why they were not chosen

# Future Work
- List of things out of the scope of this design that probably still need to be tackled
