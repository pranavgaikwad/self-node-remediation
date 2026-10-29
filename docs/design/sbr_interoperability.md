# SBR interoperability

## 1. Introduction

Add an opt-in SNR mode for clusters where SBR owns the watchdog and storage health takes priority for remediation over API health.

## 2. Motivation

Some storage backends do not support concurrent direct writes to a shared block volume, requiring SBR to use one volume per node. In such
environments, SBR's peer fencing does not work. To address this issue, we are proposing that SBR be used with NHC and SNR together to
achieve the use cases for block mode. The proposed setup is as follows:

1. SBR reports `SBRStorageUnhealthy=True` after sustained storage failure.
2. NHC applies its observation delay and health limits.
3. NHC creates an SNR CR for the node.
4. SNR attempts software reboot with its own watchdog disabled.
5. NHC can stop the SNR stage and escalate through a separate SBR CR.
6. SBR performs authorized fencing if no reboot is confirmed for that episode.
7. SNR starts or continues cleanup after confirmed fencing, including after an NHC stop.
8. SBR suppresses another reboot for that episode while SNR completes cleanup.

This document proposes some changes needed to happen in SNR to enable this flow.

There are two scenarios where SNR comes into picture:

1) API Online, Storage Offline: In this case, SBR will report `SBRStorageUnhealthy=True` and trigger a remediation through NHC. NHC is configured to
create SNR. SNR agent then coordinates a _software_ reboot of the node. SNR keeps its watchdog disabled. This is one of the changes discussed in this doc.

> _Why SNR?_: SNR provides reboot orchestration and best-effort pod cleanup through the same CR. As of today, only SBR's peers can verify reboot. Without shared RWX volume, we lose that ability. Therefore, SNR is required. NHC may subsequently escalate to SBR if storage remains unhealthy and software reboot fails. For SBR and SNR to co-exist with SBR holding the watchdog, SNR needs to *disable* its watchdog.


2) API Offline, Storage Online:  In this case, we do _not_ want to reboot as we want to prioritize only storage failures. However, SNR's peer checks
will reboot a node in case a local api failure is determined. We want to change this behavior which is the second point discussed in this document.

_Note that rest of the scenarios are out of the scope of this document because SNR is not affected in those cases_

## 3. Goals

- Propose a new mode of working `SBRAssisted` for SNR in which watchdog and autonomous self-fencing are disabled.
- Introduce a shared host file on nodes that will be used by SBR to avoid double reboots.
- Retain NHC-authorized software reboot through an SNR CR.

## 4. Non-Goals

- Alter any of the existing SNR logic in standard mode of working or change the default mode of working.

## 5. Design

### 5.1 Configuration API

The configuration API adds `spec.mode` to `SelfNodeRemediationConfig`, with `Standard` and `SBRAssisted` as the permitted values.
If the user omits this field, SNR uses `Standard` mode to preserve the existing behavior.
The shared implementation in `medik8s/common` will create a new directory for persistent state.

The changes belong in `SelfNodeRemediationConfigSpec` and `NewDefaultSelfNodeRemediationConfig`, in [selfnoderemediationconfig_types.go][config-api].
The following example shows the proposed type and field:

```go
// +kubebuilder:validation:Enum=Standard;SBRAssisted
type OperatingMode string

const (
	StandardMode    OperatingMode = "Standard"
	SBRAssistedMode OperatingMode = "SBRAssisted"
)

type SelfNodeRemediationConfigSpec struct {
	// Existing fields omitted.
	Mode OperatingMode `json:"mode,omitempty"`
}
```

The selected mode controls the behavior described in this table:

| Setting or behavior | `Standard` | `SBRAssisted` |
| --- | --- | --- |
| Watchdog initialization and feeding | SNR keeps the existing behavior. | SNR must not initialize its watchdog. |
| Fencing after API or peer isolation | SNR keeps the existing behavior. | SNR disables this fencing on workers and control-plane nodes. |
| `isSoftwareRebootEnabled` | SNR keeps the existing behavior. | The value must be `true`. |
| Shared host directory | This mode adds no directory requirement. | SNR requires a writable directory. |
| Reboot confirmation | SNR keeps the existing behavior. | SNR requires changed boot IDs when they are available. Otherwise, SNR reports an assumed reboot without confirming fencing. |
| Cleanup strategies | SNR keeps the existing behavior. | SNR keeps `Automatic`, `ResourceDeletion`, and `OutOfServiceTaint`. |
| NHC stop | SNR keeps the existing behavior. | SNR stops further reboot attempts but remains responsible for cleanup after confirmed fencing. |
| API diagnostics and isolation checks | SNR keeps the existing behavior. | These checks cannot trigger autonomous fencing. |
| Peer communication | SNR keeps the existing behavior. | SNR requires no new peer protocol for reboot evidence. |

The existing default for `isSoftwareRebootEnabled` remains `true`, so this field does not require another mutating webhook.
In `SBRAssisted` mode, admission must reject an explicit value of `false`.
The design keeps `SelfNodeRemediationSpec` unchanged.

[SNRConfigValidator.ValidateCreate and ValidateUpdate][config-webhook] must validate the mode and reject mode changes while SNR requests still have finalizers.
Controllers and agents must repeat the applicable checks if a request bypasses admission.
These checks preserve the mode requirements throughout request processing.

`SelfNodeRemediationConfigStatus` currently has no fields, so reports of degraded operation require an API extension.
The implementation must also define which component writes those reports.
Section 6.4 leaves the form of reports for individual nodes open.

### 5.2 Agent startup and deployment

The [config controller][config-controller] must pass `SNR_MODE` through `syncConfigDaemonSet()` to the [agent DaemonSet template][daemonset].
At startup, the agent uses `Standard` if the mode is absent and stops startup if the mode is unknown.
The following change to [initSelfNodeRemediationAgent()][main] keeps the existing software reboot implementation:

```go
func initSelfNodeRemediationAgent(mgr manager.Manager) {
	// Existing setup and mode validation omitted.
	var wd watchdog.Watchdog
	var rebooter reboot.Rebooter
	if mode == v1alpha1.SBRAssistedMode {
		// Initialize and check local state before registering remediation.
		rebooter = reboot.NewWatchdogRebooter(nil, ctrl.Log.WithName("rebooter"))
	} else {
		// Existing watchdog setup and rebooter selection omitted.
	}
	// Existing shared setup omitted.
	if mode != v1alpha1.SBRAssistedMode {
		// Existing controlplane.NewManager and apicheck registration omitted.
	}
	// Register the CR reconciler and existing peer server.
}
```

In `SBRAssisted` mode, SNR must not construct, open, feed, or close a watchdog.
The agent must also omit registration of the API checker and control-plane isolation manager as autonomous reboot triggers.
Software reboot through an authorized CR remains available on workers and control-plane nodes through [NewWatchdogRebooter(nil, log) and Reboot()][rebooter].
A successful command return does not, by itself, confirm that the host rebooted.

The agent uses the shared host directory that `medik8s/common` creates for persistent state.
The deployment must include any necessary changes to admission permissions, including OpenShift SCC permissions.
Existing peer communication and certificate handling remain unchanged because this evidence mechanism requires no new peer RPC or peer NetworkPolicy rule.

The existing [annotation updater][annotations] supports a nil SNR watchdog and reports a zero watchdog timeout in that case.
That value describes SNR's watchdog state and does not show that SBR's watchdog is ready.

### 5.3 Local state files

SBR and SNR use separate persistent files in the host directory that `medik8s/common` creates.
The common library defines fixed paths and supplies the file helpers and schema definitions.
Users cannot configure these paths, and each agent must follow the write ownership shown below.

| File | Writer | Readers |
| --- | --- | --- |
| SNR state file | The SNR agent on the node writes this file. | Both agents on the node can read this file. |
| SBR state file | The SBR agent on the node writes this file. | Both agents on the node can read this file. |

Each agent writes only its own file and reads the other agent's file to coordinate remediation.
Neither agent may modify, replace, or delete the other agent's file.
Because the agents write separate files, they require no shared write lock or combined read-modify-write operation.

SNR records these details in its own file:

- The agent version, startup time, shutdown marker, and disabled watchdog state.
- The effective configuration and its source generation.
- The reboot source, method, reason, and attempt time.
- The SNR kind, name, UID, generation, and target node identity.
- The associated NHC identity, when available.
- The host boot ID before the attempt and the observed boot ID after restart.
- The attempt state, observed completion, and any error that explains an unavailable boot ID.
- The SNR CR's phase and conditions, with the request identity, node identity, source resource version, and observation time.

> Note that exact data stored in the file will change as we go through implementation. Above list is just a rough idea of what to store in that file.

Before SNR issues a software reboot command, it must write a record of the pending attempt.
This record must describe an attempt without claiming that the reboot completed.
Reboots requested through SNR do not consume SBR's autonomous reboot allowance, so an exhausted allowance must not cause SNR to reject a request.

At startup, SNR must examine its previous state before it replaces the recorded boot ID.
An unchanged boot ID shows an agent restart, while a changed boot ID provides evidence of a host reboot after the recorded attempt.
SNR must associate this evidence with the pending request before it publishes the current state.
The agent must not modify another provider's CR or completion status.

A changed boot ID shows that the host reset, but it does not identify the cause of that reset.
SNR must therefore record the observed reset without claiming that a particular command caused it.

The SNR manager reports cleanup through the existing CR phase and conditions because it cannot write directly to another node's host directory.
The target agent copies these fields into the SNR state file so that SBR can inspect them.
Cleanup progress comes from these fields and the accepted reboot evidence, as described in section 5.4.
Neither the CR nor the SNR state file introduces a separate cleanup phase.

The copied progress is a cached report and cannot serve as fencing evidence or independent permission to reboot.
Missing or stale progress must not imply that cleanup completed.
Without a confirmed reboot, unfinished cleanup must not block authorized SBR fencing.
After a confirmed reboot, missing progress must not justify another reboot for the same episode.
The manager can continue eligible cleanup while the agent is unavailable and cannot update its state file.

#### State-file failure

SNR uses the same retry approach as SBR, with compatible settings supplied through the common implementation.
The source proposal specifies a five-second timeout for each file operation and a maximum of three attempts.
If the directory or SNR state file remains unusable, SNR must complete this sequence:

1. Log the failed operation and its error.
2. Report degraded operation on the SNR config when API access permits.
3. Emit an event when API access permits.
4. Prevent new reboot commands if SNR cannot record their attempts.
5. Exit so that Pod readiness and restart behavior expose the failure.

SNR must not disarm SBR's watchdog during this sequence.
An agent exit cannot cancel a software reboot command that already started, and a file failure cannot establish reboot completion.
The exit does not transfer cleanup ownership or prevent the manager from continuing cleanup based on previously accepted fencing evidence.

If SNR's file is corrupt, SNR must use the common quarantine and recovery procedure on that file only.
It must report any lost history, while an incompatible schema remains a configuration error.
An incompatible schema does not permit either agent to discard the other agent's history or its own history.
On agent restart, a corrupt file should be handled by the SNR agent (it will be rewritten with fresh data losing any history).

If SBR's file is missing, unreadable, corrupt, or incompatible, SNR must report the failure and retry the read.
SBR owns recovery of that file, so SNR must not repair or quarantine it.
Unavailable SBR records cannot establish fencing or cleanup completion.
SNR can continue updating its own file, while the manager can continue cleanup based on previously accepted fencing evidence.

### 5.4 Request status and evidence reporting

[SelfNodeRemediationStatus][request-api] must include the execution mode, reboot evidence, and method used to report reboot completion.
The existing phase and conditions continue to report cleanup progress, without another cleanup field.
The following example shows the proposed API fields:

```go
type RebootEvidence struct {
	ObservedGeneration      int64     `json:"observedGeneration"`
	NodeUID                 types.UID `json:"nodeUID"`
	BeforeBootID            string    `json:"beforeBootID,omitempty"`
	ObservedBootID          string    `json:"observedBootID,omitempty"`
	BootIDUnavailableReason string    `json:"bootIDUnavailableReason,omitempty"`
}

type SelfNodeRemediationStatus struct {
	// Existing fields omitted.
	ExecutionMode          OperatingMode   `json:"executionMode,omitempty"`
	RebootEvidence         *RebootEvidence `json:"rebootEvidence,omitempty"`
	RebootCompletionMethod string          `json:"rebootCompletionMethod,omitempty"`
}
```

The manager records `executionMode` before request preparation so that configuration changes and controller restarts preserve the request's behavior.
The target agent then reads evidence from the SNR and SBR state files and reports matching evidence on its own SNR CR.
This reporting continues after an NHC stop and can describe a reset during the SBR stage of the same remediation episode.
The agent must not attribute an unrelated reboot or evidence from another episode to the request.

The controller cannot read a host directory on another node, so the API report carries evidence from the target agent to the manager.
This mechanism requires no new peer protocol.
Within the CR status, the agent writes only evidence fields, while the manager owns execution mode, phase, completion method, and conditions.
Both components must use patches limited to their own fields and retry conflicts without replacing the other component's fields.

Before accepting evidence, the manager must check the CR UID, generation, target node identity, and recorded attempt.
The matching attempt can belong to SBR if SNR never issued a software reboot command.
Section 6.2 leaves the association between these separate requests open.

The manager uses `BootID` or `Timer` for `rebootCompletionMethod` and keeps this distinction visible in condition messages and events.
A report based on the timer must not create a false observed reboot in the SNR state file.
For prepared requests in `SBRAssisted` mode, the existing phases control cleanup as follows:

| Existing phase and evidence | Cleanup behavior |
| --- | --- |
| The phase is `Pre-Reboot-Completed`, without accepted fencing evidence. | SNR waits for evidence before cleanup can start. |
| The phase is `Reboot-Completed`, with accepted fencing evidence. | SNR executes or retries the selected cleanup strategy. |
| The phase is `Fencing-Completed`, with `Succeeded=True`. | Cleanup has met the selected strategy's completion criteria, so SNR can perform eligible request recovery. |

SNR may advance to `Reboot-Completed` only after it accepts fencing evidence under section 5.5.
Timer expiry alone cannot advance that phase in assisted mode.
The selected strategy's result controls the transition to `Fencing-Completed` through the existing phase dispatch in [remediateWithResourceRemoval()][controller].

`WaitingForFencing` becomes a reason on the existing conditions, without adding another phase.
While SNR waits, it must report `Processing=True` and `Succeeded=Unknown` with this reason.
Condition messages and events must explain the missing evidence and required operator intervention.
[updateConditions()][controller] must update reasons and messages even when condition status values do not change.

Once accepted evidence permits cleanup, the manager uses the normal processing reason until cleanup completes with the existing success reason.
The agent copies the phase and conditions without advancing them independently.
A missing report must not imply that cleanup completed.

### 5.5 Reboot detection and completion

The changes affect `didIRebootMyself()`, `rebootIfNeeded()`, and `waitForNodeRebooted()` in the [SNR controller][controller].
These functions must use the host boot ID from `/proc/sys/kernel/random/boot_id`, rather than a container or Pod identity.
For the matching pending attempt, SNR applies the following decision table:

| Boot-ID evidence | Timer state | Completion decision |
| --- | --- | --- |
| Both IDs are available and different. | The timer state has no effect. | SNR records observed completion through `BootID`. After evidence validation, it advances to `Reboot-Completed`. |
| Both IDs are available and equal. | The timer state has no effect. | SNR keeps the reboot pending because timer expiry cannot override the available evidence. |
| Boot IDs are unavailable. | The existing deadline has not expired. | SNR keeps the reboot pending. |
| Boot IDs are unavailable. | The existing deadline has expired. | SNR records an assumed reboot through `Timer`. It keeps `Pre-Reboot-Completed` while it waits for confirmed fencing. |

When SNR knows why boot IDs are unavailable, it must record that reason with the report.
A command return, container restart, or shutdown marker does not replace evidence of a changed host boot ID.
An absent API report proves neither reboot completion nor the absence of boot-ID support on the host.
Section 6.4 leaves the classification of missing reports open.

If valid fencing evidence remains unavailable, SNR must keep cleanup blocked in the existing remediation phase.
It must report `WaitingForFencing` through the existing conditions, with a message that identifies the need for operator intervention.
Another expired timeout cannot authorize cleanup, although SNR must continue accepting valid evidence if reporting recovers.
The manual recovery procedure must require an administrator to establish safe fencing before cleanup can proceed.
Section 6.2 leaves the exact recovery procedure open.

SNR uses boot-ID evidence to prevent another reboot for a completed request.
If boot IDs are unavailable, the existing uptime check remains responsible for local duplicate prevention.
An uptime read error prevents a reboot command, and the continued presence of a completed CR does not authorize another reboot.

`TimeAssumedRebooted` and `safeTimeToAssumeNodeRebootedSeconds` remain available for the timer fallback, with the stored deadline preserved across manager restarts.
This deadline belongs to SNR and must not be described as SBR's storage-detection timer.

An observed reboot and successful cleanup are separate results, and neither result establishes storage recovery.
Confirmed evidence for the same episode can permit cleanup after NHC stops SNR, subject to the deferred risk in section 6.1.
An NHC timeout must never enable further SNR reboot commands.

### 5.6 NHC stop handling

`Standard` mode keeps the existing behavior for stopped requests.
In `SBRAssisted` mode, the manager must continue reconciling cleanup and fencing evidence instead of returning unconditionally after an NHC stop.
The assisted branch of `ReconcileAgent()` must also include the missing check that prevents further reboot attempts:

```go
func (r *SelfNodeRemediationReconciler) ReconcileAgent(
	ctx context.Context, req ctrl.Request,
) (ctrl.Result, error) {
	// Existing fetch and target checks omitted.
	if r.executionMode(snr) == v1alpha1.SBRAssistedMode {
		if err := r.syncEvidenceAndRequestStatus(ctx, snr); err != nil {
			return ctrl.Result{}, err
		}
		if r.isStoppedByNHC(snr) || snr.GetDeletionTimestamp() != nil {
			return ctrl.Result{}, nil
		}
	}
	// Existing deletion handling, phase dispatch, and fresh authorization check omitted.
}
```

`syncEvidenceAndRequestStatus` reports evidence and copies the manager's phase and conditions, as described in sections 5.3 and 5.4.
These operations remain active after an NHC stop while the request exists.
The manager must continue processing evidence and eligible cleanup even when the agent cannot update the SNR state file.

Immediately before each software reboot attempt, the agent must read the CR directly from the API server.
The returned CR must have the same UID, generation, target node, and execution mode as the pending attempt.
Its phase must remain `Pre-Reboot-Completed`.
A timeout annotation, deletion timestamp, missing CR, or failed API read prevents the attempt.

Before cleanup, the manager must validate the request identity, node identity, execution mode, and accepted fencing evidence.
An NHC stop prohibits new reboot attempts but still permits eligible cleanup and its completion report.
Evidence reporting and cleanup must not require the phase to remain `Pre-Reboot-Completed`.
SNR keeps the local reboot history for diagnostics and coordination.

SNR must retain its finalizer while cleanup under its ownership remains unfinished.
A deletion request cannot bypass the fencing requirement or cause SNR to discard unfinished cleanup.
SNR releases its taints and finalizers only when the applicable cleanup and recovery criteria are satisfied.
The NHC timeout annotation remains after cleanup, so completion does not restart the expired NHC stage.

These checks cannot recall a reboot command or API change that has already started.
SNR must therefore operate without depending on cancellation of an action already in progress.
Section 6.5 assigns SBR responsibility for its separate authorization and stop behavior.

| Timer | Effect |
| --- | --- |
| SNR assumed-reboot deadline | This deadline permits assumed-reboot reports when boot IDs are unavailable, but it does not establish confirmed fencing. |
| NHC SNR-stage timeout | This timeout stops further SNR reboot attempts and can permit a separate SBR stage. SNR remains responsible for eligible cleanup. |

### 5.7 Cleanup ownership and escalation

SNR remains responsible for workload cleanup before and after NHC escalation, while each operator continues to handle its own remediation CR.
SNR uses its original CR to report cleanup after either provider confirms fencing for the same episode.
The following rules define each provider's responsibility:

| Evidence and cleanup state | SBR behavior | SNR responsibility |
| --- | --- | --- |
| No reboot is confirmed for this episode. | SBR performs authorized fencing, which an unfinished cleanup marker must not block. | SNR waits for confirmed fencing before cleanup. |
| A reboot is confirmed, but cleanup is pending or in progress. | SBR prevents another requested reboot for this episode. | SNR starts or continues eligible cleanup, including after an NHC stop. |
| A reboot is confirmed and cleanup is complete. | SBR prevents another requested reboot for this episode. | SNR reports completion and performs eligible request recovery. |

The separate state files communicate progress and observed evidence without transferring cleanup ownership to SBR.
SBR must not delay necessary fencing because cleanup remains unfinished.
These rules apply to requested remediation for the same episode and do not change SBR's autonomous fencing policy.

The design keeps `getRuntimeStrategy()` and the existing `Automatic`, `ResourceDeletion`, and `OutOfServiceTaint` choices.
Existing request and template validation continues to apply, without a new rejection of the out-of-service strategy in assisted mode.

`handleRebootCompletedPhase()` remains responsible for the resource-removal flow, including `deleteResourcesWrapper()`.
This flow keeps forced Pod deletion, checks specific to each strategy, and existing retries, with the exclusions in section 5.8.
A helper that makes one ordinary deletion pass must not replace these operations.

The selected strategy runs only after SNR accepts fencing evidence under section 5.5.
A timer-based assumption or the end of a best-effort deletion pass cannot, by itself, authorize `Fencing-Completed`.
Existing success and error handling remains in place, except for the changes to assisted completion checks in section 5.8.
Status updates for cleanup must preserve the reported reboot-completion method.

When cleanup completes, the manager sets `Fencing-Completed`, `Succeeded=True`, and `Processing=False`, with the existing `RemediationFinishedSuccessfully` reason.
The old unconditional timeout update must not overwrite this completed result.
These updates finish SNR's remaining work without removing the timeout annotation or changing SBR's CR.

The implementation must support both of these sequences:

1. If SNR's software reboot fails, NHC escalates to SBR, which resets the host.
   SNR then accepts matching fencing evidence and performs cleanup through its original CR.
2. If SNR reboots the host but cleanup remains incomplete, NHC can escalate to SBR.
   SBR prevents another reboot for that episode while SNR continues cleanup and reports completion.

### 5.8 Preserve remediation agents and probe attachments

In `SBRAssisted` mode, workload cleanup must preserve verified SBR and SNR agent Pods on the target node.
The completion check must also ignore verified VolumeAttachments for SBR probes on that node.
These requirements apply to explicitly selected strategies and the strategy that `Automatic` selects, while `Standard` behavior remains unchanged.

These exclusions prevent cleanup from deleting the SBR agent and replacing the generic ephemeral PVC for its probe.
They also allow cleanup to complete while the preserved probe remains attached.
An exclusion establishes neither fencing nor completion of application cleanup.

#### Agent identity and Pod deletion

For each excluded Pod, SNR must verify its identity against the managed SBR or SNR agent DaemonSet in the operator namespace.
The check must include the target node and controller owner reference, including the DaemonSet UID.
SNR must resolve the DaemonSet through the operator's managed configuration and expected agent identity.
Names or labels alone cannot authorize an exclusion, and SNR must not exclude every Pod in an operator namespace or every DaemonSet Pod.

The current [deleteResourcesWrapper()][controller] delegates to [resources.DeletePods()][resource-helper], which deletes all Pods on the node.
Assisted requests require an SNR-specific branch that applies the agent exclusions:

```go
func (r *SelfNodeRemediationReconciler) deleteResourcesWrapper(
	node *corev1.Node, snr *v1alpha1.SelfNodeRemediation,
) (time.Duration, error) {
	if r.executionMode(snr) == v1alpha1.SBRAssistedMode {
		return 0, r.deletePodsExceptRemediationAgents(context.Background(), node)
	}
	return 0, resources.DeletePods(context.Background(), r.Client, node.Name)
}
```

`deletePodsExceptRemediationAgents` is a proposed helper that lists Pods on the target node and verifies agent identities before deletion.
It preserves verified agents and deletes other target-node Pods with the existing zero-grace and background-propagation settings.
Pod UID preconditions must prevent deletion of a replacement Pod through an earlier Pod's name.

API lookup and deletion errors must use the existing cleanup retry mechanism.
This assisted branch must not use unfiltered `DeleteAllOf` or change the common helper used by other modes and operators.

#### Verified probe attachments

[isResourceDeletionCompleted()][controller] must accept the SNR request from `useOutOfServiceTaint()` so that it can restrict exclusions to `SBRAssisted` mode.
Before SNR excludes a VolumeAttachment, it must verify all of these relationships:

1. The attachment targets the remediation node and references the probe PV.
2. The PV's claim reference resolves to a PVC with the matching namespace, name, and UID.
3. The PVC's controller owner reference matches the UID of a verified SBR agent Pod on that node.
4. The PVC matches a configured probe volume in that Pod and its managed DaemonSet template.

A StorageClass, name prefix, or label alone cannot establish probe identity.
Application volumes and unrelated attachments remain subject to cleanup even when they share the probe's StorageClass.
If an object is missing or its relationship is ambiguous, SNR must continue counting the attachment until it disappears or passes verification.

An API lookup error leaves cleanup incomplete and causes a retry.
After a Pod or PVC replacement, SNR must check the identities again because a stale name match cannot preserve an exclusion.
The attachment loop changes as follows:

```go
func (r *SelfNodeRemediationReconciler) isResourceDeletionCompleted(
	node *corev1.Node, snr *v1alpha1.SelfNodeRemediation,
) bool {
	// Existing terminating-Pod check and attachment listing omitted.
	for _, va := range volumeAttachments.Items {
		if va.Spec.NodeName != node.Name {
			continue
		}
		if r.executionMode(snr) == v1alpha1.SBRAssistedMode {
			isProbe, err := r.isVerifiedSBRProbeAttachment(context.Background(), node, &va)
			if err != nil {
				return false
			}
			if isProbe {
				continue
			}
		}
		return false
	}
	return true
}
```

`isVerifiedSBRProbeAttachment` is a proposed helper that performs the identity checks described above.
The existing checks for terminating Pods and all non-excluded attachments remain in place.
In assisted mode, `useOutOfServiceTaint()` must not report completion after its deletion timeout if application or unverified attachments remain.

Manager RBAC must permit the DaemonSet, Pod, PVC, PV, and VolumeAttachment reads required for these checks.
Neither state file alone can authorize an exclusion because the decision depends on verified API relationships.

#### Preserve agents during out-of-service cleanup

Exclusions from direct deletion do not prevent eviction caused by the out-of-service taint.
The current [SBR DaemonSet][sbr-agent-config] tolerates all `NoExecute` taints, but the default [SNR agent template][daemonset] lacks the out-of-service toleration.
In `SBRAssisted` mode only, [syncConfigDaemonSet()][config-controller] must render this additional entry:

```yaml
spec:
  template:
    spec:
      tolerations:
        # Existing tolerations omitted.
        - key: node.kubernetes.io/out-of-service
          operator: Exists
          effect: NoExecute
```

The entry must omit `tolerationSeconds` so that the cleanup duration does not cause agent eviction.
SBR keeps its existing toleration, and integration tests must check both agents.
Together, these changes preserve the agent Pods and probe PVC during cleanup, subject to independent failures or rollouts.

## 6. Open Questions

### 6.1 Workload restart before forced cleanup

The design acknowledges the risk of workloads restarting before forced cleanup, but protection against this risk remains outside the current scope.
Resolving this risk is not a prerequisite for implementing the integration.

A changed host boot ID establishes that the previous host execution ended, but it does not show that workloads remain stopped afterward.
The existing [ResourceDeletion helper][resource-helper] requests zero-grace Pod deletion without waiting for confirmation that the containers stopped.
As explained in the [forced Pod termination guidance][forced-deletion], removal of a Pod object from the API does not prove that its containers stopped.

The following sequence illustrates the risk:

1. The node reboots and starts an original workload again.
2. The SNR agent reports the changed boot ID.
3. The kubelet stalls or loses API access while the workload continues to run.
4. SNR force-deletes the Pod object, after which Kubernetes starts a replacement on another node.
5. The original workload remains active because the kubelet has not stopped it.

The original and replacement workloads can then run at the same time.
Without separate storage or application protection, their writes can conflict and cause data corruption.
This sequence describes a possible failure that has not been reproduced.

The out-of-service strategy also depends on shutdown verification, as described in the [Kubernetes guidance][shutdown].
A changed boot ID alone does not show that the node remains off when cleanup starts.
This design keeps the existing cleanup strategies without adding protection against this risk or claiming that reboot evidence removes it.

The evidence requirement in section 5.5 remains in scope, so timer expiry alone cannot authorize cleanup.
If fencing evidence is missing, cleanup must wait and requires operator intervention.

### 6.2 Evidence and recovery after escalation

SNR remains responsible for cleanup after escalation under section 5.7.
The implementation must resolve the following questions before it can associate evidence and support manual recovery:

- How does evidence associate a separate SBR CR with the original SNR request and the same remediation episode?
- Which checks and actions must an administrator complete when fencing evidence remains unavailable?

An NHC stop neither transfers cleanup ownership nor authorizes cleanup without confirmed fencing.
If evidence remains missing, cleanup stays blocked and requires operator intervention.
This design does not require an additional automatic source of evidence.
The manual recovery procedure must establish safe fencing before it permits forced cleanup or release of retained resources.

### 6.3 Coordination with SBR reboot suppression

SBR must prevent another requested reboot when SNR already rebooted the node during the same NHC remediation.
The SBR design owns this implementation, including the case where SNR cleanup remains pending.
Without confirmed reboot evidence, a marker of unfinished cleanup must not block authorized SBR fencing.

The integration must define how SBR associates a new SBR CR with SNR's recorded reboot.
SNR supplies request identity, attempt details, observed evidence, and copied cleanup progress through its own state file.
This document does not define SBR's suppression algorithm or completion status.

### 6.4 Evidence availability and state-file compatibility

The implementation must define when boot IDs are unavailable, with separate treatment for an unavailable source and an absent or delayed API report.
An unchanged, available boot ID must not become unavailable merely because the timer expired.
The following details also remain open:

- How does the manager classify missing reports when the target never returns or cannot reach the API?
- How does each agent check the schema version when it reads the other agent's file?
- How does SNR report each node's directory failures on its config without hiding failures on other nodes?
- How do schema upgrades preserve pending attempts and records of completed requests?
- Which retained records does SBR need to associate consecutive NHC stages?

The API examples in section 5 illustrate the proposed SNR changes.
Before code generation, the implementation must define validation and status ownership for those fields.

### 6.5 SBR request authorization and fencing commitment

**SBR owns this question** and must define and implement the rules for its own remediation CR and local watchdog operations.
An existing CR does not necessarily authorize a new reboot because it can remain after an NHC stop, during deletion, or after completion.

The current [SBR local lookup](https://github.com/medik8s/storage-based-remediation/cmd/sbr-agent/main.go:1136) checks only whether the CR exists.
It does not check the timeout annotation, deletion timestamp, or completion conditions.
The separate [SBR controller stop check](https://github.com/medik8s/storage-based-remediation/internal/controller/storagebasedremediation_controller.go:360) does not protect the local watchdog operations.

SBR must resolve these questions:

1. Which request identity, target node identity, preparation steps, and conditions permit a new fencing action?
2. How does the local agent reject stopped, deleting, completed, failed, or replaced requests before it commits to fencing?
3. Which fresh API check is necessary immediately before commitment, and how does the agent handle a failed read?
4. Which operation establishes the commitment after which watchdog feeding must remain stopped until reset?
5. What happens if NHC stops or deletes the request between the final API check and commitment, or after commitment?

The SBR design must distinguish observation of a request from commitment to its reset.
These questions concern requested fencing, while SBR must define its independent autonomous fencing rules separately.
SNR keeps its own reboot stop rules and cleanup responsibility under sections 5.6 and 5.7.

## 7. Upgrade / Downgrade Concerns

Requests that have already started without `executionMode` keep `Standard` behavior.
For new requests, the manager must store the mode before preparation so that each request keeps its intended behavior.
If the agent's local mode differs from the request mode, the agent must not act on that request.

Use this sequence to prepare the transition:

1. Upgrade the CRDs, manager, and selected agents with compatible status fields and support for separate state files.
2. Pause new NHC requests.
3. Complete outstanding SNR requests, including cleanup retained after escalation.
4. Check the availability of the shared directory and schema compatibility.
5. Stop SNR watchdog ownership before enabling the SBR-owned configuration.
6. Check SNR startup, evidence reports, agent tolerations, and the NHC storage-health policy before resuming remediation.

The integration must define the full SBR transition procedure, while this document specifies only the SNR requirements.
During staggered upgrades, each agent must preserve unknown fields in its own file.
Pod replacement must also preserve both files and pending reboot evidence.

A downgrade must not proceed while requests require the new execution mode, evidence fields, or cleanup after escalation.
Before an older agent writes its file or reads the other file, the implementation must define a compatible schema transition.
An older SNR agent must not silently resume watchdog ownership under an assisted configuration.

Implementation requires regeneration of the CRDs, deepcopy code, affected deployment and OLM manifests, and configuration samples.
The proposed API mechanism for evidence reports does not require protobuf changes.

## 8. Test Plan

This section defines the tests required for implementation of the proposed design.
It does not claim that the proposed code exists or that these tests have run.

### 8.1 Unit and controller tests

The following cases cover the API, agent behavior, state files, and cleanup rules:

| Area | Required cases |
| --- | --- |
| Config API and admission | An omitted mode uses `Standard`. Admission rejects unknown modes and explicit software-reboot disablement in assisted mode. |
| Mode transitions | Requests with finalizers prevent mode changes, while requests that already started in `Standard` keep that behavior. |
| Agent startup | Assisted agents must not initialize the SNR watchdog or enable autonomous fencing through API or control-plane checks. Authorized software reboot remains available. |
| DaemonSet generation | The DaemonSet includes `SNR_MODE` and the permissions required for the shared directory. Only assisted mode adds the out-of-service toleration. |
| State-file integration | Tests cover fixed paths, separate write ownership, directory creation, and serialized updates within each agent. They also cover atomic replacement, unknown fields, record retention, and concurrent reads and writes without a shared lock. Unfinished cleanup records remain available. |
| Attempt recording | SNR stores request identity and the initial boot ID before it issues a reboot command. A failed state write prevents the command. |
| Startup recovery | SNR distinguishes an agent restart from a host reboot. It associates a changed boot ID before it replaces the previous state. |
| State failures | Tests cover missing files, corruption, incompatible schemas, exhausted retries, and degraded reports. Failure of SNR's own file causes exit without watchdog operations. A failed read must not cause SNR to modify or quarantine SBR's file. |
| Evidence reporting | SNR rejects stale generations, incorrect node identities, and unrelated episodes. Reports continue after a stop without replacing status fields owned by the other component. |
| Cleanup reporting | The agent copies the manager's phase and conditions, with source identity and observation time, without another cleanup field. Missing or stale reports do not prove completion. |
| Reboot detection | Different available IDs confirm reboot, while equal available IDs keep the request pending after timer expiry. |
| Timer fallback | If boot IDs are unavailable, SNR uses the stored deadline for assumed-reboot reports and keeps `Pre-Reboot-Completed`. Timer expiry alone permits neither cleanup nor suppression of required SBR fencing. |
| Missing fencing evidence | SNR keeps the existing phase and finalizers while it reports `WaitingForFencing` and required intervention. Reasons and messages update even when condition values remain unchanged. SNR accepts valid evidence if reporting recovers. |
| Local duplicate prevention | Completed requests do not reboot again. If boot IDs are unavailable, uptime checks remain active, and uptime errors prevent reboot commands. |
| NHC stop handling | Fresh reads prevent reboot commands for stopped, deleted, replaced, or mismatched requests. Confirmed fencing permits cleanup and completion after a stop. |
| Escalation | SNR performs cleanup after an SBR-confirmed reset and continues unfinished cleanup after its own reset. SBR prevents a duplicate reboot for the same episode. |
| Existing cleanup | Every strategy keeps its forced removal operations and retries, with the assisted exclusions and completion requirements in section 5.8. |
| Agent deletion exclusions | Verified SBR and SNR agent Pods remain, while application Pods and unrelated DaemonSet Pods undergo deletion. Tests cover label-only matches, mismatched owner UIDs, lookup retries, and Pod UID preconditions. |
| Probe completion exclusions | Cleanup can complete with only verified probe attachments remaining. Application or unverified attachments block completion, including after deletion and timeout. Tests cover wrong nodes, missing owners, mismatched UIDs, shared StorageClasses, and replacements, while Standard checks remain unchanged. |
| Restart and compatibility | SNR resumes through existing phases and accepted evidence, with conditions and deadlines preserved. Finalizers remain for unfinished cleanup, and SNR releases owned resources after recovery. Standard behavior remains unchanged. |

Implementation tests must extend the existing envtest coverage under `internal/controller/tests` and webhook coverage under `internal/webhook/v1alpha1`.
Focused integration tests in the common library must cover the separate state files and their read and write rules.
After implementation, run these commands:

1. Run `make manifests generate`.
2. Run `make test`.

### 8.2 Cluster integration tests

Each integration test must capture host boot IDs, uptime, CR identities, status changes, Pod UIDs, and VolumeAttachments.
The tests must check watchdog ownership independently and must not count a container restart as a host reboot.
The following scenarios define the expected behavior:

| Scenario | Expected SNR behavior |
| --- | --- |
| Storage failure with API access | SNR handles the NHC-created CR and records the software attempt. It reports reboot evidence and uses the selected cleanup strategy. |
| Software reboot fails with available, unchanged IDs | SNR keeps the reboot pending after timer expiry and honors an NHC stop before escalation. |
| Boot IDs are unavailable | SNR reports an assumed reboot through the timer. Cleanup waits, with required operator intervention reported while valid fencing evidence remains unavailable. |
| The agent restarts without a host reboot | SNR keeps the pending attempt without reporting a changed boot ID. |
| The host reboots while its SNR CR remains | SNR recovers matching evidence and reports it on the same CR, including after an NHC stop. It prevents another SNR reboot. |
| API or peer access fails while storage works | SNR must not trigger autonomous fencing on workers or control-plane nodes. |
| The target loses API access during a request | SNR keeps local evidence for later reports but must not issue another command without fresh authorization. |
| NHC stops SNR before reboot succeeds | SNR stops new software attempts. After SBR resets the host, SNR accepts matching evidence and starts eligible cleanup through the original CR. |
| NHC stops SNR during cleanup | SNR continues eligible cleanup while SBR uses confirmed evidence to prevent another reboot. SNR reports completion without removing the timeout annotation. |
| The copied cleanup report is missing or stale | Missing progress proves neither completion nor the need to repeat a confirmed reboot. Without confirmed reboot evidence, the missing report must not block necessary SBR fencing. |
| Both agents update their files concurrently | Each agent writes only its own file, and readers observe complete versions. Readers validate episode identity before combining records from different versions. |
| SBR's state file cannot be read | SNR reports the failure and retries without modifying SBR's file or inferring fencing. SNR file updates and eligible manager cleanup can continue using accepted evidence. |
| The shared directory becomes unusable | The agent retries and reports degraded operation before it exits without an unrecorded reboot. Eligible manager cleanup continues using accepted evidence. |
| A strategy reports an error or incomplete removal | SNR keeps the strategy's existing retry and error behavior. One best-effort deletion pass must not establish success. |
| ResourceDeletion runs with remediation agents present | SNR preserves the agent Pod UIDs and probe PVC UID while it deletes application Pods. Cleanup must not require a replacement probe. |
| OutOfServiceTaint runs with an attached probe | Both agents remain through their tolerations while SNR waits for application attachments to disappear. Cleanup can complete with the verified probe attachment present. |
| Probe ownership cannot be verified | SNR continues counting the attachment and must not infer completion from labels or stale names. |
| The node never returns or cannot report evidence | SNR keeps the phase and finalizers while it reports `WaitingForFencing` and required intervention. Elapsed time cannot authorize cleanup or establish an unobserved reboot. |
| Deletion is requested during retained cleanup | SNR keeps its finalizer until the cleanup and recovery criteria permit removal. The deletion request cannot serve as fencing evidence. |
| An upgrade or supported downgrade occurs | Tests check execution modes, evidence preservation, schema compatibility, and the absence of SNR watchdog ownership in assisted mode. |

Tests for cleanup after escalation must include both operators and the configured NHC sequence.
The section 5.8 cases must cover explicitly selected strategies and `Automatic`, with an attachable CSI driver.
These tests must use the evidence and episode association defined under sections 6.2–6.4 and 5.5.

SBR owns reboot suppression, but integration tests must check its behavior against section 5.7.
Fencing tests must require accepted evidence because timer expiry or command success alone cannot authorize cleanup.
Tests for new protection against workload restart before cleanup remain outside scope under section 6.1.

[config-api]: https://github.com/medik8s/self-node-remediation/tree/medik8s-5.8/api/v1alpha1/selfnoderemediationconfig_types.go
[request-api]: https://github.com/medik8s/self-node-remediation/tree/medik8s-5.8/api/v1alpha1/selfnoderemediation_types.go
[config-webhook]: https://github.com/medik8s/self-node-remediation/tree/medik8s-5.8/internal/webhook/v1alpha1/selfnoderemediationconfig_webhook.go
[config-controller]: https://github.com/medik8s/self-node-remediation/tree/medik8s-5.8/internal/controller/selfnoderemediationconfig_controller.go
[controller]: https://github.com/medik8s/self-node-remediation/internal/tree/medik8s-5.8/controller/selfnoderemediation_controller.go
[daemonset]: https://github.com/medik8s/self-node-remediation/tree/medik8s-5.8/install/self-node-remediation-deamonset.yaml
[main]: https://github.com/medik8s/self-node-remediation/tree/medik8s-5.8/cmd/main.go
[annotations]: https://github.com/medik8s/self-node-remediation/tree/medik8s-5.8/internal/utils/annotations.go
[rebooter]: https://github.com/medik8s/self-node-remediation/tree/medik8s-5.8/internal/reboot/rebooter.go
[resource-helper]: https://github.com/medik8s/self-node-remediation/tree/medik8s-5.8/vendor/github.com/medik8s/common/pkg/resources/resources.go
[sbr-agent-config]: https://github.com/medik8s/storage-based-remediation/tree/medik8s-5.8/internal/controller/storagebasedremediationconfig_controller.go
[forced-deletion]: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced
[shutdown]: https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/#non-graceful-node-shutdown-handling
