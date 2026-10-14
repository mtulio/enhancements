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
  - "@patrickdillon"
api-approvers: # No cluster API change: install-config only, no CRD, aggregated apiserver, webhook or finalizer. See "Operational Aspects of API Extensions".
  - None
creation-date: 2026-10-06
last-updated: 2026-10-06
status: provisional
tracking-link: # link to the tracking ticket that corresponds to this enhancement
  - https://issues.redhat.com/browse/OCPSTRAT-3724
see-also:
  - "/enhancements/cloud-integration/infrastructure-external-platform-type.md"
  - "/enhancements/installer/bootstrapping-clusters-with-capi-providers.md"
  - "/enhancements/cloud-integration/out-of-tree-provider-support.md"
  - "/enhancements/cluster-api/installing-cluster-api-components-in-ocp.md"
  # Day-2 shared DNS/LB contract; candidate replacement for part of this EP's hook surface.
  - "https://github.com/openshift/enhancements/pull/2138"
  # Earlier, unmerged attempt at DNS/LB on None and External with per-platform fields.
  - "https://github.com/openshift/enhancements/pull/1666"
replaces: []
superseded-by: []
---

# Install a platform External cluster with a partner-supplied Cluster API provider

## Summary

`platform: external` today has no installer-native provisioning path.
A partner who already maintains a Cluster API infrastructure provider
cannot use it at install time, so clusters on their platform are stood
up by Assisted Installer or UPI and the infrastructure is created
outside the installer's control.

This enhancement lets `openshift-install create cluster` provision
infrastructure for a `platform: external` cluster by running a Cluster
API provider **out-of-tree**. The provider's controller binary and
component manifests are supplied by the user in one of two mutually
exclusive forms: a provider-published container image payload, which is
the intended form, or filesystem paths, which is what the pilot proved.
The installer's existing local, ephemeral Cluster API control plane runs
them exactly as it runs the providers it embeds. Three optional user-supplied hooks
cover the cloud-specific work the installer cannot do, because on
`platform: external` it holds no cloud credentials by design.

The design is deliberately provider-agnostic: there is no Provider-specific
(OCI-specific, AWS-specific, etc) code path. It has been exercised end to
end on two providers, one reference provider embedded in the
installer (AWS/CAPA) and one that with no code specific on OpenShift
ecosystem: Oracle (CAPOCI).

## Motivation

Partners and customers deploying OpenShift on an external platform get a materially
different install experience from integrated platforms. Oracle Cloud Infrastructure is the concrete driver, but the gap is general: any partner with a working Cluster API provider carries the whole provisioning burden themselves.

`OCPSTRAT-1322` covers the day-2 half of this problem — enabling a partner CAPI
provider in-cluster for autoscaling and machine health checks. It explicitly
does not cover install-time provisioning. This enhancement is the day-0 half.

### User Stories

* As a cloud provider partner with an existing Cluster API infrastructure provider,
  I want OpenShift's installer to run my provider during install, so that my customers
  get an IPI-like experience without Red Hat writing and maintaining in-tree support
  for my platform.
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
   giving filesystem paths to its controller reference, image or binary, and component manifests.
2. Resolution of those paths in place of the embedded artifact archive, with validation
   before any side effect.
3. Acceptance of Cluster API manifests whose `kind` has no compiled-in Go type, scoped to
   the external provisioning path.
4. An optional `hooks` stanza with three lifecycle points — `infraReady`, `postProvision`,  `preDestroy` — for cloud-specific work the installer cannot perform.
5. Recording of the provider in cluster metadata, so `destroy cluster` can tear down
   through the same provider.
6. A hidden `openshift-install extract cluster-api <provider>` command that writes an
   embedded provider's artifacts to a directory, so a partner can see the exact shape the
   contract expects. (not mandatory, well used in the Pilot to reproduce stable CAPI with platform type external in integrated providers)

   **Decided: `extract` emits a complete reference implementation, in input layout.** Today
   it writes two things — the controller binary and
   `<name>-infrastructure-components.yaml` (`pkg/clusterapi/extract.go:95,105`). That is
   enough to prove the artifacts exist and not enough to be a worked reference. The
   intended behaviour is that `extract` writes **the binary, the provider's infrastructure
   CRDs and components, and the cluster manifests** for the named provider, laid out in
   **exactly the directory shape the installer expects as input** — so that a partner's
   first action is to diff their own tree against it, and their second is to run it.
   Concretely, the output directory is an install directory skeleton: provider artifacts
   where `binaryPath`/`componentsPath` would point, Cluster API manifests under
   `cluster-api/`, day-0 cluster manifests under `openshift/`, and the matching
   `install-config.yaml` fragment.

   Two things follow and are not yet settled. The cluster manifests are asset-graph output
   (`pkg/asset/manifests/clusterapi/cluster.go:212`), not embedded files, so producing them
   requires either a minimal install-config as input to `extract` or templated manifests
   with placeholder substitution; and whichever is chosen fixes the answer to "do partners
   ship manifests or manifest templates?" below. See Open Question 1.

### Workflow Description

**Partner provider developer** is a human who maintains a Cluster API infrastructure provider for a platform that OpenShift does not integrate in-tree.

**Cluster creator** is a human or automation installing OpenShift on that platform.

Provider preparation, done once by the partner provider developer:

1. Build the provider's controller binary for the installer's host architecture.
2. Collect the provider's component manifests — CRDs, RBAC, webhook configuration — into a directory.
3. Optionally run `openshift-install extract cluster-api aws` to see the layout an embedded provider uses as a worked reference.
4. Write the Cluster API manifests for a cluster on that platform, and the optional hook
   programs.
> How the partner will distributed those images? will they distribute the cluster manifests? will it be templates? can installer render those templates to save install prefix name, for instance? (prior gen manifests)

Install time, done by the cluster creator:

1. Write `install-config.yaml` with `platform.external.clusterAPI` pointing at the binary and components directory.

  **Option 1 - Local Binary strategy (PILOT)**
  Both fragments below are from pilot runs, with environment-specific values replaced by
  `CHANGE-ME`. The whole partner contract is the `clusterAPI` stanza; everything above and
  below it is ordinary `platform: external`.

   **AWS, with Cluster API Provider AWS (CAPA).** The reference arm. Note that `args` is
   what carries provider-specific flags the installer knows nothing about — here the
   feature gate CAPA requires before it will accept `AWSMachine.spec.ignition` at all.

   ```yaml
   platform:
     external:
       platformName: aws
       cloudControllerManager: External
       clusterAPI:
         name: aws
         binaryPath: /CHANGE-ME/capa-artifacts/cluster-api-provider-aws
         componentsPath: /CHANGE-ME/capa-artifacts/aws-infrastructure-components.yaml
         hooks:
           # creates api/api-int DNS; nothing else on this platform does
           infraReady:
             program: hooks/infra-hook.sh
           # creates the *.apps wildcard, which cannot be done at infraReady:
           # it points at a load balancer the cluster's own CCM builds minutes
           # into bootstrap, in response to a day-0 extra manifest
           postProvision:
             program: hooks/infra-hook.sh
             args:
             - --input-service=openshift-ingress/router-external-default
             - --input-dns-zone=CHANGE-ME
           # removes both again; neither is owned by Cluster API, so deleting
           # the Cluster will not touch them
           preDestroy:
             program: hooks/infra-hook.sh
         args:
         - --feature-gates=BootstrapFormatIgnition=true,ExternalResourceGC=true
   ```

   **OCI, with Cluster API Provider OCI (CAPOCI).** The arm that evidences the central
   claim, because CAPOCI is not embedded in the installer. Two differences are worth
   reading: `binaryPath` is a shell wrapper, which is legal on purpose
   (`pkg/clusterapi/artifacts.go:195`), and no feature gate is needed because CAPOCI has no
   opinion about bootstrap data format.

   ```yaml
   platform:
     external:
       platformName: oci
       cloudControllerManager: External
       clusterAPI:
         name: oci
         # NOT the CAPOCI binary: a wrapper translating the installer's
         # --health-addr into the provider's --health-probe-bind-address, the
         # one installer-supplied flag CAPOCI does not declare. It must
         # translate rather than drop the flag -- the installer polls /healthz
         # at exactly the address it passed.
         binaryPath: /CHANGE-ME/capoci-artifacts/capoci-shim.sh
         componentsPath: /CHANGE-ME/capoci-artifacts/oci-infrastructure-components.yaml
         hooks:
           # creates api/api-int DNS; adds a 22623 listener and backend set to
           # the API load balancer, which CAPOCI has no field for; and writes
           # the CCM and CSI config Secrets, whose inputs are OCIDs that do not
           # exist until Cluster API has built the network -- so they cannot be
           # day-0 manifests
           infraReady:
             program: hooks/infra-hook.sh
           postProvision:
             program: hooks/infra-hook.sh
             args:
             - --input-service=openshift-ingress/router-external-default
           preDestroy:
             program: hooks/infra-hook.sh
         args:
         - --init-oci-clients-on-startup=false
   ```

   On both platforms `compute[].replicas` is `0`: the installer generates no MachineSets
   for `platform: external` (`pkg/asset/machines/worker.go:807`), so day-0 workers are
   Cluster API `Machine` objects in `<install-dir>/cluster-api/`, not a compute pool.

  ___**Option 2 - CAPI Installer Image strategy**___
  **FOO provider with CAPFOO**
  
  Option with a integrated custom image built and maitained by FOO provider
  signed to resolve provider-specific dependencies such as CAPI version skew,
  infrastructure hooks (to resolve resources not supported by CAPI), etc

  ```yaml
  platform:
    external:
      platformName: foo
      cloudControllerManager: External
      clusterAPI:
        name: foo
        # Directory of the root where the CAPI assets will be to be consumed by installer.
        # Default: <install-dir>/capi-installer
        # The directory will have:
        # bin/controller
        # crd/infrastructure.yaml
        # hooks/infraReady-*
        # hooks/postProvision-*
        # hooks/preDestroy-*
        capiInstallerPath: <install-dir>/capi-installer

        # controller source: image or local
        # Can be: image or local
        # when image it's expected all resources be inside the image.
        controller:
          # image or local
          sourceType: image
          # OCI image or local path
          # When imagem resources will be extracted to capiInstallerPath
          image: quay.io/foo/openshift-capfoo:latest
          args: []

        # Single bundle manifest the local control plane loads. Decided: a single
        # file, not a directory -- every clusterctl provider already publishes
        # exactly one infrastructure-components.yaml.
        # When using the image it must be in the root.
        customResource: <install-dir>/capi-installer/infrastructure-components.yaml

        # Additional manifests directory that will be
        # automatically copyed to the installer manifest directory to rendered on boot.
        # Rendered with clusterctl's substitution -- ${VAR}, ${VAR=default},
        # ${VAR:="default"} -- over INSTALLER_-prefixed variables. An
        # unresolvable variable is a validation error, not a pass-through.
        manifestPath: <install-dir>/capi-installer/manifests

        # hooks is automatically filled to defaults.
        # change it when needs to override path and args.
        hooks:
          infraReady:
            program: <install-dir>/capi-installer/hooks/infraReady
            args: []
          postProvision:
            program: <install-dir>/capi-installer/hooks/postProvision
            args: []
          preDestroy:
            program: <install-dir>/capi-installer/hooks/preDestroy
            args: []

  ```

  **How the payload is obtained and used.** In order:

  1. The partner publishes an OCI image payload through their own official release
     process.
  2. The installer pulls it, or the user pulls it beforehand and points at a local copy.
  3. The installer extracts the image content — controller binary, infrastructure
     components, manifests, hooks — into `capiInstallerPath`, which defaults to
     `<install-dir>/capi-installer`. Every other path in the stanza is defaulted beneath
     it, so a conforming payload needs no path configuration at all.
  4. The installer renders the manifest templates, substituting its own
     `INSTALLER_`-prefixed variables — `INSTALLER_INFRA_ID`, `INSTALLER_BASE_DOMAIN` and
     the rest of the published set — using clusterctl's `${VAR}` syntax. A variable the
     installer cannot resolve fails validation before any cloud call.

  **On signing and trust.** The payload is built, maintained and shipped by the provider
  through their own release process, and that is the trust boundary. **The installer
  validates the payload's *structure*, not its *content*.** It checks that the expected
  layout is present and that the declared compatibility range admits this release; it does
  not inspect, scan or attest the controller, the manifests or the hooks. Correctness of
  what is inside is the partner's responsibility, established by their release process and
  by the validation model in the partner delivery work — not by an installer-side check.

  **The payload layout is clusterctl's, not a new one.** Before inventing a format it is
  worth noting that the three things this option needs are already conventions every
  Cluster API provider follows, because `clusterctl` requires them of a provider
  repository. Verified in both pilot providers:

  | Need | Existing convention | Evidence |
  | --- | --- | --- |
  | One components bundle | `infrastructure-components.yaml`, a single file | why `customResource` is a file, not a directory |
  | Declared contract version | `metadata.yaml`, `apiVersion: clusterctl.cluster.x-k8s.io/v1alpha3`, `kind: Metadata`, mapping each release series to a Cluster API contract (`v1beta1`, `v1beta2`) | `cluster-api-provider-aws/metadata.yaml`, `cluster-api-provider-oci/metadata.yaml` — both present and published today |
  | Manifest templates | `cluster-template*.yaml` with `${VAR}` substitution, including `${VAR=default}` and `${VAR:="default"}` | `cluster-api-provider-oci/templates/cluster-template.yaml` uses `${CLUSTER_NAME}`, `${KUBERNETES_VERSION}`, `${NAMESPACE}` among others |

  So the payload should **reuse `metadata.yaml` verbatim** for Cluster API contract skew.
  A partner publishes it already; requiring it costs them nothing and the installer gets
  the one skew answer that matters most — which Cluster API contract the controller
  implements — from a file whose semantics are defined upstream rather than by us.

  **What clusterctl does not cover, and we therefore still need.** `metadata.yaml` maps
  release series to a *Cluster API contract*. It says nothing about Kubernetes or
  OpenShift versions, and nothing about where the hooks and the controller binary live —
  neither concept exists upstream. That residue is small enough to carry in a second file
  at the payload root, deliberately shaped like `metadata.yaml` so the two read as a pair:

  ```yaml
  # <capiInstallerPath>/openshift.yaml
  apiVersion: external.openshift.io/v1alpha1
  kind: ProviderPayload
  # Compared against the Kubernetes version of the release payload being installed;
  # the install is refused outside the range. The Cluster API contract comes from
  # the adjacent clusterctl metadata.yaml and is not repeated here.
  compatibility:
    kubernetes:
      min: "1.32"
      max: "1.34"
  # Paths relative to capiInstallerPath, so the installer validates structure rather
  # than guessing it. This is the OpenShift-specific part: clusterctl has no notion
  # of a controller binary on disk or of install-time hooks. Validation is
  # exhaustive: every declared path must exist, the controller must be executable,
  # and an executable the payload did not declare is rejected.
  layout:
    controller: bin/controller
    customResource: infrastructure-components.yaml
    manifestTemplates: manifests/
    hooks:
      infraReady: hooks/infraReady
      postProvision: hooks/postProvision
      preDestroy: hooks/preDestroy
  ```

  Image labels were considered as an alternative home for the same data and rejected: a
  label is queryable without extracting, but it cannot carry the layout map and cannot be
  versioned by `apiVersion`, so the payload would end up with two sources of truth for
  one contract.

  **Decided on this option:**

  * **`controller.sourceType` is a union.** Exactly one of `image` or `local`. When
    `image` is set, every artifact comes from the payload; local paths are not consulted.
    Options 1 and 2 are therefore two modes of one field set, not two field sets.
  * **`customResource` is a single file.** `customResourcePath` is dropped. Cluster API
    providers publish exactly one `infrastructure-components.yaml`; a directory contract
    would be a generalisation with no known consumer.
  * **An unresolvable template token fails, at validation.** Before any cloud call and
    before the controller starts, consistent with how every other install-config error
    surfaces, and with clusterctl's own behaviour on an unset required variable.
  * **Credentials for a private payload are the user's problem, not a new mechanism.**
    Either the user pulls the image locally beforehand, or they append the partner's
    registry credentials to the existing `pullSecret`. No new install-config field.
  * **A tag is accepted; the reference must be fully qualified.** Starting with tags keeps
    the first implementation simple. The install log records the resolved digest and the
    artifact SHA-256, so a tagged install is still traceable after the fact even though it
    is not reproducible from the install-config alone.
  * **Hooks cannot be overridden with a local program when `sourceType: image`.** A user
    may point at a different path *inside* the payload, but may not mix a payload
    controller with a host-local hook. This keeps the payload the single unit of trust and
    keeps the existing containment rule (`pkg/types/external/validation/platform.go:105`)
    meaningful — it is simply re-anchored from the install directory to the extraction
    root.

  * **Templating follows clusterctl, not Go templates, and not kustomize.** The payload
    already borrows `metadata.yaml` and the single-bundle convention from clusterctl, and
    every provider's existing `cluster-template*.yaml` already uses `${VAR}`,
    `${VAR=default}` and `${VAR:="default"}`. Adopting the same syntax means a partner
    maintains one template dialect across both their upstream and their OpenShift
    artifacts. Kustomize is not used: it is a much larger contract than variable
    substitution, and it does not naturally express "fail if this value is unset", which
    is the behaviour decided above.
  * **Variables are uppercase with an `INSTALLER_` prefix** — `INSTALLER_INFRA_ID`,
    `INSTALLER_BASE_DOMAIN`, and so on. The prefix keeps the installer-supplied set
    distinct from a partner's own variables in the same file, so the two namespaces can
    grow independently. The exact set is the partner-facing contract and is enumerated
    when the layout is published; adding a variable is compatible, renaming one is not.
  * **`openshift.yaml` stays a file; image labels are dropped.** This was decided by
    reading clusterctl's own `metadata.yaml`: its only top-level keys are `apiVersion`,
    `kind` and `releaseSeries`, and each release series entry carries only `major`,
    `minor` and `contract`. It answers "which Cluster API contract does this provider
    release implement" and says nothing about Kubernetes versions and nothing about
    layout, so it does not resolve the skew question on its own. The residue therefore
    needs a home that can be versioned and extended, which a label set is not.
  * **A mirrored payload must be referenced by digest.** `imageDigestSources` is
    digest-oriented by name and by its deprecated predecessor's split — `imageContentSources`
    was the one that handled tags. Rather than reintroduce tag mirroring, the rule is
    narrow: tags are accepted for a connected install, and a disconnected or mirrored
    install must pin the payload by digest. Validation enforces this when
    `imageDigestSources` is set.
  * **`layout` validation is exhaustive.** Every path the payload declares must exist,
    the controller must be executable, and anything executable that was *not* declared is
    rejected. This is what makes "the installer validates structure, not content" a
    guarantee rather than a slogan: a payload cannot carry unannounced executables. It is
    decided now rather than later because tightening the rule after publication breaks
    every partner already shipping.

  **Disconnected and mirrored installs reuse `imageDigestSources`.** This is not a new
  problem. `install-config.yaml` already carries `imageDigestSources`
  (`pkg/types/installconfig.go:157`), which is how a mirrored install redirects the
  release image before any cluster exists; the installer converts it into the in-cluster
  `ImageDigestMirrorSet` afterwards (`pkg/asset/manifests/imagedigestmirrorset.go:45`).
  The provider payload resolves through the same field. `pullSecret` covers
  authentication and `imageDigestSources` covers redirection; neither needs a new
  mechanism.

2. Place the provider's Cluster API manifests in `<install-dir>/cluster-api/` and any day-0 cluster manifests in `<install-dir>/openshift/`.
3. Run `openshift-install create cluster`.
4. The installer resolves and validates the provider artifacts, starts the local Cluster API control plane, installs the provider's components, runs the provider's controller, and applies the user's Cluster API manifests.
5. On `Cluster.status.infrastructureReady`, the `infraReady` hook runs — typically to create DNS records or publish the bootstrap ignition somewhere the platform can reach.
6. Machines are provisioned; bootstrap completes; the Cluster API control plane is torn down.
7. The `postProvision` hook runs against the new cluster's kubeconfig, applying the partner's cloud controller manager, CSI driver and related configuration.
8. Install completes.

Teardown:

1. `openshift-install destroy cluster` reads the provider from cluster metadata.
2. The `preDestroy` hook runs.
3. The provider is restarted locally, its credential Secret re-applied, and the infrastructure custom resources are deleted so the provider's finalizers can run.

#### Variation: platforms that cannot carry the bootstrap ignition in instance metadata

`bootstrap.ign` is typically hundreds of kilobytes and many clouds cap instance user-data well below that. Such platforms run the install in three phases rather than two: `create manifests`, `create ignition-configs`, then — in partner tooling, before `create cluster` — upload `bootstrap.ign` to object storage, mint a pre-signed URL, and replace the on-disk file with a small pointer-ignition document. The installer reloads the file during asset resolution, so this requires no installer change.

This works, but it means **every partner reimplements the same credential hazard**: the pre-signed URL grants unauthenticated read over every secret in the bootstrap ignition.

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

This is install-config only. It is not a cluster API: it is not served by the API server, it is not persisted into the cluster, and it has no CRD, webhook, aggregated apiserver or finalizer. Validation is ordinary install-config validation returning a `field.ErrorList` (`pkg/types/external/validation/platform.go`).

The agent-based installer shares the install-config format but routes data through
assisted-service's cluster manifests, and this field is not passed through. Per the
install-config conventions, agent-based validation **warns** when the field is set to a non-default value rather than ignoring it silently.

`ClusterPlatformMetadata` gains an `External` member (`pkg/types/clustermetadata.go:46`) so `destroy cluster` can find the provider. It is written conditionally (`pkg/types/clustermetadata.go:89`); `platform: none` continues to write nothing.

Whether this is the right long-term shape — paths on a local filesystem versus an image reference, a CRD bundle, or a user-supplied manifest in the style of a cloud controller manager override — is an open question below. The pilot used paths because they are the smallest thing that proves the mechanism.

#### Proposed shape: a provider payload image

**Design sketch, not implemented.** The direction is to replace the loose filesystem paths
with a single provider-published OCI payload image, extracted by the installer into one
standard directory, with every other path defaulted beneath it. This is the Option 2
fragment in *Workflow Description*, expressed as Go:

```go
// PROPOSED, not present at pilot commit c103ca8c90.
type ClusterAPIProvider struct {
    Name string `json:"name"`

    // CAPIInstallerPath is the extraction root. Defaults to
    // <install-dir>/capi-installer. Every path below defaults beneath it, so a
    // conforming payload needs no path configuration.
    CAPIInstallerPath string `json:"capiInstallerPath,omitempty"`

    Controller ControllerSource `json:"controller"`

    // CustomResource is the single components bundle the local control plane
    // loads -- a file, not a directory: clusterctl providers publish exactly one
    // infrastructure-components.yaml.
    CustomResource string `json:"customResource,omitempty"`

    // ManifestPath is one directory, copied into the installer's manifest
    // directory and rendered on the way. Templating is not a separate mode: an
    // unresolvable variable fails validation. Substitution is clusterctl's --
    // ${VAR}, ${VAR=default}, ${VAR:="default"} -- over installer-supplied
    // variables named INSTALLER_INFRA_ID, INSTALLER_BASE_DOMAIN and so on.
    ManifestPath string `json:"manifestPath,omitempty"`

    // Defaulted from the payload layout; set only to override.
    Hooks *Hooks `json:"hooks,omitempty"`
}

// ControllerSource is a discriminated union: SourceType selects exactly one of
// Image or BinaryPath, and validation rejects the other being set.
type ControllerSource struct {
    // SourceType is "image" or "local".
    // +kubebuilder:validation:Enum=image;local
    SourceType string `json:"sourceType"`

    // Image is set when SourceType is "image". A fully qualified reference; a tag
    // is accepted. Everything else comes from the payload and no local path is
    // consulted, including hooks.
    Image string `json:"image,omitempty"`

    // BinaryPath is set when SourceType is "local". This is the pilot behaviour.
    BinaryPath string `json:"binaryPath,omitempty"`

    Args []string `json:"args,omitempty"`
}
```

This is still install-config only — it adds no cluster-serving API and does not change the
`api-approvers: None` position below. It does change the installer's responsibilities
materially: pulling and extracting an image, rendering templates, and validating a
declared payload layout are three things the pilot does not do. The unresolved parts of
the shape are the bullets under Option 2 in *Workflow Description*; they are a precondition
for implementing it, not for merging this enhancement.

#### No cluster API change is proposed, and why

**This enhancement proposes no change to the [Infrastructure][infra-api] config API, or to
any other cluster-serving API.** `api-approvers: None` in the frontmatter is deliberate and
accurate. The reason is the ordinary API-review test: a status field exists so that a
*controller* can act on it, and today nothing in the cluster would read one. Everything the
install produces is consumed by the installer itself — the provider name reaches
`destroy cluster` through `metadata.json`
(`pkg/types/clustermetadata.go:46`), and the artifact paths and SHA-256 sums reach support
through the install log. Adding a field now would mean shipping a permanently-set,
never-read value into a GA'd API, and the `Infrastructure` CVO-managed singleton is close
to the worst place to do that: see the immutability rules quoted below.

**The precedent that will apply when day-2 arrives.** `cloudControllerManager.state` is the
exact shape this will need, and it is worth reading it now because it was designed under
the same constraint. The install-config carries intent:

```yaml
platform:
  external:
    platformName: oci
    cloudControllerManager: External
```

The installer translates that once, at manifest-generation time, into Infrastructure
*status* — it is never reconciled afterwards
(`pkg/asset/manifests/external/infrastructure.go:19-36`):

```go
return &configv1.ExternalPlatformStatus{
    CloudControllerManager: configv1.CloudControllerManagerStatus{
        State: ccmState, // "External" or "None"
    },
}
```

which lands in the cluster as:

```yaml
apiVersion: config.openshift.io/v1
kind: Infrastructure
metadata:
  name: cluster
status:
  platform: External
  platformStatus:
    type: External
    external:
      cloudControllerManager:
        state: External
```

and *is* read by controllers — that is the whole point of it. `state: External` is what
causes new nodes to be tainted `node.cloudprovider.kubernetes.io/uninitialized` until the
partner's cloud controller manager initialises them, so the machine-config and node
lifecycle paths depend on it.

**Two API constraints the day-2 design must respect**, both visible on the CCM field today
(`vendor/github.com/openshift/api/config/v1/types_infrastructure.go:419-444`):

* `State` carries `+kubebuilder:validation:XValidation:rule="self == oldSelf"`,
  *"state is immutable once set"*, and `ExternalPlatformStatus` carries
  *"cloudControllerManager may not be added or removed once set"*.
* Therefore **a field shaped like `clusterAPI.state` cannot be adopted on day 2.** A
  cluster installed before the field exists can never gain it. If day-2 enablement of a
  Cluster API provider on an existing `platform: external` cluster is a requirement — and
  OCPSTRAT-1322 suggests it is — then copying the CCM shape literally forecloses it, and
  the field must either be mutable from unset, or live somewhere other than the
  `Infrastructure` singleton.

**The sketch, for the follow-up EP only.** What a day-2 consumer would plausibly want is
provenance, not a boolean — which provider provisioned this cluster and from which
artifacts, so that a cluster-capi-operator reconciling day-2 machines knows which
controller image to run:

```yaml
status:
  platformStatus:
    type: External
    external:
      cloudControllerManager:
        state: External
      # NOT PROPOSED HERE. Illustrative shape for the day-2 follow-up.
      clusterAPI:
        name: foo
        providerImage: quay.io/foo/openshift-capfoo@sha256:CHANGE-ME
```

This is a design sketch, not an API proposal. It is out of scope for this enhancement and
belongs to whichever enhancement first names a controller that reads it.

**On placement, if and when it is proposed.** [#2138][ep-2138] argues — correctly — that
configuration repeated beneath each platform discriminator is the duplication mistake
already made in the on-prem networking APIs, and places its new DNS and load-balancer
fields as siblings of `platformSpec`/`platformStatus`. That reasoning does not extend to
this field, and the distinction is worth stating once rather than rediscovering it in API
review. DNS and endpoint ownership are a *concern* that every platform has and that no
platform should have to re-declare. "Which Cluster API provider provisioned this cluster,
and from which artifacts" is *platform identity itself*: it is meaningless on every
platform except External, it has no cross-platform analogue to converge on, and it sits
beside `cloudControllerManager.state`, which is already scoped this way for the same
reason. The test is whether a second platform would ever want the same field; for DNS
policy the answer is yes, for external provider provenance it is no.

> TODO #2: We must ensure the CAPI objects of local control plane are saved in the cluster, even if it would be read-only, or as a dump of objects in a ConfigMap/Secret.
> Note: there is already a precedent and a partial implementation of this. `kube-system/cluster-config-v1` is created from the redacted install-config (`pkg/asset/manifests/operators.go:123,352`), and because the redaction switch has cases only for vSphere and Nutanix credentials (`:363,:375`), the whole `platform.external.clusterAPI` stanza — provider name, binary path, components path, args, hook programs — is already carried into the cluster today, unredacted. That is a ConfigMap, needs no API review, and is the cheapest place to put a dump of the local Cluster API objects. It is also an unchosen finding rather than a decision: whether partner hook arguments belong in a world-readable ConfigMap has not been reviewed.

[infra-api]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/config_apis/infrastructure-config-openshift-io-v1

### Topology Considerations

#### Hypershift / Hosted Control Planes

Not applicable in this enhancement. HyperShift does not use `openshift-install` to
provision a hosted cluster's infrastructure, so the install-time path described here has no HyperShift analogue. `platform: external` support on hosted control planes is being investigated separately and is explicitly out of scope here; if it proceeds it will need its own enhancement, because the mechanism — not just the configuration — differs.

#### Standalone Clusters

This enhancement targets standalone clusters exclusively. A `platform: external` cluster installed this way is an ordinary standalone cluster whose infrastructure happened to be created by a partner-supplied Cluster API provider.

#### Single-node Deployments or MicroShift

Not applicable. MicroShift does not use `openshift-install`. Single-node OpenShift on `platform: external` is not prevented by anything here — the provider would simply provision one machine — but it is untested and not a goal of this work; no SNO-specific resource or footprint consideration arises, because the installer-side change adds no runtime component to the cluster.

#### OpenShift Kubernetes Engine

Not applicable. This changes the installer and adds no cluster-side component, so there is nothing whose presence or absence differs between OKE and OpenShift Container Platform. A cluster installed this way is subject to whatever OKE restrictions already apply to `platform: external`.

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

* **The installer has no cloud credentials on `platform: external`, and must not acquire any.** This is why hooks exist. Every piece of cloud-specific work — DNS records, ignition staging, load balancer attachment — is the partner's, executed through a hook, using the partner's own credentials.
* **Hook environment carries paths, not documents.** Passing a kubeconfig or an ignition body through an environment variable puts secrets into process listings and traces.
* **`infraReady` cannot write into `<install-dir>/openshift/`.** By the time infrastructure is ready the installer has already consumed and removed that tree, so partner configuration Secrets must be applied from `postProvision` against the hook's kubeconfig. This is a measured finding, not a preference.
* **A wrapper script is a legal `binaryPath`.** Cluster API providers do not agree on flag names — the installer passes `--health-addr` (`pkg/clusterapi/external.go:155`) and at least one real provider declares `--health-probe-bind-address` instead, with pflag exiting non-zero on an unknown flag. A shim that translates the flag is the supported answer today. Note that a shim must **translate**, not drop: the installer polls `/healthz` at exactly the address it passed.
* **`destroy cluster` deliberately excludes Secrets from the saved Cluster API output**, so teardown must re-apply the provider's credential Secret before the infrastructure resource's finalizer can clear.
* **The architecture of the supplied binary is checked before use.** A cross-built
  controller fails at validation rather than at exec time.
* **Artifacts are logged by path, source and SHA-256 — never by content.**

> Note/Open question: a pain point that this EP still not propose are the hooks requirement, created to cover paths that CAPI does not cover. We must enumerate clearly what are those limitations, so that we could discuss alternatives to let partner implement those interfaces/cover those gaps. Those items will compose the requirements to have an OCP clusters vs vanila K8S, or standard CAPI implementation. Possible solutions: 1) OCP engineers list gaps and provider must implement in the CAPI implementation (such as DNS, additional listeners to KAS LB, etc) [the hard way]; 2) OCP engineers make those hooks more flexible allowing partners to "customize" their IPI deployment using platform external, this would be more flexible for BYO variants.

> Note: the hooks exist to cover what Cluster API does not. Enumerating those gaps precisely — DNS, additional listeners on the API server load balancer, ingress reachability — is prerequisite work, and the DNS and endpoint subset of it may belong to the shared networking contract rather than to hooks at all. See Open Question 13. Two shapes remain open for whatever is left: Red Hat enumerates the gaps and partners implement them in their provider [the hard way], or the hooks become a flexible customisation surface for BYO variants.

### Risks and Mitigations

**Accepting unknown manifest kinds weakens an existing client-side check.** A
user-supplied `AWSCluster` or `OCICluster` has no compiled-in Go type, so the scheme rejects it. The pilot falls back to unstructured handling. Done carelessly this would turn a misspelled `kind` on an integrated platform from a loud failure into a silent one.
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

> WRT ^ we must make sure we are handling failure scenarios and providing a clear feedback to user. For example if failure happened in the CAPI call and/or hooks, we must raise it clearly how and who they need to contat. We must invest in troubleshoot steps to mitigate the risk even more.

**Partner provider quality is unverified.** Nothing checks that a supplied provider behaves
correctly. *Mitigation:* none today. A conformance suite is an open question, and the
partner validation model is a deliverable of OCPSTRAT-3724 in its own right.

**Partners infrastructure readiness.** Nothing guarantee that the a successfull installation on a provider that Red Hat did not exercised conformance test and perf && scale validations is production ready. *Mitigation:* OpenShift engineering providers interfaces and programs to allow partners to enhance quality in their installations by self-validating (openshift-tests and/or OPCT tools) or applying to programs like VCSP (Validated Cloud and Service Provider).

### Drawbacks

* It adds a supported configuration in which a significant part of the install is executed
  by code outside the OpenShift release payload, which complicates support triage even
  with good attribution.
* It touches shared installer code — manifest handling, artifact resolution, the platform
  switch — to serve a platform most users will never use. The risk to integrated platforms
  is small but not zero, which is why it is isolated into the smallest possible changes.
* Supplying providers as **filesystem paths** is awkward for automation and for disconnected
  environments. The payload image (Option 2 in the Workflow Description) is the intended
  form; paths remain as the pilot's proven mode and as a developer and CI convenience.
  Whether they stay a supported partner-facing form is Open Question 1.
* It creates an expectation that Red Hat will help partners debug their providers. The
  intent is the opposite — partners must be able to diagnose their own provider — but the
  mechanism for that is not defined here, and nor is the venue. **Which forum do partners
  outside the VCSP programme use to reach OpenShift engineering about an external
  provider?** Today there is no answer that does not route through a programme, and the
  delivery model work must either name an existing community channel or accept that the
  only path is VCSP.

## Alternatives (Not Implemented)

**Integrate each partner platform in-tree.** This is the status quo for supported platforms and it does not scale: every platform costs Red Hat ongoing maintenance, and it is exactly what `platform: external` was created to avoid.

**Embed partner providers in the installer image.** Rejected: it puts partner code in the release payload, couples the partner's release cadence to OpenShift's, and contradicts the `platform: external` contract.

**Fetch the provider from a remote reference — an image or a git URL — rather than a path.** Not implemented in the pilot, deliberately, because it adds digest pinning, pull credentials and a mirroring story before the core mechanism is proven. It remains the most likely future direction; see Open Questions.

**Let partners fork the installer.** Already possible, and the thing OCPSTRAT-3724 explicitly wants to stop.

**Keep using Assisted Installer or UPI for external platforms.** Works today, and remains available. It does not give an IPI-like experience and leaves the cluster with no machine-management layer wired in from day 0.

**Add per-platform exceptions for the known partner.** Rejected on principle: platform External exists to fit any provider, and an exception for one invites an exception for every other.

## Open Questions [optional]

1. **Does the local-path form survive Tech Preview, or does the payload image replace it?** The shape itself is settled: the Workflow Description proposes a `controller.sourceType` union of a provider-published payload image (Option 2) and the pilot's filesystem paths (Option 1), with the payload layout, templating syntax, variable vocabulary, compatibility file and validation strictness all decided there. What is not settled is whether Option 1 remains a supported, documented form for partners, or is retained only as a developer and CI convenience once payload images exist.
2. **How is the scheme fallback for unknown kinds finally scoped?** Three candidates: restrict to `platform: external`, validate against the CRDs actually present in `componentsPath`, or register partner CRDs into the scheme at runtime. The pilot does the first; the choice is not made.
3. **Should the installer provide a first-class "bootstrap ignition is staged elsewhere, here is the pointer" mechanism?** Leaving it to each partner means each partner reimplements the same credential hazard.
4. **Should `clusterAPI.args` be able to override an installer-supplied flag by name, rather than only append?** Today's answer is a per-partner wrapper script.
5. **Do we validate user-supplied providers, and how?** A conformance suite is the obvious answer and nobody has scoped one. An OTE mechanism could be created, but requires flexibility in the test framework as well, outside of scope, as well it is openshift-specific.
6. **How are failure modes surfaced?** Installer logs only, or do we need a path that explicitly signals the failing component is not Red Hat owned?
7. **Should partner cloud controller manager and CSI components be required to tag what they create?** Teardown is not supportable without it.
8. **Should node-to-node 80/443 be a stated `platform: external` networking requirement?** Currently discovered by failure.
9.  **Who supplies the cluster's ingress and API endpoint addresses on `platform: external`?** Nothing currently asks the platform for an external address: the ingress `Service` is hand-authored per provider, and the API endpoint arrives only as whatever the partner's Cluster API provider happened to configure. There are three plausible answers and they are not equivalent.
    *(a)* Every provider hand-authors the ingress `Service` and the partner's cloud controller manager reconciles it — the status quo, and the right answer
    on a cloud-like external platform.
    *(b)* The cluster owns the endpoints through the shared `networking.loadBalancer` contract of [#2138][ep-2138], with keepalived and HAProxy on reserved VIPs — plausible on an on-prem-like external platform, and already designed, though currently scoped to `platform: None`.
    *(c)* The installer derives them from the Cluster API resources the provider produced. The platform's own character decides which applies, which argues for the contract permitting more than one rather than this enhancement picking one. See Open Question 13.

10. **Should an extra manifest directory, from pilot, be allowed to contain multiple documents?** The pilot refuses them, so that a multi-document file fails loudly rather than silently applying only the first. This is an API contract and should be decided before this lands.
11. **Boot images.** `platform: external` installs need a boot image for the target platform, and for at least one partner platform no generic, multi-tenant image is published. The capability exists in RHCOS; the artifact does not. A generic artifact-override mechanism in the shape of an `OPENSHIFT_INSTALL_RHCOS_ARTIFACTS_JSON` input is the preferred direction, explicitly **not** as a per-platform special case. Tracked separately; blocks continuous CI for the second provider only.
12. **Automatic CSR approval.** There is no cloud-specific machine approver on `platform: external`, so worker CSRs are never approved and the measured cost is manual approval on every install. This pre-dates this enhancement and applies equally to `platform: none`. A webhook comparing a CSR against the instance it claims to come from is the agreed direction; it has no owner, no repository and no schedule, and the only thing settled is that it is not the installer. Is there a CAPI native approach to handle CSR approvals?
13. **Should `platform: external` consume the shared `Infrastructure` networking contract instead of DNS and load-balancer hooks?**:
    [openshift/enhancements#2138][ep-2138] proposes moving infrastructure DNS and API/ingress endpoint ownership out of the platform union into `Infrastructure.spec.networking` / `.status.networking`, with MCO rendering CoreDNS and, where requested, the on-prem keepalived/HAProxy stack. It names External as an intended future consumer but explicitly does not enable it.
    This bears directly on the hook design. Today every cloud-specific networking action is a partner-supplied program: `infraReady` typically creates DNS records, and nothing in the installer asks the platform for an external address. If External can set `networking.dns.management: ClusterManaged`, then node-level resolution of the cluster API, internal API and application names becomes product behaviour rather than a partner script, and the hook surface narrows to work that is genuinely outside the cluster — authoritative external DNS, ignition staging, and anything the partner's own cloud requires.
    Three sub-questions, none of which this enhancement can answer alone:
    * **Address source.** The shared model knows two: discovered node addresses, and administrator-reserved VIPs. A partner Cluster API provider produces a third — a provider-allocated control-plane endpoint, reported on `Cluster.spec.controlPlaneEndpoint`, which is unknown until `Cluster.status.infrastructureReady`. It is neither a node address nor a reserved VIP, and #2138 is explicit that provider-observed endpoints must not be written into administrator intent as though they were VIPs. The mapping from a Cluster API endpoint into `status.networking.endpoints` has no owner today.
    * **Bootstrap ordering.** #2138 requires that bootstrap not depend on an in-cluster controller that needs the DNS being configured. On this path the endpoint is not known at `create cluster` time, so the installer would have to resolve policy up front and reconcile the address after `infraReady` — the same constraint their None SNO installer path carries, but with a later address.
    * **Sequencing.** #2138 is provisional, has no External owner, and leaves the acceptance component (CNO or cluster-config-operator) undecided. Nothing in this enhancement's Tech Preview criteria should depend on it. The question to settle before Tech Preview is narrower: whether the hook contract is designed as permanent, or as a bridge that a later shared-networking adoption is expected to replace for the DNS case.
    Answering this also decides whether the special-purpose hooks sketched earlier — a DNS creator, a load-balancer creator — are worth defining, or whether that work belongs in the shared contract.

[ep-2138]: https://github.com/openshift/enhancements/pull/2138

## Test Plan

Detailed test plans are maintained per delivery phase in IEEE 829 form and are not reproduced here. This section states the strategy and the evidence requirements.

**Unit.** Install-config validation, artifact resolution and validation, hook argument and environment construction, metadata round-tripping, and teardown adoption logic.

**No-regression, and this is the part that matters for review.** Three changes touch code
shared with integrated platforms: the unknown-kind fallback, extra-manifest handling, and
the robustness fixes in the Cluster API bootstrap code. Each must demonstrate — not assert
— that integrated platforms are unaffected: with the new install-config field unset,
behaviour is identical to `main`, and a malformed manifest on an integrated platform still
fails loudly.

**End-to-end.** Two Prow arms:

* an AWS/CAPA arm, as the reference example and the fast signal;
* a second arm on a provider that is **not** embedded in the installer.

**The second arm is the only one that constitutes evidence for this enhancement's central claim**, and reviewers should hold it to that standard. The installer embeds CAPA, so an AWS run cannot distinguish "ran the binary it was handed" from "ran the binary it already had". A provider that cannot be embedded can. The AWS arm tests the ergonomics of the mechanism; it does not test the mechanism.

**Teardown is verified on a second cluster.** Three teardown defects in the pilot were found only by creating a cluster after a previous one had been destroyed. A single create/destroy cycle is not sufficient evidence.

**Conformance** runs on the resulting cluster, with the known CSR-approval gap handled explicitly by the job rather than silently.

### CI Job

The Prow jobs must be created using the CCM integration, user-supplied CCM on install time.

Presubmit jobs will be created in the following repositories:

- openshift/installer

Periodic jobs:

- exercising AWS provider (already included in the pilot)
- exercising OCI provider (already included in the pilot)

## Graduation Criteria

Phases below correspond to the Jira Epics tracked under OCPSTRAT-3724.

### Dev Preview -> Tech Preview

* The mechanism is implemented: install-config API, artifact override, unknown-kind handling, user-supplied controller start, hooks, metadata, teardown, and the `extract cluster-api` helper (developer mode only).
* A reference example using an embedded provider (AWS/CAPA) installs and destroys a cluster unattended.
* Partner contract documentation exists: what a partner must supply, the hook contract, the manifest contract, and an honest statement of limitations.
* CI covers the reference arm as a presubmit (optional) and a periodic (non-blocker).
* No-regression evidence for integrated platforms exists for each shared-code change.
* Open Questions 1, 2, 5 and 6 have written, agreed answers.
* Open Question 13 is answered in its narrow form only: whether the hook contract is designed as permanent, or as a bridge that a later adoption of the shared `Infrastructure` networking contract is expected to replace for the DNS case. Nothing here waits on [#2138][ep-2138] itself, which is provisional and has no External owner.
* End-to-end testing and documentation are complete; the feature is behind a feature gate.

### Tech Preview -> GA

* A provider that is **not** embedded in the installer installs and destroys a cluster unattended, in CI, continuously.
* The partner delivery, validation and support model is agreed and recorded once, shared with OCPSTRAT-1322 — including credential handling, versioning, skew policy and the support boundary.
* Open Questions 3, 4, 7, 8, 9 and 10 are resolved.
* Upgrade, downgrade and version-skew behaviour is specified and tested.
* Support procedures and must-gather coverage exist.
* User-facing product documentation is published. (do we need?)
* Sufficient time for feedback; available by default; conformance passes.

### Removing a deprecated feature

Not applicable: this enhancement deprecates nothing. `platform: external` clusters installed by Assisted Installer or UPI remain supported and unaffected.

## Upgrade / Downgrade Strategy

Nothing from this enhancement is installed into the cluster, so there is nothing in it to upgrade. The partner-supplied provider runs only in the installer's local, ephemeral Cluster API control plane and is gone before the install completes. A cluster created this way is, from the upgrade machinery's point of view, an ordinary `platform: external`
cluster and upgrades exactly as one.

Two real consequences follow:

* **Cluster metadata must remain readable by older and newer installers.** The new `External` member is additive and optional; an installer that does not know about it ignores it, and an installer that does know about it tolerates its absence. A `destroy cluster` run with a binary older than the one that created the cluster will not find the provider and will not tear the infrastructure down — this must be documented, and it is why the provider is recorded in metadata at all.
* **Partner components installed day-0 by hooks — cloud controller manager, CSI driver — are upgraded on the partner's cadence, not OpenShift's.** That is a property of `platform: external`, not something introduced here, but the delivery model work must state it explicitly.

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

**The provider payload image is the proposed answer to day-0 skew.** A payload carries the
controller, the infrastructure components, the manifest templates and the hooks as one
versioned unit, together with the two compatibility files described in the Workflow
Description: clusterctl's `metadata.yaml`, which declares the Cluster API contract the
controller implements, and `openshift.yaml`, which declares the Kubernetes version range
the payload supports. The installer compares both against the release payload being
installed and refuses the install outside the declared range, before any cloud call.

That turns skew from a documentation question into a validated one for day 0, and it is
the same standard structure that makes a generic compliance check possible at all — the
installer can verify a payload's readiness because every payload has the same shape. It
does not resolve the day-2 case, which remains with the delivery model work.

## Operational Aspects of API Extensions

This enhancement adds **no API extension in the operational sense**: no CRD, no aggregated
apiserver, no admission or conversion webhook, no finalizer, no new controller in the
cluster. The only API change is an optional field in `install-config.yaml`, which is
consumed by the installer binary and never served, persisted or reconciled.

This holds only because no change to the `Infrastructure` config API is proposed here. The
reasoning, the `cloudControllerManager.state` precedent it would follow, and the
immutability constraint that makes the CCM shape unsafe to copy are in
*API Extensions → No cluster API change is proposed, and why*. If a later revision adds an
`Infrastructure` status field, this section and `api-approvers: None` both become wrong and
must change together.

Consequently there is no steady-state CPU, memory or API-call impact on an installed
cluster, no new failure mode for the cluster's API server, and no SLO to state for cluster
operations.

The operational impact that does exist is confined to the install itself: a failed
partner-supplied provider fails the install, with no cluster created and no partial state
beyond whatever the provider created in the partner's cloud before failing. Teardown of
that partial state is the `destroy cluster` path, which is in scope and tested.

>  **How does a partner outside VCSP demonstrate continuous testing of their provider
>  against each OpenShift version?** Inside the VCSP programme this is already solved: a
>  partner validates their cluster on every OpenShift version they want tagged as
>  supported. A standalone partner has no Prow or CI running OpenShift on their platform
>  and therefore no equivalent obligation, so nothing detects a provider that has silently
>  stopped working on a newer release. **Is the answer simply that a partner wanting Red
>  Hat support must enter the VCSP flow as a validated provider, or does this enhancement
>  need a lighter-weight evidence requirement of its own?** This is a programme question,
>  not an engineering one; it belongs to the partner delivery and support model work, and
>  is recorded here because the install path is where the absence first shows.

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
>  Note: `/healthz` is the right check and is already what the installer does. It is
>  controller-runtime's own convention, which every Cluster API provider inherits, and the
>  installer polls it at exactly the `host:port` it rendered into the controller's
>  `--health-addr` argument (`pkg/clusterapi/system.go:770`, host from
>  `suggestHealthHostPort`). Querying a Kubernetes API server endpoint would not substitute:
>  at this point in the install the only API server is the installer's local, ephemeral
>  control plane, whose health says nothing about whether the provider process came up.
>  The one real constraint this places on a provider is that it must accept `--health-addr`
>  by that name, or be wrapped by something that translates it.
3. *Infrastructure provisioning* — `Cluster.status.infrastructureReady` never becomes true.
   The authoritative detail is in the provider's own logs and in the status conditions on
   the provider's infrastructure resource, both captured in the installer's Cluster API
   output directory. **This is partner code.**
4. *A hook* — a non-zero exit fails the install with the hook's exit code.
>  **Should hooks stay general-purpose, or be specialised into named hooks with their own
>  success checks?** A specialised set — a DNS creator, a load-balancer creator — could
>  each carry a bound verification (resolve the API name; reach the API endpoint), so a
>  failure says *DNS was not created* rather than *the program exited 1*. The cost is that
>  Red Hat then has to enumerate every cloud-specific action a partner might need, and a
>  partner whose platform needs something not on the list has nowhere to put it. This
>  overlaps Open Question 13 directly: if the DNS and endpoint cases move to the shared
>  `Infrastructure` networking contract, the specialised hooks lose most of their subject.
>  Note that static validation of a hook program is not an option — the installer can check
>  that it exists, is executable and is contained within the extraction root, but cannot
>  determine whether a script is well-formed without running it.

**Routing a case.** Failures at (1) are installer issues. Failures at (2) and (3) are
almost always partner-provider issues and the provider name and artifact hash in the log
identify which component and which build. Failures at (4) are in a program the user
supplied. Support should not attempt to debug partner provider internals; it should
establish which of the four stages failed and route accordingly.

**Known gap.** Worker CSRs are not automatically approved on `platform: external`. A
cluster that reaches a healthy control plane but whose workers never join is most likely
hitting this, not a provider defect; `oc get csr` will show pending requests.

>  **Should the installer poll for pending CSRs and report them, short of approving them?**
>  Reporting is cheap and does not need an owner: the installer already waits for workers,
>  so it could say *N certificate signing requests are pending approval* instead of timing
>  out silently, which is the whole of today's diagnostic cost. Approving them is a
>  different question with a different owner and is Open Question 12 — the agreed direction
>  there is a webhook that compares a CSR against the instance it claims to come from, and
>  it is explicitly not the installer. This is also not specific to Cluster API: it applies
>  to `platform: external` and `platform: none` alike, which argues against solving it in
>  this enhancement's code.

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
