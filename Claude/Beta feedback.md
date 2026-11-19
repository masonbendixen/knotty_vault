---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 11/19/2026
Version: 0.6.1
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

Jennifer Schulte offered this feedback:

So far, I’ve only found one minor kink - when selecting “Browse services” from the My Bookings page, it somehow takes you to the Membership page. Although it does show “Browse bookable services” link on the top, it is just a bit confusing… Don’t know why it does that take you to the “Our Services” page.

Also, when navigating to a different page from the Get Started page and hit the back button on the browser, it does not take u back to where u were. For example, select “Plan your weekly schedule” on the Get Started page, then hit the back button on the browser, you’ll notice that it does not take you back to the top of the Get Started page, instead of the “Plan your weekly schedule” section. Same behavior when u select any other sections on this page.

END JENN FEEDBACK


START ASHLEY FEEDBACK

The site looks great!  I clicked around as a basic member and had a few notes (mostly on content and links, though I did find one cart issue).  I was logged in as another.rainy.night@gmail.com for almost everything - I also made another.rainy.night+bonus@gmail.com for testing.

Schedule list (home page top, and https://dv1tgxa9ok30f.cloudfront.net/my/my-schedule ) – I got confused by the statuses – there’s a small checkbox to show I’m going, and a big button that says “I can’t make it”, so at first glance I assume that I’m going to class on Monday & Weds and can’t go on Tues & Thurs, because the buttons are large in their own column.  The display at https://dv1tgxa9ok30f.cloudfront.net/classes  with the buttons with checkboxes is great.  Though I did rsvp no for Weds Nov 4th and it’s showing up with my note on the home & my schedule, but as un-rsvp’d on the Classes page. 

Could the My Schedule view at https://dv1tgxa9ok30f.cloudfront.net/calendar?view=month have an option to show just the classes I've opted in to?  I picked just weds & thurs as my weekly options, but I still see all the classes on Sun & Mon in the calendar (though the checkboxes are unchecked).  
 
Maybe a way to upgrade your membership mid-month?
 
The Browse Services link at https://dv1tgxa9ok30f.cloudfront.net/my/events goes to the membership shop at https://dv1tgxa9ok30f.cloudfront.net/shop instead of Bookable Services at https://dv1tgxa9ok30f.cloudfront.net/shop/services. 

Feature request - I don't think anyone is going to have tons of linked accounts.  Could it be a dropdown on the subscription sharing page at https://dv1tgxa9ok30f.cloudfront.net/my/subscriptions/5 that shows your available linked accounts, instead of having to search for them?
 
Would you link the cancellation policy from the cancellation warning on https://dv1tgxa9ok30f.cloudfront.net/my/events (I’m looking in Upcoming Bookings). 
 
I put two massage things (a one-hour and a two-hour) in my cart & paid for them together.  Then, I cancelled the one-hour massage and the site gave credit for the full cart amount instead of the $160 for the one-hour massage.  It let me book things using the full credit afterwards.  I like the gift card setup - it's great
  
The schedule link from the class detail page (https://dv1tgxa9ok30f.cloudfront.net/classes/2) goes to the monthly calendar at https://dv1tgxa9ok30f.cloudfront.net/calendar?view=month, but that calendar is auto-filtered by your classes, so if you’re not in it then it won’t show up and you don’t know when it happens.  It’s also the only page for me that takes a while to load for me.  Could you have the weekly info directly on the detail page, e.g Monday & Weds 6-7; Sat 10 – 12?   (It’s also tricky to find the list of all classes.)
What about a ‘Book with Caleb’ link from his card on https://dv1tgxa9ok30f.cloudfront.net/providers? 

END ASHLEY FEEDBACK


Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Revision notes and working agreement

- **v0.6.1 (11/19/2026)**: found while hand-testing Phase 1. The Class Schedules details card now shows the class's name as its heading, laid out as Mason asked: photo left; name; facts on one line; description underneath (*Class Schedules: the class card*, under 1.6). Before, the name appeared only in the **Class** picker. *Testing Phase 1 locally* now expects it, and its build setup reflects the 11/18 pin bump. Part C now also expects **Failed to load requirements** for Kit. The Requirements card's read needs `manage_class_schedule`, which predates Phase 1; OQ-32 records it.
- **v0.6 (11/18/2026)**: Phase 1 is built and tested but not shipped. Mason is holding it to ship with later phases. What changed:
  - **1.1–1.5 are done, with tests.** The 1.6 gates passed on both platforms, and the mutation check caught every mutation. The release steps (push, pin bump, `@honuware/ui` release, `sandbox.14`, the production probe) wait for ship time.
  - **1.2's expectations, corrected to what's true:**
    - `manage_products` is *allowed* on `classes` (1.1's grant), not refused.
    - GET `delete_item` answers 405, not "not found".
    - The existing CRUD tests had to sign in: they all wrote signed out.
  - **New: *Testing Phase 1 locally*** under 1.6, a click-by-click walkthrough. It covers an admin's saves and deletes, Manage Data's delete, and a non-admin with `manage_products` on Class Schedules. It also lists what only the automated tests can check.
  - **OQ-32 grew a point:** on `manage_products` alone, Class Schedules saves the class but can't load its instances or set its tags, which need `manage_class_schedule`.
  - **10.2 gained *Carried from Phase 1*:** three items found during the inventory and census.
- **v0.5 (11/17/2026)**: OQ-31 is answered and confirmed: no hardcoded policy, and the three seeded policies as proposed. Still planning only. What changed:
  - **Policies are data, and the seed only writes the starting rows.** Which policy a product uses is already picked on its product page. 8.13's **Cancellation Policies** page edits the policies themselves.
  - **The one hardcoded rule goes.** Class bookings' no-refund rule (P-6) was code (`booking_helper.cpp:900-911`). It becomes a seeded **Class bookings** policy on the class and series products. The fixed sentences about classes, series and services on the booking and My Bookings pages are built from the rows instead (8.13).
  - **Two rules stay in code:** a cancellation by the studio, and leaving a waitlist, both refund in full.
  - **Still open:** OQ-7, the bios.
- **v0.4 (11/17/2026)**: OQ-22 is decided, and Ashley's feedback is folded in. This is still planning only. What changed:
  - **Fonts: option D** (OQ-22). The server fills in the page head, as 7.4 describes. 7.4 is no longer marked as a default, and option A's fallback lines are gone from 7.4, 7.7 and 11.1.
  - **Ashley's eleven points are mapped** in *Where each piece of feedback goes* (A1–A11), and nothing is dropped. Five of them add work:
    - **Cancelling one item of a cart refunds the whole cart** (A8, F23). Her one-hour massage credited her for both massages. Checking it turned up the same fault when the studio cancels a session (P15). New **2.9** refunds only that booking's item. It ships with Phase 2, since it's money.
    - **No cancellation policy exists, so every cancellation refunds in full, at any time** (P16, F24). New **8.13** seeds your policy, shows it on a page, and links it from the cancel warning together with the exact amount. It needs your policy (OQ-31).
    - **One attendance control with three states** (A1, A2, F25; new **6.6**): *going*, *can't make it* (with the note) and *not planned*. It looks the same on the home page, My Weekly Plan, Our Classes and the calendar. Today a skipped class looks unplanned on Our Classes, and the big **I can't make it** button reads as a status.
    - **An On my plan view** on Our Classes and the calendar (A3, 6.3).
    - **The class page lists its weekly times,** and its calendar link shows that class (A9, F26, 6.5). **Book with Caleb** from the Providers page (A11, 8.6).
  - **Her other points were already in the plan:**
    - an upgrade mid-month (Phase 3, prorated by OQ-2)
    - Browse Services going to Memberships (8.8, as Jennifer found)
    - picking a linked account instead of searching (4.6)
    - the slow calendar (7.5)
    - finding All Classes (8.4 adds a link and keeps your OQ-13 menu call)
  - **New open question:** OQ-31, the cancellation policy itself.
- **v0.3 (11/17/2026)** — Mason's second round of answers folded in. Still planning only. Eight questions are now resolved; two stay open (OQ-7 bios, OQ-22 fonts). What changed:
  - **Approved as written:** the FAQ and the Silver and Gold text (OQ-9), the subscribe page before sign-in (OQ-14), sizes inside each card (OQ-15), no products regrouping (OQ-19), CloudFront for public photos (OQ-21, 7.8) and 4 database connections (OQ-30).
  - **Manage Data: the Developer role, and no rollback** (OQ-20). You asked for my view: the role is the right call, and rollback isn't worth building now. RDS already keeps seven days of point-in-time backups, and you'll be the only person using the page. Phase 10 is no longer a default.
  - **Starting a membership is redesigned** (OQ-27, 2.3):
    - Members choose **Start today**, paying for the rest of this month by the day, or **Start on the 1st** of next month, with nothing charged until then.
    - Staff get a third choice: **rest of this month free**, with next month charged now.
    - The server works out the dates and amounts from the choice, and one preview endpoint shows the same numbers on the page, signed in or not.
  - **Found while checking OQ-27 (P13):** a member who subscribes "for this month" pays the **whole** month's price, even on the 28th. The page sends the full calendar month, so nothing is prorated (F4). 2.3 fixes it.
  - **Also found (P14): every monthly renewal drops the shared seats.** The billing run makes a new entitlement each month and seats only the payer, so whoever shares a couple's or family membership loses access on the 1st (F22). 2.2 now carries the seats over. That's urgent for Phase 4, since sharing is what it makes easy.
  - **OQ-22 has a new option, and it's now the default:** the server fills in the page's `<head>` from the database, so a font change in **Site Theme** reaches first paint without a deploy. That rewrites 7.4. No timer is needed: the server builds the head when asked and caches it, and a Site Theme save clears the cache.
  - **A correction to v0.2.** I wrote that 11.1's per-page titles and preview tags would fix link previews. They wouldn't on their own. The page is a static file from S3, and Angular adds those tags only once JavaScript runs, which link previews never do. They work once the server fills them into the served page, the same mechanism as the new 7.4. 11.1 and 11.2 now say so.
- **v0.2 (11/16/2026)** — Mason's answers folded in (see *Open questions*; his answers are kept as written). Still planning only. What changed:
  - **More than one membership is allowed,** behind a warning the buyer must tick that names what they already pay for (OQ-1, D2, 2.2). A second membership doesn't give the buyer its first seat, since it's usually for someone else.
  - **Production is reinstalled from the seed** at the final release, not migrated (OQ-6, D8, Phase 13). The testers register again. Every "production edit" step is gone, along with the migrations that only fixed data or wording. 2.8's clean-up of the testers' extra memberships is gone too (OQ-5).
  - **Upgrades are prorated** (OQ-2). **A downgrade to fewer seats asks who comes off** (OQ-3, 3.1, 3.3).
  - **New Studio Settings page** (2.8). It holds the "you just bought this" window (default 10 minutes, OQ-4) and the late sign-up window.
  - **A class stays on the sign-up lists until two hours after it ends,** so walk-ins and late arrivals can still be marked (OQ-26, 6.2). It sits under **Earlier today** so it isn't mistaken for an upcoming class.
  - **No Classes menu, and Gallery stays under About** (OQ-12, OQ-13). Instead, **Our Classes opens on Full Schedule for anyone without a membership,** with the existing lock and "Requires a membership" on members' classes (6.3).
  - **No email per weekly-plan tap, and every notification email links to its settings** (OQ-28, 6.1, 8.12). Calendar entries come from the personal calendar feed instead of one email per class.
  - **A visiting teacher needs an account;** the "Led by" column is dropped (OQ-18, 9.5).
  - **Your copy is in:**
    - the new-student FAQ, as a draft for your approval (5.3)
    - the Silver and Gold descriptions (5.2)
    - your "prepares you for" pairs (6.4)

    The favicon is the logo's figure in the red fabric (OQ-25, 8.2).
  - **Manage Data becomes a developer's tool** behind its own permission, by default. Its help text, foreign-key previews, table list and history are deferred (OQ-20, Phase 10).
  - **Public photos are cached at CloudFront's edge,** in answer to your image-cache question (OQ-21, new 7.8).
  - **The questions you asked me to explain** each have a fuller write-up and a recommendation: OQ-14, 15, 19, 20, 21, 22, 27 and 30. OQ-29 needed no decision and is closed.
  - **Checked while revising:** no member page writes through the generic data API (the shop's `addItem` calls are the cart service's), so Phase 1.2 breaks no member page. The generic writes come from about twenty Manage Products pages (1.1).
- **v0.1 (11/16/2026)** — First plan from the three testers' feedback. Planning only; no code changed.
  - Every piece of feedback is mapped to a plan item in *Where each piece of feedback goes*. Nothing is dropped. One item, a site search, is left as a question (OQ-24).
  - **Checking the code turned up five problems that nobody reported.** Two of them are urgent, and they come first:
    1. **Anyone, signed in or not, can add, change or delete products and classes through the generic data API** (F1). This is live on the sandbox site today. **Phase 1** closes it and ships on its own, before anything else.
    2. **A declined card leaves an active, unpaid membership behind** (F3). The monthly billing job will charge it later, and every retry adds another one. Together with the double charge (F2), this is the likely source of Thomas's "three memberships". **Phase 2**.
    3. Choosing a future start month makes that month free (F4). Phase 2.
    4. Changing a membership cancels the old one *before* charging for the new one, so a declined upgrade leaves the member with neither (F6). Phase 3.
    5. Several smaller ones, listed under the phase that fixes each: the Pricing Overview shows staff permissions as price columns (9.6), Who's Teaching What misses every recurring class (9.8), and an invitation is lost if the email's capital letters differ (4.1).
  - Thomas's feedback is ordered first, as asked. His "most critical issue" (paying twice, three memberships) is Phase 2, sharing a couple's membership is Phase 4, and the conversion funnel is Phases 3, 5 and 6.
  - **Releases:** Phase 1 goes out alone, as soon as it is green. Phase 2 goes next. The rest ship together after verification (Phase 13).
- **Migration numbers** follow build order: app `0013`–`0016`, honuware `0004`–`0005` (v0.2: content and data-fix migrations dropped, D8). If phases are built in a different order, they are renumbered then.
- **Who builds (standing agreement, Finishing website OQ-18).** The Overview's line "I will build the C++ server and run tests myself" is superseded. **Claude builds and runs the tests on both Windows and Linux, fixes what fails, and reports the result for each platform** ("Windows: N passed. Linux: M passed."). Claude says so before long or destructive runs. Git stays Mason's: Claude only reads history. Mason commits, pushes and tags, unless he asks Claude to tag.
- **Three repos.** Some items change the framework:
  - `server_components` (honuware C++), pinned by `GIT_TAG` in both knottyyoga and CommunityFinder
  - `honuware-web-components` (`@honuware/ui`), exact-pinned in `ui/package.json`

  Each such item ends with the usual cycle: gates → Mason pushes → CI → pin bump or npm release → the app's gates. Items are tagged *(server_components)*, *(@honuware/ui)* or *(knottyyoga)*.
- **The working tree isn't clean.** The iCal feed work is uncommitted: `personal_ical_feed_helper.*`, `get_my_ical_feed.cpp`, `create_database.cpp`, `web_app.cpp`, two CMakeLists and two tests. Phase 1's release needs Mason to commit it or set it aside first, so the tag carries only what's meant to ship.

---

# Where each piece of feedback goes

**T** = Thomas, **G** = Gifford, **J** = Jennifer, **A** = Ashley, **P** = found while planning.

| # | Feedback | Plan |
|---|---|---|
| T1 | Intro workshop button should land on intro workshops; say what to do when none are scheduled | 5.1 |
| T2 | Longer membership descriptions; Silver vs Gold; choose one, two or family | 5.2 |
| T3 | "Plan your week" with no membership shows "no eligible classes" and no way to a membership | 5.5 |
| T4 | My Account: no-membership call-out first; order by what people come to do | 5.4 |
| T5, T6 | "My Subscriptions" should be "My Membership"; extra click in its empty state | 3.3 |
| T7 | Back, then Purchase again, charged twice | 2.1, 2.5, 2.6 |
| T8 | Signed in, Memberships should show my level and a way to change it | 2.2, 2.7, 3.3 |
| T9 | Couple's: "1/2 seats" plus "Assign to me"; too many steps | 4.6 |
| T10 | Sharing email has no link | 4.3 |
| T11 | Bri couldn't find the sharing request | 4.1, 4.3, 4.5, 4.6 |
| T12 | "Sharing permissions" and "sharing membership" are separate | D4, 4.6 |
| T13 | No way to go from Silver to Gold | Phase 3 |
| T14 | Class page: beginner classes lead to the others; order by level | 6.4 |
| T15 | Tapping one class in My Weekly Plan reloads everything | 6.1 |
| T16 | Sign-up list shows classes that already happened today | 6.2 |
| T17 | Our Classes offers sign-up for classes already on my plan | 6.3 |
| T18 | Aerial Rope → schedule is slow | 7.5 |
| T19 | Default Angular favicon | 8.2 |
| T20 | Footer hidden behind Android's bottom bar | 8.1 |
| T21 | Get Started button lands past step 7 | 8.1 |
| T22 | Ended up with three memberships | 2.1, 2.2 |
| T23 | Spinner pushes "Processing" to a new line | 2.6 |
| T24 | `gold_member` / `silver_member` with underscores | 3.3, 5.4 |
| T25, T26 | Mason's and Gifford's bios undersell them | 8.10 |
| T27 | Favorite Instructors does nothing | 5.4 |
| T28 | Change Password asks for the old password twice | 8.3 |
| T29 | A business overview for the owner | 9.1 |
| T30 | Manage Data gives no context; schema help; foreign-key preview | 10.1: developer-only; help deferred (OQ-20) |
| T31 | No versioning or rollback | 10.1, 10.3 (OQ-20) |
| T32 | Login prompt on the first visit to Manage Products | 9.3 |
| T33 | The grid of tiles; group it under ~5 headings | 9.2 |
| T34 | Attendance plan search can't find people without a plan | 9.4 |
| T35 | No way to create a one-off workshop | 9.5 |
| T36 | Pricing Overview: say the table is by role, and Standard is the default | 9.6 |
| T37 | Utilization reports should look forward as well as back | 9.8 |
| T38 | Leftover "decision history" wording | 8.9 |
| T39 | Session tokens vs JWTs across replicas | F15, D6 (no change) |
| T40 | Back and forward behave inconsistently | 2.6, 8.1, 8.11 |
| T41 | A raw database API is a big attack surface; red-team it | Phase 1, 10.2 |
| T42 | Inline style in the index document | D7, 7.7 |
| T43 | Script references belong in the head | 7.3, 7.7 |
| T44 | Server-rendered signed-out home page | Phase 11 |
| T45 | `site_info` is late and slow | 7.2, 7.3 |
| T46 | Fonts start late | 7.4 |
| G1 | Let newcomers explore before committing or making an account | 5.2, 5.3 |
| G2, G8, G9 | Gallery to the top level; unbury Weekly Schedule and Calendar | No menu change (OQ-12, OQ-13); 6.3 |
| G3 | A site search | Not now (OQ-24) |
| G4 | "Our" used inconsistently | 8.4 |
| G5 | Blue links | 8.7 |
| G6 | "Show me how" under the headline | 8.5 |
| G7 | Getting Started: lighter, with a prerequisites / new-student FAQ | 5.3 |
| G10 | 3+ seconds to see the schedule | 7.5, 7.6 |
| G11 | List providers with services; massage discoverability | 8.6 |
| G12 | "Events", not "Upcoming Events" | 8.4 |
| G13 | Why is "Browse bookable services →" on Memberships? | 5.2 |
| G14 | Show the subscribe page before sign-in | 5.2 |
| G15 | Account tag: add status and dates | 5.4 |
| G16 | Favorite instructors' classes | 5.4 |
| G17 | "Search people to assign" has no affordance | 4.6 |
| G18 | Manage Data: left navigation instead of a dropdown | 10.3 |
| G19 | Categorize products | 9.2 (OQ-19) |
| J1 | My Bookings' "Browse services" goes to Memberships | 8.8 |
| J2 | Back from a Get Started link doesn't return to the step | 8.1 |
| A1 | Schedule list: the big **I can't make it** button reads as the class's status | 6.6 |
| A2 | A class marked "can't make it" shows as unplanned on Our Classes | 6.6 |
| A3 | The calendar's My Schedule should show only the classes I've planned | 6.3 |
| A4 | Upgrade a membership mid-month | Phase 3 |
| A5 | My Bookings' Browse Services goes to Memberships | 8.8 (as J1) |
| A6 | Pick from my linked accounts instead of searching | 4.6 (as G17) |
| A7 | Link the cancellation policy from the cancel warning | 8.13 (OQ-31) |
| A8 | Cancelling one massage of a two-massage cart credited the whole cart | 2.9 |
| A9 | The class page's schedule link hides a class I'm not in; put the weekly times on the page; the calendar is slow | 6.5, 7.5 |
| A10 | Hard to find the list of all classes | 8.4 |
| A11 | **Book with Caleb** from his card on Providers | 8.6 |
| P1 | Generic writes need no sign-in | Phase 1 |
| P2 | Declined card leaves an active membership | 2.1 |
| P3 | A future start month is free | 2.3 |
| P4 | Two emails for one subscription | 2.4 |
| P5 | Change of membership cancels before charging | 3.1 |
| P6 | Pricing Overview shows every permission | 9.6 |
| P7 | Attendance Templates tile shown to people who can't use it | 9.4 |
| P8 | Who's Teaching What misses recurring classes; Subscription Revenue mislabels counts as money | 9.8 |
| P9 | Invitations match email case-sensitively | 4.1 |
| P10 | Registration drops `returnUrl` | 4.5 |
| P11 | The browser sends one request at a time | 7.6 |
| P12 | The seat card's sharing link goes nowhere (`/my/account#sharing`) | 4.6 |
| P13 | Subscribing "for this month" charges the whole month, whatever the date | 2.3 |
| P14 | A monthly renewal drops shared seats and seats the payer again | 2.2 |
| P15 | The studio cancelling a session also refunds the booking's whole cart | 2.9 |
| P16 | No cancellation policy is seeded, so every cancellation refunds in full | 8.13 (OQ-31) |

---

# Current state (verified against the code, 11/16/2026)

Production runs `v1.0.0-sandbox.13` (10/30) on the Square sandbox. Commit `777e5363` (11/4) is on master but not released.

**Facts the plan depends on:**

1. **The generic write endpoints check only the table allow-list, not the caller.**
   - `add_item`, `add_item_fetch_primary_key`, `update_item` and `delete_item` all call `IsTableAllowed` (server_components `platform/endpoints/add_item.cpp:59` and the same line in the others).
   - That list starts from the app's base `allowed_tables`, which applies even when signed out. It includes **`products`** and **`classes`** (`create_database.cpp:353-376`).
   - `RequireTableWriteAccess` exists for exactly this case (`endpoint_auth_helper.cpp:284-313`, "it would let any logged-in user write to every public table"), but only the photo endpoints use it.
   - The CSRF guard compares a cookie with a header (`csrf_guard.cpp:115-122`). A script that isn't a browser sets both.
   - `delete_item` is a **GET** route, so the CSRF guard skips it altogether (`csrf_guard.cpp:57-66`). With the session cookie on `SameSite=Lax`, a link on another site that an admin clicks can delete rows as that admin.
   - The base list also exposes every member's attendance plan, notification preferences and iCal feed rows to anyone (the token hash is redacted).
2. **Back, then Purchase again, charges twice.**
   - The order lives only in the page component (`checkout.component.ts:29`, `subscription-signup.component.ts:130`). Coming back builds a new component with no order and a new `crypto.randomUUID()` key (`checkout.component.ts:253-257`, `subscription-signup.component.ts:125`).
   - Both server paths create a fresh row. The "already paid" guard (`payment_helper.cpp:135-149`) and the idempotency record (scoped to `purchase_pay_card/<purchaseId>`, `:152-157`) protect only the *same* purchase.
3. **A declined first charge leaves an active membership.**
   - `CreateSubscription` inserts the `active` row with the saved card and next billing date (`subscription_helper.cpp:141`) *before* charging (`:183`).
   - On a decline it returns (`:185-189`). The endpoint answers 400 but returns normally from its transaction (`endpoints/subscriptions.cpp:439-442`), which commits.
   - The billing run picks the row up later. Each retry adds one.
4. **Start months: a later one is free, and this one isn't prorated.** The member's **Subscribe** page offers this month and the next five.
   - **A later month is free.** No charge is made: the "no entitlement yet" path makes a $0, pre-credited entitlement for that month (`subscription_helper.cpp:289-314`), and the first bill is at its *end* (`:132-133`). The page promises a charge on the 1st (`subscription-signup.component.html:104`).
   - **This month costs the whole month, whatever the date (P13).** The endpoint sends the full calendar month (`endpoints/subscriptions.cpp:423-424`), so `CalculateProratedCharge` finds nothing to prorate (`subscription_helper.cpp:51-67`). The button reads **Subscribe & Pay $160** on the 28th too.
   - **The staff page already does more** (**Manage Products** ▸ **Subscriptions** ▸ **Create Subscription**). It works out the dates in the browser and sends them (`create-subscription.component.ts:271-309`):
     - this month, with **Prorate for remaining days** (on by default)
     - next month, with **Include this month free**: the rest of this month is free, and the first charge is on the 1st, not today
     - a later month, as on the member page
   - The billing run always charges the period that starts where the current one ends (`subscription_helper.cpp:597`), at `next_billing_us`. A membership can therefore wait for its start with an empty current period and nothing granted.
5. **Nothing checks for an existing membership.** There's no guard in the server or the page. `Subscriptions::GetActiveByPersonId` exists, but only its own test calls it. The Memberships page and tier cards don't know who is signed in (`membership-tier-cards.component.ts:13-21`).
6. **Changing a membership works on the server, but members can't reach it.**
   - `POST /api/subscriptions/<id>/change_product` exists (`endpoints/subscriptions.cpp:657-744`), but only the admin page calls it.
   - An upgrade charges the full price difference and **cancels the old subscription and revokes its entitlement before charging** (`subscription_helper.cpp:1051-1085` vs `:1122`). A decline leaves neither.
   - A downgrade takes effect at period end and transfers no seats.
7. **Signing up for a membership sends two emails:** the payment confirmation (`payment_helper.cpp:285`) and "Welcome to …" (`subscription_helper.cpp:337`).
8. **Sharing is two mechanisms.**
   - A **gift permission** is the recipient's consent to the payer (`gift_permissions`, pending/accepted/denied).
   - A **seat assignment** uses a seat of an entitlement (`entitlement_assignments`).
   - A seat can only go to someone who has already accepted a gift permission (`entitlement_helper.cpp:254-259`).
   - The two have separate pages: **Purchase Sharing** and the seat card.
   - Invitations to emails without an account are stored and turned into requests at registration (`gift_permission_helper.cpp:332-358`). The match is a plain string compare (`gift_permission_invitations.cpp:39-44`), so different capitalisation loses the invitation.
9. **Both sharing emails have no link,** though the site address is already passed in (`branding.websiteUrl`, from `website_address_login`). The invitation's copy talks about "massages, spa visits". Assigning a seat sends no email at all.
10. **"Assign to myself" is a page bug.** The server gives the buyer a seat (`subscription_helper.cpp:316-326`). But the post-purchase card doesn't pass `[assignments]` to the seat component (`subscription-signup.component.html:50-54`), so it thinks the buyer has none.
    - The card's "sharing" link goes to `/my/account#sharing`, an anchor that doesn't exist.
    - The seat search filters only people who have already accepted, needs a pick from the dropdown, and says nothing when there's no match.
11. **Registration drops `returnUrl`.** Login honours it; the register link, register, check-email and verify pages don't carry it (`@honuware/ui` `login.component.html:41`, `register.component.ts:87-89`, `verify.component.ts:70`).
12. **The app scrolls an inner `div`, not the window** (`app.component.html:1` `h-screen` and `:14` `overflow-y-scroll`).
    - Angular's scroll restoration (`app.config.ts:62`) acts on the window, so every page keeps the previous page's scroll position. Back doesn't restore it, and Get Started (steps load after a spinner) lands at the bottom.
    - `h-screen` is `100vh`, which on Android Chrome is taller than the visible area, so the footer and the checkout bar sit under the browser's bottom bar.
13. **The browser sends one request at a time.** `ServerAccessProxy.serialize()` (`ServerAccess.ts:229-273`) queues every call, because "the C++ backend … rejects concurrent requests". The server runs one database connection per site (`main.cpp`'s `FixedTenantResolver`, `maxConnections` 1).
    - Start-up is three calls in a row (`/api/me`, `/api/site_info`, then `/api/remember` or `/api/get_user_info`), and the page renders only after all of them.
    - `/api/site_info` makes about 105 separate `config_secrets` reads, with no cache, and sends `Cache-Control: no-cache` (server_components `platform/endpoints/site_info.cpp`).
14. **The schedule pages do their N+1 on the server.**
    - `/api/calendar` (90 days) and `/api/me/upcoming_classes` (90 days) each derive every occurrence and, per occurrence, look up the instructors, facility, room, predecessor (about 6 queries) and advance-days again (`calendar_helper.cpp:329-346`, `attendance_template_helper.cpp:654-813`).
    - The class page derives 120 days of sessions for a recurring class whose list the page hides (`class_catalog_helper.cpp:25,180`).
15. **Sessions live in Postgres** (`sessions`, `device_tokens`), looked up on every request. Nothing about a login is held in memory, so a replica failing over loses none (T39). Every `/api/remember` uses up the device token and issues a new one, so two tabs racing can sign one out (9.3).
16. **No business overview exists.** The reports look back only (enrollment trend, open-seat heatmap: past 12 weeks of materialized sessions), or forward only over materialized rows (Who's Teaching What, which so misses recurring classes). Subscription Revenue sums all time and puts counts in money-named fields.
17. **There's no change history** for anything edited through Manage Data. `admin_table_friendly_names.description` exists and reaches the browser, but nothing shows it.
18. **Production deploys migrate; they don't reseed.** Seed changes reach only new installs, which is why the final release reinstalls (D8).
19. **Our Classes starts every signed-in person on My Schedule** (`our-classes.component.ts:113`). That view hides every class the viewer can't take, so someone with an account but no membership sees "Your current filters hide everything." The schedule's compact chip already shows a lock and "Requires a membership" on a members' class (`calendar-event.component.ts:46-49`).
20. **CloudFront caches nothing under `/api/*`** (`Managed-CachingDisabled`, Deploying to AWS), and that includes photos. A public scaled photo is sent as `public, no-cache` with an ETag (`get_scaled_photo.cpp:172-241`), so even a repeat view is a round trip to the server. A first view is a database read, since the scaled bytes are cached there.
21. **There's no general settings page.** Site Theme edits its own `site_*` values. `/api/admin/secrets` exists (`endpoints/admin_secrets.cpp`), but no page in the app or `@honuware/ui` calls it. Other values change only through `knottyyoga_test_helper --command=set_secret`. A setting a studio owner should change needs its own page (2.8).
22. **A monthly renewal drops every seat except the payer's (P14; found in v0.3 while checking OQ-27).**
    - Each month the billing run charges through `PayWithCard` (`subscription_helper.cpp:640-650`; the past-due retry is the same at `:847-856`). That makes a new entitlement and seats only the payer (`payment_helper.cpp:267-271`).
    - Nothing copies last month's seats across. `EntitlementHelper::TransferAssignments` does exactly that, but only a change of membership calls it (`subscription_helper.cpp:1136`).
    - So on the 1st, Bri's seat on a couple's membership is gone, and the seat card reads "1/2" again. A membership bought for a friend seats its payer again, too.
    - No test renews a shared membership. `GetCurrentEntitlementForSubscriptionAfterBilling` (`subscription_helper_test.cpp:599`) only checks that the entitlement is new.
23. **Cancelling one booking from a cart refunds the whole cart (A8, P15; found in v0.4).**
    - A cart checkout makes one purchase and one payment. Each booking keeps its own line, `purchase_item_id` (`cart_checkout_helper.cpp:327-352`).
    - Every cancel refunds a percentage of the **whole payment**: `RefundHelper::ProcessRefund` takes the payment's amount × the percent (`refund_helper.cpp:123-126`). The callers are:
      - a member's cancel (`booking_helper.cpp:819`, `:872`, `:930`, `:985`)
      - the studio cancelling a session (`session_cancellation_helper.cpp:142`, `:154`)
    - The purchase is then marked `refunded` while the other booking stays confirmed (`refund_helper.cpp:331-332`). So Ashley's $160 cancellation credited her for the whole cart.
    - Cancelling the second booking refunds nothing, because the newest payment row is now the negative refund (`:107-115`).
    - A refund by amount already exists for series occurrences, `ProcessPartialRefundCents` (`:137-202`). It is capped at what the purchase still has paid.
24. **No cancellation policy exists, so every cancellation refunds in full (P16; found in v0.4).**
    - `CalculateRefundPercent` returns 100% when a product has no policy (`refund_helper.cpp:56-58`). The seed creates no `cancellation_policies` rows, only their admin metadata.
    - For a service, My Bookings' warning says only "Cancellation policy applies. Refund will be calculated based on the cancellation policy." (`my-events.component.ts:235`).
    - No page shows a policy. Manage Products can attach one to a product (`product-detail.component.html:105`) but has no page for editing policies.
25. **The attendance controls disagree (A1–A3).**
    - The home page's class list and My Weekly Plan's **Upcoming** tab are one component (`upcoming-classes.component.html:83-93`). **Today's Classes** works the same way.
      - The status is a small check mark by the class name.
      - The big button beside it is the *action*: **I can't make it** on a class you're going to, **I'll be there** on one you're not. At a glance the button reads as the status.
    - Our Classes and the calendar use a check box with two states, **I'll be there** or **Tap to plan attendance** (`calendar-event.component.html:32-34`, `:100-102`).
      - A skipped class, with its note, shows as the second, the same as a class never planned.
      - The skip is already in the data those pages load (`attendance-map.ts:19-41`).
    - The calendar has the same **My Schedule** / **Full Schedule** toggle as Our Classes (`calendar-home.component.html:9-10`). It too filters by access (F19), so a member who plans two days a week still sees every class they can take.
26. **The class page doesn't say when the class runs (A9, A10).**
    - A recurring class's **When it runs** card says it "runs on a weekly schedule" and links to `/calendar` with no filter (`class-detail.component.html:268-285`).
      - The calendar opens on the eligible-only view, so a class the viewer can't take isn't there.
      - The class detail response has no weekly times.
    - Signed in, All Classes isn't in the menu. **My Classes** opens the weekly page, which links to All Classes through **Browse all classes** (`mockHeaderResponse.ts:64-79`, `our-classes.component.html:5-10`).

---

# Design decisions

- **D1 — The server enforces "once"; the page only helps.**
  - The server refuses: a second membership nobody confirmed, a paid order paid again, an unpaid membership left active. Each has its own error code that the page turns into a sentence.
  - The page avoids getting there: the order in the URL, history replaced on success, the button disabled while processing. A bug or a second tab in the page can't charge twice.
- **D2 — A second membership takes an explicit yes** (OQ-1).
  - People do pay for a friend's membership, so a person may pay for more than one. What the server refuses is a second one *by accident*: anyone who already pays for a current membership, or holds a seat on one, must send `confirm_additional_membership`. The page sends it only after the buyer ticks a warning.
  - A membership is a product of kind `subscription`. "Current" means active or past due, **or** cancelled but running until its period ends, **or** a scheduled change that starts next period. So Back, Reactivate and Change all hit the warning.
  - The warning names what you already have: "You already pay for **Gold** (for you) and **Silver** (for Bri). This adds another membership, charged separately every month. To change a membership instead, go to My Membership."
  - A second membership is usually for someone else. So if the buyer already holds a seat on a current membership, the new one doesn't seat them: its seat card starts with "Who is this membership for?" (4.6).
- **D3 — Every checkout carries its order in the URL, and success replaces history.**
  - After the first Pay click the URL gains the purchase id (`?purchase=123`, `replaceUrl`). Coming back to it shows that order: its receipt if paid, the same order if not.
  - Success navigates with `replaceUrl`, so Back lands on the page *before* checkout, never on the form. This is Thomas's "a transaction id assigned the first time".
- **D4 — Sharing is one action: invite a person to a seat.**
  - The gift permission stays as the consent record underneath. Accepting an invitation that names a seat grants the permission **and** takes the seat, in one step.
  - The member sees one **Sharing** page (people, their status, which seat each holds) and one "Add a person" field. Nobody needs to know there are two tables (T12).
- **D5 — Every link in an email works for someone who isn't signed in and may have no account.**
  - It goes to the page that does the job, and sign-in or registration returns there.
  - As a backstop, a signed-in person with a request waiting sees it as a banner on the home page and My Account. That covers losing the link.
- **D6 — Sessions stay server-side tokens in Postgres; no JWTs** (T39). They are already replica-safe (F15). A JWT would add a revocation problem: a stolen token lives until it expires, and sign-out, password reset and "end every session" (D13 of Finishing website) depend on server-side revocation.
- **D7 — Critical CSS stays inlined** (T42).
  - The inline `<style>` in the served page is Angular's *critical CSS*: the rules the first screen needs, put in the page so the first paint doesn't wait for a stylesheet. Moving it to a stylesheet would make first paint slower, not faster.
  - What *does* go is the Google Fonts CSS that the build also inlines, which Phase 7.4 replaces (OQ-22).
- **D8 — Content goes into the seed; production is reinstalled from it** (OQ-6).
  - The final release (Phase 13) reinstalls production from the seed, as Finishing website D11 did: `--install_schema --force`, then the `set_secret` block. The testers' accounts and history go with it, and they register again.
  - So content changes go **only** into the seed, each with a seed test through its reader: Getting Started, bios, descriptions, home rows, admin hints. No hand edits, and no migrations that only fix data or wording.
  - Schema changes still get an idempotent migration (the project's rule). Mason's local databases need them, and so do the two interim releases: sandbox.14 and .15 deploy with `--migrate`.
- **D9 — Framework fixes go in honuware; studio-specific screens stay in this repo.** The generic API, auth, registration, the CRUD editor and `site_info` are honuware. Memberships, sharing, scheduling, reports and the workshop wizard are knottyyoga.
- **D10 — Each fix is tested through the reader that production uses** ([[feedback_assert_through_the_reader]]):
  - "no unpaid membership" through the billing run's due-row reader and `GetCurrentMembership`
  - "seat taken on accept" through the entitlement's assignment reader
  - "writes refused" by reading the row back unchanged
  - "fewer queries" through a query counter (7.1)

---

# Phase 1 — Close the open data API *(found while planning; ships alone, first)*

F1: anyone can write to `products` and `classes` through the generic API, and a GET can delete as an admin. This is outside the testers' feedback, but it is the most urgent item in the plan, and the start of Thomas's red-team request (T41).

### 1.1 Inventory *(read-only, recorded here)*
- [x] Every caller of the generic API in `ui/` and `@honuware/ui`:
  - the writes `addItem`, `addItemFetchPrimaryKey`, `updateItem`, `deleteItem`
  - the reads `getTableRows`, `getRow`, `getFilteredTableRows`, `getDbSchema`

  For each: the table, and who reaches it (signed out, member, staff, `manage_products`, `manage_class_schedule`, admin).
- [x] Any **member** page that writes through the generic API gets a dedicated endpoint first (in 1.4), so 1.2 doesn't break it.
  - **Checked 11/16:** none do. The shop pages' `addItem` calls go to the cart service, not the generic API.
  - The generic writes come from about twenty Manage Products pages and two shared editors (`event-session-card`, `event-session-price-override-editor`). Each of their tables needs a grant to the permission that opens the page; the inventory lists any that are missing.
- [x] The same inventory for CommunityFinder's UI, which takes the same pin.

**Result (11/18):**
- **No signed-out or member page writes through the generic API,** in either app. Every write comes from a staff or admin page. All of CommunityFinder's are behind its admin guard.
- **One page 1.2 would have broken, now fixed.** Manage Products ▸ Class Schedules is opened by `manage_products`. It saves `classes`, `class_requirement_groups` and `class_requirement_group_literals`. Only `manage_class_schedule` had grants on those three. Their writes passed because the tables are on the public list, which 1.2 stops counting for writes.
  - `PopulateAdminTablePermissions` now grants all three to `manage_products`. That's a seed change only, with no migration: no seeded non-admin holds `manage_products`, so nobody's access changes on an existing database.
  - Test: `CreateDatabaseTest.ManageProductsCanEditAClassAndItsRequirements`.
- **Already broken for a non-admin before this phase, and not changed by it.** A non-admin holder of the opening permission is refused on the tables below today, because nothing grants them. Admins are unaffected, and today only admins use these pages. OQ-32 asks when to fix them.
  - `product_booking_windows`, which is nested-only and has no grant
  - `instructors`
  - `skill_levels`, for a `manage_products` holder without `manage_skills`
  - `provider_type_assignments`, `provider_availability`, `schedule_templates`
  - `home_sections`, `getting_started_steps`, `about_sections`, `announcements`, `image_carousels`, `image_carousel_photos`
  - the Staff ▸ Sessions card (`event_session_staffing`, `bookings`, `event_session_price_overrides`), for someone who holds only Instructor
- **Two side notes:**
  - The server doesn't expand `permission_implications` when checking a grant. Only the permission itself counts.
  - The `manage_products` grant on `permissions` lets its holder edit any permission row, including `admin_portal`. Admin-only would be safer. Carried to the red-team pass (10.2).
- **knottyyoga's UI doesn't use the library's `deleteItem`.** `app.config.ts:73` points the library's CRUD access at the app's own `ServerAccessNetwork`, so the app-side change in 1.5 is what knottyyoga runs. The `@honuware/ui` release matters for CommunityFinder, whose admin pages use the library's `CrudHttpAccess`.

### 1.2 Writes need a grant *(server_components: platform/endpoints)*
- [x] `add_item`, `add_item_fetch_primary_key`, `update_item`, `delete_item` call `RequireTableWriteAccess` instead of `IsTableAllowed`:
  - 401 signed out
  - 403 without a grant
  - admins, and the tables a person's permissions grant, as today
- [x] `delete_item` becomes **POST**, and the GET route goes, so the CSRF guard covers it.
- [x] `get_db_schema`: `display_templates` are filtered to the caller's allowed tables, as the column data already is.
- [x] Tests, all through `handle_full`, reading the row back afterwards to prove nothing changed:
  - signed out, each of the four writes on `products` → 401, and the row unchanged
  - a member with no grants → 403
  - `manage_products` → allowed on `products`, 403 on `classes`
  - admin → allowed
  - GET `delete_item` → not found
  - the existing CRUD tests pass unchanged

**As built (11/18):**
- **The order of the checks.** The missing-field checks come first (400), as before. Then `RequireTableWriteAccess` (401 or 403). Then `IsTableAllowed` still applies, so an admin can't write to a table that's on no list at all (400 "not an allowed table"). Reading a table never gives the right to change it.
- **The framework tests use honuware's own tables.** Each of the four write tests now has `SignedOutIsRefused` (401), `WithoutGrantIsRefused` (403) and `WithGrantIsAllowed`, and `add_item` and `delete_item` also have `GrantIsPerTable`. They run on `people` with a test permission that grants it, and every refusal reads the table back to show nothing changed. `products` and `manage_products` are app tables, so the app's half is `CreateDatabaseTest.ManageProductsCanEditAClassAndItsRequirements`, run against the real seed. One line of the list changed with 1.1: `manage_products` is now **allowed** on `classes`, because the Class Schedules page it opens saves classes.
- **GET `delete_item` answers 405**, Crow's "method not allowed", not 404. `DeleteItemTest.GetIsNotAllowed` checks that, and that the row is still there.
- **The existing CRUD tests did not pass unchanged.** Every one of them wrote signed out, which is exactly what 1.2 forbids. They now sign in as an admin, and their row comparisons leave out the caller's own `people` row. CommunityFinder's `crud_test.cpp` and `roles_permissions_test.cpp` needed the same changes: their delete uses POST, and the non-admin self-assignment now expects 403 instead of 400. A new CommunityFinder test, `AnonymousCannotChangeAPublicTable`, shows a signed-out delete and update on its public `categories` are refused and change nothing.
- **`display_templates`:** `DbSchemaTest.GetDbSchemaDisplayTemplatesOnlyForAllowedTables`, signed out with a template for `people` (allowed) and `roles` (not). `DatabaseRestHelperTest.DatabaseMetadataWithDisplayTemplates` now expects the same filtering.

### 1.3 No GET that writes *(server_components + knottyyoga)*
- [x] A census of every `CROW_ROUTE` registered for GET whose handler writes. Each becomes POST, with a test that the GET is gone. `delete_item` is the known one; the census says whether there are others.
- [x] Record the census here.

**Census (11/18):** 17 framework GET routes and 109 app GET routes (out of 245 live app routes). Every live app route names its method. Six framework routes had no `.methods()`, which Crow treats as GET. `delete_item` was one of them and is now POST. The other five are reads.
- **Writes something the user would call a change:**
  - `delete_item`: now POST. Test: `DeleteItemTest.GetIsNotAllowed` (405, row still there).
  - `/api/account_activation/<email>/<token>` marks the email verified and deletes the verification row. **It stays GET.** It's the link in the activation email, so it has to work when clicked. It reads no session, and only the secret token from the email works, so a cross-site link gains an attacker nothing. Anyone with the token can already call it directly.
- **Bookkeeping writes, left as GET:**
  - `/api/get_scaled_photo/…` stores a resized copy on a cache miss and touches its last-used time on a hit. Not a CSRF risk. A signed-out caller can, though, ask for any size up to the cap on public tables, and each new size stores another copy. Carried to 10.2.
  - `GET /api/me/notification_preferences` creates the default row on first read.
  - `/api/me/ical_feed.ics` updates the feed token's last-used time. It authenticates by the token in the URL, not the cookie.
- **Found on the way:**
  - When an activation token is wrong, the attempt counter and the admin alert are both written and then rolled back: the endpoint throws, and the transaction commits only on a normal return. So the attempt limit probably never builds up through this route. Carried to 10.2, with a test first.
  - `src/main.cpp.bak` carries ten old method-less routes (`/addclass`, `/removeclassbyid`, …). It's in no CMakeLists and isn't built. Deleting it is safe.

### 1.4 Private tables off the public list *(knottyyoga: database_helper + migration)*
- [x] `PopulateAllowedTables` stops listing these tables:
  - `attendance_templates`, `attendance_template_entries`, `attendance_template_exceptions`
  - `user_notification_preferences`
  - `ical_feed_tokens`

  Admins still reach them through the admin top-level and nested lists, and `manage_class_schedule` through its table grant. To verify against `PopulateAdminTablePermissions`.
- [x] Migration **`app/0013_private_tables_not_public`**: deletes those five `allowed_tables` rows. It is idempotent, and a second run changes nothing.
- [x] Tests, through `IsTableAllowed` (the reader every generic endpoint uses):
  - signed out, refused on all five
  - admin, allowed
  - `manage_class_schedule`, allowed on the template tables
  - the migration clears the rows on a database that has them, and running it twice is harmless
- [x] Any member write that 1.1 found moves to a dedicated endpoint here, with tests. *(1.1 found none.)*

**As built (11/18):**
- **Verified against `PopulateAdminTablePermissions`:** `manage_class_schedule` grants the three attendance-template tables. Nothing grants `user_notification_preferences` or `ical_feed_tokens`, so only admins reach those two now. No UI reads any of the five through the generic API (1.1), so no page changes.
- **The five names are in one place,** `Migration::kPrivateTablesRemovedFromPublicList` in `app_migrations.h`, which the migration and the tests share.
- **`PopulateAllowedTables` and `PopulateAdminTablePermissions` are now declared in `create_database.h`,** like `PopulateAdminMetadata`, so a test can seed them.
- **The tests ask the real endpoint** (`/api/get_table_rows`, which calls `IsTableAllowed`) over the real seed:
  - `CreateDatabaseTest.SignedOutVisitorsCannotReadMembersPrivateTables`: all five refused, and `classes` still readable as the control
  - `AdminsStillReadMembersPrivateTables`
  - `StudioManagersStillReadAttendanceTemplatesOnly`: the three templates allowed, and the other two refused
  - `AppMigrationsTest.PrivateTablesNotPublicTakesThemOffTheListAndIsIdempotent`, through `AllowedTables::IsTableAllowed`, with `products` surviving both runs

### 1.5 The browser side *(@honuware/ui: access; knottyyoga: ServerAccessNetwork)*
- [x] `deleteItem` sends POST in `crud-http-access.ts` and in any app-side copy. A spec pins the verb and path.
  - `@honuware/ui`: `crud-http-access.spec.ts` now says "deleteItem POSTs the delete_item path, never GET".
  - knottyyoga: `ServerAccessNetwork.ts`, pinned by the new `server-access-network-crud.spec.ts`. This is the copy knottyyoga's pages actually use (1.1).
- [x] `@honuware/ui` patch release; `npm install @honuware/ui@X.Y.Z --save-exact` in `ui/`. *(Mason, at ship time. knottyyoga works without it; CommunityFinder's admin delete needs it.)* ✅ 2026-11-19

### 1.6 Ship it
- [x] Gates: server_components (Windows + Linux); knottyyoga against the local checkout (Linux); Angular (Windows + Linux).
- [x] Mutation check: put `IsTableAllowed` back on each write, and GET back on delete. The new tests must catch every one.
- [x] **Mason** pushes server_components; CI; Claude bumps the pin in knottyyoga and CommunityFinder and reruns both pinned gates.
- [x] `@honuware/ui` release (Mason tags), install, Angular gates.
- [ ] Release as **`v1.0.0-sandbox.14`**: server and UI together. The new server refuses the old UI's GET delete, so they must ship as a pair.
- [ ] After deploy, Claude probes production signed out (`curl -m 10`, read-only and write attempts on a throwaway `products` row name). Every write must answer 401. Results recorded here.

*The last two wait for ship time. Mason is holding Phase 1 to ship with later phases (11/18).*

**Pin bump and library release (11/18):**
- **The pin.** honuware pushed as `838045f`. The commit message reads "Bucket CM3 — Event photos & video", but its diff against the old pin `f445a49` is exactly the 15 Phase 1 files. Both apps' `CMakeLists.txt` `GIT_TAG` now point at it.
- **The library release.** `@honuware/ui` **0.3.1** is published. Its first publish failed with npm's `E404` on the upload, which means an expired `NPM_TOKEN`. Mason renewed the token and reran the job.
- **Installed:** `npm install @honuware/ui@0.3.1 --save-exact` in knottyyoga `ui/` (from 0.3.0) and CommunityFinder `ui/` (from **0.1.2**).
- **CommunityFinder's UI went from 0.1.2 to 0.3.1 with no code changes.** 0.2 and 0.3 add the check-your-email, forgot-password and reset-password pages, but only behind *optional* `AuthRoutes` paths. CommunityFinder's `AUTH_ROUTES` sets none of them, so registration still goes to the login page and there is no **Forgot password?** link. Turning those pages on for CommunityFinder is separate work: routes, its paths, and its mail.
- **The knottyyoga Windows tree is back on the pinned honuware** (`FETCHCONTENT_SOURCE_DIR_HONUWARE` empty, `_deps\honuware-src` at `838045f`).

| Pinned gate | Windows | Linux |
|---|---|---|
| knottyyoga | 5475 | 5475; both programs built |
| CommunityFinder | 2523 | 2523; both programs built |
| knottyyoga `ui/` (0.3.1) | 4050 | 4050 |
| CommunityFinder `ui/` (0.3.1) | 487; `ng build` OK | 487 |

CommunityFinder's `ng build` prints a bundle-size *warning*: 918 kB initial against a 500 kB budget. It isn't an error, and I haven't checked whether it predates 0.3.1.

**Gate results (11/18, all against the local, unpushed honuware):**

| Suite | Windows | Linux |
|---|---|---|
| honuware (`server_components`) | 1949 | 1949 |
| knottyyoga | 5475 | 5475 |
| CommunityFinder | not run: its Windows tree is pinned | 2523; both programs built |
| knottyyoga `ui/` (Angular) | 4050 | 4050 |
| `@honuware/ui` | 514 | 514 |

- The knottyyoga Linux run also built the database helper and the server.
- **One compile fix along the way.** `create_database_test.cpp` used `DbSchema::kPermissionManageClassSchedule` without including `db_schema/permissions.h`, and failed on both platforms. The include is added.
- **One unrelated failure on Windows, cleared on rerun.** The first Windows honuware run failed four `TenantPhysicalIsolationTest` tests: "database `test_honuware_tenant_b_windows` is being accessed by other users". Nothing held that database when I checked afterwards, and a full rerun passed all 1949. Phase 1 doesn't touch those tests.

**Mutation results (11/18).** Each mutation undid one change and rebuilt, and at least one new test failed every time.
- honuware:
  - The write check turned off in `add_item` → `AddItemTest.AddItemSignedOutIsRefused`, `…WithoutGrantIsRefused` and `…GrantIsPerTable` fail.
  - The same in `add_item_fetch_primary_key` → `…SignedOutIsRefused` and `…WithoutGrantIsRefused`.
  - The same in `update_item` → `…SignedOutIsRefused` and `…WithoutGrantIsRefused`.
  - The same in `delete_item` → `DeleteItemTest.SignedOutIsRefused`, `WithoutGrantIsRefused` and `GrantIsPerTable`.
  - `delete_item` back on GET → `DeleteItemTest.GetIsNotAllowed`, and every POST delete test.
  - `display_templates` unfiltered → `DbSchemaTest.GetDbSchemaDisplayTemplatesOnlyForAllowedTables` and `DatabaseRestHelperTest.DatabaseMetadataWithDisplayTemplates`.
  - My script restored files with their old timestamps, so later mutations still built against earlier mutants. Each mutation's own tests were measured correctly, because sed always gives the mutated file a new timestamp. To clear the mutants, I touched the five files afterwards and reran the full gate: 1949 passed.
- knottyyoga:
  - The five private tables back on the public list → `CreateDatabaseTest.SignedOutVisitorsCannotReadMembersPrivateTables` and `StudioManagersStillReadAttendanceTemplatesOnly` fail.
  - Each of the three `manage_products` grants removed, one at a time → `ManageProductsCanEditAClassAndItsRequirements`.
  - The Studio Manager's grant on `attendance_templates` removed → `StudioManagersStillReadAttendanceTemplatesOnly`.
  - `app/0013` made a no-op → `AppMigrationsTest.PrivateTablesNotPublicTakesThemOffTheListAndIsIdempotent`.
  - After the restore, the same 13 tests pass, and the full gates above ran afterwards.

**What to commit, by repository:**
- `server_components`: the four write endpoints, `database_rest_helper.cpp`, and their tests (`add_item`, `add_item_fetch_primary_key`, `update_item`, `delete_item`, `db_schema` and `database_rest_helper` tests). *Committed and pushed, 11/18: `838045f`.*
- `honuware-web-components`: `crud-http-access.ts` and its spec. *Committed and tagged, 11/18: `v0.3.1`, published.*
- knottyyoga:
  - `app_migrations.h`, `app_migrations.cpp` and `app_migrations_test.cpp`
  - `create_database.h`, `create_database.cpp` and `create_database_test.cpp`
  - `ui/…/ServerAccessNetwork.ts` and the new `server-access-network-crud.spec.ts`
  - added with the pin bump (11/18): `server/knottyyoga_server/CMakeLists.txt` (the `GIT_TAG`), and `ui/package.json` with `ui/package-lock.json` (0.3.1)
- CommunityFinder: `crud_test.cpp` and `roles_permissions_test.cpp`.
  - added with the pin bump (11/18): `server/communityfinder_server/CMakeLists.txt` (the `GIT_TAG`), and `ui/package.json` with `ui/package-lock.json` (0.3.1)
  - That working tree also has other uncommitted work that isn't Phase 1: the "CM3 OQ10" photo-cap override, in `app_secret_values.*`, `create_app_tables.*`, `create_database.cpp`, `register_app_services_test.cpp` and `test/src/main.cpp`.
  - The 2523 above ran with it in place.

**Ship order when the time comes:**
1. Push `server_components`.
2. Bump the pin in both apps, then rerun both pinned gates.
3. Release `@honuware/ui`; CommunityFinder needs it.
4. Ship knottyyoga's server and UI in one release. An old UI's delete is refused by the new server.

#### Testing Phase 1 locally, before it ships
**Setting up**
1. **Server build.** Since the 11/18 pin bump, the pinned honuware (`838045f`) includes Phase 1. `out\build\x64-Debug` is back on that pin, so a normal **x64-Debug** build has everything. CommunityFinder's Windows tree is on the same pin.
   - If you go back to building a local `server_components` checkout (**x64-Debug (local honuware checkout)**), the local path stays in the CMake cache of that shared folder. To return to the pin, run `cmake --preset x64-Debug -DFETCHCONTENT_SOURCE_DIR_HONUWARE=` from `server\knottyyoga_server`, or pick **x64-Debug** in Visual Studio and use **Delete Cache and Reconfigure**. Switching the dropdown alone isn't enough.
2. **Blank database.** Run **knottyyoga_database_helper.exe**. Its launch settings already pass `--recreate_database`.
   - `--migrate` on your current database would apply `app/0013`, but it wouldn't add 1.1's new `manage_products` grants. They're seed-only (1.1), so Part C needs the recreated database.
3. **Start the site.** Start **knottyyoga_the_server.exe**, then run `ng serve` in `ui\`. The site needs no `@honuware/ui` install: its pages use the app's own `ServerAccessNetwork`, which already sends the delete as POST.

**Part A: an admin's saves and deletes still work.** This also proves the server and UI pair up: with the old UI, every delete in this part fails.
1. Click **Sign In** at the top of the page. On **Log In**, enter Email `masonbendixen@gmail.com` and Password `pass`, then click **Sign in**.
2. Go to **Admin ▸ Manage Products** and open the **Class Schedules** card.
3. Click **Add class**. In the **Add class** dialog, enter Name `Phase 1 test class` and Default capacity `10`, choose Kind **Recurring**, and click **Save**.
   - Expect: `Phase 1 test class` is selected in **Class**, with no red banner. The details card under it shows `Phase 1 test class` as its heading, beside the photo.
4. Click **Edit class**. Change Name to `Phase 1 test class edited`, and click **Save**.
   - Expect: **Class** and the details card's heading both show the new name.
   - The details card had no name until 11/19. Mason noticed while running this walkthrough. The card was then laid out as he asked: see *Class Schedules: the class card* below Part C.
5. In the **Requirements** card, type `Members` in **New group label** and click **Add group**.
   - Expect: a `Members` group appears.
6. In that group's **Add a permission**, type `platinum` and pick **platinum_fitness — Knotty Yoga Platinum Fitness Membership.**
   - Expect: a `platinum_fitness` chip.
7. Click the chip's **✕**.
   - Expect: the chip is gone.
8. Click the group's **Delete group** icon.
   - Expect: the card says **Open to everyone — no requirements.**
9. Click **Delete class**.
   - Expect: `Deleted class "Phase 1 test class edited".`, and the class is gone from **Class**.
   - With the old UI, this fails with **Could not delete class. Remove its class instances first.**
10. Go to **Admin ▸ Manage Data**. In **Select Table**, choose **Roles**. Click **New Item**, enter Name `Products Only` and Description `Phase 1 test`, and click **Create**.
11. In **Select Table**, choose **Role Permissions**. Click **New Item**, pick Role **Products Only** and Permission **manage_products**, and click **Create**. Keep this row: Part C uses it.

**Part B: Manage Data's delete.**
1. Still in **Manage Data**, choose **Roles** again and click **New Item**. Enter Name `Throwaway` and Description `delete me`, and click **Create**.
2. On the `Throwaway` row, click the **Delete** icon. In **Confirm Delete**, click **Delete**.
   - Expect: the row is gone, and no **Failed to delete row**.

**Part C: someone with `manage_products` who isn't an admin (1.1's new grants).**
1. Go to **Admin ▸ Manage Products** and open the **Roles** card. Under **Find a person**, type `Kit` and click **Kit Williams**. In **Add a role**, choose **Products Only** and click **Add**.
2. Open the menu under your name and click **Sign Out**. Sign in as `kiteagle@gmail.com` with password `kit`.
3. Go to **Admin ▸ Manage Products ▸ Class Schedules**. Click **Add class**, enter Name `Kit's class`, choose Kind **Recurring**, and click **Save**.
   - Expect: `Kit's class` appears in **Class**. That's the new grant on `classes`. Without it, the banner reads **Could not create class (the name may already be in use)** and no class appears.
   - Also expected, and not Phase 1:
     - a red banner at the top, such as **Class saved, but updating its tags failed.** or **Failed to load class instances**
     - **Failed to load requirements** inside the **Requirements** card
   - The tags, instances, schedules, and the Requirements card's list need `manage_class_schedule`, which this role doesn't have (OQ-32). The Requirements card's read has wanted it since July (`admin_class_requirements_list.cpp:64`); Phase 1 didn't change it.
   - Don't add a requirement group as Kit. It saves, because 1.1 granted the table, but the card can't load it back, so it never appears.
4. Click **Delete class**.
   - Expect: `Deleted class "Kit's class".`

**Class Schedules: the class card** *(found while hand-testing, 11/19; ui only)*
- [x] The card had no name: it showed only in the **Class** picker. Then, with Description as one of five equal columns, a long description ran down a narrow strip.
- [x] As Mason asked:
  - the photo on the left, now 240 px instead of 160, fetched at three times that size so it stays sharp
  - on the right, the name as a heading in a larger font, on its own line
  - **Default capacity**, **Kind** and **Active** sharing the next line
  - **Description** underneath, across the rest of the card
  - a card too narrow for the photo beside the details (a phone) puts the photo on top. On the narrowest phones a label wraps ("Default / capacity") so the three facts stay on one line.
- [x] Tests:
  - `class-schedule-manage.component.spec.ts`: the order of the pieces, the photo URL, a rename, and a sweep of the card at 320–1200 px wide
  - `e2e/mobile-audit/layout/class-properties.spec.ts`: the same checks on the real page at every audit size, and at 1680 and 2560 px

**Not checkable by hand, so covered by the automated tests.** No page sends these requests, and the hand tests don't use curl:
- a signed-out visitor's or a member's add, update or delete (401 or 403)
- reading the five private tables signed out
- GET on `delete_item` (405)
- CommunityFinder's public tables

The tests are listed under 1.2 and 1.4.

---

# Phase 2 — Pay once *(Thomas: "the most critical issue")*

T7, T8, T22, T23 and A8, plus F3, F4, F7, F22 and F23.

### 2.1 A failed first charge leaves nothing behind *(business_logic/payment)*
- [ ] `CreateSubscription` charges **before** it writes the subscription:
  1. Validate.
  2. Create the purchase.
  3. Charge.
  4. Only on success, insert the subscription and its charge record.

  On a decline, the pending purchase is cancelled. No subscription row, no entitlement and no next-billing date are left.
- [ ] The admin no-card path keeps its behaviour. The complimentary path (**Include this month free**) is replaced by 2.3's **rest of this month free**.
- [ ] Tests, through the billing run's due-row reader and `GetActiveByPersonId`:
  - declined card → no subscription, nothing due for billing, the purchase cancelled
  - `RunBilling` afterwards charges nothing
  - retrying three times with a declined card leaves zero subscriptions, then one after a good card
  - the success path is unchanged: one subscription, one entitlement, the buyer seated

### 2.2 A second membership needs a yes *(table helpers + business_logic/payment + endpoints; OQ-1)*
- [ ] `TableHelpers::Subscriptions::GetCurrentPaidByPerson(personId, nowUs)`: the D2 definition, in one query. Tests for each of its four cases, plus expired (not current).
- [ ] `TableHelpers::EntitlementAssignments::GetCurrentMembershipSeatsForPerson(personId, nowUs)`: the seats a person holds on current memberships, with each payer's name. Tests.
- [ ] `SubscriptionHelper::GetMembershipSummary(personId)`:
  - the memberships they pay for: the product, and who holds each seat
  - the memberships they hold a seat on: the product, and the payer
- [ ] `CreateSubscription` refuses with **`ALREADY_HAS_A_MEMBERSHIP`** when the summary isn't empty and the request doesn't say `confirm_additional_membership: true`. With the flag, the membership is created.
- [ ] **Seating the buyer.** Today the buyer always takes seat 1 (`subscription_helper.cpp:316-326`). That stays for a first membership. A buyer who already holds a seat on a current membership gets a new one with every seat empty.
- [ ] **Seats carry over at renewal (P14, F22).**
  - The billing run and the past-due retry copy the current seats onto the new month's entitlement with `TransferAssignments`, which already exists.
  - `PayWithCard` gets a flag, set only by membership charges, that skips its automatic payer seat. A payer who had no seat last month then has none this month either.
  - A membership waiting for its 1st (2.3) has no seats to copy, so its first charge seats by the rule above.
  - Tests through the seat reader and `ClassAccessHelper::CheckAccess`:
    - a couple's membership with both seats renews with both, and the second person can still book on the 2nd
    - a membership bought for a friend renews with only the friend
    - a seat removed before the 1st stays removed
    - the past-due retry does the same
    - one-off purchases still seat their payer
- [ ] `POST /api/subscriptions` and the admin create both take `confirm_additional_membership` and answer 409 `{code, summary}`. The admin Subscriptions page gets the same warning.
- [ ] Tests:
  - a second subscribe without the flag → 409 with the summary, and still exactly one subscription
  - the same for each "current" case (cancelled-but-running, past due, a pending downgrade) and for a seat holder
  - with the flag → two subscriptions, the second with nobody seated
  - expired → allowed without the flag, and the buyer seated
- [ ] Today a site has one database connection, so one person's two requests can't overlap (F13). 7.6 raises that, so this item adds a row lock on the person (`SELECT … FOR UPDATE`) inside `CreateSubscription`. Concurrency test: two simultaneous first subscribes make one subscription and one 409.

### 2.3 Start today or on the 1st *(business_logic/payment + endpoints + ui; OQ-27)*
Fixes F4 and P13. The server works out the dates and the amount from a named **start choice**, so the page no longer sends dates. Examples are for a $160 membership bought on November 17.

| Choice | Who | Charged now | Access from | Next charge |
|---|---|---|---|---|
| **Start today** (`today`) | members and staff | the rest of this month, by the day: 14 of 30 days, **$74.67** | today | $160 on December 1 |
| **Start on December 1** (`next_month`) | members and staff | nothing | December 1 | $160 on December 1, by the billing run |
| **Start today, rest of November free** (`today_rest_free`) | staff only | December's full **$160** | today | $160 on January 1 |
| **Start on the 1st of a later month** (`later_month`, with the month) | staff only | nothing | that month's 1st | on that 1st |

- [ ] **`SubscriptionStartPlanner`** (business_logic/payment): given a choice, a product and "now", it returns the period, the amount to charge now, the access start and the next billing date. It is the only place these are worked out. The checkout and the preview both call it, so they can't disagree.
  - "By the day" counts today, so the proration rule is the one `CalculateProratedCharge` and 3.1's upgrades already use. On the 1st, **Start today** costs the full month. On the last day it costs one day.
  - Unit tests for each choice on the 1st, the 17th and the last day, across a year end (December → January), and for February.
- [ ] **`CreateSubscription` follows the plan** (on top of 2.1's charge-first order):
  - **Start today:** the prorated charge now, and the period runs to the 1st. This is what the staff page's prorate path already does; the server now does it for members too.
  - **Start on December 1 / a later month:** no purchase, no entitlement, no charge. The current period runs from now to the start, with nothing granted, and `next_billing_us` is the start. On the 1st, the billing run charges and grants the month like any renewal (F4's last point). The $0 entitlement path at `subscription_helper.cpp:289-314` no longer runs for a later start, which ends the free month. It stays only for the admin no-card path (2.1).
  - **Rest of this month free:** one charge of the full price now, with access and the period from today to the end of next month. `next_billing_us` is the 1st of the month after. The charge record and the receipt say "includes 14 free days of November".
  - A declined card on the 1st makes the membership past due, as a failed renewal does. Cancelling before the 1st costs nothing.
- [ ] **Endpoints:**
  - `POST /api/subscriptions` takes `start: "today" | "next_month"`. Anything else gets 400, so a member can't send a staff-only choice.
  - The admin create takes all four choices (with `start_month` for the later one), behind `manage_subscriptions`.
  - The old date fields go from both (`start_year`/`start_month`/`charge_now`, and `period_start_us`/`period_end_us`/`charge_now`/`complimentary_period`). The page and the server ship together in sandbox.15.
  - **New `GET /api/subscriptions/start_options?product_id=`** answers signed out too. For each member choice it returns the amount today, the first full charge with its date, and the access start. It's a GET, so it needs no CSRF bootstrap. The signed-out subscribe page (5.2, OQ-14) shows these numbers.
- [ ] **The welcome email (2.4) says what happened with the money:**
  - "You've been charged $74.67 for November 17–30. Your next charge is $160 on December 1."
  - "Nothing has been charged yet. Your membership starts on December 1, with a charge of $160."
  - "You've been charged $160 for December. The rest of November is on us."
- [ ] **Pages:**
  - **Subscribe** (member): the **Start month** list becomes two choices, each with its numbers from `start_options`:
    - **Start today:** "$74.67 today for the rest of November, then $160 on the 1st of each month."
    - **Start on December 1:** "Nothing today. $160 on December 1, then on the 1st of each month."

    The button reads **Subscribe & pay $74.67** or **Subscribe**. Later months go from the member page, since you described two choices. Say if members should keep them.
  - **Create Subscription** (staff): the month list and its two checkboxes become the table's four choices. Each line says what's charged now and when the next charge is, and **a later month** opens the month list.
    - The unticked **Prorate for remaining days** (full price for a part month) goes. Say if staff use it.
- [ ] **Tests** (server):
  - **Start today** on the 17th: 14/30 of the price charged, access from today, and the billing run on December 1 charges the full price
  - **Start on December 1:**
    - nothing is charged, and `GetCurrentMembership` is empty until the 1st
    - the billing run on the 1st charges and grants December; a decline makes it past due
    - cancelling before the 1st means nothing is ever charged
  - **Rest of November free:** the full price is charged now, and access runs to December 31. The billing run on December 1 charges nothing; on January 1 it charges.
  - a member sending `today_rest_free` or `later_month` → 400; staff sending either → created
  - `start_options` signed out answers the same amounts that checkout then charges
  - `GetCurrentPaidByPerson` counts a membership waiting to start, so Back hits the 2.2 warning
  - the three email texts, against the mail double
- [ ] **Specs:**
  - each page's choices and their text
  - what each choice sends
  - the button's amount
  - the staff page's month list appearing only for a later month

### 2.4 One email per signup *(business_logic/payment)*
- [ ] A subscription's first charge sends one email: "Welcome to Gold", with the receipt (amount, card, period) folded in. `PayWithCard` gets a `suppressConfirmationEmail` flag that only `CreateSubscription` sets.
- [ ] Tests against the mail double: exactly one email per signup, containing the amount and the period. One-off purchases still send their confirmation.

### 2.5 The same order twice in a row asks first *(services + table helpers + business_logic/payment + endpoints)*
- [ ] **The window is a setting** (OQ-4):
  - `duplicate_purchase_window_minutes` in `config_secrets`. The default is **10**, and `0` turns the check off.
  - `StudioSettings` (business_logic) reads it, using the default when the row is missing, and validates writes: whole minutes, 0–1440.
  - Tests: the default, a stored value, a bad value refused.
- [ ] `TableHelpers::Purchases::FindRecentPaidMatch(payerId, items, sinceUs)`: a paid purchase by the same payer, with the same products and quantities, since the window began.
- [ ] `purchase_pay_card` and `cart_checkout` refuse with 409 **`RECENT_DUPLICATE`** (name, paid at) unless `confirm_duplicate: true`.
- [ ] Tests:
  - a second identical order inside the window → 409, and Square isn't called
  - with the flag → charged
  - a different item, or outside the window → charged without asking
  - the window set to 0 → never asks

### 2.6 The checkout pages follow the order *(ui: pages/shop)*
- [ ] **Checkout** (`checkout`):
  - after `createPurchase`, the URL gains `?purchase=<id>` (`replaceUrl`)
  - opening with that parameter loads the purchase: paid → the receipt view with no **Pay** button; pending → the same order
  - success → `replaceUrl` to the receipt view
- [ ] **Subscribe** (`subscription-signup`):
  - someone who already pays for or holds a membership sees the D2 warning above the card form:
    - **Subscribe** stays disabled until they tick **I want an additional membership**
    - **Change my membership instead** goes to **My Membership**
    - a 409 `ALREADY_HAS_A_MEMBERSHIP` (bought in another tab) shows the same warning
  - the start choice, with its amounts, is 2.3's
  - the idempotency key is made once per visit and kept with the order, so a retry reuses it
  - success → `replaceUrl`
- [ ] **Cart**: the same as checkout.
- [ ] `RECENT_DUPLICATE` opens a dialog: "You bought **Drop-in Class** at 3:42 PM. Buy it again?" with **Buy again** and **Cancel**.
- [ ] **"Processing…" stays on one line** (T23):
  - `.inline-spinner` beats Material's `display:block`: a selector of two classes, in `_patterns.scss`, with a comment saying why
  - a layout spec checks the spinner and the word share a line, at 320 and 1280 px, on all four pages that use it
- [ ] Specs:
  - each page's resume, receipt and redirect paths
  - the duplicate dialog
  - the membership warning: shown, the tick required, the flag sent
  - the key reused on retry
  - `replaceUrl` on success
- [ ] A Playwright spec for Thomas's exact path: pay, Back, Pay again. It must not reach the server's pay call a second time.

### 2.7 Memberships knows who's signed in *(ui: shop catalog, membership-tier-cards)*
- [x] Signed in with a membership, the page opens with a line naming it: "You're a **Gold** member · **Change membership**". A seat holder sees "You're on **Thomas's Gold Couple's** membership".
- [x] The tier cards take the membership summary (2.2):
  - the card you have reads **Your membership**, with **Manage**, which goes to My Membership
  - the others read **Switch to this** (Phase 3's change flow, or **My Membership** until Phase 3 lands)
  - every card keeps a smaller **Buy another membership** link, which opens the subscribe page with the D2 warning (2.6)
- [x] The catalog loads the summary when signed in. Specs: signed out, a member, a seat holder, and someone paying for two.

### 2.8 Studio Settings *(endpoint + ui)*
- [ ] `GET` and `PUT /api/manage/studio_settings` (`manage_products`). The settings are the duplicate-purchase window (2.5) and, from Phase 6, the late sign-up window (6.2). Endpoint tests: 401, 403, a read, a write, and a bad value → 400.
- [x] **Studio Settings** under **Manage Products** (a tile in the **Website** group, 9.2; until 9.2 the tile sits beside **Site Theme**). Each setting is one field with a sentence saying what it does, for example: "**Ask before charging for the same order again within** [10] minutes. 0 turns this off." Specs.
- [ ] The testers' duplicate memberships on the live site are left alone. They're Square sandbox charges, and the Phase 13 reinstall removes them (OQ-5).

### 2.9 Cancelling one item refunds only that item *(business_logic/payment + business_logic/scheduling; A8, P15)*
F23: one booking cancelled from a cart refunds the whole cart.
- [ ] **`RefundHelper::RefundBookingShare(purchaseId, purchaseItemId, percent, reason, asCredit)`** refunds one booking's share of what was paid:
  - **The share** is its line's `line_total_cents`, less the line's pro-rata part of any cart discount or coupon, divided by the bookings on that line. A guest pair (two bookings on one line, cancelled together) therefore refunds the line once. To verify: which purchase columns hold the discount.
  - **The amount** is the share × the policy's percent. It goes through `ProcessPartialRefundCents`, which is capped at what the purchase still has paid, so repeated cancels can never refund more than was charged.
  - The purchase reads `partially_refunded` until every line is refunded, then `refunded`.
  - To verify: a cart paid partly with store credit and partly by card. It should refund the store credit first, as credit, and then the card.
- [ ] **Who calls it:**
  - the four member-cancel calls in `BookingHelper::CancelBooking` (`booking_helper.cpp:819`, `:872`, `:930`, `:985`)
  - both studio-cancel calls (`session_cancellation_helper.cpp:142`, `:154`)

  `ProcessRefund` (a percent of the whole purchase) stays only where the whole purchase is meant, as in the subscription refund (`subscription_helper.cpp:446`). Series occurrences keep their own per-occurrence path (`class_series_helper.cpp:742`).
- [ ] The cancel result and its banner name the amount: "Booking cancelled. **$160** was added to your store credit."
- [ ] Tests, read back through the voucher balance reader and the purchase reader:
  - **Ashley's case:** a cart with a $160 and a $320 massage, paid together.
    - Cancel the $160 one: $160 of credit, the purchase `partially_refunded` with $320 still paid, and the other booking still confirmed.
    - Then cancel the $320 one: $320 more, and the purchase `refunded`.
  - the same by card: the test Square client sees one refund of $160, not $480
  - a cart with a coupon: each refund is its line's discounted share, and the two refunds add up to what was charged
  - the studio cancels one session of a cart: only that line is refunded
  - a guest pair: both seats cancelled, the line refunded once
  - a single-booking purchase refunds exactly as before
  - a 50% policy window on one line refunds half of that line only
- [ ] The testers' existing over-credits are sandbox money. The Phase 13 reinstall removes them (D8).

### 2.10 Ship it
- [ ] Mutation check:
  - subscription inserted before the charge
  - the `ALREADY_HAS_A_MEMBERSHIP` guard removed, and the buyer seated on a second membership
  - **Start on December 1** granting the waiting days; **Start today** charging the full month; a member allowed `today_rest_free`
  - the renewal's seat copy removed
  - the booking cancel back on the whole-purchase refund
  - the duplicate window set to zero
  - the email flag ignored
  - `replaceUrl` dropped
- [ ] Gates on both platforms; hand-testing walkthrough (written when built, blank database, exact labels). It uses Square's sandbox decline card for 2.1, and Ashley's two-massage cart for 2.9.
- [ ] Release as **`v1.0.0-sandbox.15`**, by `--migrate` (D8).

#### Testing Phase 2 locally, before it ships
Every payment goes to Square's sandbox: a local Debug database is seeded with the sandbox keys, so no real money moves. Use two windows: an ordinary one for Kit, and a private one for Mason (Part D).

**Setting up**
1. **Server build.** A normal **x64-Debug** build. Phase 2 changes only knottyyoga, so the pinned honuware (`838045f`) is fine.
2. **Blank database.** Run **knottyyoga_database_helper.exe**. Its launch settings pass `--recreate_database`. Phase 2 changes no tables, but a blank database starts Kit with no membership, no saved card and no store credit.
3. **Start the site.** Start **knottyyoga_the_server.exe**, then run `ng serve` in `ui\`.

**The amounts are for a test on November 19.** **Start today** charges by the day, today included, in UTC months (OQ-35). On the 19th that's 12 of November's 30 days of the $200.00 Gold price, so **$80.00**. After 4 PM Pacific the server's day is already the 20th, which is 11 days, **$73.33**.

**Part A: a declined card leaves nothing behind (2.1, 2.3, 2.4)**
1. Click **Sign In**, and sign in as `kiteagle@gmail.com` with password `kit`.
2. Click **Memberships** in the header.
   - Expect: the membership cards, each with **Subscribe**, and no "You're a … member" line above them.
3. On **Knotty Yoga Gold Membership**, click **Subscribe**.
   - Expect, under **When to Start**, two choices, and no **Start month** list:
     - **Start today**: "$80.00 today for the rest of November, then $200.00 on the 1st of each month."
     - **Start on December 1**: "Nothing today. $200.00 on December 1, then on the 1st of each month."
4. Under **Payment Method**, **Card** shows **Add new card...**. Enter Square's decline card `4000 0000 0000 0002`, any future expiry date, any CVV and any ZIP. Type `Decline` in **Card nickname (optional)** and click **Save Card**.
   - The sandbox may refuse to save this card. If it does, write down its message here and go on to step 7. The page charges only saved cards, so 2.1 then rests on its server tests.
5. Leave **Start today** chosen and click **Subscribe & pay $80.00**.
   - Expect: while it works, the button shows its spinner and **Processing...** on one line (T23).
   - Then a red message that the card was declined, with the form still there.
6. Click **Subscribe & pay $80.00** twice more, as the testers did.
   - Expect: the same message each time.
   - Open the menu under your name and click **My Subscriptions**. Expect: no subscription.
   - Click **Memberships**. Expect: still no "You're a … member" line.
7. Open the Gold **Subscribe** page again. In **Card**, choose **Add new card...**, enter `4111 1111 1111 1111`, type `Good` as the nickname, and click **Save Card**. Click **Subscribe & pay $80.00**.
   - Expect: **Subscription Created!**
   - If mail is set up locally, expect exactly one email to Kit, **Welcome to Knotty Yoga Gold Membership!** It says "You've been charged $80.00 for November 19–30. Your next charge is $200.00 on December 1." There's no separate payment receipt (2.4).
8. Press the browser's **Back**, then **Forward**.
   - Expect: **Subscription Created!** again, not a fresh form.

**Part B: a second membership needs a yes (2.2, 2.6, 2.7).** Still Kit.
1. Click **Memberships**.
   - Expect, above the cards: "You're a **Gold** member", with **Change membership**.
   - **Knotty Yoga Gold Membership** reads **Your membership**, with **Manage**.
   - The other cards read **Switch to this**.
   - Every card has a smaller **Buy another membership**.
2. On **Knotty Yoga Silver Membership**, click **Buy another membership**.
   - Expect, above **Payment Method**: "You already have a **Gold** membership. Subscribing adds another membership, billed separately."
   - **Subscribe & pay** is greyed out.
3. Tick **I want an additional membership**.
   - Expect: the button turns on. Don't click it.
4. Click **Change my membership instead**.
   - Expect: **My Subscriptions**, listing Kit's Gold.

**Part C: the same order twice asks first (2.5, 2.6).** Still Kit.
1. Click **Services**. On **Our Services**, under **Massage**, click **Book Now**.
2. Click a day that has times, click an hour to open it, and click **60-Minute Massage $160.00** beside one of its times.
3. Under **Payment Details**, choose **Card on File** and pick `Good`. Click **Book and Pay $160.00**.
   - Expect: **Booking Confirmed!**
4. Press the browser's **Back** to return to **Our Services**. Click **Book Now** under **Massage**, and pick a different time with **60-Minute Massage $160.00**. Pay with `Good` again: click **Book and Pay $160.00**.
   - Expect: a dialog, **Already bought**: "You bought **Massage** at 3:42 PM. Buy it again?", showing the time of step 3's payment, with **Cancel** and **Buy again**.
   - This asks even though the time is different (OQ-39).
5. Click **Cancel**.
   - Expect: the dialog closes, and a blue note reads "Nothing was charged. Your earlier order is in My Purchases." The page still has its **Book and Pay** button.
   - Menu ▸ **My Purchases**: step 3's order reads **Paid**. This one reads **Pending**, with nothing charged (OQ-47).
6. Click **Book and Pay $160.00** again, and this time click **Buy again**.
   - Expect: **Booking Confirmed!**

**Part D: Studio Settings (2.8).** Mason, in a private window.
1. Sign in as `masonbendixen@gmail.com` with password `pass`.
2. Go to **Admin ▸ Manage Products** and open the **Studio Settings** card.
   - Expect: **Duplicate orders**, and "Ask before charging for the same order again within **10** minutes." with "0 turns this off." under it.
3. Type `1441` and click **Save**.
   - Expect: "The duplicate-order window must be a whole number of minutes, from 0 to 1440". Nothing is saved.
4. Type `0` and click **Save**.
   - Expect: "Saved. The shop is using this now."
5. In Kit's window, book one more **60-Minute Massage $160.00** with `Good`.
   - Expect: no dialog, straight to **Booking Confirmed!**
6. In Mason's window, set it back to `10` and click **Save**.

**Part E: cancelling one massage refunds only that one (2.9, Ashley's cart).** Kit.
1. Click **Services**, then **Book Now** under **Massage**. Pick a time and click **60-Minute Massage $160.00**. Under **Booking Summary**, click **Add to Cart**.
   - Expect: **Shopping Cart**, with the 60-minute massage.
2. Click **Services**, then **Book Now** under **Massage** again. Pick another time, click **120-Minute Massage $300.00**, and click **Add to Cart**.
   - Expect: both massages in **Shopping Cart**.
3. Under **Payment Details**, choose **Credit Card**, enter `4111 1111 1111 1111`, and click **Pay $460.00**.
   - Expect: no dialog, since this order isn't the same as Part C's.
   - Expect: **Payment Complete!** with both lines, and **Purchase Details**.
4. Press the browser's **Back**, then **Forward**.
   - Expect: **Payment Complete!** again, with no **Pay** button (2.6).
5. Open the menu under your name and click **My Bookings**. Under **Upcoming Bookings**, find step 1's 60-minute massage and click **Cancel Booking**. Tick **Refund as store credit instead of card refund**, and click **Yes, Cancel**.
   - Expect: "Booking cancelled. **$160.00** was added to your store credit."
   - The 120-minute massage still reads **Confirmed**.
   - Before 2.9, this cancel credited the whole $460.00 (F23).
6. Menu ▸ **Profile** ▸ **Vouchers & Credits**.
   - Expect: one active credit of **$160.00**.
7. Menu ▸ **My Purchases**.
   - Expect: the cart's order reads **Partially Refunded**.
   - Opened: **Total** $460.00 and **Paid** $300.00.
8. Back on **My Bookings**, cancel the 120-minute massage, this time without ticking the box.
   - Expect: "Booking cancelled. **$300.00** will be refunded to your card."
   - **My Purchases**: the order reads **Refunded**.
   - **Vouchers & Credits**: still only the $160.00.

**Not checkable by hand, so covered by the automated tests:**
- the billing run on the 1st: a December 1 start charged and granted, and seats carried over at renewal
- a member sending a staff-only start choice (400)
- two first signups at the same moment (the row lock, OQ-46)
- coupon-discounted shares, guest pairs, a 50% policy window, and the studio cancelling one session of a cart
- the pay-twice path on Checkout, which no seeded product reaches. The Playwright spec `e2e/mobile-audit/layout/pay-once.spec.ts` runs Thomas's exact pay, Back, Pay in mock mode.

---

# Phase 3 — My Membership: see it, change it

T5, T6, T8, T13, T24 and A4, plus F6.

### 3.1 Changing a membership charges first *(business_logic/payment)*
- [ ] **Upgrade order:**
  1. Create the purchase.
  2. Charge.
  3. Only on success: cancel the old subscription, revoke its entitlement, create the new one, and move the seats.

  A decline changes nothing.
- [ ] **What an upgrade costs:** the price difference, prorated over the days left in the period with the existing `CalculateProratedCharge` (OQ-2, decided).
- [ ] **Downgrades** stay at period end. The new subscription's first entitlement carries the seat holders over.
  - **Fewer seats** (OQ-3, decided): the change must name who comes off (`remove_person_ids`), exactly enough to fit. Without that, the server refuses with **`CHOOSE_WHO_TO_REMOVE`** and lists the seat holders. The payer may name themselves.
  - The people named keep their seat until the period ends. Each gets an email when the change is made, giving the date, and the payer gets a confirmation that names them.
- [ ] Changing to the same product is refused. Changing while a change is already scheduled replaces the scheduled one.
- [ ] `PreviewChange(subscriptionId, newProductId)` writes nothing. It returns:
  - immediate or at period end
  - the charge today
  - the date it takes effect
  - when the new tier has fewer seats: the seat holders, and how many must come off
- [ ] Tests:
  - a declined upgrade leaves the old subscription active, with its entitlement and seats
  - the prorated amount on the 1st, the 15th and the last day
  - a downgrade carries seats over, through the billing run and the assignment reader
  - 3 seats to 2: refused with no one named, or with the wrong number. With one person named, they keep the seat until the period ends and lose it after, read through the billing run and the assignment reader.
  - the preview equals what the change then does

### 3.2 Endpoints *(endpoints/subscriptions)*
- [ ] `GET /api/subscriptions/<id>/change_preview?product_id=` returns the preview. Only the subscriber may call it: 404 for anyone else's subscription.
- [ ] `POST …/change_product` gets the same ownership test (to confirm), takes `remove_person_ids`, and returns the codes from 3.1.
- [ ] Endpoint tests: 401, 404 for another person's subscription, the preview shape, and a decline.

### 3.3 "My Membership" *(ui: pages/account/subscriptions, header menu, shared pipe)*
- [ ] **My Subscriptions** becomes **My Membership** everywhere: the user menu item, the My Account tile, and the page title. The URL stays.
- [ ] **With a membership:**
  - the tier and a status badge (**Active**, **Past due**, **Ends Dec 31**)
  - this period's dates and the next charge
  - the seats, with who holds each one (Phase 4 adds **Add a person**)
  - the card on file
  - **Change membership**, which opens the tier cards with each one's preview: "**Upgrade** — $42.67 today, then $X a month", or "**Switch** on Jan 1". Choosing one opens a confirm.
  - a downgrade to fewer seats asks first: "Gold Couple's has 2 seats. Choose who comes off on Jan 1:", with a check box per seat holder. **Confirm** is enabled once enough are ticked.
- [ ] **More than one membership:** each one you pay for is its own card. Then **Shared with you** lists the memberships you hold a seat on, with the payer's name.
- [ ] **Without a membership:** the tier cards themselves, with no "Browse Memberships" click (T6), plus "Not sure yet? Try an intro workshop" (5.1).
- [ ] Permission names display as words: a shared `permissionDisplayName` turns `gold_member` into **Gold member**, reusing `role.types.ts`'s rule. Used on My Membership, its detail page and User Information (T24).
- [ ] Specs: both states, the preview text, confirm and decline, the menu label, and the display names.

---

# Phase 4 — Share a membership in one step

T9–T12, G17 and A6, plus F8–F11.

### 4.1 Schema: an invitation remembers the seat *(db_schema + migration)*
- [ ] `gift_permissions.entitlement_id` and `gift_permission_invitations.entitlement_id`: nullable foreign keys to `entitlements`.
- [ ] `gift_permission_invitations.invited_email` is stored lower-cased. Lookups lower-case too (P9).
- [ ] The CLAUDE.md new-column checklist: admin column data info, friendly names, and the display template.
- [ ] Migration **`app/0014_sharing_invites_name_a_seat`**: adds both columns if missing. It is idempotent. Existing invitations aren't lower-cased, because production is reinstalled (D8).
- [ ] Tests: the columns exist on a new database and after the migration; a mixed-case invitation is found by the registration lookup.

### 4.2 Table helpers *(sql_util/table_helpers)*
- [ ] `GiftPermissions` and `GiftPermissionInvitations` read and write `entitlement_id`. The invitation lookup is case-insensitive.
- [ ] Tests for both, in their existing test files.

### 4.3 Business logic: invite a person to a seat *(business_logic/payment)*
- [ ] `MembershipSharingHelper::InviteToSeat(payerId, entitlementId, email)`, in a new `membership_sharing_helper.{h,cpp}`. It returns one of:
  - **`Seated`**: the person already accepted the payer before; the seat is assigned now.
  - **`RequestSent`**: an account exists; a pending request is made that names the seat.
  - **`InvitationSent`**: no account; an invitation is stored by email, naming the seat.
  - **`NoFreeSeat`**, **`NotYourMembership`**, **`AlreadySeated`**, **`Yourself`**.
- [ ] **Accepting** (the existing `AcceptRequest`): when the request names a seat and one is free, the seat is assigned in the same transaction. If the membership is now full, the permission is still accepted, and the payer gets an email saying the seat is full.
- [ ] **Registering** (`ProcessInvitationsForNewUser`) carries the seat onto the request it makes.
- [ ] **Emails, with links** built from `branding.websiteUrl` (D5), the address URL-encoded as the reset link does:
  - **Request:** "Thomas Potthast wants to share his Gold Couple's membership with you." **Accept or decline** → the Sharing page.
  - **Invitation:** the same sentence. **Create your account** → registration with the email filled in and a return to Sharing. **Already have an account? Sign in** → sign-in with the same return.
  - **Seated:** "You're now on Thomas's Gold Couple's membership", linking to My Membership. This is a new email.
  - The copy says *membership*, not massages and spa visits.
- [ ] Tests:
  - each outcome, read through the assignment reader and `GetGiftPermissions`
  - accept takes the seat; accept with no seat free accepts without one and emails the payer
  - registration with different capitals finds the invitation
  - each email contains its link, with a plus address correctly encoded (`mason+test@…`)
  - no brand literal in the templates

### 4.4 Endpoints
- [ ] `POST /api/entitlements/<id>/invite` `{email}`: the outcome and a sentence.
- [ ] `GET /api/gift_permissions` gains each person's seat (the membership name) for the Sharing page.
- [ ] Endpoint tests for every outcome, plus 401/404.
- [ ] The register CSRF bootstrap path is unchanged: only signed-in calls are added.

### 4.5 Registration keeps where you were going *(@honuware/ui: auth)*
- [ ] The login page's **Register** link carries `returnUrl`.
- [ ] Register reads `?email=` (pre-fills it) and `?returnUrl=`. It keeps `returnUrl` in `sessionStorage` through check-email and verify.
- [ ] After activation, login uses the stored `returnUrl` when it has no query parameter, still through `sanitizeReturnUrl`.
- [ ] Library specs for each step, including a hostile `returnUrl`.
- [ ] `@honuware/ui` minor release; install; Angular gates.

### 4.6 One Sharing page and a two-field seat card *(ui)*
- [ ] **Sharing** replaces **Purchase Sharing**. Its URL is `/my/sharing`, and the old one redirects to it.
  - **Waiting for you** comes first: each request with **Accept** and **Decline**, naming the membership and seat.
  - **People you share with**: each person, their status (**Invited**, **Waiting for them**, **Sharing**) and the seat they hold ("On your Gold Couple's membership"). **Remove** asks to confirm.
  - **Add a person**: one email field and an **Invite** button.
- [ ] **The seat card** (after Subscribe and on My Membership):
  - Seat 1 reads "**You**" (the `[assignments]` bug fixed), and **Assign to myself** never shows to someone already seated.
  - When no seat is taken yet (a second membership, 2.2), the card asks "**Who is this membership for?**" It offers **Me**, the people you already share with, and the invite field.
  - Each empty seat shows the people you already share with as buttons, then "Or invite someone new" with an email field and an **Invite** button. Enter works too. That is Gifford's and Ashley's "pick from the people I already share with" (G17, A6). With a handful of people, buttons save the extra tap a dropdown needs.
  - "No one by that name" shows when a search matches nobody (G17).
  - The sharing link goes to `/my/sharing`.
- [ ] **The banner** (D5): signed in with a request waiting, the home page and My Account show "**Thomas Potthast** wants to share his **Gold Couple's** membership with you", with **Accept** and **Decline**.
- [ ] Specs: each section, each outcome's message, the banner, the redirect, Enter on the field, and the seated state. Mock and network methods, with `ServerAccess.mock.spec.ts` cases.

### 4.7 Ship it
- [ ] Mutation check: the seat not taken on accept; the lookup case-sensitive again; the link dropped from each email; `[assignments]` unbound.
- [ ] Hand-testing walkthrough, written when built. Two browsers: Mason buys Gold Couple's; Kit is invited by email; a brand-new address is invited and registers from the email.

---

# Phase 5 — The newcomer's path

T1–T4, T27, G1, G7 and G13–G16.

### 5.1 Intro workshops *(endpoints + ui)*
- [ ] `visible_event_sessions` returns `product_code` (through its `KeyValueTable` conversion). Endpoint test.
- [ ] **Events** filters to intro workshops with `?type=intro` (product code `intro-workshop`). The heading reads **Intro Workshops**, with **See all events**.
- [ ] When none is scheduled: "No intro workshop is on the calendar right now. **Email us** and we'll set one up." **Email us** is a `mailto:` to the studio's contact address from `site_info`. Which slot holds that address is to verify.
- [ ] Every intro-workshop link points at the filtered list:
  - Getting Started's first step
  - the About block that says "Please come to our next intro workshop"
  - the new home call to action for visitors who aren't members: "**New here? Start with an intro workshop**"
- [ ] Specs: the filter, both empty states, and each link's target.

### 5.2 Memberships *(business_logic/payment + endpoints + ui)*
- [ ] **What each tier includes, from the data:** `CatalogHelper` lists, for each membership product, the classes its granted permission opens. The data is the classes' requirement groups (the `ClassAccessHelper` rules), with "needs instructor approval" when a skill is also required. It goes on the catalog response.
  - Tests through `ClassAccessHelper::CheckAccess`, against the seeded rules: Silver lists only Beginner Knotty Yoga; Gold lists all six, three marked approval.
- [ ] **One / Two / Family** (T2; OQ-15, option A): two cards, **Silver** and **Gold**, with the size choice as radio buttons inside each card (the sketch in OQ-15).
  - The sizes come from the products, grouped by the permission they grant and by `seats_default`, so a new size needs no code.
  - Each size shows its price and saving: "Couple — $300/month ($150 each, save $50 a person)".
  - Silver has no Family size, so its card shows two.
  - **Subscribe** opens the chosen product.
- [ ] **Silver vs Gold:** under the cards, a comparison of the included classes from the line above (✓, ✓ with approval, —).
- [ ] **The tier descriptions are Mason's** (OQ-9). In the seed they go on every product of the tier:
  - **Silver:** "Our introductory membership, with a selection of our most beginner-friendly and accessible classes. Start here until your instructors suggest moving on to Gold and you feel ready. Part of acrobatics is learning the terminology and body positions, and learning to move and organize yourself in a new way. These classes are the foundation for aerial, partner acro and handstand training."
  - **Gold:** "More advanced classes that also bring in elements of circus acrobatics, plus our specialty classes in handstands, aerial and partner acrobatics."
  - Seed test through the catalog reader.
  - Approved as written here (OQ-9).
- [ ] **"Browse bookable services →"** comes off Memberships (G13).
- [ ] **Look before you sign up** (G14; OQ-14, option A): the subscribe page opens signed out.
  - The route's sign-in guard comes off (`shop.routes.ts:20`).
  - Signed out, the page shows the tier, the size and both start choices with their amounts, from 2.3's `start_options` (which answers signed out).
  - Where the card form would be, it says **Sign in or create an account to continue**. Either one returns to this page with the size and start choice kept (4.5).
- [ ] Specs: the size choice with its prices and savings, Silver without Family, the comparison, the signed-out subscribe page with its amounts, and the return with the choices kept.

### 5.3 Getting Started: look first, commit last *(seed + Page Content + ui)*
- [ ] **New step order** (seed; OQ-8, decided):
  1. What to expect
  2. Try an intro workshop
  3. Find the classes for you
  4. Meet the instructors
  5. Pick a membership
  6. Create your account and plan your week

  Check-in, workshops and notifications move out of the steps, to the end of step 6 and to My Account.
- [ ] **What to expect** is the new-student FAQ (G7), from Mason's text (OQ-9).
  - Each class's own requirements stay on its class page; this step links to **All Classes** for them.
  - It is a step body in Markdown; to verify that the steps render Markdown. **Contact us** is a `mailto:` link to the studio's address (5.1).
  - The page text, approved (OQ-9): your wording, split under three headings.

  > **Is Knotty Yoga right for me?**
  > Knotty Yoga is physically rigorous. You'll keep up best if you're reasonably fit and free of significant injuries; carrying a lot of extra weight makes classes hard to keep up with. If that describes you, **contact us** about one-on-one training to get started.
  >
  > **What to wear**
  > Fitted gym clothes that aren't baggy, so nothing catches on the equipment: tights, leggings or sweatpants (grippy cotton is best) and a fitted T-shirt or tank top. Men: snug underwear that keeps things in place. No watches, metal, zippers or jewelry.
  >
  > **What to bring**
  > A water bottle (strongly encouraged). You don't need a yoga mat: our floors are padded, with good traction.
- [ ] Each step gets an `id` anchor. **Get started** links land at the top (8.1 fixes the cause).
- [ ] Seed tests through the Getting Started reader: order, titles, links.

### 5.4 My Account by what people come to do *(endpoints + ui)*
- [ ] **The membership card comes first.**
  - A member sees the tier, a status badge, the dates (G15) and **Manage membership**.
  - Someone without a membership sees "You don't have a membership yet", with **Choose a membership** and **Try an intro workshop**.
- [ ] **The tiles in four groups,** in this order (no longer alphabetical):

  | Group | Tiles |
  |---|---|
  | **Classes** | My Weekly Plan, Today's Classes, My Bookings, Workshops & Series, Attendance History |
  | **Membership & payments** | My Membership, Purchase History, Saved Cards, Vouchers & Credits |
  | **People** | Sharing, Favorite Instructors |
  | **Profile & settings** | User Information, Change Password, Notifications, My Skills |

- [ ] The **My Schedule** tile becomes **My Weekly Plan**, the name Thomas used and the page's own heading (to verify). It then no longer shares a name with the Our Classes toggle (6.3).
- [ ] **Favorite Instructors** (T27, G16): each card links to the instructor's page and lists their next three classes.
  - The favorites endpoint returns them, from the same reader the instructor page uses, so it's one call, not one per instructor.
  - Endpoint test; spec.
- [ ] Specs: both card states, the group order, and the display names.

### 5.5 No membership, no dead end *(seed + ui)*
- [ ] The home page's **Upcoming classes** row gets `hidden_when_not_member` (seed). Non-members see the Get Started band and the intro call to action instead (T3).
- [ ] **My Weekly Plan** with no membership: "You need a membership to plan classes", with **Choose a membership** and **Try an intro workshop**. A member who simply has no eligible classes keeps today's message, plus a link to the class pages.
- [ ] The upcoming-classes empty text says the same.
- [ ] Specs for each state.

---

# Phase 6 — Planning the week

T14–T17, A1–A3 and A9.

### 6.1 A tap changes one class *(business_logic/scheduling + ui)*
- [ ] `AddTemplateEntry` checks the one slot (`IsSlotEligible`) instead of rebuilding the whole eligible list.
  - Test: for every seeded slot and a Silver, a Gold and an approved Gold member, it gives the same answer as the list reader.
- [ ] The per-tap confirmation email stops (OQ-28, decided); the weekly digest says what's planned. Test against the mail double: five taps, no email.
- [ ] **Calendar entries without the emails:** a planned class reaches someone's calendar through the personal calendar feed. That's the iCal work in the tree, and **Notifications** already offers the subscription.
  - Verify the feed includes weekly-plan classes and their skips; add them if not, with tests through the feed reader.
  - **My Weekly Plan** gets an **Add your plan to your calendar** link to that subscription. Spec.
- [ ] **My Weekly Plan:**
  - a tap flips that class at once and saves it in the background
  - each class has its own busy state, so several taps can be in flight
  - a failure flips it back with a message
  - no reload, no spinner over the grid
- [ ] The **Upcoming** tab loads when it is opened, not with the page.
- [ ] Specs: three quick taps make three calls and no reload; failure reverts.

### 6.2 A class leaves the lists two hours after it ends *(services + business_logic/scheduling + endpoints + ui; OQ-26)*
- [ ] **A setting:** `late_signup_window_minutes` in `config_secrets`, default **120**, on **Studio Settings** (2.8). Read through `StudioSettings` with its default. Tests.
- [ ] **The cut-off:** a class stays on the sign-up lists until its **end + the window**. Until then it can still be signed up for or marked attended; after that it's gone.
  - Today the lists start at midnight and keep the whole day, while the sign-up check refuses a class once it has started. To verify which checks do that, and move each to this cut-off.
  - `GetUpcomingClassesForPerson`, `ResolveClassesInRange` and `GetTodayClassesForPerson` drop a class past the cut-off.
  - The same cut-off applies to a member's own sign-up, a weekly-plan "attending" mark, and staff check-in.
- [ ] **Labels, so a past class isn't mistaken for an upcoming one** (T16):
  - classes that have started sit under **Earlier today**, after the upcoming ones
  - an ended class reads "**Ended 6:00 PM** · You can still mark attendance until 8:00 PM"
  - a class under way reads "**Started 10 minutes ago**"
- [ ] The page asks from now minus the window, not from midnight.
- [ ] Tests with a fixed clock, for a 5:00–6:00 class:
  - 5:30 → listed under **Earlier today** as started; sign-up allowed
  - 7:59 → still listed; sign-up and staff check-in allowed
  - 8:01 → gone, and sign-up refused
  - the window set to 0 → gone at 6:00
- [ ] Specs for the grouping and both labels.

### 6.3 Our Classes: the right view first, and what you've planned *(ui)*
- [ ] **Without a membership, the page opens on Full Schedule** (OQ-13, decided; F19):
  - signed in **with** a current membership or seat → the eligible-only view, as today
  - signed in **without** one, or signed out → **Full Schedule**. Each members' class shows the existing lock and "**Requires a membership**", linking to **Memberships**.
  - the page knows which from the membership summary (2.2), loaded with the schedule
- [ ] **The toggle, on Our Classes and on the calendar,** gets three choices: **On my plan**, **Classes I can take** and **Full schedule**.
  - **Classes I can take** is today's **My Schedule**, renamed. It filters by access, and "My Schedule" was also the weekly-plan page's tile, which is likely what confused Thomas (OQ-29). Ashley read it on the calendar as "the classes I opted in to" (A3, F25).
  - **On my plan** is what Ashley asked for. It shows only the classes on your plan: your weekly plan, one-off plans and bookings. A class you can't make stays in the view, marked **Can't make it** (6.6), so your note is still there.
  - Members still start on **Classes I can take**. Each page remembers the last choice in the browser, and every read and write is wrapped so private mode works. Say if members with a plan should start on **On my plan** instead.
- [ ] A class on your weekly plan shows **On your weekly plan** and no sign-up control. A one-off plan shows **Planned**.
- [ ] A failed plan lookup shows "Couldn't load your plan", instead of silently treating nothing as planned (`our-classes.component.ts:219-222`).
- [ ] Specs:
  - the starting view for a member, a non-member and a visitor
  - the lock and its link
  - the rename, on both pages
  - **On my plan** for a member with two weekly days: only those days' classes, plus a one-off and a booking, with a skipped one shown as **Can't make it**
  - the remembered choice, and the page still working when the browser's storage throws
  - both planned states
  - the error

### 6.4 Which class is right for me *(db_schema + table helpers + seed + ui)*
- [ ] `classes.sort_order` (`INT NOT NULL DEFAULT 0`), with migration **`app/0015_classes_sort_order`** and the admin metadata checklist.
  - `TableHelpers::Classes` orders by `sort_order`, then name.
  - Seed order: Beginner Knotty Yoga, Beginner Partner Acro, Knotty Yoga, Handstands, Aerial Rope, Intermediate Partner Acro.
  - A **Sort order** field in Class Schedules' class dialog.
- [ ] **Prepares you for** (OQ-23): a `class_prepares_for` table (`class_id`, `prepares_class_id`, unique pair), with migration **`app/0016_class_prepares_for`**, the admin metadata checklist, and a table helper with tests. It is edited in the Class Schedules class dialog. Seed, from Mason's pairs (OQ-23):
  - Beginner Knotty Yoga → Knotty Yoga, Beginner Partner Acro
  - Knotty Yoga → Handstands, Aerial Rope, Beginner Partner Acro
  - Beginner Partner Acro → Intermediate Partner Acro
- [ ] **The class page:**
  - **Start here** on classes that need only a membership
  - on a class that needs more, a **How to get here** section: the required skills (as today), "Requires attending: Knotty Yoga" from the predecessor slot, and "Build up to it in **Beginner Knotty Yoga**" from `class_prepares_for`
  - on a class that prepares for others, "**Leads to:** Knotty Yoga, Aerial Rope"
  - the class detail endpoint returns both lists; endpoint test
- [ ] Our Classes and All Classes list by the new order.
- [ ] No new description copy (OQ-9). The **Leads to** line says it on every class that prepares for another, and Beginner Partner Acro's description already says it leads to Intermediate.
- [ ] Tests: the migration; the order through the catalog reader; specs for the badge and section.

### 6.5 From a class to when it runs *(business_logic/scheduling + endpoints + ui; A9)*
F26: the class page doesn't say when a class runs, and its calendar link can land on a calendar without it.
- [ ] **Weekly times on the class page.** The class detail response gains `weekly_times`, built in `ClassCatalogHelper` from the class's current schedule slots and grouped by time, for example:

  > Mondays & Wednesdays, 6:00–7:00 PM · Saturdays, 10:00 AM–12:00 PM

  - The times are in the studio's time zone. The location is added when a class runs in more than one.
  - A slot whose date range has ended is left out.
  - `class_schedule_slots` counts days 0 = Sunday … 6 = Saturday, unlike the weekly templates' ISO days.
  - It reads the slots, not derived sessions, so it adds nothing to the class page's load (7.5).
- [ ] The **When it runs** card shows the times above its calendar link.
- [ ] **See this class on the calendar** opens the calendar filtered to that class (`?class=<id>`) in every view. A class you can't take shows with its lock instead of on an empty calendar.
- [ ] Tests:
  - through the class detail endpoint, on the seeded schedule
  - a class with slots on two days at one time and on a third day at another, which groups as in the example
  - an ended slot left out
  - a Sunday slot comes out as Sunday
- [ ] Specs: the card's times, the link's address, and the filtered calendar showing a locked class.

### 6.6 Going, can't make it, or not planned *(ui: shared attendance control; A1, A2)*
F25: the class list's big button reads as the status, and a skipped class looks unplanned on Our Classes and the calendar.
- [ ] **A shared attendance control:** one check box with three states, from the attendance data every page already loads (`attendance-map.ts`):
  - ☑ **Going**
  - ☒ **Can't make it**, with the class name struck through and the note under it
  - ☐ **Not planned**, with "Tap to plan"
- [ ] **Tapping it:**
  - Not planned → Going.
  - Going → the existing **Can't make it?** dialog, for an optional note.
  - Can't make it → Going.
  - Each class keeps its own busy state, as in 6.1.
- [ ] **Where it goes:**
  - the home page's class list, My Weekly Plan's **Upcoming** tab and **Today's Classes**, in place of the **I can't make it** / **I'll be there** buttons (`upcoming-classes`, `today-classes`)
  - Our Classes and the calendar, in place of the two-state row (`calendar-event`)

  The status sits next to the class name, so it can't be taken for an answer about another row.
- [ ] Specs:
  - each state on each page
  - **Ashley's Wednesday:** a class skipped with a note on the home page reads **Can't make it**, with the note, on Our Classes and on the calendar
  - unticking opens the dialog
  - a failed save puts the box back
  - a layout spec at 320 and 1280 px: the control and the class name share a line

---

# Phase 7 — Speed

T18, T43, T45, T46, G10, A9 (the slow calendar) and P11.

### 7.1 Measure first *(server_components: testing; knottyyoga)*
- [ ] A **query-counting transaction provider** for tests: it wraps the test provider and counts statements. Its own test.
- [ ] Baselines, recorded here before any change: queries and time for `/api/site_info`, `/api/calendar` (90 days), `/api/me/upcoming_classes` (90 days) and `/api/classes/<id>` on the seeded schedule; and the signed-out home page's time to first paint (Playwright, against `ng serve` and the local server).

### 7.2 `site_info` from memory *(server_components: platform)*
- [ ] `SecretsHelper::LookupMany(names)` reads every slot in **one** query.
- [ ] A per-site cache of the finished answer. It is cleared by any `config_secrets`, site-font or provider write in this process, and kept at most 60 s so other replicas catch up.
- [ ] `ETag` and `Cache-Control: public, max-age=60`; a matching `If-None-Match` gets 304.
- [ ] The three content fields this app no longer reads (`site_hero_headline`, `site_hero_image_url`, `site_about_markdown`) stay. CommunityFinder may use them; to verify.
- [ ] Tests:
  - ≤ 4 queries, through the counter
  - a `set_secret` shows on the next call
  - a 304 on a matching tag
  - two sites don't share a cache entry (the existing two-tenant harness)

### 7.3 First paint doesn't wait *(ui)*
- [ ] `SiteConfigService` applies the last site config from `localStorage` at once, then fetches in the background and re-applies if it changed. A first-ever visit applies the built-in defaults instead of waiting. Every read and write is wrapped, so private mode works.
- [ ] Sign-in and site info start together at boot, not one after the other (needs 7.6).
- [ ] `PreloadAllModules` gives way to preloading the routes linked from the current page, so the lazy chunks stop competing with the first page.
- [ ] Specs: cached, uncached, and a changed config; and the boot order.

### 7.4 Fonts at first paint: the server fills in the page head *(server_components: platform; knottyyoga; ui; ui/package; OQ-22, decided)*
**Decided: OQ-22's option D.** The fonts stay a database setting, and the first byte of the page already names them. This is the biggest item in Phase 7: about twice the size of 7.8, plus a CloudFront change to how page addresses are served.

- [ ] **The template** *(ui, ui/package)*:
  - The built `index.html` gets one marked block in its head, `<!-- site-head -->` … `<!-- /site-head -->`. The block holds the defaults (the seed's font address and the site's name), so `ng serve` and a direct S3 read still work.
  - The build stops inlining the Google Fonts CSS, since that would bake in the build's fonts (D7).
  - `deploy_ui.sh` also uploads the file as `shell/index.html`.
  - A spec checks the built file has exactly one block.
- [ ] **The server keeps a copy of the template** *(server_components: platform)*:
  - It fetches `/shell/index.html` through CloudFront at start-up, then at most once a minute with `If-None-Match`. A UI deploy reaches it within a minute, so the script names always match the live build.
  - To verify: the server can reach its own CloudFront address. It already reaches Square and SES.
  - Until the first fetch succeeds, it answers 503, and CloudFront serves its last good copy (below).
- [ ] **A page route for every address outside `/api/`** *(server_components: platform)*, through Crow's catch-all. It returns the template with the block filled in, everything HTML-escaped:
  - **Fonts:** the `preconnect`s and stylesheet links from the site's font rows. They're the addresses `SiteConfigService` applies after start-up, so that link becomes a cache hit.
  - **The site's own tags:** title, description and theme colour, from `site_info` (7.2's cache).
  - **Per-page tags from an app hook** *(knottyyoga)*: one for the class, instructor and event pages that looks up the page's title, description and photo (11.1). Any other address gets the site's defaults. A request for `/` arrives as `/index.html` (the distribution's default root object) and gets the home page's tags.
  - **Caching:** kept per site and address for at most 60 s, and cleared by the writes that clear 7.2's cache. With CloudFront's 60 s on top, a Site Theme or font save shows within a minute. No timer is needed.
  - `Cache-Control: public, s-maxage=60, stale-if-error=86400`. CloudFront then answers most visits itself, and keeps serving the last good page if the server is down.
- [ ] **CloudFront** *(Mason, in the console, from steps written here when the item is built; Deploying to AWS records it)*:
  - The default behaviour (page addresses) goes to the server.
  - New behaviours keep the built files on S3: `*.js`, `*.css`, `/media/*`, `/assets/*`, `/shell/*` and the favicon files.
  - The SPA error-page fallback goes, since the server now answers every page address.
- [ ] **Tests:**
  - the page carries the seeded font address; after a font change through Site Theme's endpoint, the next request carries the new one
  - a class page has that class's title, description and photo; an unknown address gets the site's defaults; `/index.html` gets the home page's
  - a class name containing `<`, `"` or `&` comes out escaped, as a stored-XSS guard
  - `/api/…` never reaches the page route
  - two sites get their own heads (the two-tenant harness)
  - the template is fetched again when its ETag changes, and the old copy is kept when a fetch fails (test HTTP client)
  - the cache header
- [ ] The leftover D-DIN `@font-face` rules go (`mixins/_fonts.scss`).

### 7.5 The schedule pages *(business_logic/scheduling + catalog)*
- [ ] **Calendar and upcoming classes:**
  - per-request lookups for facility names, instructors, a class's advance days and each slot's predecessor
  - slots read once per range, not once per day
- [ ] **Class page:** no 120-day derivation for a recurring class; the session count is a `COUNT`.
- [ ] Tests:
  - the query count for each endpoint is below a ceiling set from 7.1's baseline (target ≥ 5× fewer)
  - the answers are identical to before on the seeded schedule (captured before the change and compared after)
- [ ] Record the after numbers next to the baselines.

### 7.6 Requests in parallel *(server_components + knottyyoga + ui)*
- [ ] Verify the comment in `ServerAccessProxy`: does the server still refuse two requests from one session at once? The pool now waits instead of failing (F13). Write a server test that makes two requests from one session at once, with a pool of 2.
- [ ] Fix whatever that finds, including the `/api/remember` rotation race (9.3).
- [ ] The fixed tenant's pool size comes from the app config, default 4 (OQ-30).
- [ ] The browser allows 4 requests in flight instead of 1.
- [ ] Tests: the concurrency test; the proxy spec (4 in flight, the 5th waits).

### 7.7 The page shell and the CDN *(ui/package)*
- [ ] With 7.4, the page comes from the server with `s-maxage=60`, so CloudFront stops checking on every visit. S3's `index.html` is then only the template behind `shell/index.html`, which the server re-reads with `If-None-Match`. To confirm in `deploy_ui.sh` that the deploy still invalidates both.
- [ ] Critical CSS stays inlined (D7). After 7.4, check the served page no longer carries the font CSS twice.
- [ ] After numbers for first paint, next to 7.1's.

### 7.8 Public photos from CloudFront's edge *(server_components: platform; @honuware/ui: access; ui/package; OQ-21)*
- [ ] **Measure first** (7.1): how many photo requests the home page and Gallery make, and how long each takes.
- [ ] **A public photo route:** `GET /api/public_photo/<table>/<id>/<width>/<height>?v=<stamp>`.
  - It serves only the tables `get_scaled_photo` already treats as public (the class, instructor, gallery and carousel photos; to list them from the code). Anything else gets 404. People's own photos stay on `get_scaled_photo`, `private, no-cache` (F20).
  - `v` is the photo's update stamp, so a replaced photo has a new address.
  - It sends `Cache-Control: public, max-age=31536000, immutable`. A stale `v` gets a redirect to the current one.
- [ ] **CloudFront:** a `/api/public_photo/*` behaviour, placed ahead of `/api/*`. It caches on the full URL (query included) and forwards no cookies. Nothing ever needs invalidating, because a new photo has a new URL.
  - Done by Mason in the console, from steps written here when the item is built. Deploying to AWS records the behaviour.
- [ ] `PhotoUrlBuilder` builds public photo URLs with the stamp. The pages that show public photos use it.
- [ ] Tests:
  - the header on each route
  - a private table refused on the public route
  - a stale stamp redirected
  - a replaced photo's URL changes (library spec)
- [ ] **No in-memory cache in the server yet.** Once CloudFront answers repeat requests, the server only sees each photo's first request per edge location. If 7.1's numbers show those first views are slow, a small LRU cache of scaled bytes in the server is the next step (OQ-21).

---

# Phase 8 — Polish and the site's navigation

T19–T21, T25, T26, T28, T38, T40, G2, G4–G6, G8, G9, G11, G12, J1, J2, A5, A7, A10, A11 and P16.

### 8.1 Scrolling, Back, and the Android footer *(ui: app shell)*
- [ ] `h-screen` → `h-dvh` on the shell, so its bottom is the visible bottom on Android. The footer and the checkout bar show without scrolling (T20).
- [ ] The inner scroller gets real scroll restoration. `AppComponent` listens for the router's `Scroll` event:
  - a new page goes to the top, or to its `#fragment` once the content has loaded
  - Back and Forward go to the saved position

  That fixes Get Started (T21, J2) and every "Back lands somewhere odd" (T40).
- [ ] Specs: top on a new page; restored on Back; fragment after a delayed load.
- [ ] A Playwright check at 390 px Android: the footer is fully visible at the end of the home page.

### 8.2 Favicon *(ui + seed)*
- [ ] The icon is **the figure hanging in the red fabric** from the logo (OQ-25, decided). That's the emblem at the left of `assets/svg/KnottyYoga_logo_white.svg`, cropped away from the wordmark.
  - Its figure is white, which disappears on a light browser tab, so the icon puts the emblem on a rounded square in the brand ink.
  - It's made as `favicon.svg`, `favicon.ico` (16 and 32 px) and a 180 px `apple-touch-icon`.
  - Checked by eye at 16 px, where the fabric may need thickening.
- [ ] `site_favicon_url` stays the studio override. A spec checks `index.html` names all three.

### 8.3 Change Password *(ui: update_user_password)*
- [ ] Fields: **Current Password**, **New Password**, **Confirm New Password**. The validator compares the two new ones.
- [ ] A wrong current password stays on the page: "That current password isn't right." This closes Finishing website OQ-43.
- [ ] Specs: the labels, mismatch, the wrong current password, and success.

### 8.4 Navigation and names *(ui: header menu + page titles)*
- [ ] There's no new **Classes** menu, and **Gallery** stays under **About** (OQ-12, OQ-13: there are already enough top-level items). 6.3 fixes the empty My Schedule view behind G9.
- [ ] **Upcoming Events** becomes **Events**, menu and page (G12).
- [ ] **"Our":** menu labels drop it, and each page's title matches the label that opened it (OQ-10, decided):
  - Classes / All Classes, Services, Instructors, Location, Providers
  - "My Classes" and "Our Classes" stop swapping titles
- [ ] **Finding All Classes** (A10, F26). Signed in, **Classes** opens the weekly page, and All Classes is only a button on it. Ashley is the third tester to look for it. The menu stays as you decided (OQ-13), and two links are added:
  - every class page starts with **← All classes**
  - the home page's **Upcoming classes** row adds **All classes →** next to its **Plan your weekly schedule →** link
- [ ] Specs: the back link on a class page, and the home row's link.
- [ ] Specs: the menu for signed out, member, staff and admin, desktop and mobile. The mobile-audit route list gains any new route.

### 8.5 "Show me how" under the headline *(seed + ui: home hero)*
- [ ] The hero section shows its link as a button when it has one. The seed gives it **Show me how** → Getting Started. Spec.

### 8.6 Services name their providers *(endpoints + ui)*
- [ ] Each service card lists who provides it ("Massage with **Caleb Ault**"), linking to the provider's bio. The data is from the providers directory reader, on the service catalog response. Endpoint test; spec.
- [ ] **Book with a provider** (A11). Each card on **Providers**, and each provider's own page, gets **Book with Caleb**.
  - It opens the service booking page showing only that provider's times (`/shop/service/<id>?provider=<personId>`), with **Show every provider** to widen it.
  - Someone who offers more than one service gets a short list of them first.
  - Today the cards have only **View profile** (`providers.component.html:46-50`). The booking page already groups times by provider (`service-booking.component.html:102-106`), but its address can't name one.
  - The providers response gains each provider's bookable services. To verify where that link lives: the provider's types or their shifts.
  - Endpoint test. Specs for the button, the one-service and several-service cases, and the filtered booking page.
- [ ] The name **Services** stays, since more services than massage are coming (OQ-11, decided).

### 8.7 Link colour *(ui: tokens)*
- [ ] A `--theme-link` token (default: the brand's primary). `.back-link`, prose links and the 15 `text-blue-600` templates use it. `design-tokens.spec.ts` covers it.

### 8.8 "Browse services" goes to Services *(ui)*
- [ ] My Bookings (both links), the cart and the service booking page go to **Services**, not Memberships. Jennifer and Ashley both hit it (J1, A5). Specs pin each target.

### 8.9 Leftover "decision history" wording *(ui + seed + migration)*
- [ ] **Site Theme:**
  - "…Type it exactly as you want it to read — it is no longer forced to capitals." becomes "Type it exactly as you want it to read." (and its spec)
  - "Stored for future use — no page renders this today." becomes "Not shown on the site yet."
- [ ] The Manage Data field hints that name a phase or ticket:
  - "(Phase 3 SL-11)", "(skill_levels.id, Phase 3)", "(Phase 4)", "(M-14; …)"
  - the "(M-13)" table description

  They are reworded in the seed only. Production gets them with the reinstall (D8).
- [ ] **Guard tests:**
  - a C++ test that no seeded hint or description matches `Phase \d|OQ-\d|\(M-\d|SL-\d`
  - an Angular spec that no template under `src/app` has visible text matching `no longer|used to|Phase \d|OQ-\d`, outside comments
  - the developer-only style guide is exempt

### 8.10 Bios *(seed)*
- [ ] Mason's and Gifford's bios, rewritten by Mason (OQ-7). They go into the seed (D8), and the seed test reads them back through the instructor reader.

### 8.11 Back and Forward, checked *(e2e)*
- [ ] A Playwright journey spec at 390 and 1280 px. It goes Home → Classes → a class → Calendar → Back ×3, Forward ×3, then the purchase, sharing and Get Started paths. Each Back returns to the previous page, at its scroll position.

### 8.12 Every email links to its settings *(business_logic: mail templates; OQ-28)*
- [ ] Each notification email (the weekly digest, reminders, sharing, class changes) ends with "**Change which emails you get**", linking to **Notifications**. The link is built from the site address.
- [ ] Emails that must always go (verify, password reset, receipts) explain instead: "You're getting this because…".
- [ ] The notification emails share one footer template, so a new email can't leave it out.
- [ ] Tests:
  - every notification template's output contains the link
  - the test lists the templates, so a new one has to be added to it

### 8.13 The cancellation policy, written down and linked *(seed + business_logic/scheduling + business_logic/payment + endpoints + ui; A7, P16; OQ-31)*
F24: no policy exists, so every cancellation refunds in full. The cancel warning names a policy that isn't there. **Decided: OQ-31.** Every policy is data: rows that staff edit, never rules in code.
- [ ] **The policies in the seed** (D8), OQ-31's three, as starting rows only. The seed writes them once at install; `--migrate` never rewrites them, so staff edits stay.
  - **Massage and other services**, attached to the service products.
  - **Workshops and events**, attached to the workshop and event products.
  - **Class bookings**: one window, no refund at any time. Attached to every class drop-in and series product. Description: "Class bookings aren't refunded. If something comes up, contact the studio; staff may give a voucher."
  - Seed tests through `CalculateRefundPercent`, the reader the cancel uses: a massage booked 48, 23 and 1 hours away; a workshop 8 days, 3 days and 1 day away; a class 48 hours away.
- [ ] **The class rule becomes that row** (business_logic/scheduling). Today P-6 is code: `booking_helper.cpp:900-911` skips the refund for any session with a class. That branch goes, and every user cancel reads its product's policy.
  - Two rules stay in code, because they aren't policy: a cancel by the studio refunds in full (`hasFreeCancelOverride`, `:926-927`), and so does leaving a waitlist, where there was never a seat (`:985-986`).
  - The cancel's `refund.reason` stays `non_refundable` whenever the policy gives nothing, for any product, and the response adds the policy's name.
  - Tests:
    - a class drop-in cancelled 48 hours ahead refunds nothing, through the seeded policy
    - the same class with its product switched to **No policy** refunds in full, which proves the rule is data
    - a series booking follows its product's policy the same way
    - a studio cancel of a class still refunds in full
    - the existing P-6 tests attach the Class bookings policy in their setup, so they still pass for the right reason
- [ ] **Cancellation Policies** under **Manage Products** (the **Memberships & sales** group, 9.2): add, rename and edit each policy's description and refund windows. Today only Manage Data can, and that becomes developer-only (10.1).
  - Each policy lists the products that use it.
  - Which policy a product uses is already chosen on the product's page (**Cancellation Policy**, `product-detail.component.html:105`). A new product starts on **No policy (full refund always)**, as today.
  - Specs.
- [ ] **A Cancellation Policy page** (`/cancellation-policy`, signed out too). It is built from the policy rows, so the page and the refund can't disagree:
  - each policy's name, its description, and its windows in words: "Full refund up to 24 hours before. No refund after that."
  - The class rule is simply the Class bookings row; the page has no fixed text about classes.
  - The footer links to it.
  - `GET /api/cancellation_policies` answers signed out. Endpoint test: signed out, with the windows in order.
- [ ] **The booking pages say the product's own policy,** built from its windows, instead of fixed sentences:
  - service booking: "Cancellation policy applies. See your booking confirmation for details." (`service-booking.component.ts:422-426`)
  - event booking (`event-booking.component.ts:177`); event sessions already carry their windows (`event_session_helper.cpp:316-340`)
  - "Series purchases are non-refundable…" on the series booking and class pages (`series-booking.component.html:76`, `class-detail.component.html:151`)
  - The shop's product response gains its policy's name and windows, as event sessions have. Endpoint test.
  - Specs: a product with a policy and one with none, on each page.
- [ ] **A cancel preview:** `GET /api/bookings/<id>/cancel_preview` returns what cancelling now gives back: the amount (2.9's share), the percent, and the policy's name. It writes nothing. Only the booking's owner or payer may call it, as with the cancel; anyone else gets 404.
  - Tests: the preview equals what the cancel then refunds, for a single booking and for one item of a cart; 401; 404 for someone else's booking.
- [ ] **The warning on My Bookings,** and on Purchase History's cancel (`purchase_history.component.html:99`), gives the number and links the policy:

  > Cancel now and you get **$160** back (full refund until 24 hours before). **Cancellation policy**

  The generic service text goes (`my-events.component.ts:235`), and so do the class-only sentences (`:248-253` before the cancel, `:358-361` after it). A class says "No refund under the **Class bookings** policy", with the same link. Events use the preview too, so a cart's event shows its own share.
- [ ] Specs: the warning for a full, a partial and no refund, including a class; the message after a cancel that refunded nothing; the link; the page.

---

# Phase 9 — Manage Products for running the business

T29, T32–T37, G19, and P6–P8.

### 9.1 Business overview *(table helpers + business_logic/reporting + endpoint + ui)*
- [ ] **`TableHelpers::BusinessMetrics`**, aggregate SQL only:
  - active memberships by product
  - monthly recurring revenue (the sum of current memberships' prices)
  - next month's forecast (memberships due to bill, less cancellations taking effect before then)
  - revenue by month (completed payments less refunds)
  - class seats planned vs capacity, from the derived schedule and the weekly plans
- [ ] **`Reporting::BusinessOverviewHelper`** builds the overview (OQ-17):
  - **Members:** total, by tier, and the change on last month
  - **Revenue:** last month, this month so far, next month's forecast
  - **Classes:** utilization, the last 4 weeks from check-ins and the next 4 weeks from plans and bookings
  - the five slots most often under 40% full, and the five most often full
- [ ] `GET /api/admin/business_overview` (`manage_products`).
- [ ] Tests through the helper, on a seeded set of subscriptions, payments, refunds and check-ins with a fixed clock: each number, each cut-off and each edge (a refund in the next month, a cancellation effective mid-month).
- [ ] **Business Overview** at the top of Manage Products: a stat row, revenue by month (12 months), and two class lists. Specs.

### 9.2 Grouped tiles *(ui: manage dashboard)*
- [ ] The tiles in headed groups (OQ-16, decided):

  | Group | Tiles |
  |---|---|
  | **Classes & scheduling** | Class Schedules, Schedule Grid, Close Classes, Scheduling Exceptions, Attendance Templates, Skills, Class Tags, Locations, Room Schedules |
  | **Staff** | Instructors, Who's Teaching What, Shift Requests, Time Off Requests, Service Providers, Roles |
  | **Memberships & sales** | Products, Pricing Overview, Price Schedules, Membership Tiers, Subscriptions, Entitlements, Event Sessions, Bundles, New Workshop, Cancellation Policies (8.13) |
  | **Discounts & credits** | Vouchers, Coupons, Comps, Refund Effectiveness |
  | **Reports** | Enrollment Trends, Open-Seat Heatmap, Room Occupancy, Subscription Revenue |
  | **Website** | Site Theme, Fonts, Page Content, Studio Settings (2.8) |

- [ ] Each tile is shown only to people who can use it. Today only **Roles** is filtered (P7).
- [ ] Specs: the groups and their order, and the per-permission tile sets. `manage-dashboard.component.spec.ts`'s flat-list tests are rewritten.

### 9.3 The login prompt *(@honuware/ui: auth; server_components: platform/auth)*
- [ ] **At start-up:** an error that isn't a 401 from `/api/me` or `/api/remember` (5xx, network, 429) is retried once. The person isn't treated as signed out on its account.
- [ ] **Mid-visit:** a 401 tries `/api/remember` once and repeats the request. Only if that fails does it clear the sign-in and go to Log In **with `returnUrl`**.
- [ ] **Two tabs:** a device token stays valid for 30 seconds after it is rotated, so the slower tab isn't signed out. The server records the previous token's hash and its expiry: two columns on `device_tokens`, framework migration **`honuware/0004_device_token_grace`**.
- [ ] Tests:
  - library specs for each path
  - server tests: the old token works for 30 s and is refused after; a reused token outside the window still ends every session, as today
- [ ] Ship as one server_components change and one `@honuware/ui` release.

### 9.4 Attendance plans: search people, not plans *(table helpers + endpoint + ui)*
- [ ] The search starts from `people`, left-joined to plans. It matches a full name ("Bri Smith") as well as each field, and pages with the shared pager instead of stopping at 25.
- [ ] A person without a plan shows **No plan yet**. Opening them shows an empty plan the admin can fill.
- [ ] The tile follows its endpoint's permission, `manage_class_schedule` (P7).
- [ ] Tests: someone with no plan is found; full-name search; paging; the permission.

### 9.5 New Workshop *(business_logic/scheduling + endpoint + ui)*
- [ ] `WorkshopHelper::CreateOneOffWorkshop`, in one transaction, makes:
  - an event product (name, description, capacity)
  - its price on the default schedule
  - one event session (date, start, duration, room)
  - the instructor's staffing. **The teacher must have an account and be an instructor** (OQ-18, decided). A visiting artist is set up under **Manage Products** ▸ **Instructors** first.
- [ ] To verify: **Instructors** can add someone who has no account yet. If it can't, it gets **Add an instructor** (name, email), which creates the account and sends the set-password email. Tests.

  Tests read the result back through the Events reader and the catalog.
- [ ] `POST /api/admin/workshops` (`manage_products`). Endpoint tests.
- [ ] **New Workshop**, a one-page form, shown on the dashboard and on Event Sessions. Its fields:
  - Title, Description
  - Date, Start time, Length
  - Room
  - Instructor, with "Visiting teacher? Add them under **Instructors** first."
  - Capacity, Price
  - Photo

  Saving opens the workshop's public page. Specs.

### 9.6 Pricing Overview says what it is *(ui)*
- [ ] An intro line: "Each column is a price for people holding that membership or role. **Standard** is what everyone else pays, and what a member pays when their column is blank."
- [ ] **Standard** becomes **Standard (everyone)**. Permission columns use display names.
- [ ] **Bug (P6):** only pricing-eligible permissions become columns, as Membership Tiers promises.
- [ ] Specs: the text, the names, and an ineligible permission's absence.

### 9.7 Products grouped *(OQ-19: nothing to build)*
- Gifford's "categorize the products" most likely meant Manage Products' grid of tiles, the same point as Thomas's, and 9.2 groups those.
- The **Products** list itself has eight seeded products, so it stays as it is. If it grows, sectioning it by each product's existing kind, plus a search box, is a small change.

### 9.8 Reports that look forward *(business_logic/reporting + ui)*
- [ ] **Enrollment Trends** and **Open-Seat Heatmap** get **Last 12 weeks** / **Next 4 weeks**:
  - back: check-ins
  - forward: plans and bookings over the derived schedule
  - both count recurring classes, not only materialized sessions
- [ ] **Who's Teaching What** counts recurring classes from the derived schedule (P8).
- [ ] **Subscription Revenue:** totals by month, and counts labelled as counts (P8).
- [ ] Each report links to the others in a **Reports** row, so they're found together (T37).
- [ ] Tests through each helper on a fixed clock; specs.

---

# Phase 10 — Manage Data becomes a developer's tool, and a red-team pass *(mostly server_components)*

T30, T31, T41 and G18.

**Decided (OQ-20):** Manage Data is a developer's tool, so it gets its own permission and a warning rather than a friendlier editor. v0.1's help text, foreign-key previews, table list and history (its 10.1–10.4) are deferred to 10.3. A bad edit can still be undone, coarsely, from RDS's seven days of point-in-time backups (Deploying to AWS).

### 10.1 Its own permission *(server_components: platform; @honuware/ui: crud, auth; knottyyoga seed)*
- [ ] A framework permission **`manage_data`**, given through a framework **Developer** role. Framework migration **`honuware/0005_manage_data_permission`** adds both, and gives the role to every current admin so CommunityFinder's admins keep access. It is idempotent.
- [ ] The Manage Data page (`/admin`), its menu item and `get_db_schema` require `manage_data`, not just admin.
  - The generic endpoints keep Phase 1's rule (a table grant), because about twenty Manage Products pages use them (1.1).
- [ ] The knottyyoga seed gives **Developer** to Mason's account only. Testers made admins for the beta no longer see Manage Data.
- [ ] A banner on the page: "Developer tool: this edits the raw data, with no undo. Day-to-day work belongs in Manage Products."
- [ ] Tests:
  - an admin without the role gets 403 on `get_db_schema` and has no menu item; with the role, both work
  - the migration grants the role to existing admins, and running it twice changes nothing
  - library specs for the guard and the banner

### 10.2 Red-team pass *(server_components + knottyyoga)*
- [ ] **Route census test:** walk every registered route. Signed out, each one must answer 401 unless it is on a reviewed public list. The list lives in the test with a reason per entry, so a new endpoint is private until someone says otherwise.
  - Crow's router exposes its rules; to verify. If it doesn't, a test-time list is checked against the anchors in `web_app.cpp`.
- [ ] **Ownership tests** for every route that takes another person's id or a row id (subscriptions, purchases, bookings, gift permissions, entitlements, templates): a second member gets 404.
- [ ] **An adversarial review by Claude** against a local server:
  - every route, signed out, as a member, as `manage_products` and as an admin
  - CSRF, IDOR, mass assignment through `update_item` (writing `person_id` or `status`), rate limits, and what `get_db_schema` reveals

  Findings are recorded here and each one is fixed with a test.
- [ ] **Carried from Phase 1** (1.1 and 1.3), each with a test first:
  - `manage_products`' grant on `permissions` lets its holder edit any permission row, `admin_portal` included
  - a wrong activation token's attempt count and admin alert are rolled back with the error, so the limit may never trip
  - `get_scaled_photo` stores a new copy for every size a signed-out caller asks for
- [ ] **Carried from Phase 2** (found while porting its tests), each with a test first:
  - a body field of the wrong JSON type (`"product_id": "abc"`) answers 500, not 400. Every endpoint that reads `Json::Value::Get<T>()` straight off the body does this, the two subscribe endpoints among them.
  - a refused signup (declined card, unknown product) answers a bare 400 with the message and no `code`, so a page can only show the text

### 10.3 Deferred (OQ-20)
- Help text: a description under each table and field, written for every table and column.
- Foreign keys show the record they point at, with **Open**.
- A left-hand list of tables instead of the dropdown (G18).
- History and restore, sized in OQ-20:
  - an `admin_change_log` that the four generic writes fill in the same transaction
  - a **History** list on each row
  - **Restore**

---

# Phase 11 — Search engines and the first page *(T44)*

### 11.1 The basics *(ui + server)*
- [ ] Each public page has its own `<title>` and `<meta name="description">`, plus Open Graph tags for links shared in messages.
  - **They must be in the served HTML** for link previews and most crawlers, which don't run JavaScript. 7.4's page route puts them there, through the app hook for the class, instructor and event pages and the site's defaults elsewhere.
  - Angular sets the same tags as the visitor moves between pages, for Google and for the browser tab.
- [ ] `robots.txt` and a `sitemap.xml`:
  - the public pages, every class and every instructor
  - generated by a server endpoint that CloudFront serves at `/sitemap.xml`
- [ ] Tests: each route's title in the browser; each page's tags in the served HTML (with 7.4's tests); the sitemap lists the seeded classes and instructors.

### 11.2 Server rendering: later, in its own plan *(OQ-21, decided)*
- A server-rendered signed-out home page means a Node runtime next to the C++ server, or a pre-render step. The page's content comes from the database at run time, so a build-time pre-render goes stale.
- **Decided:** 11.1 now. A separate plan, **Server-rendered public pages.md**, gets written if Mason wants more after Phase 7's speed work and 11.1.
- Google and Bing run JavaScript. Most other crawlers, and link previews, read only the HTML. 11.1's tags reach them because 7.4 puts the tags in the HTML; the page's body text still doesn't.
- Public photos are cached at CloudFront's edge in 7.8, which answers the image-cache half of OQ-21.

---

# Phase 12 — Verification

### 12.1 Build and test (Claude, both platforms, reported per platform)
- [ ] server_components: Windows, Linux.
- [ ] knottyyoga: Windows, Linux, pinned; same test names on both.
- [ ] CommunityFinder at the new pin: Windows, Linux.
- [ ] Angular: Windows, Linux (`ng test` as CI runs it); `ng build`; lint.
- [ ] Playwright: layout specs at all sizes (including 6.6's attendance control), the mobile audit with the hand-started :4300 server, 2.6's double-pay spec, 8.11's Back/Forward spec.
- [ ] `@honuware/ui`: specs, lint, build.

### 12.2 End to end
- [ ] A throwaway database with `--install_schema`, then the scripted checks:
  - signed-out writes refused
  - a second membership refused without the tick, allowed with it
  - a declined card leaves nothing
  - an invitation with a link registers and takes the seat
  - that seat is still there after the billing run renews the membership (P14)
  - `site_info` with a cache hit and a 304
  - a public photo with its long-lived cache header
  - a page address served with the site's font links and a class page's own tags (7.4)
  - each start choice's charge and dates, with the billing run driven forward to the 1st (2.3)
  - one booking cancelled from a two-item cart refunds only its share, and matches the cancel preview (2.9, 8.13)
- [ ] A throwaway database with `--install_schema --force` followed by the `set_secret` block, exactly as Phase 13 runs it.
- [ ] A copy of a sandbox.15-era database with `--migrate`: app migrations 0014–0016 and honuware 0004–0005. A second run changes nothing. Production is reinstalled, but Mason's local databases and CommunityFinder migrate.

### 12.3 Hand-testing walkthrough
- [ ] One walkthrough across all phases, from a blank database, with exact menu and field labels, no URLs. Each phase's own walkthrough is folded in.

---

# Phase 13 — Release to production

- [ ] Phase 1 and Phase 2 are already out (sandbox.14 and .15, both by `--migrate`).
- [ ] Before: Mason tells the testers that their accounts will be removed and they'll need to register again.
- [ ] The rest goes out as **`v1.0.0-sandbox.16`**, and production is **reinstalled from the seed** (OQ-6, D8). That means `--install_schema --force` followed by the `set_secret` block, as Finishing website did (Deploying to AWS 5.1 step 4). All content comes from the seed, with no hand edits.
- [ ] **CloudFront, in this order:**
  1. 7.8's photo behaviour, any time before the UI goes out (nothing asks for those addresses until then)
  2. the server, then the UI (which uploads `shell/index.html`)
  3. 7.4's change, sending page addresses to the server, only once the server answers them

  To undo step 3, point the default behaviour back at S3 and restore the error-page fallback.
- [ ] Afterwards: Claude probes production signed out, and the testers are asked to retry their journeys. A short list for Thomas, Gifford, Jennifer and Ashley of what changed against each of their points.

---

# Cross-references

- [[Finishing website]]: the plan this follows; its OQ-43 (a failed password change) is closed by 8.3. D13 there (what a password reset ends) is why D6 keeps server sessions.
- [[Subscriptions- Recurring billing and card management]] §6.4 "Subscription Upgrades": the server half of Phase 3, built without a member-facing page. Proration was dropped there; OQ-2 asks again.
- [[Customer user scenarios]]: scenarios 12 (multi-seat subscription), 31–33 (upgrade, downgrade, reactivate) are what Phases 3–4 make reachable.
- [[Security Review]]: Phase 1 and 10.2 extend it; the CSRF design (8.x there) is kept, with `delete_item` moved off GET.
- [[Deploying to AWS]]: the reinstall Phase 13 uses (5.1 step 4), the CloudFront behaviours that 7.4, 7.7 and 7.8 change, and the RDS point-in-time backups that stand in for Manage Data's rollback (OQ-20).
- [[Tenant Theming and Branding]]: `site_info` and the runtime fonts that 7.2–7.4 change.
- [[Componentizing the frontend]] and [[Splitting the server up into components]]: where the honuware halves of 1.2, 4.5, 7.2, 9.3 and Phase 10 live.

---

# Open questions

*Mason's answers are kept as written. "→ Plan" says what changed; "→ Explained" answers a question you asked back. Anything still open has a default the plan is written against.*

### Still open

**OQ-7 — Mason's and Gifford's bios** *(8.10)*
- Thomas: lead with your own teaching and training, not Ana Forrest's name; Beth's is the model. He says Gifford's undersells him too.
- **Default:** wait for your text, and Gifford's. Claude doesn't invent experience.
- Paste them here when ready.
- → Plan: still waiting. They're seed text (8.10), so they can arrive any time before Phase 13's reinstall.
- → v0.3: still waiting, as you said. Nothing else depends on them.

**OQ-32 — Manage Products for someone who isn't an admin** *(1.1)*
- Building Phase 1 turned up about fifteen tables that Manage Products pages save but no permission grants. Among them are booking windows, instructors, the home page and About sections, and provider availability. 1.1 has the list.
- They work for admins, and today only admins use Manage Products. A non-admin given `manage_products`, such as a future Studio Manager, would see those pages refuse to save. Phase 1 didn't cause this, and it fixed the one page it would have broken (Class Schedules).
- Class Schedules is itself only half usable on `manage_products` alone. It lists a class's instances, schedules and slots through endpoints that require `manage_class_schedule` (`admin_class_instances_list.cpp:69`), so it shows "Failed to load class instances". The class and its requirements do save. But the Requirements card reads them through `GET /api/admin/class/<id>/requirements`, which also requires `manage_class_schedule` (`admin_class_requirements_list.cpp:64`). So the card shows "Failed to load requirements", and a group added there saves without ever appearing. Mason saw both banners on 11/19 while hand-testing Part C. The seeded Studio Manager role holds `manage_class_schedule` and not `manage_products`, so it never reaches the page through the Admin menu.
- **Default:** leave it until someone who isn't an admin needs Manage Products. Then grant every page's tables in one pass, with a test per page that signs in with only that permission and saves.
- The other way is to do that pass now, as part of Phase 9, which reworks Manage Products anyway (about half a day).
- Do you expect to give anyone `manage_products` without making them an admin before the final release?

*OQ-33 to OQ-47 came out of building Phase 2 (v0.7). Each one is built to its default.*

**OQ-33 — Members can't start in a later month** *(2.3)*
- A member's **Subscribe** page offers **Start today** and **Start on December 1**. Later months are staff-only, on **Create Subscription**.
- **Default:** as built, since you described two choices (OQ-27).
- Should members keep a later-month choice?

**OQ-34 — Staff lost the unticked "Prorate for remaining days"** *(2.3)*
- Before, unticking it charged the full price for a part month. **Start today** now always charges by the day. The nearest choice is **rest of this month free**, which charges a full month and runs to the end of next month.
- **Default:** gone.
- Do staff ever charge the full price for a part month?

**OQ-35 — Months are UTC months** *(2.3)*
- The planner, and the billing run before it, count months in UTC. In Pacific time, a month starts at 4 PM (5 PM in winter) on the last day of the month before. Bought at 6 PM on November 30, **Start today** charges December's full price. A membership "starting December 1" opens at 4 PM on November 30.
- The pages show their dates in UTC so they read "December 1", not "November 30".
- **Default:** leave it until the studio's time zone is a setting. The cost is a few evening hours of free access at each month's turn.
- Is that acceptable for now?

**OQ-36 — Renewals still send two emails** *(2.4)*
- 2.4 made a signup send one email. A monthly renewal still sends the payment confirmation and the renewal notice.
- **Default:** leave it for Phase 3, which reworks the membership emails anyway.
- Fold the renewal into one email too, now or with Phase 3?

**OQ-37 — A membership without a card can only start today** *(2.1, 2.3)*
- Staff can still create a membership with no card (cash, a comp), but only with **Start today**. Without a card, the other three choices are refused, since nothing could charge it on the 1st.
- **Default:** as built.
- Do staff set up cash-paid memberships that start later?

**OQ-38 — Where "the same order twice" is checked** *(2.5)*
- The plan named `purchase_pay_card` and `cart_checkout`. The check is in `purchase_pay_card` only. The cart, checkout and the three booking pages all pay through it, so it covers every card payment.
- Orders paid entirely with store credit or a voucher aren't checked. Memberships use 2.2's warning instead.
- **Default:** as built.
- Should an order paid wholly with store credit ask too?

**OQ-39 — Two separate bookings of the same service look like the same order** *(2.5)*
- The check compares products, variants and quantities, not the booked times. Booking a 60-minute massage for Tuesday, then another for Thursday ten minutes later, asks "Buy it again?". **Buy again** books it.
- **Default:** keep it. It costs one extra click in a rare case.
- The other way is to compare the booked times too (about an hour). Which do you prefer?

**OQ-40 — Staff payments skip the check** *(2.5)*
- Check-in walk-ins and the provider's upgrade charge send `confirm_duplicate: true`. At the desk, two charges in a row are usually two people, and a dialog would get in the way.
- **Default:** staff are never asked.
- Should staff be asked too?

**OQ-41 — Changing a membership waits for Phase 3** *(2.2, 3.1)*
- Porting Phase 2's tests turned up four things in the change-membership path. Phase 2 doesn't touch that path, and none of the four is verified yet:
  - an upgrade credits the old tier's full price, whatever was paid. A prorated first month is over-credited.
  - a failed upgrade charge can shorten the member's access
  - the first month after a downgrade may never be billed
  - after a downgrade, the membership summary (2.2) lists the old and the new membership until the old one ends. So the D2 warning shows "Gold and Silver".
- **Default:** 3.1 verifies each with a test first, then fixes it.
- OK to leave them until Phase 3?

**OQ-42 — Cancelling a membership that hasn't started** *(2.3)*
- A **Start on December 1** membership cancelled before the 1st ends at once, and nothing is ever charged. It has no paid time to run out, so it doesn't read "ends December 1".
- **Default:** as built.
- Say if it should stay listed until the 1st.

**OQ-43 — Prices show cents** *(2.3, 2.6)*
- The plan's examples say "$160". The pages use the app's usual money format, so they show "$160.00" and "$74.67".
- **Default:** cents everywhere, matching the rest of the site.
- Should whole dollars drop the cents?

**OQ-44 — A cart paid partly by store credit, partly by card** *(2.9)*
- The plan wanted a cancel to refund the store credit first, as credit, and then the card. `RefundBookingShare` refunds the booking's share against the purchase's first payment only. The total is still capped at what was paid, so it can never refund too much.
- **Default:** leave it. The testers' carts were all paid by card. 8.13 reworks refunds next.
- Fix it now, or with 8.13?

**OQ-45 — Bookings from before carts named their line** *(2.9)*
- A booking made before bookings recorded their cart line shares the whole purchase with the other bookings on it, evenly. The Phase 13 reinstall removes every such booking.
- **Default:** as built.
- Nothing to decide unless you want those old bookings fixed by hand.

**OQ-46 — The two-at-once signup test** *(2.2)*
- The plan asked for a test with two simultaneous first signups. Each test runs inside one transaction that's rolled back at the end, on one connection, so a second connection can't see its rows.
- The test checks the lock instead: during a signup, the person's row is held `FOR UPDATE`. A true two-connection test needs committed rows in a scratch database (about half a day).
- **Default:** write it with 7.6, which raises the connection count and makes the race possible.
- OK to wait for 7.6?

**OQ-47 — Cancel in "Buy it again?" leaves the new order pending** *(2.6)*
- On checkout and the cart, **Cancel** asks the server to cancel the new, unpaid order. There's no such endpoint: `cancelPurchase` in the UI calls `/api/purchase_cancel`, which the server has never had. The order stays **Pending** in **My Purchases**, with nothing charged.
- On the booking pages, **Cancel** leaves the booking unpaid, as a declined card does, and says "Nothing was charged".
- A cart that holds a booking is worse off. `cart_checkout` writes its bookings as confirmed, and its service sessions as scheduled, before anything is paid. `PurchaseHelper::CancelPurchase` only changes the order's status. So even with a cancel route, a cancelled cart would keep its time slots, and an unpaid one keeps them today. Opening **Shopping Cart** again without the order in the URL (after a decline, say) makes a second unpaid order with its own bookings.
- **Default:** leave it until 8.13, which reworks cancels and refunds. A pending order charges nothing, and the Phase 13 reinstall clears the testers'.
- The other way is now: a member-side cancel for your own unpaid order that also releases its bookings and sessions, and a cart that finds your unpaid order before making another (about a day, with tests). Now, or with 8.13?

### Resolved 11/17/2026, fourth round

*Your answer to v0.4.*

**OQ-31 — The cancellation policy** *(8.13, Ashley)*
- Ashley asked for a link to the cancellation policy from the cancel warning. There's nothing to link to yet:
  - No policy exists, so every cancellation refunds in full, even an hour before a massage (F24).
  - The warning says "Cancellation policy applies" anyway.
- **What a policy is in the data:** a name, a description, and refund windows such as "100% if cancelled 24 or more hours before, 0% after". Each product names its policy. Class bookings are already non-refundable (P-6), and a cancellation by the studio always refunds in full.
- **Default:** my proposal below. These are your business terms, so edit them freely.
  - **Massage and other services:** a full refund up to 24 hours before, and none after.
    - Description: "Your provider sets this time aside for you. Cancel at least 24 hours ahead for a full refund."
  - **Workshops and events:** a full refund up to 7 days before, half up to 48 hours before, and none after.
    - Description: "Workshops are planned around who's coming. Cancel a week ahead for a full refund."
  - The refund goes back to the card, or to store credit if the person ticks it, as today.
- **Say "use the default", or write your windows and descriptions here.** Nothing ships with a policy you haven't seen.
- Mason- These are just the initial things populated in the data, correct? It isn't hardcoded and I can go into product definitions for any of these and alter them in the products page, correct? I don't want hardcoded policy.
- → Explained: yes, with one exception in today's code, which the plan now removes.
  - **The policies are database rows.** A policy is a `cancellation_policies` row (name, description) with `cancellation_policy_windows` rows (hours before, refund percent). The cancel reads them when it happens (`refund_helper.cpp:51-85`). The seed writes the starting rows once at install (D8), and `--migrate` never rewrites them, so your edits stay.
  - **Which policy a product uses:** you can change it today, on the product's page in **Manage Products**, in the **Cancellation Policy** dropdown (`product-detail.component.html:105`). Its first choice is "No policy (full refund always)".
  - **A policy's own windows and wording:** today only Manage Data can edit them. 8.13 adds a **Cancellation Policies** page under Manage Products.
  - **A change applies to the next cancellation,** including bookings made before you changed it, because the cancel reads the windows at that moment.
  - **The exception: class bookings.** Their no-refund rule (P-6) is code, not data: `booking_helper.cpp:900-911` skips the refund for any session that belongs to a class, whatever its product's policy says. The pages also carry fixed sentences about it:
    - "Class bookings are non-refundable…" (`my-events.component.ts:248-253`, `:358-361`)
    - "Series purchases are non-refundable…" (`series-booking.component.html:76`, `class-detail.component.html:151`)
    - "Cancellation policy applies…" on the service booking page (`service-booking.component.ts:422-426`)
  - **Two rules stay in code, because they aren't policy:** a cancellation by the studio refunds in full, and so does leaving a waitlist, since there was never a seat.
- → Plan: 8.13 now makes every policy data.
  - The seed writes the two defaults above, plus a third, **Class bookings**: no refund at any time, with a description that points to the studio and its vouchers.
  - The P-6 branch is deleted. Classes refund nothing because their products use that row. Switch a class product to another policy and it refunds by that one; a test proves it.
  - Every page that describes a policy builds its text from the rows.
  - To start with different numbers, write them here before Phase 13's reinstall. After that, change them on the **Cancellation Policies** page.
  - Mason- Okay, this plan sounds good to me then.
- → Plan: confirmed. 8.13 is marked decided, and the seed writes the three policies above.

### Resolved 11/17/2026, third round

*Your answer to v0.3.*

**OQ-22 — One font address** *(7.4, D7)*
- The fonts are fetched twice: the fixed weights in `index.html`, then the seeded variable weights (`wght@100..900`) after start-up.
- **Default (v0.3):** option D below: the server fills in the page head from the database. v0.2's default was A.
- **Alternative:** drop the `index.html` fonts and rely on the runtime ones from the cached site config (7.3).
- Mason- What are the pros and cons of the two options?
- → Explained: the site's fonts (Roboto and Barlow) come from Google Fonts.
  - `index.html` links one Google address in its head.
  - After the app starts, the site config links a *different* address for the same fonts. That config comes from the database, so a studio can change fonts in **Site Theme**.
  - The browser downloads both, which is Thomas's "fonts start late".
  - **Option A (default): `index.html` uses the same address as the database.**
    - Pro: the download starts in the first few milliseconds, before any JavaScript, so text draws in the right font sooner and doesn't jump. The second link is then a cache hit, not a download.
    - Con: the address is written into `index.html` when the site is built. If you change fonts in **Site Theme**, visitors download the old fonts first (wasted, not broken) until the next deploy updates `index.html`. A test catches the seed and `index.html` drifting apart.
  - **Option B: drop the fonts from `index.html` and load only the database's.**
    - Pro: one source of truth, and a font change in **Site Theme** is the only download.
    - Con: on a first visit the fonts start only after the app has booted and fetched the site config. 7.3 caches the config, so repeat visits are quicker, but they still wait for the JavaScript. Text appears in a fallback font and then swaps: the flash Thomas noticed.
  - **Recommendation: A.** Fonts change rarely, and first paint happens on every visit.
  - Mason- I'm on the fence about this. I want this to be a generic server and want this to be configurable. Is there a way to let users change these fonts in the database but have something get updated so that first paint is fast? Maybe index.html could be generated based on something that gets updated by the timer process?
- → Explained (v0.3): yes. The timer can do it, but there's a better way that needs no timer.
  - **Why it's hard today.** `index.html` is a static file on S3, written by the UI deploy. CloudFront sends that same file to every visitor of every site. The fonts are in the database, so only code that reads the database can put them in the page head: either when the page is requested, or by rewriting the file whenever they change.
  - **Option C: the timer process rewrites `index.html` on S3** when the fonts change.
    - Pro: the page stays a static file, so first paint is as fast as A.
    - Con: the server box would need permission to write the UI bucket and invalidate CloudFront, as CI's deploy key does today.
    - Con: every UI deploy overwrites the file with the build's version, so the timer has to notice and redo it. Until it does, visitors get the build's fonts.
    - Con: there's one `index.html` for the whole distribution, so two studios on one deployment can't each have their own.
  - **Option D (new default): the C++ server fills in the page head when a page is requested,** and caches it (7.4).
    - The UI build's `index.html` becomes a template with one marked block in its head. The server keeps a copy, refreshed within a minute of a UI deploy.
    - For each page address, it fills in the site's font links, title, description and theme colour from the database. Its cache lasts 60 seconds and is cleared when Site Theme saves. With CloudFront's own minute (next point), a font change shows within a minute, with no deploy and no timer.
    - CloudFront keeps each filled-in page for 60 seconds too, so most visits never reach the server. If the server is down, CloudFront keeps serving the last good copy for a day.
    - Pros:
      - fully configurable from the database
      - per site, which fits honuware's many-studios model
      - first paint about as fast as A, since CloudFront caches the page
      - **it also fixes link previews.** The same block carries each page's own title, description and preview image, which Slack, Messages and Facebook read from the HTML. That's the correction to 11.1 (see the v0.3 revision notes).
    - Cons:
      - it's the largest item in Phase 7, about twice the size of 7.8
      - every page address now goes through the server, though cached, instead of straight from S3
      - a CloudFront change, made in a set order at release (Phase 13)
  - **A middle way, if D is more than you want now:** A, plus a few lines of script in `index.html` that remember the last font address in the browser and apply it before anything else loads. Repeat visitors then get a changed font at first paint. Only a first visit after a change sees the swap. Link previews stay generic.
  - **Recommendation: D.** It's the only option that is generic, per site and fast all at once, and it does 11.1's job too. The plan is written against it (7.4, 7.7, 11.1, Phase 13), and option A stays in 7.4 as a one-item fallback.
  - **Say D, A, or the middle way.**
  - Mason- Let's go with D.
- → Plan: 7.4 as written, now marked decided. Option A's fallback lines are gone from 7.4, 7.7 and 11.1. 7.4 keeps its order of release with the CloudFront change (Phase 13), and its own undo: point the default behaviour back at S3.

### Resolved 11/17/2026, second round

*Your second round, in number order.*

**OQ-9 — New copy you need to write or approve** *(5.2, 5.3, 6.4)*
- The new-student FAQ: physical requirements, what to wear and not wear, what to bring.
- Mason- Knotty Yoga is physically rigorous and students should be passably fit, free of substantial injury, and excessive extra weight will make classes challenging and hard to keep up with the group. Contact us for suggestions of one on one training if you do not meet this criteria. You do not need a yoga mat as we have padded, nice traction floors. Please wear fitted gym clothes that are not baggy to avoid getting caught in equipment. Generally tights, leggings, or sweat pants are good (preferably grippy cotton) and a fitted tshirt or tank top. Guys wear tight underwear that keeps things snug. No watches, metal, zippers, or jewelry. Water bottles strongly encouraged.
- Longer Silver and Gold descriptions.
- Mason-
	- Silver- This is our introductory option that features a selection of our more beginner friendly and accessible options. I would start here until instructors suggest you move on to gold and you feel ready. Part of doing acrobatics is learning terminology, body positions, and learning to move and organize yourself in a new way. These classes are foundational for things like aerial, partner acro, and handstand training.
	- Gold- These are more advanced classes that also feature elements from circus acrobatics. This tier also includes some specialty classes for handstands, aerial, and partner acrobatics.
- A line on each beginner class saying it prepares you for the others.
- **Default:** Claude drafts each from your class descriptions and the About page, marked **DRAFT** here for you to edit. Nothing ships until you've approved it.
- → Plan:
  - **The FAQ** is in 5.3 as a DRAFT under three headings: *Is Knotty Yoga right for me?*, *What to wear* and *What to bring*.
    - Every point you made is kept. The changes are punctuation, "Men:" for "Guys", and one softer phrase for a page strangers read first: "carrying a lot of extra weight makes classes hard to keep up with", for "excessive extra weight will make classes challenging".
    - **Say "approved", or "use mine as written".**
  - **Silver and Gold** are in 5.2, with light punctuation edits.
  - **"Prepares you for" in the descriptions:** no new copy is needed. The class page's **Leads to** line, built from your OQ-23 pairs, says it on every class that prepares for another. Beginner Partner Acro's description already says it leads to Intermediate. Say if you still want a sentence in the descriptions.
  - Mason- Approved
- → Plan: 5.2 and 5.3 use the text as written here, without the DRAFT mark. No sentence is added to the class descriptions; the **Leads to** line does that job (6.4).

**OQ-14 — See the subscribe page before signing in** *(5.2, Gifford)*
- **Default:** yes. A visitor sees the tier, price, start month and first charge, and signs in or registers at **Subscribe**, coming back to the same page.
- **Alternative:** keep sign-in first.
- Mason- I don't fully understand this item. Can you explain it more fully and what I am choosing between?
- → Explained:
  - **Today:** on **Memberships**, a visitor who isn't signed in presses **Subscribe** on a card and lands on **Log In**, because the subscribe page requires sign-in (`shop.routes.ts:20`). Someone with no account then registers. Registration loses where they were going (F11), so they end up on the home page with no membership.
  - **What they never saw:** the subscribe page is where the money details are. It shows the start month you pick, what you pay today (part of this month), and the monthly charge after that.
  - **Gifford's point:** show those details *before* asking someone to make an account, the way an airline shows the fare before you log in.
  - **Option A (default): show the page first.**
    - A visitor sees the tier, the start month, today's charge and the monthly charge.
    - Where the card form would be, they see **Sign in or create an account to continue**. After either, they come back to the same page with their choices kept.
    - It's small: the route's sign-in guard comes off, the page hides the card form when signed out, and its price preview must answer signed out (to verify).
  - **Option B: keep sign-in first.** Registration would still return to the subscribe page, because 4.5 fixes that either way.
  - **Recommendation: A.** Every screen before the price is a place where people leave (Thomas's funnel point).
  - Mason- Let's go with A
- → Plan: 5.2, as option A describes. The amounts on the signed-out page come from 2.3's new `start_options`, which answers signed out, so v0.2's "to verify" is settled.

**OQ-15 — One / Two / Family** *(5.2, Thomas)*
- **Default:** a toggle over the cards, starting on **One**, with "Couples save $X a person each month" on **Two** and **Family**.
- **Alternative:** keep all five cards on show, adding only the saving line.
- So there would just be a gold and silver option but have a Single, Couples, and Family options as what? Radio controls or separate buttons for each option that list the savings?
- → Explained: yes, two tiers (Silver and Gold) with the sizes as a choice. There are five membership products today, shown as five cards side by side:
  - Silver $160
  - Silver Couple's $240
  - Gold $200
  - Gold Couple's $300
  - Gold Family $400 (three people)

  Thomas's idea is to ask "how many people?" so the page shows fewer choices and can say out loud that the bigger sizes save money. There are three ways to lay it out.
  - **A (default): radio buttons inside each card.** Silver's card lists its two sizes and Gold's lists three, each with its price and the saving per person. You pick one and press **Subscribe**. Every price is visible at once, and nothing changes elsewhere on the page when you pick.

    ```
    ┌ Silver ───────────────────────┐  ┌ Gold ─────────────────────────┐
    │ Beginner-friendly classes     │  │ Every class on the schedule   │
    │                               │  │                               │
    │ ● Just me      $160/month     │  │ ● Just me      $200/month     │
    │ ○ Couple       $240/month     │  │ ○ Couple       $300/month     │
    │     $120 each · save $40 each │  │     $150 each · save $50 each │
    │                               │  │ ○ Family (3)   $400/month     │
    │                               │  │     $133 each · save $67 each │
    │                               │  │                               │
    │        [ Subscribe ]          │  │        [ Subscribe ]          │
    └───────────────────────────────┘  └───────────────────────────────┘
    ```

  - **B: one set of buttons above both cards.** Three joined buttons, **Just me | Couple | Family**, with one selected. Picking one changes the price on both cards, and the saving shows under the buttons. It's tidier, but you only see one size's prices at a time, and Silver has to say "Not offered for families" under Family.

    ```
              [ Just me |▐ Couple ▌| Family ]
          Couples save up to $50 a person each month

    ┌ Silver ──────────────┐  ┌ Gold ────────────────┐
    │ $240/month for two   │  │ $300/month for two   │
    │ $120 each            │  │ $150 each            │
    │    [ Subscribe ]     │  │    [ Subscribe ]     │
    └──────────────────────┘  └──────────────────────┘
    ```

  - **C: keep the five cards,** and add the saving line to the three shared ones.
  - **Recommendation: A.** It does what Thomas asked (fewer cards, the discount said out loud) without hiding any price.
  - Mason- Let's go with A
- → Plan: 5.2, as the sketch shows.

**OQ-19 — Product categories** *(9.7)*
- **Default:** group by the existing **kind** (Memberships, Classes & drop-ins, Events & workshops, Services), plus a search box. No new column.
- **Alternative:** a free category field you fill in for each product.
- Mason- I really don't understand the issue being addressed here. Can you let me know what your asking and why?
- → Explained: Gifford wrote, under *Admin > Manage Products*: "Categorize the products so that it's easier to manage the big collection of products." That can mean two different things:
  1. **The grid of tiles on Manage Products** (32 of them). That's the same point as Thomas's "gigantic grid", and 9.2's groups, which you approved, answer it.
  2. **The Products list** (**Manage Products** ▸ **Products**), every item you sell in one list. It holds eight seeded products today, and each new workshop or event adds one.
  - v0.1 read it as 2. It proposed sectioning that list by the kind each product already has (Memberships, Classes & drop-ins, Events & workshops, Services) plus a search box; the alternative was a category field you'd fill in yourself.
  - **Default now: reading 1.** 9.2 covers it, so 9.7 builds nothing. With eight products, the list isn't big yet.
  - Say if you'd still like the Products list sectioned (a small change), or ask Gifford which he meant.
  - Mason- Let's go with the default.
- → Plan: 9.7 builds nothing.

**OQ-20 — How far "restore" goes** *(Phase 10)*
- **Default:** every change made through Manage Data (and the pages that use the same generic endpoints) is recorded and can be restored one row at a time. Changes made by dedicated pages (subscriptions, bookings, payments) aren't, since they have their own records.
- **Alternative:** record those too: larger, and touches every helper.
- Mason- Can you explain what the issue we are trying to resolve here is? I'm on the fence about Manage Data. I feel like it started out as a way to manually create items in the database before we added bespoke UI to things. Now it is more of diagnostic / developer access tool. It's kind of beyond admin. Would it maybe make more sense to add a special permission for it? I think those same endpoints that they UI hits might be used for other features in the UI? How expensive is doing this rollback feature?
- → Explained:
  - **The issue.** Thomas, as an admin, could open Manage Data and edit any row of any table, with no explanation of the columns and no undo. A wrong edit (a price, a class's required permission, a foreign key) breaks the site and leaves no record of what the value was. He suggested help text, foreign-key previews, and history with rollback. v0.1 planned all three (its 10.1–10.4).
  - **Your instinct matches the code.** Manage Data began as the way to create rows before the dedicated pages existed. Today every workflow has its own page, so Manage Data is a developer's tool. The cheapest fix for "risky" is to let fewer people reach it.
  - **Do other pages use the same endpoints? Yes.** About twenty Manage Products pages call the generic add/update/delete endpoints. Among them: Products, Pricing Overview, Membership Tiers, Class Schedules, Instructors, Locations, Page Content and the event editors. No member page does.
    - So the endpoints stay. What changes is who can open the Manage Data *page*, and the schema behind it.
    - Separately, Phase 1 makes the endpoints check the caller's grants.
  - **What rollback would cost.** There are three pieces:
    - a change log that the four generic writes fill, in the same transaction
    - a history endpoint and a list on the page
    - a restore that writes the old row back

    The log on its own is modest. Restore is where the work is: putting back a deleted row with its old id, refusing when a foreign key no longer matches, and being able to undo a restore. With tests, the whole thing is roughly the size of the sharing rework (4.1–4.4). It would also record the twenty pages' edits, which is a plus, but it's a lot of work for a tool only you use.
  - **Default now:** a **Developer** role with its own `manage_data` permission, given only to you, plus a warning banner (10.1). The help text, previews, table list and history move to 10.3, deferred.
  - **Say if you'd rather build the history anyway.** It goes back in as 10.3's last item, at the size above.
  - Mason- I think we should do the developer role. Do you think this is a good idea or that we should do the rollback functionality? It seems like a lot of work for something kind of obscure.
- → Plan, and my view: **the Developer role, and no rollback.**
  - **The role is the right fix.** The problem Thomas hit is that any admin can reach a raw editor with no undo. 10.1 takes that editor away from everyone but you.
  - **Rollback isn't worth building now:**
    - After 10.1 you're the only person who can make the edits it would undo.
    - **The disaster case is already covered.** RDS keeps seven days of point-in-time backups, and the restore drill passed on 9/25 and 10/3 (Deploying to AWS, runbook §6a).
      - It's clumsy: you restore a copy of the database to just before the mistake, then copy the old rows back.
      - It's there, though, and it covers every table, not just the ones Manage Data writes.
    - **"Who changed this price, and when?"** If you ever want that, it belongs on the twenty Manage Products pages staff actually use. 10.3's change log would cover those too, and nothing in 10.1 needs redoing to add it later.
  - Phase 10 is no longer marked as a default. 10.3 stays deferred.

**OQ-21 — Server-rendered pages** *(Phase 11)*
- **Default:** titles, descriptions, link-preview tags and a sitemap now (11.1). Full server rendering becomes its own plan, if you want it after seeing 7's speed work.
- **Alternative:** plan server rendering now.
- Wasn't the server side rendering also for SEO? Would it make sense to possibly have an image cache in memory for the home page photos, gallery, and generally images that are there for every user versus the dynamically generated ones per user? I imagine there are C++ libraries for caches that do some kind of LRU removal.
- → Explained:
  - **Yes, server rendering is mostly for search engines and link previews,** plus a faster first paint. Today the server sends an empty page and the browser builds it with JavaScript.
    - Google and Bing run the JavaScript, though Google can take days to come back and render it.
    - Most other crawlers read only the HTML. So do the previews that Messages, Slack and Facebook draw for a shared link. They see a blank page titled "Knotty Yoga".
  - **11.1 (the default, small).** Each public page gets its own title, description and preview image in the page head, and the site gets a sitemap. That fixes link previews and gives every crawler something to read. It doesn't put the page's body text into the HTML.
  - **Full server rendering (large).** Angular runs on a Node server next to the C++ one. CloudFront sends page requests to Node, which calls the C++ API and returns finished HTML.
    - That's a second server to deploy, watch and keep in step, and every page has to be checked for browser-only code.
    - It's worth a plan of its own once 11.1 and Phase 7's speed work are in and you can see what's still missing. Google Search Console shows what Google actually indexed.
  - **The image cache: yes, but the best place for it is CloudFront, not the C++ server.**
    - Today every photo request goes through CloudFront to the server, because the `/api/*` behaviour caches nothing (F20). The server answers `no-cache` with an ETag, so a repeat view is still a round trip, and a first view is a database read.
    - If the public photos (home page, gallery, classes, instructors) get an address that changes whenever the photo changes, CloudFront can keep them at its edge for a year. The server never sees a repeat request, and a replaced photo shows at once because its address is new. People's own photos stay uncached. That's new item **7.8**.
    - An in-memory LRU cache in the server would save only the database read, and each server would have its own copy. It's the fallback if 7.1 shows that first views are slow.
  - **Default:** 11.1 and 7.8 now, with server rendering as its own plan later. Say if you want that plan written now instead.
  - Mason- Let's go with the default. Is the cloudfront image cache thing going to get a work item?
- → Plan: **yes, 7.8 is that work item.**
  - Public photos get an address that changes whenever the photo does, and CloudFront keeps them at its edge for a year.
  - The CloudFront behaviour for it is yours to add in the console, from steps written when 7.8 is built, before the UI goes out (Phase 13).
  - 11.2 records server rendering as a later, separate plan.
  - **One correction:** I wrote that 11.1's tags fix link previews. They do only once they're in the HTML the server sends, which OQ-22's new option D does.

**OQ-27 — Subscribing for a later month** *(2.3)*
- Today it's free that month (F4).
- **Default:** charge on the 1st of that month, as the page says, with nothing granted before then.
- **Alternative:** charge now for that month.
- Mason- I don't understand this. What is free today?
- → Explained: the subscribe page lets you pick when the membership starts, this month or a later one. Say it's November 16 and you pick **December**.
  - **What the page says:** you'll be charged on December 1.
  - **What actually happens:** nothing is charged when you subscribe, and you get December's access at $0. The first charge is on **January 1**. Nobody ever pays for December, so it's a free month (F4, `subscription_helper.cpp:289-314`).
  - **Option A (default): charge on December 1.**
    - Nothing is charged today, and nothing is granted until December. On the 1st the billing run charges the month and grants it, like any renewal. That's what the page already promises.
    - If the card fails on the 1st, the membership goes past due, as a failed renewal does.
    - Cancelling before the 1st costs nothing.
  - **Option B: charge for December today.**
    - The card is tested while the person is still on the page.
    - But they pay two weeks before they can use it, and cancelling before December means a refund.
  - Either way December stops being free. **Recommendation: A.**
  - Mason- I'd like a way to be able to have the option to have staff create a membership that they pay now for the next month but the rest of the current month is thrown in for free. But this is something that needs to be done by staff. For users, I would like them to be able to sign up for a membership and have the option to have the billing and membership become active the next month but also have the option to make a pro rated payment for the current month and then have full month billing start for the next month.
- → Plan: 2.3 is rewritten around named **start choices**. The server works out the dates and the amount from the choice; the page no longer sends dates.
  - **Members** get your two:
    - **Start today:** the rest of this month, by the day, now; then the full price on the 1st of each month.
    - **Start on December 1:** nothing now; access and the first full charge both start on the 1st, through the billing run.
  - **Staff, on Create Subscription,** get both of those, plus:
    - **Start today, rest of this month free:** next month's full price charged now, and access from today.
    - **Start on the 1st of a later month.**
  - **Your staff option half exists already.** **Create Subscription** has an **Include this month free** box, but with it nothing is charged until the 1st. 2.3 makes it charge now, as you described.
  - **Found while checking this (P13):** a member who picks the current month today pays the whole month's price, even on the 28th. **Start today** fixes that.
  - **One preview endpoint** (`start_options`) shows the same numbers that checkout will charge, signed in or out (OQ-14).
  - **Two calls I made that you should check:**
    - Members lose the later months, since you described two choices.
    - The staff page loses **Prorate for remaining days** unticked, which charged the full price for a part month.

    Say if either should stay.

**OQ-30 — Database connections per site** *(7.6)*
- **Default:** 4, so four requests run at once. The t3.small and the RDS instance have room; to confirm against RDS's connection limit.
- **Alternative:** 2.
- Mason- Can you explain the pros and cons of each choice. You haven't given me a lot to make a decision here :)
- → Explained: the server keeps a few open connections to the database, called the pool. Each request borrows one for as long as it runs.
  - **Today** every site has **one**. So the server answers one request at a time per site, and the browser queues its requests so it never sends two at once (F13). A page that needs three calls, as the calendar does, waits for all three one after another.
  - **What a bigger pool buys.** Requests run side by side. The calendar's three calls overlap, and so do the start-up calls, so a page waits for its slowest call instead of the sum of all of them. Two visitors stop waiting on each other, too.
  - **What it costs.**
    - *Database memory.* Each connection is a Postgres process using a few MB. The RDS `db.t3.micro` (1 GB) allows on the order of 80 connections, so one site at 4 is a small share.
    - *Correctness.* With one connection, two requests from the same person can never overlap, and that hides races such as a double-tapped **Subscribe**. With more, they can. So 2.2's row lock and 7.6's concurrency test go in first, and that work is the same for 2 or 4.
    - *Later, with many studios on one database,* the count is per site. That's why it's set in the app config, where it can come down.
  - **2 vs 4.** With 2, a page's main call and one background call overlap, but the calendar's three calls still queue. With 4, the busiest page's three calls all run at once, with one to spare. The memory difference is a few MB per extra connection.
  - **Recommendation: 4,** read from the app config so changing it is one line.
  - Mason- I'll go with your recommendation.
- → Plan: 7.6, as written: 4 connections, read from the app config.

### Resolved 11/16/2026

*Your first round, in number order.*

**OQ-1 — One membership per person?** *(D2, 2.2)*
- **Default:** a person can be the payer of only one current membership. "Current" includes one that's cancelled but running to the end of its month, and a change scheduled for next month. Someone who holds a seat on another person's membership is asked "You're already on Thomas's Gold Couple's membership. Buy your own anyway?" and may go ahead.
- **Alternatives:** refuse the seat holder outright; or allow two memberships if they're different tiers.
- Mason- I actually have had people who were paying for a poor friend's membership so I don't think that we should only allow a single person to just pay for one membership. However, I would like to have the UI explicitly warn them that they are already paying for a membership and have to acknowledge this before paying for a second.
- → Plan: D2 and 2.2 are rewritten.
  - A second membership is allowed once the buyer ticks **I want an additional membership**, on a warning that names what they already pay for or share. Without the tick, the server refuses (`ALREADY_HAS_A_MEMBERSHIP`), so a second tab or a script can't skip it.
  - A second membership doesn't give the buyer its first seat, so the friend's seat is free.
  - 2.6, 2.7, 3.3 and 4.6 follow from this.

**OQ-2 — What does an upgrade cost?** *(3.1)*
- Today the server charges the **whole** month's price difference, whatever the date.
- **Default:** prorate it by the days left, with the same rule the first month already uses. Silver → Gold on the 20th of a 30-day month costs a third of the difference.
- **Alternative:** keep the full difference.
- Mason- let's prorate.
- → Plan: 3.1 prorates, as the default had it.

**OQ-3 — Downgrading to fewer seats** *(3.1)*
- Gold Family (3 seats) → Gold Couple's (2): who loses their seat?
- **Default:** the payer keeps theirs, then seats are kept in the order they were given. Whoever's left off gets an email, and so does the payer. The confirm screen names them first.
- **Alternative:** the payer must remove someone before the downgrade can be scheduled.
- Mason- it would be good to prompt who to remove.
- → Plan: 3.1 and 3.3.
  - A downgrade to fewer seats makes the payer tick who comes off, and the server refuses the change without that choice (`CHOOSE_WHO_TO_REMOVE`).
  - The people named keep their seat to the end of the period, and each gets an email.

**OQ-4 — The "you just bought this" window** *(2.5)*
- **Default:** a paid order with the same items by the same person within **10 minutes** asks "Buy it again?".
- **Alternative:** a different window, or none (rely on 2.6's page changes alone).
- Mason- This sounds like a good idea but can we make it configurable in the admin UI but default to 10min?
- → Plan: 2.5 reads the window from a setting: 10 minutes by default, and 0 turns it off. New 2.8 is a **Studio Settings** page under Manage Products for changing it. There was no settings page to put it on (F21).

**OQ-5 — The testers' extra memberships on the live site** *(2.8)*
- They are sandbox charges, but the billing run will keep charging the duplicates every month.
- **Default:** before sandbox.15, Claude lists them and Mason cancels the extras under **Manage Products** ▸ **Subscriptions**.
- **Alternative:** leave them; or reinstall production (which also removes the testers' accounts).
- Mason- This is the Square sandbox and basically monopoly money that is all going to get thrown away so I don't care as long as we make it hard for them to get in this situation after we do these updates.
- → Plan: the clean-up item is gone. The reinstall (OQ-6) removes them, and Phase 2 is what stops it happening again.

**OQ-6 — Migrate or reinstall production?** *(D8, Phase 13)*
- Finishing website reinstalled production to get its content in. The testers now have accounts and history there.
- **Default:** migrate, and make the content changes by hand from a numbered list.
- **Alternative:** reinstall from the seed, and ask the testers to register again.
- We are going to fully reinstall from seed. This is extremely test data that needs to be thrown away.
- → Plan: D8 and Phase 13 are rewritten.
  - sandbox.16 reinstalls production from the seed (`--install_schema --force` plus the `set_secret` block), and the testers register again.
  - Every "production edit" step is gone. So are the migrations that only fixed data or wording: v0.1's admin hints, admin descriptions and invitation lower-casing.
  - Schema migrations stay, for local databases and for sandbox.14 and .15, which still migrate.

**OQ-8 — Getting Started's new order** *(5.3)*
- **Default:**
  1. What to expect
  2. Try an intro workshop
  3. Find the classes for you
  4. Meet the instructors
  5. Pick a membership
  6. Create your account and plan your week
- Say if you'd order or name them differently.
- Mason- This sounds fine.
- → Plan: 5.3 as written.

**OQ-10 — The "Our" rule** *(8.4)*
- **Default:** no "Our" in the menu, and each page's title matches its menu label (**Classes**, **Services**, **Instructors**, **Location**, **Providers**, **Events**).
- **Alternative:** "Our" on every page title and none in the menu; or "Our" everywhere.
- Mason- I like the dropping our idea.
- → Plan: 8.4 as written.

**OQ-11 — "Services" for massage** *(8.6)*
- Gifford wondered if people would look for massage under "Services".
- **Default:** keep **Services**, and list the providers on it. The home page already advertises massage.
- **Alternative:** rename it **Massage**, or **Massage & Bodywork**, while massage is the only service.
- Mason- We will soon offer more than massage so lets keep it as services.
- → Plan: 8.6 keeps **Services** and lists the providers on it.

**OQ-12 — Gallery in the top menu** *(8.4, Gifford)*
- **Default:** yes, next to **Blog**, and out of **About**. Gifford expects it to carry event photos over time, as the blog does.
- **Alternative:** keep it under **About**.
- Mason- There are already enough top level menu items. Let's keep it under about.
- → Plan: 8.4 no longer moves Gallery.

**OQ-13 — A Classes menu** *(8.4, Gifford)*
- **Default:** **Classes** opens **All Classes**, **Weekly Schedule** and **Calendar**. The buttons on All Classes stay as shortcuts.
- **Alternative:** keep a single **Classes** item and make the two buttons more prominent.
- Mason- I don't like making this a menu item. I think the issue is that if you have created an account but not bought a membership, the My Schedule view shows nothing, which is confusing. I think we should default to Full Schedule view if they don't have a membership and then they see the LOCK icon with Requires a membership for the upsell.
- → Plan: 8.4 drops the Classes menu.
  - You're right about the cause (F19). Our Classes starts every signed-in person on the eligible-only view, so an account with no membership sees "Your current filters hide everything."
  - New in 6.3: anyone without a membership starts on **Full Schedule**. Members' classes there already show the lock and "Requires a membership", which 6.3 links to **Memberships**. Members still start on the eligible view.

**OQ-16 — The Manage Products groups** *(9.2)*
- **Default:** the six groups in 9.2's table.
- Say if a tile belongs elsewhere, or you'd like fewer groups (Thomas suggested about five).
- Mason- I like this grouping.
- → Plan: 9.2 as written, plus the **Studio Settings** tile in **Website** (2.8).

**OQ-17 — What the business overview counts** *(9.1)*
- **Default:**
  - **Monthly recurring revenue:** the sum of current memberships' monthly prices.
  - **Next month's forecast:** that, less memberships cancelled before the 1st and plus scheduled changes.
  - **Last month's revenue:** completed payments less refunds, by payment date.
  - **Utilization:** check-ins ÷ capacity (past) and plans + bookings ÷ capacity (next 4 weeks).
  - **Underused / full:** a slot under 40% / at 100% in most of its weeks.
- Say if you track any of these differently.
- Mason- This all sounds great.
- → Plan: 9.1 as written.

**OQ-18 — A visiting teacher without an account** *(9.5)*
- **Default:** a **Led by** field: a name shown on the workshop, stored on the session (one new column). An instructor with an account is picked as today.
- **Alternative:** require an account (a quick account by staff), or put the name in the title.
- Mason- I want to require an account. It's not that hard to make.
- → Plan: 9.5's **Led by** column and its migration are gone.
  - The form picks an instructor, and a visiting teacher is added under **Instructors** first.
  - 9.5 checks that **Instructors** can add someone with no account, and adds that if it can't.

**OQ-23 — "Build up to it in…"** *(6.4)*
- Nothing in the data says which class prepares you for which. The default shows the **predecessor** class and the **required skills**, and names Beginner Knotty Yoga as the way to build up.
- **Default:** the "build up" class is a per-class setting (**Prepares you for**, a list of classes, one new table), so it isn't hard-coded. The seed's pairs are in 6.4; correct them if they're wrong.
- **Alternative:** leave it to each class's description (no schema change).
- Mason- 
	- Beginner Knotty Yoga prepares for: Knotty Yoga and Beginner Partner Acro
	- Knotty Yoga prepares for: Handstands, Aerial Rope, Beginner Partner Acro
	- Beginner Partner Acro prepares for: Intermediate Partner Acro
- → Plan: 6.4's seed uses your pairs. They also make the class pages say it, so the descriptions need no new sentence (OQ-9).

**OQ-24 — A site search** *(Gifford, "not a strong recommendation")*
- **Default:** not now. The site has about twenty public pages, and Phase 8's menus make them easier to find.
- **Alternative:** a search over classes, instructors, events and blog posts.
- Mason- Let's skip search for now.
- → Plan: no search.

**OQ-25 — The favicon** *(8.2)*
- **Default:** made from the logo in the brand red.
- **Alternative:** Ryan designs one; drop it in the Photos folder.
- Mason- Let's go with the SVG of the human hanging in the red fabric.
- → Plan: 8.2 crops the figure-in-fabric emblem out of the logo file (`KnottyYoga_logo_white.svg`). Its figure is white, so it sits on a dark rounded square to show up on light browser tabs. If you meant a different SVG, drop it in the Photos folder.

**OQ-26 — When does a class leave the sign-up lists?** *(6.2)*
- **Default:** when it starts. **Today's Classes** keeps it until it ends.
- **Alternative:** when it ends, everywhere.
- Mason- I'd like to leave or for two hours after the class ends. It's pretty common for people to show up for a class and not sign up and then show up a few minutes late. It's nice for staff to be able to note this after the class ends (or for the person to)
- → Plan: 6.2 is rewritten.
  - A class stays on the lists until two hours after it ends, and until then it can be signed up for and checked in, by staff or the member. The two hours is a setting on Studio Settings.
  - A class that has started sits under **Earlier today** with its end time, so it doesn't read as upcoming. That keeps Thomas's point about past classes in the list.

**OQ-28 — An email on every weekly-plan tap** *(6.1)*
- Adding a class to the weekly plan sends a confirmation email each time, so five taps make five emails.
- **Default:** stop it; the weekly digest lists the plan.
- **Alternative:** keep it.
- Let's just do the weekly digest. I don't want to spam emails. The only reason to generate an email per click is if it is sending an ICS entry to add to their calendar for each class (which is useful). If we do the alternative one email per click, it would be nice to be able to adjust this in their notification settings so they have control. It would be nice to have each email have a link to adjust these settings that takes them to their notification settings page on the site.
- → Plan:
  - **6.1 stops the per-tap email.**
  - **For calendar entries,** 6.1 makes sure the personal calendar feed includes weekly-plan classes, and links to it from My Weekly Plan. The feed is already offered on **Notifications**. One subscription keeps someone's calendar current as their plan changes, with no email per class.
  - **New 8.12:** every notification email ends with a link to **Notifications**.

**OQ-29 — Which control did Thomas tap on Our Classes?** *(6.3)*
- The **My Schedule** toggle there means "classes I can attend", not "my weekly plan". A planned class shows a small checkbox. It's unclear which one let him sign up again.
- **Default:** 6.3 fixes both readings. If you can ask him, his answer goes here.
- Mason- Can you explain the issue here and what the choices are.
- → Explained, and no decision is needed. Thomas wrote that Our Classes "lets me sign up for classes that are already in my weekly schedule — despite it showing under 'My Schedule'." Two things on that page could cause it:
  1. **The toggle's name.** On Our Classes, **My Schedule** means "classes I'm allowed to take". But **My Schedule** is also the My Account tile for the weekly-plan page. So the list read as "this is my plan", while every class on it still offered sign-up.
  2. **The class chips.** A class shows its plan state only if the page managed to load your plan. When that lookup fails, the page quietly treats nothing as planned and offers sign-up everywhere (`our-classes.component.ts:219-222`).
  - 6.3 fixes both:
    - the toggle becomes **Classes I can take**, and the tile becomes **My Weekly Plan** (5.4)
    - a planned class reads **On your weekly plan**, with no sign-up control
    - a failed lookup says so
  - Asking Thomas would only tell us which of the two he hit.