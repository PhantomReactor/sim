# Executor Refactor: Loop and Parallel Handling Evolution

## Overview

The executor underwent a **major architectural refactor** from a **layer-based execution model** to a **DAG-based execution model**. This fundamentally changed how loops and parallels are handled.

---

## Architecture Comparison

### Origin/Main (Old Architecture)

```
executor/
├── index.ts                    # Main executor with layer-based execution
├── loops/loops.ts              # LoopManager - post-layer loop processing
├── parallels/parallels.ts      # ParallelManager - virtual block management
├── handlers/
│   ├── loop/loop-handler.ts    # Loop block execution handler
│   └── parallel/parallel-handler.ts  # Parallel block execution handler
├── path/path.ts                # PathTracker for routing decisions
└── resolver/resolver.ts        # InputResolver for variable resolution
```

**Execution Model:**
- **Layer-based**: Executor processed blocks in "layers" (topological levels)
- After each layer, checked if loops/parallels needed to iterate
- **Post-execution processing**: LoopManager and ParallelManager ran AFTER each layer to decide if iteration was needed

### Current Branch (New Architecture)

```
executor/
├── execution/
│   ├── executor.ts             # DAGExecutor - orchestrates DAG execution
│   ├── engine.ts               # ExecutionEngine - ready queue + parallel execution
│   ├── edge-manager.ts         # EdgeManager - intelligent edge activation
│   └── block-executor.ts       # BlockExecutor - delegates to handlers
├── dag/
│   ├── builder.ts              # DAGBuilder - compiles workflow to DAG
│   └── construction/           # DAG construction utilities
├── orchestrators/
│   ├── loop.ts                 # LoopOrchestrator - integrated loop management
│   ├── parallel.ts             # ParallelOrchestrator - integrated parallel management
│   └── node.ts                 # NodeExecutionOrchestrator - coordinates execution
└── variables/
    └── resolver.ts             # VariableResolver - multi-layer resolution
```

**Execution Model:**
- **DAG-based**: Workflow compiles to explicit DAG before execution
- **Ready queue pattern**: Nodes execute when all dependencies satisfied
- **Integrated orchestration**: Loop and parallel logic integrated into execution flow

---

## Loop Handling Differences

### Old Approach (LoopManager)

**How it worked:**
1. **Post-layer processing**: After each execution layer completed, `LoopManager.processLoopIterations()` was called
2. **Reachability analysis**: For each loop, checked if all *reachable* blocks inside the loop had been executed
3. **Manual reset**: If iteration complete, manually reset all block states in the loop
4. **Virtual loop block**: Loop block itself was re-queued to increment iteration counter
5. **Feedback detection**: Connections from loop nodes back to loop block identified as "feedback paths"

```typescript
// OLD: Loop manager checked AFTER layer execution
async processLoopIterations(context: ExecutionContext): Promise<boolean> {
  for (const [loopId, loop] of Object.entries(this.loops)) {
    const loopBlockExecuted = context.executedBlocks.has(loopId)
    const allBlocksInLoopExecuted = this.allBlocksExecuted(loop.nodes, context)
    
    if (allBlocksInLoopExecuted) {
      const currentIteration = context.loopIterations.get(loopId) || 1
      
      if (currentIteration >= maxIterations) {
        // Complete loop - activate end connections
        context.completedLoops.add(loopId)
        // Activate loop-end-source connections
      } else {
        // Reset blocks for next iteration
        this.resetLoopBlocks(loopId, loop, context)
      }
    }
  }
}
```

**Key characteristics:**
- ✅ Reachability analysis respected conditional routing
- ✅ Proper handling of router/condition blocks within loops
- ❌ Required full layer to complete before checking
- ❌ Manual state reset for each iteration
- ❌ Complex "virtual block" tracking
- ❌ Feedback path detection needed special handling

### New Approach (LoopOrchestrator)

**How it works:**
1. **DAG sentinel nodes**: Loops expanded into `__sentinel_loop_start` and `__sentinel_loop_end` nodes
2. **Integrated evaluation**: Loop continuation evaluated as part of DAG execution
3. **Edge-based routing**: Loop continuation vs. exit handled through edge activation
4. **State preservation**: Loop scope maintains all iteration state

```typescript
// NEW: Loop orchestrator integrated into execution flow
evaluateLoopContinuation(ctx: ExecutionContext, loopId: string): LoopContinuationResult {
  const scope = ctx.loopExecutions.get(loopId)
  
  // Collect iteration outputs
  const iterationResults: NormalizedBlockOutput[] = []
  for (const blockOutput of scope.currentIterationOutputs.values()) {
    iterationResults.push(blockOutput)
  }
  scope.allIterationOutputs.push(iterationResults)
  scope.currentIterationOutputs.clear()

  // Evaluate continuation condition
  if (!this.evaluateCondition(ctx, scope, scope.iteration + 1)) {
    return this.createExitResult(ctx, loopId, scope)
  }

  scope.iteration++
  
  if (scope.items && scope.iteration < scope.items.length) {
    scope.item = scope.items[scope.iteration]
  }

  return {
    shouldContinue: true,
    shouldExit: false,
    selectedRoute: EDGE.LOOP_CONTINUE, // Used for edge activation
    currentIteration: scope.iteration,
  }
}
```

**Key characteristics:**
- ✅ Loop evaluation integrated into DAG traversal
- ✅ Edge-based routing (LOOP_CONTINUE vs LOOP_EXIT edges)
- ✅ Cleaner state management with LoopScope
- ✅ No need for manual block reset - handled by DAG structure
- ✅ Support for 4 loop types: `for`, `forEach`, `while`, `doWhile`
- ✅ Condition evaluation happens at sentinel nodes

---

## Parallel Handling Differences

### Old Approach (ParallelManager)

**How it worked:**
1. **Virtual block instances**: Created virtual copies of each block for each parallel iteration
   - Example: `blockId_parallel_parallelId_iteration_0`, `blockId_parallel_parallelId_iteration_1`, etc.
2. **Post-layer processing**: After each layer, `ParallelManager.processParallelIterations()` checked if all virtual blocks completed
3. **Active path tracking**: Virtual blocks added to `activeExecutionPath` to mark them for execution
4. **Iteration context**: Stored current item/index in `context.loopItems` keyed by iteration

```typescript
// OLD: Create virtual block instances for parallel execution
createVirtualBlockInstances(
  block: SerializedBlock,
  parallelId: string,
  parallelState: ParallelState,
  executedBlocks: Set<string>,
  activeExecutionPath: Set<string>
): string[] {
  const virtualBlockIds: string[] = []

  for (let i = 0; i < parallelState.parallelCount; i++) {
    const virtualBlockId = `${block.id}_parallel_${parallelId}_iteration_${i}`

    if (executedBlocks.has(virtualBlockId)) {
      continue // Skip already executed
    }

    if (!activeExecutionPath.has(virtualBlockId) && !activeExecutionPath.has(block.id)) {
      continue // Skip if not in active path
    }

    virtualBlockIds.push(virtualBlockId)
  }

  return virtualBlockIds
}
```

**Key characteristics:**
- ✅ Clear separation of parallel iterations through naming
- ✅ Conditional routing respected within parallel branches
- ❌ Virtual block naming convention needed parsing
- ❌ Required special utilities (`VirtualBlockUtils`, `ParallelRoutingUtils`)
- ❌ Post-layer aggregation triggered parallel block re-execution
- ❌ Complex tracking of "which virtual blocks SHOULD execute"

### New Approach (ParallelOrchestrator)

**How it works:**
1. **DAG branch expansion**: Parallel blocks expanded into separate DAG nodes with branch suffixes
   - Example: `blockId<0>`, `blockId<1>`, `blockId<2>`
2. **Parallel scope tracking**: Single `ParallelScope` object tracks all branches
3. **Branch-aware execution**: Each node knows its branch index through metadata
4. **Completion detection**: Tracks `completedCount` vs `totalExpectedNodes`

```typescript
// NEW: Integrated parallel scope management
handleParallelBranchCompletion(
  ctx: ExecutionContext,
  parallelId: string,
  nodeId: string,
  output: NormalizedBlockOutput
): boolean {
  const scope = ctx.parallelExecutions.get(parallelId)
  const branchIndex = extractBranchIndex(nodeId) // Parse from nodeId like "blockId<2>"
  
  if (!scope.branchOutputs.has(branchIndex)) {
    scope.branchOutputs.set(branchIndex, [])
  }
  scope.branchOutputs.get(branchIndex).push(output)
  scope.completedCount++

  // Return true if all branches complete
  return scope.completedCount >= scope.totalExpectedNodes
}
```

**Key characteristics:**
- ✅ True parallel execution (nodes execute concurrently)
- ✅ Cleaner branch indexing with `<N>` suffix
- ✅ Single source of truth for parallel state
- ✅ Integrated into DAG execution flow
- ✅ No re-execution needed for aggregation
- ✅ Branch metadata propagated through execution

---

## Execution Flow Comparison

### Old: Layer-Based Execution

```
1. Get next execution layer (all blocks whose dependencies are satisfied)
2. Execute all blocks in layer (sequentially)
3. After layer completes:
   a. Run LoopManager.processLoopIterations()
   b. Run ParallelManager.processParallelIterations()
4. If loops/parallels need iteration:
   - Reset block states
   - Add blocks back to active path
5. Repeat until no more layers
```

**Issues:**
- Sequential execution within layers (no parallelism)
- Post-processing adds overhead
- Manual state management
- Complex virtual block tracking

### New: DAG-Based Ready Queue

```
1. Build DAG from workflow
2. Initialize ready queue with nodes that have no incoming edges
3. While queue has work:
   a. Dequeue all ready nodes
   b. Execute ALL ready nodes in PARALLEL (Promise.all)
   c. On completion, remove incoming edges from downstream nodes
   d. If downstream node becomes ready, add to queue
4. Continue until queue empty
```

**Advantages:**
- ✅ True parallel execution of independent nodes
- ✅ No post-processing overhead
- ✅ DAG structure makes dependencies explicit
- ✅ Edge activation handles routing automatically

---

## Variable Resolution

### Old: Flat Resolution with Loop/Parallel Context

```typescript
// OLD: Single resolver with manual loop/parallel item tracking
class InputResolver {
  resolveInputs(ctx, blockId, params) {
    // Check for loop context
    const loopItem = ctx.loopItems.get(blockId)
    
    // Check for parallel iteration context
    const parallelItem = ctx.loopItems.get(`parallelId_iteration_${index}`)
    
    // Resolve references
    return this.resolveReferences(params)
  }
}
```

### New: Multi-Layer Variable Resolver

```typescript
// NEW: Layered resolver with scope hierarchy
class VariableResolver {
  private resolvers: Resolver[] = [
    new LoopResolver(workflow),        // <loop.iteration>, <loop.item>
    new ParallelResolver(workflow),    // <parallel.branchIndex>
    new WorkflowResolver(workflowVariables), // <workflow.variableName>
    new EnvResolver(),                 // ${ENV_VAR}
    new BlockResolver(workflow),       // <blockId.output.path>
  ]
  
  resolveReference(reference, context) {
    // Try resolvers in order - inner scopes shadow outer scopes
    for (const resolver of this.resolvers) {
      if (resolver.canResolve(reference)) {
        return resolver.resolve(reference, context)
      }
    }
  }
}
```

**Advantages:**
- ✅ Proper scope hierarchy
- ✅ Explicit loop and parallel variable syntax
- ✅ Type-safe resolution
- ✅ Easy to add new variable types

---

## Key Improvements Summary

### Loops
1. **No more manual state reset** - DAG structure handles it
2. **Condition evaluation integrated** - not a post-processing step
3. **Edge-based routing** - LOOP_CONTINUE vs LOOP_EXIT edges
4. **Better state management** - LoopScope is clean and explicit
5. **Support for while/doWhile** - more loop types

### Parallels
1. **True parallelism** - nodes execute concurrently via ready queue
2. **Cleaner branch indexing** - `<N>` suffix vs long virtual IDs
3. **No re-execution for aggregation** - results collected as branches complete
4. **Integrated into DAG** - not a post-processing concern
5. **Branch-aware pause/resume** - can pause individual branches

### Overall Architecture
1. **Explicit DAG structure** - dependencies are clear
2. **Ready queue pattern** - enables true parallelism
3. **Intelligent edge activation** - routes based on outputs
4. **Snapshot/resume support** - can serialize and restore full state
5. **Better separation of concerns** - orchestrators vs handlers vs executors

---

## Migration Impact

The refactor is a **breaking internal change** but maintains API compatibility:

✅ **Preserved:**
- Serialized workflow format
- Block handler interface
- Execution result structure
- Variable reference syntax (mostly)

🔄 **Changed:**
- Internal execution flow (layer → DAG)
- Loop/parallel state management (managers → orchestrators)
- Variable resolution (flat → layered)
- Pause/resume implementation (simpler → more robust)

❌ **Removed:**
- Virtual block naming complexity
- Post-layer processing hooks
- Manual state reset logic
- Feedback path detection

The new architecture is **more maintainable**, **more performant**, and **more capable** (true parallelism, better pause/resume, cleaner state management).

