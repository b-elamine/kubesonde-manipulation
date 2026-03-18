![workflow](https://github.com/kubesonde/kubesonde/actions/workflows/go_main.yaml/badge.svg)
![frontend_test](https://github.com/kubesonde/kubesonde/actions/workflows/frontend_dev.yaml/badge.svg)
![frontend_deployment](https://github.com/kubesonde/kubesonde/actions/workflows/deploy_frontend.yaml/badge.svg)
[![Netlify Status](https://api.netlify.com/api/v1/badges/df3643ab-e317-4b96-b5c2-de937837b375/deploy-status)](https://app.netlify.com/sites/testksonde/deploys)

![Kubesonde logo](frontend/public/logo257.png "Kubesonde logo")

# Kubesonde — Deep Contributor Guide

> **Paper:** [Analyzing Microservice Connectivity with Kubesonde — FSE 2023](https://dl.acm.org/doi/10.1145/3611643.3613899)

Kubesonde is a **Kubernetes network security auditing tool**. It automatically probes every pod-to-pod, pod-to-service, and pod-to-internet network path inside a cluster, then compares what is *actually* reachable against what the operator *expects* to be allowed or denied. The goal is to surface policy misconfigurations (e.g., a pod that should be isolated but can still reach the internet, or reach a database it has no business talking to).

![kubesonde infra](docs/kubesonde.png "kubesonde infrastructure")

---

## Table of Contents

1. [The Problem Kubesonde Solves](#1-the-problem-kubesonde-solves)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Repository Layout](#3-repository-layout)
4. [Backend Deep Dive (crd/)](#4-backend-deep-dive-crd)
   - [CRD API Types](#41-crd-api-types)
   - [Entry Point & Controller Manager](#42-entry-point--controller-manager)
   - [The Reconciler](#43-the-reconciler)
   - [Event Listener](#44-event-listener)
   - [Event Storage (in-memory state)](#45-event-storage-in-memory-state)
   - [Probe Commands — how a probe is built](#46-probe-commands--how-a-probe-is-built)
   - [Dispatcher — the scheduling engine](#47-dispatcher--the-scheduling-engine)
   - [Inner — remote command execution](#48-inner--remote-command-execution)
   - [Monitor Controller](#49-monitor-controller)
   - [Debug Container](#410-debug-container)
   - [Recursive Probing — the heartbeat loop](#411-recursive-probing--the-heartbeat-loop)
   - [REST API Server](#412-rest-api-server)
   - [State Package](#413-state-package)
5. [Complete Data Flow (step by step)](#5-complete-data-flow-step-by-step)
6. [Output Data Format](#6-output-data-format)
7. [Frontend Deep Dive (frontend/)](#7-frontend-deep-dive-frontend)
8. [Docker Helper Images (docker/)](#8-docker-helper-images-docker)
9. [Running Kubesonde](#9-running-kubesonde)
   - [Prerequisites](#91-prerequisites)
   - [Install & Configure](#92-install--configure)
   - [Fetch Results](#93-fetch-results)
   - [Cleanup](#94-cleanup)
10. [Development Setup](#10-development-setup)
    - [Backend](#101-backend)
    - [Frontend](#102-frontend)
    - [Running Tests](#103-running-tests)
11. [Key Design Decisions & Trade-offs](#11-key-design-decisions--trade-offs)
12. [Known Limitations & TODOs (where to contribute)](#12-known-limitations--todos-where-to-contribute)
13. [Contributing](#13-contributing)
14. [Credits](#14-credits)

---

## 1. The Problem Kubesonde Solves

Kubernetes `NetworkPolicy` resources describe *intended* connectivity rules, but they do not tell you whether those rules are correctly applied by the CNI plugin, whether a misconfiguration leaves unexpected paths open, or whether a newly deployed pod inherits a dangerous default. Manual verification does not scale across tens of services.

Kubesonde answers: **"What can actually reach what, right now, in this running cluster?"**

It does this *from inside the cluster*, injecting short-lived ephemeral containers into pods and running real TCP/UDP scans — not simulated policy evaluations. The results are ground-truth.

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                         │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         Kubesonde Controller Manager Pod                │   │
│  │                                                         │   │
│  │  Reconciler ──starts──► EventListener  (watches pods   │   │
│  │      │                                  and services)  │   │
│  │      ├──────────────► Dispatcher       (priority queue │   │
│  │      │                                  + rate limit)  │   │
│  │      ├──────────────► RecursiveProbing (re-runs every  │   │
│  │      │                                  20 seconds)    │   │
│  │      └──────────────► MonitorController(reads netstat  │   │
│  │                                         from pods)     │   │
│  │                                                         │   │
│  │  REST API Server (port 2709) ◄── GET /probes           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   App Pod A  │  │   App Pod B  │  │   App Pod C  │  ...    │
│  │ [+debugger]  │  │ [+debugger]  │  │ [+debugger]  │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│          │nmap/nslookup exec'd via K8s pod/exec API             │
└─────────────────────────────────────────────────────────────────┘
          │
          │  kubectl port-forward :2709
          ▼
┌─────────────────────────────┐
│  User Machine               │
│  curl localhost:2709/probes │
│  → results.json             │
│                             │
│  React UI (Netlify)         │
│  Upload results.json        │
│  → Cytoscape.js graph       │
└─────────────────────────────┘
```

The system is composed of two independently deployable pieces:

| Piece | Language | Where it runs |
|-------|----------|---------------|
| **Controller** (`crd/`) | Go | Inside your K8s cluster as a Deployment |
| **Frontend** (`frontend/`) | TypeScript / React | Netlify (or locally via `npm run dev`) |

They are decoupled: the controller writes JSON; the frontend reads JSON. No live connection between them is needed.

---

## 3. Repository Layout

```
kubesonde/
├── crd/                            Go backend — the Kubernetes operator
│   ├── api/v1/
│   │   ├── kubesonde_types.go      CRD schema: KubesondeSpec, ProbingAction, etc.
│   │   ├── probe_server_types.go   Output types: ProbeOutput, ProbeOutputItem, etc.
│   │   └── zz_generated.deepcopy.go  Auto-generated deepcopy (do not edit)
│   ├── cmd/
│   │   └── main.go                 Entry point: starts manager + REST API
│   ├── config/                     Kustomize manifests (CRD, RBAC, samples)
│   ├── controllers/
│   │   ├── debug-container/        Manages ephemeral debugger containers
│   │   ├── dispatcher/             Priority queue probe scheduler
│   │   ├── event-storage/          In-memory store: active pods, probes, services
│   │   ├── events/                 K8s informer: pod & service lifecycle events
│   │   ├── inner/                  Executes remote commands via pod/exec API
│   │   ├── metrics/                Prometheus metrics
│   │   ├── monitor/                Reads netstat output from debugger containers
│   │   ├── probe_command/          Builds nmap/nslookup commands from pod specs
│   │   ├── recursive-probing/      Heartbeat loop that re-queues all probes
│   │   ├── state/                  Mutable global state (netstat results, etc.)
│   │   └── utils/                  Shared helpers (label matching, etc.)
│   ├── internal/controller/
│   │   └── kubesonde_controller.go The Reconcile() function
│   ├── rest_apis/                  HTTP server on :2709
│   ├── Dockerfile
│   ├── Makefile
│   └── go.mod
│
├── frontend/                       React + TypeScript UI
│   ├── src/
│   │   ├── components/             UI components (graph, table, filters, upload)
│   │   ├── entities/               Data models and graph builder
│   │   ├── hoc/                    App root and routing
│   │   ├── mock/                   Sample data for UI development
│   │   └── utils/                  Graph algorithm helpers
│   ├── public/                     Static assets (logo, etc.)
│   ├── package.json
│   └── vite.config.js
│
├── docker/
│   ├── gonetstat/                  Go-based netstat binary (runs inside pods)
│   └── pynetstat/                  Python-based netstat alternative
│
├── examples/                       Real probe output JSON files (great for UI dev)
│   ├── wordpress.json
│   ├── gitlab.json
│   ├── orangehrm.json
│   └── bitbucket.json
│
├── docs/                           Architecture diagram
├── kubesonde.yaml                  Full K8s manifest (CRD + RBAC + Deployment)
└── README.md
```

---

## 4. Backend Deep Dive (`crd/`)

### 4.1 CRD API Types

**File:** `crd/api/v1/kubesonde_types.go`

This file defines what users write when they create a Kubesonde resource in their cluster.

```go
type KubesondeSpec struct {
    Namespace     string         // which namespace to probe
    Probe         string         // "all" = probe everything, "none" = only use Include list
    DebuggerImage string         // override the debugger container image
    MonitorImage  string         // override the monitor container image
    Include       []IncludedItem // explicit probes WITH expected outcomes
    Exclude       []ExcludedItem // probes to skip entirely
}
```

**Example CRD instance:**
```yaml
apiVersion: security.kubesonde.io/v1
kind: Kubesonde
metadata:
  name: my-scan
spec:
  namespace: production
  probe: all
  include:
    - fromPodSelector: frontend
      toPodSelector: database
      port: "5432"
      protocol: TCP
      expected: Deny   # we expect this to be blocked
```

`ActionType` can be `"Allow"` or `"Deny"`. `ProbeType` can be `"all"` or `"none"`.

**File:** `crd/api/v1/probe_server_types.go`

This file defines the **output** structures — what the REST API returns as JSON.

```go
type ProbeOutput struct {
    Items                      []ProbeOutputItem   // each individual probe result
    Errors                     []ProbeOutputError  // probes that failed to execute
    PodNetworkingV2            PodNetworkingInfoV2 // actual listening ports per pod
    PodConfigurationNetworking PodNetworkingInfoV2 // ports declared in pod spec
}

type ProbeOutputItem struct {
    Type            ProbeOutputItemType  // "Probe" or "Information"
    ExpectedAction  ActionType           // what we expected ("Allow" or "Deny")
    ResultingAction ActionType           // what actually happened
    Source          ProbeEndpointInfo    // source pod/service
    Destination     ProbeEndpointInfo    // destination pod/service/internet
    Protocol        string               // "TCP", "UDP", "SCTP"
    Port            string
    Timestamp       int64
}
```

`ProbeEndpointType` classifies each endpoint as `"Pod"`, `"Service"`, or `"Internet"`.

---

### 4.2 Entry Point & Controller Manager

**File:** `crd/cmd/main.go`

This is where the Go process starts. It does two things:

1. Registers the `KubesondeReconciler` with the controller-runtime manager, which handles leader election, health checks, and metric exposure.
2. Calls `restapis.ServeHTTP()` to start the HTTP server on port `2709`.

The controller-runtime manager watches the Kubernetes API for changes to `Kubesonde` objects and triggers `Reconcile()` on each change.

---

### 4.3 The Reconciler

**File:** `crd/internal/controller/kubesonde_controller.go`

`Reconcile()` is called by the controller-runtime framework whenever a `Kubesonde` CRD object is created, updated, or deleted. This is the **main entry point** into all probing logic.

```go
func (r *KubesondeReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // Fetch the Kubesonde object
    r.Get(ctx, req.NamespacedName, &Kubesonde)

    // Launch all subsystems as goroutines
    go kubesondeDispatcher.Run(apiClient)           // starts consuming the probe queue
    go kubesondeEvents.InitEventListener(...)        // watches for pod/service changes
    go recursiveprobing.RecursiveProbing(..., 20*time.Second) // re-runs probes every 20s
    go kubesondemonitor.RunMonitorContainers(...)    // reads netstat from pods

    return ctrl.Result{}, nil
}
```

**Important nuance:** Every call to `Reconcile()` launches new goroutines. This means if the CRD is updated multiple times, you will have multiple sets of goroutines. The current code does not guard against this (a known issue — see section 12).

---

### 4.4 Event Listener

**File:** `crd/controllers/events/listener.go`, `pod.event.go`, `service.event.go`

Uses the Kubernetes **shared informer** mechanism to watch for pod and service changes in real-time (re-syncs every 5 seconds).

**Pod events:**
- `AddFunc`: When a new pod is detected and it matches the target namespace/spec, `AddPodEvent()` is called. This injects a debugger ephemeral container into the pod and builds probe commands for it.
- `DeleteFunc`: Removes the pod from the active pods store.
- `UpdateFunc`: Handles pods transitioning away from `Running` state.

**Service events:**
- `AddFunc`: When a new service with a valid `ClusterIP` appears, it is added to the services store.
- Services named `kubernetes` or `kube-dns` are explicitly excluded from probing.

The listener runs an infinite loop that logs active pod counts every 30 seconds.

---

### 4.5 Event Storage (in-memory state)

**Files:** `crd/controllers/event-storage/probe.go`, `recorder.go`

This package is the **global in-memory database**. It holds:

| Store | Type | Purpose |
|-------|------|---------|
| `commands` | `map[string]KubesondeCommand` | All probe definitions, keyed by a unique string (`src-cmd-destIP-destPort-protocol`) |
| `_activePods` | `map[string]CreatedPodRecord` | Currently running pods, with deployment/replicaset metadata |
| `_deletedPods` | `map[string]DeletedPodRecord` | Recently deleted pods |
| `_services` | `[]v1.Service` | All discovered services |

**Critical detail:** The deduplication key for probes is:
```
{SourcePodName}-{Command}-{DestinationIPAddress}-{DestinationPort}-{Protocol}
```
`AddProbe()` is a no-op if the key already exists, preventing redundant probes.

---

### 4.6 Probe Commands — how a probe is built

**Files:** `crd/controllers/probe_command/probe_commands.go`, `probe.command.go`

This is where the actual shell commands are constructed. Kubesonde uses `nmap` (and `nslookup` for DNS) running inside the target pod's debugger container.

**Command templates:**
```go
const nmapTCPCommand  = "nmap --open --version-intensity=0 --max-retries=3 -T5 -n -sT -Pn -p %d %s"
const nmapUDPCommand  = "nmap --open --version-intensity=0 --max-retries=3 -T5 -n -sU -p %d %s"
const nmapSCTPCommand = "nmap --open -sY -p %d %s"
const dnsUDPCommand   = "nslookup -timeout=5 %s %s"
```

**How a probe command is built:**

For **pod-to-pod** probing, `buildCommand()` iterates all source/destination pod pairs and all ports declared in the destination pod's spec. For each combination it creates a `KubesondeCommand` with:
- `Command`: the nmap string to run
- `ContainerName: "debugger"` — always runs in the ephemeral container, not the app container
- `ProbeChecker`: a function that parses nmap stdout to determine Allow/Deny
- `Action: DENY` — the default expected outcome is deny (policy of least privilege)

**Probe checker functions:**
```go
func NmapSucceded(output string) bool {
    // Returns true if nmap reports "1 host up" AND port is "open"
}
func CurlSucceded(output string) bool {
    // Returns true if HTTP status code < 500
}
func NslookupSucceded(output string) bool {
    // Returns true if output contains "Server:"
}
```

**Fixed external targets** — every pod always gets probes against:
- `Google DNS` (8.8.8.8:53 TCP + UDP)
- `Google HTTP` (google.com:80 TCP)
- `Google HTTPS` (google.com:443 TCP)
- `Kube DNS` (kube-dns.kube-system.svc.cluster.local:53 TCP + UDP)

This tells you whether pods have unrestricted internet access.

**Key builder functions:**

| Function | What it builds |
|----------|----------------|
| `BuildCommandsFromPodSelectors(pods, ns)` | Full cross-product: every pod to every other pod + internet |
| `BuildCommandsToServices(pod, services)` | One pod to all services |
| `BuildTargetedCommands(target, pods)` | All probes involving a specific pod (used on pod add event) |
| `BuildTargetedCommandsToDestination(pods, dest, ports, protocols)` | All pods → specific destination ports (used by monitor) |
| `BuildCommandFromService(pods, service)` | All pods → a specific service on TCP+UDP+SCTP |

---

### 4.7 Dispatcher — the scheduling engine

**Files:** `crd/controllers/dispatcher/probe_dispatcher.go`, `priority_queue.go`

The dispatcher is a **priority queue** (Go's `container/heap`) that serializes probe execution to avoid overwhelming the cluster.

```
SendToQueue(probes, LOW)   ← called by RecursiveProbing (bulk re-runs)
SendToQueue(probes, HIGH)  ← called by MonitorController (new ports discovered)
```

`Run()` is an infinite loop that:
1. Pops the highest-priority item from the heap.
2. Calls `inner.InspectAndStoreResult()` to execute the probe and save the result.
3. Sleeps for `50ms` between probes (= ~20 probes/second maximum throughput).

A semaphore (`semaphore.NewWeighted(1)`) serializes access to the queue between the producer goroutines and the consumer loop.

---

### 4.8 Inner — remote command execution

**File:** `crd/controllers/inner/pods_controller.go`

This is the layer that actually talks to the Kubernetes API to run a shell command inside a container.

`runRemoteCommand()` uses the **SPDY executor** (Kubernetes `pod/exec` subresource) to run a command with a 5-second timeout. It returns the stdout, which is then passed to the `ProbeChecker` function to determine Allow/Deny.

```go
req := client.CoreV1().RESTClient().Post().
    Resource("pods").Namespace(ns).Name(podName).
    SubResource("exec")
// ...sets command, container="debugger", stdin, stdout, stderr
exec.Stream(...)
return checker(stdout.String())
```

`InspectAndStoreResult()` orchestrates this: runs the command, maps the boolean result to `Allow`/`Deny`, and stores a `ProbeOutputItem` in the results store.

---

### 4.9 Monitor Controller

**File:** `crd/controllers/monitor/monitor_controller.go`

This controller supplements the static port discovery (from pod specs) with **runtime port discovery** via netstat.

Every 10 seconds it:
1. Gets all active pods from event-storage.
2. For each pod that doesn't yet have a netstat result, checks if the debugger ephemeral container is running.
3. Calls `debug_container.RunMonitorContainerProcess()` to execute the netstat binary inside the debugger container.
4. Parses the JSON netstat output (a list of `{type, laddr: [ip, port]}` entries).
5. Filters out loopback (`127.0.0.1`) addresses.
6. Builds new `HIGH` priority probes for any ports that were not declared in the pod spec.

This is important: a pod might be listening on a port that is not declared in `containers[].ports[]`. The monitor catches these hidden ports.

---

### 4.10 Debug Container

**File:** `crd/controllers/debug-container/ephemeralContainer_controller.go`

Manages the **ephemeral container** injected into each application pod. The ephemeral container runs a custom image containing `nmap`, `nslookup`, and the netstat binary.

Key functions:
- `EphemeralContainerExists(pod)` — checks if pod already has a debugger container
- `EphemeralContainersRunning(pod)` — checks if the debugger container's state is Running
- `RunMonitorContainerProcess(client, ns, podName)` — executes the netstat binary and returns raw stdout/stderr buffers

---

### 4.11 Recursive Probing — the heartbeat loop

**File:** `crd/controllers/recursive-probing/recursive_probing.go`

This is elegantly simple. Every 20 seconds, if the dispatcher queue is empty, it re-queues all known probes at LOW priority:

```go
func RecursiveProbing(kubesonde Kubesonde, when time.Duration) {
    task := func() {
        if QueueSize() == 0 {
            go RunProbing() // re-queues all probes from event-storage
        }
        RecursiveProbing(kubesonde, when) // reschedule itself
    }
    time.AfterFunc(when, task) // non-blocking timer
}
```

This means Kubesonde continuously re-validates your network state. If a NetworkPolicy is changed, the next 20-second cycle will pick it up.

---

### 4.12 REST API Server

**Files:** `crd/rest_apis/apis.go`, `crd/rest_apis/handler.go`

A minimal Go HTTP server listening on port `2709`. There is a single endpoint:

```
GET /probes  →  returns ProbeOutput as JSON
```

The handler calls `state.GetResults()` and serializes the result. CORS headers are set to allow the Netlify frontend to fetch directly.

---

### 4.13 State Package

**File:** `crd/controllers/state/probe_state_controller.go`

Holds the **final results** (as opposed to event-storage which holds probe definitions). Key stores:

- Results list (`[]ProbeOutputItem`) — every executed probe outcome
- Netstat results (`PodNetworkingInfoV2`) — runtime port maps per pod
- Pod-has-netstat tracking (`[]string`) — which pods have already been monitored

---

## 5. Complete Data Flow (step by step)

Here is the exact sequence from cluster creation to frontend visualization:

```
1. kubectl apply -f kubesonde.yaml
   → Creates CRD definition, RBAC roles, and the controller Deployment

2. kubectl apply -f probe.yaml  (your Kubesonde CR)
   → Reconciler.Reconcile() is called
   → Launches 4 goroutines: Dispatcher, EventListener, RecursiveProbing, Monitor

3. EventListener starts K8s informers
   → For each running pod in the target namespace:
       a. AddPodEvent() is called
       b. An ephemeral "debugger" container is injected into the pod
       c. BuildTargetedCommands() generates nmap commands for that pod
       d. Commands are stored in event-storage and added to state
       e. Commands are queued in the Dispatcher at HIGH priority

4. Dispatcher.Run() loops forever
   → Pops a KubesondeCommand from the priority queue
   → Calls inner.runRemoteCommand() which uses pod/exec API to run nmap inside
     the "debugger" container of the SOURCE pod, targeting the DESTINATION IP:port
   → NmapSucceded(stdout) returns true/false → "Allow" or "Deny"
   → Stores ProbeOutputItem in state

5. RecursiveProbing fires every 20 seconds
   → If queue is empty, re-queues all known probes at LOW priority
   → This continuously re-validates network state

6. MonitorController fires every 10 seconds
   → Executes netstat inside each pod's debugger container
   → Discovers ports NOT declared in the pod spec
   → Builds new HIGH-priority probes for those ports

7. User fetches results
   kubectl port-forward deployment/kubesonde-controller-manager 2709
   curl localhost:2709/probes > results.json

8. User opens https://kubesonde.jackops.dev
   → Uploads results.json
   → React frontend parses ProbeOutput JSON
   → graph.builder.ts transforms items into Cytoscape nodes/edges
   → Graph renders: green edges = Allow, red edges = Deny,
     orange edges = policy mismatch (expected Deny, got Allow)
```

---

## 6. Output Data Format

The JSON returned by `GET /probes` has this shape:

```json
{
  "items": [
    {
      "type": "Probe",
      "expectedAction": "Deny",
      "resultingAction": "Allow",
      "source": {
        "type": "Pod",
        "name": "wordpress-abc123",
        "namespace": "default",
        "IPAddress": "10.244.1.5",
        "labels": "app=wordpress"
      },
      "destination": {
        "type": "Internet",
        "name": "Google",
        "IPAddress": "google.com"
      },
      "protocol": "TCP",
      "port": "443",
      "timestamp": 1704067200
    }
  ],
  "errors": [],
  "podNetworkingv2": {
    "wordpress-abc123": [
      {"port": "8080", "ip": "0.0.0.0", "protocol": "TCP"}
    ]
  },
  "podConfigurationNetworking": {
    "wordpress-abc123": [
      {"port": "8080", "ip": "0.0.0.0", "protocol": "TCP"}
    ]
  }
}
```

**Key fields:**
- `expectedAction` vs `resultingAction`: when these differ, it's a policy violation.
- `podNetworkingv2`: ports discovered at runtime via netstat (the ground truth).
- `podConfigurationNetworking`: ports declared in the pod spec (may differ from runtime).

---

## 7. Frontend Deep Dive (`frontend/`)

The frontend is a standalone React application that **never connects to the cluster directly**. It only works with the JSON file you download.

**Key components:**

| Component | File(s) | Role |
|-----------|---------|------|
| App root | `src/hoc/App.tsx` | Router, layout |
| Graph view | `src/components/graph/` | Cytoscape.js canvas, node/edge rendering |
| Graph controller | `src/components/graph-controller/` | Port/protocol filter, search |
| Graph popup | `src/components/graph-popup/` | Sidebar detail on node click |
| Table view | `src/components/table/` | Tabular listing of all probe results |
| Upload | `src/components/upload/` | JSON file picker |

**Data pipeline inside the frontend:**

```
Upload JSON file
  → probeOutput.ts: parse and type-check
  → graph.builder.ts: transform ProbeOutputItems into
      { nodes: [{id, label, type}], edges: [{source, target, port, protocol, outcome}] }
  → Cytoscape.js: renders nodes as circles, edges as arrows
      green  = Allow (as expected)
      red    = Deny (as expected)
      orange = MISMATCH (expected Deny, got Allow — this is the important one)
```

**Local development:**
```bash
cd frontend
npm install
npm run dev       # starts Vite dev server at http://localhost:5173
```

Use files from `examples/` to develop without a running cluster.

---

## 8. Docker Helper Images (`docker/`)

These images provide the binaries that run **inside the ephemeral debugger container**:

- **`docker/gonetstat/`**: A Go binary that reads `/proc/net/tcp`, `/proc/net/tcp6`, `/proc/net/udp`, `/proc/net/udp6` and outputs JSON: `[{"type":1,"laddr":["0.0.0.0","8080"]}]`. Type 1 = TCP, 2 = UDP.
- **`docker/pynetstat/`**: A Python equivalent of the same.

The controller image (built from `crd/Dockerfile`) includes `nmap` and `nslookup`. The netstat binary runs separately inside the same ephemeral container.

---

## 9. Running Kubesonde

### 9.1 Prerequisites

- A running Kubernetes cluster (Minikube, Kind, GKE, EKS, etc.)
- `kubectl` configured to talk to it
- The app you want to test already deployed (e.g., `helm install wordpress bitnami/wordpress`)

```bash
git clone https://github.com/kubesonde/kubesonde.git
cd kubesonde
```

### 9.2 Install & Configure

**Step 1 — Deploy Kubesonde:**
```bash
kubectl apply -f kubesonde.yaml
```
This creates:
- The `kubesonde-system` namespace
- The `Kubesonde` CRD definition
- RBAC roles (ClusterRole + ClusterRoleBinding with full `*` permissions — needed to inject ephemeral containers)
- The controller Deployment

**Step 2 — Create a scan:**
```yaml
# probe.yaml
apiVersion: security.kubesonde.io/v1
kind: Kubesonde
metadata:
  name: kubesonde-sample
spec:
  namespace: default   # scan this namespace
  probe: all           # probe everything by default
```
```bash
kubectl apply -f probe.yaml
```

**Wait** — probing takes time. For a cluster with ~10 pods, wait 3–5 minutes. For more pods, wait longer. You can watch progress:
```bash
kubectl logs -n kubesonde-system deployment/kubesonde-controller-manager -f
```

### 9.3 Fetch Results

```bash
# Open a tunnel to the controller's REST API
kubectl --namespace kubesonde-system port-forward deployment.apps/kubesonde-controller-manager 2709

# In another terminal, download results
curl localhost:2709/probes > results.json
```

Then go to [https://kubesonde.jackops.dev](https://kubesonde.jackops.dev) and upload `results.json`.

### 9.4 Cleanup

```bash
kubectl delete -f probe.yaml       # remove the scan object
kubectl delete -f kubesonde.yaml   # remove everything else
```

---

## 10. Development Setup

### 10.1 Backend

**Requirements:** Go 1.21+, `kubectl`, a running cluster (or `envtest` for unit tests)

```bash
cd crd

# Install dependencies
go mod download

# Generate CRD manifests and deepcopy code (run after modifying api/v1/ files)
make generate
make manifests

# Run controller locally against your current kubeconfig context
make run

# Build the Docker image
make docker-build IMG=ghcr.io/kubesonde/kubesonde:dev

# Lint
golangci-lint run ./...
```

**Adding a new controller subsystem:**
1. Create a new package under `crd/controllers/your-package/`
2. Add a `go` call to it in `kubesonde_controller.go`'s `Reconcile()` method
3. If it needs to read/write state, use the `event-storage` and `state` packages

**After modifying `crd/api/v1/` types:**
```bash
make generate   # regenerates zz_generated.deepcopy.go
make manifests  # regenerates CRD YAML in config/crd/
```

### 10.2 Frontend

**Requirements:** Node.js 18+

```bash
cd frontend
npm install
npm run dev      # dev server with hot reload
npm run build    # production build to dist/
npm run test     # Jest unit tests
```

To develop without a cluster, use the example files:
```
frontend/src/mock/    ← mock data
examples/*.json       ← real probe outputs
```

### 10.3 Running Tests

**Backend (Go):**
```bash
cd crd
go test ./...                # all tests
go test ./controllers/...   # controller tests only
```

Tests use the Ginkgo BDD framework. Test files are named `*_test.go` and `*_suite_test.go`.

**Frontend:**
```bash
cd frontend
npm run test
```

---

## 11. Key Design Decisions & Trade-offs

| Decision | Reason | Trade-off |
|----------|--------|-----------|
| **Ephemeral containers for probing** | Does not modify the app pod spec; no sidecar injection | Requires K8s 1.23+ (ephemeral containers GA); needs broad RBAC |
| **nmap over curl** | Works for TCP, UDP, SCTP; does not require the app to serve HTTP | Slower than a simple TCP connect; requires nmap in the debugger image |
| **In-memory state** | Simple, fast, no external dependency | State is lost if the controller pod restarts; no persistence across runs |
| **JSON file as the interface between backend and frontend** | Frontend can work offline; no CORS issues in production | User must manually download and upload the file |
| **Priority queue with 20 probes/second limit** | Avoids flooding the CNI / kube-apiserver | Large clusters with many pods will take a long time to complete a full sweep |
| **Default expected action = Deny** | Assumes a zero-trust baseline | Users must explicitly set `expected: Allow` in `Include` for connections they know should work |
| **Re-probe every 20 seconds** | Catches changes to NetworkPolicies in near-real-time | High API server load in clusters with many pods |

---

## 12. Known Limitations & TODOs (where to contribute)

These are taken directly from `TODO` comments in the source code — they are the best places to start contributing:

| Location | Issue | Difficulty |
|----------|-------|-----------|
| `kubesonde_controller.go:61` | **Pod deletion not handled** — when a pod is deleted, its probes are not removed from state | Medium |
| `kubesonde_controller.go:62` | **Resource deletion not handled** — when the Kubesonde CR is deleted, state is not cleared | Medium |
| `kubesonde_controller.go:42` | **No fake clock for testing** — makes the reconciler hard to unit test | Medium |
| `dispatcher/probe_dispatcher.go:57` | Dispatcher loop is polling-based (`for{}`), could be event-driven | Medium |
| `probe_server_types.go:95` | `PodNetworking` (v1) field can be deleted — `PodNetworkingV2` replaces it | Easy |
| `monitor_controller.go:165` | Namespace is hardcoded to first active pod's namespace instead of from the spec | Easy |
| `monitor_controller.go:202` | SCTP is not handled in netstat output parsing | Easy |
| `kubesonde_controller.go` | Multiple `Reconcile()` calls launch multiple goroutine sets — no deduplication | Hard |
| General | No persistence — results are lost on controller restart | Hard |
| General | Frontend is disconnected — user must manually download and re-upload JSON | Hard |

---

## 13. Contributing

Contributions are welcome. The best flow:

1. Fork the repo and create a feature branch.
2. For backend changes: run `go test ./...` and ensure `make manifests` produces clean output.
3. For API type changes: run `make generate && make manifests`.
4. For frontend changes: run `npm run test`.
5. Open a PR describing what you changed and why.

**Good first issues** (easy, impactful):
- Fix the namespace hardcode in `monitor_controller.go:165`
- Add SCTP support to netstat parsing (`monitor_controller.go:202`)
- Remove the deprecated `PodNetworking` v1 field and update the frontend to only use `podNetworkingv2`
- Add unit tests for `BuildCommandsFromPodSelectors` in `probe_command/`

---

## 14. Credits

Logo by [Elisabetta Russo](https://stelladigitale.it) — info@stelladigitale.it

Licensed under the [Apache License 2.0](LICENSE).
