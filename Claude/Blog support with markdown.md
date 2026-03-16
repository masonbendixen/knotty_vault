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

There should be a New Post button that brings up a blog authoring screen. There should be fields for Name, Author, a bool checkbox for Draft, and a date / time picker to choose when to post and then a button for Post Now that sets the post to post immediately at the current time. Then there should be a Save Post and Cancel buttons. Below these, there should be a left and right pane for authoring. To the left we should host ngx-markdown with syntax highlighting for authoring markdown and then on the right, we should show the fully rendered markdown. Clicking Save Post should save this entry to the database. The Edit button in the grid control should bring up the same page but with the existing data from the existing post populating the controls.

There should be a new top level menu entry that required no permissions called Blog. Clicking this should bring the latest Blog posts up that shows the five most recent blog posts with next and previous posts buttons at the bottom. At the top of the page, there should be buttons for navigating year / month / day with all three only having entries for items for which there are blog posts as well as an option for All Years / All Months / All Days. Selecting All Years automatically chooses All Months / All Days. Choosing All Months automatically chooses All Days. Blog posts should be rendered with the Name as a title, then the author, the date modified last, and then the body converted to HTML to be displayed. If the current user has the author_blog permission, we should also show a button Edit Post / Delete Post. Delete Post should prompt for confirmation.

# Place plan here