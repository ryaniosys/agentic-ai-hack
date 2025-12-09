# Agentic AI Hackathon: AI Coding Agent Instructions

## Project Overview
This is a **multi-agent insurance claims processing system** using Azure AI services. The architecture spans 5 progressive challenges that build from foundational document processing to a complete agent orchestration platform.

**Core Pattern**: Sequential challenges → Specialized agents (Claim Reviewer, Policy Checker, Risk Analyzer) → Master orchestrator in Challenge 5.

## Architecture & Data Flow

### Challenge Progression & Key Components
- **Challenge 0**: Infrastructure setup (Azure resources in `challenge-0/iac/`)
- **Challenge 1**: Document processing pipeline (text + image via GPT-4-mini vision, indexing in Azure AI Search)
- **Challenge 2**: First agent creation (Azure AI Agent Service + portal configuration)
- **Challenge 3**: Observability & evaluation (Azure AI Foundry metrics)
- **Challenge 4**: Specialized agents with Semantic Kernel plugins (Claim Reviewer, Risk Analyzer, Policy Checker)
- **Challenge 5**: Agent orchestration with GroupChat pattern (`challenge-5/main.py`)

### Key Services & Integration Points
| Service | Role | Location in Code |
|---------|------|-----------------|
| **Azure AI Foundry** | Model, agent, and evaluation hosting | Agent IDs in `.env`, endpoints referenced in `challenge-5/main.py` |
| **Azure Cosmos DB** | Insurance claims & crash reports storage | `CosmosDBPlugin` in `challenge-5/agents/tools.py` |
| **Azure AI Search** | Vectorized policy & document search | Challenge 1 notebooks |
| **Azure Blob Storage** | Document upload & retrieval | Challenge 1 setup |
| **Azure OpenAI** | GPT-4 model deployment | `AZURE_OPENAI_DEPLOYMENT_NAME` env var |

### Data Flow Pattern
1. Documents → Blob Storage → GPT-4-mini processing → AI Search vectorization
2. Claims → Cosmos DB (structured JSON documents)
3. Agents query via **Semantic Kernel plugins** (not direct REST calls) → Cosmos DB for claim retrieval
4. Master agent orchestrates via **GroupChatOrchestration** (Challenge 5)

## Critical Development Workflows

### Environment Setup
```bash
# Root directory setup
az login --use-device-code
cd challenge-0 && ./get-keys.sh --resource-group YOUR_RG_NAME  # Populates .env at repo root

# Challenge-specific setup (especially Challenge 5)
cd challenge-5
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
# Copy .env from root to challenge-5 directory
```

### Agent Development (Challenge 4 & 5)
- **Portal-based agents**: Create in Azure AI Foundry UI, retrieve IDs, store in `.env` as `CLAIM_REV_AGENT_ID`, `RISK_ANALYZER_AGENT_ID`, `POLICY_CHECKER_AGENT_ID`
- **Semantic Kernel integration**: Agents loaded via `AzureAIAgent(client=client, definition=client.agents.get_agent(agent_id=...))` in `main.py`
- **Custom plugins**: Inherit from Semantic Kernel patterns; see `CosmosDBPlugin` using `@kernel_function` decorator

### Running Challenge 5 API
```bash
cd challenge-5
func host start  # Uses FastAPI + Azure Functions runtime; requires pip install from requirements.txt
```

## Code Patterns & Conventions

### Semantic Kernel Plugins (Challenge 4+)
```python
# Location: challenge-5/agents/tools.py pattern
from semantic_kernel.functions import kernel_function

class CosmosDBPlugin:
    @kernel_function(description="Human-readable tool description")
    def method_name(self) -> Annotated[str, "return type hint"]:
        """Tool logic here"""
```
**Important**: Plugins are passed to agents via `plugins=[plugin_instance]` parameter. Agents invoke them via tool use.

### Agent Orchestration (Challenge 5)
```python
# Location: challenge-5/main.py
from semantic_kernel.agents import GroupChatOrchestration, RoundRobinGroupChatManager

# 1. Create specialized agents with plugins
claim_reviewer_agent = AzureAIAgent(client=client, definition=..., plugins=[cosmos_plugin])

# 2. Create final decision agent
approver_agent = ChatCompletionAgent(
    name="ApproverAgent",
    instructions="Make final APPROVED/DENIED decision",
    service=AzureChatCompletion(...)
)

# 3. Orchestrate via GroupChat
runtime = InProcessRuntime()
manager = RoundRobinGroupChatManager(agents=[reviewer, analyzer, checker, approver])
```
**Pattern**: Agents run sequentially in round-robin; approver makes final decision based on all outputs.

### Environment Variables (Challenge 0+)
All stored in `.env` at repo root; challenges inherit via `load_dotenv(override=True)`:
- `AI_FOUNDRY_PROJECT_ENDPOINT`: Azure AI Foundry endpoint
- `AZURE_OPENAI_DEPLOYMENT_NAME`: GPT-4 deployment name
- `COSMOS_ENDPOINT`, `COSMOS_KEY`: Cosmos DB credentials
- `CLAIM_REV_AGENT_ID`, `RISK_ANALYZER_AGENT_ID`, `POLICY_CHECKER_AGENT_ID`: Agent IDs from portal
- `AZURE_SEARCH_ENDPOINT`, `AZURE_SEARCH_KEY`: Search service credentials

## Project-Specific Conventions

### File Organization
- **Challenge notebooks**: Jupyter notebooks (`*.ipynb`) are primary teaching/execution format; each challenge builds incrementally
- **Challenge 5 API**: Only challenge with FastAPI (`main.py`) and production code; others are exploratory notebooks
- **Plugins**: Always in `agents/tools.py` within respective challenge directory; inherit `@kernel_function` decorator

### Async Patterns (Challenge 5)
```python
# Main.py uses async/await throughout
async def get_specialized_agents() -> list[Agent]:
    async with DefaultAzureCredential() as creds:
        client = AzureAIAgent.create_client(credential=creds, endpoint=endpoint)
        # Load agents asynchronously
```
**Key**: Must use `DefaultAzureCredential()` for local dev; runs in browser during Codespaces.

### Request/Response Format (Challenge 5 API)
```python
class ClaimRequest(BaseModel):
    claimId: str
    policyNumber: str

# Response
{"decision": "APPROVED" or "DENIED", "justification": "..."}
```

### Notebook Execution Order
- Challenge 1: `1.document-processing.ipynb` → `2.document-vectorization.ipynb` (sequential)
- Challenge 4: `claim_reviewer.ipynb`, `risk_analyser.ipynb`, then check `solution/` folder for reference
- Challenge 5: `orchestration.ipynb` (shows orchestration pattern), then `main.py` for API

## Common Pitfalls & Solutions

1. **Missing agent IDs**: Challenge 5 requires agent IDs from Azure AI Foundry portal. If `.env` lacks `CLAIM_REV_AGENT_ID` etc., agents won't load. Create via UI first.
2. **Cosmos DB connectivity**: `CosmosDBPlugin` expects `COSMOS_ENDPOINT` and `COSMOS_KEY` in `.env`. Check via `.test_connection()` method.
3. **Wrong Python version**: Challenges use Python 3.11 (`python3.11 -m venv`); some systems default to 3.10.
4. **Async credential issues**: Use `DefaultAzureCredential()` for Azure CLI login; avoid `AzureCliCredential` in Challenge 5 main.py.
5. **Challenge 5 API requires correct plugin instantiation**: Plugins must be instances (not classes) passed to agents.

## Tools & Extensions Reference
- **Dev container**: Includes Python 3.11, Azure CLI, Docker, Bicep (see `.devcontainer/devcontainer.json`)
- **Jupyter support**: Pre-installed; use VS Code Jupyter extension
- **Azure Functions**: Challenge 5 uses `func host start` for local testing

## Key Files for Reference
- Architecture & resource definitions: `challenge-0/iac/azuredeploy.bicep`
- Document processing foundation: `challenge-1/{1,2}.document-*.ipynb`
- Agent orchestration example: `challenge-5/orchestration.ipynb`
- Production API: `challenge-5/main.py` (FastAPI + agent orchestration)
- Plugin template: `challenge-5/agents/tools.py` (Semantic Kernel CosmosDB plugin)
