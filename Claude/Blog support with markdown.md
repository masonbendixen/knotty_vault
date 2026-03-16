---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 3/16/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I'd like to add a blog to the website. I'd like to add an author_blog permission that allows CRUD style blog post creation. Blog posts should be stored in a table called blog_posts. Besides an auto incrementing 64 bit integer id called id, they should have a name, author, body, bool draft, created_at_us, modifed_at_us, and a post_at_us. In addition, the table should be marked as having photo support.

On the client side, under the admin tab, their should be a new menu item called Blog Posts that is guarded by the author_blog permission. Going to this page should have a row of blog posts with pagination sorted by created_at_us. The user should be able to choose a year and month and have an option for All Years and All Months (choosing All Years automatically selects All Months) that let you filter posts. There should be edit and delete icons for each post.

There should be a New Post button that brings up a blog authoring screen. There should be fields for Name, Author, a bool checkbox for Draft, and a date / time picker to choose when to post and then a button for Post Now that sets the post to post immediately at the current time. Below these

# Steps
- List of steps to accomplish this task.