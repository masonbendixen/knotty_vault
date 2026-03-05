---
fileClass: Project
Category: Implementation
Status: Active
Author: Mason Bendixen
Reviewers: 
Date: 12/10/2025
Version: 0.1
tags: 
---
# Overview
This is the client side piece to [[Authentication, security]]

# Background Research
- We have implemented server side authentication. Now we need to implement the client side portion.

# Requirements
- Register workflow (client side)
	- Create page to do client login
	- Direct to page to check email
	- Done but redirects to login for now
- Login workflow (client side)
	- email / password with remember me
	- Show user name and maybe picture if we enable photo support
	- Done without photo support
- Update UI to show pages based on permissions
	- Hide dashboard for non admin users
	- Done as things currently stand
- Remember me workflow (client side)
	- Call remember me URL and then function like login workflow if successful
	- Done with client side testing but need server side
- Change email, password, and name (client side)
- Logout (client side)

# What I'm working on 12/11
- Getting the client back up and run
	- Can run locally with ng serve -c local
	- The header service is in:
		- ui\src\app\shared\services\header\header.service.ts
		- Delegates most of the heavy lifting to mockHeaderResponse
		- There is an initiateHeaderButtonAction with SignIn/SignOut that delegates to AuthService
		- There is also an observable bool for headerMenuOpen
	- mockHeaderResponse
		- ui\src\app\shared\services\header\mockHeaderResponse.ts
		- Has a function mockHeaderResponse that returns an Observable\<HeaderData\>
	- header.types
		- ui\src\app\shared\services\header\header.types.ts
		- There is a HeaderBaseButton that gets augmented to a discriminated union into HeaderButton. There is an interface HeaderData that just contains a menu that is an array of HeaderButton
	- header.component.ts
		- ui\src\app\shared\components\header\header.component.ts
		- Has the HeaderService and AuthService
			- Subscribes to AuthService.authData$ and refreshes based on being authenticated and the role
			- Subscribes to HeaderService.headerData$ and refreshes the header data that is copied locally
		- header.component.html does the heavy lifting of controlling the display of the menus
	- How does Login work
		- loginButton: HeaderButton
			- kind: HeaderButtonKind.Action
			- action: HeaderButtonAction.SignIn
		- There is a userDropdown
			- This has user specific information
			- There is also a Sign Out button with the SignOut action
		- Depending on whether we are authenticated or not, we add userDropdown or loginButton
		- In the HTML, for a HeaderButtonKind.Action, we call initiateHeaderButtonAction(button.action)
			- This calls HeaderService.initiateHeaderButtonAction(action)
			- Based on SignIn/SignOut, this calls AuthService.signOutUser/signInUser

# What I'm working on 12/13
- Get the SignIn to take me to a login page
	- We need a login component
	- The AuthService needs to take in a Router that can navigate to the right page OR I need to change it from an action to just a URL to the login page
		- The URL solution seems like a better idea
	- The plan
		- Make the Sign In be an internal link
		- Change it in the mockHeaderResponse.ts
		- Add auth/components/login
			- Make this the login form with:
			- Title: Sign In
			- Username field with label
			- Password field with label
			- Remember me checkbox
			- Need to create account URL
			- Have the form handler just show a message box for now with the user information
		- Wire this into auth.routes.ts
		- Change the entry for sign in to be an internal link to this component
	- The implementation
		- Created branch: client_auth_1_sign_in
		- Copilot prompt
		> I'd like to make a standalone angular component called login under src/app/auth/components. Please create the folder and then the component.ts, component.scss, component.spec.ts, and component.html with loging prepended to all of these. This will be a reactive form with the title "Sign in" and then an edit box with the label Username:, a password edit box with the label Password:, a button labelled "Sign in" and then a hyperlink after that says "Need to create an account?". Please give the edit box for the username the id username, the edit box for the password the id password, and the id for the submit button the id signIn. Inside the component's ts file, please have the event handler for the submit button build the text "Username is {username} and password is {password}" and then show a message box with this text in response to the button being pressed. Please create unit tests that fill in the username and password and verify that a message box is displayed with the correct values and then dismiss the message box.
		- Done
- Add a register component that registers and account with the system for email validation
	- The plan
		- Add auth/components/register
			- Title: Register
			- email field with label
			- First name field with label
			- Last name field with label
			- Password field with label
			- Reenter password field with label
			- Have a handler on the fom that just shows a message box with the user information
		- Wire this into auth.routes.js
		- Change the URL on the login page to point to this
		- Verify that the passwords match
	- The implementation
		- Created branch: client_auth_2_register
		- Copilot prompt
		> I'd like to make a standalone angular component called register under src/app/auth/components. Please create the folder and then the component.ts, component.scss, component.spec.ts, and component.html with register prepended to all of these. This will be a reactive form with the title "Sign in" and then an edit box with the label Email:, a password edit box with the label Password:, a reenter password edit box with the label Reenter password:, a button labelled "Register". Please give the edit box for the email the id email, the edit box for the password the id password, the id for the second password edit the id password2, the id for firstName being first_name, the id for last name being last_name, and the id for the register button the id Register. Inside the component's ts file, please have the event handler for the register button build the text "Email is {email}, first name is {first_name}, last name is {last_name}, and password is {password}" and then show a message box with this text in response to the button being pressed. Please create unit tests that fill in the various fields verify that a message box is displayed with the correct values and then dismiss the message box. Please use angular material components for the controls. Please add validation that makes sure both passwords are the same but only show the validation error if both edit boxes are dirty and not matching.
		- Done

# What I'm working on 12/16
- How do I wrap the network for other things
	- For local:
		- environment.ts
			- File replacement
				- "replace": "src/environments/environment.ts"
				- "with": "src/environments/environment.development.ts"
			- There is a struct environment that has the field production and this is set true or false
		- ServerAccess
			- File replacement
				- "replace": "src/app/portal/services/network_abstraction/ServerAccess.ts"
				- "with": "src/app/portal/services/network_abstraction/ServerAccess.mock.ts"
			- There is a ServerAccess interface that is provided by SERVER_ACCESS_IMPLEMENTATION_TOKEN
			- ServerAccessProxy implements ServerAccess but forwards to the implementation of the interface via the token
			- under network access, ServerAccess implements these calls via HTTP wrappers to the calls. I need to make the calls with some extra attributes
			- under ServerAccessMock.ts, I have mocked out versions of these APIs.
			- I need to add:
				- /api/register/first_name/last_name/email/password
					- API has no response to map to bool return on success
				- /api/login (email / password / bool remember)
					- On success, returns a redirect to main web page
				- /api/me
					- API has no response to map to bool return on success
				- /api/remember
					- API has no response to map to bool return on success
				- /api/get_user_info
					- Returns JSON
			- Need to pass {withCredentials: true} at last param to get or post
			- This will cause redirects not to automatically redirect {redirect: 'manual'}
			- The error handling to subscribe is error: (err: HttpErrorResponse) => {}
				- err.status - HTTP error code
				- err.statusTest - code converted to string
				- err.error - server provided body (if any)
				- err.url - request URL
				- err.message - client-side summary message
- Task: add various methods needed to ServerAccess and implemented classes
	- The plan
		- The methods to add:
			- register(firstName: string, lastName: string, email: string, password: string) : Observable\<void\>
			- login(email: string, password: string, remember: bool) : Observable\<void\>
			- me() : Observable\<void\>
			- remember(): Observable\<void\>
			- getUserInfo(): Observable\<UserInfo\>
		- Default behavior for mock is that user is logged in as Mason Bendixen with admin access and all the permissions.
			- Roles: admin, user, teachers
			- Permissions: admin_portal
			- We have the state:
				- deviceToken: false
				- sessionToken: true
				- UserInfo -> set to values
			- Logout clears all of these
			- me and remember will fail until login has been clicked
			- Register will change the values for email, first name, last name
			- Login will set the values to the default state with admin permissions but use the values in user info and set session and device token to correct values
	- The implementation
		- Created branch: client_auth_3_auth_server_access
		- Copilot prompt:
		> In the file: src\app\portal\types\ServerAccess.ts, I added these methods: register, login, logout, me, remember, and getUserInfo. src\app\portal\services\ServerAccess.ts implements the interface to forward to the impl contained member. Please do the same for all of these new methods. src\app\portal\services\network_abstraction\ServerAccess.ts implements this interface for the network. Please add network calls for all of these new methods. For all of the get/post methods, please pass in {withCredentials: true} as the last parameter to the get or post. register is a get with the url /api/register/first_name/last_name/email/password. Please build this URL and make the call. login is a post the the url /api/login with LoginInfo as the post body. logout is the URL /api/logout and is jut a post with no body. me is the URL /api/me and is a post with no body. remember is the URL /api/remember and is a post with no body. get_user_info is the URL /api/get_user_info and is a post with no body but returns an Observable\<UserInfo\>. For login, please also add {redirect: 'manual'} to the withCredentials to prevent automatic redirect. src\app\portal\services\network_abstraction\ServerAccess.mock.ts is another implementation of the interface that we need to add the new methods to. This is for the local, disconnected user case for angular where we don't connect to the server and are doing a fake implementation for ng serve local testing. Please add two UserInfo fields (userInfoDefault / userInfoCurrent). Have them initially have first name Mason, last name Bendixen, email masonbendixen@gmail.com, roles [admin, user, teachers], and permissions [admin_portal]. Have a bool field deviceToken that is false and another bool field sessionToken that is set to true. Have register change the userInfoDefault to replace the first name, last name, and email with the passed in values. Have login replace userInfoCurrent with a copy of userInfoDefault and set sessionToken to true and deviceToken to the value of remember. Have logout set userInfoCurrent to null and deviceToken and sessionToken to false. Have me() return normally if sessionToken is set but throw an error simulating an HTTP 401 error if sessionToken is false. Have remember() return normally if deviceToken is set but throw an error simulating an HTTP 401 error if deviceToken is false. Have getUserInfo return userInfoCurrent or throw an error simulating an HTTP 401 error if the field is null. Add tests to the spec file at the end of the file that by default getUserInfo returns the userInfoDefault and that me() returns successfully and that remember() returns a 401 error. Make a call to logout and verify that me, remember, and getUserInfo all get a 401 error. Make a call to register with new values and then call login with remember false and then getUserInfo. Validate that getUserInfo has the updated first name, last name, and email and verify me() succeeds but that remember() returns 401. Repeat the last test with remember set to true and verify all the same things except that remember succeeds as expected.
	- Done
- Took Lucas's changes but need to do cleanup
	- Created branch: client_auth_4_lucas_cleanup
	- Submitted
	- Created branch: client_auth_5_register_component_style
	- Submitted 
- Existing AuthService
	- signInUser - called by header.service.ts but no longer used
	- signOutUser - called by header.service.tst
	- authData$ - subscribed to in header.component.ts in ngOnInit and used to trigger refreshHeaderData
	- authData - non observable
		- calendar.service.ts
			- Used for the default calendar view? Why is there a defaultCalendarView in the auth settings?
		- month-view.component.ts
			- Why is there an authData accessor property on MonthViewComponent that just exposes the same property on the auth service?
			- There is an isAdmin() function on the component that accesses the role of the auth data
		- week-view.component.ts
			- Same stuff as month-view.component.ts
		- auth-guard.ts
			- Protects non admin users from seeing a lot of the auth related pages
			- 
- Need to make an AuthService that has:
	- type AuthData = 
		- {isAuth: false;}
		- | {isAuth: true; firstName: Mason; lastName: Bendixen; isAdmin: bool, roles: [], permissions: []};
	- Use a behavior subject that defaults to isAuth false
	- Leave the current behavior subject and both observable and static information
	- updateAuthData(userInfo: UserInfo)
		- src\app\portal\types\ServerAccess.ts is where UserInfo is defined
	- tryTokenLogin() : Observable\<bool\>
		- Try calling me() if that fails, call remember(), if that succeeds, call me() if that succeeds, then call getUserInfo and update the behavior subject
	- login(email: string, password: string, remember: bool) : Observable\<void\>
		- On success, call getUserInfo and update the behavior subject
	- logout() : Observable\<void\>
		- Call logout and when that succeeds, set the observable back to not auth
	- register(firstName: string, lastName: string, email: string, password: string) : Observable\<void\>
	- Created branch: client_auth_6_auth_service
	- Copilot prompt:
	>I need to add a method to AuthService called udpateAuthData(userInfo: UserInfo). UserInfo is defined in src\app\portal\types\ServerAccess.ts. Build an AuthData with isAuth: true, firstName: userInfo.first_name, lastName: userInfo.last_name, email: userInfo.email, roles: userInfo.roles, permissions: userInfo.permissions, isAdmin is true if userInfo.roles contains the string 'admin'. Call \_authDataSubject.next on this AuthData. To the constructor, add a ServerAccess from src\app\portal\services\ServerAccess.ts that is injected via SERVER_ACCESS_TOKEN. Add a method called tryTokenLogin(): Observable\<boolean\> and have it call ServerAccess.me() and subscribe to the Observable. If that publishes, call ServerAccess.getUserInfo and when that publishes, call updateAuthData on the result as a side effect. If the first call to me() fails with HTTP status code 401, then call ServerAccess.remember(). If that publishes, call ServerAccess.me(). If that publishes, call ServerAccess.getUserInfo() and when that publishes call updateAuthData with the result if that publishes as a side effect. The method returns an Observable\<boolean\> and should return an Observable that is built on the subscriptions to these other chained Observables and only returns true if we make it down a pathway all the way down to a side effect call to updateAuthData. In the other cases, we should return false to the subscribers. Add a method called login(email: string, password: string, remember: boolean): Observable\<void\>. This should make a LoginInfo and make a call to ServerAccess.login(). We should return an Observable that listens on if the login observable publishes, triggers a call to ServerAccess.getUserInfo if that publishes, makes a call to updateAuthData with that UserInfo as a side effect, and then sends completion to the subscriber or forwards any errors. Add a method logout(): Observable\<void\> that calls ServerAccess.logout, makes a call to \_authDataSubject.next with the isAuth: false UserData as a side effect and then passes the observable to the user. Add a method register(firstName: string, lastName: string, email: string, password: string): Observable\<void\> that makes call to ServerAccess.register and returns the resulting Observable. Please create an auth.service.spec.ts file next to auth.service.ts for unit tests for AuthService. Please create a ServerAccessMock from src\app\portal\services\network_abstraction\ServerAccess.mock.ts and then pass that into the constructor for ServerAccessProxy from src\app\portal\services\ServerAccess.ts that you pass when creating AuthService to the constructor. I will describe tests that verify that authData$ gets a value, in all these cases, I mean for you to subscribe to the observable and then make sure that the value requested is publishes and matches the stated value. Create a test to verify that Subscribing to authData$ returns the AuthData with isAuth false and that calling authData returns the AuthData with isAuth false. Create a test to verify that calling tryTokenLogin and then subscribing eventually publishes true and that both authData$ and authData get the default user info from ServerAccessMock. Then call ServerAccess.register with a new first name, last name, and email followed by a call to login after that completes and verify that authData$ and authData get the new values. Add a test that calls ServerAccess.logout and then ServerAccess.login with remember set to true and verifies that calling tryTokenLogin's observable publishes true. Add a test that calls ServerAccess.logout and verifies that calling tryTokenLogin's observable eventually returns false. Create a test that calls ServerAccess.logout but then calls login and verifies after that has published, that authData$ and authData both have the default values from ServerAccessMock. Create a test to verify logout() but making a call to login() verifiying that authData$ and authData have default values and then calling logout() and making sure that authData$ publishes AuthData is isAuth: false and that authData contains the same. Add a test that verifies register() by checking that authData has the default values from ServerAccess mock after calling register with new values but that calling login() after causes them to change to the values passed to register.
- Copilot prompt to inject the AuthService correctly for HeaderService
> Can you alter src\app\shared\services\header\header.service.spec.ts so that it creates an AuthService using the same way as in src\app\shared\services\auth\auth.service.spec.ts so that there is a ServerAccessMock and ServerAccessProxy?
- Done!

# What I'm working on 12/17
- Wire up auth into the UI
	- The plan
		- Make register call register and then take you to the login page
		- Make the login page call login and then take you to the home page
		- Make sure the admin UI shows up
		- Show the username in the upper right  hand corner
	- The implementation
		- Created branch: client_auth_7_auth_wiring
		- Copilot prompt for register
		> I'd like to change src\app\auth\components\register\register.component.ts so that onRegister() no longer builds message and calls alert. Instead, I want to have the constructor now take AuthService from src\app\shared\services\auth\auth.service.ts and make a call to register with the various fields passed in. Subscribe to the observable and then navigate to /p/login. You might need to pass the router in. Please alter the register.component.spec.ts to setup the AuthService like src\app\shared\services\auth\auth.service.spec.ts. Change the tests so that instead of looking for the alert to be called, verify that we get a request to migrate to the correct url /p/login. Do another test that simulates the user filling out the form with values, submits the form, and then you check that we get a request to navigate to the login page. At that point, call login on the AuthService after subscribing to authData$ and verify that authData$ publishes the values entered into the register form.
		- Copilot prompt for login
		> I'd like to change src\app\auth\components\login\login.component.ts so that onSignIn no longer builds message and calls alert. Instead, I want to have the constructor now take AuthService from src\app\shared\services\auth\auth.service.ts and make a call to login. Please add a check box control labeled "Remember Me" after the password field but before the button. Please give it the id remember. Make sure to pass email, password, and the bool to login. After the call to login publishes, please navigate to the homepage "/". You might need to pass in the router to do this. Please alter login.component.spec.ts to setup the AuthService like src\app\shared\services\auth\auth.service.spec.ts. Change the tests so that instead of looking for the alert to be called, verify that we get a request to migrate to the correct url /. Also validate that after login completes, verify that authData on AuthService now has isAuth and isAdmin set to true and verify that the default user information from src\app\portal\services\network_abstraction\ServerAccess.mock.ts is what is in authData.
		- Copilot query to change the prompt for the username
		> In src\app\shared\services\header\mockHeaderResponse.ts, can you change the title for userDropdown so that it looks in src\app\shared\services\auth\auth.service.ts in the AuthService for authData and checks if isAuth is set and then uses first name for the title otherwise the word 'User'
		- Actually, I want to have it automatically update. Here is the copilot prompt
		> In src\app\shared\services\header\mockHeaderResponse.ts, there is a userDropdown with the title 'User'. 
	- Done!

# What I'm working on 12/18
- Get things working on the client with the actual server
	- Default configuration is local
	- Need to run with ng serve -c development to connect to server
	- Need to run create database on server first
- Need to get the database helper running
	- src\db_schema\make_database_info.cpp is missing a lot of db_schema items
		- It has:
			- MakeAllowedTablesTable(databaseInfo);
			- MakeClassesTable(databaseInfo);
			- MakePeopleTable(databaseInfo);
			- MakePhotoInstancesTable(databaseInfo);
			- MakeSourcePhotosTable(databaseInfo);
			- MakeScaledPhotosTable(databaseInfo);
			- MakeAdminTopLevelTablesTable(databaseInfo);
			- MakeAdminColumnDataInfoTable(databaseInfo);
			- MakeAdminColumnFriendlyNamesTable(databaseInfo);
			- MakeAdminTableFriendlyNamesTable(databaseInfo);
	- Fixing this
		- Need to upgrade to a newer VS
			- Notes for upgrading from 17.14.12 to 17.14.23
		- Created branch: client_auth_8_fix_db_schema
		- Copilot prompt
		> In make_database_info.cpp, there are a set of Make{table_name}Table calls. I have a bunch of these that are missing. Can you go through all the headers in db_schema and look for a Make{table_name}Table function and then add all the missing entries to this file in alphabetical order? Please don't change anything else about the file or reformat anything. Please add the missing header includes but also do that in alphabetical order.
		- That isn't really correct for dependencies. Copilot prompt:
		> If you scan the cpp files in db_schema, you will see that some of them reference other tables for foreign keys. For instance, if you look at src\db_schema\role_assignments.cpp, you will see that it has references via AddColumnForeignKeyRef to kPeopleTable and kRolesTable so MakeRoleAssignmentsTable needs to happen after the people table and roles tables. Can you rearrange the Make{table_name}Table calls so that they default to alphabetical order but the dependent tables are after the tables they depend on?
		- Need to do CreateTables correctly. Copilot prompt
		> Can you use the order of tables from MakeDatabaseInfo() from make_database_info.cpp and then modify CreatetTables so that the same tables are present in the same order but calling CreateTable with the right constant.

# What I'm working on 12/22
- Getting an exception when making a call
	- I can send a simple HTML formatted email to masonbendixen@hotmail.com
	- Locations of interest:
		- Sending email:
			- src\util\mail\mail_helper.cpp
		- Unit test that sends email:
			- src\util\mail\mail_helper_test.cpp
		- In register where we call the email building function and send it:
			- src\auth\person.cpp
		- Where we create the email:
			- src\auth\person_verify_mail.cpp
	- Things to try:
		- Try sending the simple test message to gmail in the normal workflow
			- That works
		- Try sending the complicated email to hotmail in the test workflow
	- I need to URL safe encode the base64 encoding and then HTML escape the final result and email
		- Switched to URL safe base64 encoding
		- Things are working now but I need the account_activation endpoint
	- Submit these changes now that the email is getting sent in the correct format
	- Done
- Create account_activation endpoint
	- Copilot prompt:
	> Make a new endpoint for the server called account_activation. You can base it off of endpoints/register.cpp. Create three files names account_activation but with .h/.cpp/\_test.cpp appended for each file. The URL should be /api/account_activation/\<string>/\<string> and the function should be void AccountActivation\(EndpointAuthHelper& endpointAuthHelper, std::string_view email, std::string_view base64EncodedActivationToken\). Pass the email and encoded token to PersonHelper::VerifyPersonEmail. In the test file, create a AccountActivationTest test with the name AccountActivationBasic that is based on endpoints/register_test.cpp and auth/person_test.cpp's VerifyPersonEmailBasic. After completing the workflow, PersonHelper::IsPerson should return true. Add other tests for AccountActivationEmailNotFound and AccountActivationInvalidToken.
	- Created branch: client_auth_9_account_activation
	- src\auth\cookie_manager.cpp SetCookie is setting the readonly parser. Manually set the header

# What I'm working on 12/24
- GetRolesForUser / GetPermissionsForUser throw exceptions. Fix this to just return empty collections
	- For role assignments, copilot query:
	> I changed GetRoleAssignmentsForPerson and GetRoleAssignmentsForRole to no longer throw for no results. Can you modify GetRoleAssignmentsForPersonNotFound and GetRoleAssignmentsForRoleNotFound accordingly to expect and empty collection and implement GetRoleAssignmentsForPersonNoAssignments and GetRoleAssignmentsForRoleNoAssignments?
	- For role permissions, copilot query:
	> I changed GetRolePermissionsForRole and GetRolePermissionsForPermission to no longer throw for no results. Can you modify GetRolePersmissionForRoleNotFound and GetRolePermissionForPermissionNotFound accordingly to expect an empty collection and implement GetRolePersmissionForRoleNoRolePermissions and GetRolePermissionForPermissionNoPermissions?
	- Copilot query to add tests for no roles and permissions
	> Can you implement GetRolesForUserNoRoles and GetPermissionsForUserNoPermissions? Note that unlike the NotFound test cases, both of these should return empty collecitions.

# What I'm working on 12/29
- Let's do a hack so that any name that gets passed in with Mason as the first name gets admin access
	- Got everything working and checked it

# What I'm working on 12/30
- Let make json_util just Json for a namespace
	- Created branch: client_auth_10_json
- Adding the Json variant
	- Created branch: client_auth_11_json_variant
	- Copilot prompt for testing:
	> Can you implement the unit tests for json_value.h inside the file json_value_test.cpp? Can you look at the existing tests files in this directory for an example. Please place everything inside the namespace Json and then an anonymous namespace inside that. Please do not indent the tests within the two namespaces (ie. each function starts at the first character of the line). I'd like the tests to be JsonValueTest and please create a test named member function name with Basic appended for each positive test case. Please create a positive test case for each public method. For special things like constructors and assignment operators, please use Constructor or Assignment for the method name. For overloaded functions, Please use Pascal naming with the type name appended (like ConstructorStringBasic or AssignmentIntBasic). For the complicated things like arrays or compound objects, please add test cases for things like Empty. For things like ToString, please do a variety of tests for the basic types and complex ones.
	- Submitted
- Switch over to using this in REST helper

# What I'm working on 12/31
- Make register workflow redirect to login
	- Created branch: client_auth12_register_login
	- Done
- Add a FromText(string_view jsonText) factory method
	- Created branch: client_auth_13_from_text
	- Copilot query
	> I would like to add a factory method right below FromRValue and FromWValue called static Value FromText\(std::string_view jsonText\). Please place this below both these functions in the header and implementation file and place the test FromTextBasic underneath FromWValueBasic. Use crow::json::load to parse the text into an RValue and then return FromCrow.
- Convert from wvalue/rvalue to Json::Value
	- Created branch: client_auth_14_move_to_json_value
	- Copilot query
	> Search the whole tree and convert all usage of crow::json::wvalue and crow::json::rvalue to util/json_value.h's Json::Value class. wvalue is a writable JSON wrapper and rvalue is a readonly JSON wrapper with more inspection support. Json::Value is a std::variant based class that you helped me write that has better language integration and can support all the needed scenarios. In particular, dump maps to ToString and load maps to FromText. Please skip util/json_util.h/cpp/test.cpp. Obviously don't touch util/json_value.h or test.cpp. It would be great if you can have a list of files that you have modified with apply buttons in the chat pane so I can work my way through applying the changes.
	- Here is the list of files:
		- endpoints/add_item.h/cpp/test.cpp
		- endpoints/add_item_fetch_primary_key.h/cpp/test.cpp
		- endpoints/db_schema.h/cpp/test.cpp
		- endpoints/delete_item_test.cpp
		- endpoints/get_row.h/cpp/test.cpp
		- endpoints/get_rows_by_column.h/cpp/test.cpp
		- endpoints/get_table_rows.h/cpp/test.cpp
		- endpoints/get_user_info.h/cpp/test.cpp
		- endpoints/login.h/cpp/test.cpp
		- endpoints/update_item.h/cpp/test.cpp
		- sql_util/json/database_rest_helper.h/cpp/test.cpp
		- test/src/util/json_test_util.h/cpp/test.cpp
		- test/src/util/json_value_matcher.h/cpp/test.cpp
	- Here is the new copilot query:
	> Please make the changes to the following files. Any boundaries that return a rvalue or wvalue can return a Variant. At the crow response level, please do response.write(value.ToText()). Here are the files:
- Make page to change client info
	- Created branch: client_auth_14_move_to_json_value
	- Copilot prompt
	> I would like to create a new endpoint update_user_info. Use that name to create a .h, .cpp, and \_test.cpp files. Make the url "/api/update_user_info". Please base it on get_user_info.h/cpp/test.cpp. If the user is not logged in, do the same security checks as GetUserInfo. Please make the function void UpdateUserInfo()
	- Can you update sql_util/json/database_rest_helper_test.cpp
	- Only one done so far is get_rows_by_column
	- Can you do: endpoints/add_item.h/cpp/test.cpp
	- Can you do: endpoints/add_item_fetch_primary_key.h/cpp/test.cpp
	- Can you do: endpoints/db_schema.h/cpp/test.cpp
	- Can you do: endpoints/delete_item.h/.cpp/test.cpp
	- Can you do: endpoints/get_row.h/cpp/test.cpp
	- Can you do: endpoints/get_table_rows.h/cpp/test.cpp
	- Can you do: endpoints/get_user_info.h/cpp/test.cpp
	- Can you do: endpoints/login.h/cpp/test.cpp
	- Can you do: endpoints/update_item.h/cpp/test.cpp
- Notes for things to change
	- Make JsonFromKeyValueTable return a Json::Value
		- Done
	- Same thing for Dataresults
		- Done
	- Contemplate switching all the internal things to key value table
		- Done
	- Add a HasChild accessor for the Value class
		- Done
	- Add array accessors directly to value instead of needing to build a JsonObject
		- Done
	- Add a Value assignment operator for std::string_view
		- Done
	- Add columns to skip to JsonWvalueMatcher and rename it
- Done

# What I'm working on 1/1
- Do cleanup for Json::Value class
	- Cleanup to do for this change
		- Make JsonFromKeyValueTable return a Json::Value
		- Same thing for Dataresults
		- Add a HasChild accessor for the Value class
		- Add array accessors directly to value instead of needing to build a JsonObject
		- Add a Value assignment operator for std::string_view
	- Implementation
		- Created branch: client_auth_15_json_cleanup
		- Copilot query:
		> Can you implement the tests StringKeyIndexOperatorBasic, StringKeyIndexOperatorConstBasic, IntIndexOperatorBasic, IntIndexOperatorConstBasic, HasChildBasic, HasChildConstBasic
	- Done!

# What I'm working on 1/4
- Switch over to KeyValueTable from DataResults
	- The plan
		- sql_util
			- database_access
				- database_crud_helpers.h
					- GetTableRows
						- Done
					- GetRowsByColumn
						- Done
					- GetRow
						- Done
				- transaction.h
					- RunSqlStatementReturningDataResults
					- RunSqlStatementReturningDataResultsHelper
					- Done
			- table_helpers
				- admin_alerts
					- GetAdminAlerts
					- Done
				- people
					- GetPeople
					- GetPersonById
					- LookupPersonByEmail
					- Done
		- test\src\util
			- json_test_util.h
				- Add CompareKeyValueTableMinusKeys
				- Add CompareKeyTableTableArrayMinusKeys
				- Not needed
			- test_helper.h
				- Add CompareKeyValueTableMinusKeys
				- Add CompareKeyTableTableArrayMinusKeys
				- Done
	- The implementation
		- In phases
		- Do the test helper work
			- Created branch: client_auth_16_key_value_table
			- Done
		- Do the transaction.h work
			- Created branch: client_auth_17_transaction
			- Done
		- Do the database_crud_helpers work
			- Created branch: client_auth_18_database_crud_helpers
		- Completed all of it!
- What is left to do on auth
	- Logout
	- Change user info
	- Wire up the auth guard for remember me / session

# What I'm working on 1/7
- Logout
	- The plan
		- app\portal\services\ServerAccess.ts
			- There already is a logout to call the server
		- app\shared\services\auth\auth.service.ts
			- Has logout() method that is already wired up
			- Make this prompt the user if they are sure
			- Then call logout on ServerAccess
			- Navigate to login page after
	- The implementation
		- Created branch: client_auth_19_logout
		- Copilot query for what is going wrong
		> I have a click handler for the sign out menu option that calls AuthService.logout. I prompt the user to confirm and then call ServerAccess.logout. Inside ServerAccess.logout, I do an alert that is showing up but then the next line that calls /api/logout on the server never happens. I have looked at the networking log in chrome and no call is made and I never get the call on the server. I'm calling it the same way as the other endpoints on the server which work. Do you know what might be going wrong?
	- Done
- Change user info
	- The plan
		- Add a created_at string to get_user_info on the server
			- Done
		- Add a user_information component
			- Modify client get_user_info struct for roles, permissions, and created_at
			- Show the basic user information from get_user_info
			- Have a "Edit user information" button that shows a popup for now
			- Have a "Change password" button that shows a popup for now
			- Done
		- Add update_user_info endpoint on the server
			- Allow changing email, first name, last name
			- Done
		- Add update_user_password endpoint on the server
			- Send old password and new password, regenerate all the info
			- Done
		- Add a update_user_info component
			- Have edit boxes with current info as well as Update / Cancel buttons
			- Make call to server and then navigate back to user_information component
			- Wire into user_information component
		- Add a update_user_password component
			- Have an enter old password and two enter new password edit boxes
			- Make call to server and then navigate back to user_information
			- Wire into user_information component
	- The implementation
		- Add a created_at string to get_user_info on the server
			- Created branch: client_auth_20_server_user_info
			- Copilot query to write a test for GetCreatedAt
			> Can you implement that test GetCreatedAtBasic? Please look at the other tests in this file and create a person and then call GetCreatedAt. Please verify that the string returned for the person just created is of the form "January 8, 2026". In other words, the first word should be the long version of the twelve month names. The second should be a one to two digit value between 1 and 31, the last value should be four digit year code with the first two digits being 20. Feel free to use the regex library in doing this.
			- Copilot query to check created_at
			> Can you modify the test GetUserInfoBasic so that after it validates email, first_name, and last_name, it also validates created_at? Please look at the test in sql_util/table_helpers/people_test.cpp in GetCreatedAtBasic and do similar validation.
			- Done
		- Add a user_information component
			- Created branch: client_auth_21_user_information
			- Updated UserInfo with created at
			- Touch up ServerAccessMock
			- Copilot query for user_information component
			> I'd like to create a component src\app\auth\components\user_information with user_information.component.ts/spec.ts/scss/html files. Please inject the AuthService from src\app\shared\services\auth\auth.service.ts. I would like to use Angular material componenst like the login and register components. Inside onInit, please subscribe to the AuthService.authData\$. I would like you to title the page: User Information. Then have text fields First Name:, Last Name:, Email:, and User Since: and have each of these followed by the corresponding value from authData\$. Note that User Since: corresponds to createdAt. Then have two buttons titled "Edit user" and "Change password". Create methods on the component for handlers that for now just create a popup with the text of the button for now. Create tests that verify that the page is shown but with the values from src\app\portal\services\network_abstraction\ServerAccess.mock.ts for userInfoDefault. Add tests that show that each button causes a popup with the appropriate text shown.
			- Done
		- Add update_user_info endpoint on the server
			- Created branch: client_auth_22_update_user_info_server
			- Copilot query:
			> I want to create a new endpoint called set_user_info. You can base it off of src\endpoints\get_user_info.cpp and src\endpoints\login.cpp and you will make use of src\auth\person.h's UpdateInfo. The post will contain JSON that has the fields first_name, last_name, and email. You will convert the request body to a Json::Value. For each of these fields that are present, you will extract the field and put it into a PersonInfo to call UpdateInfo. Missing a field from the POST is not an error. Please use the logged in user like get_user_info. The URL for this endpoint should be /api/set_user_info. Please create the files set_user_info with a h, cpp, and underscore test.cpp files. Use the same namespace as the other components. In the test file, please place the tests under UpdateUserInfoTest and create a UpdateUserInfoBasic that validates basic functionality \(make the call to the endpoint and validate the state of the database after with PersonHelper::LookupPerson\). Please also do a UpdateUserInfoNoUser and make sure that 401 is returned. Please make sure to end the response. The function should be void SetUserInfo\(EnpointAuthHelper& endpointAuthHelper, const crow::request& req, crow::response& resd, const Json::Value& message\). Please update CMakeLists.txt and web_app.cpp.
			- Done
		- Add update_user_password endpoint on the server
			- Created branch: client_auth_23_update_user_password_server
			- Copilot query:
			> I want to create a new endpoint called update_user_password. You can base it off of src\endpoints\set_user_info.cpp and you will make use of src\auth\person.h's UpdatePassword. The post will contain JSON that has the fields old_password and new_password. You will convert the request body to a Json::Value. Lookup the person to fetch the email like SetUserInfo. Once that is done, use PersonHelper::VerifyPassword to validate old_password. Then do UpdatePassword with new_password. Please use the logged in user like set_user_info. The URL for this endpoint should be /api/update_user_password. Please create the files update_user_password with a h, cpp, and underscore test.cpp files. Use the same namespace as the other components. In the test file, please place the tests under UpdateUserPasswordTest and create a UpdateUserPasswordBasic that validates basic functionality \(make the call to the endpoint and validate the state of the database after with PersonHelper::VerifyPassword\). Please also do a UpdateUserPasswordNoUser and make sure that 401 is returned. Please make sure to end the response. The function should be void UpdateUserPassword\(EnpointAuthHelper& endpointAuthHelper, const crow::request& req, crow::response& resd, const Json::Value& message\). Please update src\endpoints\CMakeLists.txt and src\endpoints\web_app.cpp.
			- Done
		- Add a update_user_info component
			- Created branch: client_auth_24_update_user_info_client
			- Added setUserInfo to ServerAccess
			> I'd like to add a method to AuthService public setUserInfo(firstName: string, lastName: string, email: string): Observable\<void>. It should call serverAccess.setUserInfo and return the observable but it should also make a call when that call completes to serverAccess.getUserInfo and subscribe to that call and then do \_authDataSubject.next() on the result. I'd like to add a method public doSetUserInfo(firstName: string, lastName: string, email: string): void. It should internally call setUserInfo and subscribe to it. Please add tests to the corresponding spec file to make sure that calling doSetUserInfo publishes to those subscibed to \_authData\$.
			- Copilot prompt for component:
			> I'd like to create a component src\app\auth\components\update_user_info with update_user_info.component.ts/spec.ts/scss/html files. Please inject the AuthService from src\app\shared\services\auth\auth.service.ts. I would like to use Angular material components like the user_information component. Inside onInit, please subscribe to the AuthService.authData\$. I would like you to title the page: User Information. Then have edit text boxes labelled First Name:, Last Name:, Email: and have each of these start out with the value from the corresponding value from authData\$.  Then have two buttons titled "Update" and "Cancel". Please use reactive forms. Please inject the angular router. Please have update call AuthService.setUserInfo and subscribe to the observable and then navigate to /p/profile when it completes. Have Cancel just navigate to /p/profile. Create tests that verify that the page is shown but with the values from src\app\portal\services\network_abstraction\ServerAccess.mock.ts for userInfoDefault. Add a test that changes the values and makes sure that the authData$ changes on AuthService are published and that we navigate to the correct page. Make sure we navigate to the correct place on cancel.
			- Copilot prompt for edit-user
			> Can you change the test for 'Edit user button shows popup' so that instead makes sure that the router navigates to '/p/update-user-info'
			- Done
		- Add a update_user_password component
			- Created branch: client_auth_25_update_user_password_client
			- Added updateUserPassword to ServerAccess
			- Add updateUserPassword method to AuthService copilot prompt
			> I'd like to add a method to AuthService called public updateUserPassword(oldPassword: string, newPassword: string): Observable\<void>. It should call ServerAccess.updateUserPassword and return the observable. I'd like to add a method public doUpdateUserPasswordoldPassword: string, newPassword: string): void. Internally, it should call updateUserPassword and subscribe to it.
			- Copilot prompt for component
			> I'd like to create a component src\app\auth\components\update_user_password with update_user_password.component.ts/spec.ts/scss/html files. Please inject the AuthService from src\app\shared\services\auth\auth.service.ts. I would like to use Angular material components like the user_information component. I would like you to title the page: Change Password. Please look at src\app\auth\components\register\register.component.html and src\app\auth\components\register\register.component.ts. I would like an edit box called Old Password: and then another labelled Reenter Old Password:. Please to the same thing as the register component and have a button toggle the password visibility and also to a matching password validator like the register component. Then create another edit box called New Password: that also has a password visibility button and participates in the password visibility scheme. Please make all of these required. Then have two buttons titled "Update" and "Cancel". Please use reactive forms. Please inject the angular router. Please have update call AuthService.updateUserPassword and subscribe to the observable and then navigate to /p/user_information when it completes. Have Cancel just navigate to /p/user_information. Create tests that verify that the page is shown and that the passwords are obscured by default when entered. Do another test to toggle visibility and verify that all are visible. Add a test that enters the same old password in both old password edit boxes and another password in the new password and verify that clicking Update calls on AuthService and that we navigate to the correct page. Make sure we navigate to the correct place on cancel.
			- Copilot prompt to update test:
			> Can you change the test for 'Change password button shows popup' so that it checks that we navigate to '/p/update_user_password' instead of showing an alert?
			- Done

# Branches
- client_auth_1_sign_in
	- Submitted
- client_auth_2_register
	- Submitted
- client_auth_3_auth_server_access
	- Submitted
- client_auth_4_lucas_cleanup
	- Submitted
- client_auth_5_register_component_style
	- Submitted
- client_auth_6_auth_service
	- Submitted
- client_auth_7_auth_wiring
	- Submitted
- client_auth_8_fix_db_schema
	- Submitted
- client_auth_9_account_activation
	- Submitted
- client_auth_10_json
	- Submitted
- client_auth_11_json_variant
	- Submitted
- client_auth12_register_login
	- Submitted
- client_auth_13_from_text
	- Submitted
- client_auth_14_move_to_json_value
	- Submitted
- client_auth_15_json_cleanup
	- Submitted
- client_auth_16_key_value_table
	- Submitted
- client_auth_17_transaction
	- Submitted
- client_auth_18_database_crud_helpers
	- Submitted
- client_auth_19_logout
	- Submitted
- client_auth_20_server_user_info
	- Submitted
- client_auth_21_user_information
	- Submitted
- client_auth_22_update_user_info_server
	- Submitted
- client_auth_23_update_user_password_server
	- Submitted
- client_auth_24_update_user_info_client
	- Submitted
- client_auth_25_update_user_password_client
	- Submitted
- 