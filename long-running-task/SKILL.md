---
name: long-running-task
description: Guide for orchestrating long-running development tasks using Claude Code's checkpoint system. Use when working on tasks that (1) span multiple hours or sessions, (2) require complex multi-step implementations, (3) involve parallel sub-agents, (4) need state persistence and recovery capabilities, or (5) risk interruption and require resumability. Covers checkpoint management, state persistence strategies, parallel task coordination, and recovery workflows.
license: MIT
---

# Long-Running Task Orchestration

This skill provides patterns and best practices for orchestrating long-running development tasks using Claude Code's checkpoint system.

## Core Principles

1. **Minimize State Capture** - Capture state only at necessary moments (before user prompts, at milestones)
2. **Layered Recovery** - Support independent recovery of conversation, code, and composite states
3. **Session Boundary Transparency** - Checkpoints persist across sessions for true resumability

## When to Use This Skill

- Tasks expected to take more than 30 minutes
- Multi-file refactoring or migration projects
- Complex feature implementations with multiple phases
- Tasks using parallel sub-agents
- Work that may be interrupted and needs recovery

## Checkpoint System Overview

### Four Core Components

1. **State Snapshot Engine** - Captures workspace state at key moments
2. **Cross-Session Persistence Layer** - Enables recovery after session ends
3. **Recovery Selector** - Three granularity levels for restoration
4. **Resource Isolation Manager** - Manages parallel sub-agent states

### What Checkpoints Capture

- Currently open files and edit positions
- Terminal session state (environment variables, current directory)
- Claude's conversation context
- Sub-agent running states

## Workflow

### Step 1: Assess Task Complexity

Before starting, evaluate:

| Factor | Low Complexity | High Complexity |
|--------|---------------|-----------------|
| Duration | < 30 min | > 2 hours |
| Files affected | 1-3 | 10+ |
| Dependencies | None | Multiple modules |
| Parallel work | No | Yes |
| Risk of interruption | Low | High |

For high-complexity tasks, proceed with checkpoint-aware workflow.

### Step 2: Create Checkpoint Strategy

Based on task type, configure appropriate checkpoint frequency:

**Exploratory Tasks**: High frequency (every user interaction)
- Maximizes flexibility for backtracking
- Higher storage overhead acceptable

**Batch Processing Tasks**: Milestone-based (every 100 files, every phase)
- Balance recovery granularity with efficiency
- Set clear milestone markers

**Parallel Tasks**: Independent strategies per sub-agent
- Compute-intensive agents: More frequent checkpoints
- I/O-intensive agents: Lower frequency

### Step 3: Implement Task with Checkpoints

```
Task Start
    |
    v
[Create Initial Checkpoint] <-- User: /checkpoint or auto
    |
    v
Phase 1 Execution
    |
    v
[Milestone Checkpoint] <-- At phase completion
    |
    v
Phase 2 Execution
    |
    v
... (continue pattern)
    |
    v
Task Complete
```

### Step 4: Handle Interruptions

When resuming after interruption:

1. **Identify Last Checkpoint** - Check timestamp and task context
2. **Choose Recovery Mode**:
   - **Conversation Only**: Code is correct, conversation drifted
   - **Code Only**: Code issues, conversation valuable
   - **Composite**: Full rollback to checkpoint state
3. **Resume Execution** - Continue from checkpoint state

### Step 5: Manage Parallel Sub-Agents

For parallel task execution:

1. Each sub-agent maintains independent state space
2. Main agent coordinates checkpoint synchronization
3. Handle resource conflicts (ports, file locks) proactively

See `references/sub-agent-patterns.md` for detailed patterns.

## Recovery Modes

### Conversation-Only Recovery

Use when:
- Code changes are correct
- Conversation has drifted off-topic
- Want to restart discussion from decision point

Preserves: All code changes, file states
Resets: Conversation context to selected point

### Code-Only Recovery

Use when:
- Code changes introduced problems
- Conversation history is valuable
- Want to rollback files while keeping context

Preserves: Full conversation history
Resets: File states to checkpoint

### Composite Recovery

Use when:
- Complete restart needed
- Both code and conversation should revert
- "Time travel" to specific task moment

Resets: Everything to checkpoint state

## Important Limitations

### Bash Command Changes Not Tracked

`rm`, `mv`, `cp` and similar bash operations are NOT captured by checkpoints.

**Workarounds**:
1. Prefer Claude's file editing tools over bash commands
2. Manually create checkpoints before/after bash operations: `/checkpoint`
3. Wrap critical bash operations in scripts with state logging

### External Changes Not Tracked

Manual edits or IDE changes are not captured.

**Workarounds**:
1. Establish clear workflow: work entirely in Claude Code OR create checkpoints when switching tools
2. Use file system monitoring to detect external changes
3. Periodically verify checkpoint state matches actual files

### Not a Git Replacement

Checkpoints are for session-level recovery, not version control.

**Best Practice**:
- Commit to Git at key milestones
- Treat checkpoints as "workspace snapshots"
- Treat Git commits as "release candidates"
- Consider automating checkpoint-to-branch synchronization

## Storage Backend Selection

| Backend | Best For | Pros | Cons |
|---------|----------|------|------|
| Local Filesystem | Personal dev | Lowest latency | No redundancy, no sync |
| Cloud Storage (S3, GCS) | Team/Enterprise | High availability, durable | Network latency |
| Hybrid (local cache + cloud) | Balanced | Best of both | Architecture complexity |

**Recommended Configuration**:
- Local cache: 24 hours of checkpoints
- Cloud sync: Every 5 minutes or 10 checkpoints
- Retention: 90 days (production), 30 days (development)

## Monitoring Parameters

For production deployments, monitor:

**Storage Health**:
- Checkpoint storage usage (alert at 80%, critical at 90%)
- Checkpoint creation success rate (target: >99.9%)
- Recovery latency (P95 < 2s, P99 < 5s)

**Task Execution**:
- Average task duration distribution
- Checkpoint frequency per task
- Recovery operation frequency and causes

**Resource Isolation**:
- Sub-agent memory peaks
- File lock conflicts
- Port occupation conflicts

See `references/monitoring-guide.md` for detailed metrics and alerting setup.

## Quick Reference Commands

| Action | Command/Action |
|--------|---------------|
| Create checkpoint | `/checkpoint` or wait for next prompt |
| List checkpoints | Check session history |
| Recover to checkpoint | Select from checkpoint list |
| Check task status | Review TodoWrite state |

## References

For detailed information on specific topics:
- `references/checkpoint-patterns.md` - Advanced checkpoint patterns and strategies
- `references/monitoring-guide.md` - Production monitoring and alerting setup
- `references/sub-agent-patterns.md` - Parallel sub-agent coordination patterns
