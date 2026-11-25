# AI Product Launch Intelligence Agent

## 1. Project Overview

This project is a multi-agent AI application designed to analyze product launches, market sentiment, and launch performance.

It uses Agno to coordinate specialized AI agents, Horizon as the language model, Firecrawl for web research, and Streamlit for the user interface.

The objective is to improve an existing application by applying Agno documentation and best practices, focusing on reliability, maintainability, and resource management.

## 2. Original Project

This project is based on the Product Launch Intelligence Agent from the Awesome LLM Apps repository.

[Original GitHub repository](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/advanced_ai_agents/multi_agent_apps)

## 3. Multi-Agent Architecture

The application consists of three specialized agents coordinated by a Team.

| Component | Responsibility |
|---|---|
| Product Launch Analyst | Competitive positioning, launch strategies, strengths, and weaknesses. |
| Market Sentiment Specialist | Customer feedback, social media sentiment, and brand perception. |
| Launch Metrics Specialist | KPIs, user adoption, and launch performance. |
| Product Intelligence Team | Task delegation, agent coordination, and report generation. |

## 4. Technologies

- Python
- Agno 3.2.1
- Horizon (OpenAILike)
- Firecrawl
- Streamlit
- Python-dotenv
- Git

## 5. Refactoring Improvements

### 5.1. Correct Team Response Typing

**Problem:** The original project used `RunOutput` for Team responses, although this type is intended for individual Agent executions.

**Before:**

```python
from agno.run.agent import RunOutput

resp: RunOutput = product_intelligence_team.run(prompt)
```

**After:**

```python
from agno.run.team import TeamRunOutput

resp: TeamRunOutput = product_intelligence_team.run(prompt)
```

This correction was applied to all six Team executions, including intermediate analysis responses.

**Benefits:** Correct type annotations, consistency with Agno, and easier maintenance.

**Concept:** Type Safety.

### 5.2. Separation of Agent Roles, Descriptions, and Instructions

**Problem:** Agent expertise, responsibilities, and instructions were combined inside the `description` parameter.

**Before (simplified):**

```python
sentiment_analyst = Agent(
    name="Market Sentiment Specialist",
    description=dedent("""
        You are a market research expert...
        Your expertise includes:
        ...
    """),
)
```

**After (simplified):**

```python
sentiment_analyst = Agent(
    name="Market Sentiment Specialist",
    role="Analyze market sentiment and consumer perception.",
    description="You are a market research expert specializing in sentiment analysis and consumer perception tracking.",
    instructions=[
        "Analyze social media sentiment and customer feedback.",
        "Identify positive and negative sentiment drivers.",
        "Track brand perception trends across platforms.",
        "Monitor customer satisfaction and review patterns.",
        "Provide actionable insights on market reception.",
    ],
)
```

This structure was applied to all three agents while retaining their business instructions. The unused `dedent` import was also removed.

**Benefits:** Clearer responsibilities, better context organization, and clearer task delegation.

**Concepts:** Context Engineering, Agent Delegation.

### 5.3. Retry Configuration

**Problem:** Exponential backoff was configured without explicitly enabling additional retry attempts.

**Before:**

```python
exponential_backoff=True,
delay_between_retries=2,
```

**After:**

```python
retries=3,
exponential_backoff=True,
delay_between_retries=2,
```

This configuration was applied to all three agents.

**Benefits:** Improved resilience against supported temporary execution errors.

**Concepts:** Fault Tolerance, Exponential Backoff.

### 5.4. Centralized Debugging

**Problem:** The debug configuration was repeated in each agent and in the Team.

**Before:**

```python
agent = Agent(
    ...
    debug_mode=True,
)

team = Team(
    ...
    debug_mode=True,
    show_members_responses=True,
)
```

**After:**

```python
agent = Agent(
    ...
)

team = Team(
    ...
    debug_mode=True,
    show_members_responses=True,
)
```

The individual debug declarations were removed from all three agents, using Agno's native Team debug propagation.

**Benefits:** Reduced duplication and centralized debugging configuration.

**Concepts:** DRY, Debugging.

### 5.5. Simplified Response Content Extraction

**Problem:** The original code used unnecessary `hasattr()` checks, although `TeamRunOutput` already provides a `content` attribute.

**Before:**

```python
bullets.content if hasattr(bullets, "content") else str(bullets)

return resp.content if hasattr(resp, "content") else str(resp)
```

**After:**

```python
bullets.content or ""

return resp.content or ""
```

After introducing empty response validation in the next refactoring, intermediate responses are accessed directly using `bullets.content`.

**Benefits:** Simpler response handling and improved readability.

**Concept:** Clean Code.

### 5.6. Empty Response Validation

**Problem:** The application could execute a second Team run even when the initial response was empty.

**Before:**

```python
long_text = expand_competitor_report(
    bullets.content or "",
    company_name
)
```

**After:**

```python
if not bullets.content:
    raise ValueError("No analysis content returned.")

long_text = expand_competitor_report(
    bullets.content,
    company_name
)
```

This validation was added to all three analysis types. Errors are handled by the existing `try/except` blocks.

**Benefits:** Prevents unnecessary Team executions and report generation without input data.

**Concepts:** Defensive Programming, Error Handling.

### 5.7. Tool Call Limitation

**Problem:** The agents had no explicit limit on the number of tool calls.

**Before:**

```python
tools=[FirecrawlTools(
    enable_search=True,
    enable_crawl=True,
    poll_interval=10
)],
```

**After:**

```python
tools=[FirecrawlTools(
    enable_search=True,
    enable_crawl=True,
    poll_interval=10
)],
tool_call_limit=10,
```

The native Agno parameter was added to all three agents. The limit applies to each agent run, not to the total number of Firecrawl credits.


**Benefits:** Better control over tool usage and potential reduction of unnecessary API consumption.

**Concepts:** Tool Management, Resource Management.

### 5.8. Horizon Integration and Centralized LLM Configuration

**Problem:** The original project used GPT-4o, repeated model initialization, and managed API keys through the Streamlit sidebar.

**Before:**

```python
from agno.models.openai import OpenAIChat

model=OpenAIChat(id="gpt-4o")
```

**After:**

```python
from agno.models.openai.like import OpenAILike

model = OpenAILike(
    id=os.environ["IFM_MODEL"],
    api_key=os.environ["IFM_API_KEY"],
    base_url=os.environ["IFM_BASE_URL"],
)
```

A single model instance is reused by the three agents and the Team:

```python
model=model,
```

The configuration was externalized into a `.env` file and loaded using an explicit path:

```python
load_dotenv(Path(__file__).with_name(".env"))
```

Required environment variables are validated before initializing the multi-agent system:

```python
required_keys = (
    "IFM_MODEL",
    "IFM_API_KEY",
    "IFM_BASE_URL",
    "FIRECRAWL_API_KEY",
)

missing_keys = [
    key for key in required_keys
    if not os.getenv(key)
]
```

The `.env` file is excluded from Git through `.gitignore`.

**Benefits:** Horizon integration, reduced configuration duplication, safer API key management, and easier deployment.

**Concepts:** Model Integration, DRY, Configuration Management, Security.

## 6. Project Structure

```text
product-launch/
├── product_launch_intelligence_agent.py
├── requirements.txt
├── .env.example
├── .gitignore
└── README.md
```

## 7. Installation

Create a virtual environment:

```bash
python -m venv .venv
```

Activate it on Windows:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the dependencies:

```bash
pip install -r requirements.txt
```

## 8. Environment Configuration

Create a `.env` file using `.env.example` as a template:

```env
IFM_MODEL=
IFM_API_KEY=
IFM_BASE_URL=
FIRECRAWL_API_KEY=
```

Fill in the required configuration using your own credentials. Never commit the `.env` file to Git.

## 9. Running the Application

```bash
streamlit run product_launch_intelligence_agent.py
```

Enter a company name and select an analysis type:

- Competitor Analysis
- Market Sentiment
- Launch Metrics

## 10. Testing and Results

Functional testing was performed using Spotify as the target company.

### Functional Tests

| Analysis | Result | Firecrawl Credits |
|---|---|---:|
| Competitor Analysis | Report generated | 54 |
| Market Sentiment | Report generated | 32 |
| Launch Metrics | Report generated | 71 |
| **Total** | **3/3 reports generated** | **157** |

### Resource Optimization

During the Competitor Analysis test, Firecrawl credit consumption decreased from 92 to 54 credits, representing an observed reduction of 41.3%, assuming equivalent test conditions.

The logs also confirmed that Agno executed 10 tool calls and blocked 3 additional calls during a Market Sentiment Specialist run.

## 11. Conclusion

The eight improvements enhance code maintainability, context engineering, fault tolerance, configuration management, and resource control while preserving the application's main functionalities.
