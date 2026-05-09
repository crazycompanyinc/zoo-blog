# How to Monitor AI Agents in Production: The Complete Observability Stack (2026)

**Reading time: 18 minutes | Level: Intermediate-Advanced | Code: Complete, production-ready**

You shipped your AI agent to production. Users love it. Then at 3 AM, your phone buzzes: the agent is stuck in an infinite loop, burning through $400 in API credits per hour, and hallucinating responses to paying customers.

Sound familiar? It happened to us. Multiple times.

Here's the uncomfortable truth: **most teams building AI agents have zero observability**. They deploy and pray. When something breaks, they find out from angry users — not from alerts.

This guide is the exact observability stack we built at ZOO after a $2,400 incident that could have been caught in 30 seconds with proper monitoring. We're giving you everything: the architecture, the code, the dashboards, and the alerting rules.

By the end, you'll have:

- **Full trace visibility** into every agent decision, tool call, and LLM invocation
- **Cost tracking per user, per agent, per tool** — know exactly where money goes
- **Latency breakdowns** that show you WHERE time is spent (embedding? LLM? tool execution?)
- **Quality metrics** that catch hallucinations and degraded outputs BEFORE users complain
- **Alerting rules** that wake you up when it matters, not on every false positive

**This is the system we use for every ZOO client deployment.** Let's build it.

---

## Table of Contents

1. [Why AI Agent Observability Is Different](#why-different)
2. [The Three Pillars of Agent Observability](#three-pillars)
3. [Architecture Overview](#architecture)
4. [Step 1: Structured Tracing with OpenTelemetry](#tracing)
5. [Step 2: Cost Tracking Pipeline](#cost-tracking)
6. [Step 3: Quality Monitoring](#quality-monitoring)
7. [Step 4: Building the Dashboard](#dashboard)
8. [Step 5: Alerting Rules That Actually Work](#alerting)
6. [Complete Implementation](#complete-implementation)
7. [What We Learned (Including the $2,400 Mistake)](#lessons)

---

## Why AI Agent Observability Is Different <a name="why-different"></a>

Traditional software observability tracks requests, errors, and latency. That's necessary but insufficient for AI agents. Here's what makes agents different:

### Non-Determinism
The same input can produce different outputs. You can't just check `response == expected`. You need statistical quality monitoring over time.

### Cascading Costs
One agent invocation might trigger 3-15 LLM calls (planning, tool selection, execution, validation, summarization). A single runaway agent can generate 10,000+ API calls in minutes.

### Silent Failures
Agents rarely crash. They degrade. An agent that should return structured JSON starts returning markdown. An agent that should search the database starts hallucinating answers. Users get bad results but no error appears in your logs.

### Multi-Step Reasoning
A single "request" involves a chain of LLM calls, tool executions, and decisions. If step 7 of 12 fails, you need to know WHY — not just that the request failed.

### Tool Dependency Chains
Agents call external APIs, databases, search engines. A failure in any dependency can cause the agent to retry, loop, or hallucinate — and you won't see it without proper tracing.

---

## The Three Pillars of Agent Observability <a name="three-pillars"></a>

```
┌─────────────────────────────────────────────────────────┐
│                  AGENT OBSERVABILITY                     │
├──────────────────┬──────────────────┬───────────────────┤
│    TRACES        │    METRICS        │    QUALITY        │
│                  │                   │                   │
│ What happened    │ How much it cost  │ How good the      │
│ Step by step     │ How long it took  │ output was        │
│ Where it failed  │ How many tokens   │ Did it drift?     │
│                  │                   │                   │
│ OpenTelemetry    │ Prometheus/       │ Custom eval       │
│ Jaeger/Tempo     │ Grafana           │ pipeline          │
└──────────────────┴──────────────────┴───────────────────┘
```

Most teams only implement metrics (if that). You need all three.

---

## Architecture Overview <a name="architecture"></a>

```
User Request
     │
     ▼
┌─────────────┐    Traces     ┌──────────────┐
│  AI Agent   │──────────────▶│  Tempo/Jaeger │
│  (Python)   │               │  (Trace Store)│
│             │    Metrics     └──────────────┘
│  ┌───────┐ │──────────────▶┌──────────────┐
│  │ LLM   │ │               │  Prometheus   │
│  │ Calls │ │               │  + Grafana    │
│  └───────┘ │               └──────────────┘
│             │    Quality
│  ┌───────┐ │──────────────▶┌──────────────┐
│  │ Tools │ │               │  Eval Store   │
│  │       │ │               │  (Postgres)   │
│  └───────┘ │               └──────────────┘
│             │
│  ┌───────┐ │    Costs      ┌──────────────┐
│  │Costs  │ │──────────────▶│  Cost DB      │
│  │Tracker│ │               │  (Postgres)   │
│  └───────┘ │               └──────────────┘
└─────────────┘
```

---

## Step 1: Structured Tracing with OpenTelemetry <a name="tracing"></a>

OpenTelemetry (OTel) is the industry standard for distributed tracing. It works with any LLM provider and any observability backend.

### Installation

```bash
pip install opentelemetry-api opentelemetry-sdk
pip install opentelemetry-exporter-otlp-proto-http
pip install opentelemetry-instrumentation-openai
pip install opentelemetry-instrumentation-httpx
```

### Core Tracing Setup

```python
# observability/tracing.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.semconv.resource import ResourceAttributes
import os

def setup_tracing(service_name: str = "ai-agent"):
    """Initialize OpenTelemetry tracing for AI agents."""
    
    resource = Resource.create({
        ResourceAttributes.SERVICE_NAME: service_name,
        ResourceAttributes.SERVICE_VERSION: "1.0.0",
        "deployment.environment": os.getenv("ENV", "production"),
    })
    
    provider = TracerProvider(resource=resource)
    
    # Export to Tempo, Jaeger, or any OTLP-compatible backend
    otlp_exporter = OTLPSpanExporter(
        endpoint=os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "http://localhost:4318/v1/traces"),
    )
    provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
    
    trace.set_tracer_provider(provider)
    return trace.get_tracer(service_name)

# Global tracer
tracer = setup_tracing()
```

### Instrumenting LLM Calls

```python
# observability/llm_tracing.py
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode
from dataclasses import dataclass, field
from typing import Any, Optional
import time
import json

tracer = trace.get_tracer("ai-agent")

@dataclass
class LLMCallMetrics:
    """Metrics captured from a single LLM call."""
    model: str
    prompt_tokens: int = 0
    completion_tokens: int = 0
    total_tokens: int = 0
    latency_ms: float = 0.0
    cost_usd: float = 0.0
    finish_reason: str = ""
    error: Optional[str] = None

# Cost per 1K tokens (update with current pricing)
MODEL_COSTS = {
    "gpt-4o": {"input": 0.0025, "output": 0.01},
    "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
    "claude-3-5-sonnet": {"input": 0.003, "output": 0.015},
    "claude-3-haiku": {"input": 0.00025, "output": 0.00125},
    "gemini-1.5-pro": {"input": 0.00125, "output": 0.005},
    "gemini-1.5-flash": {"input": 0.000075, "output": 0.0003},
}

def calculate_cost(model: str, prompt_tokens: int, completion_tokens: int) -> float:
    """Calculate the cost of an LLM call."""
    costs = MODEL_COSTS.get(model, MODEL_COSTS["gpt-4o"])
    return (prompt_tokens / 1000 * costs["input"]) + (completion_tokens / 1000 * costs["output"])

def instrument_llm_call(func):
    """Decorator to instrument any LLM call with tracing."""
    async def wrapper(*args, **kwargs):
        model = kwargs.get("model", "unknown")
        messages = kwargs.get("messages", [])
        
        # Create a span for this LLM call
        with tracer.start_as_current_span(
            "llm.call",
            attributes={
                "llm.model": model,
                "llm.provider": kwargs.get("provider", "openai"),
                "llm.message_count": len(messages),
                "llm.max_tokens": kwargs.get("max_tokens", 0),
                "llm.temperature": kwargs.get("temperature", 0),
            }
        ) as span:
            start_time = time.time()
            
            try:
                result = await func(*args, **kwargs)
                
                latency_ms = (time.time() - start_time) * 1000
                
                # Extract token usage from response
                usage = getattr(result, "usage", None)
                prompt_tokens = getattr(usage, "prompt_tokens", 0) if usage else 0
                completion_tokens = getattr(usage, "completion_tokens", 0) if usage else 0
                total_tokens = getattr(usage, "total_tokens", 0) if usage else 0
                
                cost = calculate_cost(model, prompt_tokens, completion_tokens)
                
                # Add result attributes to span
                span.set_attributes({
                    "llm.usage.prompt_tokens": prompt_tokens,
                    "llm.usage.completion_tokens": completion_tokens,
                    "llm.usage.total_tokens": total_tokens,
                    "llm.cost_usd": round(cost, 6),
                    "llm.latency_ms": round(latency_ms, 2),
                    "llm.finish_reason": getattr(result, "finish_reason", "unknown"),
                })
                
                # Store metrics for aggregation
                await store_llm_metrics(LLMCallMetrics(
                    model=model,
                    prompt_tokens=prompt_tokens,
                    completion_tokens=completion_tokens,
                    total_tokens=total_tokens,
                    latency_ms=latency_ms,
                    cost_usd=cost,
                    finish_reason=getattr(result, "finish_reason", ""),
                ))
                
                return result
                
            except Exception as e:
                span.set_status(StatusCode.ERROR, str(e))
                span.record_exception(e)
                raise
    
    return wrapper
```

### Instrumenting Tool Calls

```python
# observability/tool_tracing.py
from opentelemetry import trace
from typing import Callable
import time
import traceback

tracer = trace.get_tracer("ai-agent")

def instrument_tool(tool_name: str):
    """Decorator to instrument agent tool calls."""
    def decorator(func: Callable):
        async def wrapper(*args, **kwargs):
            with tracer.start_as_current_span(
                "agent.tool_call",
                attributes={
                    "tool.name": tool_name,
                    "tool.input": str(kwargs)[:500],  # Truncate for storage
                }
            ) as span:
                start_time = time.time()
                
                try:
                    result = await func(*args, **kwargs)
                    latency_ms = (time.time() - start_time) * 1000
                    
                    span.set_attributes({
                        "tool.latency_ms": round(latency_ms, 2),
                        "tool.success": True,
                        "tool.output_size": len(str(result)),
                    })
                    
                    return result
                    
                except Exception as e:
                    latency_ms = (time.time() - start_time) * 1000
                    span.set_attributes({
                        "tool.latency_ms": round(latency_ms, 2),
                        "tool.success": False,
                        "tool.error": str(e),
                    })
                    span.record_exception(e)
                    raise
        
        return wrapper
    return decorator


# Usage example:
@instrument_tool("web_search")
async def web_search(query: str) -> str:
    """Search the web for information."""
    # ... actual search implementation
    pass

@instrument_tool("database_query")
async def database_query(sql: str) -> list:
    """Query the database."""
    # ... actual DB implementation
    pass

@instrument_tool("code_executor")
async def execute_code(code: str) -> str:
    """Execute Python code in sandbox."""
    # ... actual code execution
    pass
```

### Full Agent Run Tracing

```python
# observability/agent_tracing.py
from opentelemetry import trace
from dataclasses import dataclass, field
from typing import List
import time
import uuid

tracer = trace.get_tracer("ai-agent")

@dataclass
class AgentRunContext:
    """Context for a single agent run (end-to-end request)."""
    run_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    user_id: str = ""
    agent_name: str = ""
    start_time: float = field(default_factory=time.time)
    total_cost: float = 0.0
    total_tokens: int = 0
    llm_calls: int = 0
    tool_calls: int = 0
    errors: List[str] = field(default_factory=list)

async def trace_agent_run(user_id: str, agent_name: str, input_text: str):
    """Context manager for tracing an entire agent run."""
    ctx = AgentRunContext(user_id=user_id, agent_name=agent_name)
    
    with tracer.start_as_current_span(
        "agent.run",
        attributes={
            "agent.run_id": ctx.run_id,
            "agent.name": agent_name,
            "agent.user_id": user_id,
            "agent.input_length": len(input_text),
        }
    ) as span:
        try:
            yield ctx
            
            # After agent completes, set final attributes
            total_time = (time.time() - ctx.start_time) * 1000
            span.set_attributes({
                "agent.total_time_ms": round(total_time, 2),
                "agent.total_cost_usd": round(ctx.total_cost, 4),
                "agent.total_tokens": ctx.total_tokens,
                "agent.llm_calls": ctx.llm_calls,
                "agent.tool_calls": ctx.tool_calls,
                "agent.error_count": len(ctx.errors),
                "agent.success": len(ctx.errors) == 0,
            })
            
        except Exception as e:
            span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
            span.record_exception(e)
            raise
```

---

## Step 2: Cost Tracking Pipeline <a name="cost-tracking"></a>

This is the part that saves you thousands. You need to know exactly where money goes.

### Database Schema

```sql
-- migrations/001_cost_tracking.sql
CREATE TABLE llm_cost_logs (
    id SERIAL PRIMARY KEY,
    run_id VARCHAR(36) NOT NULL,
    user_id VARCHAR(255),
    agent_name VARCHAR(255) NOT NULL,
    model VARCHAR(100) NOT NULL,
    prompt_tokens INTEGER NOT NULL DEFAULT 0,
    completion_tokens INTEGER NOT NULL DEFAULT 0,
    total_tokens INTEGER NOT NULL DEFAULT 0,
    cost_usd DECIMAL(10, 6) NOT NULL DEFAULT 0,
    latency_ms FLOAT,
    tool_name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    
    -- Indexes for common queries
    INDEX idx_run_id (run_id),
    INDEX idx_user_id (user_id),
    INDEX idx_agent_name (agent_name),
    INDEX idx_created_at (created_at),
    INDEX idx_model (model)
);

-- Materialized view for daily cost summaries
CREATE MATERIALIZED VIEW daily_cost_summary AS
SELECT 
    DATE(created_at) as date,
    agent_name,
    model,
    COUNT(*) as call_count,
    SUM(prompt_tokens) as total_prompt_tokens,
    SUM(completion_tokens) as total_completion_tokens,
    SUM(cost_usd) as total_cost_usd,
    AVG(latency_ms) as avg_latency_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) as p95_latency_ms
FROM llm_cost_logs
GROUP BY DATE(created_at), agent_name, model;

-- Refresh every hour
-- (set up as a cron job or use pg_cron)
```

### Cost Tracker Implementation

```python
# observability/cost_tracker.py
import asyncio
import asyncpg
import json
from datetime import datetime, date
from typing import Optional, Dict, List
from dataclasses import dataclass

@dataclass
class CostAlert:
    """Alert triggered when cost thresholds are exceeded."""
    alert_type: str  # "daily_budget", "per_user_spike", "runaway_agent"
    severity: str    # "warning", "critical"
    message: str
    current_value: float
    threshold: float

class CostTracker:
    """Track and analyze AI agent costs in real-time."""
    
    def __init__(self, db_url: str):
        self.db_url = db_url
        self.db: Optional[asyncpg.Connection] = None
        # In-memory counters for real-time checks
        self._daily_cost = 0.0
        self._user_costs: Dict[str, float] = {}
        self._last_reset = date.today()
    
    async def connect(self):
        self.db = await asyncpg.connect(self.db_url)
    
    async def log_call(
        self,
        run_id: str,
        user_id: str,
        agent_name: str,
        model: str,
        prompt_tokens: int,
        completion_tokens: int,
        cost_usd: float,
        latency_ms: float,
        tool_name: str = None,
    ):
        """Log an LLM call and check for cost anomalies."""
        
        # Reset daily counters if it's a new day
        if date.today() != self._last_reset:
            self._daily_cost = 0.0
            self._user_costs = {}
            self._last_reset = date.today()
        
        # Update in-memory counters
        self._daily_cost += cost_usd
        self._user_costs[user_id] = self._user_costs.get(user_id, 0) + cost_usd
        
        # Persist to database
        await self.db.execute("""
            INSERT INTO llm_cost_logs 
            (run_id, user_id, agent_name, model, prompt_tokens, completion_tokens, 
             total_tokens, cost_usd, latency_ms, tool_name)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
        """, run_id, user_id, agent_name, model, prompt_tokens, completion_tokens,
             prompt_tokens + completion_tokens, cost_usd, latency_ms, tool_name)
        
        # Check for anomalies
        alerts = await self._check_anomalies(user_id, agent_name, cost_usd)
        return alerts
    
    async def _check_anomalies(self, user_id: str, agent_name: str, cost: float) -> List[CostAlert]:
        """Check for cost anomalies that need immediate attention."""
        alerts = []
        
        # Check daily budget
        daily_budget = float(os.getenv("DAILY_BUDGET_USD", "100"))
        if self._daily_cost > daily_budget * 0.8:
            alerts.append(CostAlert(
                alert_type="daily_budget",
                severity="warning" if self._daily_cost < daily_budget else "critical",
                message=f"Daily cost ${self._daily_cost:.2f} is at {self._daily_cost/daily_budget*100:.0f}% of budget ${daily_budget}",
                current_value=self._daily_cost,
                threshold=daily_budget,
            ))
        
        # Check per-user spike (any user spending >$5 in 5 minutes)
        user_spike_threshold = float(os.getenv("USER_SPIKE_THRESHOLD_USD", "5.0"))
        if self._user_costs.get(user_id, 0) > user_spike_threshold:
            alerts.append(CostAlert(
                alert_type="per_user_spike",
                severity="critical",
                message=f"User {user_id} has spent ${self._user_costs[user_id]:.2f} recently",
                current_value=self._user_costs[user_id],
                threshold=user_spike_threshold,
            ))
        
        return alerts
    
    async def get_daily_summary(self, target_date: date = None) -> dict:
        """Get cost summary for a specific day."""
        target_date = target_date or date.today()
        
        row = await self.db.fetchrow("""
            SELECT 
                COUNT(*) as total_calls,
                SUM(cost_usd) as total_cost,
                SUM(total_tokens) as total_tokens,
                AVG(latency_ms) as avg_latency,
                COUNT(DISTINCT user_id) as unique_users,
                COUNT(DISTINCT run_id) as unique_runs
            FROM llm_cost_logs
            WHERE DATE(created_at) = $1
        """, target_date)
        
        return {
            "date": str(target_date),
            "total_calls": row["total_calls"],
            "total_cost_usd": float(row["total_cost"] or 0),
            "total_tokens": row["total_tokens"] or 0,
            "avg_latency_ms": float(row["avg_latency"] or 0),
            "unique_users": row["unique_users"],
            "unique_runs": row["unique_runs"],
        }
    
    async def get_cost_by_agent(self, days: int = 7) -> List[dict]:
        """Get cost breakdown by agent over the last N days."""
        rows = await self.db.fetch("""
            SELECT 
                agent_name,
                DATE(created_at) as date,
                SUM(cost_usd) as daily_cost,
                COUNT(*) as calls
            FROM llm_cost_logs
            WHERE created_at >= NOW() - INTERVAL '%s days'
            GROUP BY agent_name, DATE(created_at)
            ORDER BY date DESC, daily_cost DESC
        """, days)
        
        return [dict(row) for row in rows]
```

---

## Step 3: Quality Monitoring <a name="quality-monitoring"></a>

This is what separates amateur agent deployments from professional ones.

### The Quality Pipeline

```python
# observability/quality.py
from dataclasses import dataclass
from typing import List, Optional, Callable
from enum import Enum
import json
import re

class QualityIssueType(Enum):
    HALLUCINATION = "hallucination"
    STRUCTURE_VIOLATION = "structure_violation"
    INCOMPLETE_RESPONSE = "incomplete_response"
    REPETITION = "repetition"
    OFF_TOPIC = "off_topic"
    SAFETY_VIOLATION = "safety_violation"

@dataclass
class QualityIssue:
    issue_type: QualityIssueType
    severity: str  # "low", "medium", "high", "critical"
    description: str
    evidence: str

@dataclass
class QualityReport:
    run_id: str
    overall_score: float  # 0.0 to 1.0
    issues: List[QualityIssue]
    response_length: int
    has_citations: bool
    structure_valid: bool

class QualityMonitor:
    """Monitor the quality of agent outputs."""
    
    def __init__(self):
        self.checks: List[Callable] = [
            self._check_structure,
            self._check_repetition,
            self._check_completeness,
            self._check_hallucination_markers,
        ]
    
    async def evaluate(
        self,
        run_id: str,
        user_input: str,
        agent_output: str,
        expected_format: str = "json",
        context: str = "",
    ) -> QualityReport:
        """Run all quality checks on an agent response."""
        
        issues = []
        
        for check in self.checks:
            found = await check(user_input, agent_output, expected_format, context)
            issues.extend(found)
        
        # Calculate overall score
        severity_weights = {"low": 0.05, "medium": 0.15, "high": 0.3, "critical": 0.5}
        total_penalty = sum(severity_weights.get(i.severity, 0.1) for i in issues)
        overall_score = max(0.0, 1.0 - total_penalty)
        
        return QualityReport(
            run_id=run_id,
            overall_score=round(overall_score, 2),
            issues=issues,
            response_length=len(agent_output),
            has_citations=self._has_citations(agent_output),
            structure_valid=expected_format != "json" or self._is_valid_json(agent_output),
        )
    
    async def _check_structure(
        self, user_input: str, output: str, expected_format: str, context: str
    ) -> List[QualityIssue]:
        """Check if the output matches the expected structure."""
        issues = []
        
        if expected_format == "json":
            try:
                json.loads(output)
            except json.JSONDecodeError:
                # Try to extract JSON from markdown code blocks
                json_match = re.search(r'```json\s*([\s\S]*?)\s*```', output)
                if not json_match:
                    issues.append(QualityIssue(
                        issue_type=QualityIssueType.STRUCTURE_VIOLATION,
                        severity="high",
                        description="Output is not valid JSON",
                        evidence=output[:200],
                    ))
        
        return issues
    
    async def _check_repetition(
        self, user_input: str, output: str, expected_format: str, context: str
    ) -> List[QualityIssue]:
        """Check for repetitive content (common agent failure mode)."""
        issues = []
        
        sentences = output.split(". ")
        if len(sentences) > 3:
            # Check for repeated sentences
            seen = set()
            repeats = []
            for s in sentences:
                normalized = s.strip().lower()
                if normalized in seen and len(normalized) > 20:
                    repeats.append(s[:100])
                seen.add(normalized)
            
            if len(repeats) > 2:
                issues.append(QualityIssue(
                    issue_type=QualityIssueType.REPETITION,
                    severity="high",
                    description=f"Found {len(repeats)} repeated sentences",
                    evidence="; ".join(repeats[:3]),
                ))
        
        return issues
    
    async def _check_completeness(
        self, user_input: str, output: str, expected_format: str, context: str
    ) -> List[QualityIssue]:
        """Check if the response appears complete."""
        issues = []
        
        # Check for truncated responses
        truncation_markers = [
            "...",
            "[truncated]",
            "I cannot continue",
            "As an AI",
            "I apologize, but",
        ]
        
        for marker in truncation_markers:
            if output.rstrip().endswith(marker):
                issues.append(QualityIssue(
                    issue_type=QualityIssueType.INCOMPLETE_RESPONSE,
                    severity="medium",
                    description=f"Response appears to end with truncation marker: '{marker}'",
                    evidence=output[-200:],
                ))
        
        return issues
    
    async def _check_hallucination_markers(
        self, user_input: str, output: str, expected_format: str, context: str
    ) -> List[QualityIssue]:
        """Check for common hallucination patterns."""
        issues = []
        
        hallucination_patterns = [
            (r"I (?:don't|do not) have (?:access|information)", 
             "Model claims lack of information"),
            (r"As of my last (?:update|training)",
             "Model references training cutoff"),
            (r"I (?:made|make) up",
             "Model admits to fabricating information"),
        ]
        
        for pattern, description in hallucination_patterns:
            match = re.search(pattern, output, re.IGNORECASE)
            if match:
                issues.append(QualityIssue(
                    issue_type=QualityIssueType.HALLUCINATION,
                    severity="medium",
                    description=description,
                    evidence=match.group(0),
                ))
        
        return issues
    
    def _has_citations(self, output: str) -> bool:
        """Check if the output contains citations or references."""
        citation_patterns = [
            r'\[\d+\]',           # [1], [2], etc.
            r'\[\w+,\s*\d{4}\]',  # [Smith, 2024]
            r'source:',            # Source: ...
            r'reference:',         # Reference: ...
            r'according to',       # According to ...
        ]
        return any(re.search(p, output, re.IGNORECASE) for p in citation_patterns)
    
    def _is_valid_json(self, text: str) -> bool:
        try:
            json.loads(text)
            return True
        except json.JSONDecodeError:
            return False
```

---

## Step 4: Building the Dashboard <a name="dashboard"></a>

Here's a Grafana dashboard configuration that shows everything you need:

```json
{
  "dashboard": {
    "title": "AI Agent Observability",
    "panels": [
      {
        "title": "Cost Today",
        "type": "stat",
        "targets": [{
          "expr": "sum(llm_cost_usd_total)"
        }],
        "thresholds": [
          {"value": 0, "color": "green"},
          {"value": 50, "color": "yellow"},
          {"value": 100, "color": "red"}
        ]
      },
      {
        "title": "Cost by Agent (7d)",
        "type": "barchart",
        "targets": [{
          "expr": "sum by (agent_name) (llm_cost_usd_total)"
        }]
      },
      {
        "title": "P95 Latency by Model",
        "type": "timeseries",
        "targets": [{
          "expr": "histogram_quantile(0.95, sum by (le, model) (llm_latency_ms_bucket))"
        }]
      },
      {
        "title": "Token Usage by Model",
        "type": "piechart",
        "targets": [{
          "expr": "sum by (model) (llm_tokens_total)"
        }]
      },
      {
        "title": "Error Rate",
        "type": "gauge",
        "targets": [{
          "expr": "rate(llm_calls_total{status='error'}[5m]) / rate(llm_calls_total[5m])"
        }]
      },
      {
        "title": "Quality Score Distribution",
        "type": "histogram",
        "targets": [{
          "expr": "agent_quality_score_bucket"
        }]
      },
      {
        "title": "Top Cost Users",
        "type": "table",
        "targets": [{
          "expr": "topk(10, sum by (user_id) (llm_cost_usd_total))"
        }]
      },
      {
        "title": "Tool Call Success Rate",
        "type": "barchart",
        "targets": [{
          "expr": "sum by (tool_name) (tool_calls_total) / sum by (tool_name) (tool_calls_total)"
        }]
      }
    ]
  }
}
```

---

## Step 5: Alerting Rules That Actually Work <a name="alerting"></a>

Bad alerting is worse than no alerting. Here are the rules we use:

```yaml
# alerting/agent-alerts.yml
groups:
  - name: ai_agent_alerts
    rules:
      # CRITICAL: Runaway agent detection
      - alert: RunawayAgent
        expr: |
          (
            sum by (run_id) (rate(llm_cost_usd_total[5m])) > 0.5
          )
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Runaway agent detected - {{ $labels.run_id }}"
          description: "Agent is spending >$0.50/min. Possible infinite loop."
          runbook_url: "https://wiki.internal/runaway-agent-runbook"
      
      # CRITICAL: Daily budget exceeded
      - alert: DailyBudgetExceeded
        expr: |
          sum(increase(llm_cost_usd_total[1h])) > 80
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Daily AI budget at ${{ $value }}"
          description: "Approaching daily budget limit. Consider rate limiting."
      
      # WARNING: High error rate
      - alert: HighAgentErrorRate
        expr: |
          rate(llm_calls_total{status="error"}[10m]) / rate(llm_calls_total[10m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High LLM error rate: {{ $value | humanizePercentage }}"
          description: "More than 5% of LLM calls are failing."
      
      # WARNING: Latency spike
      - alert: AgentLatencySpike
        expr: |
          histogram_quantile(0.95, rate(llm_latency_ms_bucket[5m])) > 10000
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "P95 latency above 10 seconds"
          description: "LLM responses are unusually slow. Check provider status."
      
      # WARNING: Quality degradation
      - alert: QualityDegradation
        expr: |
          avg_over_time(agent_quality_score[30m]) < 0.7
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Agent quality score dropped below 0.7"
          description: "Output quality has degraded. Check for model drift or prompt issues."
      
      # INFO: Unusual token usage pattern
      - alert: UnusualTokenUsage
        expr: |
          stddev_over_time(llm_tokens_total[1h]) > 3 * avg_over_time(llm_tokens_total[1h])
        for: 15m
        labels:
          severity: info
        annotations:
          summary: "Unusual token usage variance detected"
          description: "Token usage is highly variable. Some requests may be unusually large."
```

---

## Complete Implementation <a name="complete-implementation"></a>

Here's how to wire everything together in your agent:

```python
# agent.py - Complete agent with observability
from observability.tracing import tracer
from observability.llm_tracing import instrument_llm_call
from observability.tool_tracing import instrument_tool
from observability.agent_tracing import trace_agent_run
from observability.cost_tracker import CostTracker
from observability.quality import QualityMonitor
from openai import AsyncOpenAI
import os

class ObservableAgent:
    """AI agent with full observability built in."""
    
    def __init__(self, name: str, db_url: str):
        self.name = name
        self.client = AsyncOpenAI()
        self.cost_tracker = CostTracker(db_url)
        self.quality_monitor = QualityMonitor()
    
    async def initialize(self):
        await self.cost_tracker.connect()
    
    @instrument_llm_call
    async def _call_llm(self, messages: list, model: str = "gpt-4o-mini", **kwargs):
        """Make an LLM call (automatically traced)."""
        return await self.client.chat.completions.create(
            model=model,
            messages=messages,
            **kwargs,
        )
    
    @instrument_tool("search")
    async def search(self, query: str) -> str:
        """Search tool (automatically traced)."""
        # ... search implementation
        return "search results"
    
    async def run(self, user_id: str, input_text: str) -> str:
        """Run the agent with full observability."""
        
        async with trace_agent_run(user_id, self.name, input_text) as ctx:
            # Step 1: Plan
            plan = await self._call_llm(
                model="gpt-4o-mini",
                messages=[
                    {"role": "system", "content": "You are a planning assistant. Output JSON with 'steps' array."},
                    {"role": "user", "content": input_text},
                ],
            )
            
            # Track cost
            usage = plan.usage
            cost = calculate_cost("gpt-4o-mini", usage.prompt_tokens, usage.completion_tokens)
            ctx.total_cost += cost
            ctx.total_tokens += usage.total_tokens
            ctx.llm_calls += 1
            
            alerts = await self.cost_tracker.log_call(
                run_id=ctx.run_id,
                user_id=user_id,
                agent_name=self.name,
                model="gpt-4o-mini",
                prompt_tokens=usage.prompt_tokens,
                completion_tokens=usage.completion_tokens,
                cost_usd=cost,
                latency_ms=0,  # Traced separately
            )
            
            # Handle cost alerts
            for alert in alerts:
                if alert.severity == "critical":
                    await self._handle_critical_alert(alert)
            
            # Step 2: Execute (simplified)
            response = plan.choices[0].message.content
            
            # Step 3: Quality check
            quality = await self.quality_monitor.evaluate(
                run_id=ctx.run_id,
                user_input=input_text,
                agent_output=response,
                expected_format="json",
            )
            
            if quality.overall_score < 0.5:
                ctx.errors.append(f"Quality score too low: {quality.overall_score}")
            
            return response
    
    async def _handle_critical_alert(self, alert):
        """Handle critical cost alerts."""
        # Send to PagerDuty, Slack, etc.
        print(f"🚨 CRITICAL: {alert.message}")
        # In production: await send_slack_alert(alert)
```

---

## What We Learned (Including the $2,400 Mistake) <a name="lessons"></a>

### The Incident

On March 15, 2026, one of our client's agents got stuck in a loop. A user sent a malformed request that caused the agent to repeatedly call the search tool, get confused by the results, and try again. And again. And again.

In 4 hours, it made **47,000 LLM calls** and **120,000 tool calls**. The bill: **$2,447.83**.

We found out when the client's CFO called asking why their OpenAI bill was 10x the normal amount.

### What We Built After

The observability stack above. Here's what it would have caught:

1. **Runaway agent alert** — would have fired at call #500 (2 minutes in), not hour 4
2. **Cost spike alert** — would have notified us at $50, not $2,400
3. **Tool call loop detection** — would have identified the repeated search pattern
4. **Quality degradation** — would have caught that the agent's outputs were getting worse, not better

### Key Lessons

1. **Set hard limits, not just alerts.** Every agent should have a max-calls-per-run limit (e.g., 20 LLM calls) and a max-cost-per-run limit (e.g., $1.00). When hit, the agent returns a graceful error instead of looping.

2. **Monitor quality, not just uptime.** An agent that returns bad results is worse than one that returns no results. Quality monitoring catches silent failures.

3. **Cost per user matters more than total cost.** One abusive user can bankrupt you. Track costs per user and implement per-user rate limits.

4. **Alert on rate of change, not absolute values.** A jump from 100 to 10,000 calls/hour is more important than a steady 10,000 calls/hour.

5. **Test your observability.** Run a known-bad input through your agent and verify that alerts fire. If your observability isn't tested, you won't know it's broken until it's too late.

---

## Next Steps

You now have the complete observability stack. Here's what to do:

1. **Today:** Add tracing to your LLM calls. Even just logging token counts and costs will save you money.
2. **This week:** Set up the cost tracking database and daily summary queries.
3. **This month:** Implement quality monitoring and the Grafana dashboard.
4. **Ongoing:** Review your alerting rules weekly. Tune thresholds based on false positive rate.

---

## Need Help Implementing This?

**This is exactly the kind of work we do at ZOO.** We've built observability systems for AI agents processing millions of requests per day. If you're shipping AI agents to production and need them to be reliable, observable, and cost-effective, we can help.

**→ [Get a free Production Readiness Audit](https://zoo.dev/contact)** — We'll review your agent architecture and identify the top 3 observability gaps. No strings attached.

**What you get:**
- 30-minute architecture review call
- Written report with prioritized recommendations
- Cost optimization estimate (most clients save 40-80%)
- Implementation roadmap

**[Schedule your free audit →](https://zoo.dev/contact)**

---

*Last updated: May 2026 | Author: ZOO Engineering | Tags: AI agents, observability, OpenTelemetry, production AI, cost optimization, monitoring*
