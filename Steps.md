n8n is a versatile, open-source automation platform that enables the creation of sophisticated workflows, including complex AI agents, by combining visual development with custom code. It offers over 400 pre-built integrations and thousands of community nodes.

Here is a step-by-step guide to creating an n8n integration for an AI agent on WhatsApp, drawing on the provided sources:

### Step 1: Configure WhatsApp Business API in Meta

1.  **Register and Create an App:**
    *   Go to the **Meta Developer Portal** registration page and log in with your Facebook account.
    *   Click **"Create App"** and select **"Other"** as the use case, then **"Businesses"** as the app type.
    *   Provide an app name and contact email, then confirm to create the app.
2.  **Set up WhatsApp Product:**
    *   On the **"Add products to your app"** page, click **"Set up"** on WhatsApp.
    *   Follow instructions to set up a business profile on the **Meta business tools** website.
    *   Return to your Meta app's dashboard and click **"Continue"** to integrate WhatsApp.
3.  **Obtain Credentials:**
    *   Access **App Settings > Basic** to find your **App ID** and **App Secret**. These correspond to the Client ID and Client Secret in n8n.
    *   In the **WhatsApp section**, go to **API Setup** to generate an **Access Token** and retrieve your **Business Account ID**. You may need to add your personal phone number to test.

### Step 2: Set up n8n and Evolution API (Self-Hosted Recommendation)

1.  **Choose Hosting for n8n:**
    *   n8n can be **self-hosted** (e.g., on a Hostinger VPS with Docker) for full control and data privacy, or used via n8n Cloud.
    *   For Hostinger VPS, the KVM2 plan is recommended for most projects due to its cost-benefit and performance.
    *   Select **Ubuntu 24.04** with **Easy Panel** as the operating system during VPS setup.
    *   Create a **root password** for your VPS.
2.  **Install n8n via Easy Panel:**
    *   Access Easy Panel and **create a new project** (e.g., "servidor").
    *   Add **n8n** as a service from the templates.
    *   Create an n8n user account.
3.  **Install Evolution API (for Unofficial WhatsApp API):**
    *   **Evolution API** is a non-official WhatsApp API often used with n8n. It can be installed as another service on the same Hostinger VPS via Easy Panel templates.
    *   Access the **Evolution API's environment variables** (e.g., in Easy Panel under "Environment") and locate the `authentication API key`.
    *   Go to the Evolution API Manager URL (provided after deployment), paste the `authentication API key`, and log in.
    *   **Create a new instance** in the Evolution API Manager (e.g., "Enzo" or "adept"), save the generated token.
    *   **Generate a QR code** for your instance and scan it with WhatsApp on your phone to connect. (If the QR code doesn't appear, you might need to change the `config_session_phone_version` variable in the Evolution API's environment settings).

### Step 3: Connect Credentials in n8n

You will need to set up credentials for various external services within n8n.

1.  **OpenAI:**
    *   Go to the OpenAI platform, navigate to **API Keys**, and create a new secret key.
    *   In n8n, go to **Credentials**, click **"Add Credential"**, search for "OpenAI", and paste your API key.
2.  **Redis (for message buffering):**
    *   Create a new Redis database (e.g., on Redis Cloud, using a free plan and selecting a Brazil region for better latency).
    *   Obtain the **Host, Port, and Password** for your new Redis database.
    *   In n8n, add a new **Redis credential** using these details.
3.  **Supabase (for lead management and Vector Store):**
    *   Create a new project in Supabase (e.g., using a free cloud plan).
    *   **Note down your database password** during project creation.
    *   From your Supabase project settings, find the **Project URL** and the **Service Role Secret** (API key) under the **Data API** section.
    *   In n8n, add a new **Supabase API credential** with the Project URL and Service Role Secret.
    *   **Database Setup:** Create a new table named `leads_whatsapp` with columns like `session_ID` (text, primary key), `name` (text), and `status` (text). Apply a security policy allowing `ALL` permissions with `true` for `Set`.
    *   **Vector Store Setup:** In Supabase's SQL Editor, run the provided SQL script to create the `documents` table for your vector store. Enable and create a security policy for this table too.
4.  **Postgres (for AI agent memory):**
    *   In n8n, add a new **Postgres credential**. You will connect to Postgres through Supabase's transaction pool.
    *   Find the **Host, Port, and User** information in your Supabase project under **Connection > Transaction Pooler**.
    *   Use the **database password** you noted earlier for Supabase as the password for this Postgres connection.
5.  **Google (Calendar, Drive):**
    *   Go to **Google Cloud Console** and create a new project.
    *   Enable the **Google Calendar API** and **Google Drive API** in the API Library.
    *   Configure the **OAuth consent screen** (External user type, app name, support email) and **publish the app**.
    *   Create **OAuth client ID credentials** for a **Web application**.
    *   In n8n, when creating a new Google credential (for Calendar or Drive), copy the provided **Callback URL** and add it as an **Authorized Redirect URI** in your Google OAuth client settings.
    *   Copy the **Client ID** and **Client Secret** from Google to n8n to complete the connection.

### Step 4: Build the Main n8n Workflow (AI Agent for WhatsApp)

1.  **Webhook Trigger:**
    *   Create a new workflow in n8n. Add a **Webhook** node as the trigger.
    *   Set the **HTTP Method** to **POST**.
    *   Define a **Path** (e.g., `receber-mensagens`).
    *   Copy the **Webhook URL** (test URL) from n8n.
    *   In the **Evolution API Manager**, go to **Events Webhook**. Paste the n8n webhook URL, enable it, and activate the `webhook base64` (for media like audio/images) and `messages upsert` options (to trigger on incoming messages).
    *   **Test the webhook** by sending a message to your connected WhatsApp number. Once confirmed working, update to the production webhook URL in Evolution Manager.

2.  **Initial Data Processing & Lead Management:**
    *   Add an **Edit Fields (Set)** node to extract and set workflow variables like `nome`, `telefone`, `mensagem`, and `tipo_mensagem` from the incoming webhook data.
    *   Use a **Supabase (Get Rows)** node to check if the user (lead) already exists in your `leads_whatsapp` table using their `telefone` (session ID).
    *   Add an **If** node:
        *   If the `session_ID` is empty (new lead), use a **Supabase (Create Row)** node to add the new lead's `telefone` and `nome` to the `leads_whatsapp` table.
        *   If the `status` of the existing lead is `transferido`, or if the message came `from_me` (manual human intervention), route to a **No Operation** node to prevent the AI from responding.

3.  **Handle Different Message Types:**
    *   Add a **Switch** node that routes the workflow based on the `tipo_mensagem` variable (e.g., `conversation` for text, `audio_message` for audio, `image_message` for images).
    *   **For Text Messages:** Use an **Edit Fields (Set)** node to simply set the `mensagem` variable with the text content.
    *   **For Audio Messages:**
        *   Set a `data` variable with the Base64 audio string.
        *   Use a **Convert to File** node to convert the Base64 string to a file.
        *   Use an **OpenAI (Transcribe Audio)** node to convert the audio file to text, ensuring `file mp3` and `mime type mp3` are specified.
        *   Set the transcribed text to the `mensagem` variable.
    *   **For Image Messages:**
        *   Similar to audio, get the Base64 image data, convert to file (remember to clear `file name` and `mime type` parameters for images).
        *   Use an **OpenAI (Analyze Image)** node to generate a description of the image.
        *   Add an **If** node to check if the image also has a `caption`. If so, combine the caption and the AI's description into the `mensagem` variable. Otherwise, use only the description.

4.  **Message Buffering and Accumulation:**
    *   Connect all message type paths to a **Redis (Push Data to a Redis List)** node. The key for the list should be the user's `telefone`, and the value to push is the `mensagem`. Enable the `tail` option for ordering.
    *   Add a **Wait** node for 30 seconds to allow fragmented messages to accumulate.
    *   Retrieve all messages from the Redis list using a **Redis (Get the Value of a Key from Redis)** node, with `telefone` as the key.
    *   Implement a complex series of **If** nodes to ensure correct message accumulation: if the last message in Redis is *not* the current message being processed, it means a new message arrived, and the current workflow execution should be stopped (`No Operation`) to wait for the subsequent execution (which will have the full buffer).
    *   If it's the latest message (or no new messages arrived), concatenate all messages from the Redis list into a single `mensagem` variable (e.g., using `redis1.json.property_name.join(',')`).
    *   After accumulation, use a **Redis (Delete Key from Redis)** node to clear the message buffer for that user.

5.  **Main AI Agent Configuration:**
    *   Add an **AI Agent** node.
    *   Set the `User Message` to the concatenated `mensagem` variable.
    *   Select your **LLM** (e.g., OpenAI's GPT 4.1 Mini).
    *   Configure **Memory** to **Postgres**, using the user's `telefone` as the `Session ID` to maintain conversational context.
    *   **Prompt Engineering:**
        *   Define the agent's **Role and Objective** (e.g., "Julia, SDR from Adept," with expertise and an empathetic personality). Use `role prompting` and `emotion prompting` techniques.
        *   Specify the **Task**, breaking it into sequential sub-tasks (e.g., "greet," "collect name," "investigate problem," "propose appointment," "manage schedule") using `chain of thought prompting` and providing examples.
        *   Set **Specific Rules** for communication (e.g., "natural, clear, objective," "avoid repetition," "break objections," "always follow one step at a time," "never ask more than one question at a time"). Include current date and time.
        *   List **Available Tools** (created as sub-workflows in the next step) and their usage descriptions.
        *   Provide **Operational Context** (e.g., "you serve leads via WhatsApp, your goal is to qualify them and schedule diagnostic calls").
    *   Connect the **Tools** (sub-workflows) to this main AI Agent using the **Call Workflow Tool** option.

6.  **AI Agent for Response Formatting (Optional but Recommended):**
    *   Add a second **AI Agent** node for formatting the main agent's response.
    *   **System Prompt:** "You are a message formatting agent for WhatsApp. Your only function is to divide the received message into multiple short, natural, and well-structured messages.".
    *   The `User Message` input for this agent is the output from the main AI agent.
    *   Use a cheaper LLM (e.g., GPT 4.1 Nano) and set the **Temperature** to `0.1` for rigid prompt adherence.

7.  **Send Response Messages to WhatsApp:**
    *   Use an **Edit Fields (Set)** node to process the formatter's output. Store the formatted message as an array, splitting it by double newlines (`\n\n`) to create individual messages.
    *   Add a **Split Out** node to process each item in the array individually.
    *   Add a **Loop Over Items** node to iterate through and send each message.
    *   Inside the loop, add an **HTTP Request** node to send messages via the Evolution API:
        *   Set **HTTP Method** to **POST**.
        *   Construct the **URL** as `Evolution API URL/message/send/YOUR_INSTANCE_NAME`.
        *   Add the Evolution API key to the **Headers**.
        *   In the **Body (JSON)**, include the `number` (user's `telefone`) and the `text` (the current message from the loop, ensuring proper JSON formatting).
    *   Add a **Wait** node (e.g., 4-5 seconds) after each message to simulate human-like typing and reading.

### Step 5: Create Tools as Sub-Workflows

Each tool is a separate n8n workflow that is triggered by "When executed by another workflow".

1.  **Tool: Buscar Horários (Search for Available Times)**:
    *   **Trigger:** When executed by another workflow. Input: `telefone`.
    *   **Google Calendar (Get Many Events):** Search for busy events in the next few days (e.g., 3 days, until 23:59).
    *   Format event times (e.g., `dd/mm/yyyy hh:mm`) and aggregate them into a list.
    *   **OpenAI Message Model:** Process busy times and suggest 5 available slots. The prompt should specify working hours, exclude Sundays, suggest times at least 3 hours after the current time, and maintain a low temperature (e.g., 0.1) for rigid adherence to instructions.
2.  **Tool: Verificar Disponibilidade (Check Availability)**:
    *   **Trigger:** When executed by another workflow. Input: `time` (requested by lead), `telefone`.
    *   **OpenAI Message Model:** Standardize informal dates/times (e.g., "Tuesday at 10 AM") into UTC format (start time, end time). Use a more robust model like GPT-4.1 Mini for this task.
    *   **Google Calendar (Get Many Events):** Check for events in the standardized requested time frame. Ensure `always output data` is enabled in settings to avoid workflow breaks if no events are found.
    *   Return whether the requested time slot is available.
3.  **Tool: Agendar Horário (Schedule Appointment)**:
    *   **Trigger:** When executed by another workflow. Input: `telefone`, `horario` (chosen by lead), `email` (of lead), `nome_empresa` (of lead).
    *   **OpenAI Message Model:** Format the chosen time into UTC start/end times.
    *   **Google Calendar (Create Event):** Create a calendar event with summary, description (including lead's company), and add the lead's email as an attendee.
    *   **Supabase (Update Row):** Change the lead's `status` to `transferido` in the `leads_whatsapp` table.
    *   **(Optional) Internal Notification:** Use an **AI Agent** node to summarize the conversation and an **HTTP Request** node to send this summary as a notification to a human agent's fixed phone number via Evolution API.
4.  **Tool: Transferir Atendimento (Transfer to Human Agent)**:
    *   **Trigger:** When executed by another workflow. Input: `telefone`.
    *   **Supabase (Update Row):** Change the lead's `status` to `transferido`.
    *   **HTTP Request:** Send a notification to a human agent's fixed phone number (via Evolution API) with the lead's name and phone number.
5.  **Tool: Dados Adept (Retrieval Augmented Generation - RAG)**:
    *   **Knowledge Base Upload (Separate Workflow):**
        *   Use a **Manual Trigger** to initiate the upload.
        *   **Google Drive (Download File):** Download your knowledge base document (e.g., PDF).
        *   **Extract From File:** Extract text data from the PDF.
        *   **Supabase Vector Store (Add Document to Store):** Insert the extracted text into your `documents` table. Configure `text splitting` (recursive, chunk size 1000, chunk overlap 100) and choose an `embedding model` (e.g., `text-embedding-3-small`).
    *   **Tool Workflow:**
        *   **Trigger:** When executed by another workflow.
        *   **Supabase Vector Store (Get Many):** This node retrieves relevant information.
        *   Input: User query.
        *   Table name: `documents`. Limit the number of returned chunks (e.g., 4).
        *   Crucially, use the **same embedding model** that was used to upload the documents (e.g., `text-embedding-3-small`).

### Step 6: Activate and Optimize

1.  **Activate Workflow:** Once all components are set up, activate your main n8n workflow.
2.  **Test and Optimize:**
    *   Interact with your AI agent via WhatsApp and continuously test its responses and tool usage.
    *   **Review execution logs** in n8n to debug and understand the workflow's behavior.
    *   **Refine your prompts** for both the main AI agent and the formatting agent. Adjust rules, examples, and context to improve the AI's tone, communication, personality, and accuracy.
    *   Address formatting issues (e.g., preventing lists or extra newlines in WhatsApp messages) by explicitly instructing the formatting AI agent.