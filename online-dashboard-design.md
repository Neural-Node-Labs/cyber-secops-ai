# Final Specification Design: Cyber-SecOps Web Dashboard Backend & API Interface

**System Framework:** Component-Based Development (CBD) Interface-First Standard

**Document Identifier:** `dashboard_system_design.md`

**Status:** Approved Architecture Specification

**Language Baseline:** TypeScript 5.x Strict Mode (Type-Safe Interface Definitions Only)

---

## 1. Architectural Topology & Interface Contract Flow

The Cyber-SecOps Web Dashboard functions as an isolated single-page application (SPA) plane. To preserve absolute system isolation, it never communicates directly with the kernel or daemon internals. It operates entirely by establishing a bidirectional connection to the daemon via a type-safe API Gateway layer.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        DAEMON PROCESS PLANE                            │
│                                                                        │
│   ┌─────────────────────┐                    ┌─────────────────────┐   │
│   │    REACT_ENGINE     │                    │ DECISION_VALIDATOR  │   │
│   └──────────┬──────────┘                    └──────────┬──────────┘   │
└──────────────┼──────────────────────────────────────────┼──────────────┘
               │ (Internal Inter-Process Events)         │
               ▼                                          ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY / ADAPTOR PLANE                     │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                      DAEMON BACKEND ADAPTOR                     │   │
│   │            (Translates Internal Telemetry into JSON)           │   │
│   └──────────────────────────────┬─────────────────────────────────┘   │
└──────────────────────────────────┼─────────────────────────────────────┘
                                   │
               ┌───────────────────┴───────────────────┐
               ▼ (Secure Transport)                    ▼ (Action Controls)
       [HTTPS REST Endpoints]                 [Secure WebSockets (WSS)]
               │                                       │
┌──────────────┼───────────────────────────────────────┼─────────────────┐
│              ▼                                       ▼                 │
│   ┌─────────────────────┐                    ┌─────────────────────┐   │
│   │    REST_CLIENT      │                    │    WS_STREAM_BUS    │   │
│   │ (Historical / Auth) │                    │ (Real-Time State)   │   │
│   └──────────┬──────────┘                    └──────────┬──────────┘   │
│              │                                       │                 │
│              └───────────────────┬───────────────────┘                 │
│                                  ▼                                     │
│                     ┌─────────────────────────┐                        │
│                     │   DASHBOARD RECOUPLED   │                        │
│                     │    STATE STORE LAYER    │                        │
│                     └────────────┬────────────┘                        │
│                                  ▼                                     │
│                     ┌─────────────────────────┐                        │
│                     │  TYPESCRIPT UI ENGINE   │                        │
│                     │  (Component Hierarchy)  │                        │
│                     └─────────────────────────┘                        │
│                        DASHBOARD FRONTEND PLANE                        │
└────────────────────────────────────────────────────────────────────────┘

```

---

## 2. No-Code System Registry & Component Tree

Following the strict CBD principle, the dashboard front-end map is broken into structural, independent layout modules. No UI library definitions are mixed into this blueprint core.

```
DashboardRoot
 ├── MetricCardPanel (Sub-components: CPUCard, MemoryCard, ConnectionStatus)
 ├── ReactStreamConsole (Sub-components: LoopStepper, ReasoningTerminal)
 ├── SwarmOrchestratorGrid (Sub-components: AgentNodeCard, DeploymentControl)
 └── CircuitBreakerConsole (Sub-components: PolicyViolationAlert, ActionOverrideForm)

```

---

## 3. Type-Safe Core Interfaces (`/src/types/daemon-api.ts`)

This file represents the definitive interface contract library. It guarantees strict parameter alignment across backend emitters and front-end consumers.

```typescript
/**
 * Cyber-SecOps Daemon API & Dashboard Data Contract Definitions
 * Version: 1.0.0
 * Strict Mode Constraints Enforced
 */

export type ProviderType = 'CLOUD_LLM' | 'LOCAL_LLAMA' | 'RULE_BASED_MATRIX';
export type AgentStatusType = 'ACTIVE' | 'ISOLATE' | 'TERMINATED' | 'QUARANTINE';
export type RuleVerdictType = 'APPROVED' | 'VIOLATION' | 'STRICT_DENY_FALLBACK';
export type OverrideActionType = 'FORCE_DENY' | 'FORCE_APPROVE';

/**
 * Global Metadata Header passed with every WebSocket telemetry event.
 */
export interface IDaemonEventHeader {
  readonly timestamp: string; // ISO 8601 UTC String
  readonly messageId: string; // UUID v4
  readonly systemPrivilegeLevel: 'ROOT' | 'SYSTEM' | 'DEGRADED';
}

/**
 * 1. SCHEMA: METRIC_CARD_PANEL Telemetry Payload
 */
export interface ISystemTelemetryPayload {
  readonly cpuUtilization: number; // Percentage: 0.00 to 100.00
  readonly memoryUtilization: number; // Percentage: 0.00 to 100.00
  readonly activeProvider: ProviderType;
  readonly connectedSwarmAgentCount: number;
  readonly isIpcConnected: boolean;
}

export interface IDaemonTelemetryEvent {
  readonly header: IDaemonEventHeader;
  readonly body: ISystemTelemetryPayload;
}

/**
 * 2. SCHEMA: REACT_STREAM_CONSOLE Event Payload
 */
export interface IReActStep {
  readonly stepNumber: number;
  readonly thought: string;
  readonly action: string;
  readonly observation: string;
}

export interface IReActReasoningChainPayload {
  readonly currentGoal: string;
  readonly telemetryContextSnapshotId: string;
  readonly activeStep: IReActStep;
}

export interface IReActReasoningEvent {
  readonly header: IDaemonEventHeader;
  readonly body: IReActReasoningChainPayload;
}

/**
 * 3. SCHEMA: SWARM_ORCHESTRATOR_GRID Sub-Agent Structural Metrics
 */
export interface IAgentNode {
  readonly agentId: string;
  readonly agentClass: 'NetworkAgent' | 'DiskAgent' | 'ProcessMonitorAgent' | 'MemoryGuardAgent';
  readonly status: AgentStatusType;
  readonly processedEventCount: number;
  readonly isolatedResourceIdentifier?: string;
}

export interface ISwarmClusterPayload {
  readonly parentTaskName: string;
  readonly clusterDeploymentStatus: 'STABLE' | 'DEGRADED' | 'CRITICAL_FAILURE';
  readonly activeAgents: readonly IAgentNode[];
}

export interface ISwarmClusterEvent {
  readonly header: IDaemonEventHeader;
  readonly body: ISwarmClusterPayload;
}

/**
 * 4. SCHEMA: CIRCUIT_BREAKER_CONSOLE Policy Verification Structures
 */
export interface IComplianceEvaluation {
  readonly evaluationId: string;
  readonly proposedActionSummary: string;
  readonly rulesEvaluated: readonly string[];
  readonly verdict: RuleVerdictType;
  readonly signatureVerificationKey?: string;
}

export interface ICircuitBreakerEvent {
  readonly header: IDaemonEventHeader;
  readonly body: IComplianceEvaluation;
}

/**
 * OUTBOUND CONTROL PAYLOADS (Dashboard Actions -> Daemon Ingestion Interface)
 */
export interface IAdminOverridePayload {
  readonly evaluationId: string;
  readonly decisionOverride: OverrideActionType;
  readonly hardwareAuthToken: string; // Cryptographic validation token signed outside browser space
  readonly administratorSignature: string;
}

export interface ISwarmMutationRequest {
  readonly actionType: 'SPAWN' | 'KILL' | 'ISOLATE_HOST';
  readonly targetAgentId?: string;
  readonly agentClassToSpawn?: string;
}

```

---

## 4. Abstract Service Inter-Component API Framework (`/src/api/service-contracts.ts`)

This outlines how the front-end components interact with data feeds using TypeScript type abstractions without binding to explicit framework tools (like React, Vue, or Angular).

```typescript
import {
  IDaemonTelemetryEvent,
  IReActReasoningEvent,
  ISwarmClusterEvent,
  ICircuitBreakerEvent,
  IAdminOverridePayload,
  ISwarmMutationRequest
} from '../types/daemon-api';

export type TelemetryCallback = (event: IDaemonTelemetryEvent) => void;
export type ReActStreamCallback = (event: IReActReasoningEvent) => void;
export type SwarmClusterCallback = (event: ISwarmClusterEvent) => void;
export type CircuitBreakerCallback = (event: ICircuitBreakerEvent) => void;
export type ConnectionErrorCallback = (error: Error) => void;

/**
 * DECOUPLED WEBSOCKET INTERFACE SERVICE CONTRACT
 * Explicitly defines the black-box contract for WebSocket connectivity streams.
 */
export interface IDaemonWebSocketStreamBus {
  connect(gatewayUrl: string): Promise<boolean>;
  disconnect(): void;
  
  // Event Registration Subscriptions
  subscribeToMetrics(callback: TelemetryCallback): string; // Returns Subscription UID
  subscribeToReasoningLoop(callback: ReActStreamCallback): string;
  subscribeToSwarmChanges(callback: SwarmClusterCallback): string;
  subscribeToCircuitBreaker(callback: CircuitBreakerCallback): string;
  
  unsubscribe(subscriptionUid: string): boolean;
  
  // Error handling hooks
  registerErrorHandler(callback: ConnectionErrorCallback): void;
}

/**
 * REST OUTBOUND MUTATION SERVICE CONTRACT
 * Structural pattern managing outbound actions passed across network boundary paths.
 */
export interface IDaemonControlRestClient {
  submitAdministrativeOverride(payload: IAdminOverridePayload): Promise<{ success: boolean; authorizationLogToken: string }>;
  mutateSwarmCluster(mutation: ISwarmMutationRequest): Promise<{ status: 'PROVISIONED' | 'REJECTED'; reason?: string }>;
  fetchHistoricalTraceLogs(sinkType: 'SYSTEM' | 'LLM_INTERACTION', limit: number): Promise<readonly string[]>;
}

```

---

## 5. UI Stability Controls & Fail-Secure Execution Blueprint

To shield the front-end from crashing under high telemetry throughput during active mitigation cycles, the implementation of the above interfaces must adhere to the following stability criteria:

1. **State Isolation Protection (Black Box State Partitioning):** Components must copy incoming network state tokens by value using structured cloning algorithms before mutation checks occur. Direct data object reference coupling across the view layer is forbidden.
2. **Buffer Throttling Threshold Limits:** The concrete layer wrapping `IDaemonWebSocketStreamBus` must incorporate an internal sampling queue framework. Incoming message streams exceeding 40 distinct transactions per second must collapse down into single consolidated snapshots before passing execution threads up to visual state managers.
3. **Fail-Secure Visualization Rules:** If the socket drops out, the local visualization data stores must lock instantly into place, render an overlay showing `[COMMUNICATION_LINK_TERMINATED]`, and continuously log low-level connection retries to browser storage.

---

### Phase I Gate Verification Check

The type contracts and functional definitions are complete.

