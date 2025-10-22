# Kubernetes MCP Server Test Examples

This directory contains examples for testing the **same Kubernetes MCP server** using different AI agents.

## Structure

```
kube-mcp-server/
├── README.md                    # This file
├── claude-code/                 # Test with Claude Code
│   ├── agent.yaml
│   ├── eval.yaml
│   ├── mcp-config.yaml
│   └── tasks/
│       ├── create-pod.yaml
│       ├── setup.sh
│       ├── verify.sh
│       └── cleanup.sh
└── openai-agent/                # Test with OpenAI-compatible agents
    ├── agent.yaml
    ├── agent-wrapper.sh
    ├── eval.yaml
    ├── mcp-config.yaml
    └── tasks/
        ├── create-pod.yaml
        ├── setup.sh
        ├── verify.sh
        └── cleanup.sh
```

## What This Tests

Both examples test the **same Kubernetes MCP server** running at `http://localhost:8080/mcp`:
- Creates an nginx pod named `web-server` in the `create-pod-test` namespace
- Verifies the pod is running
- Validates that the agent called appropriate Kubernetes tools
- Cleans up resources

The only difference is **which AI agent** executes the task.

## Prerequisites

- Kubernetes cluster (kind, minikube, or any cluster)
- kubectl configured
- Kubernetes MCP server running at `http://localhost:8080/mcp`
- Built binaries: `gevals` and `agent`

## Running Examples

### Option 1: Claude Code

```bash
./gevals run examples/kube-mcp-server/claude-code/eval.yaml
```

**Requirements:**
- Claude Code installed and in PATH

**Tool Usage:**
- Claude typically uses pod-specific tools like `pods_run`, `pods_create`

---

### Option 2: OpenAI-Compatible Agent (Built-in)

```bash
# Set your model credentials
export MODEL_BASE_URL='https://your-api-endpoint.com/v1'
export MODEL_KEY='your-api-key'
export MODEL_NAME='your-model-name'

# Run the test
./gevals run examples/kube-mcp-server/openai-agent/eval.yaml
```

**Tested Models:**
- ✅ **Mistral Small 24B** - uses `pods_run`
- ✅ **Llama 4 Scout 17B** - uses `resources_create_or_update`
- ❌ **Llama 3.2 3B** - too small, struggles with tool calling

**Tool Usage:**
- Different models may choose different tools (`pods_*` or `resources_*`)
- Both approaches work correctly

## Assertions

Both examples use flexible assertions that accept either tool approach:

```yaml
toolPattern: "(pods_.*|resources_.*)"  # Accepts both pod-specific and generic resource tools
```

This makes the tests robust across different AI models that may prefer different tools.

## Key Difference: Agent Configuration

### Claude Code (claude-code/agent.yaml)
```yaml
commands:
  argTemplateMcpServer: "--mcp-config {{ .File }}"
  argTemplateAllowedTools: "mcp__{{ .ServerName }}__{{ .ToolName }}"
  runPrompt: |-
    claude {{ .McpServerFileArgs }} --print "{{ .Prompt }}"
```

### OpenAI Agent (openai-agent/agent.yaml)
```yaml
commands:
  argTemplateMcpServer: "{{ .File }}"
  runPrompt: |-
    examples/kube-mcp-server/openai-agent/agent-wrapper.sh {{ .McpServerFileArgs }} "{{ .Prompt }}"
```

The wrapper script extracts the proxy URL from the config file since the `agent` binary requires `--mcp-url` directly.

## Expected Results

Both examples should produce:
- ✅ Task passed - pod created successfully
- ✅ Assertions passed - appropriate tools were called
- ✅ Verification passed - pod exists and is running

Results saved to: `gevals-<eval-name>-out.json`
