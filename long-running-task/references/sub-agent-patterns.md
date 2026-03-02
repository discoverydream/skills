# Sub-Agent Patterns for Long-Running Tasks

## Table of Contents

1. [Sub-Agent Architecture](#sub-agent-architecture)
2. [State Isolation Patterns](#state-isolation-patterns)
3. [Coordination Strategies](#coordination-strategies)
4. [Resource Management](#resource-management)
5. [Checkpoint Integration](#checkpoint-integration)
6. [Common Patterns](#common-patterns)

## Sub-Agent Architecture

### Overview

When orchestrating long-running tasks, sub-agents enable parallel execution while maintaining state isolation.

```
                    Main Agent
                         |
        +----------------+----------------+
        |                |                |
   Sub-Agent A      Sub-Agent B      Sub-Agent C
   (State A)         (State B)        (State C)
        |                |                |
        v                v                v
    Task A           Task B           Task C
```

### Sub-Agent Types

| Type | Purpose | Example Use Case |
|------|---------|------------------|
| Explorer | Investigate codebase | Finding files, understanding patterns |
| Builder | Create/modify code | Implementing features, refactoring |
| Tester | Verify implementations | Running tests, checking coverage |
| Analyzer | Deep analysis | Security audit, performance profiling |

## State Isolation Patterns

### Independent State Spaces

Each sub-agent maintains its own state:

```yaml
Sub-Agent State Components:
  - Working directory context
  - Open file handles
  - Conversation history
  - Checkpoint markers
  - Resource allocations
```

### State Boundary Rules

1. **No Direct State Sharing** - Sub-agents don't access each other's state
2. **Communication via Main Agent** - All coordination through orchestrator
3. **Explicit State Transfer** - When needed, serialize through main agent

### State Isolation Implementation

```
Main Agent Checkpoint (M1)
    |
    +-- Sub-Agent A Checkpoint (M1-A)
    |       |
    |       +-- Independent state space A
    |
    +-- Sub-Agent B Checkpoint (M1-B)
            |
            +-- Independent state space B
```

## Coordination Strategies

### Fan-Out Pattern

Execute independent tasks in parallel:

```
Main Agent
    |
    +-- Spawn Agent A -> Process Module A
    +-- Spawn Agent B -> Process Module B
    +-- Spawn Agent C -> Process Module C
    |
    v
Collect results and aggregate
```

**Use When**:
- Tasks are independent
- No dependencies between sub-tasks
- Order of completion doesn't matter

### Pipeline Pattern

Sequential stages with parallel processing:

```
Stage 1          Stage 2          Stage 3
(Analyze)        (Implement)      (Verify)
    |                |                |
    v                v                v
[Agent A1]  ->   [Agent B1]  ->   [Agent C1]
[Agent A2]  ->   [Agent B2]  ->   [Agent C2]
[Agent A3]  ->   [Agent B3]  ->   [Agent C3]
```

**Use When**:
- Clear stage dependencies
- Each stage can parallelize
- Throughput optimization needed

### Map-Reduce Pattern

Distribute work and aggregate results:

```
          Map Phase
              |
    +---------+---------+
    |         |         |
[Agent 1] [Agent 2] [Agent 3]
    |         |         |
    v         v         v
  Result1   Result2   Result3
    |         |         |
    +---------+---------+
              |
          Reduce Phase
              |
              v
         Aggregated Result
```

**Use When**:
- Large dataset/files to process
- Results need aggregation
- Final output is summary/combined

## Resource Management

### Port Allocation

Prevent port conflicts between sub-agents:

```yaml
# Port pool configuration
port_pool:
  range_start: 8000
  range_end: 8999
  allocation_strategy: dynamic

# Per-agent allocation
sub_agents:
  agent_a:
    ports: [8001, 8002]
  agent_b:
    ports: [8003, 8004]
```

### File Lock Management

Coordinate file access:

```yaml
# File lock protocol
lock_config:
  timeout: 30s
  retry_interval: 1s
  max_retries: 30

# Lock acquisition pattern
acquire_lock:
  1. Check if file is locked
  2. If locked, wait or request main agent mediation
  3. If unlocked, acquire lock
  4. Perform operation
  5. Release lock
```

### Memory Budgets

Allocate memory per sub-agent:

```yaml
memory_config:
  total_budget: 4GB
  per_agent_limit: 1GB
  overflow_strategy: queue

# Memory monitoring
memory_thresholds:
  warning: 80%
  critical: 95%
  action: throttle_or_spawn
```

## Checkpoint Integration

### Hierarchical Checkpoints

Maintain checkpoint hierarchy:

```
Main Checkpoint (MC-1)
    |
    +-- Sub-Agent A Checkpoint (MC-1-A1)
    |       |
    |       +-- Sub-Agent A Checkpoint (MC-1-A2)
    |
    +-- Sub-Agent B Checkpoint (MC-1-B1)
            |
            +-- Sub-Agent B Checkpoint (MC-1-B2)
```

### Recovery with Sub-Agents

When recovering with parallel sub-agents:

1. **Identify Main Checkpoint** - Determine recovery point
2. **Reconstruct Sub-Agent States** - Load each sub-agent's checkpoint
3. **Verify Task Topology** - Ensure all sub-agents accounted for
4. **Resume Execution** - Continue from checkpoint state

### Synchronization Points

Create synchronized checkpoints:

```yaml
sync_checkpoint:
  trigger: milestone_reached
  actions:
    1. Signal all sub-agents to create checkpoint
    2. Wait for all checkpoints to complete
    3. Create main checkpoint with references
    4. Resume execution
```

## Common Patterns

### Pattern 1: Codebase Analysis

**Scenario**: Analyze large codebase across multiple dimensions

```yaml
task: codebase_analysis
main_agent:
  role: coordinator
  spawn:
    - agent: security_analyzer
      scope: all files
      output: security_report
    - agent: dependency_analyzer
      scope: package files
      output: dependency_graph
    - agent: complexity_analyzer
      scope: source files
      output: complexity_metrics
  aggregate:
    - combine all reports
    - identify correlations
    - prioritize issues
```

### Pattern 2: Multi-Module Refactoring

**Scenario**: Refactor multiple related modules

```yaml
task: multi_module_refactor
main_agent:
  role: orchestrator
  phases:
    - name: planning
      agent: architect_agent
      output: refactoring_plan
    - name: execution
      spawn:
        - agent: module_a_refactor
          dependencies: [refactoring_plan]
        - agent: module_b_refactor
          dependencies: [refactoring_plan]
      checkpoints: after_each_module
    - name: verification
      agent: test_agent
      scope: all modules
```

### Pattern 3: Batch File Processing

**Scenario**: Process large number of files

```yaml
task: batch_file_processing
main_agent:
  role: distributor
  config:
    batch_size: 100
    parallel_agents: 4
  workflow:
    - divide files into batches
    - spawn agents for each batch
    - checkpoint every 100 files
    - aggregate results
    - handle failures with retry
```

### Pattern 4: Incremental Build

**Scenario**: Build system with incremental checkpoints

```yaml
task: incremental_build
main_agent:
  role: build_coordinator
  workflow:
    - analyze changed files
    - determine build order
    - spawn agents per component
    - checkpoint after each component
    - verify at each stage
  recovery:
    - resume from last successful component
    - skip already built artifacts
```

## Error Handling in Parallel Tasks

### Sub-Agent Failure

```yaml
failure_handling:
  single_agent_failure:
    action: retry_or_skip
    max_retries: 3
    fallback: mark_as_failed
  critical_agent_failure:
    action: halt_all
    checkpoint: create_emergency
    notify: main_agent
  resource_exhaustion:
    action: queue_or_throttle
    monitoring: memory_usage
```

### Recovery After Failure

```yaml
recovery_workflow:
  1. Identify failed sub-agent
  2. Check checkpoint availability
  3. Options:
     a. Retry from last checkpoint
     b. Spawn replacement agent
     c. Redistribute work to other agents
  4. Resume execution
```

## Best Practices

### Do

- Create checkpoints before spawning sub-agents
- Define clear state boundaries
- Use explicit coordination through main agent
- Monitor resource usage per agent
- Handle failures gracefully

### Don't

- Share mutable state between agents
- Allow direct agent-to-agent communication
- Ignore resource constraints
- Skip checkpoint creation at milestones
- Assume sub-agent order of completion
