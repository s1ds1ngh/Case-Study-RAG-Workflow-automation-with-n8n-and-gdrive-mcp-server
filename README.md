# n8n AI Agent with Local MCP Integration (Docker + npx)

This repository contains an n8n workflow demonstrating how to integrate the Model Context Protocol (MCP) with a locally running n8n instance (via Docker) to enable AI Agents to dynamically discover and use external tools, such as web search, without needing persistent server installations.

This approach utilizes the `npx` command within n8n credentials to run MCP servers on-the-fly.

## Key Concepts

* **n8n AI Agent:** A powerful node in n8n that can reason, plan, and execute tasks using Large Language Models (LLMs) and available tools.
* **Model Context Protocol (MCP):** An open standard designed to simplify communication between AI models (like those used by the n8n AI Agent) and external tools, data sources, or APIs. It acts like a universal translator.
* **`npx` Method:** Allows running Node.js packages (like MCP servers) from the npm registry without permanently installing them. We leverage this within n8n's command line credentials.
* **Docker:** Used to run n8n in an isolated container environment locally.

## Goal

The primary goal of this workflow is to showcase:

1. Running n8n locally via Docker with the necessary flag (`N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE`) enabled for AI Agent tool usage.
2. Installing and utilizing the `n8n-nodes-mcp` community node.
3. Configuring n8n credentials to dynamically run an MCP server (e.g., Brave Search) using `npx`.
4. Building an n8n AI Agent workflow that can:
   * Discover available tools via MCP (`List Tools`).
   * Intelligently select the appropriate tool based on a user query and tool descriptions/schemas.
   * Execute the selected tool via MCP (`Execute Tool`) with parameters determined by the AI model.

## Prerequisites

* **Docker:** Installed and running on your local machine. [Docker Installation Guide](https://docs.docker.com/get-docker/)
* **Node.js & npm:** Recommended for potential troubleshooting and ensuring `npx` is available. [Node.js Installation Guide](https://nodejs.org/)
* **Basic n8n Knowledge:** Familiarity with creating workflows, adding nodes, and configuring credentials.
* **(Optional) API Keys:** If you plan to use MCP servers that require authentication (like Brave Search), you'll need the corresponding API key.

## Setup Steps

Follow these steps to get the environment and workflow running:

### Step 1: Run n8n Locally via Docker (with Tool Usage Enabled)

Open your terminal or command prompt and run the following Docker command:

```bash
docker run -it --rm --name n8n -p 5678:5678 \
-v n8n_data:/home/node/.n8n \
-e N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=true \
docker.n8n.io/n8nio/n8n
```

* **`-p 5678:5678`**: Maps the container's port 5678 to your local machine's port 5678.
* **`-v n8n_data:/home/node/.n8n`**: Creates a Docker volume named `n8n_data` to persist your n8n workflows and data.
* **`-e N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=true`**: **Crucial flag!** This environment variable allows the AI Agent node to treat community nodes (like the MCP Client) as executable tools.
* **`--rm`**: Automatically removes the container when it stops.
* **`-it`**: Runs the container interactively.

Wait for n8n to start. You can access it in your browser at `http://localhost:5678`.

### Step 2: Install the n8n MCP Community Node

1. In your n8n UI (at `http://localhost:5678`), navigate to **Settings** > **Community Nodes**.
2. Click **Install**.
3. Enter `n8n-nodes-mcp` in the search box.
4. Read and agree to the risks associated with community nodes.
5. Click **Install**.

### Step 3: Configure MCP Client Credential (using `npx`)

This example uses the Brave Search MCP server. You can adapt this for other servers by finding their respective `npx` commands.

1. Find the `npx` command for the desired MCP server. For Brave Search: `npx run -y @modelcontextprotocol/server-brave-search`
2. In your n8n workflow canvas, add an **MCP Client** node (from the community nodes section).
3. In the node parameters, click the dropdown for **Credential to connect with** and select **Create New Credential**.
4. Configure the credential:
   * **Connect using:** `Command Line (STDIO)`
   * **Command:** `npx`
   * **Arguments:** `run -y @modelcontextprotocol/server-brave-search` (or arguments for your chosen server)
   * **Environments:** (Optional but required for authenticated servers)
     * Click **Add Environment Variable**.
     * **Name:** `BRAVE_API_KEY` (or the variable name expected by the server)
     * **Value:** `YOUR_BRAVE_SEARCH_API_KEY_HERE` (*Replace with your actual key*)
   * **Credential Name:** Give it a descriptive name (e.g., `MCP Brave Search (npx)`)
5. Click **Save**.

## Workflow Explanation

Import the workflow JSON file provided in this repository into your n8n instance. The workflow consists of the following main nodes:

1. **When chat message received (Chat Trigger):** Starts the workflow when a message is sent via the n8n chat interface.
2. **AI Agent:** The core orchestrator.
   * **Chat Model:** Configured to use an LLM (e.g., Groq Chat Model, OpenAI, etc. - *ensure you have the corresponding credential configured in n8n*).
   * **Memory:** Connected to a `Simple Memory` node to retain conversation history within a session (Session ID linked from Chat Trigger).
   * **System Prompt:** Instructs the AI on how to behave, specifically how to identify and use tools via MCP:
     ```
     You are a helpful assistant

     1. Find all the tools available
     2. From the <user query> work out which tool is best for the job based on the descriptions, and pass the name of that tool to the third step, and for other params to be passed when executing the tool take reference from the schema section of the tool.
     3. Use executeTool, passing in the correct parameters, for executing the tool
     ```
3. **MCP Client Tool (List Tools):**
   * Connected to the AI Agent's `Tool` output handle.
   * Uses the `MCP Brave Search (npx)` credential (created in Step 3).
   * **Operation:** `List Tools`. This allows the AI Agent to ask "What tools can I use with this credential?".
4. **MCP Client Tool (Execute Tool):**
   * Also connected to the AI Agent's `Tool` output handle.
   * Uses the same `MCP Brave Search (npx)` credential.
   * **Operation:** `Execute Tool`.
   * **Tool Name:** Set dynamically using an expression like `{{ $fromAI('tool', 'selected tool to execute') }}` to get the tool name decided by the AI Agent.
   * **Tool Parameters:** Set to `Defined automatically by the model`. This allows the AI Agent to determine the necessary parameters (like the search query) based on the tool's schema (which it learned from the List Tools step) and the user's request.

## Usage / Testing

1. **Activate** the imported n8n workflow.
2. Click the **Chat** button in the n8n UI (usually bottom right).
3. **Test Tool Discovery:** Type `What tools do you have?` or `list available tools`. The AI Agent should interact with the "List Tools" MCP node and respond with the available Brave Search tools (e.g., `brave_web_search`, `brave_local_search`), their descriptions, and expected parameters.
4. **Test Tool Execution:** Ask a question that requires web search, for example: `Tell me about the latest developments in the Model Context Protocol.`
5. **Observe:**
   * **Chat:** The AI Agent should respond with information gathered using the Brave Search tool.
   * **n8n Executions Log:** Examine the workflow execution. You should see the AI Agent node making decisions, and the "Execute Tool" MCP Client node being called with the `tool` set to `brave_web_search` and `Tool_Parameters` containing your query.

## Troubleshooting / Notes

* **`N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=true`:** Ensure this environment variable is correctly set when starting the Docker container. Without it, the AI Agent cannot use the MCP Client node as a tool.
* **`npx` Command:** Double-check the `npx` command and arguments for the specific MCP server you are trying to run. Ensure the package name (`@modelcontextprotocol/server-brave-search`) is correct.
* **API Keys:** Verify that any required API keys (like `BRAVE_API_KEY`) are correctly added to the **Environments** section of the MCP Client credential in n8n and that the key itself is valid.
* **Firewall:** Ensure your local machine's firewall allows `npx` to download and run packages, and allows connections if the MCP server needs to reach external APIs.
* **LLM Credentials:** Make sure the Chat Model credential used in the AI Agent node (e.g., Groq, OpenAI) is correctly configured and valid.
* **Docker Volume:** Using the `-v n8n_data:/home/node/.n8n` volume ensures your work is saved even if you stop and restart the container.

## Conclusion

This workflow demonstrates a flexible and efficient way to extend n8n AI Agents using the Model Context Protocol and the `npx` execution method. It lowers the barrier to integrating external tools by avoiding the need for persistent server setups for many common utilities, allowing your AI Agents to interact with external information and services dynamically.
