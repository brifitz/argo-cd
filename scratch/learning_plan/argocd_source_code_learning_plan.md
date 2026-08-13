# Argo CD Source Code Learning Plan

**Approach:** modify → debug → inspect → understand impact  
**Primary target:** Argo CD **v3.5.1**  
**Goal:** build a strong working understanding of Argo CD internals and component interaction, to the point where small real source-code changes and bug fixes are approachable.

---

## 1. What this plan is trying to achieve

This is not primarily an Argo CD *user* course. The objective is to understand the Argo CD codebase by repeatedly doing the following:

1. identify a small piece of important source code;
2. make a deliberately small change to it;
3. run the relevant Argo CD component under the debugger;
4. trigger the code path from the CLI, Kubernetes, or Git;
5. stop at a breakpoint;
6. inspect the real Go values flowing through the system;
7. step through the surrounding calls;
8. observe the visible effect of the change;
9. revert or refine the change;
10. explain how that code fits into the larger Argo CD architecture.

The artificial changes are not intended to be good production changes. They are probes: a controlled way of making an internal path observable.

By the end of the plan you should be able to reason about an end-to-end flow such as:

```text
Git repository
      |
      v
argocd-repo-server
      |
      | generated desired manifests
      v
argocd-application-controller
      |
      | desired state vs live state
      v
diff / health / reconciliation
      |
      | sync operation
      v
GitOps Engine
      |
      v
Kubernetes API
      |
      v
live-state cache / watchers
      |
      v
Application status becomes Synced / Healthy
```

You should also understand how the following fit around that central flow:

```text
                    +-----------------------+
                    |     argocd-server     |
CLI / UI ---------->| API + auth + RBAC     |
                    +-----------+-----------+
                                |
                                v
                       Application resources

+----------------------+                 +----------------------+
| ApplicationSet       |                 | Redis / caches       |
| controller           |                 |                      |
+----------+-----------+                 +----------+-----------+
           |                                         |
           v                                         |
     Application CRs                                 |
           |                                         |
           +--------------------+--------------------+
                                |
                                v
                     Application Controller
```

---

# Part I — Environment and architectural orientation

## Chapter 1 — Build Argo CD and establish the debugger workflow

### Objective

Create a repeatable development environment in which one Argo CD process can be launched from VS Code under the Go debugger while the remaining components continue to run normally.

This workflow will be reused in almost every later chapter.

### Pin the source tree

Use a fixed release rather than `master`:

```bash
git clone https://github.com/argoproj/argo-cd.git
cd argo-cd

git checkout v3.5.1
git switch -c learning/argocd-v3.5.1
```

Check:

```bash
git status
git describe --tags --always
```

Expected version:

```text
v3.5.1
```

### Local Kubernetes cluster

Use a disposable local Kubernetes cluster, for example `kind`.

```bash
kind create cluster --name argocd-src
kubectl cluster-info
kubectl get nodes
```

Install the Argo CD resources from the checked-out source tree:

```bash
kubectl create namespace argocd

kubectl apply \
  -n argocd \
  --server-side \
  --force-conflicts \
  -f manifests/install.yaml

kubectl config set-context --current --namespace=argocd
```

The important idea is that the Kubernetes cluster contains the Argo CD CRDs, ConfigMaps, Secrets and other cluster-side state, but during source development the Argo CD processes themselves can run on your laptop.

### Run Argo CD locally

Start by proving that the repository works unmodified:

```bash
make start-local ARGOCD_GPG_ENABLED=false GIT_TAG=v3.5.1
```

The explicit `GIT_TAG=v3.5.1` is required for this checkout because the current
commit is also associated with a `stable` Git tag. Without the override, the
locally built API server can identify its internal gRPC client as
`argocd-client/stable`, which is not a valid semantic version and causes the UI
API requests to fail. Keep the `GIT_TAG` value aligned with the Argo CD version
being studied.

Useful checks:

```bash
goreman run status
kubectl get applications
```

Argo CD's local API server normally listens on port `8080`.

Convenient CLI settings:

```bash
export ARGOCD_SERVER=127.0.0.1:8080
export ARGOCD_OPTS="--plaintext --insecure"
```

### The debugger pattern

When debugging one component, do **not** run a second copy of that same process.

For example, when debugging the API server, run the other processes but exclude `api-server`:

```bash
make run exclude=api-server
```

Then launch the API server from VS Code.

For other chapters the same pattern becomes:

```text
debug repo-server
    -> run everything except repo-server

debug application controller
    -> run everything except controller

debug ApplicationSet controller
    -> run everything except applicationset-controller
```

Use the top-level `Procfile` as the source of truth for:

- executable/component name;
- command-line arguments;
- ports;
- environment variables.

Do not copy a launch configuration from an old blog post and assume it is current.

### First artificial change

Pick a very small API-server path and add a local diagnostic branch, for example:

```go
if app.Name == "learning-app" {
    log.Infof("LEARNING: observed application %s", app.Name)
}
```

The purpose is merely to prove:

```text
source edit
  -> launch component under debugger
  -> trigger request
  -> hit breakpoint
  -> inspect variable
  -> see changed behaviour
```

### Breakpoint exercise

Set a breakpoint in the Application API service and trigger a command such as:

```bash
argocd app list
```

or:

```bash
argocd app get learning-app
```

At the breakpoint inspect:

- request object;
- `context.Context`;
- application name;
- namespace;
- service receiver;
- downstream Kubernetes client;
- returned `Application`.

### Completion criterion

Do not move on until you can:

- stop exactly one Argo CD component;
- start the rest of Argo CD;
- launch the missing component from VS Code;
- hit a breakpoint from an external action;
- inspect Go variables;
- make a source edit and observe its effect.

That capability is the foundation for the whole plan.

---

## Chapter 2 — Repository map through executable entry points

### Objective

Understand the source tree without spending hours passively reading directories.

Start at executable entry points and follow construction of major components.

### Focus areas

Explore these areas:

```text
cmd/
controller/
reposerver/
server/
applicationset/
pkg/apis/application/
util/
```

For each executable, find:

1. its command construction;
2. configuration/flags;
3. client construction;
4. service/controller construction;
5. `Run`, `Start`, or server registration;
6. long-running worker loops.

### Main binaries to locate

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-applicationset-controller
argocd-commit-server
argocd
```

### Debugger exercise

Put startup breakpoints in three components on separate runs:

- API server;
- repo server;
- application controller.

For each, inspect the major dependencies passed into the component.

Examples:

```text
Kubernetes clients
repository clients
cache objects
settings manager
database abstraction
informers
work queues
metrics servers
```

### Artificial change

Add one clearly marked startup log line to each of the three processes:

```text
LEARNING: api server constructed
LEARNING: repo server constructed
LEARNING: application controller constructed
```

Then remove the lines.

### What you should understand

At the end of this chapter you should be able to answer:

- Which process owns the user-facing API?
- Which process talks to Git and renders manifests?
- Which process performs reconciliation?
- Which process creates `Application` resources from `ApplicationSet`?
- Which state lives in Kubernetes?
- Which interactions are in-process calls and which are gRPC calls?

---

# Part II — The central Argo CD data model

## Chapter 3 — `Application` and `AppProject`

### Objective

Understand the objects that drive most Argo CD behaviour.

### Source focus

Start with the application API types under:

```text
pkg/apis/application/
```

Find the definitions of:

```go
Application
ApplicationSpec
ApplicationStatus
ApplicationSource
ApplicationDestination
SyncPolicy
Operation
OperationState
AppProject
```

Do not try to memorize every field.

Concentrate on the fields that participate in the normal reconciliation path.

### Test application

Create a tiny Git repository containing:

```text
learning-app/
  deployment.yaml
  service.yaml
```

Create an Argo CD `Application` pointing to it.

Use a unique name throughout the plan:

```text
learning-app
```

This makes debugger conditions easy:

```text
app.Name == "learning-app"
```

### Debugging exercise

Run the application controller under the debugger.

Trigger:

```bash
kubectl edit application learning-app
```

or modify a source field with `kubectl patch`.

Stop where the controller obtains the `Application`.

Inspect:

```text
app.Name
app.Namespace
app.Spec.Source
app.Spec.Destination
app.Spec.Project
app.Spec.SyncPolicy
app.Status.Sync
app.Status.Health
app.Status.OperationState
```

### Artificial change

For `learning-app` only, emit a diagnostic line containing:

```text
application name
repo URL
target revision
source path
destination namespace
current sync status
```

### Second experiment: AppProject

Create a dedicated project:

```text
learning-project
```

Move the Application into it.

Set a breakpoint where the controller retrieves or validates the project.

Inspect:

```text
project name
source restrictions
destination restrictions
resource permissions
application namespace permissions
```

### Mental model to retain

An `Application` is not the deployment itself.

It is the object describing the relationship:

```text
desired source
      +
destination cluster/namespace
      +
sync behaviour
      +
observed status
```

The controller continuously turns that specification into reconciliation work.

---

# Part III — Git to desired state

## Chapter 4 — Repo Server: from repository to Kubernetes manifests

### Objective

Understand exactly how Argo CD obtains desired state.

### Component under debugger

```text
argocd-repo-server
```

### Primary source area

Start with:

```text
reposerver/repository/repository.go
```

A particularly useful entry point in v3.5.1 is:

```go
func (s *Service) GenerateManifest(
    ctx context.Context,
    q *apiclient.ManifestRequest,
) (*apiclient.ManifestResponse, error)
```

### Initial experiment

Use the simplest possible source first:

```text
plain YAML directory
```

Avoid Helm and Kustomize until you understand the basic flow.

Trigger a hard refresh:

```bash
argocd app get learning-app --hard-refresh
```

### Breakpoint 1

Stop at `Service.GenerateManifest`.

Inspect the request:

```text
repository
revision
resolved revision, when available
application source
path
destination
namespace
application name
manifest generation options
multiple-source flags
```

### Follow the path

Step far enough to identify:

```text
repository acquisition
revision resolution
working tree / temporary directory
manifest type detection
manifest generation
parsing into Kubernetes objects
response construction
```

Do not step into every Git helper.

The target is the architecture, not every utility function.

### Artificial source change

For `learning-app` only, modify the generated desired state by adding an annotation:

```yaml
metadata:
  annotations:
    learning.argocd.local/reposerver: "true"
```

The exact implementation point is deliberately something you should locate after observing the generated object list.

### Observe impact

After rebuilding/restarting the repo server:

```bash
argocd app get learning-app --hard-refresh
argocd app diff learning-app
```

Observe where the annotation appears.

Then decide:

- Is it visible only in target state?
- Does it make the Application `OutOfSync`?
- What would happen on sync?

### Debugger inspection

Inspect one generated Kubernetes object:

```text
GroupVersionKind
metadata.name
metadata.namespace
annotations
labels
spec
```

### Second experiment: Kustomize

Replace the plain YAML source with a minimal Kustomize directory.

Repeat the breakpoint.

Identify the point where the repo server selects Kustomize rather than the plain-directory renderer.

### Third experiment: Helm

Use a tiny Helm chart.

Repeat again.

Inspect:

```text
Helm parameters
value files
release name
namespace
generated YAML
```

### Completion criterion

You should be able to explain:

```text
ApplicationSource
      ->
ManifestRequest
      ->
repo server
      ->
Git checkout / revision
      ->
renderer
      ->
manifest objects
      ->
ManifestResponse
```

---

## Chapter 5 — Repo-server caching and repeat requests

### Objective

Understand why two identical manifest requests do not necessarily perform identical work.

### Experiment

With the repo server under the debugger:

1. hard refresh the application;
2. record which expensive paths execute;
3. repeat without changing Git;
4. compare the second execution;
5. make a Git commit;
6. refresh again.

### Inspect

Look for:

```text
revision-derived keys
manifest cache keys
cache hit/miss state
source path
render options
resolved commit SHA
```

### Artificial change

For `learning-app` only, bypass one cache lookup or force an obvious cache-miss branch.

Do **not** generalize the change.

The learning objective is to compare execution traces:

```text
normal request
vs
forced miss
```

### Questions to answer

- Which work is avoided by caching?
- Which inputs participate in a cache key?
- Why would the same repository at a different commit need another entry?
- Why must generation options influence cache identity?
- What is cached in-process and what is shared through Redis?

---

# Part IV — Reconciliation: the heart of Argo CD

## Chapter 6 — Application Controller work queues and first reconciliation

### Objective

Follow an Application from event to reconciliation worker.

### Component under debugger

```text
argocd-application-controller
```

### Primary source area

Start with:

```text
controller/appcontroller.go
```

The controller structure contains important objects including:

```text
appRefreshQueue
appComparisonTypeRefreshQueue
appOperationQueue
projectRefreshQueue
appHydrateQueue
hydrationQueue
appInformer
appLister
appStateManager
stateCache
```

### First breakpoint

Locate:

```go
ApplicationController.Run(...)
```

Observe the worker goroutines that process the queues.

Set breakpoints in the refresh worker path, especially:

```text
processAppRefreshQueueItem
```

### Trigger

Change the Application:

```bash
kubectl annotate application learning-app \
  learning.argocd.local/trigger="$(date +%s)" \
  --overwrite
```

### Inspect

At the queue-processing breakpoint inspect:

```text
queue key
application namespace/name
cached Application
comparison level
project
refresh reason
worker state
```

### Artificial change

Add a narrow branch:

```text
if application == learning-app
    emit "LEARNING: refresh worker reached"
```

Then move the branch progressively deeper into the reconciliation call stack.

This lets you construct a call-path map from actual execution.

### Draw this after the experiment

Create your own simplified trace:

```text
Application event
    ->
informer handler
    ->
refresh requested
    ->
work queue
    ->
refresh worker
    ->
Application fetched
    ->
project fetched
    ->
state comparison
    ->
status update
```

Do not copy the diagram from documentation. Build it from the debugger.

---

## Chapter 7 — Desired state versus live state

### Objective

Understand the central Argo CD comparison.

### Primary source area

Focus on:

```text
controller/state.go
```

A key method in v3.5.1 is:

```go
func (m *appStateManager) CompareAppState(...)
```

### Prepare a controlled difference

Start synced:

```bash
argocd app sync learning-app
```

Then manually alter the live Deployment:

```bash
kubectl edit deployment <your-deployment>
```

For example change an environment variable or replica count.

### Breakpoint

Stop in:

```text
CompareAppState
```

Inspect:

```text
Application
AppProject
revision
ApplicationSource
target objects
live objects
sync status under construction
resource comparison results
health result
conditions
```

### Important distinction

Identify explicitly:

```text
TARGET = desired object generated from Git
LIVE   = object currently observed in Kubernetes
```

Track the same Deployment through the debugger so you are always comparing one known object.

### Artificial change

Choose one harmless field or annotation and temporarily cause the comparison path for `learning-app` to ignore it.

For example:

```text
learning.argocd.local/ignore-me
```

Demonstrate:

```text
live differs from Git
       |
       v
normally OutOfSync
       |
custom local comparison behaviour
       |
       v
Synced
```

without changing the actual live resource.

### Key questions

- Where are target objects obtained?
- Where are live objects obtained?
- Where does normalization happen?
- Where is the final `Modified` result produced?
- How is application-level sync state derived from resource-level results?
- Which code belongs to Argo CD and which belongs to GitOps Engine?

---

## Chapter 8 — Normalization, ignore-differences and diff strategies

### Objective

Go one level deeper than "Argo CD runs a diff."

### Experiments

Create several differences one at a time:

1. annotation difference;
2. field changed by a Kubernetes controller;
3. reordered list field where meaningful;
4. field configured under `ignoreDifferences`.

For each one:

```bash
argocd app diff learning-app
```

Stop before and after normalization.

### Inspect

For a chosen resource compare:

```text
raw target
raw live
normalized target
normalized live
predicted live state, where applicable
diff result
Modified flag
```

### Artificial change

Add a temporary custom normalization rule for a single demo annotation on `learning-app`.

Then remove it and implement the same outcome using Argo CD's supported ignore-difference configuration.

### Learning point

This exercise should make clear why:

```text
raw JSON inequality
```

is not equivalent to:

```text
Argo CD considers resource OutOfSync
```

---

## Chapter 9 — Argo CD versus GitOps Engine

### Objective

Understand the boundary between the Argo CD repository and the GitOps Engine dependency.

### Why this matters

When stepping through comparison or sync code, execution will eventually enter packages from GitOps Engine.

Without recognizing that boundary it can look as if important Argo CD code is "missing."

### Exercise

Start in:

```text
controller/state.go
```

Follow a comparison until the debugger enters GitOps Engine.

Record:

```text
last Argo CD function
first GitOps Engine function
data passed across the boundary
result returned
```

Repeat the exercise during sync.

### Build a responsibility table

Create your own notes with two columns:

```text
Argo CD
-------
Application CR orchestration
repo-server interaction
project/config handling
status persistence
controller queues
API surface

GitOps Engine
-------------
diff primitives
resource sync machinery
health/sync support used by Argo CD
```

Refine that table from what you actually observe.

### Artificial change

Do not edit the dependency yet.

Instead, change one input constructed by Argo CD immediately before a GitOps Engine call.

Observe how the downstream engine behaviour changes.

### Completion criterion

You should be able to answer:

> "If I find a bug in sync ordering or diff behaviour, how do I determine whether the fix belongs in Argo CD or GitOps Engine?"

---

# Part V — From OutOfSync to applied Kubernetes resources

## Chapter 10 — Manual sync and the operation queue

### Objective

Trace the full sync operation.

### Preparation

Create an intentional Git change so the Application becomes:

```text
OutOfSync
```

Do not auto-sync yet.

### Trigger

```bash
argocd app sync learning-app
```

### Breakpoints

Use the application controller.

Trace:

```text
Application operation appears
      ->
appOperationQueue
      ->
processAppOperationQueueItem
      ->
SyncAppState / sync construction
      ->
GitOps Engine sync context
      ->
resource tasks
```

### Inspect

At useful points inspect:

```text
Application.Operation
Application.Status.OperationState
requested revision
sync options
resource list
resource hooks
resource tasks
phase
wave
prune flag
dry-run state
```

### Artificial change

For `learning-app` only, add one extra diagnostic condition around a selected resource.

For example:

```text
when Kind == ConfigMap && Name == learning-config
    log the sync phase/wave/task state
```

### Observe operation state

While stopped in the debugger, compare:

```bash
kubectl get application learning-app -o yaml
```

with the in-memory `OperationState`.

Step until status is persisted.

### Completion criterion

You should understand the difference between:

```text
refresh/comparison work
```

and:

```text
sync/operation work
```

and why Argo CD uses distinct queues for them.

---

## Chapter 11 — Sync phases, waves, hooks and pruning

### Objective

Understand ordering rather than merely applying one resource.

### Demo repository

Expand the application to include:

```text
Namespace or ConfigMap
Deployment
Service
Job hook
one resource that can later be removed from Git
```

Add sync-wave annotations.

### Experiment 1: waves

Set breakpoints where sync tasks are ordered.

Inspect:

```text
resource identity
phase
wave
hook type
task ordering
```

Change one wave number in Git and rerun.

### Experiment 2: hook

Add a simple `PreSync` or `PostSync` Job.

Observe:

```text
hook recognition
hook task creation
execution phase
completion handling
```

### Experiment 3: prune

1. sync a ConfigMap;
2. remove it from Git;
3. refresh;
4. sync with pruning enabled.

Inspect how the resource transitions from:

```text
live + desired
```

to:

```text
live only
```

and eventually to a prune task.

### Artificial change

For `learning-app`, temporarily log the final ordered task list immediately before execution.

The output should include:

```text
phase
wave
kind
namespace
name
prune?
```

This is a powerful way to make sync planning visible.

---

# Part VI — API server, CLI, authentication and RBAC

## Chapter 12 — CLI request to API server

### Objective

Follow a normal CLI command into the API implementation.

### Component under debugger

```text
argocd-server
```

### Primary source area

Focus on:

```text
server/application/application.go
```

### First request

Use:

```bash
argocd app get learning-app
```

Set a breakpoint in the corresponding Application service method.

### Trace

Follow:

```text
CLI
  ->
gRPC request
  ->
API server
  ->
Application service
  ->
RBAC
  ->
Kubernetes / controller / repo-server-facing logic
  ->
gRPC response
  ->
CLI
```

### Inspect

```text
request message
context
claims
application name
namespace
project
RBAC name
Kubernetes client request
returned Application
```

### Artificial change

For `learning-app` only, add a harmless response-side diagnostic behaviour.

Prefer something visible but non-destructive, for example logging an extra summary field rather than changing persistent state.

### Second request: sync

Run:

```bash
argocd app sync learning-app
```

Stop in the API server before the Application operation is persisted.

Then switch the debugger to the application controller and observe the same operation being consumed.

### Key insight

The API server does not perform the entire deployment itself.

The request mutates or requests state that the reconciliation machinery then acts upon.

---

## Chapter 13 — Authentication and RBAC in a real request

### Objective

Understand authorization by observing an actual Application request.

### Important setup

Local development commonly disables auth for convenience.

For this chapter, enable authentication explicitly according to the current `Procfile` and development configuration.

### Experiment

Create two roles:

```text
reader
denied-user
```

Give the first permission to read `learning-app`.

Deny the second.

### Breakpoint

Stop around Application authorization enforcement.

In v3.5.1, Application API code uses the RBAC enforcer when deciding whether a caller may act on an Application.

Inspect:

```text
claims
resource type
action
RBAC resource name
project
application name
allow/deny result
```

### Compare two executions

```text
allowed request
denied request
```

Keep the breakpoint identical.

Write down which inputs differ.

### Artificial change

For `learning-app` only, temporarily add a diagnostic statement immediately before the enforcer call containing:

```text
subject
resource
action
object
```

Do not bypass security as the main experiment.

The point is to observe authorization decisions, not to create an insecure version of Argo CD.

### Completion criterion

You should understand where authentication ends and authorization begins, and where Application-specific object names enter RBAC evaluation.

---

# Part VII — ApplicationSet

## Chapter 14 — ApplicationSet generation and reconciliation

### Objective

Understand how one `ApplicationSet` becomes multiple `Application` resources.

### Component under debugger

```text
argocd-applicationset-controller
```

### Primary source area

Start with:

```text
applicationset/controllers/applicationset_controller.go
```

A central method in v3.5.1 is:

```go
func (r *ApplicationSetReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,
) (ctrl.Result, error)
```

### Start simple

Use a List generator:

```yaml
generators:
  - list:
      elements:
        - name: dev
          namespace: learning-dev
        - name: test
          namespace: learning-test
```

The template should create Applications named something like:

```text
learning-dev
learning-test
```

### Breakpoint 1

Stop at `Reconcile`.

Inspect:

```text
ctrl.Request
NamespacedName
ApplicationSet
generators
template
sync policy
status
```

### Breakpoint 2

Stop after generator parameters have been produced.

Inspect the parameter maps.

### Breakpoint 3

Stop when desired `Application` objects have been rendered.

Compare:

```text
generator input
      ->
parameter map
      ->
template
      ->
Application
```

### Breakpoint 4

Stop around create/update logic.

In v3.5.1 useful code is under methods such as:

```text
createOrUpdateInCluster
createInCluster
getCurrentApplications
```

### Artificial change

For generated Applications only, inject:

```yaml
metadata:
  annotations:
    learning.argocd.local/generated-by-debugger: "true"
```

Observe the annotation on the created `Application`.

### Cross-component exercise

Now stop the ApplicationSet controller and run it normally.

Debug the **ordinary application controller**.

Change the ApplicationSet.

Observe:

```text
ApplicationSet controller
      ->
new/updated Application CR
      ->
Kubernetes watch/informer
      ->
Application controller
      ->
normal reconciliation
```

This is one of the most important cross-component exercises in the plan.

---

## Chapter 15 — ApplicationSet update, delete and generator behaviour

### Objective

Understand that ApplicationSet is not simply a one-time templating command.

### Experiment 1: generator changes

Add a third List-generator element:

```text
stage
```

Observe the new desired Application.

### Experiment 2: template changes

Change a label or destination namespace in the template.

Inspect how the controller computes the desired update.

### Experiment 3: removed generator output

Remove one element.

Stop before the corresponding Application deletion/update decision.

Inspect:

```text
current Applications
desired Applications
ownership
differences
deletion candidate
```

### Artificial change

For the learning ApplicationSet only, log a compact reconciliation plan:

```text
CREATE learning-stage
UPDATE learning-dev
DELETE learning-test
```

Do not actually alter the deletion semantics.

### Optional extension

Repeat using a Git generator.

The goal is to see how the generator implementation changes while the later pipeline:

```text
parameters -> template -> desired Applications -> reconcile
```

remains conceptually similar.

---

# Part VIII — Live state, events and caches

## Chapter 16 — Live-state cache and Kubernetes watches

### Objective

Understand how Argo CD learns that Kubernetes has changed.

### Component

```text
argocd-application-controller
```

### Preparation

Start from a synced Application.

Then mutate a live object directly:

```bash
kubectl edit deployment <deployment>
```

### Debug path

Follow the live-resource update toward the controller.

A useful source area is the application controller's object-update handling.

Observe the path that decides which Application manages the changed object and requests a refresh.

### Inspect

```text
ObjectReference
group
version
kind
namespace
name
managed-by Application map
application key
comparison level
refresh request
operation requeue decision
```

### Artificial change

For a particular demo Deployment, add a local diagnostic branch reporting:

```text
resource changed
managed by application
comparison level requested
whether operation queue is also re-enqueued
```

### Self-heal experiment

Enable automated self-heal for the demo Application.

Manually modify the live Deployment again.

Observe:

```text
Kubernetes update
      ->
live-state observation
      ->
Application refresh
      ->
OutOfSync
      ->
self-heal sync
      ->
resource restored
```

### Key question

After this exercise you should be able to explain why Argo CD is not conceptually equivalent to a cron job that periodically runs:

```bash
git pull
kubectl diff
```

---

## Chapter 17 — Redis and application-state caches

### Objective

Understand where shared caching fits into the component architecture.

### Observe before changing anything

Use Redis inspection carefully in the disposable development environment.

At the same time, set breakpoints at Argo CD cache calls reached during:

```text
Application refresh
managed-resources retrieval
resource-tree retrieval
manifest generation
```

### Compare

Run:

```bash
argocd app get learning-app
argocd app get learning-app
```

and:

```bash
argocd app get learning-app --hard-refresh
```

Compare execution.

### Inspect

When you encounter cache operations, record:

```text
caller
key
value type
TTL if visible
cache hit/miss
what expensive operation would otherwise occur
```

### Artificial change

For `learning-app`, bypass one selected cache read.

Observe which downstream work now repeats.

Then revert.

### Mental model

Distinguish at least these concepts:

```text
Git/repository/manifest caching
application-state caching
resource-tree / managed-resource caching
live-state observation
```

Do not treat "Redis cache" as one homogeneous blob.

---

# Part IX — Controller mechanics

## Chapter 18 — Work queues, retries and requeues

### Objective

Understand how the application controller schedules work over time.

### Revisit

```text
appRefreshQueue
appComparisonTypeRefreshQueue
appOperationQueue
projectRefreshQueue
appHydrateQueue
hydrationQueue
```

### Controlled failure

Introduce a reversible failure in the demo Application.

Good examples:

```text
invalid Git revision
invalid manifest
temporary unreachable source
invalid destination namespace/project rule
```

Prefer failures that cannot damage anything outside the disposable local cluster.

### Debugger exercise

Follow:

```text
queue item
      ->
worker
      ->
error
      ->
retry / requeue
```

Inspect:

```text
queue key
retry count where exposed
rate limiter
requeue delay
comparison level
error state
Application conditions
```

### Artificial change

Reduce one relevant retry/requeue delay locally, only for the learning environment.

The exact change should be chosen after identifying the mechanism in the debugger.

Observe the difference.

Then revert.

### Questions

- Which work is event-driven?
- Which work is periodic?
- Why are refresh and operation queues separate?
- What happens if reconciliation fails?
- How does Argo CD prevent a transient failure from permanently losing work?
- Which updates cause immediate refresh versus delayed processing?

---

## Chapter 19 — Project refreshes and dependency fan-out

### Objective

Understand what happens when a shared object affects many Applications.

### Experiment

Create:

```text
learning-app-a
learning-app-b
```

Both use:

```text
learning-project
```

Modify the `AppProject`.

### Breakpoints

Follow the project informer/update path and inspect which Applications become candidates for refresh.

### Artificial change

Add a temporary diagnostic list:

```text
Project learning-project changed.
Applications affected:
- learning-app-a
- learning-app-b
```

### Learning point

This is a useful example of controller fan-out:

```text
one shared configuration object
      ->
multiple dependent Applications
      ->
multiple reconciliations
```

This pattern appears in real controller bugs involving stale state, race conditions and excess reconciliation.

---

# Part X — Scaling and distribution

## Chapter 20 — Application-controller sharding

### Objective

Understand how controller scaling changes ownership of work.

### Source focus

Explore:

```text
controller/sharding/
```

and the sharding fields used by `ApplicationController`.

### Start conceptually

Do not build a large cluster.

Instead, identify:

```text
what is being sharded
how a shard is chosen
where cluster/controller mapping is stored or computed
where the controller decides whether it may process an Application
```

### Debugger experiment

Use two registered clusters if practical, even if both are local/disposable.

Set a breakpoint at the shard-selection/ownership decision.

Inspect:

```text
cluster identity
shard number
controller identity
distribution method
mapping/cache
```

### Artificial change

For the learning environment, add a log line showing:

```text
cluster X -> shard Y
```

Do not rewrite the sharding algorithm.

### Completion criterion

You should understand the architectural reason sharding exists and the point where it changes controller behaviour.

---

# Part XI — Source hydration and commit server

## Chapter 21 — Hydration path and commit server

### Objective

Understand the newer hydration path without allowing it to obscure the traditional Argo CD architecture.

Do this chapter **after** the standard repo-server → compare → sync flow is comfortable.

### Source clues

The v3.5.1 application controller includes hydration-related queues and a commit-server client.

Find:

```text
appHydrateQueue
hydrationQueue
hydrator
commit client
```

### First task

Trace construction of these objects at controller startup.

Do not enable the feature immediately.

Understand where it attaches to the normal controller.

### Enable in the local environment

Use a disposable Git repository specifically for hydration experiments.

Follow the current Argo CD configuration for enabling Source Hydrator.

### Breakpoints

Stop at:

```text
hydration request creation
hydration queue processing
hydrated manifest preparation
commit-server request
commit result
Application status update
```

### Inspect

```text
dry source revision
hydrated target
commit metadata
commit SHA
repository destination
hydration status
```

### Artificial change

Change one harmless piece of generated commit metadata for the learning repository, such as a diagnostic string in the commit message.

Observe it reach the commit-server path.

### Architecture comparison

Write your own comparison:

```text
Traditional:
Git source
 -> repo server renders
 -> controller compares
 -> controller syncs to Kubernetes

Hydration:
dry source
 -> render/hydrate
 -> hydrated output written to Git
 -> deployment flow consumes hydrated state
```

The purpose is to understand the additional control-plane path, not to master every hydration feature.

---

# Part XII — Cross-process tracing

## Chapter 22 — Follow one Git commit through the whole system

### Objective

Create an executable mental model of Argo CD.

No new subsystem is introduced in this chapter.

### Preparation

Start with:

```text
learning-app = Synced + Healthy
```

Change one visible field in Git, for example:

```yaml
spec:
  template:
    metadata:
      labels:
        learning-version: "2"
```

Commit it.

### Trace 1 — repo server

Debug:

```text
argocd-repo-server
```

Trigger refresh.

Record:

```text
requested revision
resolved commit
source path
generated object
new label
```

### Trace 2 — application controller comparison

Restart with:

```text
argocd-application-controller
```

under the debugger.

Record:

```text
target object
live object
diff
sync status
health
Application status update
```

Confirm:

```text
OutOfSync
```

### Trace 3 — API server

Debug:

```text
argocd-server
```

Run:

```bash
argocd app sync learning-app
```

Record the API request and resulting operation.

### Trace 4 — operation worker

Return the debugger to:

```text
argocd-application-controller
```

Record:

```text
operation queue item
sync context
resource task
Kubernetes apply
operation state
```

### Trace 5 — live-state observation

After Kubernetes changes, observe the live-state event and new comparison.

Confirm:

```text
Synced
Healthy
```

### Required output from the chapter

Create a personal trace document containing one line per important boundary:

```text
1. Git commit:
2. Repo-server request:
3. Resolved SHA:
4. Generated manifest:
5. Controller queue key:
6. Target object:
7. Live object:
8. Diff result:
9. Application status:
10. Sync API request:
11. Operation queue key:
12. Sync task:
13. Kubernetes object:
14. Live-state event:
15. Final status:
```

If you can fill this out from debugger observations, you have moved from "I know Argo CD concepts" to "I can follow Argo CD source execution."

---

# Part XIII — Failure-oriented debugging

## Chapter 23 — Manifest-generation failure

### Objective

Learn to debug a realistic failure from symptom backwards.

### Create the failure

Commit invalid YAML or a deliberately broken Kustomize/Helm configuration.

### Start from the user-visible symptom

Run:

```bash
argocd app get learning-app
```

Observe the Application condition.

### Then trace backwards

Debug repo-server.

Find:

```text
renderer invocation
error
gRPC response/error
controller receipt
Application condition construction
```

### Artificial change

Improve the diagnostic locally for this one failure by attaching one additional piece of context.

For example:

```text
source path
renderer type
resolved revision
```

Do not redesign error handling.

### Goal

Practice the same workflow you would use on a real issue:

```text
user-visible symptom
      ->
status condition
      ->
controller error
      ->
RPC
      ->
repo-server
      ->
root cause
```

---

## Chapter 24 — Reconciliation/sync failure

### Objective

Debug a failure that occurs *after* desired manifests have been generated successfully.

### Controlled failure examples

Use a disposable resource that fails safely, for example:

```text
invalid immutable-field update
invalid Kubernetes object
failing PreSync hook
permission error in a deliberately restricted namespace
```

### Trace

Follow:

```text
sync task
      ->
Kubernetes operation
      ->
error
      ->
operation state
      ->
Application status
      ->
retry/requeue behaviour
```

### Inspect

```text
resource task
phase
wave
operation message
error
sync result
Application.Status.OperationState
```

### Completion criterion

You should be able to distinguish quickly between:

```text
manifest-generation problem
comparison problem
API/RBAC problem
sync-engine problem
Kubernetes API problem
controller retry problem
```

That classification ability is extremely useful when approaching real issues.

---

# Part XIV — Tests as executable documentation

## Chapter 25 — Read tests around code you now understand

### Objective

Use tests to deepen understanding after you already know the runtime path.

Do **not** start the learning plan by reading hundreds of tests.

### Pick four areas

Find focused tests around:

```text
repo-server manifest generation
Application comparison
ApplicationSet reconciliation
Application API/RBAC
```

For each:

1. read one test;
2. identify the production function it exercises;
3. run just that package/test;
4. add a temporary assertion;
5. make it fail;
6. restore it;
7. make a tiny production change;
8. update/add a test.

### Why now?

Tests become much easier to understand after the debugger has shown you the real structures.

You should be able to look at a fixture such as an `Application`, manifest response, or unstructured object and associate it with something you have already inspected in memory.

### Learning outcome

Move from:

```text
"I can modify running code"
```

to:

```text
"I can modify it and prove the behaviour with a focused test"
```

---

# Part XV — Capstone: a real bug-sized source change

## Chapter 26 — Find and implement one small real change

### Objective

Work without a pre-selected artificial edit.

The change does not need to be merged upstream.

The point is to reproduce the workflow of a real contribution.

### Scope rules

Choose something small enough that it has:

```text
one primary component
one understandable trigger
one reproducible behaviour
one or a few production functions
focused tests
```

Avoid as a first capstone:

```text
large UI work
major architectural proposals
deep authentication redesign
large multi-controller refactors
performance work requiring production-scale clusters
wide dependency upgrades
```

### Workflow

#### 1. Reproduce

Write the smallest reproduction you can.

Record:

```text
expected behaviour
actual behaviour
trigger
component likely responsible
```

#### 2. Find the owning process

Classify it:

```text
Git/rendering        -> repo-server
Application state    -> application-controller
CLI/API              -> argocd-server
ApplicationSet       -> applicationset-controller
sync primitive       -> perhaps GitOps Engine
hydration            -> controller / commit-server path
```

#### 3. Find the source path

Use:

```bash
rg "<relevant symbol or error text>"
```

Do not begin by browsing randomly through directories.

#### 4. Break at the suspected point

Run only that component under the debugger.

Reproduce the problem.

Confirm that the breakpoint hits.

#### 5. Inspect before changing

Write down:

```text
key inputs
unexpected value
branch taken
downstream result
```

#### 6. Make the smallest change

Prefer:

```text
one condition
one transformation
one error-handling correction
one missing edge case
```

over a refactor.

#### 7. Add/update tests

Prove:

```text
old code -> failing test
new code -> passing test
```

where practical.

#### 8. Run the real system

Repeat the original reproduction with the modified component.

#### 9. Explain the impact

You should be able to state:

```text
what changed
why it changed
which component owns it
which code path reaches it
what state crosses into/out of it
what tests prove it
what could regress
```

### Capstone success criterion

You are ready to start attempting small Argo CD issues when you can perform this workflow without needing a prewritten source-path walkthrough.

---

# Part XVI — Recommended study order

Use this order rather than jumping randomly among components.

## Phase A — establish the mechanics

```text
1. Build + local debugger
2. Executable/source-tree map
3. Application/AppProject
```

## Phase B — learn the core GitOps path

```text
4. Repo server
5. Repo-server caching
6. Controller work queues
7. Desired vs live
8. Diff/normalization
9. GitOps Engine boundary
10. Sync operation
11. Waves/hooks/pruning
```

This is the most important phase.

If you paused the plan after Phase B, you would already have a meaningful understanding of Argo CD internals.

## Phase C — learn surrounding control-plane components

```text
12. API server
13. Auth/RBAC
14. ApplicationSet
15. ApplicationSet update/delete
16. Live-state watches
17. Redis/cache
```

## Phase D — understand controller behaviour at scale

```text
18. Retries/requeues
19. Project fan-out
20. Sharding
```

## Phase E — newer/advanced paths

```text
21. Hydration + commit server
22. End-to-end cross-process trace
```

## Phase F — contribution readiness

```text
23. Manifest-generation failure
24. Sync failure
25. Tests
26. Real bug-sized capstone
```

---

# Part XVII — Debugger notebook template

For each chapter, keep a short entry using this format.

```markdown
## Experiment: <name>

### Trigger
What external action caused this path?

### Process
Which Argo CD process am I debugging?

### Breakpoint
File:
Function:

### Important inputs
- ...
- ...

### Important objects
- ...
- ...

### Calls I stepped through
1. ...
2. ...
3. ...

### RPC/process boundary
Did execution call another Argo CD process?
If so, which one?

### Artificial change
What did I change?

### Observed impact
What changed externally?

### What this code owns
One or two sentences.

### What this code does NOT own
One or two sentences.

### Questions remaining
- ...
```

This notebook will become more useful than passive source notes because every entry is connected to code that you personally executed.

---

# Part XVIII — Core source paths to become familiar with

These are navigation anchors, not a list to memorize.

```text
cmd/
    executable entry points and flags

pkg/apis/application/
    Application, AppProject, ApplicationSet-related API types

controller/appcontroller.go
    Application controller construction, queues, workers and event handling

controller/state.go
    desired/live comparison and sync-state orchestration

controller/sharding/
    application-controller distribution/sharding

reposerver/repository/repository.go
    repository access and manifest generation

server/application/application.go
    Application-facing API service and authorization paths

applicationset/controllers/application_set_controller.go
or
applicationset/controllers/applicationset_controller.go
    ApplicationSet reconciliation (verify exact filename in v3.5.1)

util/
    shared cache, Git, Kubernetes, settings, RBAC and other supporting utilities
```

When a path in this document disagrees with your checked-out source tree, trust the **v3.5.1 source tree** and locate the symbol with:

```bash
rg "SymbolName"
```

Examples:

```bash
rg "GenerateManifest"
rg "CompareAppState"
rg "processAppRefreshQueueItem"
rg "processAppOperationQueueItem"
rg "ApplicationSetReconciler"
```

Function names are more useful navigation anchors than copied line numbers.

---

# Part XIX — Things deliberately not emphasized

This plan intentionally avoids turning into a broad contributor handbook.

The following are secondary unless a real issue requires them:

```text
React UI internals
documentation tooling
release engineering
backport procedure
maintainer process
contributor etiquette
CI administration
Helm chart maintenance
large-scale production sizing
Argo CD installation patterns
generic GitOps tutorials
```

You need enough Argo CD *usage* knowledge to trigger source-code paths, but source execution remains the main subject.

Similarly, Helm and Kustomize are included because repo-server uses them; this is not intended to become a Helm/Kustomize course.

---

# Part XX — What you should know when the plan is complete

You should be able to explain, from source-code experience rather than just documentation:

## Architecture

- what each major Argo CD process owns;
- how the processes communicate;
- why repo-server is separate from the application controller;
- how ApplicationSet fits into ordinary Application reconciliation;
- where Redis/cache state fits;
- where GitOps Engine begins.

## Reconciliation

- how an Application event becomes queue work;
- how desired state is requested;
- how live state is obtained;
- how comparison produces sync/health state;
- how status is written back;
- how Kubernetes updates trigger later reconciliations.

## Sync

- how a sync request enters through the API;
- how it becomes an Application operation;
- how the operation queue consumes it;
- how resources become sync tasks;
- how waves/hooks/pruning affect execution;
- how the Kubernetes result becomes operation state.

## Debugging

- how to run one component from VS Code and the others normally;
- how to move across process/RPC boundaries;
- how to inspect Kubernetes unstructured objects;
- how to identify whether a bug belongs to repo-server, controller, API server, ApplicationSet or GitOps Engine;
- how to start with a visible symptom and trace to its source.

## Contribution readiness

You should be in a realistic position to take a small Argo CD issue, reproduce it, identify the responsible component, stop at the relevant source code, understand the local state, implement a focused change, test it and verify it end-to-end.

That does **not** mean the entire Argo CD codebase will be familiar. It means the codebase should no longer feel opaque, and you will have a repeatable method for investigating areas you have not previously studied.

---

# Reference material

The plan is based on Argo CD v3.5.1 and the current official developer architecture/debugging documentation as checked on **2026-08-12**.

Useful references:

- Argo CD releases:  
  https://github.com/argoproj/argo-cd/releases

- Running Argo CD locally:  
  https://argo-cd.readthedocs.io/en/latest/developer-guide/running-locally/

- Debugging a local Argo CD instance:  
  https://argo-cd.readthedocs.io/en/latest/developer-guide/debugging-locally/

- Component architecture:  
  https://argo-cd.readthedocs.io/en/stable/developer-guide/architecture/components/

- v3.5.1 Application controller:  
  https://github.com/argoproj/argo-cd/blob/v3.5.1/controller/appcontroller.go

- v3.5.1 application state/comparison code:  
  https://github.com/argoproj/argo-cd/blob/v3.5.1/controller/state.go

- v3.5.1 repo-server repository/manifest code:  
  https://github.com/argoproj/argo-cd/blob/v3.5.1/reposerver/repository/repository.go

- v3.5.1 Application API service:  
  https://github.com/argoproj/argo-cd/blob/v3.5.1/server/application/application.go

- v3.5.1 ApplicationSet controller:  
  https://github.com/argoproj/argo-cd/blob/v3.5.1/applicationset/controllers/applicationset_controller.go

- v3.5.1 controller sharding:  
  https://github.com/argoproj/argo-cd/blob/v3.5.1/controller/sharding/sharding.go

---

# Final principle

When you encounter an unfamiliar Argo CD subsystem, do not begin by trying to read all of it.

Use the same method again:

```text
find a real trigger
      ->
identify the owning process
      ->
find an entry point
      ->
set a breakpoint
      ->
inspect the real data
      ->
make a tiny controlled change
      ->
observe the consequence
      ->
step one layer deeper
```

That is the core learning technique this entire plan is designed to reinforce.
