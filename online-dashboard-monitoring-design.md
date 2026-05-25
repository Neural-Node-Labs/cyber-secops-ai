# Component-Based Development (CBD) Design: Online Monitoring Dashboard

**System Standard:** CBD UI-First Standard

**Document Identifier:** `online_ui_monitor_design.md`

**Deployment Target:** Web-Based Secure Operations Console

**Type Safety Baseline:** TypeScript 5.x Strict Mode

---

## 1. Component Tree Architecture & Hierarchy

Following CBD core principles, the user interface is structured as a collection of self-contained, decoupled, hierarchical black boxes. Components communicate exclusively via explicit data properties (`IN`) and event emitters (`OUT`), ensuring they remain completely agnostic of each other's internal layouts.

```
DashboardLayout (Root Workspace)
 ├── NavigationHeaderPanel (System Level Status, Global Kill-Switch)
 ├── WorkspaceGrid (Responsive Visual Grid Engine)
 │    ├── TelemetryMetricSection (High-Level CPU, Memory, IPC Monitors)
 │    ├── ActivityProcessingSection (Real-Time Daemon Activity Matrix)
 │    └── CoreReasoningSection (Unified Task / Sub-Task / ReAct Engine Map)
 └── DynamicExecutionSection (Swarm Orchestrator Grid & Cluster Node Map)

```

---

## 2. No-Code Blueprint UI Component Registry

The registry below represents the **UI Source of Truth**. Each block defines a specific monitoring requirement mapped to its atomic interface boundary.

| Visual Component Name | Logical Function | Component Boundaries / Sub-Trees | Expected Inbound State (IN) | Dispatched Outbound Events (OUT) |
| --- | --- | --- | --- | --- |
| `METRIC_GAUGE_PANEL` | Real-time hardware consumption tracking | `CpuGauge`, `MemoryGauge`, `ConnectionAlert` | `cpuLoad`, `memLoad`, `ipcStatus` | `onThresholdBreached` |
| `DAEMON_ACTIVITY_LOG` | Low-level processing stream visualizer | `LogStreamWindow`, `SinkFilterSelector` | `logEntries[]`, `activeSinkType` | `onFilterChanged`, `onLogExport` |
| `HIERARCHY_TREE_VIEW` | Visual tree mapping Core Commands down to Sub-Tasks | `GoalNode`, `TaskNode`, `SubTaskNode` | `activeGoal`, `runningTasks[]` | `onNodeSelected`, `onTaskAbort` |
| `REACT_CONSOLE_STREAM` | Incremental reasoning loop telemetry feed | `ThoughtBlock`, `ActionBlock`, `ObservationBlock` | `currentStep`, `executionTimeMs` | `onLoopPaused` |
| `SWARM_GRID_MONITOR` | Distributed cluster sub-agent resource mapping | `AgentNodeCard`, `ResourceClusterBadge` | `activeAgents[]`, `clusterHealth` | `onAgentTerminate`, `onAgentSpawn` |

---

## 3. TypeScript Interface Contracts (`/src/types/ui-contracts.ts`)

These definitions provide a type-safe contract framework ensuring absolute parameter validation between the backend gateway stream and the rendering DOM nodes.

```typescript
/**
 * UI Component-Based Development Contract Definitions
 * Strict Structural Typing Enforced
 */

export type NodeStatus = 'IDLE' | 'EXECUTING' | 'SUCCESS' | 'FAULT_TRIPPED' | 'QUARANTINE';
export type ComponentViewPhase = 'GOAL' | 'TASK' | 'SUB_TASK' | 'REACT_LOOP';

/**
 * 1. Interface for HIERARCHY_TREE_VIEW (Tracking Goal -> Task -> Sub-Task)
 */
export interface ITaskHierarchyNode {
  readonly id: string;
  readonly name: string;
  readonly type: ComponentViewPhase;
  readonly description: string;
  readonly status: NodeStatus;
  readonly executionTimeMs: number;
  readonly children?: readonly ITaskHierarchyNode[];
}

export interface IHierarchyTreeProps {
  readonly activeHierarchyRoot: ITaskHierarchyNode;
  readonly selectedNodeId?: string;
}

export interface IHierarchyTreeEmitters {
  readonly onNodeSelect: (nodeId: string) => void;
  readonly onTerminateSignal: (nodeId: string, authSignature: string) => void;
}

/**
 * 2. Interface for SWARM_GRID_MONITOR (Tracking Swarm Clustering & Memory Spaces)
 */
export interface ISwarmAgentData {
  readonly agentId: string;
  readonly alias: string;
  readonly assignmentSubTaskId: string;
  readonly isolatedMemoryRange: {
    readonly startAddress: string;
    readonly endAddress: string;
    readonly allocatedBytes: number;
  };
  readonly securityStatus: NodeStatus;
  readonly dynamicSkillsInjected: readonly string[];
}

export interface ISwarmGridProps {
  readonly swarmClusterName: string;
  readonly agents: readonly ISwarmAgentData[];
  readonly totalClusterMemoryAllocatedBytes: number;
}

export interface ISwarmGridEmitters {
  readonly onIsolateAgentMemory: (agentId: string) => void;
  readonly onInjectSkillPayload: (agentId: string, skillCodeSignature: string) => void;
}

/**
 * 3. Interface for REACT_CONSOLE_STREAM (The Thought-Action-Observation Viewer)
 */
export interface IReActIterationState {
  readonly timestamp: string;
  readonly iterationIndex: number;
  readonly associatedTaskId: string;
  readonly thoughts: readonly string[];
  readonly plannedAction: string;
  readonly resultingObservation?: string;
  readonly policyApproved: boolean;
}

export interface IReActConsoleProps {
  readonly activeLoopState: IReActIterationState;
  readonly isThrottled: boolean;
}

```

---

## 4. UI Stability, Fail-Secure Operations, and Throttling Strategy

Because the backend daemon operates at low-level machine execution speeds, a high-throughput breach mitigation event could flood the web frontend with thousands of state changes per second. To prevent UI rendering degradation, the architecture enforces these non-negotiable boundaries:

1. **Decoupled State Slicing (Data Virtualization):** Components are completely forbidden from mutating shared runtime state objects directly. Incoming telemetry inputs must pass through a strict immutable projection step (using deep-freeze copy patterns) before updating views.
2. **Dynamic Canvas Render Throttling:** The interface uses an internal frame buffer gateway. If incoming websocket packets exceed a frequency threshold of 30 items per second, the state collector dynamically collapses intermediate updates into single consolidated delta frames, shielding browser process threads from visual bottlenecks.
3. **Fail-Secure System Disconnect Rendering:** If the communication transport channel fails, the dashboard must instantly trigger an isolated fail-secure fallback loop:
* The user view layers must immediately switch to a read-only frozen mode.
* An immutable overlay element must display: `[LINK_TERMINATED: RUNTIME_DAEMON_STATE_FROZEN]`.
* Inter-component communication must be blocked from executing user override actions until cryptographic re-authentication succeeds.



---

### Phase I Gate Verification Check

The online dashboard structural interfaces and type definitions are finalized.

