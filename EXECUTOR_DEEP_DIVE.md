# Executor Deep Dive: Old vs New Architecture

## Table of Contents
1. [Execution Flow](#execution-flow)
2. [Dependency Resolution](#dependency-resolution)
3. [Loop Mechanics](#loop-mechanics)
4. [Parallel Execution](#parallel-execution)
5. [Condition Evaluation](#condition-evaluation)
6. [Variable Resolution](#variable-resolution)
7. [Trigger Handling](#trigger-handling)
8. [Edge Construction & Routing](#edge-construction--routing)

---

## 1. Execution Flow

### Old System: Layer-Based Execution

**Core Concept:** Execute blocks in "layers" where each layer contains blocks whose dependencies are all satisfied.

```typescript
// OLD: Main execution loop
async execute(workflowId: string, startBlockId?: string): Promise<ExecutionResult> {
  let hasMoreLayers = true
  let iteration = 0
  
  while (hasMoreLayers && iteration < MAX_ITERATIONS) {
    // 1. Get next execution layer
    const nextLayer = this.getNextExecutionLayer(context)
    
    if (nextLayer.length === 0) {
      // Check if parallels have more work
      hasMoreLayers = this.hasMoreParallelWork(context)
    } else {
      // 2. Execute all blocks in layer SEQUENTIALLY
      const outputs = await this.executeLayer(nextLayer, context)
      
      // 3. POST-EXECUTION: Process loops and parallels
      const loopReachedMax = await this.loopManager.processLoopIterations(context)
      await this.parallelManager.processParallelIterations(context)
      
      // 4. Check if there's more work
      const updatedNextLayer = this.getNextExecutionLayer(context)
      if (updatedNextLayer.length === 0) {
        hasMoreLayers = false
      }
    }
    
    iteration++
  }
}
```

**Layer Determination Logic:**

```typescript
// OLD: getNextExecutionLayer - determines which blocks can execute
private getNextExecutionLayer(context: ExecutionContext): string[] {
  const pendingBlocks = new Set<string>()
  
  for (const block of this.workflow.blocks) {
    // Skip executed or disabled blocks
    if (context.executedBlocks.has(block.id) || !block.enabled) {
      continue
    }
    
    // CRITICAL: Only consider blocks in active execution path
    if (!context.activeExecutionPath.has(block.id)) {
      continue
    }
    
    // Check dependencies
    const incomingConnections = this.workflow.connections.filter(
      conn => conn.target === block.id
    )
    
    const allDependenciesMet = this.checkDependencies(
      incomingConnections,
      context.executedBlocks,
      context
    )
    
    if (allDependenciesMet) {
      pendingBlocks.add(block.id)
    }
  }
  
  // Special handling for parallels
  this.processParallelBlocks(activeParallels, context, pendingBlocks)
  
  return Array.from(pendingBlocks)
}
```

**Key Characteristics:**
- ✅ Simple conceptual model (layers = "rounds" of execution)
- ❌ **Sequential execution within layers** - no true parallelism
- ❌ **Post-processing overhead** - loop/parallel checks after every layer
- ❌ **Active path tracking** - manual management of which blocks are "active"

### New System: DAG-Based Ready Queue

**Core Concept:** Execute nodes as soon as all their dependencies are satisfied, using a ready queue pattern.

```typescript
// NEW: Main execution loop
async run(triggerBlockId?: string): Promise<ExecutionResult> {
  this.initializeQueue(triggerBlockId)
  
  while (this.hasWork()) {
    await this.processQueue()
  }
  
  await this.waitForAllExecutions()
  
  return {
    success: true,
    output: this.finalOutput,
    logs: context.blockLogs,
    metadata: context.metadata
  }
}

private hasWork(): boolean {
  return this.readyQueue.length > 0 || this.executing.size > 0
}
```

**Ready Queue Processing:**

```typescript
// NEW: Process queue with true parallelism
private async processQueue(): Promise<void> {
  // Launch ALL ready nodes in parallel
  while (this.readyQueue.length > 0) {
    const nodeId = this.dequeue()
    if (!nodeId) continue
    
    // DON'T await - launch async and track
    const promise = this.executeNodeAsync(nodeId)
    this.trackExecution(promise)
  }
  
  // Wait for at least one to complete before checking queue again
  if (this.executing.size > 0) {
    await this.waitForAnyExecution()
  }
}

private trackExecution(promise: Promise<void>): void {
  this.executing.add(promise)
  promise.finally(() => {
    this.executing.delete(promise)
  })
}
```

**Node Completion Handling:**

```typescript
// NEW: When a node completes, activate downstream edges
private async handleNodeCompletion(
  nodeId: string,
  output: NormalizedBlockOutput,
  isFinalOutput: boolean
): Promise<void> {
  const node = this.dag.nodes.get(nodeId)
  
  // Process outgoing edges to find newly ready nodes
  const readyNodes = this.edgeManager.processOutgoingEdges(node, output, false)
  
  // Add newly ready nodes to queue
  this.addMultipleToQueue(readyNodes)
}
```

**Key Characteristics:**
- ✅ **True parallelism** - independent nodes execute concurrently
- ✅ **No post-processing** - loop/parallel logic integrated
- ✅ **Automatic dependency tracking** - DAG edges handle it
- ✅ **Event-driven** - nodes added to queue as dependencies satisfy

---

## 2. Dependency Resolution

### Old System: Manual Dependency Checking

```typescript
// OLD: Check if all dependencies are met
private checkDependencies(
  incomingConnections: Connection[],
  executedBlocks: Set<string>,
  context: ExecutionContext
): boolean {
  if (incomingConnections.length === 0) {
    return true
  }
  
  return incomingConnections.every(conn => {
    const sourceExecuted = executedBlocks.has(conn.source)
    const sourceBlockState = context.blockStates.get(conn.source)
    const hasSourceError = sourceBlockState?.output?.error !== undefined
    
    // Handle error edges
    if (conn.sourceHandle === 'error') {
      return sourceExecuted && hasSourceError
    }
    
    // Handle regular edges
    if (conn.sourceHandle === 'source' || !conn.sourceHandle) {
      return sourceExecuted && !hasSourceError
    }
    
    // For router blocks, check if this specific path was chosen
    const sourceBlock = this.workflow.blocks.find(b => b.id === conn.source)
    if (sourceBlock?.metadata?.id === BlockType.ROUTER) {
      const selectedTarget = context.decisions.router.get(conn.source)
      return sourceExecuted && selectedTarget === conn.target
    }
    
    // For condition blocks, check if this condition path was chosen
    if (sourceBlock?.metadata?.id === BlockType.CONDITION) {
      const selectedConditionId = context.decisions.condition.get(conn.source)
      return sourceExecuted && conn.sourceHandle === `condition-${selectedConditionId}`
    }
    
    return sourceExecuted
  })
}
```

**Problems with Old Approach:**
1. **Repeated scanning**: Every iteration re-checks all blocks
2. **N×M complexity**: For N blocks with M connections, checks N×M dependencies per layer
3. **No edge deactivation**: Unselected paths still count as dependencies
4. **Manual router/condition handling**: Special cases everywhere

### New System: Edge-Based Dependency Tracking

```typescript
// NEW: DAG node structure with explicit edges
export interface DAGNode {
  id: string
  block: SerializedBlock
  incomingEdges: Set<string>      // Set of source node IDs
  outgoingEdges: Map<string, DAGEdge>  // Map of edge ID to edge data
  metadata: NodeMetadata
}

// NEW: Edge manager processes completions
processOutgoingEdges(
  node: DAGNode,
  output: NormalizedBlockOutput,
  skipBackwardsEdge = false
): string[] {
  const readyNodes: string[] = []
  
  for (const [edgeId, edge] of node.outgoingEdges) {
    // Determine if this edge should activate
    const shouldActivate = this.shouldActivateEdge(edge, output)
    
    if (!shouldActivate) {
      // Deactivate this edge AND all descendants
      this.deactivateEdgeAndDescendants(node.id, edge.target, edge.sourceHandle)
      continue
    }
    
    // Remove this node from target's incoming edges
    const targetNode = this.dag.nodes.get(edge.target)
    targetNode.incomingEdges.delete(node.id)
    
    // If target has no more incoming edges, it's ready
    if (this.isNodeReady(targetNode)) {
      readyNodes.push(targetNode.id)
    }
  }
  
  return readyNodes
}

isNodeReady(node: DAGNode): boolean {
  if (node.incomingEdges.size === 0) {
    return true
  }
  
  // Check if all remaining incoming edges are deactivated
  const activeIncomingCount = this.countActiveIncomingEdges(node)
  return activeIncomingCount === 0
}
```

**Intelligent Edge Activation:**

```typescript
// NEW: Determine if edge should activate based on output
private shouldActivateEdge(edge: DAGEdge, output: NormalizedBlockOutput): boolean {
  const handle = edge.sourceHandle
  
  // Condition blocks: activate only the selected condition's edge
  if (handle?.startsWith(EDGE.CONDITION_PREFIX)) {
    const conditionValue = handle.substring(EDGE.CONDITION_PREFIX.length)
    return output.selectedOption === conditionValue
  }
  
  // Router blocks: activate only the selected route's edge
  if (handle?.startsWith(EDGE.ROUTER_PREFIX)) {
    const routeId = handle.substring(EDGE.ROUTER_PREFIX.length)
    return output.selectedRoute === routeId
  }
  
  // Loop edges
  switch (handle) {
    case EDGE.LOOP_CONTINUE:
      return output.selectedRoute === EDGE.LOOP_CONTINUE
    case EDGE.LOOP_EXIT:
      return output.selectedRoute === EDGE.LOOP_EXIT
    case EDGE.ERROR:
      return !!output.error
    case EDGE.SOURCE:
      return !output.error
    default:
      return true
  }
}
```

**Advantages:**
- ✅ **O(1) edge removal** - just delete from Set
- ✅ **Automatic pruning** - deactivate unreachable paths immediately
- ✅ **No rescanning** - edges removed once
- ✅ **Declarative routing** - edge handles encode logic

---

## 3. Loop Mechanics

### Old System: Post-Layer Loop Processing

**Loop Handler Execution:**

```typescript
// OLD: Loop handler sets up iteration context
async execute(
  block: SerializedBlock,
  inputs: Record<string, any>,
  context: ExecutionContext
): Promise<BlockOutput> {
  const loop = context.workflow?.loops?.[block.id]
  
  // Initialize iteration counter
  if (!context.loopIterations.has(block.id)) {
    context.loopIterations.set(block.id, 1)
  }
  
  const currentIteration = context.loopIterations.get(block.id) || 1
  
  // Evaluate max iterations based on loop type
  if (loop.loopType === 'forEach') {
    const forEachItems = await this.evaluateForEachItems(loop.forEachItems, context, block)
    const itemsLength = Array.isArray(forEachItems) 
      ? forEachItems.length 
      : Object.keys(forEachItems).length
    maxIterations = itemsLength
    
    // Set current item for this iteration
    context.loopItems.set(`${block.id}_items`, forEachItems)
    const arrayIndex = currentIteration - 1
    const currentItem = Array.isArray(forEachItems)
      ? forEachItems[arrayIndex]
      : Object.entries(forEachItems)[arrayIndex]
    context.loopItems.set(block.id, currentItem)
  } 
  else if (loop.loopType === 'while' || loop.loopType === 'doWhile') {
    // Evaluate condition using InputResolver
    const shouldContinueLoop = await evaluateConditionExpression(
      loop.whileCondition,
      context,
      block,
      this.resolver
    )
    
    if (!shouldContinueLoop) {
      // Exit loop - activate loop-end connections
      context.completedLoops.add(block.id)
      const loopEndConnections = context.workflow?.connections.filter(
        conn => conn.source === block.id && conn.sourceHandle === 'loop-end-source'
      )
      for (const conn of loopEndConnections) {
        context.activeExecutionPath.add(conn.target)
      }
      return { completed: true }
    }
  }
  
  // Activate loop-start connections (blocks inside loop)
  const loopStartConnections = context.workflow?.connections.filter(
    conn => conn.source === block.id && conn.sourceHandle === 'loop-start-source'
  )
  for (const conn of loopStartConnections) {
    context.activeExecutionPath.add(conn.target)
  }
  
  return { currentIteration, maxIterations, loopType: loop.loopType }
}
```

**Post-Layer Loop Manager:**

```typescript
// OLD: After each layer, check if loops should iterate
async processLoopIterations(context: ExecutionContext): Promise<boolean> {
  for (const [loopId, loop] of Object.entries(this.loops)) {
    // Skip completed loops
    if (context.completedLoops.has(loopId)) continue
    
    // Check if loop block executed
    const loopBlockExecuted = context.executedBlocks.has(loopId)
    if (!loopBlockExecuted) continue
    
    // Check if ALL blocks inside loop have executed
    const allBlocksInLoopExecuted = this.allBlocksExecuted(loop.nodes, context)
    
    if (allBlocksInLoopExecuted) {
      const currentIteration = context.loopIterations.get(loopId) || 1
      const maxIterations = this.getMaxIterations(loopId, loop, context)
      
      if (currentIteration >= maxIterations) {
        // Loop complete - aggregate results and activate end connections
        const results = this.collectIterationResults(loopId, context)
        context.blockStates.set(loopId, {
          output: { loopId, results, completed: true },
          executed: true,
          executionTime: 0
        })
        context.completedLoops.add(loopId)
        
        const loopEndConnections = context.workflow?.connections.filter(
          conn => conn.source === loopId && conn.sourceHandle === 'loop-end-source'
        )
        for (const conn of loopEndConnections) {
          context.activeExecutionPath.add(conn.target)
        }
      } else {
        // RESET LOOP STATE for next iteration
        context.loopIterations.set(loopId, currentIteration + 1)
        this.resetLoopBlocks(loopId, loop, context)
        
        // Re-execute loop block
        context.executedBlocks.delete(loopId)
        context.blockStates.delete(loopId)
      }
    }
  }
}

private resetLoopBlocks(loopId: string, loop: Loop, context: ExecutionContext): void {
  // Clear execution state for all blocks in loop
  for (const nodeId of loop.nodes) {
    context.executedBlocks.delete(nodeId)
    context.blockStates.delete(nodeId)
    context.activeExecutionPath.delete(nodeId)
    context.decisions.router.delete(nodeId)
    context.decisions.condition.delete(nodeId)
  }
}
```

**Reachability Analysis:**

```typescript
// OLD: Determine which blocks in loop SHOULD have executed
private allBlocksExecuted(nodeIds: string[], context: ExecutionContext): boolean {
  // Build graph of connections within loop
  const loopConnections = context.workflow?.connections.filter(
    conn => nodeIds.includes(conn.source) && nodeIds.includes(conn.target)
  )
  
  // Find entry points (nodes with external incoming connections)
  const entryBlocks = nodeIds.filter(nodeId =>
    ConnectionUtils.isEntryPoint(nodeId, nodeIds, context.workflow?.connections || [])
  )
  
  // Traverse graph to find all REACHABLE blocks
  const reachableBlocks = new Set<string>()
  const toVisit = [...entryBlocks]
  
  while (toVisit.length > 0) {
    const currentBlockId = toVisit.shift()!
    if (reachableBlocks.has(currentBlockId)) continue
    
    reachableBlocks.add(currentBlockId)
    
    const block = context.workflow?.blocks.find(b => b.id === currentBlockId)
    const outgoing = loopConnections.filter(conn => conn.source === currentBlockId)
    
    // For router blocks, only follow selected path
    if (block?.metadata?.id === BlockType.ROUTER) {
      const selectedTarget = context.decisions.router.get(currentBlockId)
      if (selectedTarget && nodeIds.includes(selectedTarget)) {
        toVisit.push(selectedTarget)
      }
    }
    // For condition blocks, only follow selected condition
    else if (block?.metadata?.id === BlockType.CONDITION) {
      const selectedConditionId = context.decisions.condition.get(currentBlockId)
      if (selectedConditionId) {
        const selectedConnection = outgoing.find(
          conn => conn.sourceHandle === `condition-${selectedConditionId}`
        )
        if (selectedConnection?.target) {
          toVisit.push(selectedConnection.target)
        }
      }
    }
    // For regular blocks, follow error or success path
    else {
      const blockState = context.blockStates.get(currentBlockId)
      const hasError = blockState?.output?.error !== undefined
      
      for (const conn of outgoing) {
        if (conn.sourceHandle === 'error' && hasError) {
          toVisit.push(conn.target)
        } else if ((conn.sourceHandle === 'source' || !conn.sourceHandle) && !hasError) {
          toVisit.push(conn.target)
        }
      }
    }
  }
  
  // Check if all reachable blocks executed
  for (const reachableBlockId of reachableBlocks) {
    if (!context.executedBlocks.has(reachableBlockId)) {
      return false
    }
  }
  
  return true
}
```

**Problems with Old Approach:**
1. **Post-execution overhead**: Reachability analysis runs after EVERY layer
2. **Manual state reset**: Must explicitly clear all block states
3. **Complex routing logic**: Special cases for routers, conditions, errors
4. **While loop evaluation timing**: Condition evaluated at loop block, not at continuation point

### New System: DAG Sentinel Nodes

**Loop DAG Structure:**

```
Original:    [Loop Block] ──loop-start──> [Block A] ──> [Block B] ──loop-end──> [Loop Block]

DAG:         [__sentinel_loop_start_loopId] ──> [Block A] ──> [Block B] ──> [__sentinel_loop_end_loopId]
                                                                                    │
                                                                                    ├── LOOP_CONTINUE ──┐
                                                                                    │                   │
                                                                                    └── LOOP_EXIT ──────┤
                                                                                                        │
                        [Next Block After Loop] <───────────────────────────────────────────────────────┘
```

**Loop Orchestrator Initialization:**

```typescript
// NEW: Initialize loop scope when sentinel start executes
initializeLoopScope(ctx: ExecutionContext, loopId: string): LoopScope {
  const loopConfig = this.dag.loopConfigs.get(loopId)
  
  const scope: LoopScope = {
    iteration: 0,
    currentIterationOutputs: new Map(),
    allIterationOutputs: []
  }
  
  switch (loopConfig.loopType) {
    case 'for':
      scope.maxIterations = loopConfig.iterations || DEFAULTS.MAX_LOOP_ITERATIONS
      scope.condition = buildLoopIndexCondition(scope.maxIterations)
      break
      
    case 'forEach':
      const items = this.resolveForEachItems(ctx, loopConfig.forEachItems)
      scope.items = items
      scope.maxIterations = items.length
      scope.item = items[0]  // Set first item
      scope.condition = buildLoopIndexCondition(scope.maxIterations)
      break
      
    case 'while':
      scope.condition = loopConfig.whileCondition
      break
      
    case 'doWhile':
      scope.condition = loopConfig.doWhileCondition
      scope.skipFirstConditionCheck = true  // DoWhile runs once before check
      break
  }
  
  ctx.loopExecutions.set(loopId, scope)
  return scope
}
```

**Loop Continuation Evaluation:**

```typescript
// NEW: Evaluate at sentinel end node
evaluateLoopContinuation(ctx: ExecutionContext, loopId: string): LoopContinuationResult {
  const scope = ctx.loopExecutions.get(loopId)
  
  // 1. Collect outputs from this iteration
  const iterationResults: NormalizedBlockOutput[] = []
  for (const blockOutput of scope.currentIterationOutputs.values()) {
    iterationResults.push(blockOutput)
  }
  scope.allIterationOutputs.push(iterationResults)
  scope.currentIterationOutputs.clear()
  
  // 2. Evaluate condition (for while/doWhile/for/forEach)
  const isFirstIteration = scope.iteration === 0
  const shouldSkipFirstCheck = scope.skipFirstConditionCheck && isFirstIteration
  
  if (!shouldSkipFirstCheck) {
    if (!this.evaluateCondition(ctx, scope, scope.iteration + 1)) {
      // Exit loop
      return this.createExitResult(ctx, loopId, scope)
    }
  }
  
  // 3. Increment iteration
  scope.iteration++
  
  // 4. Update forEach item for next iteration
  if (scope.items && scope.iteration < scope.items.length) {
    scope.item = scope.items[scope.iteration]
  }
  
  return {
    shouldContinue: true,
    shouldExit: false,
    selectedRoute: EDGE.LOOP_CONTINUE,  // This activates continue edge
    currentIteration: scope.iteration
  }
}

private evaluateCondition(ctx: ExecutionContext, scope: LoopScope, iteration: number): boolean {
  if (!scope.condition) return false
  
  // Temporarily set iteration for evaluation
  const currentIteration = scope.iteration
  scope.iteration = iteration
  
  const result = this.evaluateWhileCondition(ctx, scope.condition, scope)
  
  // Restore original iteration
  scope.iteration = currentIteration
  return result
}
```

**Edge Activation Based on Result:**

```typescript
// NEW: EdgeManager activates appropriate edge
private shouldActivateEdge(edge: DAGEdge, output: NormalizedBlockOutput): boolean {
  switch (edge.sourceHandle) {
    case EDGE.LOOP_CONTINUE:
      return output.selectedRoute === EDGE.LOOP_CONTINUE
    case EDGE.LOOP_EXIT:
      return output.selectedRoute === EDGE.LOOP_EXIT
    default:
      return true
  }
}
```

**State Restoration on Continue:**

```typescript
// NEW: Restore loop state for next iteration
clearLoopExecutionState(loopId: string): void {
  const loopConfig = this.dag.loopConfigs.get(loopId)
  const sentinelStartId = buildSentinelStartId(loopId)
  const sentinelEndId = buildSentinelEndId(loopId)
  
  // Unmark sentinel nodes and loop blocks as executed
  this.state.unmarkExecuted(sentinelStartId)
  this.state.unmarkExecuted(sentinelEndId)
  for (const loopNodeId of loopConfig.nodes) {
    this.state.unmarkExecuted(loopNodeId)
  }
}

restoreLoopEdges(loopId: string): void {
  const loopConfig = this.dag.loopConfigs.get(loopId)
  const sentinelStartId = buildSentinelStartId(loopId)
  const sentinelEndId = buildSentinelEndId(loopId)
  const allLoopNodeIds = new Set([sentinelStartId, sentinelEndId, ...loopConfig.nodes])
  
  // Restore incoming edges for all loop nodes (except backward edge)
  for (const nodeId of allLoopNodeIds) {
    const nodeToRestore = this.dag.nodes.get(nodeId)
    
    for (const [potentialSourceId, potentialSourceNode] of this.dag.nodes) {
      if (!allLoopNodeIds.has(potentialSourceId)) continue
      
      for (const [_, edge] of potentialSourceNode.outgoingEdges) {
        if (edge.target === nodeId) {
          const isBackwardEdge = edge.sourceHandle === EDGE.LOOP_CONTINUE
          if (!isBackwardEdge) {
            nodeToRestore.incomingEdges.add(potentialSourceId)
          }
        }
      }
    }
  }
}
```

**Advantages:**
- ✅ **No post-processing** - continuation evaluated as part of DAG traversal
- ✅ **Explicit sentinel nodes** - loop structure visible in DAG
- ✅ **Edge-based routing** - LOOP_CONTINUE vs LOOP_EXIT edges
- ✅ **Clean state management** - LoopScope tracks everything
- ✅ **Better condition timing** - evaluated at sentinel end, not loop block

---

## 4. Parallel Execution

### Old System: Virtual Block Instances

**Virtual Block Creation:**

```typescript
// OLD: Create virtual copies of each block for each iteration
createVirtualBlockInstances(
  block: SerializedBlock,
  parallelId: string,
  parallelState: ParallelState,
  executedBlocks: Set<string>,
  activeExecutionPath: Set<string>
): string[] {
  const virtualBlockIds: string[] = []
  
  for (let i = 0; i < parallelState.parallelCount; i++) {
    // Generate virtual block ID
    const virtualBlockId = `${block.id}_parallel_${parallelId}_iteration_${i}`
    
    // Skip if already executed
    if (executedBlocks.has(virtualBlockId)) {
      continue
    }
    
    // Skip if not in active path
    if (!activeExecutionPath.has(virtualBlockId) && !activeExecutionPath.has(block.id)) {
      continue
    }
    
    virtualBlockIds.push(virtualBlockId)
  }
  
  return virtualBlockIds
}
```

**Parallel Block Processing in Layer:**

```typescript
// OLD: Special handling in getNextExecutionLayer
private processParallelBlocks(
  activeParallels: Map<string, ParallelState>,
  context: ExecutionContext,
  pendingBlocks: Set<string>
): void {
  for (const [parallelId, parallelState] of activeParallels) {
    const parallel = this.workflow.parallels?.[parallelId]
    
    // Process each iteration
    for (let iteration = 0; iteration < parallelState.parallelCount; iteration++) {
      if (this.isIterationComplete(parallelId, iteration, parallel, context)) {
        continue
      }
      
      this.processParallelIteration(parallelId, iteration, parallel, context, pendingBlocks)
    }
  }
}

private processParallelIteration(
  parallelId: string,
  iteration: number,
  parallel: Parallel,
  context: ExecutionContext,
  pendingBlocks: Set<string>
): void {
  const iterationBlocks = new Map<string, IterationBlockInfo>()
  
  // Build map of virtual blocks for this iteration
  for (const nodeId of parallel.nodes) {
    const virtualBlockId = VirtualBlockUtils.generateParallelId(nodeId, parallelId, iteration)
    const isExecuted = context.executedBlocks.has(virtualBlockId)
    
    if (isExecuted) continue
    
    // Find dependencies within this iteration
    const dependencies: string[] = []
    const incomingConnections = this.workflow.connections.filter(
      conn => conn.target === nodeId && parallel.nodes.includes(conn.source)
    )
    
    for (const conn of incomingConnections) {
      const depVirtualId = VirtualBlockUtils.generateParallelId(conn.source, parallelId, iteration)
      dependencies.push(depVirtualId)
    }
    
    iterationBlocks.set(virtualBlockId, {
      virtualBlockId,
      originalBlockId: nodeId,
      dependencies,
      isExecuted
    })
  }
  
  // Find blocks with no unmet dependencies within this iteration
  for (const [virtualBlockId, blockInfo] of iterationBlocks) {
    const unmetDependencies = blockInfo.dependencies.filter(depId => {
      return !context.executedBlocks.has(depId) && iterationBlocks.has(depId)
    })
    
    if (unmetDependencies.length === 0) {
      pendingBlocks.add(virtualBlockId)
      
      // Store mapping
      context.parallelBlockMapping.set(virtualBlockId, {
        originalBlockId: blockInfo.originalBlockId,
        parallelId: parallelId,
        iterationIndex: iteration
      })
    }
  }
}
```

**Execution Context Setup:**

```typescript
// OLD: Set up context for virtual block
setupIterationContext(
  context: ExecutionContext,
  parallelInfo: { parallelId: string; iterationIndex: number }
): void {
  const parallelState = context.parallelExecutions?.get(parallelInfo.parallelId)
  
  if (parallelState?.distributionItems) {
    const currentItem = this.getIterationItem(parallelState, parallelInfo.iterationIndex)
    
    // Store current item for this iteration
    const iterationKey = `${parallelInfo.parallelId}_iteration_${parallelInfo.iterationIndex}`
    context.loopItems.set(iterationKey, currentItem)
    context.loopItems.set(parallelInfo.parallelId, currentItem)
    context.loopIterations.set(parallelInfo.parallelId, parallelInfo.iterationIndex)
  }
}
```

**Post-Layer Parallel Manager:**

```typescript
// OLD: Check completion after each layer
async processParallelIterations(context: ExecutionContext): Promise<void> {
  for (const [parallelId, parallel] of Object.entries(this.parallels)) {
    if (context.completedLoops.has(parallelId)) continue
    
    const parallelBlockExecuted = context.executedBlocks.has(parallelId)
    if (!parallelBlockExecuted) continue
    
    const parallelState = context.parallelExecutions?.get(parallelId)
    if (!parallelState) continue
    
    // Check if all virtual blocks executed
    const allVirtualBlocksExecuted = this.areAllVirtualBlocksExecuted(
      parallelId,
      parallel,
      context.executedBlocks,
      parallelState,
      context
    )
    
    if (allVirtualBlocksExecuted) {
      // Re-execute parallel block to aggregate results
      context.executedBlocks.delete(parallelId)
      context.activeExecutionPath.add(parallelId)
      
      // Remove child nodes from active path
      for (const nodeId of parallel.nodes) {
        context.activeExecutionPath.delete(nodeId)
      }
    }
  }
}
```

**Problems with Old Approach:**
1. **Still sequential within layer**: Virtual blocks in same layer execute sequentially
2. **Complex ID management**: Parsing `blockId_parallel_parallelId_iteration_N`
3. **Re-execution for aggregation**: Parallel block must run again to collect results
4. **Routing complexity**: Special utilities needed to determine which virtual blocks should execute

### New System: DAG Branch Expansion

**Branch Node Structure:**

```
Original:    [Block A] ──> [Parallel] ──> [Block B] ──> [Block C] ──> [Parallel] ──> [Block D]

DAG:         [Block A] ──┬──> [Block B<0>] ──> [Block C<0>] ──┬──> [Block D]
                         ├──> [Block B<1>] ──> [Block C<1>] ──┤
                         ├──> [Block B<2>] ──> [Block C<2>] ──┤
                         └──> [Block B<3>] ──> [Block C<3>] ──┘
                         
Where <N> suffix = branch index
```

**Parallel Scope Management:**

```typescript
// NEW: Initialize parallel scope
initializeParallelScope(
  ctx: ExecutionContext,
  parallelId: string,
  totalBranches: number,
  terminalNodesCount = 1
): ParallelScope {
  const scope: ParallelScope = {
    parallelId,
    totalBranches,
    branchOutputs: new Map(),  // Map<branchIndex, outputs[]>
    completedCount: 0,
    totalExpectedNodes: totalBranches * terminalNodesCount
  }
  
  ctx.parallelExecutions.set(parallelId, scope)
  return scope
}
```

**Branch Completion Tracking:**

```typescript
// NEW: Track completion as branches finish
handleParallelBranchCompletion(
  ctx: ExecutionContext,
  parallelId: string,
  nodeId: string,  // e.g., "blockId<2>"
  output: NormalizedBlockOutput
): boolean {
  const scope = ctx.parallelExecutions.get(parallelId)
  const branchIndex = extractBranchIndex(nodeId)  // Extract 2 from "blockId<2>"
  
  // Initialize outputs array for this branch if needed
  if (!scope.branchOutputs.has(branchIndex)) {
    scope.branchOutputs.set(branchIndex, [])
  }
  
  // Add output to this branch's results
  scope.branchOutputs.get(branchIndex)!.push(output)
  scope.completedCount++
  
  // Check if all branches complete
  return scope.completedCount >= scope.totalExpectedNodes
}
```

**Result Aggregation:**

```typescript
// NEW: Aggregate results when all branches complete
aggregateParallelResults(ctx: ExecutionContext, parallelId: string): ParallelAggregationResult {
  const scope = ctx.parallelExecutions.get(parallelId)
  
  // Collect results in branch order
  const results: NormalizedBlockOutput[][] = []
  for (let i = 0; i < scope.totalBranches; i++) {
    const branchOutputs = scope.branchOutputs.get(i) || []
    results.push(branchOutputs)
  }
  
  // Store aggregated results
  this.state.setBlockOutput(parallelId, { results })
  
  return {
    allBranchesComplete: true,
    results,
    completedBranches: scope.totalBranches,
    totalBranches: scope.totalBranches
  }
}
```

**Branch Metadata Extraction:**

```typescript
// NEW: Extract parallel context from node ID
extractBranchMetadata(nodeId: string): ParallelBranchMetadata | null {
  const branchIndex = extractBranchIndex(nodeId)  // Parse <N> suffix
  if (branchIndex === null) return null
  
  const baseId = extractBaseBlockId(nodeId)  // Get blockId from "blockId<2>"
  const parallelId = this.findParallelIdForNode(baseId)
  if (!parallelId) return null
  
  const parallelConfig = this.dag.parallelConfigs.get(parallelId)
  const { totalBranches, distributionItem } = this.getParallelConfigInfo(
    parallelConfig,
    branchIndex
  )
  
  return {
    branchIndex,
    branchTotal: totalBranches,
    distributionItem,  // Current item for this branch
    parallelId
  }
}
```

**True Parallel Execution:**

```typescript
// NEW: Branches execute concurrently via ready queue
// When Block A completes, ALL entry nodes of parallel are added to queue

private async handleNodeCompletion(nodeId: string, output: any): Promise<void> {
  const node = this.dag.nodes.get(nodeId)
  
  // Get all newly ready nodes
  const readyNodes = this.edgeManager.processOutgoingEdges(node, output)
  
  // This might include: [BlockB<0>, BlockB<1>, BlockB<2>, BlockB<3>]
  // All are added to queue and execute CONCURRENTLY
  this.addMultipleToQueue(readyNodes)
}

private async processQueue(): Promise<void> {
  while (this.readyQueue.length > 0) {
    const nodeId = this.dequeue()
    
    // Launch async - don't await
    const promise = this.executeNodeAsync(nodeId)
    this.trackExecution(promise)
  }
  
  // Wait for at least one to complete
  if (this.executing.size > 0) {
    await this.waitForAnyExecution()
  }
}
```

**Advantages:**
- ✅ **True concurrent execution** - branches run in parallel via Promise.all
- ✅ **Simpler ID scheme** - `<N>` suffix vs long virtual IDs
- ✅ **No re-execution** - results aggregated as branches complete
- ✅ **Integrated routing** - branches are just DAG nodes with normal edge activation

---

## 5. Condition Evaluation

### Old System: Function-Based Evaluation

**Condition Handler:**

```typescript
// OLD: Condition handler evaluates and activates path
async execute(
  block: SerializedBlock,
  inputs: Record<string, any>,
  context: ExecutionContext
): Promise<BlockOutput> {
  // 1. Parse conditions JSON
  let conditions: Array<{ id: string; title: string; value: string }> = []
  conditions = JSON.parse(inputs.conditions || '[]')
  
  // 2. Build evaluation context
  const sourceBlockId = context.workflow?.connections.find(
    conn => conn.target === block.id
  )?.source
  const sourceOutput = context.blockStates.get(sourceBlockId)?.output
  
  const evalContext = {
    ...(typeof sourceOutput === 'object' ? sourceOutput : {}),
    ...(context.loopItems.get(block.id) || {})
  }
  
  // 3. Evaluate conditions in order
  let selectedCondition = null
  
  for (const condition of conditions) {
    if (condition.title === 'else') {
      selectedCondition = condition
      break
    }
    
    // Evaluate condition expression
    const conditionMet = await evaluateConditionExpression(
      condition.value,
      context,
      block,
      this.resolver,
      evalContext
    )
    
    if (conditionMet) {
      selectedCondition = condition
      break
    }
  }
  
  // 4. Store decision
  const decisionKey = context.currentVirtualBlockId || block.id
  context.decisions.condition.set(decisionKey, selectedCondition.id)
  
  // 5. Return output with selected path info
  return {
    ...sourceOutput,
    conditionResult: true,
    selectedPath: {
      blockId: targetBlock.id,
      blockType: targetBlock.metadata?.id,
      blockTitle: targetBlock.metadata?.name
    },
    selectedConditionId: selectedCondition.id
  }
}
```

**Condition Expression Evaluation:**

```typescript
// OLD: Evaluate condition using Function constructor
async function evaluateConditionExpression(
  conditionExpression: string,
  context: ExecutionContext,
  block: SerializedBlock,
  resolver: InputResolver,
  evalContext: Record<string, any>
): Promise<boolean> {
  // 1. Resolve all references first
  let resolvedConditionValue = conditionExpression
  
  // Resolve <variable.name> references
  resolvedConditionValue = resolver.resolveVariableReferences(resolvedConditionValue, block)
  
  // Resolve <blockId.output.path> references
  resolvedConditionValue = resolver.resolveBlockReferences(resolvedConditionValue, context, block)
  
  // Resolve ${ENV_VAR} references
  resolvedConditionValue = resolver.resolveEnvVariables(resolvedConditionValue)
  
  // 2. Evaluate the RESOLVED condition
  const conditionMet = new Function(
    'context',
    `with(context) { return ${resolvedConditionValue} }`
  )(evalContext)
  
  return Boolean(conditionMet)
}
```

**Problems:**
1. **Pre-resolution required**: All references resolved before evaluation
2. **Context building**: Manual assembly of evaluation context
3. **`with` statement**: Deprecated and problematic
4. **Active path management**: Manual addition to activeExecutionPath

### New System: Edge Handle Based Routing

**Condition Handler:**

```typescript
// NEW: Condition handler just determines which option selected
async execute(
  ctx: ExecutionContext,
  block: SerializedBlock,
  inputs: Record<string, any>
): Promise<BlockOutput> {
  // Parse conditions
  const conditions = JSON.parse(inputs.conditions || '[]')
  
  // Evaluate each condition
  let selectedCondition = null
  for (const condition of conditions) {
    if (condition.title === 'else') {
      selectedCondition = condition
      break
    }
    
    // Evaluate using resolver's integrated evaluation
    const conditionMet = this.evaluateCondition(ctx, block, condition.value)
    
    if (conditionMet) {
      selectedCondition = condition
      break
    }
  }
  
  // Return output with selectedOption
  return {
    selectedOption: selectedCondition.id,  // This is KEY - used by EdgeManager
    selectedPath: {
      blockId: targetBlock.id,
      blockType: targetBlock.metadata?.id,
      blockTitle: targetBlock.metadata?.name
    }
  }
}
```

**Edge Activation:**

```typescript
// NEW: EdgeManager automatically activates matching edge
private shouldActivateEdge(edge: DAGEdge, output: NormalizedBlockOutput): boolean {
  // Edge has sourceHandle = "condition-{conditionId}"
  if (edge.sourceHandle?.startsWith(EDGE.CONDITION_PREFIX)) {
    const conditionValue = edge.sourceHandle.substring(EDGE.CONDITION_PREFIX.length)
    
    // Compare with selectedOption from output
    return output.selectedOption === conditionValue
  }
  
  return true
}

// EdgeManager.processOutgoingEdges() will:
// 1. Check all outgoing edges from condition block
// 2. Only activate edge where sourceHandle matches output.selectedOption
// 3. Deactivate all other edges
// 4. Add target of activated edge to ready queue
```

**Edge Handle Generation:**

```typescript
// NEW: DAG builder generates condition edge handles
private generateSourceHandle(
  source: string,
  target: string,
  sourceHandle: string | undefined,
  metadata: EdgeMetadata,
  workflow: SerializedWorkflow
): string | undefined {
  if (!sourceHandle && isConditionBlockType(metadata.blockTypeMap.get(source))) {
    const conditions = metadata.conditionConfigMap.get(source)
    
    if (conditions && conditions.length > 0) {
      // Find which edge index this is
      const edgesFromCondition = workflow.connections.filter(c => c.source === source)
      const edgeIndex = edgesFromCondition.findIndex(e => e.target === target)
      
      // Match edge to corresponding condition
      if (edgeIndex >= 0 && edgeIndex < conditions.length) {
        const correspondingCondition = conditions[edgeIndex]
        return `${EDGE.CONDITION_PREFIX}${correspondingCondition.id}`
      }
    }
  }
  
  return sourceHandle
}
```

**Advantages:**
- ✅ **Declarative routing** - edges encode which condition they represent
- ✅ **No manual path activation** - EdgeManager handles it automatically
- ✅ **Automatic pruning** - unselected paths deactivated immediately
- ✅ **Consistent with routers** - same pattern for all routing blocks

---

## 6. Variable Resolution

### Old System: Flat Resolution with Manual Context

**InputResolver Structure:**

```typescript
// OLD: Single resolver with multiple resolution methods
class InputResolver {
  private blockById: Map<string, SerializedBlock>
  private blockByNormalizedName: Map<string, SerializedBlock>
  private loopsByBlockId: Map<string, string>
  private parallelsByBlockId: Map<string, string>
  
  resolveInputs(block: SerializedBlock, context: ExecutionContext): Record<string, any> {
    const inputs = block.config.params
    const result: Record<string, any> = {}
    
    for (const [key, value] of Object.entries(inputs)) {
      if (typeof value === 'string') {
        const trimmedValue = value.trim()
        
        // Check for direct variable reference: <variable.name>
        const directVariableMatch = trimmedValue.match(/^<variable\.([^>]+)>$/)
        if (directVariableMatch) {
          const variable = this.findVariableByName(directVariableMatch[1])
          result[key] = this.getTypedVariableValue(variable)
          continue
        }
        
        // Check for direct loop reference: <loop.property>
        const directLoopMatch = trimmedValue.match(/^<loop\.([^>]+)>$/)
        if (directLoopMatch) {
          const containingLoopId = this.loopsByBlockId.get(block.id)
          if (containingLoopId) {
            const pathParts = directLoopMatch[1].split('.')
            const loopValue = this.resolveLoopReference(
              containingLoopId,
              pathParts,
              context,
              block,
              false
            )
            result[key] = JSON.parse(loopValue)
            continue
          }
        }
        
        // Check for direct parallel reference: <parallel.property>
        const directParallelMatch = trimmedValue.match(/^<parallel\.([^>]+)>$/)
        if (directParallelMatch) {
          // Similar logic...
        }
        
        // Resolve general references
        let resolved = value
        resolved = this.resolveVariableReferences(resolved, block)
        resolved = this.resolveBlockReferences(resolved, context, block)
        resolved = this.resolveEnvVariables(resolved)
        result[key] = resolved
      } else {
        result[key] = value
      }
    }
    
    return result
  }
}
```

**Three-Pass Resolution:**

```typescript
// OLD: Separate resolution passes

// Pass 1: Variable references
resolveVariableReferences(template: string, block: SerializedBlock): string {
  return template.replace(/<variable\.([^>]+)>/g, (match, varPath) => {
    const variable = this.findVariableByName(varPath)
    if (variable) {
      return this.formatVariableValue(variable.value)
    }
    return match
  })
}

// Pass 2: Block references
resolveBlockReferences(template: string, context: ExecutionContext, block: SerializedBlock): string {
  return template.replace(/<([^>]+)>/g, (match, refPath) => {
    // Parse reference like "blockName.output.content"
    const parts = refPath.split('.')
    const blockRef = parts[0]
    
    // Find block by name or ID
    const referencedBlock = this.blockByNormalizedName.get(blockRef) || this.blockById.get(blockRef)
    if (!referencedBlock) return match
    
    // Check if block is in accessible blocks
    if (this.accessibleBlocksMap) {
      const currentAccessible = this.accessibleBlocksMap.get(block.id) || new Set()
      if (!currentAccessible.has(referencedBlock.id)) {
        return match // Block not accessible
      }
    }
    
    // Get block output
    const blockState = context.blockStates.get(referencedBlock.id)
    if (!blockState) return match
    
    // Navigate path
    let value = blockState.output
    for (let i = 1; i < parts.length; i++) {
      value = value?.[parts[i]]
    }
    
    return this.formatValue(value)
  })
}

// Pass 3: Environment variables
resolveEnvVariables(template: string): string {
  return template.replace(/\$\{([^}]+)\}/g, (match, varName) => {
    return this.environmentVariables[varName] || match
  })
}
```

**Loop Reference Resolution:**

```typescript
// OLD: Loop reference resolution
resolveLoopReference(
  loopId: string,
  pathParts: string[],
  context: ExecutionContext,
  block: SerializedBlock,
  withinTemplate: boolean
): string | null {
  const property = pathParts[0]
  
  switch (property) {
    case 'index':
      const index = this.loopManager?.getLoopIndex(loopId, block.id, context) || 0
      return String(index)
      
    case 'item':
      const currentItem = this.loopManager?.getCurrentItem(loopId, context)
      if (currentItem === undefined) return null
      
      // Navigate deeper path if needed
      if (pathParts.length > 1) {
        let value = currentItem
        for (let i = 1; i < pathParts.length; i++) {
          value = value?.[pathParts[i]]
        }
        return withinTemplate ? this.formatValue(value) : JSON.stringify(value)
      }
      
      return withinTemplate ? this.formatValue(currentItem) : JSON.stringify(currentItem)
      
    case 'currentIteration':
      const iteration = (context.loopIterations.get(loopId) || 1) - 1
      return String(iteration)
      
    default:
      return null
  }
}
```

**Problems:**
1. **Three separate passes**: Inefficient string replacement
2. **No scope hierarchy**: Variables, loops, parallels all at same level
3. **Manual loop context**: `loopsByBlockId` map required
4. **Direct vs template**: Different handling for direct references vs templates
5. **Format inconsistency**: JSON.stringify sometimes, toString other times

### New System: Layered Resolver Chain

**VariableResolver Architecture:**

```typescript
// NEW: Layered resolver with resolver chain
export class VariableResolver {
  private resolvers: Resolver[]
  
  constructor(
    workflow: SerializedWorkflow,
    workflowVariables: Record<string, any>,
    private state: ExecutionState
  ) {
    // Resolvers tried in order - inner scopes first
    this.resolvers = [
      new LoopResolver(workflow),        // <loop.iteration>, <loop.item>
      new ParallelResolver(workflow),    // <parallel.branchIndex>, <parallel.distributionItem>
      new WorkflowResolver(workflowVariables), // <workflow.variableName>
      new EnvResolver(),                 // ${ENV_VAR}
      new BlockResolver(workflow),       // <blockId.output.path>
    ]
  }
  
  resolveInputs(
    ctx: ExecutionContext,
    currentNodeId: string,
    params: Record<string, any>,
    block?: SerializedBlock
  ): Record<string, any> {
    const resolved: Record<string, any> = {}
    
    for (const [key, value] of Object.entries(params)) {
      resolved[key] = this.resolveValue(ctx, currentNodeId, value, undefined, block)
    }
    
    return resolved
  }
  
  private resolveValue(
    ctx: ExecutionContext,
    currentNodeId: string,
    value: any,
    loopScope?: LoopScope,
    block?: SerializedBlock
  ): any {
    // Handle primitives
    if (value === null || value === undefined) return value
    
    // Handle arrays recursively
    if (Array.isArray(value)) {
      return value.map(v => this.resolveValue(ctx, currentNodeId, v, loopScope, block))
    }
    
    // Handle objects recursively
    if (typeof value === 'object') {
      return Object.entries(value).reduce((acc, [k, v]) => ({
        ...acc,
        [k]: this.resolveValue(ctx, currentNodeId, v, loopScope, block)
      }), {})
    }
    
    // Handle strings with templates
    if (typeof value === 'string') {
      return this.resolveTemplate(ctx, currentNodeId, value, loopScope, block)
    }
    
    return value
  }
}
```

**Template Resolution:**

```typescript
// NEW: Single-pass template resolution
private resolveTemplate(
  ctx: ExecutionContext,
  currentNodeId: string,
  template: string,
  loopScope?: LoopScope,
  block?: SerializedBlock
): string {
  const resolutionContext: ResolutionContext = {
    executionContext: ctx,
    executionState: this.state,
    currentNodeId,
    loopScope
  }
  
  // Replace all <reference> patterns
  const referenceRegex = new RegExp(`<([^>]+)>`, 'g')
  
  let result = template.replace(referenceRegex, (match) => {
    const resolved = this.resolveReference(match, resolutionContext)
    
    if (resolved === undefined) return match
    
    // Format based on block type
    const blockType = block?.metadata?.id
    const isInTemplateLiteral = this.isInTemplateLiteral(template)
    
    return this.blockResolver.formatValueForBlock(resolved, blockType, isInTemplateLiteral)
  })
  
  // Also resolve environment variables
  const envRegex = new RegExp(`\\$\\{([^}]+)\\}`, 'g')
  result = result.replace(envRegex, (match) => {
    const resolved = this.resolveReference(match, resolutionContext)
    return typeof resolved === 'string' ? resolved : match
  })
  
  return result
}
```

**Resolver Chain:**

```typescript
// NEW: Try each resolver in order
private resolveReference(reference: string, context: ResolutionContext): any {
  for (const resolver of this.resolvers) {
    if (resolver.canResolve(reference)) {
      return resolver.resolve(reference, context)
    }
  }
  
  return undefined
}
```

**Individual Resolvers:**

```typescript
// NEW: LoopResolver
export class LoopResolver implements Resolver {
  canResolve(reference: string): boolean {
    return /^<loop\./i.test(reference)
  }
  
  resolve(reference: string, context: ResolutionContext): any {
    // Extract property: <loop.iteration> => "iteration"
    const match = reference.match(/<loop\.([^>]+)>/i)
    if (!match) return undefined
    
    const pathParts = match[1].split('.')
    const property = pathParts[0]
    
    // Get loop scope from context
    const loopScope = context.loopScope || this.findLoopScopeForNode(context)
    if (!loopScope) return undefined
    
    switch (property) {
      case 'iteration':
        return loopScope.iteration
        
      case 'index':
        return loopScope.iteration
        
      case 'item':
        if (pathParts.length === 1) {
          return loopScope.item
        }
        // Navigate deeper: <loop.item.name>
        return this.navigatePath(loopScope.item, pathParts.slice(1))
        
      default:
        return undefined
    }
  }
}

// NEW: ParallelResolver
export class ParallelResolver implements Resolver {
  canResolve(reference: string): boolean {
    return /^<parallel\./i.test(reference)
  }
  
  resolve(reference: string, context: ResolutionContext): any {
    const match = reference.match(/<parallel\.([^>]+)>/i)
    if (!match) return undefined
    
    const property = match[1]
    
    // Extract parallel metadata from current node ID
    const parallelMetadata = this.extractParallelMetadata(context.currentNodeId)
    if (!parallelMetadata) return undefined
    
    switch (property) {
      case 'branchIndex':
        return parallelMetadata.branchIndex
        
      case 'distributionItem':
        return parallelMetadata.distributionItem
        
      default:
        return undefined
    }
  }
}

// NEW: BlockResolver
export class BlockResolver implements Resolver {
  canResolve(reference: string): boolean {
    return /^<[^>]+>$/.test(reference) && !reference.startsWith('<loop.') && !reference.startsWith('<parallel.')
  }
  
  resolve(reference: string, context: ResolutionContext): any {
    // Parse: <blockId.output.content> => ["blockId", "output", "content"]
    const parts = reference.slice(1, -1).split('.')
    const blockRef = parts[0]
    
    // Find block by ID or normalized name
    const block = this.findBlock(blockRef)
    if (!block) return undefined
    
    // Get block state
    const blockState = context.executionState.getBlockStates().get(block.id)
    if (!blockState) return undefined
    
    // Navigate output path
    let value = blockState.output
    for (let i = 1; i < parts.length; i++) {
      value = value?.[parts[i]]
    }
    
    return value
  }
  
  formatValueForBlock(value: any, blockType?: string, isInTemplateLiteral?: boolean): string {
    // Special formatting based on block type
    if (blockType === BlockType.FUNCTION && isInTemplateLiteral) {
      // Inside template literal - preserve type
      if (typeof value === 'string') {
        return value.replace(/`/g, '\\`')
      }
      return String(value)
    }
    
    if (blockType === BlockType.CONDITION) {
      // Condition blocks need proper quotes
      if (typeof value === 'string') {
        return `'${value.replace(/'/g, "\\'")}'`
      }
      if (typeof value === 'object') {
        return JSON.stringify(value)
      }
      return String(value)
    }
    
    // Default formatting
    if (typeof value === 'string') {
      return value
    }
    if (typeof value === 'object') {
      return JSON.stringify(value)
    }
    return String(value)
  }
}
```

**Advantages:**
- ✅ **Single-pass resolution** - one regex replace handles all references
- ✅ **Proper scope hierarchy** - loop/parallel variables shadow outer scopes
- ✅ **Type-aware formatting** - different blocks format differently
- ✅ **Extensible** - easy to add new resolver types
- ✅ **Context-aware** - loop scope passed through resolution

---

## 7. Trigger Handling

### Old System: Manual Trigger Discovery

**Trigger Validation:**

```typescript
// OLD: Check for trigger blocks during validation
private validateWorkflow(startBlockId?: string): void {
  if (startBlockId) {
    const startBlock = this.workflow.blocks.find(block => block.id === startBlockId)
    if (!startBlock || !startBlock.enabled) {
      throw new Error(`Start block ${startBlockId} not found or disabled`)
    }
    return
  }
  
  // Check for trigger blocks
  const hasTriggerBlocks = this.workflow.blocks.some(block => {
    // Dedicated trigger blocks
    if (block.metadata?.category === 'triggers') return true
    // Blocks with trigger mode enabled
    if (block.config?.params?.triggerMode === true) return true
    return false
  })
  
  if (!hasTriggerBlocks) {
    // Legacy: require starter block
    const starterBlock = this.workflow.blocks.find(
      block => block.metadata?.id === BlockType.STARTER
    )
    if (!starterBlock || !starterBlock.enabled) {
      throw new Error('Workflow must have an enabled starter block')
    }
  }
}
```

**Execution Start:**

```typescript
// OLD: Create execution context with manual start block
private createExecutionContext(workflowId: string, startTime: Date, startBlockId?: string): ExecutionContext {
  const context = {
    workflowId,
    blockStates: new Map(Object.entries(this.initialBlockStates)),
    executedBlocks: new Set(),
    blockLogs: [],
    metadata: { startTime: startTime.toISOString(), duration: 0 },
    environmentVariables: this.environmentVariables,
    workflowVariables: this.workflowVariables,
    decisions: { router: new Map(), condition: new Map() },
    completedLoops: new Set(),
    loopExecutions: new Map(),
    parallelExecutions: new Map(),
    activeExecutionPath: new Set(),
    workflow: this.workflow,
    loopIterations: new Map(),
    loopItems: new Map()
  }
  
  // Find start block and add to active path
  let startBlock: SerializedBlock | undefined
  
  if (startBlockId) {
    startBlock = this.workflow.blocks.find(b => b.id === startBlockId)
  } else {
    startBlock = this.workflow.blocks.find(
      b => b.metadata?.id === BlockType.STARTER
    )
  }
  
  if (startBlock) {
    context.activeExecutionPath.add(startBlock.id)
    
    // Set initial output for start block
    context.blockStates.set(startBlock.id, {
      output: this.workflowInput,
      executed: false,
      executionTime: 0
    })
  }
  
  return context
}
```

**Problems:**
1. **Manual path activation**: Must explicitly add start block to activeExecutionPath
2. **No trigger types**: All triggers treated the same
3. **Hardcoded starter block**: Special case for BlockType.STARTER

### New System: DAG Initialization with Trigger Resolution

**Start Block Resolution:**

```typescript
// NEW: Resolve start block using StartBlockPath utilities
private initializeStarterBlock(
  context: ExecutionContext,
  state: ExecutionState,
  triggerBlockId?: string
): void {
  let startResolution: ReturnType<typeof resolveExecutorStartBlock> | null = null
  
  if (triggerBlockId) {
    const triggerBlock = this.workflow.blocks.find(b => b.id === triggerBlockId)
    if (!triggerBlock) {
      throw new Error(`Trigger block not found: ${triggerBlockId}`)
    }
    
    startResolution = buildResolutionFromBlock(triggerBlock)
    
    if (!startResolution) {
      startResolution = {
        blockId: triggerBlock.id,
        block: triggerBlock,
        path: StartBlockPath.SPLIT_MANUAL
      }
    }
  } else {
    startResolution = resolveExecutorStartBlock(this.workflow.blocks, {
      execution: 'manual',
      isChildWorkflow: false
    })
    
    if (!startResolution?.block) {
      logger.warn('No start block found in workflow')
      return
    }
  }
  
  // Skip if already has state (resume scenario)
  if (state.getBlockStates().has(startResolution.block.id)) {
    return
  }
  
  // Build appropriate output based on trigger type
  const blockOutput = buildStartBlockOutput({
    resolution: startResolution,
    workflowInput: this.workflowInput,
    isDeployedExecution: this.contextExtensions?.isDeployedContext === true
  })
  
  state.setBlockState(startResolution.block.id, {
    output: blockOutput,
    executed: false,
    executionTime: 0
  })
}
```

**Trigger Path Types:**

```typescript
// NEW: Explicit trigger path types
export enum StartBlockPath {
  // Webhooks
  WEBHOOK_CATCH = 'webhook-catch',
  WEBHOOK_CHAT = 'webhook-chat',
  
  // Schedule triggers
  SCHEDULE_CRON = 'schedule-cron',
  SCHEDULE_INTERVAL = 'schedule-interval',
  
  // Manual triggers
  SPLIT_MANUAL = 'split-manual',
  STARTER = 'starter',
  
  // Email triggers
  EMAIL_RECEIVE = 'email-receive',
  
  // Special
  WORKFLOW_TRIGGER = 'workflow-trigger'
}

export function resolveExecutorStartBlock(
  blocks: SerializedBlock[],
  options: {
    execution: 'manual' | 'webhook' | 'schedule' | 'email'
    webhookPath?: string
    scheduleName?: string
    isChildWorkflow?: boolean
  }
): StartBlockResolution | null {
  // For child workflows, use workflow trigger
  if (options.isChildWorkflow) {
    const workflowTrigger = blocks.find(
      b => b.metadata?.id === BlockType.WORKFLOW_TRIGGER
    )
    if (workflowTrigger) {
      return {
        blockId: workflowTrigger.id,
        block: workflowTrigger,
        path: StartBlockPath.WORKFLOW_TRIGGER
      }
    }
  }
  
  // For webhooks, find matching webhook catch block
  if (options.execution === 'webhook' && options.webhookPath) {
    const webhookCatch = blocks.find(
      b => b.metadata?.id === BlockType.WEBHOOK_CATCH &&
           b.config.params?.path === options.webhookPath
    )
    if (webhookCatch) {
      return {
        blockId: webhookCatch.id,
        block: webhookCatch,
        path: StartBlockPath.WEBHOOK_CATCH
      }
    }
  }
  
  // For schedules, find matching schedule block
  if (options.execution === 'schedule' && options.scheduleName) {
    const scheduleBlock = blocks.find(
      b => (b.metadata?.id === BlockType.SCHEDULE_CRON ||
            b.metadata?.id === BlockType.SCHEDULE_INTERVAL) &&
           b.metadata?.name === options.scheduleName
    )
    if (scheduleBlock) {
      return {
        blockId: scheduleBlock.id,
        block: scheduleBlock,
        path: scheduleBlock.metadata.id === BlockType.SCHEDULE_CRON
          ? StartBlockPath.SCHEDULE_CRON
          : StartBlockPath.SCHEDULE_INTERVAL
      }
    }
  }
  
  // Default: find starter block
  const starter = blocks.find(b => b.metadata?.id === BlockType.STARTER)
  if (starter) {
    return {
      blockId: starter.id,
      block: starter,
      path: StartBlockPath.STARTER
    }
  }
  
  return null
}
```

**Queue Initialization:**

```typescript
// NEW: Initialize ready queue from start block
private initializeQueue(triggerBlockId?: string): void {
  const pendingBlocks = this.context.metadata.pendingBlocks
  
  // Resume scenario
  if (pendingBlocks && pendingBlocks.length > 0) {
    for (const nodeId of pendingBlocks) {
      this.addToQueue(nodeId)
    }
    this.context.metadata.pendingBlocks = []
    return
  }
  
  // Normal start: add trigger block to queue
  if (triggerBlockId) {
    this.addToQueue(triggerBlockId)
    return
  }
  
  // Find start node in DAG
  const startNode = Array.from(this.dag.nodes.values()).find(
    node => node.block.metadata?.id === BlockType.START_TRIGGER ||
            node.block.metadata?.id === BlockType.STARTER
  )
  
  if (startNode) {
    this.addToQueue(startNode.id)
  } else {
    logger.warn('No start node found in DAG')
  }
}
```

**Advantages:**
- ✅ **Type-safe trigger paths**: Enum for all trigger types
- ✅ **Automatic start resolution**: No manual activeExecutionPath management
- ✅ **Context-aware outputs**: Different output structure per trigger type
- ✅ **Resume support**: Handles pending blocks for pause/resume

---

## 8. Edge Construction & Routing

### Old System: Connection-Based Routing

**Connections as Source of Truth:**

```typescript
// OLD: Workflow uses flat connection array
interface SerializedWorkflow {
  blocks: SerializedBlock[]
  connections: SerializedConnection[]  // Array of { source, target, sourceHandle?, targetHandle? }
  loops?: Record<string, SerializedLoop>
  parallels?: Record<string, SerializedParallel>
}

interface SerializedConnection {
  source: string
  target: string
  sourceHandle?: string  // e.g., "loop-start-source", "condition-abc123"
  targetHandle?: string
}
```

**Dependency Checking:**

```typescript
// OLD: Check dependencies by scanning connections
private checkDependencies(
  incomingConnections: SerializedConnection[],
  executedBlocks: Set<string>,
  context: ExecutionContext
): boolean {
  return incomingConnections.every(conn => {
    const sourceExecuted = executedBlocks.has(conn.source)
    
    // Special handling for different source handles
    if (conn.sourceHandle === 'error') {
      const hasError = context.blockStates.get(conn.source)?.output?.error
      return sourceExecuted && hasError
    }
    
    if (conn.sourceHandle?.startsWith('condition-')) {
      const conditionId = conn.sourceHandle.substring('condition-'.length)
      const selectedCondition = context.decisions.condition.get(conn.source)
      return sourceExecuted && selectedCondition === conditionId
    }
    
    if (conn.sourceHandle === 'loop-start-source') {
      // Loop block executed and not complete
      return sourceExecuted && !context.completedLoops.has(conn.source)
    }
    
    return sourceExecuted
  })
}
```

**Problems:**
1. **Repeated connection scanning**: Every block scans all connections
2. **No edge deactivation**: Unselected paths still in connection array
3. **Manual handle interpretation**: Special cases for each handle type
4. **No explicit DAG**: Topology implicit in connections

### New System: Explicit DAG with Edge Metadata

**DAG Structure:**

```typescript
// NEW: Explicit DAG with nodes and edges
export interface DAGNode {
  id: string
  block: SerializedBlock
  incomingEdges: Set<string>      // Set of source node IDs
  outgoingEdges: Map<string, DAGEdge>  // Map of edgeId -> edge data
  metadata: NodeMetadata
}

export interface DAGEdge {
  target: string
  sourceHandle?: string
  targetHandle?: string
  isActive?: boolean  // Undefined means active, false means deactivated
}

export interface DAG {
  nodes: Map<string, DAGNode>
  loopConfigs: Map<string, SerializedLoop>
  parallelConfigs: Map<string, SerializedParallel>
}
```

**Edge Construction Process:**

```typescript
// NEW: EdgeConstructor builds DAG edges
export class EdgeConstructor {
  execute(
    workflow: SerializedWorkflow,
    dag: DAG,
    blocksInParallels: Set<string>,
    blocksInLoops: Set<string>,
    reachableBlocks: Set<string>,
    pauseTriggerMapping: Map<string, string>
  ): void {
    // 1. Wire regular edges (block-to-block)
    this.wireRegularEdges(
      workflow,
      dag,
      blocksInParallels,
      blocksInLoops,
      reachableBlocks,
      loopBlockIds,
      parallelBlockIds,
      metadata,
      pauseTriggerMapping
    )
    
    // 2. Wire loop sentinel nodes
    this.wireLoopSentinels(dag, reachableBlocks)
    
    // 3. Wire parallel blocks (expand branches)
    this.wireParallelBlocks(workflow, dag, loopBlockIds, parallelBlockIds, pauseTriggerMapping)
  }
}
```

**Regular Edge Wiring:**

```typescript
// NEW: Wire edges with handle generation
private wireRegularEdges(
  workflow: SerializedWorkflow,
  dag: DAG,
  blocksInParallels: Set<string>,
  blocksInLoops: Set<string>,
  reachableBlocks: Set<string>,
  loopBlockIds: Set<string>,
  parallelBlockIds: Set<string>,
  metadata: EdgeMetadata,
  pauseTriggerMapping: Map<string, string>
): void {
  for (const connection of workflow.connections) {
    let { source, target } = connection
    
    // Generate sourceHandle if not provided (for conditions/routers)
    let sourceHandle = this.generateSourceHandle(
      source,
      target,
      connection.sourceHandle,
      metadata,
      workflow
    )
    
    // Transform loop block connections to sentinels
    if (loopBlockIds.has(source)) {
      source = buildSentinelEndId(source)
      sourceHandle = EDGE.LOOP_EXIT
    }
    if (loopBlockIds.has(target)) {
      target = buildSentinelStartId(target)
    }
    
    // Skip if edge crosses loop boundary
    if (this.edgeCrossesLoopBoundary(source, target, blocksInLoops, dag)) {
      continue
    }
    
    // Wire the edge
    this.addEdge(dag, source, target, sourceHandle, connection.targetHandle)
  }
}
```

**Condition Handle Generation:**

```typescript
// NEW: Generate condition edge handles automatically
private generateSourceHandle(
  source: string,
  target: string,
  sourceHandle: string | undefined,
  metadata: EdgeMetadata,
  workflow: SerializedWorkflow
): string | undefined {
  // If handle already set, use it
  if (sourceHandle) return sourceHandle
  
  // For condition blocks without handle, generate from condition config
  if (isConditionBlockType(metadata.blockTypeMap.get(source))) {
    const conditions = metadata.conditionConfigMap.get(source)
    
    if (conditions && conditions.length > 0) {
      // Find edge index
      const edgesFromCondition = workflow.connections.filter(c => c.source === source)
      const edgeIndex = edgesFromCondition.findIndex(e => e.target === target)
      
      // Match to condition by index
      if (edgeIndex >= 0 && edgeIndex < conditions.length) {
        const correspondingCondition = conditions[edgeIndex]
        return `${EDGE.CONDITION_PREFIX}${correspondingCondition.id}`
      }
    }
  }
  
  // For router blocks, generate from target
  if (metadata.routerBlockIds.has(source)) {
    return `${EDGE.ROUTER_PREFIX}${target}`
  }
  
  return undefined
}
```

**Loop Sentinel Wiring:**

```typescript
// NEW: Wire loop sentinels with continue/exit edges
private wireLoopSentinels(dag: DAG, reachableBlocks: Set<string>): void {
  for (const [loopId, loopConfig] of dag.loopConfigs) {
    const sentinelStartId = buildSentinelStartId(loopId)
    const sentinelEndId = buildSentinelEndId(loopId)
    
    // Find entry and terminal nodes in loop
    const { startNodes, terminalNodes } = this.findLoopBoundaryNodes(
      loopConfig.nodes,
      dag,
      reachableBlocks
    )
    
    // Wire sentinel start -> entry nodes
    for (const startNodeId of startNodes) {
      this.addEdge(dag, sentinelStartId, startNodeId)
    }
    
    // Wire terminal nodes -> sentinel end
    for (const terminalNodeId of terminalNodes) {
      this.addEdge(dag, terminalNodeId, sentinelEndId)
    }
    
    // Wire backward edge: sentinel end -> sentinel start (LOOP_CONTINUE)
    this.addEdge(
      dag,
      sentinelEndId,
      sentinelStartId,
      EDGE.LOOP_CONTINUE,
      undefined,
      true  // isLoopBackEdge - not added to incomingEdges initially
    )
  }
}
```

**Parallel Branch Expansion:**

```typescript
// NEW: Expand parallel blocks into branch nodes
private wireParallelBlocks(
  workflow: SerializedWorkflow,
  dag: DAG,
  loopBlockIds: Set<string>,
  parallelBlockIds: Set<string>,
  pauseTriggerMapping: Map<string, string>
): void {
  for (const [parallelId, parallelConfig] of dag.parallelConfigs) {
    const { entryNodes, terminalNodes, branchCount } = this.findParallelBoundaryNodes(
      parallelConfig.nodes,
      parallelId,
      dag
    )
    
    // Wire edges TO parallel block (expand to all branches)
    for (const connection of workflow.connections) {
      if (connection.target === parallelId) {
        for (const entryNodeId of entryNodes) {
          for (let i = 0; i < branchCount; i++) {
            const branchNodeId = buildBranchNodeId(entryNodeId, i)
            if (dag.nodes.has(branchNodeId)) {
              this.addEdge(
                dag,
                connection.source,
                branchNodeId,
                connection.sourceHandle,
                connection.targetHandle
              )
            }
          }
        }
      }
      
      // Wire edges FROM parallel block (collect from all branches)
      if (connection.source === parallelId) {
        for (const terminalNodeId of terminalNodes) {
          for (let i = 0; i < branchCount; i++) {
            const branchNodeId = buildBranchNodeId(terminalNodeId, i)
            if (dag.nodes.has(branchNodeId)) {
              this.addEdge(
                dag,
                branchNodeId,
                connection.target,
                connection.sourceHandle,
                connection.targetHandle
              )
            }
          }
        }
      }
    }
  }
}
```

**Edge Activation/Deactivation:**

```typescript
// NEW: Activate and deactivate edges during execution
processOutgoingEdges(
  node: DAGNode,
  output: NormalizedBlockOutput,
  skipBackwardsEdge = false
): string[] {
  const readyNodes: string[] = []
  
  for (const [edgeId, edge] of node.outgoingEdges) {
    // Skip backward loop edges if requested
    if (skipBackwardsEdge && this.isBackwardsEdge(edge.sourceHandle)) {
      continue
    }
    
    // Determine if edge should activate
    const shouldActivate = this.shouldActivateEdge(edge, output)
    
    if (!shouldActivate) {
      const isLoopEdge = edge.sourceHandle === EDGE.LOOP_CONTINUE ||
                         edge.sourceHandle === EDGE.LOOP_EXIT
      
      // Deactivate edge and descendants (unless it's a loop edge)
      if (!isLoopEdge) {
        this.deactivateEdgeAndDescendants(node.id, edge.target, edge.sourceHandle)
      }
      
      continue
    }
    
    // Remove this edge from target's incoming edges
    const targetNode = this.dag.nodes.get(edge.target)
    targetNode.incomingEdges.delete(node.id)
    
    // Check if target is now ready
    if (this.isNodeReady(targetNode)) {
      readyNodes.push(targetNode.id)
    }
  }
  
  return readyNodes
}

// Recursively deactivate unreachable paths
private deactivateEdgeAndDescendants(
  sourceId: string,
  targetId: string,
  sourceHandle?: string
): void {
  const edgeKey = this.createEdgeKey(sourceId, targetId, sourceHandle)
  
  if (this.deactivatedEdges.has(edgeKey)) {
    return  // Already deactivated
  }
  
  this.deactivatedEdges.add(edgeKey)
  
  const targetNode = this.dag.nodes.get(targetId)
  if (!targetNode) return
  
  // If target has other active incoming edges, don't deactivate descendants
  const hasOtherActiveIncoming = this.hasActiveIncomingEdges(targetNode, sourceId)
  if (hasOtherActiveIncoming) {
    return
  }
  
  // Recursively deactivate all outgoing edges
  for (const [_, outgoingEdge] of targetNode.outgoingEdges) {
    this.deactivateEdgeAndDescendants(targetId, outgoingEdge.target, outgoingEdge.sourceHandle)
  }
}
```

**Advantages:**
- ✅ **Explicit DAG structure** - topology is first-class data
- ✅ **O(1) edge operations** - Set/Map operations
- ✅ **Automatic pruning** - unreachable paths deactivated immediately
- ✅ **Handle generation** - conditions/routers get handles automatically
- ✅ **Branch expansion** - parallels expanded during DAG construction
- ✅ **Sentinel nodes** - loops have explicit start/end nodes

---

## Summary of Key Transformations

| Aspect | Old (Layer-Based) | New (DAG-Based) |
|--------|-------------------|-----------------|
| **Execution Model** | Sequential layers with post-processing | Concurrent ready queue |
| **Parallelism** | Virtual blocks, still sequential in layer | True parallel via Promise.all |
| **Loop Iteration** | Post-layer reachability analysis + reset | Sentinel nodes with edge-based routing |
| **Dependencies** | Repeated connection scanning | DAG edge removal, O(1) ready check |
| **Condition Routing** | Function evaluation + manual path activation | Edge handles + automatic activation |
| **Variable Resolution** | Three-pass string replacement | Single-pass layered resolver chain |
| **Trigger Handling** | Manual activeExecutionPath addition | Type-safe start resolution + queue init |
| **Edge Management** | Implicit in connections array | Explicit DAG edges with activation state |

The new architecture is fundamentally **declarative** and **event-driven**, where the old architecture was **imperative** and **polling-based**. This enables true parallelism, cleaner code, and better state management.

