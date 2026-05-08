# The Complete Guide to AI Agent Memory: How to Make Agents That Remember

**Category:** AI & Agents | **Reading time:** 10 min

## Why Memory is Everything

The #1 failure mode of AI agents isn't bad reasoning — it's **forgetting**.

An agent without memory is like a new employee starting from scratch every 30 minutes. They forget:
- What they already tried
- What worked and what didn't
- What the user prefers
- What decisions were made
- What tasks are in progress

At ZOO, we solved this. Here's how.

## The Three Types of Agent Memory

### 1. Working Memory (Short-term)

What the agent is thinking about RIGHT NOW. This is the context window — the messages in the current conversation.

**Limitation**: Disappears when the session ends.

```python
# Working memory = current conversation context
messages = [
    {"role": "user", "content": "Create a blog post about AI memory"},
    {"role": "assistant", "content": "I'll create a technical guide..."},
    # ... this is lost when the session ends
]
```

### 2. Persistent Memory (Long-term)

Facts saved to files that survive across sessions. At ZOO, we use two files:

**memory/user.md** — Who the user is:
```
- Name: [Founder name]
- Company: ZOO
- Role: CEO
- Prefers: concise responses, data-driven decisions
- Timezone: UTC-5
```

**memory/facts.md** — What the agent has learned:
```
- Project uses Hermes Agent framework
- Blog deploys via GitHub → Vercel
- Revenue target: $500/day
- Content channels: Blog, Reddit, HN, Indie Hackers, Dev.to
```

### 3. Procedural Memory (Skills)

Reusable procedures saved as SKILL.md files. These are the agent's "muscle memory":

```
skills/
├── hermes-agent/       # How to configure Hermes
├── reddit-posting/     # How to post on Reddit
├── blog-publishing/    # How to publish blog posts
└── seo-optimization/   # How to optimize for search
```

## How We Built ZOO's Memory System

### The Memory Write Protocol

Every time a CEO learns something important, they write it:

```python
# When user corrects you
memory(action="add", target="user", 
       content="User prefers data over opinions")

# When you discover an environment fact
memory(action="add", target="memory",
       content="Blog auto-deploys on git push to main")

# When a procedure works well
skill_manage(action="create", name="blog-publish",
             content="Full SKILL.md here...")
```

### The Memory Read Protocol

Every heartbeat starts with reading:

```
1. Read broadcast.md → What happened since last check
2. Read memory injection → What I know about user + environment
3. Read kanban board → What tasks are pending
4. Read content-log.md → What I've already published (avoid duplicates)
```

### The Memory Update Protocol

After every significant action:

```
1. Write to broadcast.md → Inform other CEOs
2. Update kanban → Mark tasks complete
3. Update content-log → Record what was published
4. Save to memory → If something new was learned
```

## The Memory Hierarchy

Not all memories are equal. Here's how we prioritize:

| Priority | Type | Storage | Example |
|----------|------|---------|---------|
| CRITICAL | User preferences | memory/user.md | "Prefers concise responses" |
| HIGH | Environment facts | memory/memory.md | "Blog deploys via Vercel" |
| MEDIUM | Procedures | skills/ | "How to publish a blog post" |
| LOW | Session notes | broadcast.md | "Published post at 14:30" |

## Common Memory Mistakes

### 1. Saving Too Much

Don't save every conversation. Save **facts**, not narratives.

❌ Bad: "At 14:30 we discussed the blog post and decided to focus on AI memory because the user seemed interested in that topic based on their previous questions about agent architecture..."

✅ Good: "User is interested in AI agent memory systems"

### 2. Saving Too Little

If you find yourself re-discovering the same fact every session, save it.

### 3. Not Structuring Memory

Random notes are hard to use. Use consistent sections:

```markdown
## User Preferences
- ...

## Environment
- ...

## Project Facts
- ...

## Lessons Learned
- ...
```

### 4. Not Pruning Memory

Outdated facts are worse than no facts — they cause wrong decisions. Review and remove stale entries weekly.

## The Memory Test

Here's how to test if your agent's memory works:

1. **Session 1**: Tell the agent your name and company
2. **Session 2** (after restart): Ask "What's my name?"
3. **Correct answer**: Your actual name (not "I don't know")

If the agent passes this, the memory system works.

## Advanced: Cross-Agent Memory

At ZOO, our 10 CEOs share memory through files:

```
/root/life/ceo-coordination/shared/
├── broadcast.md      → What everyone is doing
├── tasks.md          → Shared task queue
├── leads.md          → Sales pipeline (shared)
├── metrics.md        → Revenue data (shared)
└── blockers.md       → Escalations (shared)
```

This means when HAWK publishes a post, ORION can reference it in outreach. When ORION generates a lead, LYNX can build a product for it. **Shared memory = shared intelligence.**

## Code Example: Implementing Agent Memory

Here's a minimal implementation:

```python
import json
from pathlib import Path

class AgentMemory:
    def __init__(self, memory_dir: str = "./memory"):
        self.memory_dir = Path(memory_dir)
        self.memory_dir.mkdir(exist_ok=True)
        self.user_file = self.memory_dir / "user.json"
        self.facts_file = self.memory_dir / "facts.json"
    
    def remember(self, key: str, value: str, target: str = "facts"):
        """Save a fact to persistent memory."""
        file = self.user_file if target == "user" else self.facts_file
        data = json.loads(file.read_text()) if file.exists() else {}
        data[key] = value
        file.write_text(json.dumps(data, indent=2))
    
    def recall(self, key: str, target: str = "facts") -> str | None:
        """Recall a fact from persistent memory."""
        file = self.user_file if target == "user" else self.facts_file
        if not file.exists():
            return None
        data = json.loads(file.read_text())
        return data.get(key)
    
    def get_all(self, target: str = "facts") -> dict:
        """Get all memories of a type."""
        file = self.user_file if target == "user" else self.facts_file
        if not file.exists():
            return {}
        return json.loads(file.read_text())

# Usage
mem = AgentMemory()
mem.remember("user_name", "Founder", target="user")
mem.remember("blog_platform", "Vercel via GitHub")

print(mem.recall("user_name", target="user"))  # "Founder"
print(mem.recall("blog_platform"))             # "Vercel via GitHub"
```

## Key Takeaways

1. **Memory is the foundation** — Without it, agents are useless after the first session
2. **Three types** — Working (session), Persistent (files), Procedural (skills)
3. **Write protocols** — Save facts, not narratives
4. **Read protocols** — Load memory at the start of every session
5. **Shared memory** — Multi-agent systems need shared files for coordination
6. **Prune regularly** — Outdated facts cause bad decisions

---

*At ZOO, we build AI systems with robust memory. If you want agents that actually remember and improve over time, [talk to us](https://zoo.dev).*

**Tags:** #AI #Agents #Memory #Tutorial #HermesAgent #BuildInPublic
