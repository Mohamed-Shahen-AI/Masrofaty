# Masrofaty: A Telegram Expense Management Chatbot

**Masrofaty** is a smart personal finance assistant that operates entirely within Telegram. It leverages the power of n8n, LangChain, OpenRouter's AI models, and Google Sheets to help users track their daily expenses effortlessly using natural language (specifically Egyptian Arabic slang). It can log new expenses and query past ones, all through a simple conversation.

---

## System Overview
This project is built using **n8n**, a fair-code workflow automation tool. The core of Masrofaty is a multi-node workflow that acts as the chatbot's brain. It processes incoming Telegram messages, uses an AI model to interpret user intent, interacts with a Google Sheet, and sends back appropriate responses.

### 1. The Workflow Diagram
The workflow image below provides a high-level visual representation of how a user message is processed and how different services are orchestrated:

![Masrofaty Workflow Diagram](Masrofaty%20Workflow%20Image.png)
*Figure 1: The visual workflow diagram of Masrofaty, showing the interconnected nodes.*

---

### 2. Detailed Node Breakdown
The n8n workflow is composed of several specialized nodes, each responsible for a specific function. The following is a detailed breakdown of each node, organized by their logical order and type:

#### A. The Start Node
*   **Telegram Trigger:** This is the entry point. It "listens" for specific updates in a dedicated Telegram bot. It is configured to trigger on "message" updates. This ensures that every text message, photo, or location sent to the bot is processed by the workflow.

#### B. Logic & Routing Nodes
*   **If:** This is a crucial security and personalization node. It checks the username of the incoming Telegram message. It compares it against a hardcoded value (e.g., `ebr_salem`).
    *   **If True:** The workflow continues to process the message.
    *   **If False:** It directs the flow to a "Not my user" node, ensuring the bot's features are only accessible to the intended owner.
*   **Switch (mode: Rules):** This node acts as a message router. It analyzes the text of the incoming user message and uses rules to decide the next path.
    *   **'Add' Output:** If the user sends a message like 'a' (or, in a more complex setup, "add" or "أضف"), the workflow is directed to a manual, guided expense logging form.
    *   **'user message' Output:** If the user sends any other text message (e.g., "صرفت 50 جنيه بنزين"), it is considered natural language input for the AI to process, and the workflow directs the flow to the AI Agent.

#### C. The AI Core (LangChain Integration)
This is the most complex and powerful part of the workflow, where the natural language understanding occurs.

*   **AI Agent (LangChain):** This node is the central intelligence unit. It is a LangChain-powered agent that understands user intent from text. It is given a detailed `systemMessage` (the core prompt) that instructs it to act as a "Personal Finance Assistant."
    *   It is trained to understand natural language, particularly Egyptian Arabic slang.
    *   It has critical rules, such as storing specific fields (Item Name, Description, Category) in the user's provided language (Arabic), not English.
    *   It knows how to use the available Google Sheets Tool nodes to either append a new expense or query for expense summaries.
    *   It has a predefined list of approved Arabic categories (e.g., 'مواصلات', 'طعام وشراب').
*   **OpenRouter Chat Model:** This is the language model that powers the AI Agent. It connects to OpenRouter, a platform that provides a unified API for various LLMs. It is specifically configured to use a powerful model like `nvidia/nemotron-3-super-120b-a12b:free`. This model is responsible for interpreting the prompt and user message to generate structured output or make tool-use decisions.

#### D. Tools Nodes
These nodes are "tools" that the AI Agent can dynamically choose to execute based on its interpretation of the user's request.

*   **Append row in sheet in Google Sheets (Tool):** This node is used when the AI Agent decides to log a new expense. It's configured to add a new row to a specific Google Sheet and sheet name.
    *   **Column Mapping:** It maps data from the AI's output to the sheet's columns (Item Name, Item Description, Price, Category, Date/Time). The use of `$fromAI()` expressions shows that it expects structured data from the AI.
*   **Get row(s) in sheet in Google Sheets (Tool):** This node is used when the AI Agent decides to query existing expenses (e.g., "صرفت كام الشهر ده؟"). It's configured to fetch data from the same Google Sheet, which the AI then summarizes for the user.
*   **Send message and wait for response (Telegram):** This node is an alternative, manual path for logging an expense. If the Switch node directs the flow here (e.g., user sent 'a'), it sends a guided Telegram message asking for expense details (Date, Item Description). The workflow pauses here, waiting for the user to fill out a custom Telegram form. Once the user replies, the workflow resumes and passes the form data to the AI Agent to process.
*   **Not my user (Set Node):** This node is a placeholder and message-setting node. If the If node fails (wrong user), this node is executed. It simply defines a message like: "This chatbot created for another client, if you need one like this contact me +201289775133."

#### E. Output & Merging Nodes
*   **Merge:** This node consolidates input from different paths. It can receive data from:
    1. Input 1 (from AI Agent or Not my user paths).
    2. Input 2 (from the guided Send message and wait for response path).
    Its purpose is to make sure that whatever the outcome (AI response, error message, or next steps), a single piece of text is prepared for final delivery.
*   **Send a text message (Telegram):** This node delivers the final response to the user's chat. The message content is a variable (`={{ $json.output }}` or similar), which is the final combined text from the Merge node.
*   **Send a text message2 (Telegram):** A secondary, follow-up message node. It might be used for persistent prompt, such as: "press 'a' to Add" to remind the user of the quick-add command, and it sends the current time for reference.

---

### 3. Prerequisite Setup
Before importing the workflow, you need to set up the following:

*   **n8n Instance:** A running instance of n8n (e.g., on a server, in Docker, or n8n Cloud).
*   **Telegram Bot:**
    *   Create a new bot via @BotFather on Telegram.
    *   Obtain your Telegram API Key.
*   **Google Cloud Platform Project & Sheets API:**
    *   Create a new GCP project.
    *   Enable the Google Sheets API.
    *   Create OAuth2 credentials for a Web Application, which will give you a Client ID and Client Secret.
    *   Create a blank Google Sheet.
*   **OpenRouter Account:**
    *   Create an account on OpenRouter.
    *   Generate a new API Key.

---

### 4. Installation and Configuration
Follow these steps to deploy Masrofaty:

1.  **Import the Workflow:**
    *   Create a new empty workflow in n8n.
    *   Copy the JSON content of the workflow and paste it into the editor. It will automatically populate the nodes.
2.  **Set Up Credentials:** You will see red warnings on some nodes indicating missing credentials. You need to create and assign new credentials in n8n:
    *   **Telegram Trigger, Send a text message, etc.:** Create a new Telegram API credential and paste your Telegram API Key.
    *   **OpenRouter Chat Model:** Create a new OpenRouter API credential and paste your API Key.
    *   **Get row(s)/Append row in sheet:** Create a new Google Sheets OAuth2 API credential. This is a crucial step:
        *   Copy the OAuth Client ID and OAuth Client Secret from your GCP project.
        *   The n8n credential creation screen will provide an "OAuth Callback URL." Copy this URL.
        *   Go back to your GCP project and add this callback URL to the "Authorized redirect URIs" for your OAuth2 credentials.
        *   Finally, click "Sign in with Google" in the n8n credential creation screen to grant access.
3.  **Configure Node Parameters:**
    *   **If:** Change the right value `ebr_salem` to your own Telegram username.
    *   **Get row(s)/Append row in sheet:**
        *   For Document ID, select `id` mode. You need to find and paste the ID of your Google Sheet.
        *   Ensure the Sheet Name (e.g., 'Sheet1') is correct and points to the right tab in your sheet.
    *   **AI Agent:** Double-check the systemMessage prompt. It's pre-configured, but you can refine the language, tone, or categories.
    *   **Set (Not my user):** Change the contact information if needed.
4.  **Activate the Workflow:** Once all warnings are cleared, click the "Activate" toggle in the top-right corner.

---

### 5. Usage
Once activated, simply message your Telegram bot. Here are some example commands and interactions:

**A. Natural Language Logging (Arabic):**
*   "صرفت 25 جنيه سحلب في القهوة" (Item: سحلب, Price: 25, Description: في القهوة)
*   "دفعت 150 جنيه بنزين" (Item: بنزين, Price: 150)
*   "جبت فول مدمس ب 45 جنيه للفطار" (Item: فول مدمس, Price: 45, Description: للفطار)
*   The bot will process this, add a row to your sheet, and confirm in Arabic.

**B. Guided Logging (for dates/descriptions):**
*   Send 'a'. The bot will present a form asking for "Date" and "Item Description." This is useful if you want to be more specific or log an expense for a past date.

**C. Querying Expenses (Arabic):**
*   "صرفت كام الشهر ده؟" (Fetches a summary of expenses for the current month).
*   "إجمالي مصاريف الأسبوع" (Fetches a summary for the current week).
*   The bot will query the sheet, calculate the sum, and provide a textual summary.

---

### Contributing
This is a personal project, but suggestions and improvements are welcome. Feel free to fork the repository and submit a pull request.

### Author
**Muhammad Shaheen** - [LinkedIn](https://www.linkedin.com/in/muhammad-shaheen-ai/)
