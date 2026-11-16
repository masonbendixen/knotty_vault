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

Jennifer Schulte offered this feedback:

So far, I’ve only found one minor kink - when selecting “Browse services” from the My Bookings page, it somehow takes you to the Membership page. Although it does show “Browse bookable services” link on the top, it is just a bit confusing… Don’t know why it does that take you to the “Our Services” page.

Also, when navigating to a different page from the Get Started page and hit the back button on the browser, it does not take u back to where u were. For example, select “Plan your weekly schedule” on the Get Started page, then hit the back button on the browser, you’ll notice that it does not take you back to the top of the Get Started page, instead of the “Plan your weekly schedule” section. Same behavior when u select any other sections on this page.

END JENN FEEDBACK





Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Revision notes and working agreement

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
- **Migration numbers** follow build order: app `0013`–`0019`, honuware `0004`–`0006`. If phases are built in a different order, they are renumbered then.
- **Who builds (standing agreement, Finishing website OQ-18).** The Overview's line "I will build the C++ server and run tests myself" is superseded. **Claude builds and runs the tests on both Windows and Linux, fixes what fails, and reports the result for each platform** ("Windows: N passed. Linux: M passed."). Claude says so before long or destructive runs. Git stays Mason's: Claude only reads history. Mason commits, pushes and tags, unless he asks Claude to tag.
- **Three repos.** Some items change the framework:
  - `server_components` (honuware C++), pinned by `GIT_TAG` in both knottyyoga and CommunityFinder
  - `honuware-web-components` (`@honuware/ui`), exact-pinned in `ui/package.json`

  Each such item ends with the usual cycle: gates → Mason pushes → CI → pin bump or npm release → the app's gates. Items are tagged *(server_components)*, *(@honuware/ui)* or *(knottyyoga)*.
- **The working tree isn't clean.** The iCal feed work is uncommitted: `personal_ical_feed_helper.*`, `get_my_ical_feed.cpp`, `create_database.cpp`, `web_app.cpp`, two CMakeLists and two tests. Phase 1's release needs Mason to commit it or set it aside first, so the tag carries only what's meant to ship.

---

# Where each piece of feedback goes

**T** = Thomas, **G** = Gifford, **J** = Jennifer, **P** = found while planning.

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
| T30 | Manage Data gives no context; schema help; foreign-key preview | 10.1–10.3 |
| T31 | No versioning or rollback | 10.4 |
| T32 | Login prompt on the first visit to Manage Products | 9.3 |
| T33 | The grid of tiles; group it under ~5 headings | 9.2 |
| T34 | Attendance plan search can't find people without a plan | 9.4 |
| T35 | No way to create a one-off workshop | 9.5 |
| T36 | Pricing Overview: say the table is by role, and Standard is the default | 9.6 |
| T37 | Utilization reports should look forward as well as back | 9.8 |
| T38 | Leftover "decision history" wording | 8.9 |
| T39 | Session tokens vs JWTs across replicas | F15, D6 (no change) |
| T40 | Back and forward behave inconsistently | 2.6, 8.1, 8.11 |
| T41 | A raw database API is a big attack surface; red-team it | Phase 1, 10.5 |
| T42 | Inline style in the index document | D7, 7.7 |
| T43 | Script references belong in the head | 7.3, 7.7 |
| T44 | Server-rendered signed-out home page | Phase 11 |
| T45 | `site_info` is late and slow | 7.2, 7.3 |
| T46 | Fonts start late | 7.4 |
| G1 | Let newcomers explore before committing or making an account | 5.2, 5.3 |
| G2, G8, G9 | Gallery to the top level; unbury Weekly Schedule and Calendar | 8.4 |
| G3 | A site search | OQ-24 |
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
| G19 | Categorize products | 9.7 |
| J1 | My Bookings' "Browse services" goes to Memberships | 8.8 |
| J2 | Back from a Get Started link doesn't return to the step | 8.1 |
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
4. **A future start month is free.** With a later month chosen, no charge is made: the "no entitlement yet" path makes a $0, pre-credited entitlement for that month (`subscription_helper.cpp:289-314`), and the first bill is at its *end* (`:132-133`). The page promises a charge on the 1st (`subscription-signup.component.html:104`).
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
18. **Production is migrated, not reseeded.** Seed changes reach only new installs. Content that lives in the database (Getting Started steps, home sections, bios, membership descriptions) reaches the live site through a migration or a hand edit in **Manage Products** (D8).

---

# Design decisions

- **D1 — The server enforces "once"; the page only helps.**
  - The server refuses: a second membership, a paid order paid again, an unpaid membership left active. Each has its own error code that the page turns into a sentence.
  - The page avoids getting there: the order in the URL, history replaced on success, the button disabled while processing. A bug or a second tab in the page can't charge twice.
- **D2 — A person holds at most one current membership as its payer** (OQ-1).
  - A membership is a product of kind `subscription`.
  - "Current" means active or past due, **or** cancelled but running until its period ends, **or** a scheduled change that starts next period. That way Back, Reactivate and Change can't make an overlap.
  - Holding a *seat* on someone else's membership doesn't count. Buying your own then asks "You're already on Thomas's Gold Couple's membership. Buy your own anyway?" (OQ-1).
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
- **D8 — Content changes go into the seed and into a numbered list of hand edits for production** (OQ-6).
  - Production is migrated, not reinstalled, so the testers keep their accounts.
  - Text Mason may already have edited (Getting Started steps, bios, descriptions) is **not** overwritten by a migration. Each phase lists its production edits with exact menu paths, done in Phase 13.
  - Schema and seed-*metadata* changes (allow-lists, admin field hints) do go through idempotent migrations.
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
- [ ] Every caller of the generic API in `ui/` and `@honuware/ui`:
  - the writes `addItem`, `addItemFetchPrimaryKey`, `updateItem`, `deleteItem`
  - the reads `getTableRows`, `getRow`, `getFilteredTableRows`, `getDbSchema`

  For each: the table, and who reaches it (signed out, member, staff, `manage_products`, `manage_class_schedule`, admin).
- [ ] Any **member** page that writes through the generic API gets a dedicated endpoint first (in 1.4), so 1.2 doesn't break it. Expected: none, because the weekly plan and notification preferences have their own endpoints. To confirm.
- [ ] The same inventory for CommunityFinder's UI, which takes the same pin.

### 1.2 Writes need a grant *(server_components: platform/endpoints)*
- [ ] `add_item`, `add_item_fetch_primary_key`, `update_item`, `delete_item` call `RequireTableWriteAccess` instead of `IsTableAllowed`:
  - 401 signed out
  - 403 without a grant
  - admins, and the tables a person's permissions grant, as today
- [ ] `delete_item` becomes **POST**, and the GET route goes, so the CSRF guard covers it.
- [ ] `get_db_schema`: `display_templates` are filtered to the caller's allowed tables, as the column data already is.
- [ ] Tests, all through `handle_full`, reading the row back afterwards to prove nothing changed:
  - signed out, each of the four writes on `products` → 401, and the row unchanged
  - a member with no grants → 403
  - `manage_products` → allowed on `products`, 403 on `classes`
  - admin → allowed
  - GET `delete_item` → not found
  - the existing CRUD tests pass unchanged

### 1.3 No GET that writes *(server_components + knottyyoga)*
- [ ] A census of every `CROW_ROUTE` registered for GET whose handler writes. Each becomes POST, with a test that the GET is gone. `delete_item` is the known one; the census says whether there are others.
- [ ] Record the census here.

### 1.4 Private tables off the public list *(knottyyoga: database_helper + migration)*
- [ ] `PopulateAllowedTables` stops listing these tables:
  - `attendance_templates`, `attendance_template_entries`, `attendance_template_exceptions`
  - `user_notification_preferences`
  - `ical_feed_tokens`

  Admins still reach them through the admin top-level and nested lists, and `manage_class_schedule` through its table grant. To verify against `PopulateAdminTablePermissions`.
- [ ] Migration **`app/0013_private_tables_not_public`**: deletes those five `allowed_tables` rows. It is idempotent, and a second run changes nothing.
- [ ] Tests, through `IsTableAllowed` (the reader every generic endpoint uses):
  - signed out, refused on all five
  - admin, allowed
  - `manage_class_schedule`, allowed on the template tables
  - the migration clears the rows on a database that has them, and running it twice is harmless
- [ ] Any member write that 1.1 found moves to a dedicated endpoint here, with tests.

### 1.5 The browser side *(@honuware/ui: access; knottyyoga: ServerAccessNetwork)*
- [ ] `deleteItem` sends POST in `crud-http-access.ts` and in any app-side copy. A spec pins the verb and path.
- [ ] `@honuware/ui` patch release; `npm install @honuware/ui@X.Y.Z --save-exact` in `ui/`.

### 1.6 Ship it
- [ ] Gates: server_components (Windows + Linux); knottyyoga against the local checkout (Linux); Angular (Windows + Linux).
- [ ] Mutation check: put `IsTableAllowed` back on each write, and GET back on delete. The new tests must catch every one.
- [ ] **Mason** pushes server_components; CI; Claude bumps the pin in knottyyoga and CommunityFinder and reruns both pinned gates.
- [ ] `@honuware/ui` release (Mason tags), install, Angular gates.
- [ ] Release as **`v1.0.0-sandbox.14`**: server and UI together. The new server refuses the old UI's GET delete, so they must ship as a pair.
- [ ] After deploy, Claude probes production signed out (`curl -m 10`, read-only and write attempts on a throwaway `products` row name). Every write must answer 401. Results recorded here.

---

# Phase 2 — Pay once *(Thomas: "the most critical issue")*

T7, T8, T22 and T23, plus F3, F4 and F7.

### 2.1 A failed first charge leaves nothing behind *(business_logic/payment)*
- [ ] `CreateSubscription` charges **before** it writes the subscription:
  1. Validate.
  2. Create the purchase.
  3. Charge.
  4. Only on success, insert the subscription and its charge record.

  On a decline, the pending purchase is cancelled. No subscription row, no entitlement and no next-billing date are left.
- [ ] The complimentary and admin no-card paths keep their behaviour.
- [ ] Tests, through the billing run's due-row reader and `GetActiveByPersonId`:
  - declined card → no subscription, nothing due for billing, the purchase cancelled
  - `RunBilling` afterwards charges nothing
  - retrying three times with a declined card leaves zero subscriptions, then one after a good card
  - the success path is unchanged: one subscription, one entitlement, the buyer seated

### 2.2 One current membership per person *(table helpers + business_logic/payment + endpoints)*
- [ ] `TableHelpers::Subscriptions::GetCurrentForPerson(personId, nowUs)` is the D2 definition, in one query. Tests for each of its four cases, plus expired (not current).
- [ ] `SubscriptionHelper::GetCurrentMembership(personId)`.
- [ ] `CreateSubscription` refuses with **`ALREADY_A_MEMBER`**, carrying the current subscription's id and product name. Admin creation gets the same refusal: an admin changes a membership, not adds one.
- [ ] Holding a seat on someone else's membership: the refusal is **`ALREADY_ON_A_SHARED_MEMBERSHIP`** unless the request says `confirm_own_membership: true` (OQ-1).
- [ ] `POST /api/subscriptions` answers 409 with `{code, subscription_id, product_name}`.
- [ ] Tests:
  - second subscribe → 409, and still exactly one subscription
  - the same for each "current" case: cancelled-but-running, past due, a pending downgrade
  - expired → allowed
  - a seat holder → refused without the flag, allowed with it
- [ ] The database serializes a person's requests today, because each site has one connection (F13). 7.6 raises that, so it adds a row lock on the person (`SELECT … FOR UPDATE`) inside `CreateSubscription`, with a concurrency test.

### 2.3 A later start month is billed on its 1st *(business_logic/payment; OQ-27)*
- [ ] Subscribing for a future month makes no entitlement and no charge now. `next_billing_us` is that month's start, and the billing run charges it on the 1st, as the page says.
- [ ] Tests:
  - the billing run on the 1st charges and creates the entitlement
  - nothing is granted before then
  - `GetCurrentForPerson` already counts it (so Back can't double it)

### 2.4 One email per signup *(business_logic/payment)*
- [ ] A subscription's first charge sends one email: "Welcome to Gold", with the receipt (amount, card, period) folded in. `PayWithCard` gets a `suppressConfirmationEmail` flag that only `CreateSubscription` sets.
- [ ] Tests against the mail double: exactly one email per signup, containing the amount and the period. One-off purchases still send their confirmation.

### 2.5 The same order twice in a row asks first *(table helpers + business_logic/payment + endpoints)*
- [ ] `TableHelpers::Purchases::FindRecentPaidMatch(payerId, items, sinceUs)`: a paid purchase by the same payer, with the same products and quantities, in the last 10 minutes (OQ-4).
- [ ] `purchase_pay_card` and `cart_checkout` refuse with 409 **`RECENT_DUPLICATE`** (name, paid at) unless `confirm_duplicate: true`.
- [ ] Tests:
  - a second identical order inside the window → 409, and Square isn't called
  - with the flag → charged
  - a different item, or outside the window → charged without asking

### 2.6 The checkout pages follow the order *(ui: pages/shop)*
- [ ] **Checkout** (`checkout`):
  - after `createPurchase`, the URL gains `?purchase=<id>` (`replaceUrl`)
  - opening with that parameter loads the purchase: paid → the receipt view with no **Pay** button; pending → the same order
  - success → `replaceUrl` to the receipt view
- [ ] **Subscribe** (`subscription-signup`):
  - signed-in members are sent to **My Membership** with "You're already a Gold member. Change it here." The same happens for a 409 `ALREADY_A_MEMBER`.
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
  - the key reused on retry
  - `replaceUrl` on success
- [ ] A Playwright spec for Thomas's exact path: pay, Back, Pay again. It must not reach the server's pay call a second time.

### 2.7 Memberships knows who's signed in *(ui: shop catalog, membership-tier-cards)*
- [ ] The tier cards take the current membership.
  - Its own card reads **Your membership** (no Subscribe).
  - The others read **Switch to this**, going to Phase 3's change flow, or to **My Membership** until Phase 3 lands.
- [ ] The catalog loads the current membership when signed in. Specs: signed out, a member, a seat holder.

### 2.8 The testers' extra memberships in production (OQ-5)
- [ ] Before release, Claude lists the duplicate and unpaid subscriptions in production through **Manage Products** ▸ **Subscriptions**, and records them here.
- [ ] Mason cancels the extras there, by the steps written at the time. That matters because the billing run would charge them on the 1st.

### 2.9 Ship it
- [ ] Mutation check:
  - subscription inserted before the charge
  - the `ALREADY_A_MEMBER` guard removed
  - the duplicate window set to zero
  - the email flag ignored
  - `replaceUrl` dropped
- [ ] Gates on both platforms; hand-testing walkthrough (written when built, blank database, exact labels). It uses Square's sandbox decline card for 2.1.
- [ ] Release as **`v1.0.0-sandbox.15`**, after 2.8.

---

# Phase 3 — My Membership: see it, change it

T5, T6, T8, T13 and T24, plus F6.

### 3.1 Changing a membership charges first *(business_logic/payment)*
- [ ] **Upgrade order:**
  1. Create the purchase.
  2. Charge.
  3. Only on success: cancel the old subscription, revoke its entitlement, create the new one, and move the seats.

  A decline changes nothing.
- [ ] **What an upgrade costs:** the price difference, prorated over the days left in the period with the existing `CalculateProratedCharge` (OQ-2).
- [ ] **Downgrades** stay at period end. The new subscription's first entitlement carries the seat holders over.
  - If it has fewer seats, the payer keeps theirs and the others are kept in assignment order up to the new size.
  - The rest get an email, and so does the payer (OQ-3).
- [ ] Changing to the same product is refused. Changing while a change is already scheduled replaces the scheduled one.
- [ ] `PreviewChange(subscriptionId, newProductId)` writes nothing. It returns:
  - immediate or at period end
  - the charge today
  - the date it takes effect
  - which seat holders would lose their seat
- [ ] Tests:
  - a declined upgrade leaves the old subscription active, with its entitlement and seats
  - the prorated amount on the 1st, the 15th and the last day
  - a downgrade carries seats over, through the billing run and the assignment reader
  - shrinking from 3 seats to 2
  - the preview equals what the change then does

### 3.2 Endpoints *(endpoints/subscriptions)*
- [ ] `GET /api/subscriptions/<id>/change_preview?product_id=` returns the preview. Only the subscriber may call it: 404 for anyone else's subscription.
- [ ] `POST …/change_product` gets the same ownership test (to confirm), and returns the codes from 3.1.
- [ ] Endpoint tests: 401, 404 for another person's subscription, the preview shape, and a decline.

### 3.3 "My Membership" *(ui: pages/account/subscriptions, header menu, shared pipe)*
- [ ] **My Subscriptions** becomes **My Membership** everywhere: the user menu item, the My Account tile, and the page title. The URL stays.
- [ ] **With a membership:**
  - the tier and a status badge (**Active**, **Past due**, **Ends Dec 31**)
  - this period's dates and the next charge
  - the seats, with who holds each one (Phase 4 adds **Add a person**)
  - the card on file
  - **Change membership**, which opens the tier cards with each one's preview: "**Upgrade** — $42.67 today, then $X a month", or "**Switch** on Jan 1". Choosing one opens a confirm.
- [ ] **Without a membership:** the tier cards themselves, with no "Browse Memberships" click (T6), plus "Not sure yet? Try an intro workshop" (5.1).
- [ ] Permission names display as words: a shared `permissionDisplayName` turns `gold_member` into **Gold member**, reusing `role.types.ts`'s rule. Used on My Membership, its detail page and User Information (T24).
- [ ] Specs: both states, the preview text, confirm and decline, the menu label, and the display names.

---

# Phase 4 — Share a membership in one step

T9–T12 and G17, plus F8–F11.

### 4.1 Schema: an invitation remembers the seat *(db_schema + migration)*
- [ ] `gift_permissions.entitlement_id` and `gift_permission_invitations.entitlement_id`: nullable foreign keys to `entitlements`.
- [ ] `gift_permission_invitations.invited_email` is stored lower-cased. Lookups lower-case too (P9).
- [ ] The CLAUDE.md new-column checklist: admin column data info, friendly names, and the display template.
- [ ] Migration **`app/0014_sharing_invites_name_a_seat`**: adds both columns if missing and lower-cases existing invitation emails. It is idempotent.
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
  - Each empty seat shows the people you already share with as buttons, then "Or invite someone new" with an email field and an **Invite** button. Enter works too.
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
- [ ] **One / Two / Family** (T2, OQ-15): a toggle over the cards, grouped by `seats_default`, starting on **One**. **Two** and **Family** read "Couples save $X a person each month", computed from the prices.
- [ ] **Silver vs Gold:** under the cards, a comparison of the included classes from the line above (✓, ✓ with approval, —). Longer descriptions come from Mason (OQ-9).
- [ ] **"Browse bookable services →"** comes off Memberships (G13).
- [ ] **Look before you sign up** (G14, OQ-14): the subscribe page opens signed out. It shows the tier, the price, the start month and the first charge. **Subscribe** reads **Sign in or create an account to continue**, returning here (4.5).
- [ ] Specs: the toggle, the saving line, the comparison, the signed-out subscribe page, and the return.

### 5.3 Getting Started: look first, commit last *(seed + Page Content + ui)*
- [ ] **New step order** (seed; and production edits per D8, OQ-8):
  1. What to expect
  2. Try an intro workshop
  3. Find the classes for you
  4. Meet the instructors
  5. Pick a membership
  6. Create your account and plan your week

  Check-in, workshops and notifications move out of the steps, to the end of step 6 and to My Account.
- [ ] **What to expect** is the new-student FAQ (G7): physical requirements by class, what to wear and not wear, what to bring ("nothing — no mat; the floor is padded"). Mason's text (OQ-9). It is a step body in Markdown; to verify the steps render Markdown.
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

- [ ] **Favorite Instructors** (T27, G16): each card links to the instructor's page and lists their next three classes.
  - The favorites endpoint returns them, from the same reader the instructor page uses, so it's one call, not one per instructor.
  - Endpoint test; spec.
- [ ] Specs: both card states, the group order, and the display names.

### 5.5 No membership, no dead end *(seed + ui)*
- [ ] The home page's **Upcoming classes** row gets `hidden_when_not_member` (seed, and a production edit). Non-members see the Get Started band and the intro call to action instead (T3).
- [ ] **My Weekly Plan** with no membership: "You need a membership to plan classes", with **Choose a membership** and **Try an intro workshop**. A member who simply has no eligible classes keeps today's message, plus a link to the class pages.
- [ ] The upcoming-classes empty text says the same.
- [ ] Specs for each state.

---

# Phase 6 — Planning the week

T14–T17.

### 6.1 A tap changes one class *(business_logic/scheduling + ui)*
- [ ] `AddTemplateEntry` checks the one slot (`IsSlotEligible`) instead of rebuilding the whole eligible list.
  - Test: for every seeded slot and a Silver, a Gold and an approved Gold member, it gives the same answer as the list reader.
- [ ] The per-tap confirmation email stops (OQ-28); the weekly digest already says what's planned. Test against the mail double.
- [ ] **My Weekly Plan:**
  - a tap flips that class at once and saves it in the background
  - each class has its own busy state, so several taps can be in flight
  - a failure flips it back with a message
  - no reload, no spinner over the grid
- [ ] The **Upcoming** tab loads when it is opened, not with the page.
- [ ] Specs: three quick taps make three calls and no reload; failure reverts.

### 6.2 Lists start now *(business_logic/scheduling + ui)*
- [ ] `GetUpcomingClassesForPerson` and `ResolveClassesInRange` start at **now**, so a class that has started is gone from the sign-up lists (OQ-26). **Today's Classes** keeps a class until it ends, since people use it at the door.
- [ ] The page asks from now, not midnight.
- [ ] Tests with a fixed clock: 5:30 pm drops the 5:00 class from upcoming; Today's keeps it until 6:00.

### 6.3 Our Classes shows what you've planned *(ui)*
- [ ] The **My Schedule** toggle becomes **Classes I can attend**. It filters by access, and its old name collided with the plan page.
- [ ] A class on your weekly plan shows **On your weekly plan** and no sign-up control. A one-off plan shows **Planned**.
- [ ] A failed plan lookup shows "Couldn't load your plan", instead of silently showing nothing as planned.
- [ ] Specs for each. Ask Thomas which control he tapped (OQ-29); this item covers both readings.

### 6.4 Which class is right for me *(db_schema + table helpers + seed + ui)*
- [ ] `classes.sort_order` (`INT NOT NULL DEFAULT 0`), with migration **`app/0015_classes_sort_order`** and the admin metadata checklist.
  - `TableHelpers::Classes` orders by `sort_order`, then name.
  - Seed order: Beginner Knotty Yoga, Beginner Partner Acro, Knotty Yoga, Handstands, Aerial Rope, Intermediate Partner Acro.
  - A **Sort order** field in Class Schedules' class dialog.
- [ ] **Prepares you for** (OQ-23): a `class_prepares_for` table (`class_id`, `prepares_class_id`, unique pair), with migration **`app/0016_class_prepares_for`**, the admin metadata checklist, and a table helper with tests. It is edited in the Class Schedules class dialog. Seed: Beginner Knotty Yoga prepares for Knotty Yoga, Beginner Partner Acro and Aerial Rope; Beginner Partner Acro for Intermediate Partner Acro; Knotty Yoga for Handstands and Aerial Rope.
- [ ] **The class page:**
  - **Start here** on classes that need only a membership
  - on a class that needs more, a **How to get here** section: the required skills (as today), "Requires attending: Knotty Yoga" from the predecessor slot, and "Build up to it in **Beginner Knotty Yoga**" from `class_prepares_for`
  - on a class that prepares for others, "**Leads to:** Knotty Yoga, Aerial Rope"
  - the class detail endpoint returns both lists; endpoint test
- [ ] Our Classes and All Classes list by the new order.
- [ ] The beginner descriptions say they prepare you for the others: Mason's copy (OQ-9), seed and a production edit.
- [ ] Tests: the migration; the order through the catalog reader; specs for the badge and section.

### 6.5 From a class to its schedule *(ui)*
- [ ] **See this class on the calendar** opens the calendar filtered to that class (`?class=<id>`). Spec.

---

# Phase 7 — Speed

T18, T43, T45, T46, G10 and P11.

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

### 7.4 Fonts *(seed + ui)*
- [ ] `index.html` and the seeded font rows use **the same** Google Fonts URL (OQ-22), so the runtime link is a cache hit, not a second download. A test checks that the seed's URL equals the one in `index.html`.
- [ ] `index.html` gets a `preload` for that stylesheet next to its `preconnect`s.
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
- [ ] `index.html`'s cache moves from `max-age=0` to `s-maxage=60`. The deploy already invalidates it, and CloudFront then stops checking S3 on every visit. To confirm in `deploy_ui.sh`.
- [ ] Critical CSS stays inlined (D7). After 7.4, check the built `index.html` no longer carries the font CSS twice.
- [ ] After numbers for first paint, next to 7.1's.

---

# Phase 8 — Polish and the site's navigation

T19–T21, T25, T26, T28, T38, T40, G2, G4–G6, G8, G9, G11, G12, J1 and J2.

### 8.1 Scrolling, Back, and the Android footer *(ui: app shell)*
- [ ] `h-screen` → `h-dvh` on the shell, so its bottom is the visible bottom on Android. The footer and the checkout bar show without scrolling (T20).
- [ ] The inner scroller gets real scroll restoration. `AppComponent` listens for the router's `Scroll` event:
  - a new page goes to the top, or to its `#fragment` once the content has loaded
  - Back and Forward go to the saved position

  That fixes Get Started (T21, J2) and every "Back lands somewhere odd" (T40).
- [ ] Specs: top on a new page; restored on Back; fragment after a delayed load.
- [ ] A Playwright check at 390 px Android: the footer is fully visible at the end of the home page.

### 8.2 Favicon *(ui + seed)*
- [ ] A Knotty Yoga icon from the logo, in the brand red: `favicon.ico` (16/32), a 180 px `apple-touch-icon` and an SVG (OQ-25).
- [ ] `site_favicon_url` stays the studio override. A spec checks `index.html` names all three.

### 8.3 Change Password *(ui: update_user_password)*
- [ ] Fields: **Current Password**, **New Password**, **Confirm New Password**. The validator compares the two new ones.
- [ ] A wrong current password stays on the page: "That current password isn't right." This closes Finishing website OQ-43.
- [ ] Specs: the labels, mismatch, the wrong current password, and success.

### 8.4 Navigation and names *(ui: header menu + page titles)*
- [ ] **Classes** becomes a menu: **All Classes**, **Weekly Schedule**, **Calendar** (G9, OQ-13). The red buttons on All Classes stay as shortcuts.
- [ ] **Gallery** moves to the top level, next to **Blog** (G8, OQ-12).
- [ ] **Upcoming Events** becomes **Events**, menu and page (G12).
- [ ] **"Our":** menu labels drop it, and each page's title matches the label that opened it (OQ-10):
  - Classes / All Classes, Services, Instructors, Location, Providers
  - "My Classes" and "Our Classes" stop swapping titles
- [ ] Specs: the menu for signed out, member, staff and admin, desktop and mobile. The mobile-audit route list gains any new route.

### 8.5 "Show me how" under the headline *(seed + ui: home hero)*
- [ ] The hero section shows its link as a button when it has one. The seed gives it **Show me how** → Getting Started, plus a production edit. Spec.

### 8.6 Services name their providers *(endpoints + ui)*
- [ ] Each service card lists who provides it ("Massage with **Caleb Ault**"), linking to the provider's bio. The data is from the providers directory reader, on the service catalog response. Endpoint test; spec.
- [ ] The name **Services** stays (OQ-11).

### 8.7 Link colour *(ui: tokens)*
- [ ] A `--theme-link` token (default: the brand's primary). `.back-link`, prose links and the 15 `text-blue-600` templates use it. `design-tokens.spec.ts` covers it.

### 8.8 "Browse services" goes to Services *(ui)*
- [ ] My Bookings (both links), the cart and the service booking page go to **Services**, not Memberships (J1). Specs pin each target.

### 8.9 Leftover "decision history" wording *(ui + seed + migration)*
- [ ] **Site Theme:**
  - "…Type it exactly as you want it to read — it is no longer forced to capitals." becomes "Type it exactly as you want it to read." (and its spec)
  - "Stored for future use — no page renders this today." becomes "Not shown on the site yet."
- [ ] The Manage Data field hints that name a phase or ticket:
  - "(Phase 3 SL-11)", "(skill_levels.id, Phase 3)", "(Phase 4)", "(M-14; …)"
  - the "(M-13)" table description

  They are reworded in the seed. Migration **`app/0017_admin_hints_plain_words`** updates only rows that still hold the old text.
- [ ] **Guard tests:**
  - a C++ test that no seeded hint or description matches `Phase \d|OQ-\d|\(M-\d|SL-\d`
  - an Angular spec that no template under `src/app` has visible text matching `no longer|used to|Phase \d|OQ-\d`, outside comments
  - the developer-only style guide is exempt

### 8.10 Bios *(seed + production edit)*
- [ ] Mason's and Gifford's bios, rewritten by Mason (OQ-7). The seed is updated, and production is edited under **Manage Products** ▸ **Instructors**. The seed test reads them back through the instructor reader.

### 8.11 Back and Forward, checked *(e2e)*
- [ ] A Playwright journey spec at 390 and 1280 px. It goes Home → Classes → a class → Calendar → Back ×3, Forward ×3, then the purchase, sharing and Get Started paths. Each Back returns to the previous page, at its scroll position.

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
- [ ] The tiles in headed groups (OQ-16):

  | Group | Tiles |
  |---|---|
  | **Classes & scheduling** | Class Schedules, Schedule Grid, Close Classes, Scheduling Exceptions, Attendance Templates, Skills, Class Tags, Locations, Room Schedules |
  | **Staff** | Instructors, Who's Teaching What, Shift Requests, Time Off Requests, Service Providers, Roles |
  | **Memberships & sales** | Products, Pricing Overview, Price Schedules, Membership Tiers, Subscriptions, Entitlements, Event Sessions, Bundles, New Workshop |
  | **Discounts & credits** | Vouchers, Coupons, Comps, Refund Effectiveness |
  | **Reports** | Enrollment Trends, Open-Seat Heatmap, Room Occupancy, Subscription Revenue |
  | **Website** | Site Theme, Fonts, Page Content |

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

### 9.5 New Workshop *(db_schema + business_logic/scheduling + endpoint + ui)*
- [ ] `event_sessions.led_by_name` (nullable), with migration **`app/0018_event_sessions_led_by`** and the admin metadata checklist (OQ-18). The public event card and page show it when there's no staffed instructor. Tests through the visible-events reader.
- [ ] `WorkshopHelper::CreateOneOffWorkshop`, in one transaction, makes:
  - an event product (name, description, capacity)
  - its price on the default schedule
  - one event session (date, start, duration, room)
  - the instructor's staffing
  - a **Led by** name for a visiting teacher without an account (OQ-18)

  Tests read the result back through the Events reader and the catalog.
- [ ] `POST /api/admin/workshops` (`manage_products`). Endpoint tests.
- [ ] **New Workshop**, a one-page form, shown on the dashboard and on Event Sessions. Its fields:
  - Title, Description
  - Date, Start time, Length
  - Room
  - Instructor, *or* Led by
  - Capacity, Price
  - Photo

  Saving opens the workshop's public page. Specs.

### 9.6 Pricing Overview says what it is *(ui)*
- [ ] An intro line: "Each column is a price for people holding that membership or role. **Standard** is what everyone else pays, and what a member pays when their column is blank."
- [ ] **Standard** becomes **Standard (everyone)**. Permission columns use display names.
- [ ] **Bug (P6):** only pricing-eligible permissions become columns, as Membership Tiers promises.
- [ ] Specs: the text, the names, and an ineligible permission's absence.

### 9.7 Products grouped *(ui)*
- [ ] The product list is sectioned by kind: Memberships, Classes & drop-ins, Events & workshops, Services. It gets a search box. The kind filter stays (OQ-19). Specs.

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

# Phase 10 — Manage Data you can trust *(mostly server_components and @honuware/ui)*

T30, T31, T41 and G18.

### 10.1 Help text *(server_components: platform db_schema + seed; knottyyoga seed + migration)*
- [ ] `admin_column_friendly_names.description` (framework migration `honuware/0005_column_descriptions`).
- [ ] The editor shows the table's description under its title. Each column's description shows under its field, for every control type including the foreign-key picker.
- [ ] Claude writes a one-line description for every app table and every column a person edits. App migration **`app/0019_admin_descriptions`** fills empty ones only.
- [ ] Tests: through `AdminColumnFriendlyNames`; a seed test that every admin top-level table has a description.

### 10.2 Foreign keys show what they point at *(@honuware/ui: controls)*
- [ ] The picker shows the chosen record's display template under the field, with **Open** to edit that row.
- [ ] Specs.

### 10.3 A list of tables, and a word of warning *(@honuware/ui: crud; ui: admin)*
- [ ] A left-hand list of tables (grouped as in 9.2), searchable, replacing the dropdown (G18). It collapses to a menu on a phone.
- [ ] A banner at the top: "This edits the raw data. Day-to-day work belongs in Manage Products. Every change here is recorded (History)."
- [ ] Specs.

### 10.4 History and restore *(server_components: platform)*
- [ ] **`admin_change_log`:** table, row key, action, the row before and after (JSON), who and when. A framework table, with migration **`honuware/0006_admin_change_log`**.
- [ ] `add_item`, `add_item_fetch_primary_key`, `update_item` and `delete_item` write to it **in the same transaction** as the change.
- [ ] `GET /api/admin/change_log?table=&row=` (admin) lists the history.
- [ ] `POST /api/admin/change_log/<id>/restore` (admin) writes the *before* image back through the same write path. That write is logged too, so a restore can itself be undone.
- [ ] A restore that would break a foreign key fails with the database's reason (OQ-20).
- [ ] Tests:
  - each write logs, read through the log reader
  - restoring an update, a delete and an add
  - a restore that a foreign key refuses
  - a rolled-back write leaves no log
- [ ] **History** on each row, and a **Recent changes** list on Manage Data. Specs.

### 10.5 Red-team pass *(server_components + knottyyoga)*
- [ ] **Route census test:** walk every registered route. Signed out, each one must answer 401 unless it is on a reviewed public list. The list lives in the test with a reason per entry, so a new endpoint is private until someone says otherwise.
  - Crow's router exposes its rules; to verify. If it doesn't, a test-time list is checked against the anchors in `web_app.cpp`.
- [ ] **Ownership tests** for every route that takes another person's id or a row id (subscriptions, purchases, bookings, gift permissions, entitlements, templates): a second member gets 404.
- [ ] **An adversarial review by Claude** against a local server:
  - every route, signed out, as a member, as `manage_products` and as an admin
  - CSRF, IDOR, mass assignment through `update_item` (writing `person_id` or `status`), rate limits, and what `get_db_schema` reveals

  Findings are recorded here and each one is fixed with a test.

---

# Phase 11 — Search engines and the first page *(T44)*

### 11.1 The basics *(ui + server)*
- [ ] Each public page sets its own `<title>` and `<meta name="description">`, plus Open Graph tags for links shared in messages.
- [ ] `robots.txt` and a `sitemap.xml`:
  - the public pages, every class and every instructor
  - generated by a server endpoint that CloudFront serves at `/sitemap.xml`
- [ ] Tests: each route's title; the sitemap lists the seeded classes and instructors.

### 11.2 Server rendering: a decision *(OQ-21)*
- [ ] A server-rendered signed-out home page means a Node runtime next to the C++ server, or a pre-render step. The page's content comes from the database at run time, so a build-time pre-render goes stale.
  - **Default:** do 11.1 now, and write a separate plan, **Server-rendered public pages.md**, if Mason wants more.
  - Google and Bing run JavaScript. Most other crawlers, and link previews, read only the HTML, which 11.1's tags cover.

---

# Phase 12 — Verification

### 12.1 Build and test (Claude, both platforms, reported per platform)
- [ ] server_components: Windows, Linux.
- [ ] knottyyoga: Windows, Linux, pinned; same test names on both.
- [ ] CommunityFinder at the new pin: Windows, Linux.
- [ ] Angular: Windows, Linux (`ng test` as CI runs it); `ng build`; lint.
- [ ] Playwright: layout specs at all sizes, the mobile audit with the hand-started :4300 server, 2.6's double-pay spec, 8.11's Back/Forward spec.
- [ ] `@honuware/ui`: specs, lint, build.

### 12.2 End to end
- [ ] A throwaway database with `--install_schema`, then the scripted checks:
  - signed-out writes refused
  - one membership per person
  - a declined card leaves nothing
  - an invitation with a link registers and takes the seat
  - `site_info` with a cache hit and a 304
- [ ] A pre-release database with `--migrate`: app migrations 0013–0019 and honuware 0004–0006. A second run changes nothing.

### 12.3 Hand-testing walkthrough
- [ ] One walkthrough across all phases, from a blank database, with exact menu and field labels, no URLs. Each phase's own walkthrough is folded in.

---

# Phase 13 — Release to production

- [ ] Phase 1 and Phase 2 are already out (sandbox.14 and .15).
- [ ] The rest goes as **`v1.0.0-sandbox.16`**, by `--migrate`. No reinstall, so the testers keep their accounts (OQ-6).
- [ ] The production content edits, done by Mason after deploy, each with its exact menu path:
  - Getting Started's steps (5.3)
  - the home rows (5.5, 8.5)
  - the bios (8.10)
  - the membership descriptions (5.2)
  - the class descriptions (6.4)
- [ ] Afterwards: Claude probes production signed out, and the testers are asked to retry their journeys. A short list for Thomas, Gifford and Jennifer of what changed against each of their points.

---

# Cross-references

- [[Finishing website]]: the plan this follows; its OQ-43 (a failed password change) is closed by 8.3. D13 there (what a password reset ends) is why D6 keeps server sessions.
- [[Subscriptions- Recurring billing and card management]] §6.4 "Subscription Upgrades": the server half of Phase 3, built without a member-facing page. Proration was dropped there; OQ-2 asks again.
- [[Customer user scenarios]]: scenarios 12 (multi-seat subscription), 31–33 (upgrade, downgrade, reactivate) are what Phases 3–4 make reachable.
- [[Security Review]]: Phase 1 and 10.5 extend it; the CSRF design (8.x there) is kept, with `delete_item` moved off GET.
- [[Tenant Theming and Branding]]: `site_info` and the runtime fonts that 7.2–7.4 change.
- [[Componentizing the frontend]] and [[Splitting the server up into components]]: where the honuware halves of 1.2, 4.5, 7.2, 9.3 and Phase 10 live.

---

# Open questions

*Each has a default the plan is written against. Answer under the question; "→ Plan" will say what changed.*

### Still open

**OQ-1 — One membership per person?** *(D2, 2.2)*
- **Default:** a person can be the payer of only one current membership. "Current" includes one that's cancelled but running to the end of its month, and a change scheduled for next month. Someone who holds a seat on another person's membership is asked "You're already on Thomas's Gold Couple's membership. Buy your own anyway?" and may go ahead.
- **Alternatives:** refuse the seat holder outright; or allow two memberships if they're different tiers.
- Mason- I actually have had people who were paying for a poor friend's membership so I don't think that we should only allow a single person to just pay for one membership. However, I would like to have the UI explicitly warn them that they are already paying for a membership and have to acknowledge this before paying for a second.

**OQ-2 — What does an upgrade cost?** *(3.1)*
- Today the server charges the **whole** month's price difference, whatever the date.
- **Default:** prorate it by the days left, with the same rule the first month already uses. Silver → Gold on the 20th of a 30-day month costs a third of the difference.
- **Alternative:** keep the full difference.
- Mason- let's prorate.

**OQ-3 — Downgrading to fewer seats** *(3.1)*
- Gold Family (3 seats) → Gold Couple's (2): who loses their seat?
- **Default:** the payer keeps theirs, then seats are kept in the order they were given. Whoever's left off gets an email, and so does the payer. The confirm screen names them first.
- **Alternative:** the payer must remove someone before the downgrade can be scheduled.
- Mason- it would be good to prompt who to remove.

**OQ-4 — The "you just bought this" window** *(2.5)*
- **Default:** a paid order with the same items by the same person within **10 minutes** asks "Buy it again?".
- **Alternative:** a different window, or none (rely on 2.6's page changes alone).
- Mason- This sounds like a good idea but can we make it configurable in the admin UI but default to 10min?

**OQ-5 — The testers' extra memberships on the live site** *(2.8)*
- They are sandbox charges, but the billing run will keep charging the duplicates every month.
- **Default:** before sandbox.15, Claude lists them and Mason cancels the extras under **Manage Products** ▸ **Subscriptions**.
- **Alternative:** leave them; or reinstall production (which also removes the testers' accounts).
- Mason- This is the Square sandbox and basically monopoly money that is all going to get thrown away so I don't care as long as we make it hard for them to get in this situation after we do these updates.

**OQ-6 — Migrate or reinstall production?** *(D8, Phase 13)*
- Finishing website reinstalled production to get its content in. The testers now have accounts and history there.
- **Default:** migrate, and make the content changes by hand from a numbered list.
- **Alternative:** reinstall from the seed, and ask the testers to register again.
- We are going to fully reinstall from seed. This is extremely test data that needs to be thrown away.

**OQ-7 — Mason's and Gifford's bios** *(8.10)*
- Thomas: lead with your own teaching and training, not Ana Forrest's name; Beth's is the model. He says Gifford's undersells him too.
- **Default:** wait for your text, and Gifford's. Claude doesn't invent experience.
- Paste them here when ready.

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

**OQ-9 — New copy you need to write or approve** *(5.2, 5.3, 6.4)*
- The new-student FAQ: physical requirements, what to wear and not wear, what to bring.
- Mason- Knotty Yoga is physically rigorous and students should be passably fit, free of substantial injury, and excessive extra weight will make classes challenging and hard to keep up with the group. Contact us for suggestions of one on one training if you do not meet this criteria. You do not need a yoga mat as we have padded, nice traction floors. Please wear fitted gym clothes that are not baggy to avoid getting caught in equipment. Generally tights, leggings, or sweat pants are good (preferably grippy cotton) and a fitted tshirt or tank top. Guys wear tight underwear that keeps things snug. No watches, metal, zippers, or jewelry. Water bottles strongly encouraged.
- Longer Silver and Gold descriptions.
- Mason-
	- Silver- This is our introductory option that features a selection of our more beginner friendly and accessible options. I would start here until instructors suggest you move on to gold and you feel ready. Part of doing acrobatics is learning terminology, body positions, and learning to move and organize yourself in a new way. These classes are foundational for things like aerial, partner acro, and handstand training.
	- Gold- These are more advanced classes that also feature elements from circus acrobatics. This tier also includes some specialty classes for handstands, aerial, and partner acrobatics.
- A line on each beginner class saying it prepares you for the others.
- **Default:** Claude drafts each from your class descriptions and the About page, marked **DRAFT** here for you to edit. Nothing ships until you've approved it.

**OQ-10 — The "Our" rule** *(8.4)*
- **Default:** no "Our" in the menu, and each page's title matches its menu label (**Classes**, **Services**, **Instructors**, **Location**, **Providers**, **Events**).
- **Alternative:** "Our" on every page title and none in the menu; or "Our" everywhere.
- Mason- I like the dropping our idea.

**OQ-11 — "Services" for massage** *(8.6)*
- Gifford wondered if people would look for massage under "Services".
- **Default:** keep **Services**, and list the providers on it. The home page already advertises massage.
- **Alternative:** rename it **Massage**, or **Massage & Bodywork**, while massage is the only service.
- Mason- We will soon offer more than massage so lets keep it as services.

**OQ-12 — Gallery in the top menu** *(8.4, Gifford)*
- **Default:** yes, next to **Blog**, and out of **About**. Gifford expects it to carry event photos over time, as the blog does.
- **Alternative:** keep it under **About**.
- Mason- There are already enough top level menu items. Let's keep it under about.

**OQ-13 — A Classes menu** *(8.4, Gifford)*
- **Default:** **Classes** opens **All Classes**, **Weekly Schedule** and **Calendar**. The buttons on All Classes stay as shortcuts.
- **Alternative:** keep a single **Classes** item and make the two buttons more prominent.
- Mason- I don't like making this a menu item. I think the issue is that if you have created an account but not bought a membership, the My Schedule view shows nothing, which is confusing. I think we should default to Full Schedule view if they don't have a membership and then they see the LOCK icon with Requires a membership for the upsell.

**OQ-14 — See the subscribe page before signing in** *(5.2, Gifford)*
- **Default:** yes. A visitor sees the tier, price, start month and first charge, and signs in or registers at **Subscribe**, coming back to the same page.
- **Alternative:** keep sign-in first.
- Mason- I don't fully understand this item. Can you explain it more fully and what I am choosing between?

**OQ-15 — One / Two / Family** *(5.2, Thomas)*
- **Default:** a toggle over the cards, starting on **One**, with "Couples save $X a person each month" on **Two** and **Family**.
- **Alternative:** keep all five cards on show, adding only the saving line.
- So there would just be a gold and silver option but have a Single, Couples, and Family options as what? Radio controls or separate buttons for each option that list the savings?

**OQ-16 — The Manage Products groups** *(9.2)*
- **Default:** the six groups in 9.2's table.
- Say if a tile belongs elsewhere, or you'd like fewer groups (Thomas suggested about five).
- Mason- I like this grouping.

**OQ-17 — What the business overview counts** *(9.1)*
- **Default:**
  - **Monthly recurring revenue:** the sum of current memberships' monthly prices.
  - **Next month's forecast:** that, less memberships cancelled before the 1st and plus scheduled changes.
  - **Last month's revenue:** completed payments less refunds, by payment date.
  - **Utilization:** check-ins ÷ capacity (past) and plans + bookings ÷ capacity (next 4 weeks).
  - **Underused / full:** a slot under 40% / at 100% in most of its weeks.
- Say if you track any of these differently.
- Mason- This all sounds great.

**OQ-18 — A visiting teacher without an account** *(9.5)*
- **Default:** a **Led by** field: a name shown on the workshop, stored on the session (one new column). An instructor with an account is picked as today.
- **Alternative:** require an account (a quick account by staff), or put the name in the title.

**OQ-19 — Product categories** *(9.7)*
- **Default:** group by the existing **kind** (Memberships, Classes & drop-ins, Events & workshops, Services), plus a search box. No new column.
- **Alternative:** a free category field you fill in for each product.

**OQ-20 — How far "restore" goes** *(10.4)*
- **Default:** every change made through Manage Data (and the pages that use the same generic endpoints) is recorded and can be restored one row at a time. Changes made by dedicated pages (subscriptions, bookings, payments) aren't, since they have their own records.
- **Alternative:** record those too: larger, and touches every helper.

**OQ-21 — Server-rendered pages** *(Phase 11)*
- **Default:** titles, descriptions, link-preview tags and a sitemap now (11.1). Full server rendering becomes its own plan, if you want it after seeing 7's speed work.
- **Alternative:** plan server rendering now.

**OQ-22 — One font address** *(7.4, D7)*
- The fonts are fetched twice: the fixed weights in `index.html`, then the seeded variable weights (`wght@100..900`) after start-up.
- **Default:** both use the seeded address (variable weights), so the second is a cache hit. Critical CSS stays inline.
- **Alternative:** drop the `index.html` fonts and rely on the runtime ones from the cached site config (7.3).

**OQ-23 — "Build up to it in…"** *(6.4)*
- Nothing in the data says which class prepares you for which. The default shows the **predecessor** class and the **required skills**, and names Beginner Knotty Yoga as the way to build up.
- **Default:** the "build up" class is a per-class setting (**Prepares you for**, a list of classes, one new table), so it isn't hard-coded. The seed's pairs are in 6.4; correct them if they're wrong.
- **Alternative:** leave it to each class's description (no schema change).

**OQ-24 — A site search** *(Gifford, "not a strong recommendation")*
- **Default:** not now. The site has about twenty public pages, and Phase 8's menus make them easier to find.
- **Alternative:** a search over classes, instructors, events and blog posts.

**OQ-25 — The favicon** *(8.2)*
- **Default:** made from the logo in the brand red.
- **Alternative:** Ryan designs one; drop it in the Photos folder.

**OQ-26 — When does a class leave the sign-up lists?** *(6.2)*
- **Default:** when it starts. **Today's Classes** keeps it until it ends.
- **Alternative:** when it ends, everywhere.

**OQ-27 — Subscribing for a later month** *(2.3)*
- Today it's free that month (F4).
- **Default:** charge on the 1st of that month, as the page says, with nothing granted before then.
- **Alternative:** charge now for that month.

**OQ-28 — An email on every weekly-plan tap** *(6.1)*
- Adding a class to the weekly plan sends a confirmation email each time, so five taps make five emails.
- **Default:** stop it; the weekly digest lists the plan.
- **Alternative:** keep it.

**OQ-29 — Which control did Thomas tap on Our Classes?** *(6.3)*
- The **My Schedule** toggle there means "classes I can attend", not "my weekly plan". A planned class shows a small checkbox. It's unclear which one let him sign up again.
- **Default:** 6.3 fixes both readings. If you can ask him, his answer goes here.

**OQ-30 — Database connections per site** *(7.6)*
- **Default:** 4, so four requests run at once. The t3.small and the RDS instance have room; to confirm against RDS's connection limit.
- **Alternative:** 2.

### Resolved

*None yet.*