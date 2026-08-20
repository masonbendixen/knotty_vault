---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 8/20/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I've been demoing the application getting it ready to deploy and have found a number of issues. I'm just going to list them. If you can try to group them into buckets to implement, ask me questions, and build an implementation plan to accomplish these, that would be great. Please use the code base and these documents for context:

- [[Component Inventory for Designer]]
- [[Splitting the server up into components]]
- [[Componentizing the frontend]]
- [[Converting the server to a multi tenant Saas architecture]]
- [[Deploying to AWS]]
- [[Home page work and cleanup items]]
- [[Tenant Theming and Branding]]
- [[Website Makeover]]

Here are the items that we need to improve before being ready to ship:
- After changing the contact email for the site, the footer doesn’t refresh until refreshing the page.
- The How test looks section does the font sizes in rem units. Can we have a toggle to switch this to points and pixels in addition to REM units?
- Font service doesn’t seem to persist in the Fonts admin portal. I set it, and it works, but returning to the page to add a new one, I need to Select Google for each one.
- We need to alphabetize the portal entries for the admin, staff, and personal profile portals
- Studio locations need a timezone associated with them (this should be noted in one of the documents)
- Let’s make the font weight for various things configurable in page settings. We currently hard coded it to semi bold for the menu font but it would be nice if that was configurable (generally the 100...900). It would be nice if there was both the number and a description like bold, normal, demibold, etc.
- The image carousel shows up above the banner. Let’s put this as a configurable item in the home page editor that we can move the location. I don’t also really love the cropping. Instead of forcing the aspect ratio to match one image, let certain images just be wider. Also, have an option to decide if we show the text.
	- I'd like an image carousel editor in the home page. You can name various image carousels and add images to them. Currently there is an image carousel table with just a single set of images. I'd like to support creating multiple names image carousels and have each carousel have a set of images and descriptions with CRUD type support and reordering. Each carousel should have a display name and description.
	- I'd like to add a new item to the home page portal with the type image carousel that lets you select from the list of names image carousels to display on the home page and be positioned.
	- I'd like an Image Carousels tab in the site theme admin portal editor that let's you order the carousels to be shown on the About / Gallery menu item page and have the ability to add / remove / reorder names carousel entries as well as choose whether the title and/or description is shown.
	- All these stuff should be persisted in the JSON site theme file.
- Membership, Upcoming Classes, and Upcoming Events should be different items that can be added to the home page and the position on the home page should be able to be ordered with the existing home page items. (Currently they just get added before or below all the configurable items). These will all be new types that can be chosen when adding an item in addition to carousel.
- The upcoming classes on the home page background should be –theme-surface-tint but the individual cards should be --theme-background
- Can the background image of banner / Come join the fun! be parallax scroll?
- Do you still have the access to Ryan's figma? If so, in Ryan’s design for Get started, there is an image with a Get Started with a fire font. This image, the text, and the button should all be stacked on top of each other, centered, and use the menu color scheme and be the full width of the page. The image is at Icon / GetStarted in Frame 127
- Can we get the icons for C:\Users\mason\source\repos\knottyyoga\server\knottyyoga_server\out\build\x64-Debug\src\database_helper\img tier_icon_solo.png and the other three images and use those as the icons for the Become a member on the home page and on the Memberships page
	- Can you make the membership panel on the home page look like the items when you click the Memberships menu item and go to that page?
- The That which doesn’t kill you makes you hotter on the footer should be an image and then get the image from Ryan's figma.
- Let’s change the About page from being just plain markdown to supporting alternating image / text blocks like the home page. And allow them to be reordered.
- Add an instruction on the favorite support for instructors on the instructors page
- What do preferences do in the Instructors portal?
- Have Our Classes go to the Browse all classes (for non logged in users) on the current Our Classes page and then have a My Classes that takes you to the current Our Classes page. Let's keep things the same as now for logged in users.
- On services, let’s add an image for each service and then show the image above each service in the Services page from main menu. Need to add image support to the database and then the Bookable Service panel in the product editor.
- Events don’t have an image. Let’s add an image. Let’s also add this as another configurable thing that can be placed on the home page. The home page shows a truncated version of the event session whereas the upcoming events page shows more. Let’s have the home page match the Upcoming Events page. Let's show the description from the product page.
- About / Our location with image / description. Have a button that triggers a map.
- On the Memberships page, let's call the page Memberships, not Shop. Let's also make the prices 


Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here