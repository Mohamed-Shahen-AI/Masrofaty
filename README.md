# Masrofaty: AI-Powered Personal Finance Assistant

## Project Overview
Masrofaty is an automated expense tracking system that utilizes Artificial Intelligence to process natural language financial inputs. Built on **n8n**, the system enables users to manage their personal finances through a Telegram interface, automatically categorizing and logging data into Google Sheets.

The system is specifically optimized to interpret **Egyptian Arabic slang** and convert informal descriptions into structured financial records.

![Masrofaty Workflow](Masrofaty%20workflow%20Image.png)

## Technical Architecture
The workflow leverages an Agentic AI model to handle unstructured data through the following stages:

1.  **Ingestion**: A Telegram Trigger receives text input (e.g., "دفعت 50 جنيه بنزين").
2.  **Authentication**: An "If" node verifies the user's identity based on their Telegram username to restrict access to authorized clients.
3.  **Natural Language Processing (NLP)**:
    *   **AI Agent**: Orchestrates the interaction between the user and the tools.
    *   **Language Model**: Utilizes `nemotron-3-super-120b` via OpenRouter to parse Arabic slang and extract entities (Item, Price, Category).
4.  **Tool Execution**:
    *   **Append Row**: Logs new transactions with timestamps, item names, and calculated categories.
    *   **Get Rows**: Retrieves historical data when the user requests a financial summary.
5.  **Feedback**: Sends a structured confirmation or summary back to the user via Telegram.

## Core Components

| Component | Technology | Responsibility |
| :--- | :--- | :--- |
| **Telegram Trigger** | Telegram Bot API | Receives user input and commands. |
| **AI Agent** | LangChain Agent | Manages logic flow and tool selection. |
| **Language Model** | OpenRouter (Nemotron) | Extracts financial data from Egyptian Arabic text. |
| **Database Tool** | Google Sheets API | Stores transaction history and performs data lookups. |
| **Logical Router** | n8n If Node | Validates user permissions and handles unauthorized access. |

## Data Structure
The Google Sheets database is configured with the following schema:
*   **Item Name**: The primary name of the purchased item/service.
*   **Item Description**: Additional context provided by the user.
*   **Price**: Numeric value of the expense.
*   **Category**: Automated classification (e.g., طعام وشراب، مواصلات، سكن وفواتير).
*   **Date/Time**: ISO timestamp of the transaction.

## Configuration Requirements

### API Credentials
*   **Telegram API**: Bot token for communication.
*   **OpenRouter API**: Access key for the LLM.
*   **Google OAuth2**: Authorized access to the target spreadsheet.

### Environment Parameters
*   **Authorized Username**: The workflow is hardcoded to validate a specific Telegram handle before processing AI requests.
*   **System Prompt**: Defined within the AI Agent to enforce language rules and category constraints.

## Operational Features
*   **Slang Parsing**: Accurately identifies Egyptian terms like "جبت", "فولت", or "عشيت".
*   **Error Handling**: Unauthorized users receive a predefined message with contact information for service acquisition.
*   **Retry Logic**: The OpenRouter node is configured to retry on failure to ensure system reliability during API downtime.
