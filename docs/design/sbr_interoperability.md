# SBR interoperability

## Introduction

Add an opt-in SNR mode for clusters where SBR owns the watchdog and storage health takes priority for remediation over API health.

## Motivation

Some storage backends do not support concurrent direct writes to a shared block volume, requiring SBR to use one volume per node. In such
environments, SBR's peer fencing does not work. To address this issue, we are proposing that SBR be used with NHC and SNR together to
achieve the use cases for block mode. The proposed setup is as follows:

1. NHC configured to create remediations when `SBRStorageUnhealthy=True` condition is set.
2. NHC is configured to use escalating remediations: first, SNR; then SBR.
3. SBR arms the hardware watchdog.
4. SNR does _not_ arm the watchdog and relies solely on software reboot. SNR also _disables_ its self fencing mechanism. 

This document proposes some changes needed to happen in SNR to enable this flow.

There are two scenarios where SNR comes into picture:

1) API Online, Storage Offline: In this case, SBR will report `SBRStorageUnhealthy=True` and trigger a remediation through NHC. NHC is configured to
create SNR. SNR agent then coordinates a _software_ reboot of the node. SNR keeps its watchdog disabled. This is one of the changes discussed in this doc.

> _Why SNR?_: SNR provides reboot orchestration and best-effort pod cleanup through the same CR. As of today, only SBR's peers can verify reboot. Without shared RWX volume, we lose that ability. Therefore, SNR is required. NHC may subsequently escalate to SBR if storage remains unhealthy and software reboot fails. For SBR and SNR to co-exist with SBR holding the watchdog, SNR needs to *disable* its watchdog.


2) API Offline, Storage Online:  In this case, we do _not_ want to reboot as we want to prioritize only storage failures. However, SNR's peer checks
will reboot a node in case a local api failure is determined. We want to change this behavior which is the second point discussed in this document.

_Note that rest of the scenarios are out of the scope of this document because SNR is not affected in those cases_

## Goals

- Propose a new mode of working `SBRAssisted` for SNR in which watchdog and autonomous self-fencing are disabled.
- Retain NHC-authorized software reboot through an SNR CR.

## Non-Goals

- Alter any of the existing SNR logic in standard mode of working or change the default mode of working.

## Design

### 1. Introduce _spec.mode_ field on SNRConfig

Add `spec.mode: Standard | SBRAssisted` to `SelfNodeRemediationConfig`.

An omitted value for this field will default to `Standard` in which SNR will continue to work as-is.

Location: [api/v1alpha1/selfnoderemediationconfig_types.go][config-api],
`SelfNodeRemediationConfigSpec` and `NewDefaultSelfNodeRemediationConfig`.
Define `OperatingMode` with `StandardMode` and `SBRAssistedMode` constants.

```go
type SelfNodeRemediationConfigSpec struct {
    // ...
    // +kubebuilder:default=Standard
    // +kubebuilder:validation:Enum=Standard;SBRAssisted
    // +optional
    Mode OperatingMode `json:"mode,omitempty"`
}
```

| Setting or behavior | `Standard` | `SBRAssisted` |
| --- | --- | --- |
| Watchdog initialization and feeding | Existing behavior | Watchdog never initialized |
| API-loss / peer-isolation fencing | Existing behavior | Disabled |
| `isSoftwareRebootEnabled` | Existing behavior | Must be `true` |
| `safeTimeToAssumeNodeRebootedSeconds` | Existing behavior | Existing assumed-reboot timer; cleanup uses ordinary Pod deletion only |
| API health-check and control-plane diagnostic settings | Existing behavior | Inactive |
| Peer dial/request timeouts and `hostPort` | Existing behavior | Unchanged; not used for reboot confirmation |

In [SNRConfigValidator.ValidateCreate / ValidateUpdate][config-webhook], reject
`SBRAssisted` with software reboot disabled. Reject mode changes while any SNR request
still owns a finalizer. Enforce enum and field relationships in the CRD as well;
controllers repeat safety checks rather than relying only on admission.

To reduce friction, we can also add a new mutating webhook to automatically set
`isSoftwareRebootEnabled` to `true` in case its omitted. If its explicitely set 
to False, we should still fail.

Keep `SelfNodeRemediationSpec` unchanged.

Add only the execution mode to [SelfNodeRemediationStatus][request-api]:

```go
type SelfNodeRemediationStatus struct {
    // ...
    ExecutionMode OperatingMode `json:"executionMode,omitempty"`
}
```

The manager records `executionMode` before request preparation.
This preserves request behavior across configuration changes and controller restarts.


### 2. Disable autonomous fencing and reuse the existing software rebooter

Pass `SNR_MODE` through
[SelfNodeRemediationConfigReconciler.syncConfigDaemonSet][config-controller] and
[install/self-node-remediation-deamonset.yaml][daemonset]. Missing means
`Standard`; an unknown value fails startup. Require software reboot to be enabled
in `SBRAssisted`, including when admission has been bypassed.

Location: [cmd/main.go][main], `initSelfNodeRemediationAgent`.

```go
func initSelfNodeRemediationAgent(mgr manager.Manager) {
    // ... parse and validate mode ...
    var wd watchdog.Watchdog
    var rebooter reboot.Rebooter
    if mode == v1alpha1.SBRAssistedMode {
        rebooter = reboot.NewWatchdogRebooter(nil, ctrl.Log.WithName("rebooter"))
    } else {
        // ... existing watchdog setup, registration and rebooter selection ...
    }
    // ... existing annotation updater and shared setup ...
    if mode != v1alpha1.SBRAssistedMode {
        // ... existing controlplane.NewManager and apicheck registration ...
    }
    // ... existing CR reconciler and peer server registration ...
}
```

In `SBRAssisted`, never construct, open, feed, or close a watchdog. Do not register
the API checker or control-plane isolation manager. This disables autonomous
fencing on both workers and control-plane nodes while retaining CR-triggered
software reboot.

[NewWatchdogRebooter(nil, log) and Reboot()][rebooter] already select the software
reboot path. Reuse that implementation and its command list; no separate
software rebooter or command-execution framework is introduced.
A nil return from `Reboot()` does not confirm reboot.
Failed, stuck, or ineffective commands do not stop the manager's timer.
An already-dispatched command can finish after NHC timeout.

Keep `rebootIfNeeded()` and `didIRebootMyself()` in [selfnoderemediation_controller.go][controller].
The existing uptime check suppresses another reboot after an apparent host restart.
It does not write status or gate manager completion.
The existing clock assumptions remain unchanged.

Keep existing peer-server, certificate, and DaemonSet mount plumbing unchanged.
No new peer RPC or network-policy rule is required.
The existing
[NewAnnotationUpdater / UpdateNodeAnnotations][annotations] already handles a
nil watchdog: software reboot capability remains true and SNR watchdog timeout
is zero. That annotation does not establish SBR watchdog readiness.

**Example:** API access and all peer connections fail while storage remains
healthy. `apicheck.Start()` and the control-plane isolation branches do not run;
SNR issues no reboot. SBR continues feeding its watchdog.

### 3. Stop further attempts when NHC stops the request

Keep the stopped-request branch in `ReconcileManager()`.
Add the missing NHC stop check to `ReconcileAgent()` in [selfnoderemediation_controller.go][controller].
Scope this change to `SBRAssisted`.

```go
func (r *SelfNodeRemediationReconciler) ReconcileAgent(
    ctx context.Context, req ctrl.Request,
) (ctrl.Result, error) {
    // ... existing fetch, deletion and target checks ...
    if r.executionMode(snr) == v1alpha1.SBRAssistedMode && r.isStoppedByNHC(snr) {
        return ctrl.Result{}, nil
    }
    // ... existing phase dispatch; refresh authorization before reboot ...
}
```

This uncovers a gap in following scenario: SNR fails to reboot the node due to a transient issue, NHC moves onto SBR after timeout, SNR retries reboot while SBR is attempting to reboot too. 
It can lead to double reboots. This is especially important in SBRAssisted mode because there isn't a watchdog and software reboots can fail more often than watchdog.

> Note: This is an existing problem in SNR, so it might be worth fixing it for both Standard and SBRAssisted mode.

In `SBRAssisted`, read the CR directly from the API server immediately before each reboot attempt.
Do not use the cached CR for this check.
The CR UID, target node, and mode must match the pending attempt.
The phase must still be `Pre-Reboot-Completed`.
A timeout annotation, deletion timestamp, missing CR, or failed API read prevents the attempt.
These checks prevent new commands only.
They cannot cancel an already-issued reboot command.

The two timers serve different purposes:

| Timer | Effect after expiry |
| --- | --- |
| SNR's assumed-reboot timer | The manager advances an active request to `Reboot-Completed`. Best-effort workload cleanup follows. |
| NHC's SNR stage timeout | [NHC][nhc-controller] can advance to the configured SBR stage. Storage must remain unhealthy. Storm protection must permit remediation. |


### 4. Use ordinary best-effort cleanup after assumed completion

In case if software reboot fails, and NHC falls back to using SBR to reboot a node via watchdog. The forced cleanup of pods can be problematic.
There could be a small time window where after forced cleanup, workloads may keep running on the node being remediated as well as on another node (due to force delete).
We want to avoid that. Therefore, `SBRAssisted` must bypass SNR's forced workload removal paths.
Do not apply the out-of-service taint. Kubernetes requires shutdown confirmation before that taint. See [Kubernetes shutdown requirements][shutdown].

Location: [selfnoderemediation_controller.go][controller],
`handleRebootCompletedPhase()` and `getRuntimeStrategy()`.
For `SBRAssisted`, route `Automatic` and `ResourceDeletion` to a new `attemptWorkloadCleanup()` helper.
Prevent `Automatic` from selecting the out-of-service path in this mode.
Reject explicit `OutOfServiceTaint` through [SNRValidator][request-webhook] and [SNRTemplateValidator][template-webhook].
Repeat that validation in the reconciler.
Keep `Standard` strategy selection unchanged.

```go
func (r *SelfNodeRemediationReconciler) handleRebootCompletedPhase(
    ctx context.Context, node *corev1.Node, snr *v1alpha1.SelfNodeRemediation,
    rmNodeResources removeNodeResources,
) (ctrl.Result, error) {
    if r.executionMode(snr) == v1alpha1.SBRAssistedMode {
        return r.attemptWorkloadCleanup(ctx, node, snr)
    }
    // ... existing resource removal and success handling ...
}
```

`attemptWorkloadCleanup()` checks the current CR and target Node.
It makes one pass with a 30-second context deadline.
Select workload pods bound to the node and created before the SNR CR.
Exclude DaemonSet pods, mirror pods, and SNR/SBR agents.
Use UID preconditions and normal termination grace for deletion.
Retain finalizers.
The creation-time cutoff excludes replacement pods on later retries.
No persistent cleanup-target list is added.
Recheck the request's stop/deletion state before each delete.
An in-flight delete cannot be recalled.

Do not wait for node attachments or call `isResourceDeletionExpired()`.
Kubelet and CSI handle normal termination and detach.
Record failed or skipped cleanup through events and condition messages.
Do not retry solely to guarantee workload removal.
A controller restart or failed status write may repeat ordinary deletion.

After this pass, advance to `Fencing-Completed` with `Processing=False` and `Succeeded=True`.
The condition message must identify assumed reboot completion and best-effort cleanup.
Success does not confirm host reboot, Pod removal, volume detach, or storage recovery.
An observed NHC stop or deletion takes precedence over this update.
Status write failures retain the existing retry behavior.

### 5. Upgrade/downgrade concerns

- Already-started requests without `executionMode` keep `Standard` behavior.
  Persist the mode for new requests before preparation; a local/request mode
  mismatch prevents the agent from acting.
- Upgrade CRDs, manager, and all selected agents before enabling `SBRAssisted`.
  Old managers can still select forced cleanup.
  Do not downgrade while requests in the new mode remain.
- Pause NHC during the mode transition, finish outstanding SNR requests, confirm
  SNR has released the watchdog, then arm SBR. Resume NHC after its storage health
  reporting is ready. This is a maintenance operation, not a live watchdog
  ownership transfer.
- Regenerate CRDs, deepcopy code, affected deployment/OLM manifests, and config
  samples. No protobuf or peer NetworkPolicy changes are required.
  
## Test Plan

### Unit and controller tests

| Area / code | Cases and required result |
| --- | --- |
| Config API and `SNRConfigValidator` | Omitted mode remains Standard. Reject unknown modes and SBRAssisted with software reboot disabled. Reject mode changes while requests own finalizers. |
| `syncConfigDaemonSet`, `initSelfNodeRemediationAgent` | Render mode env. Never construct a watchdog or register API/control-plane fencing in SBRAssisted. Retain CR handling and the nil-watchdog software path. |
| `SelfNodeRemediationStatus` | Persist execution mode before preparation. Only the manager writes phases and conditions. |
| `didIRebootMyself`, `rebootIfNeeded` | Preserve the uptime check and its clock assumptions. A positive check suppresses reboot without status writes. Read errors prevent dispatch. |
| `prepareReboot`, `waitForNodeRebooted` | Retain the existing timer. Deadline expiry advances the phase without agent reports, including after restart. |
| Existing rebooter integration | Failed, stuck, and ineffective commands do not block manager completion. Command return values do not prove reboot. |
| `ReconcileAgent`, `ReconcileManager` | Reject stopped, deleted, replaced, or mismatched CRs before dispatch. Fresh reads detect stale cached phases. In-flight commands remain possible. |
| `getRuntimeStrategy`, request/template validation | SBRAssisted never selects out-of-service cleanup. Reject explicit OutOfServiceTaint, including admission bypass. Preserve Standard behavior. |
| `attemptWorkloadCleanup` | Preserve grace, finalizers, newer pods, agents, and probe storage. Never force-delete Pods or delete VolumeAttachments. Cleanup errors do not block completion. |
| Lifecycle and compatibility | Restart in each phase. Retry failed status writes. Release SNR taints/finalizers on deletion. Preserve SBR cordons. Missing execution mode on started requests remains Standard. |

Use the existing controller envtest suite under `internal/controller/tests`.
Use webhook tests under `internal/webhook/v1alpha1`.
Add coverage for cleanup routing, stop checks, and startup wiring.
Run `make manifests generate` and `make test` for the implementation.
This document does not claim implementation tests have run.

### Cluster integration tests

Capture host boot IDs, host uptime, CR timestamps, and phase changes in a disposable cluster.
Use boot IDs only as independent test evidence.
Also capture watchdog ownership, Pod UIDs, and VolumeAttachments.
A container restart is not a host reboot.

| Scenario | Expected result |
| --- | --- |
| API available, storage lost | NHC creates one SNR CR. Agent attempts software reboot. Manager advances after its deadline. Cleanup remains ordinary and best-effort. |
| Software reboot fails or hangs | Manager can finish SNR without confirmed reboot. NHC can escalate despite Succeeded=True when storage remains unhealthy and gates permit. |
| Storm protection blocks remediation | SBR escalation waits, including after successful SNR completion. No forced cleanup occurs during that wait. |
| Storage recovers before escalation | NHC can end remediation without creating an SBR CR. No later reboot guarantee exists. |
| API lost, storage available | No autonomous reboot on workers or control-plane nodes, including complete peer isolation. |
| API and storage lost before any CR | SBR alone handles reboot. No SNR recovery request or guaranteed workload cleanup follows. |
| Target loses API during an active request | No new command without authorization. Manager completion remains independent of agent reports. Already-issued commands can finish. |
| Manager or NHC loses API | Writes pause for the affected controller. Recovery resumes current CR state. Expired SNR deadlines still permit assumed completion. |
| Storage remains down after reboot | SNR completion does not establish storage recovery. NHC escalation remains subject to its timeout and storm gates. |
| Cleanup fails or pods remain terminating | SNR records best-effort completion. No forced deletion, out-of-service taint, or direct VolumeAttachment deletion occurs. |
| Node never returns | Manager can complete SNR through its timer. Pending Pod termination does not prove reboot or detach. |
| Restart, deletion and mode transition | Stopped requests remain stopped. Completed requests remain terminal. Finalizers and taints retain normal cleanup. Watchdog ownership never overlaps. |

[config-api]: ../../api/v1alpha1/selfnoderemediationconfig_types.go
[request-api]: ../../api/v1alpha1/selfnoderemediation_types.go
[config-webhook]: ../../internal/webhook/v1alpha1/selfnoderemediationconfig_webhook.go
[request-webhook]: ../../internal/webhook/v1alpha1/selfnoderemediation_webhook.go
[template-webhook]: ../../internal/webhook/v1alpha1/selfnoderemediationtemplate_webhook.go
[config-controller]: ../../internal/controller/selfnoderemediationconfig_controller.go
[controller]: ../../internal/controller/selfnoderemediation_controller.go
[daemonset]: ../../install/self-node-remediation-deamonset.yaml
[main]: ../../cmd/main.go
[annotations]: ../../internal/utils/annotations.go
[rebooter]: ../../internal/reboot/rebooter.go
[nhc-controller]: ../../../node-healthcheck-operator/internal/controller/nodehealthcheck_controller.go
[shutdown]: https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/#non-graceful-node-shutdown-handling
