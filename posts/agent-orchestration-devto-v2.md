---
title: "Multi-Agent Orchestration in Python: The Pattern That Replaced Our Management Team"
published: false
description: "How we run a company with 10 AI agents using a hub-and-spoke orchestration pattern. Full Python code, real Week 1 data, and the 3 rules that make it work."
tags: python, aiagents, orchestration, startup, tutorial
---

# Multi-Agent Orchestration in Python: The Pattern That Replaced Our Management Team

Everyone's building AI agents. Few are making money with them.

The problem isn't the models. GPT-4 is capable. Claude is capable. The problem is **orchestration** — getting agents to work together as a system, not as isolated chatbots.

After running [ZOO](https://zootechnologies.com) with 10 AI CEO agents for a week, we learned something counterintuitive:

**The agents themselves are the easy part. The management layer is where projects die.**

## The Pattern: Hub-and-Spoke Agent Architecture

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

## Implementation: Agent Orchestrator in Python

Here's a production-ready pattern you can adapt today:

```python
import asyncio
import json
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
    created_at: str = field(default_factory=lambda: datetime.now().isoformat())
    completed_at: Optional[str] = None

@dataclass
class Agent:
    name: str
    domain: str
    status: AgentStatus = AgentStatus.IDLE
    current_task: Optional[str] = None
    completed_tasks: list = field(default_factory=list)
    kanban_board: str = "default"

class Orchestrator:
    def __init__(self):
        self.agents: dict[str, Agent] = {}
        self.task_queue: list[Task] = []
        self.completed: list[Task] = []
        self.blockers: list[dict] = []

    def register_agent(self, name: str, domain: str, board: str = "default"):
        self.agents[name] = Agent(
            name=name, domain=domain, kanban_board=board
        )
        print(f"✅ Agent registered: {name} ({domain}) — Board: {board}")

    def create_task(
        self, task_id: str, description: str,
        assignee: str, priority: TaskPriority = TaskPriority.P1,
        dependencies: list = None
    ) -> Optional[Task]:
        if assignee not in self.agents:
            print(f"❌ Agent '{assignee}' not found")
            return None
        task = Task(
            id=task_id, description=description,
            assigned_to=assignee, priority=priority,
            dependencies=dependencies or []
        )
        self.task_queue.append(task)
        self.task_queue.sort(key=lambda t: t.priority.value)
        print(f"📋 Task created: [{priority.name}] {task_id} → {assignee}")
        return task

    def get_next_task(self, agent_name: str) -> Optional[Task]:
        agent = self.agents.get(agent_name)
        if not agent:
            return None
        for task in self.task_queue:
            if task.status != AgentStatus.IDLE:
                continue
            if agent.domain not in task.description.lower():
                continue
            deps_met = all(
                any(t.id == dep and t.status == AgentStatus.DONE
                    for t in self.completed)
                for dep in task.dependencies
            )
            if deps_met:
                return task
        return None

    def execute_task(self, task_id: str, output: str):
        for task in self.task_queue:
            if task.id == task_id:
                task.status = AgentStatus.DONE
                task.output = output
                task.completed_at = datetime.now().isoformat()
                agent = self.agents.get(task.assigned_to)
                if agent:
                    agent.status = AgentStatus.IDLE
                    agent.current_task = None
                    agent.completed_tasks.append(task_id)
                self.completed.append(task)
                self.task_queue.remove(task)
                print(f"✅ Task done: {task_id} by {task.assigned_to}")
                return True
        return False

    def report(self) -> dict:
        return {
            "agents": {
                name: {
                    "status": agent.status.value,
                    "domain": agent.domain,
                    "tasks_completed": len(agent.completed_tasks)
                }
                for name, agent in self.agents.items()
            },
            "queue_size": len(self.task_queue),
            "completed": len(self.completed),
            "blockers": self.blockers,
            "timestamp": datetime.now().isoformat()
        }
```

## Usage Example

```python
orch = Orchestrator()

# Register agents
orch.register_agent("HAWK", "content marketing community pr", "hawk")
orch.register_agent("ORION", "outreach sales business development", "orion")
orch.register_agent("VIPER", "deployment infrastructure devops", "viper")
orch.register_agent("LYNX", "product design sales-pages", "lynx")

# Create tasks
orch.create_task("PH-001", "content: write producthunt listing", "HAWK", TaskPriority.P0)
orch.create_task("OUT-001", "outreach: send follow-up emails", "ORION", TaskPriority.P0)
orch.create_task("DEP-001", "deployment: create payment links", "VIPER", TaskPriority.P0)

# Complete a task
orch.execute_task("PH-001", "Blog post published successfully")

# System report
print(json.dumps(orch.report(), indent=2))
```

## The Three Rules That Make It Work

### Rule 1: Every Agent Has a Kanban Board

No shared task list. Each agent owns its board. The orchestrator only sees cross-agent dependencies.

### Rule 2: P0 Tasks Get Human Escalation

When a P0 task is blocked for more than 1 cycle, it escalates to a human. Don't let agents spin on unsolvable problems.

### Rule 3: Output Validation Before Shipping

Never let agent output go directly to customers without a quality gate.

```
Agent writes → Quality check → Human review (P0 only) → Ship
```

## Real Results: Week 1 Data

| Agent | Tasks Completed | Output |
|-------|----------------|--------|
| HAWK | 10+ content pieces | Blog posts, PH launch kit, Reddit drafts |
| ORION | 94 outreach emails | 25 qualified leads, $77K pipeline |
| VIPER | 5 product deployments | 5 live products, checkout system |
| LYNX | 5 sales pages | HTML/CSS landing pages |
| PULSE | 4 dashboards | Revenue tracking, system health |
| NEMO | 3 market scans | 10 product opportunities |

**Cost: ~$0 in labor. Output: equivalent to a 5-person startup team.**

## What We Got Wrong

Transparency: Week 1 revenue was $0.

The agents produced content, pipeline, and products. But we couldn't close because:

1. **Stripe wasn't fully configured** — Payment Links for 3 of 5 products were missing
2. **Distribution channels were blocked** — Reddit, Dev.to, HN need human-verified accounts
3. **No warm audience** — First-day launch with zero followers

These are fixable problems. The agent system worked. The monetization layer is catching up.

## Try It Yourself

This pattern works for any team running 3+ AI agents:

1. **Start with the orchestrator** — Task routing + state tracking
2. **Give each agent a domain** — No overlapping responsibilities
3. **Implement P0 escalation** — Humans handle what agents can't
4. **Validate outputs** — Quality gate before anything ships

---

**We're launching on ProductHunt May 12** with our Landing Page Templates. [Check out ZOO →](https://zootechnologies.com/products)

*If you found this useful, try our free [AI Agent ROI Calculator](https://crazycompanyinc.github.io/zootechnologies-web/tools/ai-roi-calculator.html) — no signup required.*
