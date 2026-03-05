---
fileClass: Project
Category: Research
Status: Active
---
# Overview
I need HTTPS support eventually but especially for Square Webhooks to call back to the server. For production, I will need to have public / private keys and build with HTTPS support. For dev scenarios, I can get by with ngrok. The basic workflow will look like:

Run Crow on plain HTTP locally (say http://localhost:8080), then let ngrok provide the public HTTPS endpoint.

Start your server:

- localhost:8080
- Start ngrok:
- ngrok http 8080
- ngrok will print something like:
- https://abc123.ngrok-free.app
- Put that HTTPS URL into the Square Developer Console webhook subscription URL.
- Square will call ngrok over HTTPS, ngrok forwards to your local HTTP server.

# Sources
- Links to sources here
	- Square webhooks
		- https://developer.squareup.com/docs/webhooks/overview?utm_source=chatgpt.com
	- How to do HTTPS in Crow
		- https://crowcpp.org/1.2.1/guides/ssl/?utm_source=chatgpt.com
	- Middleware for Crow including SSL
		- https://crowcpp.org/master/reference/classcrow_1_1_crow.html?utm_source=chatgpt.com
	- How to setups Crow on Linux
		- https://crowcpp.org/master/getting_started/setup/linux/?utm_source=chatgpt.com
	- 

# Things to investigate
- List of things that need further investigation