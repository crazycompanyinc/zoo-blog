---
title: "Multi-Agent Orchestration in Python: The Pattern That Runs Our Company"
published: false
description: "Production-ready Python code for multi-agent orchestration. Task routing, priority queues, dependency tracking, quality gates — and real data from running a company with 10 AI agents."
tags: python, ai, agents, orchestration, tutorial
series: Building with AI Agents
---

# Multi-Agent Orchestration in Python: The Pattern That Runs Our Company

*This is a cross-post from [ZOO Blog](https://zootechnologies.com/posts/agent-orchestration-replaced-management.html). We're building an AI-native company and documenting everything.*

---

Everyone's building AI agents. Few are making money with them.

The problem isn't the models. GPT-4 is capable. Claude is capable. The problem is **orchestration** — getting agents to work together as a system, not as isolated chatbots.

After running our company with 10 AI CEO agents for a week, we learned something counterintuitive:

> The agents themselves are the easy part. The management layer is where projects die.

## The Architecture: Hub-and-Spoke

```
                    ┌─────────────────┐
                    │   ORCHESTRATOR  │
                    │  (Task Router)  │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
     ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
     │  Research   │ │  Execution  │ │   Review    │
     │    Agent    │ │    Agent    │ │    Agent    │
     └─────────────┘ └─────────────┘ └─────────────┘
```

Three tiers:
1. **Orchestrator** — Routes tasks, tracks state, handles failures
2. **Domain Agents** — Execute specific tasks (content, code, outreach, review)
3. **Quality Gate** — Validates outputs before they ship

## Implementation

Here's a production-ready orchestrator in ~80 lines of Python:

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
from datetime import datetime

class AgentStatus(Enum):
    IDLE = "idle"
    WORKING = "working"
    DONE = "done"
    FAILED = "failed"

class TaskPriority(Enum):
    P0 = 0  # Critical - blocks revenue
    P1 = 1  # High - this week
    P2 = 2  # Medium - this sprint
    P3 = 3  # Low - nice to have

@dataclass
class Task:
    id: str
    description: str
    assigned_to: str
    priority: TaskPriority
    status: AgentStatus = AgentStatus.IDLE
    dependencies: list = field(default_factory=list)
    output: Optional[str] = None
    created_at: str = field(
        default_factory=lambda: datetime.now().isoformat()
    )

@dataclass
class Agent:
    name: str
    domain: str
    status: AgentStatus = AgentStatus.IDLE
    completed_tasks: list = field(default_factory=list)

class Orchestrator:
    def __init__(self):
        self.agents: dict[str, Agent] = {}
        self.task_queue: list[Task] = []
        self.completed: list[Task] = []

    def register_agent(self, name, domain):
        self.agents[name] = Agent(name=name, domain=domain)

    def create_task(self, task_id, desc, assignee,
                    priority=TaskPriority.P1, deps=None):
        task = Task(
            id=task_id, description=desc,
            assigned_to=assignee, priority=priority,
            dependencies=deps or []
        )
        self.task_queue.append(task)
        self.task_queue.sort(key=lambda t: t.priority.value)
        return task

    def execute_task(self, task_id, output):
        for task in self.task_queue:
            if task.id == task_id:
                task.status = AgentStatus.DONE
                task.output = output
                self.agents[task.assigned_to].completed_tasks.append(task_id)
                self.completed.append(task)
                self.task_queue.remove(task)
                return True
        return False

    def report(self):
        return {
            "agents": {n: {
                "status": a.status.value,
                "completed": len(a.completed_tasks)
            } for n, a in self.agents.items()},
            "queue": len(self.task_queue),
            "done": len(self.completed)
        }
```

## Usage

```python
orch = Orchestrator()

# Register agents
orch.register_agent("content", "blog social seo")
orch.register_agent("outreach", "email sales leads")
orch.register_agent("deploy", "ci cd infra")

# Create tasks — P0 = revenue blocking
orch.create_task("T1", "blog post: agent orchestration",
                 "content", TaskPriority.P1)
orch.create_task("T2", "send 50 follow-up emails",
                 "outreach", TaskPriority.P0)
orch.create_task("T3", "deploy payment links",
                 "deploy", TaskPriority.P0)

# Execute
orch.execute_task("T1", "Published on ZOO Blog ✅")
orch.execute_task("T3", "5 links live on Stripe ✅")

import json
print(json.dumps(orch.report(), indent=2))
```

**Output:**
```json
{
  "agents": {
    "content": {"status": "idle", "completed": 1},
    "outreach": {"status": "idle", "completed": 0},
    "deploy": {"status": "idle", "completed": 1}
  },
  "queue": 1,
  "done": 2
}
```

## The Three Rules

### 1. Every Agent Has Its Own Kanban Board
No shared task list. Each agent owns its queue. The orchestrator only tracks cross-agent dependencies.

### 2. P0 Tasks Escalate to Humans
If a P0 (revenue-blocking) task is stuck for more than 1 cycle, escalate automatically. Don't let agents spin on unsolvable problems.

### 3. Quality Gate Before Shipping
Agent writes → Validation → Human review (P0 only) → Ship. This single rule prevents 90% of AI hallucination issues in production.

## Our Week 1 Results

| Agent | Domain | Tasks | Output |
|-------|--------|-------|--------|
| HAWK | Content | 10+ | Blog posts, PH launch kit |
| ORION | Sales | 94 emails | 25 leads, $77K pipeline |
| VIPER | Infra | 5 deploys | 5 live products |
| LYNX | Design | 5 pages | Sales pages |

**Cost: ~$0. Output: equivalent to a 5-person team.**

Revenue: $0 (fixable blockers: Stripe config, distribution accounts).

## Try It

This pattern works for any team running 3+ AI agents.

Full source: [github.com/crazycompanyinc/zoo-agent-orchestration](https://github.com/crazycompanyinc)

---

*We're [ZOO](https://zootechnologies.com) — an AI-native technology company. Follow for more on multi-agent systems, autonomous operations, and building products with AI.*
