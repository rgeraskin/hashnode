---
title: "AI with Kubernetes: Operations for Developers 🤖"
seoTitle: "Kubernetes and AI: Developer Operations Guide"
seoDescription: "Discover beginner-friendly AI tools like Claude, Cursor, and K9s with HolmesGPT to simplify Kubernetes cluster management using natural language"
datePublished: Fri Mar 28 2025 23:27:08 GMT+0000 (Coordinated Universal Time)
cuid: cm8tewpbi000109jl936c3w5y
slug: ai-with-kubernetes
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1743204874819/2a7738b0-2d70-4332-96a7-c208afef6a25.png
tags: ai, kubernetes, devops, cursor, chatgpt, claudeai

---

Artificial Intelligence is making Kubernetes cluster management accessible to everyone, even those without deep Kubernetes expertise. By integrating AI tools through the Model Context Protocol (MCP), beginners can interact with clusters using natural language, avoiding complex `kubectl` commands.

This article explores three beginner-friendly approaches — [Claude Desktop](https://claude.ai/download), [Cursor](https://www.cursor.com), and [K9S](https://k9scli.io) with [HolmesGPT](https://github.com/robusta-dev/holmesgpt) — each designed to simplify Kubernetes management for newcomers.

> The Model Context Protocol (MCP) is a powerful framework for integrating AI models into development environments. MCP enables seamless communication between AI systems and tools like Kubernetes by providing a standardized way to pass context and commands. With MCP, you can query cluster states, automate resource management, or even troubleshoot issues using natural language. To use MCP effectively with Kubernetes, you’ll need a compatible server setup that bridges your AI tool with kubectl, the Kubernetes command-line interface.

## Approach 1: Claude Desktop

##### In my opinion, [Claude Desktop](https://claude.ai/download) is an excellent solution for communicating with MCP servers today because of its user-friendly interface. I will use `mcp-server-kubernetes` to link Claude with Kubernetes.

### Step 1: Configure Claude Desktop

1. ##### Configure Claude Desktop MCP Settings: go to **Settings =&gt; Developer** menu to add MCP server or just edit `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) with:
    

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "npx",
      "args": ["mcp-server-kubernetes"]
    }
  }
}
```

2. Save and restart Claude.
    

### Step 2: Chat with Your Cluster

Open a chat in Claude and ask something like:

* “List deployments in default k8s ns”
    
* “Show me nginx logs”
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1743199513208/76eeebd2-8809-4198-aa6a-eae8ffceae62.png align="center")

> See full example conversation [here](https://claude.ai/share/6c2c4d8d-43ae-422f-bcfa-c12dec42913a).

Claude uses `mcp-server-kubernetes` to handle the technical details, making Kubernetes approachable for beginners by translating plain English into actions.

## Approach 2: Cursor IDE

[Cursor](https://www.cursor.com) is a VS Code fork designed to be a more AI-focused code editor. While it may not be as convenient as Claude, if you spend a lot of time in an IDE, it can be useful to have Kubernetes access with natural language right in the same window.

### Step 1: Install mcp server

Now I will use [kubectl-mcp-server](https://github.com/rohitg00/kubectl-mcp-server). Also, the mcp server from previous approach could be used too.

```bash
pipx install kubectl-mcp-tool
```

### Step 2: Configure Cursor

1. Go to **Cursor =&gt; Settings =&gt; Cursor Settings =&gt; MCP**.
    
2. Click **"Add new global MCP server"**.
    
3. Add:
    

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "~/.local/pipx/venvs/kubectl-mcp-tool/bin/python",
      "args": [
        "-m",
        "kubectl_mcp_tool.minimal_wrapper"
      ]
    }
  }
}
```

### Step 3: Interact with Your Cluster

Open chat in Cursor and use prompts like:

* “List all pods in default k8s ns”
    
* “What is the status of my nginx k8s deployment?”
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1743200348371/0bb49fa0-b04b-491c-b0f9-feab1d68ee28.png align="center")

## Approach 3: Using K9s with HolmesGPT

[HolmesGPT](https://github.com/robusta-dev/holmesgpt), from Robusta, is a tool that simplifies Kubernetes troubleshooting. It investigates issues automatically, requiring no prior expertise. Use it via the Robusta SaaS platform or CLI with queries like `holmes ask "what pods are unhealthy and why?"`.

We can integrate it with k9s, a terminal-based UI that allows you to interact with your Kubernetes clusters. It's a popular tool for working with Kubernetes, making it easier to navigate, observe, and manage deployed applications.

Actually, this approach is different from the previous ones. It doesn’t use MCP. However, it is powerful, so I have to mention it.

### Step 1: Configuring K9s with HolmesGPT

1. Install HolmesGPT ([instructions](https://github.com/robusta-dev/holmesgpt/blob/master/docs/installation.md#cli-installation))
    
2. ```bash
    # for mac
    brew tap robusta-dev/homebrew-holmesgpt
    brew install holmesgpt
    ```
    

2. Export OpenAI API key, for example with `~/.zshrc`: `export OPENAI_API_KEY=XXX`
    
    I bet you can easily Google how to get an API key. Certainly, you can use [other AIs](https://github.com/robusta-dev/holmesgpt/blob/master/docs/api-keys.md).
    
3. Add Holmes as plugin to k9s: edit `~/Library/Application Support/k9s/plugins.yaml` (mac). See detailed instructions [here](https://github.com/robusta-dev/holmesgpt/blob/master/docs/k9s.md).
    

```yaml
plugins:
  holmesgpt:
    shortCut: Shift-H
    description: Ask HolmesGPT
    scopes:
      - all
    command: bash
    background: false
    confirm: false
    args:
      - -c
      - |
        holmes ask "why is $NAME of $RESOURCE_NAME in namespace $NAMESPACE not working as expected?"
        echo "Press 'q' to exit"
        while : ; do
        read -n 1 k <&1
        if [[ $k = q ]] ; then
        break
        fi
        done
  custom-holmesgpt:
    shortCut: Shift-Q
    description: Custom HolmesGPT Ask
    scopes:
      - all
    command: bash
    background: false
    confirm: false
    args:
      - -c
      - |
        INSTRUCTIONS="# Edit the line below. Lines starting with '#' will be ignored."
        DEFAULT_ASK_COMMAND="why is $NAME of $RESOURCE_NAME in namespace $NAMESPACE not working as expected?"
        QUESTION_FILE=$(mktemp)

        echo "$INSTRUCTIONS" > "$QUESTION_FILE"
        echo "$DEFAULT_ASK_COMMAND" >> "$QUESTION_FILE"

        # Open the line in the default text editor
        ${EDITOR:-nano} "$QUESTION_FILE"

        # Read the modified line, ignoring lines starting with '#'
        user_input=$(grep -v '^#' "$QUESTION_FILE")
        echo running: holmes ask "\"$user_input\""

        holmes ask "$user_input"
        echo "Press 'q' to exit"
        while : ; do
        read -n 1 k <&1
        if [[ $k = q ]] ; then
        break
        fi
        done
```

4. Start K9s, select pod and ask Holmes with `Shift-H` or `Shift-Q`
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1743201575892/8c713aca-2265-47c9-9ddd-55e478e9513b.png align="center")

HolmesGPT can do much more. Check the [repository](https://github.com/robusta-dev/holmesgpt) for more details. Also, there are other tools, like [k8sgpt](https://k8sgpt.ai), that can make the life easier. Maybe some of them can be integrated with Lens too.

## Why This Matters for Beginners

These tools democratize Kubernetes:

* **Claude**: A desktop app where anyone can chat with their cluster.
    
* **Cursor**: An IDE that lets developers code and manage k8s without expertise.
    
* **K9s**: A visual tool with AI to troubleshoot effortlessly.
    

Needless to say, all of this is not meant for production use. Moreover, MCP servers are still full of bugs and have limited functionality. However, it can still be useful for local development.

## Conclusion

Claude Desktop, Cursor, and K9s with HolmesGPT empower beginners to manage Kubernetes clusters with AI, no solid K8s experience required. They turn tasks into simple conversations, making DevOps accessible to all. Try them out and start exploring Kubernetes!