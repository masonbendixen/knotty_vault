---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 11/16/2026
Version: 0.1
tags: 
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I put the website with largely full functionality up with the Square sandbox and had my students try it. I'd like to take this feedback and have you convert it into a plan. I note higher priority items in comments.

Her is my feedback from my student Thomas Potthast. He's an M2 manager at Microsoft and has extensive feedback and his comments should be high priority.

Overall: functionality is all there, but it feels like I am managing database tables instead of going through user journeys. The most critical issue I found: It is far too easy to be repeatedly charged and end up with multiple subscriptions. Sharing and setting up a couple's membership is also complex and unfriendly. Besides these, the biggest opportunity I see is revamping user flows. Right now, I would be concerned about conversion from views to sign-ups.  

User Experience 

 Sign Up 

- Ideally the intro workshop button takes you to something filtered specifically to intro workshops. I would also consider a call to action for intros specifically, where if none are scheduled you give them a message to email and set something up.  
    

- I would suggest slightly expanding the textual description for the memberships. There are enough options on the page that it might be best to more explicitly call out the differences between silver and gold. Likewise, you may want to give an option to say if they are looking for membership for one, two, or family. That both lets you reduce the options on the page and gives you an opportunity to explicitly say couples and family membership are discounted. 
    

- Once the account is created, the homepage prompts you to plan your week. Going to it shows no eligible classes, because you don't have a membership. I would make the messaging clearer when there is no membership at all (and explicitly surface intro workshops), and give a clearer funnel for a membership. It's actually pretty tricky to get from that page to a membership sign up. 
    

- If I don't have a membership, that should be the top thing on the my account page. Likewise, I would recommend thinking about the list order for My Account based on user jobs. I likely want to use that page to sign up for new classes, pay or upgrade my membership, or see which workshops are coming up. Those should be easy to find.  
    

- The title to manage my membership should be "My Membership" (with a clear call out of I don't have one). My subscription took three scans through the list to figure out that was where I change my membership. 
    

- "My subscriptions" as a page title has the same issue. There's also an unnecessary click in the empty state on that page. If I don't have a membership, your goal is to minimize the amount of thinking I need to do in order to get one. Every click is a funnel dropoff. 
    

- I accidentally hit back and ended up on the purchase page again. I then accidentally hit the button to re-purchase. It looks like it would have charged me twice (two emailed receipts). I would make sure you have a transaction id assigned the first time and make sure that purchases can only happen once. Second submit should not charge. 
    

- It's way too easy to end up paying for the same membership twice. Membership when I am logged in should show me my current level and an option to change, not the non-membership checkout flow. 
    

Sharing /  Membership 

- I like that signing up for a couple's membership immediately prompts me to add the second person. However, the ux is wonky. It shows 1/2 seats assigned (presumably to me), but also a button that says assign to me. I would make it clearer that one is assigned to me and I am assigning the other. The sharing flow also has a ton of steps 
    

- There are no links in the email I send someone to share with them. The most important thing in that mail is making it easy for the person I am sharing with to create an account. Right now the email doesn't even have a URL.  
    

- When Bri created her account, she couldn't find the sharing request. Ideally this would have been embedded in a link in the email that she could click once, sign up/in, and then immediately be prompted to accept or deny the request. This was not a smooth flow.  
    

- I do not enjoy how separated the "sharing permissions" and "sharing membership" experiences are. It feels like I am managing database tables, not planning gym time with my wife. 
    

- There isn't a way to convert a silver membership to a gold one. It looks like it is cancel and re sign up. There needs to be an upsell/upgrade flow. 
    

Registering for Classes 

-  On the class detail page, I would suggest explicitly reinforcing that the one of goals of beginner (or regular) knotty yoga classes is to help you meet the prerequisites for the other classes. In that vein, it might be worth ordering the classes by requirement (or another way to make it immediately clear what is appropriate at each level of fitness). Right now if you have no baseline it seems confusing. 
    

-  I find it very frustrating that the entire schedule reloads when I selected one item in "My weekly plan". I want only that item to update. My intent is to quickly tap the classes I am planning on attending. 
    

-  The sign up page seems to only filter by day. It shows me classes that already happened today. It should filter by time as well.  
    

- The page that shows classes ("Our Classes") lets me sign up for classes that are already in my weekly schedule - despite it showing under "My Schedule". It should reflect that I am already planning on attending. 
    

-  Going to the aerial rope page and clicking through to scheduling led to a surprisingly slow loading experience for the class list. 
    

UI Polish 

- The favicon appears to be the default Angular one 
    

- Footer gets cutoff by the bottom bar on Android. Scrolling up then down fixes it 
    

- Get started button always seems to take me to the bottom of the list (past step 7) 
    

- I ended up with three memberships. It should not be this easy to accidentally do so.  
    

- The loading icon pushes the word processing down to the next line on mobile when you check out. It should stay on the same line.  
    

- I do not think people want to see the underscores in the tags on my subscriptions (gold_member, silver_member).  
    

- In your bio, I would suggest better emphasizing your own teaching and training experience. To me it drastically undersells your experience, and most readers have no idea who Ana forrest is. Beth's is a great example to follow.  
    

- For me, Gifford's bio also somewhat undersells his abilities.  
    

- It isn't really clear what favorite instructors does. I would expect to see their upcoming classes or another action, vs just seeing the list.  
    

- Change password seems to duplicate the wrong field. I am not sure if I expect to have to enter the old password at all since I am already logged in, but I definitely do not expect to do so twice. I expect the old password at most once and the new one twice.  
    

Admin Experience 

Enormous functionality, but I would personally want a clearer dashboard that gives me an overview of how the business is running – number of memberships, forecasted monthly revenue, last month's revenue, class utilization, classes that are commonly underutilized or usually at capacity. This is information that lets me make decisions. Right now the experiences are powerful but expose all the complexity from the start.  

Manage Data 

- Manage data gives me zero context as to what I am managing. It looks like this is the raw table editing frontend vs Manage Product's UI. After realizing that it made more sense, but even then some of these feel like unless I have the schema memorized I will break something. At a minimum I would consider explanatory text for schema fields and an experience for foreign keys that shows a preview of the record being edited.  
    

- I don't see any versioning or way to rollback changes that break things. Combined with the above, granting access to anyone seems very risky.  
    

Manage Products 

- On my first trip to manage products, I immediately hit a login prompt despite being logged in.  
    

- The gigantic grid of abilities is exceedingly non-intuitive. I would recommend ~5 subheaders grouping closely related activities (e.g. something around staffing, something around staffing, something about pricing / vouchers / comps, something about usage and monetization.  
    

- When searching attendance plans, I would suggest ensuring all users show, even those who don't have one. I was very confused when I could not find Bri.  
    

- I tried to figure out how I would create a one-off workshop (e.g., visiting artist). I could not.   
    

- As a tool, pricing overview is really nice. It's cool that you can set pricing by role. I would consider making it clear that's what the big table does, and that Standard is the default if not set.  
    

- I would really like more of the utilization experiences (enrollment trends, open seat heatmap, etc) to be both forward looking as well as backwards. Historic can use checkin data and forward looking can use signup/registration data. It looks like functionality mostly exists, but it is both separate and somewhat non-intuitive to find.  
    

- Nit: LLMs like to leave references to previous decisions in the output. I would do a quick pass to look for any of them. One example: On the Site Theme page, there's the string "Type it exactly as you want it to read — it is no longer forced to capitals." 
    

Architecture / Tech 

- I have mixed feelings on Session Tokens vs client-side JWT's for auth. This site is very unlikely to hit the scale where that will be an issue, but I would be a bit concerned about any cross-replica edge cases (e.g. the backing information is in memory and the node fails over to another node, will the session token's use info and claims still be there). I appreciate setting the cookie as http only though.  
    

- Back and forth buttons work somewhat inconsistently throughout the site.  
    

- Having what is effectively a raw DB api is a very large attack surface and exposes plenty of details about your site. I would have the most cooperative model you can red-team it.  
    

- I would consider moving all the inline style out of your returned index document to a stylesheet.  
    

- It looks like all the JS chunks start loading after the initial document returns. I would consider putting the appropriate references to them in the HEAD of your document and streaming that back as soon as possible. That could get you back 60 ms or so of FCP / LCP / PLT on my machine.  
    

- I would be a strong advocate of having a server rendered non-logged in homepage. Not every SEO crawler supports JS, and it can be used to give a better user experience.  
    

- That site_info call is very late in the boot sequence and is not quick (589ms start time and 1.3s response on my machine). I would look at what is driving it and consider a caching story at least. It seems to be public strings, which can be served from memory with expiry.  
    

- The font requests starts a bit late. I would see about pulling it in sooner to avoid fouc.

END THOMAS POTTHAST FEEDBACK

Here is feedback from a student and teacher at my studio, Gifford Cheung:

**

# Summary

Along with a number of observations, probably my biggest idea is to ask if the site can help newbies get information and explore without feeling the burden of committing to a subscription or even setting up an account. This would apply most to the Getting Started as well as the check out flow.

Some of the information architecture deserves some thought as well: moving Gallery out of About, surfacing the Weekly Schedule and Calendar, and thinking a bit about the word “Services”.

I have a set of [5 personas](https://docs.google.com/document/d/1Ad-E-2IWAg87DmZNZUnevD2ciiZSLWQoJSCKwL9wnwI/edit?tab=t.z9beu2t8cwhz) that I generately quickly to help me evaluate. Presumably, they could be applied with an agent-based analysis, but for now, I generally walked throught the site myself.

# Main site

## Overall

- Consider adding a search feature? (Not a strong recommendation)
    
- Make the use of the word "Our" consistent. Sometimes the About page has items as "Our" and sometimes not. We have "Our Classes" but we have Services (with page title "Our services").
    
- Visual - links are styled in blue, do you want them to be a different color, conforming to your design system?
    

## Landing page

- Consider also putting the nice red button "Show me how" button under "Get into the best shape of your life, the right way."
    

## Getting Started

- Great page to have in general
    
- Feel a little heavy on commitment right away to pick a membership, plan, and get highly involved. How about something light:
    

- Most of the steps include information and exploration and only the last step is a commitment: "Sign Up for an account and start browsing our members classes" or something like that.
    

- Consider adding a Pre-requisites Step here or a New Student FAQ elsewhere
    

- Physical requirements
    
- What to Wear (including what not to wear)
    
- What to Bring (nothing!, no mat)
    

## About

- Consider moving Gallery to the top level near Blog. Yes, it helps you understand what KY is about, but you might start using the Gallery in a way that is similar to the Blog, current event photos, etc...
    

## Classes

- Weekly Schedule and Calendar
    

- Unbury these pages a little and consider making them siblings of All Classes. Currently, its navigation is not consistent with the other pages, it exists as red buttons within a different page while most other content is available from the top navigation.
    
- If possible, improve performance so that you don't have to wait 3+ seconds to see the schedule
    

## Services

- List the Providers alongside the services so that you can, for example, see “Our Message therapists: Caleb, etc…” and click on profiles to see more information.
    
- (Minor question, not necessarily substantiated) Massage services might have a discoverability challenge. This could be unclear if people think that Knotty Yoga is a typical workout studio. It's good that you have that on the Landing page, curious if a better name the top would help too.
    

## Upcoming Events

- Consider calling it Events instead of Upcoming Events
    

## Memberships

- Unsure why "Browse bookable services →" is at the top of the page. I am assuming a visitor would not be looking for bookable services in the Membership page.  
    Memberships > Subscribe
    
- Instead of taking you to the Sign in page first, consider showing the form on shop/subscribe/4 instead so that prospective shoppers can look over the financial commitment before creating an account. I'm taking inspiration for airline sites and Amazon where you can see what's in your cart and don't have to log in or sign up for an account until necessary.
    

## Account

- There is a tag that shows your current subscription, recommend adding details from My Subscriptions such as "Active" and subscription dates  
    Account > Favorite Instructors
    
- Recommend showing classes and events that your Favorite Instructor is teaching  
    Account > My Subscriptions
    
- Unsure how the “Search people to assign” feature works. I typed "Tiana" and didn't know if I was supposed to click a button or press Enter. Consider how to add more affordances: a Search button etc. Or, if you are likely to have sharing permissions with a small number of people, show that list here and have the subscriber choose from the existing list. E.g., If sharing is set up, show the list of people shared with and a link under that says, "Add a person". Else, show an Empty State list with a call to action to "Add a person to your sharing list".
    

# Admin

## Admin > Tables

- Instead of a dropdown, how about a left navigation panel listing each item?
    

##  Admin > Manage Products

- Recommendation: Categorize the products so that it’s easier to manage the big collection of products
    

END GIFFORD FEEDBACK





Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Place plan here