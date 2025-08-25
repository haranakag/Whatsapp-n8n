# Whatsapp-n8n

n8n is a powerful, source-available automation platform that excels at creating flexible solutions with both traditional workflows and complex AI automations. 
It allows for the building of sophisticated, deeply connected systems and offers a visual workflow builder alongside custom coding capabilities in JavaScript or Python.

Below is a conceptual integration diagram for n8n, illustrating its role in orchestrating AI agents and various external services, particularly in a WhatsApp context:

**n8n Integration Diagram (Conceptual)**

```
+--------------------------------------------------------------------------------------------------------------------------------+
|                                                           External Systems                                                   |
+--------------------------------------------------------------------------------------------------------------------------------+
|                                                                                                                                |
|  +---------------------+        +---------------------+        +---------------------+        +---------------------+        |
|  |       WhatsApp      | <----->|    Evolution API    | <----->|      Google         | <----->|      Google Drive   |        |
|  |  (User Interaction) |        | (Unofficial WhatsApp|        |      Calendar       |        | (Knowledge Base PDFs)|        |
|  +---------------------+        |        API)         |        | (Availability,       |        +---------------------+        |
|                                 +---------------------+        |    Appointments)    |                                         |
|                                                                +---------------------+                                         |
|                                                                                                                                |
|  +---------------------+        +---------------------+        +---------------------+                                         |
|  |       Webhooks      | <----->|       Supabase      | <----->|        Redis        |                                         |
|  |  (Other Triggers/   |        | (Lead DB, Vector    |        | (Message Buffer/    |                                         |
|  |       Data)         |        |    Store for RAG)   |        |        Cache)       |                                         |
|  +---------------------+        +---------------------+        +---------------------+                                         |
|                                                                                                                                |
+--------------------------------------------------------------------------------------------------------------------------------+
                                       |
                                       |  Data/Events
                                       V
+--------------------------------------------------------------------------------------------------------------------------------+
|                                                          n8n Platform (Workflow Orchestration)                               |
|--------------------------------------------------------------------------------------------------------------------------------|
|                                                                                                                                |
|  **Trigger Node (e.g., WhatsApp On Messages, Webhook Call)**                                                                   |
|  - Receives incoming messages/events from external systems (e.g., Evolution API for WhatsApp).|
|                                       |
|                                       V
|  **Workflow Logic & Data Pre-processing (e.g., Set, If, Switch, Edit Fields Nodes)**                                           |
|  - Sets variables, filters messages, handles different message types (text, audio, image).|
|  - Processes binary data (e.g., Base64 for audio/image) and converts to files for AI analysis.     |
|  - **Redis Buffer:** Manages message buffering to concatenate fragmented user inputs.|
|                                       |
|                                       V
|  **AI Agent Node (Core Intelligence & Action)**                                                                                |
|  - **Input:** Takes processed user message as input.                                                  |
|  - **Brain:**                                                                                                                  |
|    - **Large Language Model (LLM):** Utilizes models like OpenAI's GPT, Anthropic's Claude, Gemini for reasoning and response generation.|
|    - **Memory (Postgres):** Stores interaction history for context, allowing the AI to maintain a humanized conversation.|
|  - **Instructions (Prompt):** Defines the agent's role, task, rules, and how to use tools (e.g., "You are Julia, an SDR...").|
|  - **Tools Integration (Call Workflow Tool):** Accesses specialized sub-workflows for external actions.|
|    - **Tool: Buscar Horários (Google Calendar):** Finds available meeting slots.                    |
|    - **Tool: Verificar Disponibilidade (Google Calendar):** Checks if a specific time slot is free.|
|    - **Tool: Agendar Horário (Google Calendar, Supabase):** Schedules meetings, updates lead status in Supabase.|
|    - **Tool: Transferir Atendimento (Supabase, Evolution API):** Changes lead status to "transferred" and notifies human agent.|
|    - **Tool: Dados Adept (Supabase Vector Store - RAG):** Retrieves information from a custom knowledge base (PDFs from Google Drive).|
|  - **OpenAI Services:**                                                                                                         |
|    - **Transcribe Audio:** Converts voice messages to text.                                        |
|    - **Analyze Image:** Describes image content.                                                    |
|                                       |
|                                       V
|  **Response Generation & Formatting (e.g., AI Agent for Formatting, Split Out, Loop Over Items, HTTP Request Nodes)**         |
|  - A dedicated AI agent may format the main AI's response into multiple, natural-sounding messages for WhatsApp.|
|  - Splits the formatted response into individual messages and loops to send them sequentially with delays.|
|  - Sends messages back to WhatsApp via the Evolution API.                                         |
|                                                                                                                                |
+--------------------------------------------------------------------------------------------------------------------------------+
                                       |
                                       |  Response/Actions
                                       V
+--------------------------------------------------------------------------------------------------------------------------------+
|                                                           External Systems                                                   |
+--------------------------------------------------------------------------------------------------------------------------------+
|                                                                                                                                |
|  +---------------------+        +---------------------+        +---------------------+        +---------------------+        |
|  |       WhatsApp      | <----->|    Evolution API    | <----->|      Google         | <----->|      Human Agent    |        |
|  | (Formatted Response)|        | (Unofficial WhatsApp|        |      Calendar       |        | (Notifications,     |        |
|  +---------------------+        |        API)         |        | (Event Creation)    |        |   Transferred Leads)  |        |
|                                 +---------------------+        +---------------------+        +---------------------+        |
|                                                                                                                                |
+--------------------------------------------------------------------------------------------------------------------------------+

**Key Concepts and Flow:**

1.  **Input Reception:** The process begins when an event occurs in an external system, most commonly a message sent to **WhatsApp**.
This message is received by an **Evolution API** instance (often self-hosted on a **Hostinger VPS**) which then triggers an n8n workflow via a webhook.
2.  **Data Pre-processing in n8n:**
    *   The n8n workflow immediately processes the incoming data. This involves setting variables (like sender name, phone number, message content, message type).
    *   A **Switch node** directs the workflow based on message type (text, audio, image) for appropriate handling.
    *   **Audio messages** are converted from Base64 to a file, then transcribed into text using **OpenAI**.
    *   **Image messages** (with or without captions) are downloaded, then analyzed by **OpenAI** to generate a description.
    *   **Redis** acts as a message buffer, collecting fragmented messages from the user over a short period (e.g., 30 seconds) and concatenating them into a single, coherent input for the AI.
3.  **AI Agent Processing:**
    *   The concatenated message (or transcribed audio/image description) is fed into the **main AI Agent node** within n8n.
    *   This AI agent's "Brain" uses an **LLM** (e.g., GPT-4.1 Mini, GPT-4o) for reasoning and an **Postgres database** for memory, allowing it to maintain conversational context.
    *   The agent operates under specific "Instructions" (prompts) that define its role (e.g., "Julia, SDR from Adept"), objectives (e.g., qualify leads, schedule meetings), and communication style.
    *   **Tools:** The AI agent can autonomously call other n8n workflows as "tools" to perform actions. These tools include:
        *   **Google Calendar** for checking availability ("Buscar Horários", "Verificar Disponibilidade") and scheduling meetings ("Agendar Horário").
        *   **Supabase** to manage lead status (e.g., marking a lead as "transferred" to a human agent) and as a **Vector Store** for **RAG** (Retrieval Augmented Generation) to retrieve information from a custom knowledge base (e.g., PDFs from Google Drive).
        *   "Transferir Atendimento" tool: Updates the lead's status and sends a notification to a human agent if the AI determines human intervention is needed.
4.  **Output Generation & Delivery:**
    *   After processing and potentially using tools, the AI generates a response.
    *   A second **AI Agent node** can be used as a "formatter" to break down the main AI's response into multiple, shorter, more natural-sounding messages for WhatsApp.
    *   These messages are then split and sent one by one with small delays via HTTP requests to the **Evolution API**, which delivers them back to the user on **WhatsApp**.
    *   In cases like scheduling, the system also creates events in **Google Calendar** and sends notifications to relevant human agents via WhatsApp.

**Deployment:** n8n can be **self-hosted** (e.g., on a Hostinger VPS with Docker) for full control and data privacy, or used via **n8n Cloud**. Hostinger specifically offers easy setup with n8n VPS templates.
