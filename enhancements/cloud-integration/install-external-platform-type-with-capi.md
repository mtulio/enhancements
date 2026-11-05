---
title: install-external-platform-type-with-capi
authors:
  - "@mtulio"
reviewers:
  - "@patrickdillon, installer, for the Cluster API bootstrap control plane and the asset graph"
  - "@JoelSpeed, API and Architect group, for the platform External contract and the partner support boundary and provider packaging"
  - "@elmiko, platform External design, and cluster-capi-operator for the day-0/day-2 boundary"
approvers:
  - "@patrickdillon"
api-approvers: # Feature Gate API approval
  - "@JoelSpeed"
creation-date: 2026-10-06
last-updated: 2026-11-04
status: provisional
tracking-link:
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


platform: external` is the supported way a provider integrates OpenShift with their own cloud components — their CCM, their CSI driver — with no provider code in the release payload. What it specifies today is the running cluster. Installation stays on the agnostic path: the user supplies the infrastructure, and the installer takes it from there.
A partner who already maintains a Cluster API infrastructure provider cannot use it at install time, so clusters on their platform are stood up by Assisted Installer or UPI and the infrastructure is created outside the installer's control.

This enhancement lets `openshift-install create cluster` provision
infrastructure for a `platform: external` cluster by running a Cluster
API provider **out-of-tree**. The provider's controller binary, component
manifests, cluster and day-0 manifest templates and lifecycle hooks are
supplied as a single **provider-published container image payload**, which
the partner builds, signs and versions through their own release process.

The pilot proved the mechanism with a second form — the same artifacts as
filesystem paths on the installer host — which is implemented and retained
as developer and CI scaffolding, but is not proposed as a partner-facing
form; it is recorded under Alternatives.
The installer's existing local, ephemeral Cluster API control plane runs
them exactly as it runs the providers it embeds. Three lifecycle hooks, declared
by the payload rather than by the cluster creator, cover the cloud-specific work
the installer cannot do, because on `platform: external` it holds no cloud
credentials by design.

The design is deliberately provider-agnostic: there is no Provider-specific
(AWS-specific, OCI-specific, etc) code path. It has been exercised end to
end on two providers, one reference provider embedded in the
installer (AWS/CAPA) and one with no OpenShift-specific code at all: Oracle's CAPOCI.

## Motivation

Partners and customers deploying OpenShift on an external platform get a materially
different install experience from integrated platforms. Oracle Cloud Infrastructure
is the concrete driver, but the gap is general: any partner with a working Cluster
API provider carries the whole provisioning burden themselves.

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
4. Three lifecycle points — `infraReady`, `postProvision`, `preDestroy` — at which the
   installer runs a program the provider payload declares, for cloud-specific work the
   installer cannot perform. These are declared in the payload's own descriptor, not in
   `install-config.yaml`: a hook is an integrator's interface, not a cluster creator's.
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

Provider preparation, done once by the **partner provider developer**:

1. Build the provider's controller binary for the installer's host architecture.
2. Collect the provider's component manifests — CRDs, RBAC, webhook configuration — into
   the single `infrastructure-components.yaml` bundle `clusterctl` already requires.
3. Write the Cluster API manifest templates for a cluster on that platform, the day-0
   manifest templates their cloud needs, and the optional hook programs.
4. Write the payload descriptor, `openshift.yaml`, declaring the layout. The installer
   validates it, and it is what lets a partner name their own paths rather than adopt the
   installer's. For the AWS/CAPA reference provider:
```yaml
# Provider payload descriptor for the AWS/CAPA reference example.
#
# This file is the OpenShift-specific half of the payload contract. The other
# half is clusterctl's own metadata.yaml, which sits next to this file at the
# payload root and is copied verbatim from the provider repository by the
# Containerfile. metadata.yaml answers "which Cluster API contract does this
# provider release implement"; it says nothing about Kubernetes versions and
# nothing about where the controller binary or the hooks live, because neither
# concept exists upstream. That residue is what this file carries.
#
# Everything here describes structure. The installer validates that the
# declared layout is present and that the compatibility range admits the
# release being installed; it does not inspect, scan or attest the controller,
# the manifests or the hooks. Correctness of the content is the provider's
# responsibility, established by their release process.
apiVersion: external.openshift.io/v1alpha1
kind: ProviderPayload

# Compared against the Kubernetes version this installer binary was built
# against. Outside this range the install is refused, before any cloud call.
# Not the release image's Kubernetes version: the installer does not read one
# out of the release, and with OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE the
# two can differ. Reading it from the release instead is a design question,
# not a detail -- see Open Question 14.
#
# Only 1.36 (OpenShift 5.1) is exercised in CI today. The wider band below
# illustrates the shape of the contract; it is not a tested claim, and a
# provider publishing a real payload is asserting something their own release
# process has verified.
compatibility:
  kubernetes:
    min: "1.33"
    max: "1.37"

# Paths are relative to capiInstallerPath -- the directory the payload is
# extracted into, which defaults to <install-dir>/capi-installer. A conforming
# payload therefore needs no path configuration in install-config.yaml at all.
#
# Validation is exhaustive: every declared path must exist, the controller must
# be executable, and an executable the payload did not declare is rejected. An
# undeclared helper script is not "extra", it is a payload that fails to
# install.
layout:
  # The same path for every provider, so layout validation does not have to
  # know that AWS calls its binary `manager`. Two example builds produce it:
  # CAPA.Containerfile copies it out of the provider's own released controller
  # image, and Containerfile builds it from provider source.
  #
  # Either way the binary must be demonstrably the one the installer was
  # handed rather than one it already embedded. On AWS that distinction is the
  # whole point: the installer embeds CAPA, so a payload that reused the
  # embedded build would prove nothing. CAPA.Containerfile settles it by
  # shipping a *different*, upstream-released CAPA version from the one the
  # installer vendors -- the version logged at controller startup is then
  # proof of which binary ran.
  controller: bin/controller

  # A single file, not a directory. Every clusterctl provider already publishes
  # exactly one infrastructure-components.yaml.
  #
  # Only its CustomResourceDefinitions and webhook configurations are applied:
  # the installer runs the controller as a local OS process, so the Namespace,
  # RBAC, Deployment and credentials Secret the bundle also contains are never
  # applied to the local control plane. That is why the provider's own
  # published bundle can be used unmodified and still carry no credential -- its
  # ${AWS_B64ENCODED_CREDENTIALS} slot is inert here, and the controller takes
  # its credentials from the environment the installer starts it in.
  customResource: infrastructure-components.yaml

  # Cluster API objects -- Cluster, AWSCluster, Machine, AWSMachine -- applied
  # to the installer's local, ephemeral Cluster API control plane. Nothing in
  # this tree reaches the installed cluster.
  clusterTemplates: cluster-api/

  # Day-0 manifests folded into <install-dir>/manifests/ and applied by
  # bootstrap. These do reach the installed cluster. A different destination
  # and a different lifetime from clusterTemplates above, which is why the two
  # are separate keys rather than one rendered directory.
  manifestTemplates: manifests/

  # One script serves all three phases, dispatching on the
  # OPENSHIFT_INSTALL_HOOK environment variable the installer sets. It is
  # declared three times rather than left as an undeclared helper because an
  # executable the payload did not declare is rejected.
  #
  # Hooks are copied verbatim, never rendered. This script expands
  # ${OPENSHIFT_INSTALL_INFRA_ID}, ${REGION} and others as *shell* variables at
  # run time; substituting them at render time would destroy it.
  hooks:
    infraReady: hooks/infra-ready.sh
    postProvision: hooks/post-provision.sh
    preDestroy: hooks/pre-destroy.sh

```
5. The result is one tree. Below is the AWS/CAPA reference provider's, carrying the
   Cluster API templates, the hooks, and the day-0 manifests that deliver the cloud
   controller manager and the ingress load-balancer `Service`:
```sh
$ tree payload/
payload/
├── cluster-api
│   ├── cluster.yaml
│   └── machines
│       ├── 10_bootstrap.yaml
│       ├── 20_master.yaml
│       └── 30_worker.yaml
├── hooks
│   ├── infra-ready.sh
│   ├── lib
│   │   └── common.sh
│   ├── post-provision.sh
│   └── pre-destroy.sh
├── manifests
│   ├── 99_external-00-kubelet-providerid-master.yaml
│   ├── 99_external-00-kubelet-providerid-worker.yaml
│   ├── 99_external-02-ccm-aws.yaml
│   └── 99_external-03-ingress-nlb.yaml
└── openshift.yaml

```
6. Assemble that tree into an OCI image and publish it through their own release process.
   Nothing about the build is OpenShift-specific: the image filesystem *is* the payload,
   so an ordinary `Containerfile` that copies these paths in is the whole of it. A worked
   reference implementation, buildable and publishable end to end, is
   [cluster-api-provider-foo-openshift](https://github.com/mtulio/cluster-api-provider-foo-openshift).

Install time, done by the **cluster creator**:

1. Write `install-config.yaml` with `platform.external.clusterAPI` pointing at the
   provider payload image.

  **The proposed form: a provider-published CAPI installer image.**
  **FOO provider with CAPFOO**

  A single image, built, signed and maintained by the FOO provider, resolving
  provider-specific concerns such as CAPI version skew and infrastructure hooks (for
  resources CAPI does not model) inside one versioned artifact.

  > The pilot proved the mechanism with a second form — the provider supplied as
  > **filesystem paths** on the installer host. That form is implemented and remains in
  > the tree as developer and CI scaffolding, but it is **not proposed as a
  > partner-facing form**. It is recorded in full, with the pilot's evidence, under
  > [Alternatives](#alternatives-not-implemented).

  ```yaml
  platform:
    external:
      platformName: foo
      cloudControllerManager: External
      clusterAPI:
        name: foo

        # Where the provider comes from. A discriminated union: exactly one of
        # the members below is set, and sourceType names which.
        controller:
          sourceType: Image
          image:
            ref: quay.io/foo/openshift-capfoo@sha256:CHANGE-ME

        # Extra flags appended to the controller's command line, whichever
        # source it came from.
        args: []

        # Values for the provider's own ${VAR} tokens in the payload's template
        # trees. This is the cluster creator's customisation interface, and the
        # only part of the stanza a conforming payload requires them to write.
        # INSTALLER_-prefixed names are reserved to the installer and rejected
        # here. Variable names are recorded in the install log.
        variables:
          FOO_REGION: us-east-1

        # Everything below is optional and shown only to say what it means. A
        # conforming payload needs none of it.
        #
        # Paths are relative, never absolute: capiInstallerPath is relative to
        # the install directory and must stay inside it, and the two below are
        # relative to the extraction root, so an override selects between
        # things the payload shipped rather than reaching outside it.
        #
        # capiInstallerPath: capi-installer       # default
        # customResource: infrastructure-components.yaml
        # manifestPath: manifests
        #
        # There is no hooks stanza. Hooks are declared by the payload, in
        # openshift.yaml -- see "Hooks are the integrator's interface" below.
  ```

  **How the payload is obtained and used.** In order:

  2. The installer pulls it over the registry API, authenticating with `pullSecret` and
     redirecting through `imageDigestSources`. **A disconnected or air-gapped install
     mirrors the payload to a reachable registry**; there is no "point at an image in the
     local container store" mode, because the installer speaks to a registry and does not
     read podman's or docker's image storage.
  3. The installer extracts the image content — controller binary, infrastructure
     components, both template trees, hooks — into `capiInstallerPath`, which defaults to
     `<install-dir>/capi-installer`. Every other path in the stanza is defaulted beneath
     it, so a conforming payload needs no path configuration at all.
  4. The installer renders both template trees, substituting its own
     `INSTALLER_`-prefixed variables — `INSTALLER_INFRA_ID`, `INSTALLER_BASE_DOMAIN` and
     the rest of the published set — plus the user's `variables`, using clusterctl's
     `${VAR}` syntax. A variable the installer cannot resolve fails before any cloud call.
     The Cluster API tree lands where a hand-written one would, and the day-0 manifest
     tree lands in the extra-manifest directory, so a payload install and a hand-written
     one are the same install from that point on — which is what makes the two CI arms
     comparable.

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
  # Compared against the Kubernetes version the installer was built against; the
  # install is refused outside the range. The Cluster API contract comes from the
  # adjacent clusterctl metadata.yaml and is not repeated here -- and is not
  # checked today; see Version Skew Strategy.
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
    # Two template trees, not one. clusterTemplates is applied to the
    # installer's local, ephemeral control plane and reaches nothing;
    # manifestTemplates is folded into the install directory's manifests and
    # does reach the cluster. Same rendering, different destination and
    # different lifetime.
    clusterTemplates: cluster-api/
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

  **Reference implementations.** Both pilot providers are expressed as payloads below.
  The AWS/CAPA one is the convenient arm; **the OCI/CAPOCI one is the arm that evidences
  the central claim**, because CAPOCI is not embedded in the installer. Read them
  together: the install-config side is nearly identical and nearly empty, and everything
  provider-specific has moved inside the payload, which is the result the form is chosen
  for.

  **AWS, with Cluster API Provider AWS (CAPA).**

  ```yaml
  platform:
    external:
      platformName: aws
      cloudControllerManager: External
      clusterAPI:
        name: aws
        controller:
          sourceType: Image
          image:
            ref: quay.io/CHANGE-ME/openshift-capa@sha256:CHANGE-ME
        # the feature gate CAPA requires before it will accept
        # AWSMachine.spec.ignition at all -- a provider-specific flag the
        # installer knows nothing about
        args:
        - --feature-gates=BootstrapFormatIgnition=true,ExternalResourceGC=true
        variables:
          AWS_REGION: CHANGE-ME
          AWS_DNS_ZONE: CHANGE-ME
  ```

  **OCI, with Cluster API Provider OCI (CAPOCI).** Everything else defaults from the
  payload, including all three hooks.

  ```yaml
  platform:
    external:
      platformName: oci
      cloudControllerManager: External
      clusterAPI:
        name: oci
        controller:
          sourceType: Image
          image:
            ref: quay.io/CHANGE-ME/openshift-capoci@sha256:CHANGE-ME
        args:
        - --init-oci-clients-on-startup=false
        variables:
          OCI_COMPARTMENT_ID: CHANGE-ME
          OCI_REGION: CHANGE-ME
  ```

  ```yaml
  # inside the CAPOCI payload: <capiInstallerPath>/openshift.yaml
  apiVersion: external.openshift.io/v1alpha1
  kind: ProviderPayload
  compatibility:
    kubernetes:
      min: "1.32"
      max: "1.34"
  layout:
    # NOT the CAPOCI binary: a wrapper translating the installer's
    # --health-addr into the provider's --health-probe-bind-address, the one
    # installer-supplied flag CAPOCI does not declare. It must translate
    # rather than drop the flag -- the installer polls /healthz at exactly
    # the address it passed. A layout.controller that is a wrapper is legal
    # on purpose, and this is why.
    controller: bin/capoci-shim.sh
    customResource: infrastructure-components.yaml
    clusterTemplates: cluster-api/
    manifestTemplates: manifests/
    hooks:
      # creates api/api-int DNS; adds a 22623 listener and backend set to the
      # API load balancer, which CAPOCI has no field for; and writes the CCM
      # and CSI config Secrets, whose inputs are OCIDs that do not exist until
      # Cluster API has built the network -- so they cannot be day-0 manifests
      infraReady: hooks/infraReady
      postProvision: hooks/postProvision
      preDestroy: hooks/preDestroy
  ```

  **What moving CAPOCI into a payload changes, and what it does not.** The mechanism is
  unchanged; what changes is where the provider-specific knowledge lives.

  * The `--health-addr` shim stops being something the cluster creator has to know about
    and write a path to. It becomes `layout.controller` inside the artifact the partner
    publishes, which is the party that knows the flag is undeclared.
  * The three hooks stop being separate files the creator places and wires per phase. The
    payload declares them and the installer defaults each phase from the layout.
  * `postProvision` loses its `--input-service=openshift-ingress/router-external-default`
    argument: a payload hook has nowhere to carry an argument list, so a partner folding a
    hook into a payload folds its flags into the program's own defaults.
  * The CCM and CSI configuration that the pilot's `infraReady` wrote as Secrets stays in
    the hook. Their inputs are OCIDs that do not exist until Cluster API has built the
    network, so they cannot become `manifestTemplates` — the payload moves delivery, not
    the ordering constraint.
  * Nothing in the installer becomes OCI-aware. The AWS and OCI stanzas above differ only
    in the image reference, the provider's own flags and the provider's own variables.

  **Decided on the payload image:**

  * **`controller` is a discriminated union, spelled the way the API conventions spell
    one.** `sourceType` is a required discriminator with PascalCase values `Image` and
    `Local`, and each names a pointer-to-struct member carrying only that mode's fields.
    The proposed form and the pilot's local form are therefore two members of one union,
    not two overlapping field sets — which is what makes it cheap to keep `Local` as
    developer scaffolding without it becoming a second partner-facing contract. See
    *API Extensions* for the shape and for what this replaces.
  * **`customResource` is a single file.** `customResourcePath` is dropped. Cluster API
    providers publish exactly one `infrastructure-components.yaml`; a directory contract
    would be a generalisation with no known consumer.
  * **An unresolvable template token fails the install.** It fails when the payload's
    templates are rendered, during `create manifests` — before any cloud call and before
    the controller starts, matching clusterctl's behaviour on an unset required variable.
    It is **not** an install-config validation error: the token lives in the payload, not
    in `install-config.yaml`, so there is no field path to attach it to and it cannot be
    reported as a `field.ErrorList` entry alongside the rest. Partners and cluster
    creators should expect it from `create manifests`, not from `create install-config`.
  * **Credentials for a private payload are the user's problem, not a new mechanism.**
    The user appends the partner's registry credentials to the existing `pullSecret`, or
    mirrors the payload into a registry they already have credentials for and redirects it
    with `imageDigestSources`. No new install-config field. Note that `pullSecret` is the
    *only* credential source: the installer writes it to a temporary `--registry-config`
    file rather than letting `oc` fall back to the host's ambient docker or containers
    auth, so the same install-config pulls the same image on another machine.
  * **A tag is accepted; the reference must be fully qualified.** Starting with tags keeps the first implementation simple. The install log records the resolved digest and the artifact SHA-256, so a tagged install is still traceable after the fact even though it is not reproducible from the install-config alone.
  * **Hooks are the integrator's interface, and the payload is their only home.** A hook
    is provider-specific work — DNS, an extra load-balancer listener, ignition staging —
    that the partner who wrote the provider is the only party able to write correctly. It
    is not a knob a cluster creator turns. So `install-config.yaml` has no `hooks` stanza
    at all: the payload declares each phase in `openshift.yaml` and the installer runs
    what it declares, resolved against the extraction root and required to stay inside it.
    `variables` is the cluster creator's interface and `openshift.yaml` is the
    integrator's, and the two do not overlap.

    An earlier design let a cluster creator override a phase with a host-local program.
    That is dropped. It bought a developer convenience and cost a second resolution root,
    a per-phase origin rule, and a mixed trust boundary in which the payload was the unit
    of trust for some hooks and not others — and the first implementation of it was
    already wrong, relocating all three hooks when one was overridden. Developers
    iterating on a hook use `sourceType: Local`, which is what that mode is for.
  * **A payload hook must default its own arguments.** The layout names a program and
    nothing else, so there is nowhere in the payload for a hook to carry an argument list.
    A partner moving a hook from the local-path form to a payload has to fold its flags
    into the program's own defaults.

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

  **Disconnected and mirrored installs reuse `imageDigestSources`.** This is not a new problem. `install-config.yaml` already carries `imageDigestSources` (`pkg/types/installconfig.go:157`), which is how a mirrored install redirects the release image before any cluster exists; the installer converts it into the in-cluster `ImageDigestMirrorSet` afterwards (`pkg/asset/manifests/imagedigestmirrorset.go:45`).
  The provider payload resolves through the same field. `pullSecret` covers authentication and `imageDigestSources` covers redirection; neither needs a new mechanism.

2. With a payload, place nothing: the installer renders the payload's two template trees
   into `<install-dir>/external-install/` and `<install-dir>/external-install/extra-manifests/`
   itself. In the local-path mode the cluster creator writes those same two directories by
   hand. Note the directory is `external-install/` and deliberately **not** the installer's
   own `cluster-api/` (`pkg/types/external/doc.go:23`): that one is an asset load path
   whose contents the asset store discards when an upstream dependency is dirty, and it is
   also where the installer unpacks the Cluster API and envtest binaries, so it is
   installer scratch space rather than a user interface.
   Note also that `compute[].replicas` is `0` on this platform regardless of form: the installer generates no MachineSets for `platform: external` (`pkg/asset/machines/worker.go:807`), so day-0 workers are Cluster API `Machine` objects in `<install-dir>/external-install/machines/`, not a compute pool.
3. Run `openshift-install create cluster`.
4. The installer resolves and validates the provider artifacts, starts the local Cluster API control plane, installs the provider's components, runs the provider's controller, and applies the user's Cluster API manifests.
5. On `Cluster.status.infrastructureReady`, the `infraReady` hook runs — typically to create DNS records or publish the bootstrap ignition somewhere the platform can reach.
6. Machines are provisioned; bootstrap completes; the Cluster API control plane is torn down.
7. The `postProvision` hook runs against the new cluster's kubeconfig, applying the partner's cloud controller manager, CSI driver and related configuration.
8. Install completes.

Teardown:

1. `openshift-install destroy cluster` reads the provider from cluster metadata.
2. The `preDestroy` hook runs.
3. The provider is restarted locally and the infrastructure custom resources are deleted so the provider's finalizers can run. The restarted controller reaches the cloud the same way it did during the install — through the environment of the `openshift-install` process — so `destroy cluster` must be run with the same cloud credentials in the environment as `create cluster` was.

#### Variation: platforms that cannot carry the bootstrap ignition in instance metadata

`bootstrap.ign` is typically hundreds of kilobytes and many clouds cap instance user-data well below that. Such platforms run the install in three phases rather than two: `create manifests`, `create ignition-configs`, then — in partner tooling, before `create cluster` — upload `bootstrap.ign` to object storage, mint a pre-signed URL, and replace the on-disk file with a small pointer-ignition document. The installer reloads the file during asset resolution, so this requires no installer change.

This works, but it means **every partner reimplements the same credential hazard**: the pre-signed URL grants unauthenticated read over every secret in the bootstrap ignition.

Whether the installer should offer a first-class mechanism is an open question below.

### API Extensions

One new optional field on an existing platform stanza, `platform.external.clusterAPI`, of
type `*ClusterAPIProvider` (`pkg/types/external/platform.go:39`). `platform: external`
already exists and is unchanged otherwise. The proposed shape is below; the loose
filesystem paths the pilot used are the union's other member, and are recorded under
*Alternatives*.

```go
// pkg/types/external/platform.go
type ClusterAPIProvider struct {
    // Name is the provider's short name, recorded in cluster metadata so that
    // destroy cluster can tear down through the same provider.
    // +required
    Name string `json:"name"`

    // Controller says where the provider comes from.
    // +required
    Controller ProviderController `json:"controller"`

    // Args are extra flags appended to the controller's command line,
    // independent of which source it came from. A provider's own flags are
    // the partner's business and the installer does not interpret them; it
    // supplies --health-addr and a kubeconfig of its own, and appends these.
    // +optional
    Args []string `json:"args,omitempty"`

    // Variables are values for the ${VAR} tokens in the payload's template
    // trees, and are the cluster creator's customisation interface. Names
    // beginning with "INSTALLER_" are reserved to the installer and rejected
    // here. Names are recorded in the install log; values are not.
    // +optional
    Variables map[string]string `json:"variables,omitempty"`

    // CapiInstallerPath is the payload extraction root, relative to the
    // install directory and required to stay inside it. Defaults to
    // "capi-installer". The two paths below are relative to that root and
    // default to what the payload's own layout declares, so an override
    // selects between things the payload shipped and cannot reach outside it.
    // A conforming payload needs none of the three.
    // +optional
    CapiInstallerPath string `json:"capiInstallerPath,omitempty"`
    // +optional
    CustomResource string `json:"customResource,omitempty"`
    // +optional
    ManifestPath string `json:"manifestPath,omitempty"`
}

// ProviderController is a discriminated union over the ways a provider can be
// supplied.
// +union
type ProviderController struct {
    // SourceType is the discriminator. It is required and has no default:
    // nothing has shipped, so there is no install-config to stay compatible
    // with, and a defaulted discriminator would mean an empty controller
    // stanza silently selecting a mode.
    // +unionDiscriminator
    // +kubebuilder:validation:Enum=Image;Local
    // +required
    SourceType ProviderSourceType `json:"sourceType"`

    // Image is the proposed form: one OCI image carrying every artifact.
    // +optional
    Image *ImageSource `json:"image,omitempty"`

    // Local is the developer and CI form, not documented to partners. See
    // Alternatives, and Open Question 1 on whether it survives to GA.
    // +optional
    Local *LocalSource `json:"local,omitempty"`
}

type ImageSource struct {
    // Ref is a fully qualified image reference. A digest is required for a
    // mirrored or disconnected install and accepted everywhere; a tag is
    // accepted for a connected install.
    // +required
    Ref string `json:"ref"`
}

type LocalSource struct {
    // Path is a directory on the installer host laid out as a payload root:
    // an openshift.yaml and the artifacts it declares. It is the extracted
    // form of what an image carries, so the two members differ only in how
    // the bits arrive and share every rule about what they contain.
    // +required
    Path string `json:"path"`
}
```

**What this shape replaces, and why the change is worth making before anything ships.**
The pilot grew these fields one at a time and ended with three ways to say the same
thing — `binaryPath` and `controller.path`; `args` on the provider and `args` on the
controller — with `sourceType` an optional lowercase enum defaulted to `local`, the two
modes' fields interleaved in one flat struct, a `hooks` stanza duplicating what the
payload already declares, and at-most-one-of enforced by hand in validation. That is a
union in behaviour without being one in shape, and it is exactly
the arrangement [`dev-guide/api-conventions.md`][api-conventions] exists to prevent:
members must be pointers to structs, must be optional, the discriminator should be
required, and its values should be PascalCase matching a member's json tag. The version
above satisfies all four, deletes the duplicate spellings, and makes "which mode is this"
answerable from the document rather than from the validation code.

The `Local` member shrinks to a single path for the same reason. In the pilot it was two
unrelated paths — a binary and a components bundle — because the pilot had no descriptor
to hold a layout. Now that `openshift.yaml` exists, the honest description of the local
mode is "a payload root that is already extracted", and saying it that way means one
layout, one validation path and one set of rules across both members rather than two.
An image is then nothing more than a transport for a directory. This also answers where
hooks come from in local mode: from the same `openshift.yaml`, because there is only one.

`install-config.yaml` is not served by an API server and does not go through OpenShift API
review, so none of this is mandatory. It is still worth doing: install-config fields are
effectively permanent — the conventions in `pkg/types` forbid removing or renaming one —
so a shape chosen now is a shape supported indefinitely. The cost of fixing it is zero
today and unbounded after the first release.

[api-conventions]: https://github.com/openshift/enhancements/blob/master/dev-guide/api-conventions.md#unions

**There is no `hooks` field.** The pilot had one, and it is removed rather than carried
forward. A hook is a provider-specific program — it creates DNS records, adds a listener
to a load balancer, stages ignition — written by the party that wrote the provider. A
cluster creator cannot usefully author one and should not be offered the field. Hooks are
declared only by the payload, in `openshift.yaml`, which is the integrator's interface in
the same way `variables` is the cluster creator's. Dropping the field also removes a
second hook-resolution root, a per-phase origin rule, and a trust boundary that ran
through the middle of the hook set.

**This is install-config only, in both modes.** It is not a cluster API: it is not served
by the API server, it is not persisted into the cluster, and it has no CRD, webhook,
aggregated apiserver or finalizer. Validation is ordinary install-config validation
returning a `field.ErrorList` (`pkg/types/external/validation/platform.go`). Moving to a
payload image does not change that — the image is pulled and extracted by the installer
process and nothing about the payload reaches the cluster's API.

The agent-based installer shares the install-config format but routes data through assisted-service's cluster manifests, and this field is not passed through. Per the install-config conventions, agent-based validation **warns** when the field is set to a non-default value rather than ignoring it silently.

`ClusterPlatformMetadata` gains an `External` member (`pkg/types/clustermetadata.go:46`) so `destroy cluster` can find the provider. It is written conditionally (`pkg/types/clustermetadata.go:89`); `platform: none` continues to write nothing. In image mode it also carries the resolved payload digest, which is what lets `destroy cluster` run without contacting a registry.

The payload declares its own layout in `openshift.yaml` at its root
(`pkg/infrastructure/external/payload/descriptor.go:37`), carrying the Kubernetes
compatibility range and a path for each artifact. **The layout has two template keys, not
one**, and the distinction is the part worth reviewing: `clusterTemplates` (`:81`) renders
into `<install-dir>/external-install/` and is applied to the installer's local, ephemeral
control plane, while `manifestTemplates` (`:86`) renders into
`<install-dir>/external-install/extra-manifests/` and reaches the installed cluster. They
share nothing but the rendering step, so collapsing them into one directory would make a
payload unable to say which of its objects outlive the install
(`pkg/infrastructure/external/payload/resolve.go:382`).

Five things the implementation settled that the sketch did not say:

* **Only CRDs and webhook configurations are installed from the components bundle.** The
  installer runs the controller as a local OS process, so the bundle's `Namespace`, RBAC,
  `Deployment` and any `${AWS_B64ENCODED_CREDENTIALS}` Secret in it are inert. A payload is
  therefore a generic, multi-tenant artifact: **it carries no credential, and the
  credential boundary is unchanged by this mode.** The cluster's cloud credentials still
  arrive as a user-supplied manifest staged into the extra-manifest directory, never from
  the payload.
* **The installer's variable set is small and fixed:** `INSTALLER_INFRA_ID`,
  `INSTALLER_CLUSTER_NAME`, `INSTALLER_BASE_DOMAIN`, `INSTALLER_CLUSTER_DOMAIN` and
  `INSTALLER_PUBLISH` (`resolve.go:411`). It is only what the installer knows without
  knowing the provider's cloud. Adding a variable is compatible; renaming one is not. Only
  the names are logged, never the values.
* **Hooks are never rendered.** The layout names a program and nothing else, so a payload
  hook has nowhere to carry arguments and must default its own. This is a real constraint
  on partners, discovered by writing one.
* **Every hook resolves against one root, the extraction directory, and must stay inside
  it.** The pilot allowed a second root, for hooks the install-config named, and the
  per-phase origin rule that came with it was implemented wrongly first — overriding
  `postProvision` relocated `infraReady` too and broke an install on a hook nothing was
  wrong with. Removing the install-config field removes the rule and the second root.
* **A phase the payload does not declare is skipped**, with a warning naming the
  consequence. Only `infraReady` has one worth stating: nothing publishes `api-int`, and
  the install dies at the bootkube `resolve-api-int-url` stage about forty minutes later,
  with an error that points nowhere near the cause.

One pull per install, pinned: the resolution is cached per process and recorded in
`.openshift_install_payload.json` in the install directory, so `create manifests` and
`create cluster` are two processes and one payload, a floating tag cannot resolve twice to
two different images inside one install, and `destroy cluster` needs no registry at all
(`resolve.go:135`). The resolved digest and the extracted tree are logged by path and
checksum, and the digest is recorded in `metadata.json` whether the process that wrote it
pulled the payload or reused an earlier command's extraction — which is the usual case,
since the command that writes the surviving metadata is the second one.

This remains install-config only — it adds no cluster-serving API, and the API approval
named in the frontmatter is for the feature gate, not for these fields. It does enlarge
the installer's responsibilities materially: pulling and extracting an image, rendering two
template trees, and validating a declared payload layout are three things the pilot did
not do.

**Pulling is delegated to `oc image extract`, not to a vendored registry client.** The
first implementation used `go-containerregistry` and cost 22,861 vendored lines across
seven modules — `docker/cli`, `docker/distribution`, `docker/docker-credential-helpers`,
`containerd/stargz-snapshotter/estargz`, `vbatts/tar-split` and two others — to support
roughly 1,500 lines of payload handling. The installer already extracts files from images
without any of that, and not only on the agent path: `pkg/asset/rhcos/releaseextract.go`
and `pkg/infrastructure/baremetal/bootstrap.go:175` both shell out to `oc`, the latter
during `create cluster`, which is this code's own phase. Doing it a second, different way
would have put two registry clients in one binary disagreeing about mirrors, proxies and
authentication. **The payload feature therefore adds no vendored dependency at all**:
`go.mod`, `go.sum` and `vendor/` are unchanged by it.

Two costs are taken in exchange, and both are real:

* **A runtime dependency on `oc`**, checked with `exec.LookPath` before anything is
  created and only when a payload image is configured, so no other platform and not this
  platform's local-path mode acquires it. The error names the alternative
  (`binaryPath`/`componentsPath`) rather than failing with `exec: "oc": not found`.
* **The executable bit is not preserved.** `oc image extract` normalises every regular
  file to `0640`; `--preserve-ownership` would keep the mode but calls `lchown` and so
  fails for any non-root user, which the installer is. The installer therefore restores
  the bit from the payload's own `openshift.yaml` declaration, for the declared controller
  and hooks and nothing else. This is an improvement rather than a workaround: an
  undeclared program shipped in a payload stays unexecutable on disk, so the descriptor is
  no longer merely a description of what can run — it is what makes it able to run. The
  previous design could only *detect* an undeclared executable after extraction.

One behaviour is weaker than the vendored client and is called out rather than absorbed:
the size bound on the extracted tree is checked after extraction rather than during it,
because `oc` writes the tree itself and offers no size limit. It is a sanity check on what
arrived, not a guard on what is written — which is still more than the installer's other
`oc image extract` call sites do, none of which bound anything.

#### No cluster API change is proposed, and why

**This enhancement proposes no change to the [Infrastructure][infra-api] config API, or to
any other cluster-serving API.** The api-approver named in the frontmatter is there for the
feature gate this ships behind — an `openshift/api` change — and for nothing else; no
`Infrastructure` field is proposed. The reason is the ordinary API-review test: a status field exists so that a
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

### What the pilot proved, and where the proposal departs from it

This enhancement is not a design on paper. It is the written-up result of a pilot that
installed and destroyed OpenShift clusters on two clouds, and the proposal differs from
what the pilot built in three specific places. Both halves matter to a reviewer: the
evidence is what makes the mechanism credible, and the departures are what is actually
being asked for.

**What the pilot built and ran.** A working `platform: external` install driven by a
Cluster API provider the installer does not embed. The provider arrived as loose files on
the installer host — a controller binary and a components bundle named by path in
`install-config.yaml` — and the installer started it against its own local, ephemeral
Cluster API control plane. Two providers were exercised: AWS with CAPA as the convenient
reference arm, and OCI with CAPOCI as the arm that evidences the central claim, because
CAPOCI cannot be embedded. Clusters were created and destroyed on both.

**What that established, and which the proposal inherits unchanged.** None of these are
mode-specific; the payload form changes how the provider is delivered, not any of them.

* The installer can run a provider it does not ship, against its own control plane, and
  reach `Install complete!` on a platform with no in-tree support.
* The installer holds no cloud credentials on `platform: external` and must not acquire
  any. This is why hooks exist at all.
* The three hook phases are the right three, and `infraReady` is not optional: without
  something publishing `api-int`, bootkube fails and the install dies far from the cause.
* Hooks receive paths, not documents, because passing a kubeconfig or an ignition body
  through the environment puts secrets into process listings and traces.
* A wrapper script in place of the provider binary is legal and necessary — it is how
  CAPOCI's one undeclared flag was reconciled without a per-platform code path.
* Teardown must be verified on a *second* cluster. Three teardown defects were found only
  by creating a cluster after a previous one had been destroyed.
* Three shared-code touches — manifest handling, artifact resolution, the platform switch
  — are the full blast radius on integrated platforms.

**Where the proposal departs.** The pilot answered "can the installer run a provider it
does not embed?" and deliberately left every *delivery* question open. Those are what the
proposal closes:

1. **Delivery becomes an OCI payload image**, not host-local paths. Paths have no
   publication story, no immutable identity, no pull authentication, no mirroring and no
   way to version the binary, components, templates and hooks as one thing. An image has
   all of those already, through `pullSecret` and `imageDigestSources`. The pilot's form
   survives as developer and CI scaffolding only — see *Alternatives* — and Open Question 1
   is whether it survives at all.
2. **The payload describes itself.** `openshift.yaml` declares the layout, so the cluster
   creator wires nothing: every path defaults beneath the extraction root, hooks default
   per phase from the layout, and validation is exhaustive — including rejecting an
   executable the payload did not declare. In the pilot the creator named each path by
   hand and an undeclared program was simply a program.
3. **Day-0 manifests and Cluster API templates travel with the provider**, rendered by the
   installer with clusterctl substitution, as two separate trees because they have
   different destinations and different lifetimes. The pilot had the creator supply both
   by hand, which made it possible to ship a provider that could not actually bring up a
   cluster without out-of-band instructions.

Everything else — the hook contract, the `OPENSHIFT_INSTALL_*` environment, the
single-object extra-manifest rule, the architecture check, logging artifacts by path and
checksum rather than content — carries over as it was proved.

### Implementation Details/Notes/Constraints

Proposed file locations in `openshift/installer`:

| Area | Location |
| --- | --- |
| Install-config API and validation | `pkg/types/external/platform.go`, `pkg/types/external/validation/platform.go` |
| Artifact override and validation | `pkg/clusterapi/artifacts.go` |
| Running a user-supplied controller | `pkg/clusterapi/external.go` |
| External provisioning path | `pkg/infrastructure/external/clusterapi/` |
| Lifecycle hooks | `pkg/infrastructure/external/hooks/` |
| Teardown | `pkg/destroy/external/` |
| Platform routing | `pkg/infrastructure/platform/platform.go:59` |

The payload-image mode adds one package, `pkg/infrastructure/external/payload/`: descriptor
and layout validation, the registry pull and extraction, template rendering, and the single
resolver every other stage calls. Nothing outside that package reads a descriptor,
resolves a layout path or renders a template, so the payload contract has exactly one
implementation. It is entered from `pkg/asset/manifests/openshift.go:324`, during
`create manifests`, immediately before the extra manifests are folded in.

Constraints discovered by running it, not by design:

* **The installer has no cloud credentials on `platform: external`, and must not acquire any.** This is why hooks exist. Every piece of cloud-specific work — DNS records, ignition staging, load balancer attachment — is the partner's, executed through a hook, using the partner's own credentials.
* **Hook environment carries paths, not documents.** Passing a kubeconfig or an ignition body through an environment variable puts secrets into process listings and traces.
* **`infraReady` cannot write into `<install-dir>/openshift/`.** By the time infrastructure is ready the installer has already consumed and removed that tree, so partner configuration Secrets must be applied from `postProvision` against the hook's kubeconfig. This is a measured finding, not a preference.
* **A wrapper script is a legal `binaryPath`.** Cluster API providers do not agree on flag names — the installer passes `--health-addr` (`pkg/clusterapi/external.go:155`) and at least one real provider declares `--health-probe-bind-address` instead, with pflag exiting non-zero on an unknown flag. A shim that translates the flag is the supported answer today. Note that a shim must **translate**, not drop: the installer polls `/healthz` at exactly the address it passed.
* **`destroy cluster` deliberately excludes Secrets from the saved Cluster API output**, so nothing credential-bearing is written to the install directory for teardown to replay. The restarted provider therefore has to find its credentials the same way it did at install time — in the process environment — which is what makes teardown work without the installer ever holding a cloud credential, and what makes "run destroy from the same shell you ran create from" a requirement rather than a convention.
* **The provider controller and the hook programs inherit the installer's process environment**, plus a small set of `OPENSHIFT_INSTALL_*` variables the installer adds, one of which is the path to the cluster-admin kubeconfig. This is the actual credential contract and it is worth stating plainly: the payload carries no credential, and it is handed every credential the `openshift-install` process can see. A narrower contract — an explicit allowlist of variables to forward — is possible and is not proposed here, because every pilot provider and every hook needed the cloud CLI configuration that only the ambient environment supplies.
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

**The `clusterAPI` stanza is persisted into the cluster, and nothing redacts it.** The
installer writes a copy of the install-config to `kube-system/cluster-config-v1`, with
`pullSecret` removed and a per-platform redaction applied — and that redaction currently
has cases for vSphere and Nutanix only (`pkg/asset/manifests/operators.go:352`). Every
field proposed here therefore lands in a ConfigMap readable by anything with cluster-wide
read. Image references and paths are not sensitive. `variables` is: it is free-form
`map[string]string` filled in by the cluster creator, and a partner can define any
variable they like, including one whose value is a credential.
*Mitigation:* `platform: external` needs its own case in that redaction, and the EP
proposes adding one that drops `variables` entirely rather than attempting to classify
individual keys. The documented rule — variable *names* are logged, values are not —
should hold for the persisted copy too, and today it does not. This is the one finding
here that is a live defect rather than a design choice.

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
* The proposal keeps a second, non-proposed supply mode alive in the code. The payload
  image is the partner-facing form; the pilot's **filesystem paths** form remains valid in
  the API as developer and CI scaffolding, recorded under Alternatives. That is still two
  code paths to keep honest and two arms to run in CI, and the honest framing is that the
  cost is accepted for the convenience of provider developers, not because the shape is
  wanted. Whether `local` is removed outright before GA is Open Question 1.
* The payload-image mode requires `oc` on the installer host. The local-path mode and
  every other platform are unaffected, and the check runs before anything is created, but
  it is still a new external-binary dependency on an install path — acceptable only
  because the installer already shells out to `oc` during `create cluster` on the
  baremetal path and throughout the agent and RHCOS artifact paths.
* It creates an expectation that Red Hat will help partners debug their providers. The
  intent is the opposite — partners must be able to diagnose their own provider — but the
  mechanism for that is not defined here, and nor is the venue. **Which forum do partners
  outside the VCSP programme use to reach OpenShift engineering about an external
  provider?** Today there is no answer that does not route through a programme, and the
  delivery model work must either name an existing community channel or accept that the
  only path is VCSP.

## Alternatives (Not Implemented)

### Supply the provider as local filesystem paths — the pilot's form, not proposed

This entry is the exception to the section's title, and the exception is deliberate.
**This alternative *is* implemented**: it is how the mechanism was first proved, on both
pilot providers, and it stays in the tree as developer and CI scaffolding. It is recorded
here rather than in the Proposal because it is **not proposed as a partner-facing form**.
The detail below is kept in full so the pilot's evidence survives review rather than being
re-derived.

**What it is.** The provider's artifacts already present on the installer host, with no
image pulled. Everything else in the flow — validation, the local Cluster API control
plane, the hooks, metadata, teardown — is identical.

**The fragments below are the pilot's, verbatim, not the proposed shape.** They are kept
as run so the evidence is reviewable. Three differences from what *API Extensions*
proposes are worth naming before reading them: the pilot's fields were flat rather than
inside a `local` union member; the pilot named a binary and a components bundle
separately, where the proposal names one already-extracted payload root; and the pilot's
`hooks` stanza lived in `install-config.yaml`, where the proposal has the payload declare
them in both modes. The pilot had no descriptor, which is why it needed all three.

**Why the payload image is preferred.** Every objection below is about *delivery*, not
about whether the mechanism works. That is precisely the point: local paths answered
"can the installer run a provider it does not embed?", and left every delivery question
open. An OCI image answers all of them with machinery the installer and the cluster
already have.

| Requirement | Local paths | Payload image |
| --- | --- | --- |
| Partner can publish through their existing release process | no — the files must arrive on the host by some other means | yes, it is an ordinary image build |
| Pinned to an immutable identity | no | yes, by digest |
| Disconnected / mirrored install | no story; paths are host-local | `imageDigestSources`, already in install-config (`pkg/types/installconfig.go:157`) |
| Pull authentication | none | `pullSecret`, already in install-config |
| Signing and provenance | nothing to sign | the partner's own image signing |
| Versioned as one unit | no — binary, components, templates and hooks version independently | yes, one tag or digest is one contract |
| Self-describing layout | no — the user wires each path by hand | `openshift.yaml` declares it; validation is exhaustive |
| Hooks and day-0 manifests travel with the provider | no, supplied separately | yes, defaulted from the payload layout |

**Pilot configuration, retained for review.** Both fragments are from real pilot runs,
with environment-specific values replaced by `CHANGE-ME`. The whole partner contract is
the `clusterAPI` stanza; everything above and below it is ordinary `platform: external`.

**AWS, with Cluster API Provider AWS (CAPA).** The reference arm. Note that `args` is what
carries provider-specific flags the installer knows nothing about — here the feature gate
CAPA requires before it will accept `AWSMachine.spec.ignition` at all.

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

**OCI, with Cluster API Provider OCI (CAPOCI).** The arm that evidences the central claim,
because CAPOCI is not embedded in the installer. Two differences are worth reading:
`binaryPath` is a shell wrapper, which is legal on purpose
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

**What the pilot established with this form, and which the payload inherits.** The
wrapper-script case above is the clearest example: a `binaryPath` that is a shell script
rather than the provider binary is legal, and it is how CAPOCI's one undeclared flag was
reconciled without a per-platform code path. The payload's `layout.controller` carries the
same property. The hook contract, the `OPENSHIFT_INSTALL_*` environment, the single-object
extra-manifest rule, the teardown-on-a-second-cluster requirement and the three shared-code
touches were all discovered and fixed through this form, and none of them are
mode-specific.

**Status.** `local` remains valid in the API and exercised in CI, because a developer
iterating on a provider should not have to build and push an image per change. Whether it
is removed from the tree entirely before GA is Open Question 1. It is not documented to
partners, and no partner-facing material should present it as a supported way to ship a
provider.

### Other alternatives considered

**Integrate each partner platform in-tree.** This is the status quo for supported platforms and it does not scale: every platform costs Red Hat ongoing maintenance, and it is exactly what `platform: external` was created to avoid.

**Embed partner providers in the installer image.** Rejected: it puts partner code in the release payload, couples the partner's release cadence to OpenShift's, and contradicts the `platform: external` contract.

**Fetch the provider from a git URL rather than an image.** Rejected. The image half of this — a provider-published OCI payload — is implemented and described above; a git reference was not, because it has no digest-pinning, mirroring or pull-authentication story that `pullSecret` and `imageDigestSources` do not already give an image for free.

**Pull the payload with an in-process registry client.** Implemented first, with `go-containerregistry`, and then removed. It worked and it resolved the same digest the current code does, but it vendored 22,861 lines across seven modules — including `docker/cli` and `containerd/stargz-snapshotter/estargz` — into the installer for one platform's optional mode, and it put a second registry client in a binary that already has one. Shelling out to `oc image extract` costs a runtime dependency on `oc` and the loss of the executable bit, both described above, and adds nothing to the vendor tree. A hand-rolled minimal registry client was also considered and rejected: it trades the SBOM cost for maintaining digest, manifest-list, mirror and authentication handling that `oc` already implements and that the cluster will use anyway.

**Let partners fork the installer.** Already possible, and the thing OCPSTRAT-3724 explicitly wants to stop.

**Keep using Assisted Installer or UPI for external platforms.** Works today, and remains available. It does not give an IPI-like experience and leaves the cluster with no machine-management layer wired in from day 0.

**Add per-platform exceptions for the known partner.** Rejected on principle: platform External exists to fit any provider, and an exception for one invites an exception for every other.

## Open Questions [optional]

1. **Is `controller.sourceType: Local` removed from the tree before GA, or kept as developer and CI scaffolding?** The partner-facing question is **decided**: the payload image is the proposed form, and the local-path form is not documented to partners — see Alternatives. The payload layout, templating syntax, variable vocabulary, compatibility file and validation strictness are all settled too. What remains open is narrower and internal: keeping `Local` costs a second code path and a second CI arm in perpetuity, and buys provider developers an iteration loop that does not require building and pushing an image per change. If it stays, it needs a stated support status so it is not mistaken for a partner contract.
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

14. **Where does the `openshift.yaml` contract live, who owns its schema, and how does a
    partner validate a payload before an install?** The descriptor is authored by a third
    party and carries `apiVersion: external.openshift.io/v1alpha1` and
    `kind: ProviderPayload`, so it looks like a Kubernetes API. Today it is none of the
    things that implies: the only definition is a Go struct in the installer, parsed with
    strict unmarshalling that rejects any other `apiVersion` outright, and there is no
    published schema, no conversion, no owner of the `external.openshift.io` group and no
    API review. A partner's first indication that their descriptor is wrong is an install
    that fails. That is tolerable for Dev Preview and not for a contract partners build
    release pipelines around. Four sub-questions, and they are not independent:
    * **Installer-only, or `openshift/api`?** Installer-only keeps the type next to its
      one consumer and skips API review, but it means the contract versions with the
      installer and has no machine-readable form a partner can check against.
      `openshift/api` gives a reviewed type, generated OpenAPI/CRD schema, deepcopy and
      conversion — the ordinary machinery — at the cost of a type in `openshift/api` that
      no API server ever serves. A third option is to keep it installer-local and **drop
      the GVK**, making it an unversioned installer config file that promises nothing.
      The current arrangement is the worst of the three: it promises API semantics it
      does not implement.
    * **How does `v1alpha1` become `v1beta1`?** Strict unmarshalling of a fixed
      `apiVersion` means a bump breaks every published payload *and* every older
      installer, in both directions, with no conversion path. Whatever the answer to the
      first sub-question, the skew rule has to be written down before partners publish.
    * **Can a partner validate without installing?** The natural answer is a
      `openshift-install` subcommand that takes an image reference, extracts it, and runs
      exactly the layout, executable and compatibility validation the install would —
      reporting pass or fail and nothing else. It needs no install-config, no pull secret
      and no cloud, which is what makes it usable in a partner's CI.
    * **Is `create manifests` that gate?** Partly, today: it already resolves and
      validates the payload (`pkg/asset/manifests/openshift.go:324`), so a descriptor
      error does surface before any cloud call. But it is a poor partner-readiness check —
      it needs a complete install-config including a pull secret and an SSH key, and it
      produces a manifest tree containing TLS and CA private keys and the kubeadmin
      password hash. Asking a partner to run it in CI to validate a descriptor hands them
      a credential-store side effect for a structural check. It should stay the install's
      gate, and not be the partner's.

    Related: the Kubernetes compatibility range is compared against the version the
    installer binary was built against rather than the release image's, which is the same
    ownership question seen from the other side — the installer is asserting a fact about
    the release from its own build rather than reading it.

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

**End-to-end.** Arms along two axes — which provider, and which supply mode:

* an AWS/CAPA arm with the provider supplied as a **payload image** — the proposed form,
  and therefore the arm that must stay green;
* an AWS/CAPA arm with the provider supplied as **local paths**, covering the developer
  and CI scaffolding mode. It is kept because the two modes are different code from the
  install-config down and a passing image arm says nothing about the path arm — not
  because the path form is partner-facing;
* an arm on a provider that is **not** embedded in the installer.

**The third arm is the only one that constitutes evidence for this enhancement's central claim**, and reviewers should hold it to that standard. The installer embeds CAPA, so an AWS run cannot distinguish "ran the binary it was handed" from "ran the binary it already had". A provider that cannot be embedded can. The AWS arms test the ergonomics of the mechanism; they do not test the mechanism. Building the payload's CAPA binary from provider source at a pinned ref narrows the gap but does not close it.

**Evidence to date for the payload-image mode.** One AWS/CAPA cluster, installed from a
payload image on a local registry, reached
`Install complete!` in 38m20s with no `level=error` or `level=fatal` in the install log.
OpenShift `5.1.0-ec.1`; every cluster operator `Available=True`, `Progressing=False`,
`Degraded=False`; three nodes `Ready`. The log records the tag resolving to a digest, one
pull in `create manifests` and a reuse rather than a second pull in `create cluster`, both
template trees rendered to their separate destinations, and both the `infraReady` and
`postProvision` hooks executing from the extracted payload rather than from the install
directory. This is a single create cycle on one provider: it is evidence that the mode
works, not that it is finished, and it does not discharge the teardown requirement below.

**Teardown is verified on a second cluster.** Three teardown defects in the pilot were found only by creating a cluster after a previous one had been destroyed. A single create/destroy cycle is not sufficient evidence.

**Conformance** runs on the resulting cluster, with the known CSR-approval gap handled explicitly by the job rather than silently.

### CI Job

The Prow jobs must be created using the CCM integration, user-supplied CCM on install time.

Presubmit jobs will be created in the following repositories:

- openshift/installer

Periodic jobs:

- AWS provider, provider supplied as a payload image — the proposed form
- AWS provider, provider supplied as local paths (already included in the pilot) — covers the developer and CI scaffolding mode
- OCI provider (already included in the pilot)

The AWS image arm installs from an image the pull request built, not from a published tag:
the payload is a ci-operator `images:` build declared as a job dependency
(`external-capi-payload-aws`), so no Red Hat-published partner artifact is implied by the
job existing. Its controller is built from CAPA source at a pinned ref rather than copied
out of `openshift-install`, which is what keeps the arm from quietly re-testing the
embedded provider.

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
the payload supports.

**Only one of the two is checked, and that is a gap this proposal should close.** The
installer compares `openshift.yaml`'s Kubernetes range against the version it was itself
built against, and refuses the install outside it, before any cloud call. It does not read
`metadata.yaml` at all, so the Cluster API contract a provider declares is unverified —
which is the skew most likely to break a partner, because it is the one that moves when
upstream Cluster API does and not when OpenShift does. Requiring `metadata.yaml` in the
payload and checking its `releaseSeries` contract against the Cluster API version the
installer runs is proposed here as Dev Preview work, not described as existing behaviour.

With both files checked, skew becomes a validated question for day 0 rather than a
documented one, and the standard payload structure is what makes a generic check possible
at all — the installer can verify any payload's readiness because every payload has the
same shape. None of it resolves the day-2 case, which remains with the delivery model work.

## Operational Aspects of API Extensions

This enhancement adds **no API extension in the operational sense**: no CRD, no aggregated
apiserver, no admission or conversion webhook, no finalizer, no new controller in the
cluster. The only API change is an optional field in `install-config.yaml`, which is
consumed by the installer binary and never served, persisted or reconciled.

This holds only because no change to the `Infrastructure` config API is proposed here. The
reasoning, the `cloudControllerManager.state` precedent it would follow, and the
immutability constraint that makes the CCM shape unsafe to copy are in
*API Extensions → No cluster API change is proposed, and why*. If a later revision adds an
`Infrastructure` status field, this section becomes wrong and the api-approver's remit
widens from the feature gate to the config API; the two must change together.

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
