# FE Assistant: Slack Allow-List Implementation Guide

To implement an allow-list so only authorized Frontend (FE) Developers can use the Slack bot, you need to intercept the message **before** it reaches the AI Agent node in your UnifyApps workflow.

Since you are receiving the `event.user` field (which contains the Slack User ID, e.g., `U12345678`) from the Slack trigger, here are the three best ways to build this in UnifyApps, ordered from simplest to most scalable:

## Method 1: The Hardcoded JS Node (Quickest & Easiest)
If your FE team is small and doesn't change often, you can hardcode the allow-list directly into a JavaScript node.

1. **Find the Slack User IDs:** In Slack, click on a developer's profile picture -> click the three dots (`...`) -> **Copy member ID**.
2. **Add a JS Node:** In UnifyApps, place a JavaScript node right after the Slack trigger (and before the AI Agent).
3. **Add this code:**
   ```javascript
   // List of allowed FE Dev Slack Member IDs
   const allowedUsers = ["U0123ABCD", "U0987WXYZ", "U5555FFFF"]; 
   
   // Check if the sender is in the list
   const isAllowed = allowedUsers.includes(event.user);
   
   result = { isAllowed: isAllowed };
   ```
4. **Add a Condition Node:**
   - **If Yes (`isAllowed == true`):** Route to the AI Agent node.
   - **If No (`isAllowed == false`):** Route to a Slack "Send Message" node that replies: *"Sorry, only authorized FE developers can run deployment commands."* (Stop workflow).

## Method 2: The UnifyApps Database (Best for Maintenance)
If you don't want to edit the workflow every time a new developer joins the team, store the list in a UnifyApps database.

1. **Create a Database Table:** In UnifyApps, create a new table called `FE Bot Allowlist`. Add a column for `Slack User ID`.
2. **Add a Database Lookup Node:** Place this after your Slack trigger. Set it to query the `FE Bot Allowlist` table where `Slack User ID` equals `event.user`.
3. **Add a Condition Node:** 
   - Check if the Database Lookup returned a record (e.g., `lookupResult is not empty`).
   - **If Yes:** Route to AI Agent.
   - **If No:** Route to "Unauthorized" Slack message.

## Method 3 (Database Edition): Checking Emails via UnifyApps Database
This is the most robust method for large teams. It allows you to maintain a list of friendly email addresses (e.g., `dev@company.com`) in the UnifyApps database, rather than dealing with cryptic Slack Member IDs (like `U123456`).

### Part 1: Set up the Allow-list Database
1. In the UnifyApps main menu, navigate to **Data / Databases**.
2. Create a new table named `FE_Bot_Allowlist`.
3. Add a single text column named `Email`.
4. Add the email addresses of your authorized Frontend Developers as records in this table.

### Part 2: Build the Workflow Logic
In your Slack Deployment automation, insert these nodes **between** the Slack Trigger and the AI Agent:

1. **Add a Slack "Get User Info" Node:**
   - Action: `users.info` (Get user information)
   - Input `User ID`: Map this to the `{b} event.user` from the Slack trigger.
   - *Why?* This converts the cryptic Slack ID into the user's actual profile, giving us their email address.

2. **Add a UnifyApps Database "Find Record" Node:**
   - Action: Find Record (or Search Records).
   - Table: Select your `FE_Bot_Allowlist` table.
   - Filter/Condition: Set `Email` **equals** the email returned from the Slack node in Step 1 (usually mapped as `{b} profile.email`).

3. **Add a Condition Node:**
   - **Condition:** Check if the Database node returned a result (e.g., `Result is not empty` or `Total Records > 0`).
   - **If Yes (True):** Connect this path to your AI Agent node. (The user is authorized!)
   - **If No (False):** Connect this path to a new Slack "Send Message" node. Send a message to `{b} event.channel` saying: *"Sorry, you are not on the authorized list of FE Developers to run this bot."* (Stop the workflow here).

## Recommendation
**Method 1** is recommended to start. It takes 2 minutes to set up, requires zero external API calls (making your bot faster), and works perfectly if your FE team is small.
