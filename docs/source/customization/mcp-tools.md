<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->
# MCP Tools

Model Context Protocol (MCP) is an open protocol that standardizes how applications provide context to LLMs. You can use MCP to connect the AIQ Blueprint to external tools and data sources served by remote MCP servers, without writing any custom Python code. Since the AIQ Blueprint is built on the NVIDIA NeMo Agent toolkit, MCP integration is available through configuration.

For the full MCP documentation, refer to the [NeMo Agent Toolkit MCP Client Guide](https://docs.nvidia.com/nemo/agent-toolkit/latest/).

## Prerequisites

Install MCP support if it is not already available:

```bash
uv pip install "nvidia-nat[mcp]"
```

## Adding MCP Tools

Use `mcp_client` to connect to an MCP server and make its tools available to the deep researcher. The `mcp_client` automatically discovers all tools served by the MCP server and registers them as functions.

### Step 1: Define the MCP client in the `function_groups` section

Add a `function_groups` section to your config (at the same level as `functions`):

```yaml
function_groups:
  mcp_financial_tools:
    _type: mcp_client
    server:
      transport: streamable-http
      url: "http://localhost:9901/mcp"
```

This connects to the MCP server at the given URL and registers all of its tools under the group name `mcp_financial_tools`.

**Transport options:**

- `streamable-http` (recommended): modern HTTP-based transport for new deployments
- `sse`: Server-Sent Events, supported for backwards compatibility
- `stdio`: standard input/output for local process communication

### Step 2: Add the function group to each agent's `tools` list

The agents will not use the MCP tools unless the function group appears in their `tools` list. Add it to the agents that should have access:

```yaml
# (inside the existing functions: section)
  intent_classifier:
    _type: intent_classifier
    tools:
      - web_search_tool
      - knowledge_search
      - mcp_financial_tools

  clarifier_agent:
    _type: clarifier_agent
    tools:
      - web_search_tool
      - knowledge_search
      - mcp_financial_tools

  shallow_research_agent:
    _type: shallow_research_agent
    tools:
      - web_search_tool
      - knowledge_search
      - mcp_financial_tools

  deep_research_agent:
    _type: deep_research_agent
    tools:
      - advanced_web_search_tool
      - knowledge_search
      - mcp_financial_tools
```

## Filtering and Renaming Tools

By default, `mcp_client` exposes all tools from the MCP server. You can rename or override descriptions for specific tools using `tool_overrides`:

```yaml
function_groups:
  mcp_financial_tools:
    _type: mcp_client
    server:
      transport: streamable-http
      url: "http://localhost:9901/mcp"
    tool_overrides:
      get_stock_quote:
        alias: "stock_price"
        description: "Returns the current stock price for a given ticker symbol."
      get_earnings_report:
        description: "Returns the latest quarterly earnings report for a company."
```

A complete example config is available at `configs/config_web_frag_mcp.yml`.

## Authenticated MCP Tools

MCP tools that require OAuth2 authentication (for example, corporate Jira, Confluence, or internal data platforms) are not supported in the current version of the AIQ Blueprint. The NeMo Agent toolkit provides an `mcp_oauth2` authentication provider, but it is not yet compatible with the blueprint's backend and frontend. Support for authenticated MCP tools is planned for an upcoming release.

For non-authenticated MCP servers, or MCP servers that use service account credentials (set through environment variables on the server side), use the `mcp_client` approach described above.

## UI Limitations

MCP tools added through the configuration file will be available to the agents at the backend level, but they will not automatically appear in the demo UI. Displaying custom MCP tools in the UI requires changes to both the backend and frontend. Built-in support for this is planned for an upcoming version. In the meantime, the demo UI source code is provided and can be modified to surface additional tools as needed.

## Prompt Tuning

Adding MCP tools to the config makes them available to the agents, but the agents' prompts may not reference them. For the agents to use MCP tools effectively, you should tune the relevant prompts so that the agent knows when and how to invoke the new tools. Each customization is different: the prompt changes depend on the tool's purpose and how it fits into the research workflow.

The NeMo Agent toolkit agents use tool descriptions for routing decisions. If the MCP server provides poor or generic tool descriptions, you can override them through the `tool_overrides` configuration to help the agent select the right tool for each query.

For more on prompt customization, refer to [Prompts](./prompts.md).

## Discovering MCP Tools

You can list the tools served by any MCP server:

```bash
nat mcp client tool list --url http://localhost:9901/mcp
```

To get details about a specific tool:

```bash
nat mcp client tool list --url http://localhost:9901/mcp --tool get_financial_data
```
