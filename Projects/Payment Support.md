---
fileClass: Project
Category: Implementation
Status: Active
Author: Mason Bendixen
Reviewers: 
Date: 12/18/2025
Version: 0.1
tags: 
---
# Overview
Need to create payment support for the website. Current plan is to use square. Let's do a thin slice through the system just to take payment for an intro workshop.

# Background Research
- They have a Web Payments SDK where they provide controls that collect the credit card and an amount and then give you a token you send to the server to synchronize. The controls appear on your site but are hosted by them and they handle collecting the credit card and taking the payment.
	- Video and info here:
		- https://www.youtube.com/watch?v=DNMXYO9JjGU
		- https://developer.squareup.com/docs/web-payments/overview?utm_source=chatgpt.com
		- Requires the use of secure contexts
			- [https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Secure_Contexts](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Secure_Contexts)
		- Must be used along with the Payments API and Customers API
			- https://developer.squareup.com/reference/square/payments-api
			- https://developer.squareup.com/reference/square/customers-api
		- It works like:
			- Configure the Web Payments SDK client library with your application to render a payment method form and generate a payment token.
			- Configure the Payments API, or another backend service, to take the payment token and process the payment.
	- Web Payments SDK Quickstart
		- https://developer.squareup.com/docs/web-payments/quickstart
	- Web Payments SDK showcase
		- https://square.github.io/web-payments-showcase
	- It is granular and you only need to write code for the payment methods you support (card, ACH, Google pay, Apple Pay, etc)
	- Promise based pattern using async / await
	- The payment tokens for any of the supported methods all work the same on the server
	- Can use the Cards API to store a card on file for a customer
		- https://developer.squareup.com/docs/cards-api/walkthrough-seller-card
	- To create a customer profile for a payment, you need to collect one of these:
		- First name (we have)
		- Family name (we have)
		- Company name (we won't have)
		- Email address (we have)
		- Phone number (we don't have)
		- Can use these to create a customer profile
			- https://developer.squareup.com/docs/customers-api/use-the-api/keep-records#create-a-customer-profile
	- There is a Web Payments SDK quickstart application
		- https://github.com/square/web-payments-quickstart
		- https://github.com/square/web-payments-quickstart/blob/main/README.md
	- Here are some assorted scenarios
		- Customize the Card Entry Form
			- https://developer.squareup.com/docs/web-payments/customize-styles
		- Verify the Buyer When Using a Payment Token
			- https://developer.squareup.com/docs/web-payments/sca
		- Add a Content Security Policy
			- https://developer.squareup.com/docs/web-payments/content-security-policy
		- Exception Handling
			- https://developer.squareup.com/docs/web-payments/exception-handling

# Working on the design 1/12
- Copilot prompt:
> I have a web server written in C++ with Crow with a postgres backend that I access through the c++ client library. I build with CMake and use Conan to pull down all the libraries. The front end is written in Angular with Typescript. I have a people table with an id field. I need to take payments. I'm thinking of using Square with the web payments SDK on the client and the Payment SDK on the server. I also need a payments table on the server to track payments for a given user. I want to put together a list of steps and tasks I need to do to complete this task. I need to figure out what tables I need, what I need to setup with square, what APIs to call, and how to test this in development mode versus production. I also don't currently have HTTPs enabled on the server so I need to figure out what is needed to set that up for both production and development. I also need to support callbacks from Square so I need to figure out how to expose my server running on my local machine on the public internet to test the callback support.
- Follow up prompt
> Yes, I would like subscriptions and cards on file. Can you give me more examples of the APIs on the client and server from square that I need to call. Maybe write some sample code or point me to documentation.
- Follow up prompt
> Yes, I would like multiple tiers and add ons for subscriptions. Can I also keep a card on file for a customer that I use for random charges besides subscriptions? Can you give me all the steps, sample code, links, and database tables in one response?
- Follow up prompt
> I'd like a list of offerings that are not related to subscriptions. Many of my services like personal training or massage will not be subscription based so I don't want subscription id to be in the payments table. I would also like a table of one time payment offerings and to track bookings for those separately as well as subcriptions. I'd like each of those to cause a payment to be created but would like information about them to be tracked in another table as well (in other words, have a table that tracks a subscription id to a payment id and another that tracks a one time payment id linking that to a payment id). I also would like the timestamps to use a bigint with microseconds since epoch. I've found that converts nicer to a string and is easier to work with from code. Please regenerate all the steps with these changes.
- I need to allow one customer / person to make a payment for another user's service. For example for a gift or I have spouses and parents pay for their spouse or child. I also have couple's and family memberships where there is one payment made that provides a feature for multiple users. So basically, I need to decouple the person id of who is making the payment from the service. I also need to decouple a service from an individual user to I need to have a number of people allowed for a specific service that is paid for and then have up to that number of people with instances of that service. For instance, Unlimited membership for Februrary for three different people. Can you recreate the whole document in progress with these changes?
- Follow up prompt
> This is looking a lot better but I need some more changes. I have a permissions table with an id column. I would like to allow different prices for the same product based on the permission. For example, I have silver, gold, and platinum monthly memberships. If someone has one of those memberships, they would have the silver, gold, or platinum permission and the price is lower for massage or personal training based on the permission or different services might not even be offered for people not at a given permission. I'd also like a text description for each payment so that a user can see payments made in their portal to see what money they have spent, when, what for, and to possibly request a refund. I noticed that your make all string table entries text instead of varchar? Is there a compelling reason to do this as text versus varchar? If not, can you switch over to varchar? Also, I have been using auto incrementing integers for ids instead of a 64 bit into or UUID. Is this going to cause a big issue?
- More details in prompt
> I have a people table with an id. I have a roles table with an id as well. I have a role_assignments table with an id but also a person_id and a role_id that assigns users to roles. I have a permissions table with an id. I have a role_permissions table with an id but also a role_id and permission_id that let's you assign permissions to roles. Doing a join on these can give you the effective permissions for a given user. I will only give platinum permission to platinum members and not give them gold and silver permission even though platinum is effectively a superset of gold and silver because I really only want the user to see the price specifically for platinum and not the more expensive offerings. There will also be an public permission that is essentially available to everyone who isn't a monthly member and is a default granted permission. When entering prices for services, there should be entries for each entry. I would like this to be as light weight as possible and to decouple the pricing per role as much as possible from the actual product being sold. I'd also like to support vouchers or other methods that essentially make a payment without any money. Any thoughts on how to support that?
- Copilot query
> I'd like to follow up on the treating public as implicit versus data driven. I suspect I'd like to have public be something explicit. For instance if massage is $180 for the public, $160 for silver, $150 for gold, and $140 for platinum, I don't want the three membership tiers to see the public pricing. I'm not sure without having an explicit entry for public, I can handle things that are priced the same for everyone (but I need to add entries for public, gold, platinum, and silver to do that) versus removing public as an option if a lower price is available. I'm not sure I need an environment in price_schedules but I like the price schedules concept since it will simplify price increases. It feels like product_prices should have a price_schedule_id. I don't think we should have an environment on product prices. It also seems like price schedule should not have currency. It should just be a time window for prices. I don't know how I feel about cadence on product_prices. I feel like I'd rather just have separate tables for things like monthly versus annual cadence products because I still need to track actual payments and what period for which the purchase belongs to. I also don't really want to do an is_available on product_prices. I think that would better be handled by permissions. I like the provider with the amount_cents and covered_amount_cents since that could be used to cover cash sales as well as coupons and sales.I like the voucher / voucher_redepmtion concept.
- Copilot follow up
> I definitely DO NOT want to add a pricing_permission_id to the people table. This is bad denormalization on my most used table. I can also do that easily via a view and keep the view in memory in a mem cache if I need to speed things up. I'm not sure why I need to have a product_permission_visibility table at all. If there are entries for a given product for a given permission, they can buy it. Otherwise they can't. That seems simpler than add a NACK. I think it will be better to add product_prices entries for all different permissions. It simplifies things a lot and isn't particularly hard when updating prices to add four entries entries programmatically. The matrix seems to be optimizing for the case that there are multiple products that will all have the same pricing. I don't think that will be a common scenario. I think the most common case for sharing would be a product that has the same pricing regardless of permission so I would need to replicate entries for every permission including the psuedo everyone permission. I think that the better way to handle this would be to drop having an everyone permission and support a null entry for permission. Null permission would then apply to all permissions for which there isn't a specific entry. So if there is an entry for null and nothing else, all permissions including non members would get that price. Conversely, if I had a null entry as well as one for platinum, non members and silver and gold would get the null pricing and just platinum would get the bespoke price. I like having the subsciption plan grant a permission and having seats and cadence. What is portal_note on payments? I see that subscription has a product id so this would allow one to be purchased. Since subscriptions grant permissions, their pricing could have a null permission entry to indicate that this pricing applies to everyone. How do we track binding the seats of a purchased subscription to individual users? 
- Clarifying
> Thanks for the note on portal note. It would be nice to be able to add a note about this specific purchase from the app. Can we add a trigger for either entitlements or entitlement_assignments to make sure that there aren't more assignments than there are seats for a given entitlement? What is purchase_item_id on entitlements? What is provider_environment on entitlements? I'd like to see products again so I can see how entitlements and products or purchases are linked. In particular, I'd like to have the number of seats be in one place. It feels like a mistake to just put seats on subscriptions. For instance, I could purchase a couples massage where there would be two people but that is still a one time purchase, not a subscription. As far as permissions and how they are calculated, I think it would be better to have an effective permissions. I would still have users, roles, and permissions. There would still be bindings from users to roles and then permissions based on roles. For instance, admin type permissions or permissions for teachers are very independent of purchase. So would being staff. But I can add code when fetching permission to join the people, role assignments, and role permission assignments to calculate permissions and then I would add in the calculated permissions based on subscription and just add those to the list. I agree that trying to add and remove people based on payment and cadence ending would be messy so calculating it seems like a better idea.
- Answering questions
> I'd like to drop provider_environment on entitlements. Does it make sense to move seats_total from product_entitlement to product? To me couple's massage or couple's membership. For validity_kind on product_entitlement_rules, a lot of my services are monthly so I imagine that would be indicated here. 
- Clarifying
> Okay, I'm okay with not moving seats_total to products. I like the making period a first class rule idea. I would like monthly to mean calendar month versus 30 days from activation. It just simplifies things. Especially along year boundaries.
- Clarifying
> I'm fine with using UTC. I'm already doing micros since epoch and I'd like to be independent of a specific time zone. For the month, I like the idea of defaulting to the current month but allowing specifying a month. I frequently let a new person buy a subscription for the next month but get the rest of this month free. I like the idea of a specific month option and making changes to support that. I prefer the Option 3 Postgres computes option for month window computation. 
- Refining
> For the rest of the month question, I like the Option A. I like the store “activation_us” suggestion. 
- Generating requirements
> I'd like to back up a bit and start working on a design document. I'd like an overview, requirements, assumptions, high level design, low level design, and althernatives explored. Let start with listing requirements since that should probably guide the rest of the document.

# Design is out for review
- [[Payment Design Document]]

# What I'm working on 1/27
- Migrate existing endpoints to JSON error responses (new `ErrorResponse` helper)
	- Created Claude design for this
	- [[JSON Error Response Handling]]
	- This is completed.
- Add base tables: products, price_schedules, product_prices, product_entitlement_rules
	- Let's create a Claude design for this
	- [[Payment base SQL Tables]]
	- This is complete

# What I'm working on 1/28
- Let's add table helpers for the new tables
	- Created Claude design for this
	- [[Payment base table helpers]]

# What I'm working on 1/29
- User scenarios for Payment Design Document
- Created document to work on this for claude
- [[Customer user scenarios]]

# What I'm working on 2/2
- Set up Square sandbox credentials and secrets management
	- Creating document to work on this with Claude
	- [[Square credentials and Sandbox setup]]

# What I'm working on 2/3
- Product browsing and quoting endpoints
	- Created document to work on this with Claude
	- [[Product browsing and quoting endpoints]]
	- This is all complete.

# What I'm working on 2/6
- Purchase creation with server-side pricing
	- Created document to work on this with Claude
	- [[Purchase creation with server-side pricing]]

# What I'm working on 2/18
- [[Purchase creation with server-side pricing]]
	- Still working on this. Have client side purchase support
	- Needed to get basic certificate support working
		- [[HTTPS Support]]
	- This is all complete with user purchase, dashboard, and purchase history.
	- Thin slice is complete!