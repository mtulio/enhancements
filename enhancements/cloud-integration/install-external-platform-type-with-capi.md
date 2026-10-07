---
title: install-external-platform-type-with-capi
authors:
  - "@mtulio"
  - "@rvanderp3"
  - "@elmiko"
reviewers: # Include a comment about what domain expertise a reviewer is expected to bring and what area of the enhancement you expect them to focus on.
  - TBD, "installer, for the Cluster API bootstrap control plane and the asset graph"
  - TBD, "cloud-team, for the platform External contract and the partner support boundary"
  - TBD, "cluster-capi-operator, for the day-0/day-2 boundary and provider packaging"
approvers: # This should be a single approver.
  - TBD
api-approvers: # No cluster API change: install-config only, no CRD, aggregated apiserver, webhook or finalizer. See "Operational Aspects of API Extensions".
  - None
creation-date: 2026-10-06
last-updated: 2026-10-06
status: provisional
tracking-link: # link to the tracking ticket that corresponds to this enhancement
  - https://issues.redhat.com/browse/OCPSTRAT-3724
see-also:
  - ["/enhancements/cloud-integration/infrastructure-external-platform-type.md"](https://github.com/openshift/enhancements/blob/master/enhancements/cloud-integration/infrastructure-external-platform-type.md)
  - ["/enhancements/installer/bootstrapping-clusters-with-capi-providers.md"](https://github.com/openshift/enhancements/blob/master/enhancements/installer/bootstrapping-clusters-with-capi-providers.md)
  - ["/enhancements/cloud-integration/out-of-tree-provider-support.md"](https://github.com/openshift/enhancements/blob/master/enhancements/cloud-integration/out-of-tree-provider-support.md)
  - ["/enhancements/cluster-api/installing-cluster-api-components-in-ocp.md"](https://github.com/openshift/enhancements/blob/master/enhancements/cluster-api/installing-cluster-api-components-in-ocp.md)
replaces: []
superseded-by: []
---

# Install a platform External cluster with a partner-supplied Cluster API provider

## Summary

`platform: external` today has no installer-native provisioning path. A partner who
already maintains a Cluster API infrastructure provider cannot use it at install time,
so clusters on their platform are stood up by Assisted Installer or UPI and the
infrastructure is created outside the installer's control.

This enhancement lets `openshift-install create cluster` provision infrastructure for a
`platform: external` cluster by running a Cluster API provider **the installer was never
compiled against**. The provider's controller binary and component manifests are supplied
by the user as paths; the installer's existing local, ephemeral Cluster API control plane
runs them exactly as it runs the providers it embeds. Three optional user-supplied hooks
cover the cloud-specific work the installer cannot do, because on `platform: external` it
holds no cloud credentials by design.

The design is deliberately provider-agnostic: there is no OCI-specific, AWS-specific or
any other platform-specific code path. It has been exercised end to end on two providers,
one embedded in the installer (CAPA) and one that cannot be (CAPOCI).

## Motivation

Partners and customers deploying OpenShift on an external platform get a materially
different install experience from integrated platforms. Oracle Cloud Infrastructure is the
concrete driver, but the gap is general: any partner with a working Cluster API provider
carries the whole provisioning burden themselves.

`OCPSTRAT-1322` covers the day-2 half of this problem — enabling a partner CAPI provider
in-cluster for autoscaling and machine health checks. It explicitly does not cover
install-time provisioning. This enhancement is the day-0 half.

### User Stories

* As a cloud provider partner with an existing Cluster API infrastructure provider, I want
  OpenShift's installer to run my provider during install, so that my customers get an
  IPI-like experience without Red Hat writing and maintaining in-tree support for my
  platform.
* As a cloud provider partner, I want to plug my provider in without forking the installer
  or getting code merged into the OpenShift release payload, so that my release cadence is
  mine.
* As a cluster administrator installing on a partner platform, I want
  `openshift-install create cluster` and `openshift-install destroy cluster` to work the
  way they do everywhere else, so that my automation and my mental model carry over.
* As an OpenShift installer maintainer, I want external-platform support to add no risk to
  integrated platforms, so that I can review these changes quickly and trust the blast
  radius.
* As a Red Hat support engineer, I want to be able to tell immediately whether a failed
  install failed in Red Hat code or in a partner-supplied component, so that I route the
  case correctly.

### Goals

* `openshift-install create cluster` provisions infrastructure for a `platform: external`
  cluster using a Cluster API provider supplied by the user at install time.
* `openshift-install destroy cluster` removes that infrastructure through the same
  provider that created it.
* The contract a partner must satisfy is documented, stable and provider-agnostic.
* The partner supplies credentials to their own provider without the installer handling
  them and without forking the installer.
* No behavioural change whatsoever for any integrated platform, or for `platform: none`.
* CI covers at least one external-platform CAPI install path continuously.

### Non-Goals

* **The day-2 machine-management surface.** Autoscaling, machine health checks, MachineSet
  adoption and pivoting Cluster API controllers into the installed cluster are
  `OCPSTRAT-1322`. The Cluster API control plane this enhancement uses is local to the
  installer process and ephemeral; nothing from it reaches the installed cluster.
* **Changing how integrated platforms provision.** No existing platform's behaviour,
  validation or artifact resolution changes.
* **Shipping partner providers in the OpenShift release payload.** Everything the partner
  supplies is out-of-payload, by definition of `platform: external`.
* **Red Hat supporting partner provider code.** The support boundary is a deliverable of
  this work, not an obligation it creates.
* **Publishing boot images for partner platforms.** Tracked separately; see
  [Open Questions](#open-questions-optional).
* **Automatic CSR approval for externally-provisioned nodes.** A pre-existing gap on both
  `platform: external` and `platform: none`; see
  [Open Questions](#open-questions-optional).

## Proposal

The installer already stands up a local Cluster API control plane during bootstrap and
runs infrastructure providers against it; this is the mechanism described in
*bootstrapping-clusters-with-capi-providers*. Today that control plane may only run
providers whose binaries are embedded in the installer image.

This enhancement adds, for `platform: external` only:

1. An install-config stanza, `platform.external.clusterAPI`, naming the provider and
   giving filesystem paths to its controller binary and component manifests.
2. Resolution of those paths in place of the embedded artifact archive, with validation
   before any side effect.
3. Acceptance of Cluster API manifests whose `kind` has no compiled-in Go type, scoped to
   the external provisioning path.
4. An optional `hooks` stanza with three lifecycle points — `infraReady`,
   `postProvision`, `preDestroy` — for cloud-specific work the installer cannot perform.
5. Recording of the provider in cluster metadata, so `destroy cluster` can tear down
   through the same provider.
6. A hidden `openshift-install extract cluster-api <provider>` command that writes an
   embedded provider's artifacts to a directory, so a partner can see the exact shape the
   contract expects.

### Workflow Description

**Partner provider developer** is a human who maintains a Cluster API infrastructure
provider for a platform that OpenShift does not integrate in-tree.

**Cluster creator** is a human or automation installing OpenShift on that platform.

Provider preparation, done once by the partner provider developer:

1. Build the provider's controller binary for the installer's host architecture.
2. Collect the provider's component manifests — CRDs, RBAC, webhook configuration — into
   a directory.
3. Optionally run `openshift-install extract cluster-api aws` to see the layout an
   embedded provider uses as a worked reference.
4. Write the Cluster API manifests for a cluster on that platform, and the optional hook
   programs.

Install, done by the cluster creator:

1. Write `install-config.yaml` with `platform.external.clusterAPI` pointing at the binary
   and components directory.
2. Place the provider's Cluster API manifests in `<install-dir>/cluster-api/` and any
   day-0 cluster manifests in `<install-dir>/openshift/`.
3. Run `openshift-install create cluster`.
4. The installer resolves and validates the provider artifacts, starts the local Cluster
   API control plane, installs the provider's components, runs the provider's controller,
   and applies the user's Cluster API manifests.
5. On `Cluster.status.infrastructureReady`, the `infraReady` hook runs — typically to
   create DNS records or publish the bootstrap ignition somewhere the platform can reach.
6. Machines are provisioned; bootstrap completes; the Cluster API control plane is torn
   down.
7. The `postProvision` hook runs against the new cluster's kubeconfig, applying the
   partner's cloud controller manager, CSI driver and related configuration.
8. Install completes.

Teardown:

1. `openshift-install destroy cluster` reads the provider from cluster metadata.
2. The `preDestroy` hook runs.
3. The provider is restarted locally, its credential Secret re-applied, and the
   infrastructure custom resources are deleted so the provider's finalizers can run.

#### Variation: platforms that cannot carry the bootstrap ignition in instance metadata

`bootstrap.ign` is typically hundreds of kilobytes and many clouds cap instance user-data
well below that. Such platforms run the install in three phases rather than two:
`create manifests`, `create ignition-configs`, then — in partner tooling, before
`create cluster` — upload `bootstrap.ign` to object storage, mint a pre-signed URL, and
replace the on-disk file with a small pointer-ignition document. The installer reloads the
file during asset resolution, so this requires no installer change.

This works, but it means **every partner reimplements the same credential hazard**: the
pre-signed URL grants unauthenticated read over every secret in the bootstrap ignition.
Whether the installer should offer a first-class mechanism is an open question below.

### API Extensions

One new optional field on an existing platform stanza, `platform.external.clusterAPI`, of
type `*ClusterAPIProvider`
(`pkg/types/external/platform.go:39`, pilot branch `pilot-platform-external-capi` at
commit `c103ca8c90`). `platform: external` already exists and is unchanged otherwise.

```go
// pkg/types/external/platform.go:62
type ClusterAPIProvider struct {
    Name           string   `json:"name"`
    BinaryPath     string   `json:"binaryPath"`
    ComponentsPath string   `json:"componentsPath"`
    Args           []string `json:"args,omitempty"`
    Hooks          *Hooks   `json:"hooks,omitempty"`
}

// pkg/types/external/platform.go:115
type Hooks struct {
    InfraReady    *Hook `json:"infraReady,omitempty"`
    PostProvision *Hook `json:"postProvision,omitempty"`
    PreDestroy    *Hook `json:"preDestroy,omitempty"`
}

// pkg/types/external/platform.go:168
type Hook struct {
    Program string   `json:"program"`
    Args    []string `json:"args,omitempty"`
}
```

This is install-config only. It is not a cluster API: it is not served by the API server,
it is not persisted into the cluster, and it has no CRD, webhook, aggregated apiserver or
finalizer. Validation is ordinary install-config validation returning a
`field.ErrorList` (`pkg/types/external/validation/platform.go`).

The agent-based installer shares the install-config format but routes data through
assisted-service's cluster manifests, and this field is not passed through. Per the
install-config conventions, agent-based validation **warns** when the field is set to a
non-default value rather than ignoring it silently.

`ClusterPlatformMetadata` gains an `External` member
(`pkg/types/clustermetadata.go:46`) so `destroy cluster` can find the provider. It is
written conditionally (`pkg/types/clustermetadata.go:89`); `platform: none` continues to
write nothing.

Whether this is the right long-term shape — paths on a local filesystem versus an image
reference, a CRD bundle, or a user-supplied manifest in the style of a cloud controller
manager override — is an open question below. The pilot used paths because they are the
smallest thing that proves the mechanism.

### Topology Considerations

#### Hypershift / Hosted Control Planes

Not applicable in this enhancement. HyperShift does not use `openshift-install` to
provision a hosted cluster's infrastructure, so the install-time path described here has
no HyperShift analogue. `platform: external` support on hosted control planes is being
investigated separately and is explicitly out of scope here; if it proceeds it will need
its own enhancement, because the mechanism — not just the configuration — differs.

#### Standalone Clusters

This enhancement targets standalone clusters exclusively. A `platform: external` cluster
installed this way is an ordinary standalone cluster whose infrastructure happened to be
created by a partner-supplied Cluster API provider.

#### Single-node Deployments or MicroShift

Not applicable. MicroShift does not use `openshift-install`. Single-node OpenShift on
`platform: external` is not prevented by anything here — the provider would simply
provision one machine — but it is untested and not a goal of this work; no SNO-specific
resource or footprint consideration arises, because the installer-side change adds no
runtime component to the cluster.

#### OpenShift Kubernetes Engine

Not applicable. This changes the installer and adds no cluster-side component, so there is
nothing whose presence or absence differs between OKE and OpenShift Container Platform. A
cluster installed this way is subject to whatever OKE restrictions already apply to
`platform: external`.

### Implementation Details/Notes/Constraints

File locations on the pilot branch `pilot-platform-external-capi` at commit `c103ca8c90`:

| Area | Location |
| --- | --- |
| Install-config API and validation | `pkg/types/external/platform.go`, `pkg/types/external/validation/platform.go` |
| Artifact override and validation | `pkg/clusterapi/artifacts.go` |
| Running a user-supplied controller | `pkg/clusterapi/external.go` |
| External provisioning path | `pkg/infrastructure/external/clusterapi/` |
| Lifecycle hooks | `pkg/infrastructure/external/hooks/` |
| Teardown | `pkg/destroy/external/` |
| Platform routing | `pkg/infrastructure/platform/platform.go:59` |

Constraints discovered by running it, not by design:

* **The installer has no cloud credentials on `platform: external`, and must not acquire
  any.** This is why hooks exist. Every piece of cloud-specific work — DNS records,
  ignition staging, load balancer attachment — is the partner's, executed through a hook,
  using the partner's own credentials.
* **Hook environment carries paths, not documents.** Passing a kubeconfig or an ignition
  body through an environment variable puts secrets into process listings and traces.
* **`infraReady` cannot write into `<install-dir>/openshift/`.** By the time infrastructure
  is ready the installer has already consumed and removed that tree, so partner
  configuration Secrets must be applied from `postProvision` against the hook's kubeconfig.
  This is a measured finding, not a preference.
* **A wrapper script is a legal `binaryPath`.** Cluster API providers do not agree on flag
  names — the installer passes `--health-addr` (`pkg/clusterapi/external.go:155`) and at
  least one real provider declares `--health-probe-bind-address` instead, with pflag
  exiting non-zero on an unknown flag. A shim that translates the flag is the supported
  answer today. Note that a shim must **translate**, not drop: the installer polls
  `/healthz` at exactly the address it passed.
* **`destroy cluster` deliberately excludes Secrets from the saved Cluster API output**, so
  teardown must re-apply the provider's credential Secret before the infrastructure
  resource's finalizer can clear.
* **The architecture of the supplied binary is checked before use.** A cross-built
  controller fails at validation rather than at exec time.
* **Artifacts are logged by path, source and SHA-256 — never by content.**

### Risks and Mitigations

**Accepting unknown manifest kinds weakens an existing client-side check.** A
user-supplied `AWSCluster` or `OCICluster` has no compiled-in Go type, so the scheme
rejects it. The pilot falls back to unstructured handling. Done carelessly this would turn
a misspelled `kind` on an integrated platform from a loud failure into a silent one.
*Mitigation:* the fallback is scoped to the external provisioning path, not the shared
asset load path, and it warns rather than being silent. This is the single highest-risk
change in the set and is called out again under Open Questions, because the pilot's
scoping is provisional rather than chosen.

**A partner-supplied binary runs on the installer host.** It is executed with the
installer's privileges against the local Cluster API control plane.
*Mitigation:* path, executable-bit and host-architecture validation happen before any
cloud call; the artifact's SHA-256 is recorded. This does not make the binary trusted —
and that is the honest position. Running an untrusted provider is the user's decision, in
the same category as running an untrusted Terraform provider.

**A pre-signed bootstrap-ignition URL is credential-equivalent.** On platforms needing the
three-phase install, that URL grants unauthenticated read over every bootstrap secret.
*Mitigation:* partner tooling must mint it with the shortest workable lifetime and delete
it during teardown; the reference example verifies the object and the pre-signed request
are gone after destroy rather than assuming it. A first-class installer mechanism would
remove the need for each partner to get this right independently — see Open Questions.

**Support attribution.** An install can now fail inside code Red Hat did not write.
*Mitigation:* the provider name and artifact hashes are recorded in metadata and logs; see
Support Procedures. Whether something stronger than logs is needed is an open question.

**Partner provider quality is unverified.** Nothing checks that a supplied provider behaves
correctly. *Mitigation:* none today. A conformance suite is an open question, and the
partner validation model is a deliverable of OCPSTRAT-3724 in its own right.

### Drawbacks

* It adds a supported configuration in which a significant part of the install is executed
  by code outside the OpenShift release payload, which complicates support triage even
  with good attribution.
* It touches shared installer code — manifest handling, artifact resolution, the platform
  switch — to serve a platform most users will never use. The risk to integrated platforms
  is small but not zero, which is why it is isolated into the smallest possible changes.
* Supplying providers as filesystem paths is awkward for automation and for disconnected
  environments, and may not be the shape we want long term.
* It creates an expectation that Red Hat will help partners debug their providers.

## Alternatives (Not Implemented)

**Integrate each partner platform in-tree.** This is the status quo for supported
platforms and it does not scale: every platform costs Red Hat ongoing maintenance, and it
is exactly what `platform: external` was created to avoid.

**Embed partner providers in the installer image.** Rejected: it puts partner code in the
release payload, couples the partner's release cadence to OpenShift's, and contradicts the
`platform: external` contract.

**Fetch the provider from a remote reference — an image or a git URL — rather than a
path.** Not implemented in the pilot, deliberately, because it adds digest pinning, pull
credentials and a mirroring story before the core mechanism is proven. It remains the most
likely future direction; see Open Questions.

**Let partners fork the installer.** Already possible, already happening, and the thing
OCPSTRAT-3724 explicitly wants to stop.

**Keep using Assisted Installer or UPI for external platforms.** Works today, and remains
available. It does not give an IPI-like experience and leaves the cluster with no
machine-management layer wired in from day 0.

**Add per-platform exceptions for the known partner.** Rejected on principle: platform
External exists to fit any provider, and an exception for one invites an exception for
every other.

## Open Questions [optional]

1. **What is the long-term shape of the provider reference?** Filesystem paths proved the
   mechanism. An image reference, a CRD bundle, or a user-supplied manifest in the style of
   existing cloud controller manager overrides would all be more operable. This should be
   settled before Tech Preview.
2. **How is the scheme fallback for unknown kinds finally scoped?** Three candidates:
   restrict to `platform: external`, validate against the CRDs actually present in
   `componentsPath`, or register partner CRDs into the scheme at runtime. The pilot does
   the first; the choice is not made.
3. **Should the installer provide a first-class "bootstrap ignition is staged elsewhere,
   here is the pointer" mechanism?** Leaving it to each partner means each partner
   reimplements the same credential hazard.
4. **Should `clusterAPI.args` be able to override an installer-supplied flag by name,
   rather than only append?** Today's answer is a per-partner wrapper script.
5. **Do we validate user-supplied providers, and how?** A conformance suite is the obvious
   answer and nobody has scoped one.
6. **How are failure modes surfaced?** Installer logs only, or do we need a path that
   explicitly signals the failing component is not Red Hat owned?
7. **Should partner cloud controller manager and CSI components be required to tag what
   they create?** Teardown is not supportable without it.
8. **Should node-to-node 80/443 be a stated `platform: external` networking requirement?**
   Currently discovered by failure.
9. **Is the ingress `Service` something `platform: external` should generate, or must every
   provider hand-author it?** Nothing currently asks the platform for an external address.
10. **Should an extra manifest be allowed to contain multiple documents?** The pilot
    refuses them, so that a multi-document file fails loudly rather than silently applying
    only the first. This is an API contract and should be decided before this lands.
11. **Boot images.** `platform: external` installs need a boot image for the target
    platform, and for at least one partner platform no generic, multi-tenant image is
    published. The capability exists in RHCOS; the artifact does not. A generic
    artifact-override mechanism in the shape of an
    `OPENSHIFT_INSTALL_RHCOS_ARTIFACTS_JSON` input is the preferred direction, explicitly
    **not** as a per-platform special case. Tracked separately; blocks continuous CI for
    the second provider only.
12. **Automatic CSR approval.** There is no cloud-specific machine approver on
    `platform: external`, so worker CSRs are never approved and the measured cost is manual
    approval on every install. This pre-dates this enhancement and applies equally to
    `platform: none`. A webhook comparing a CSR against the instance it claims to come from
    is the agreed direction; it has no owner, no repository and no schedule, and the only
    thing settled is that it is not the installer.

## Test Plan

Detailed test plans are maintained per delivery phase in IEEE 829 form and are not
reproduced here. This section states the strategy and the evidence requirements.

**Unit.** Install-config validation, artifact resolution and validation, hook argument and
environment construction, metadata round-tripping, and teardown adoption logic.

**No-regression, and this is the part that matters for review.** Three changes touch code
shared with integrated platforms: the unknown-kind fallback, extra-manifest handling, and
the robustness fixes in the Cluster API bootstrap code. Each must demonstrate — not assert
— that integrated platforms are unaffected: with the new install-config field unset,
behaviour is identical to `main`, and a malformed manifest on an integrated platform still
fails loudly.

**End-to-end.** Two Prow arms:

* an AWS/CAPA arm, as the reference example and the fast signal;
* a second arm on a provider that is **not** embedded in the installer.

**The second arm is the only one that constitutes evidence for this enhancement's central
claim**, and reviewers should hold it to that standard. The installer embeds CAPA, so an
AWS run cannot distinguish "ran the binary it was handed" from "ran the binary it already
had". A provider that cannot be embedded can. The AWS arm tests the ergonomics of the
mechanism; it does not test the mechanism.

**Teardown is verified on a second cluster.** Three teardown defects in the pilot were
found only by creating a cluster after a previous one had been destroyed. A single
create/destroy cycle is not sufficient evidence.

**Conformance** runs on the resulting cluster, with the known CSR-approval gap handled
explicitly by the job rather than silently.

## Graduation Criteria

Phases below correspond to the Jira Epics tracked under OCPSTRAT-3724.

### Dev Preview -> Tech Preview

* The mechanism is implemented: install-config API, artifact override, unknown-kind
  handling, user-supplied controller start, hooks, metadata, teardown, and the
  `extract cluster-api` helper.
* A reference example using an embedded provider (AWS/CAPA) installs and destroys a cluster
  unattended.
* Partner contract documentation exists: what a partner must supply, the hook contract, the
  manifest contract, and an honest statement of limitations.
* CI covers the reference arm as a presubmit and a periodic.
* No-regression evidence for integrated platforms exists for each shared-code change.
* Open Questions 1, 2, 5 and 6 have written, agreed answers.
* End-to-end testing and documentation are complete; the feature is behind a feature gate.

### Tech Preview -> GA

* A provider that is **not** embedded in the installer installs and destroys a cluster
  unattended, in CI, continuously.
* The partner delivery, validation and support model is agreed and recorded once, shared
  with OCPSTRAT-1322 — including credential handling, versioning, skew policy and the
  support boundary.
* Open Questions 3, 4, 7, 8, 9 and 10 are resolved.
* Upgrade, downgrade and version-skew behaviour is specified and tested.
* Support procedures and must-gather coverage exist.
* User-facing product documentation is published.
* Sufficient time for feedback; available by default; conformance passes.

### Removing a deprecated feature

Not applicable: this enhancement deprecates nothing. `platform: external` clusters
installed by Assisted Installer or UPI remain supported and unaffected.

## Upgrade / Downgrade Strategy

Nothing from this enhancement is installed into the cluster, so there is nothing in it to
upgrade. The partner-supplied provider runs only in the installer's local, ephemeral
Cluster API control plane and is gone before the install completes. A cluster created this
way is, from the upgrade machinery's point of view, an ordinary `platform: external`
cluster and upgrades exactly as one.

Two real consequences follow:

* **Cluster metadata must remain readable by older and newer installers.** The new
  `External` member is additive and optional; an installer that does not know about it
  ignores it, and an installer that does know about it tolerates its absence. A
  `destroy cluster` run with a binary older than the one that created the cluster will not
  find the provider and will not tear the infrastructure down — this must be documented,
  and it is why the provider is recorded in metadata at all.
* **Partner components installed day-0 by hooks — cloud controller manager, CSI driver —
  are upgraded on the partner's cadence, not OpenShift's.** That is a property of
  `platform: external`, not something introduced here, but the delivery model work must
  state it explicitly.

## Version Skew Strategy

The relevant skew is between the Cluster API core version the installer runs and the
Cluster API version a partner's provider was built against. Across the pilot's runs a
one-minor-version difference caused no observable problem, so no check is implemented and
none is proposed yet.

What is unresolved, and belongs to the delivery model work rather than to the installer:
what skew the project is willing to support, whether anything should warn on skew, and
what happens when core Cluster API moves and a partner provider has not followed. An
admin-ack-style gate has been suggested for the day-2 case; for day-0 the failure is
contained — the install fails before a cluster exists — which is why this is a
documentation and policy question rather than a code one.

There is no skew between installer and cluster components introduced here, because this
enhancement adds no cluster component.

## Operational Aspects of API Extensions

This enhancement adds **no API extension in the operational sense**: no CRD, no aggregated
apiserver, no admission or conversion webhook, no finalizer, no new controller in the
cluster. The only API change is an optional field in `install-config.yaml`, which is
consumed by the installer binary and never served, persisted or reconciled.

Consequently there is no steady-state CPU, memory or API-call impact on an installed
cluster, no new failure mode for the cluster's API server, and no SLO to state for cluster
operations.

The operational impact that does exist is confined to the install itself: a failed
partner-supplied provider fails the install, with no cluster created and no partial state
beyond whatever the provider created in the partner's cloud before failing. Teardown of
that partial state is the `destroy cluster` path, which is in scope and tested.

## Support Procedures

**Determining that this path was used.** `metadata.json` in the install directory records
the external platform and the Cluster API provider name. The installer log records, for
each artifact, its path, its source and its SHA-256 — never its content.

**Detecting failure.** An install using this path fails in one of four places, in order:

1. *Install-config validation* — the provider stanza is malformed, a path does not exist,
   the binary is not executable, or it was built for the wrong architecture. Fails before
   any cloud call. Symptom: a `field.ErrorList` error naming the field.
2. *Provider start-up* — the controller exits immediately, typically on an unrecognised
   flag. Symptom: the local Cluster API control plane reports the provider unhealthy and
   `/healthz` never answers. This is the most common first-time failure and it is a partner
   provider issue, not an installer issue.
3. *Infrastructure provisioning* — `Cluster.status.infrastructureReady` never becomes true.
   The authoritative detail is in the provider's own logs and in the status conditions on
   the provider's infrastructure resource, both captured in the installer's Cluster API
   output directory. **This is partner code.**
4. *A hook* — a non-zero exit fails the install with the hook's exit code.

**Routing a case.** Failures at (1) are installer issues. Failures at (2) and (3) are
almost always partner-provider issues and the provider name and artifact hash in the log
identify which component and which build. Failures at (4) are in a program the user
supplied. Support should not attempt to debug partner provider internals; it should
establish which of the four stages failed and route accordingly.

**Known gap.** Worker CSRs are not automatically approved on `platform: external`. A
cluster that reaches a healthy control plane but whose workers never join is most likely
hitting this, not a provider defect; `oc get csr` will show pending requests.

**Consequences of disabling.** Removing the `clusterAPI` stanza from a future install
simply returns that install to the pre-existing behaviour — `platform: external` with no
installer-native provisioning. Existing clusters are unaffected, because nothing from this
path runs in them. The one irreversible consequence is that an installer without this
support cannot destroy a cluster created with it.

## Infrastructure Needed [optional]

* Prow CI capacity for two end-to-end install arms, one on an integrated-provider cloud
  account and one on a partner platform, including credential management for the latter.
* A cluster profile and account for the partner platform.
* A published, generic, multi-tenant boot image for the partner platform — see Open
  Question 11. Without it, the second CI arm cannot run continuously.
