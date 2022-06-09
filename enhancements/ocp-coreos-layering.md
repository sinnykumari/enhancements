---
title: OpenShift CoreOS Layering
authors:
  - "@yuqi-zhang"
  - "@jkyros"
  - "@mkenigs"
  - "@kikisdeliveryservice"
  - "@cheesesashimi"
  - "@cgwalters"
  - "@darkmuggle (emeritus)"
  - "@sinnykumari"
reviewers:
  - "@mrunalp"
approvers:
  - "@mrunalp"
creation-date: 2021-10-19
last-updated: 2022-06-09
tracking-link:
  - https://issues.redhat.com/browse/GRPA-4059
  - https://issues.redhat.com/browse/MCO-165
---

# OpenShift Layered CoreOS

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Operational readiness criteria is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

Change RHEL CoreOS as shipped in OpenShift to be a "base image" that can be used as in layered container builds and then booted.  This will allow custom 3rd party agents delivered via RPMs installed in a container build.  The MCO will roll out and monitor these custom builds the same way it does for the "pristine" CoreOS image today.

This is the OpenShift integration of [ostree native containers](https://fedoraproject.org/wiki/Changes/OstreeNativeContainer) or [CoreOS layering](https://github.com/coreos/enhancements/pull/7) via the MCO.

## Motivation

1. We want to support more customization, such as 3rd party security agents (often in RPM format).
1. It is very difficult today to roll out a hotfix (kernel or userspace).
1. It should be possible to ship configuration with the same mechanism as adding content.
1. There are upgrade problems today caused by configuration updates of the MCO applied separately from OS changes.
1. It is difficult to inspect and test in advance what configuration will be created by the MCO without having it render the config and upgrade a node.

### Goals

- [ ] Administrators can add custom code alongside configuration via an opinionated, tested declarative build system and via e.g. `Dockerfile` as an unvalidated generic mechanism
- [ ] The output is a container that can be pushed to a registry, inspected, and processed via security scanners. 
- [ ] Further, it can be executed as a regular container (via podman or as a pod) *and* booted inside or outside of a cluster
- [ ] Gain [transactional configuration+code changes](https://github.com/openshift/machine-config-operator/issues/1190) by having configuration+encode captured atomically in a container
- [ ] Avoid breaking existing workflow via MachineConfig (including extensions)
- [ ] Avoid overwriting existing custom modifications (such as files managed by other operators) during upgrades

### Non-Goals

- While the base CoreOS layer/ostree-container tools will be usable outside of OpenShift, this enhancement does not cover or propose any in-cluster functionality for exporting the forward-generated image outside of an OpenShift cluster. In other words, it is not intended to be booted (`$ rpm-ostree rebase <image>`) from outside of a cluster.
- This proposal does not cover generating updated "bootimages"; see https://github.com/openshift/enhancements/pull/201 and https://github.com/coreos/fedora-coreos-tracker/issues/1151
- Don't change existing workflow for RHEL worker nodes

## Proposal

1. The `machine-os-content` shipped as part of the release payload will change format to the new "native ostree-container" format, in which the OS content appears as any other OCI/Docker container.  
  (In contrast today, the existing `machine-os-content` has an ostree repository inside a UBI image, which is hard to inspect and cannot be used for derived builds).  For more information, see [ostree-rs-ext](https://github.com/ostreedev/ostree-rs-ext/) and [CoreOS layering](https://github.com/coreos/enhancements/pull/7). 
2. Documentation and tooling will be available for generating derived images from this base image
3. This tooling will be used by the MCO to create derived images


### User Stories

#### What works now continues to work

An OpenShift administrator at example.corp is happily using OpenShift 4 (with RHEL CoreOS) in several AWS clusters today, and has only a small custom MachineConfig object to tweak host level auditing.  They do not plan to use any complex derived builds, and expect that upgrading their existing cluster continues to work and respect their small audit configuration change with no workflow changes.

#### Adding a 3rd party security scanner/IDS

example.bank's security team requires a 3rd party security agent to be installed on bare metal machines in their datacenter.  The 3rd party agent comes as an RPM today, and requires its own custom configuration. While the 3rd party vendor has support for execution as a privileged daemonset on their roadmap, it is not going to appear soon.

The team uses the OCP docs and looks at examples such as https://github.com/coreos/coreos-layering-examples/ and writes a Dockerfile that looks similar to the Tailscale example that adds a 3rd party yum repository, and installs the 3rd party-security-agent.rpm. Note that the base image here would be published and consumable outside of the release image (as well as part of it).

They build this image in one of their “central” clusters and publish it into their air-gapped internal registry (registry.internal.example.bank) as `registry.internal.example.bank/examplecorp/rhel-coreos-base:4.12`.  (Note: The image could be built outside of a cluster too using any container build tools).

They change a configuration (CRD) which causes their staging 4.12 cluster to use this new image for worker nodes (see below for more information on the configuration). The MCO in the staging cluster rolls out this image, including draining nodes in the same way it handles MachineConfig and OS updates today.


When the rollout is successful to worker nodes on staging, they also roll it out to the control plane nodes.

They deploy the change across the rest of their (e.g. 5) production clusters.

A few weeks later, Red Hat releases a kernel security update as part of a cluster image, which appears as a new image at registry.redhat.io/rhel-coreos-8.6:4.12. This triggers a rebuild on their central cluster, which is successful and pushed to the mirror registry.

The cluster is configured to poll for changes to this image in the remote registry, and when that happens, the updated image will be automatically rolled out to all nodes - and note this will happen *without* an `oc adm upgrade`.  (Note: rollout will be configurable)



#### Kernel hotfix in cluster

example.corp runs OCP on aarch64 on bare metal.  An important regression is found that only affects the aarch64 architecture on some bare metal platforms.  While a fix is queued for a RHEL 8.x z-stream, there is also risk in fast tracking the fix to *all* OCP platforms.  Because this fix is important to example.corp, a hotfix is provided via a pre-release `kernel.rpm`.

The OCP admins at example.corp get a copy of this hotfix RPM into their internal data store, and configure a derived build where the kernel RPMs are applied. The MCO builds a derived image and rolls it out.

(Note: this flow would likely be explained as a customer portal document, etc.)

Later, a fixed kernel with a newer version is released in the main OCP channels.  The override continues to apply, and will currently require manual attention at some point later to remove the override when we can guarantee the non-hotfix kernel has the desired fix.

A future enhancement will help automate the above.  For example, we may support associating a Bugzilla number with the override.  Then, the system can more intelligently look at an upgrade and see whether a new kernel package that is chronologically/numerically higher actually fixes the BZ - and only drop the override once it does.

#### Externally built image

As we move towards having users manage many clusters (10, 100 or more), it will make sense to support building a node image centrally. This will allow submitting the image to a security scanner or review by a security team before deploying to clusters.

Acme Corp has 300 clusters distributed across their manufacturing centers. They want to centralize their build system in their main data center, and just distribute those images to single node edge machines. They provide a `custom-coreos-imagestream` object at installation time, and their node CoreOS image is deployed during the installation of each cluster without a build operation.

In the future, we may additionally support a signature mechanism (IMA/fs-verity) that allows these images can be signed centrally.  Each node verifies this image rather than generating a custom state.

(Note some unanswered questions below)

### External image build workflow

The administrator will see a number of documented example Dockerfiles, like https://github.com/coreos/coreos-layering-examples/

They can copy and extend these and build them using any container build system they like.
The administrator can also use non-Dockerfile build systems such as Tekton or buildah.

The result will be a container image that can be passed via an API to clusters, in a similar way as way user workloads (pods).

### API Extensions ([MCO-280](https://issues.redhat.com/browse/MCO-280))

Note we will continue to support MachineConfig.  The key goal is to allow the user to override the rhel-coreos base image.

#### Current default proposal: Configure via imagestream

This is currently prototyped in the https://github.com/openshift/machine-config-operator/tree/layering branch.  Here there’s an imagestream per MachineConfigPool, and the administrator can specify an image which overrides the default rhel-coreos base image.

##### User workflow in layering model
Using layering model, users will be able to build, test and use a custom image based on their requirements.
The user experience is described in the sub-sections below:

###### Create Customer Base Image
- First of all the user will create a Customer Base Image. This image can run on compatible OCP clusters such as HyperShift, Heterogeneous cluster, Single Node OpenShift.
- Create a Dockerfile that uses compatible rhcos image and add the custom content (e.g. rpm, files) on top of it:
  ```
  FROM registry.redhat.io/rhel-coreos:4.11
  RUN cd /etc/yum.repos.d/ && \
    curl -LO <user_yum_repo> && \
    rpm-ostree install <pkg> && \
    rpm-ostree cleanup -m && \
    systemctl enable <services> && \
    ostree container commit
  ADD <conf_file> /etc/<conf_file>
  ```
- Build the image using any image build tool (e.g. podman) and push it to the registry (e.g. quay)
- Generated Customer Base Image can be manually tested or integrated with a CI workflow to run desired internal tests. For example, this image can be run through a security scanner the same as any other image.
- Once all the tests on the image are green, it can be tagged for use in a OCP cluster, like:
  ```
  oc tag --source=docker quay.io/examplecorp/custom-rhel-coreos:4.11 machine-config-operator/coreos-external:latest
  ```
- Once the image has been tagged,  some verification will be performed in the cluster to ensure that rhcos version present in the Customer Base Image is compatible with the cluster and feedback will be provided accordingly.
    - **On compatibility check passed**: “quay.io/examplecorp/custom-rhel-coreos:4.11 successfully tagged”
    - **On compatibility check failed**: “failed to tag quay.io/examplecorp/custom-rhel-coreos:4.11: rhcos version 89.2423423 not compatible with OCP 4.11.5, expected: rhcos version 89.43920432”

###### Create Final Pool Images
When the Customer Base Image has been successfully tagged into cluster, a final image will be created for each pool.
- The Final Pool Image is cluster & pool specific
- It includes templates and other cluster pool specific content which gets layered on top of the Customer Base Image.
- By default the Customer Base Image will be applied to all pools unless the user specifies a target pool. There will also be an imagestream per pool that can be used for overrides as well.
- The user can watch the pool to keep track of build progress for the pool image to troubleshoot in case build fails.
- Final pool image generated from this build will be also available to customer to pull and test. This exact image will be used by MCO to update nodes in the desired pools.

**Note**: This text is proposing a new CRD and some custom flow.  Another alternative is having the final pool image be pushed to an imagestream, and simply using pool pause to control final rollout.

###### Rollout of Final Image to Node
- Once the built final pool image has been tested by the user, they can initiate a rollout to a specific pool:
   ```
   $ oc rollout custombuild mcp/worker
   This is a disruptive action and will replace the current rhcos image with a custom build. Are you sure?
  ```
- Rollout follows the existing MCO rollout paradigm - i.e. the images roll out to max unavailable nodes in a pool. This will cause reboots.
- After rollout has finished, there is an easy way to see which builds are present in the pools:
  ```
  $ oc get custombuilds -n openshift-machine-config-operator
  BUILD         POOL         BASEIMAGE     STATUS        LOCATION
  cp-layers    master        “quay.io…”    Pending          TBD
  w-layers     worker        “quay.io…”    Done         image-registry...
  ```

###### Reverting to Standard OS
- At any given time, the user should be able to revert to the stock rhcos image for that cluster which comes in via release image:
  ```
  $ oc delete custombuild mcp/worker
  This is a disruptive action and will replace custom build XXXX.XX with rhelcoreos 89.12. Are you sure?
  ```
- This action will remove all custom layers and cause nodes to use the image coming in through osimageurl with no modifications.
- Non-negotiable: roll-backs to stock must be tested and supported

###### Providing Customer Base Image during cluster upgrade
- OS can be updated with Customer Base Image during a cluster upgrade. For this, the user will need to create the Customer Base Image using the incoming rhcos version and supply to upgrade so that default osimageurl is not used:
  ```
  $ oc add upgrade —layeredsource -f
  ```
- Final image gets generated in-cluster once MCO upgrades.
- Preflight check is required here on incoming OS image vs user supplied image. Otherwise, the user will get to 84% and learn that their supplied image has a big problem.

#### Spike to investigate and flesh out: Special pod

In this proposal we support a “special” pod (specific label/annotation) which actually is the host OS.  We also implicitly support using e.g. daemonsets with node selectors to roll out the OS change across multiple pools.  The MachineConfigPool would actually just create a daemonset.  There’d be no need for a “node controller” in the MCO.

A key advantage of this approach is that we could make it work to also use ConfigMaps and particularly Secret objects alongside the pod.  Secret management is highly complex, and it’s already a best practice to *not* embed secrets in container images.

Note that this could work on top of a high level ImageStream proposal to support node specific overrides. 

### Implementation details

####  Where the image will live
Discuss in cluster build and external registry
Related spikes:

https://issues.redhat.com/browse/MCO-281
https://issues.redhat.com/browse/MCO-286


#### Upgrade flow

See [OSUpgrades](https://github.com/openshift/machine-config-operator/blob/master/docs/OSUpgrades.md) for the current flow before this proposal.

1. The "base image" will be part of the release image, the same as today.
1. The CVO will replace a ConfigMap in the MCO namespace with the OS payload reference, as it does today.
1. The MCO will update an `imagestream` object (e.g. `openshift-machine-config-operator/rhel-coreos`) when this ConfigMap changes.
1. The MCO builds a container image for each `MachineConfigPool`, using the new base image

1. Each machineconfig pool will support a `custom-external-coreos` imagestream for pulling externally built images
1.MCD continues to perform drains and reboots,, but for the most part does not change files in `/etc` or install packages per node
1. The Machine Configuration Server (MCS) will only serve a "bootstrap" Ignition configuration (pull secret, network configuration) sufficient for the node to pull the target container image.

For clusters without any custom MachineConfig at all, the MCO will deploy the result of the `mco-coreos` build.

<!-- current drawing: https://docs.google.com/drawings/d/1cGly8mjYDezVjUQiUEWd63IO_S2ZrgSbv1cT5X1KJIk/edit -->
    
#### Preserving `MachineConfig`

We cannot just drop `MachineConfig` as an interface to node configuration. Hence, the MCO will be responsible for starting new builds on upgrades or when new machine config content is rendered.

For most configuration, instead of having the MCD write files on each node, it will be added into the image build run on the cluster.
To be more specific, most content from the Ignition `systemd/units` and `storage/files` sections (in general, files written into `/etc`) will instead be injected into an internally-generated `Dockerfile` (or equivalent) that performs an effect similar to the example from the [CoreOS layering enhancement](https://github.com/coreos/enhancements/blob/main/os/coreos-layering.md#butane-as-a-declarative-input-format-for-layering).

```dockerfile=
FROM <coreos>
# This is needed 
ADD mco-rendered-config.json /etc/mco-rendered-config.json
ADD ignition.json /tmp/ignition.json
RUN ignition-liveapply /tmp/ignition.json && rm -f /tmp/ignition.json
```

This build process will be tracked via a `mco-coreos-build` `BuildConfig` object which will be monitored by the operator.

The output of this build process will be pushed to the `imagestream/mco-coreos`, which should be used by further build processes.

#### Handling booting old nodes

We can't switch the format of the oscontainer easily because older clusters may have older bootimages with older `rpm-ostree` that won't understand the new container format.  Hence firstboot upgrades would just fail.

Options:

- Double reboot; but we'd still need to ship the old image format in addition to new
  And really the only sane way to ship both is to generate the old from the new; we
  could do that in-cluster or per node or pre-generated as part of the payload
- Try to run rpm-ostree itself as a container
- Force bootimage updates (can't be a 100% solution due to UPI)

NOTE: Verify that we're doing node scaling post-upgrade in some e2e tests

#### Preserving old MCD behaviour for RHEL nodes

The RHEL 8 worker nodes in-cluster will require us to continue support existing file/unit write as well as provision (`once-from`) workflows.  See also [openshift-ansible and MCO](https://github.com/openshift/machine-config-operator/issues/1592).

#### Handling extensions

We need to preserve support for [extensions](https://github.com/openshift/enhancements/blob/master/enhancements/rhcos/extensions.md).  For example, `kernel-rt` support is key to many OpenShift use cases.

Extensions move to a `machine-os-content-extensions` container that has RPMs.  However, this is an implementation detail of the pipeline/MCO and should not be used by customers directly - only indirectly via specifying `kernelType` in a MachineConfig or via `extensions`.

#### Kernel Arguments

For now, updating kernel arguments will continue to happen via the MCD on each node via executing `rpm-ostree kargs` as it does today. See also https://github.com/ostreedev/ostree/issues/479


#### Ignition

Ignition will continue to handle the `disks` and `filesystem` sections - for example, LUKS will continue to be applied as it has been today.

However, see below.

##### Per machine state, the pointer config

See [MCO issue 1720 "machine-specific machineconfigs"](https://github.com/openshift/machine-config-operator/issues/1720).
We need to support per machine/per node state like static IP addresses and hostname.

##### Transitioning existing systems

This is a complex and multi-faceted topic.  First, let's assume the node is already
running a new enough host stack (most specifically rpm-ostree with container support).

The system is running e.g. 4.11 and will be upgrading to e.g. 4.12 which will transition
to the new format.  In this case, the user is not customizing the image at all, and we just
want to perform a seamless transition.

One key aspect we need to handle here is the "per machine state" above.

- Node currently has a large set of files written to e.g. `/etc` and `/usr/local` via
  Ignition.
- The MCO asks the node to deploy the new format image
- The MCD detects it is transitioning from old format to native container format, and
  *does not perform any filesystem modifications at all*.
- MCD simply runs `rpm-ostree rebase` + drain + `reboot`

With this approach, any locally-modified files `/etc` will be preserved by this upgrade, such
as static IP addresses, etc.  This means that e.g. rollback will still work in theory.

However, there are two problems:

First, we will need to be careful to keep e.g. `kubelet.service` in `/etc`, and not as
part of the container build also migrate it to `/usr` (which we can do now!).  If we
moved the unit, then we'd have *two* copies.

This risk is greater for helper binaries (things in `/usr/local` i.e. `/var/usrlocal`).
Those must be moved into `/usr/bin` in the image.  We will need to be careful to ensure
that the binaries in `/usr/bin` are preferred by scripts.  In general, we should be
able to handle that by using absolute paths.

#### Ignition (pointer vs provisioning vs in-image)

We will likely keep the pointer configuration as is.  Attempting to scope in
changing that now brings its own risks.

However, the MCS will need to serve a "provisioning configuration" for example,
with at least a pull secret and other configuration sufficient to pull images.
(Alternatively, we may be able to provide a "bootstrap pull secret" that allows
 doing a pull-through from the in-cluster registry)

 If we do try changing the pointer config, we may be able to have that be the sole source of provisioning configuration, and then there is no MCS at all - everything else lives in the container image.

#### Drain and reboot

The MCO will continue to perform drain and reboots.

#### Single Node OpenShift

Clearly this mechanism needs to work on single node too.  It would be a bit silly to build a container image and push it to a registry on that node, only to pull it back to the host.  But it would (should) work.

#### Reboots and live apply

The MCO has invested in performing some types of updates without rebooting.  We will need to retain that functionality.

Today, `rpm-ostree` does have `apply-live`.  One possibility is that if just e.g. the pull secret changes, the MCO still builds a new image with the change, but compares the node state (current, new) and executes a targeted command like `rpm-ostree apply-live --files /etc/kubernetes/pull-secret.json` that applies just that change live.

Or, the MCD might handle live changes on its own, writing files instead to e.g. `/run/kubernetes/pull-secret.json` and telling the kubelet to switch to that.

Today the MCO supports [live updating](https://github.com/openshift/machine-config-operator/pull/2398) the [node certificate](https://docs.openshift.com/container-platform/4.9/security/certificate_types_descriptions/node-certificates.html).

#### Node firstboot/bootstrap

Today the MCO splits node bootstrapping into two locations: Ignition (which provisions all Ignition subfields of a MachineConfig) and `machine-config-daemon-firstboot.service`, which runs before kubelet to provision the rest of the MC fields, and reboots the node to complete provisioning.

We can't quite put *everything* configured via Ignition into our image build.  At the least, we will need the pull secret (currently `/var/lib/kubelet/config.json`) in order to pull the image to the node at all.  Further, we will also need things like the image stream for disconnected operation.

(Proposed spike: Create “openshift-node-base” which has the stock MachineConfig https://issues.redhat.com/browse/MCO-286 )

In our new model, Ignition will likely still have to perform subsets of MachineConfig (e.g. disk partitioning) that we do not modify post bootstrapping.
It will also need to write certain credentials for the node to access relevant objects, such as the pull secret. The main focus of the served Ignition config will be, compared to today, setting up the MCD-firstboot.service to fetch and pivot to the layered image.

This initial ignition config we serve through the MCS will also contain all the files it wrote, which is then encapsulated in the MCD firstboot to be removed, since we do not want to have any "manually written files". We need to be mindful to preserve anything provided via the pointer config, because we need to support that for per-machine state.

Alternatively, we could change the node firstboot join to have a pull secret that only allows pulling "base images" from inside the cluster.

Analyzing and splitting this "firstboot configuration" may turn out to be a nontrivial amount of work, particularly in corner cases.

A mitigation here is to incrementally move over to things we are *sure* can be done via the image build.

##### Compatibility with openshift-ansible/windows containers

There are other things that pull Ignition:

- [openshift-ansible for workers](https://github.com/openshift/openshift-ansible/blob/c411571ae2a0b3518b4179cce09768bfc3cf50d5/roles/openshift_node/tasks/apply_machine_config.yml#L23)
- [openshift-ansible for bootstrap](https://github.com/openshift/openshift-ansible/blob/e3b38f9ffd8e954c0060ec6a62f141fbc6335354/roles/openshift_node/tasks/config.yml#L70) fetches MCS
- [windows node for openshift](https://github.com/openshift/windows-machine-config-bootstrapper/blob/016f4c5f9bb814f47e142150da897b933cbff9f4/cmd/bootstrapper/initialize_kubelet.go#L33)

#### Intersection with https://github.com/openshift/enhancements/pull/201

In the future, we may also generate updated "bootimages" from the custom operating system container.  There is a lot to write about this, but a likely direction is that
[RHEL Image Builder](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/composing_a_customized_rhel_system_image/composer-description_composing-a-customized-rhel-system-image)
will gain the ability to take a CoreOS-derived container image, and generate a disk image (e.g. Azure image, ISO) which just "wraps" it.

Image Builder would also be a natural place to use the underlying technology outside of OCP for RHEL.

Both of these things are out of scope of this enhancement at this time however.


#### Using RHEL packages - entitlements and bootstrapping

Today, installing OpenShift does not require RHEL entitlements - all that is necessary is a pull secret.

This CoreOS layering functionality will immediately raise the question of supporting `yum -y install $something` as part of their node, where `$something` is not part of our extensions that are available without entitlement.

For cluster-internal builds, it should work to do this "day 2" via [existing RHEL entitlement flows](https://docs.openshift.com/container-platform/4.9/cicd/builds/running-entitled-builds.html#builds-source-secrets-entitlements_running-entitled-builds).

Another alternative will be providing an image built outside of the cluster.

It may be possible in the future to perform initial custom builds on the bootstrap node for "day 1" customized CoreOS flows, but adds significant complexity around debugging failures.  We suspect that most users who want this will be better served by out-of-cluster image builds.

### Future Work

#### Heterogeneous clusters

To support [Heterogeneous clusters](https://github.com/openshift/enhancements/pull/1014), we clearly now need to generate
an image per supported node architecture.

Today as noted, OCP native builds do not support this.  It seems likely that the MCO will need to
generate a build per supported architecture itself, and handle rolling them out.  This is its
own large topic, such as whether we try to use a fully-fledged solution for that or hand-craft
something that just handles monitoring OCP native builds across multiple architectures.
As of right now the internal registry doesn't support manifest lists either.

Multiarch Spike: https://issues.redhat.com/browse/MCO-277

#### Hypershift

[Hypershift]https://github.com/openshift/hypershift has its own implementation of inplace upgrading for nodepools today. A special mode of the MCD is used in conjunction with Hypershift’s hosted control plane operator to facilitate this. We would definitely want feature parity in Hypershift eventually, as well as the benefits layering workflow would have for managing large amounts of hosted clusters. This can lag behind self-driving OCP for the time being.

### Risks and Mitigations

We're introducing a whole new level of customization for nodes, and because this functionality will be new, we don't yet have significant experience with it.  There are likely a number of potentially problematic "unknown unknowns".

To say this another way: until now we've mostly stuck to the model that user code should run in a container, and keep the host relatively small.  This could be perceived as a major backtracking on that model.

This also intersects heavily with things like [out of tree drivers](https://github.com/openshift/enhancements/pull/357).

We will need some time to gain experience with what works and best practices, and develop tooling and documentation.

It is likely that the initial version will be classified as "Tech Preview" from the OCP product perspective.

#### Supportability of two update mechanisms

If for some reason we cannot easily upgrade existing FCOS/RHCOS systems provisioned prior to the existence of this functionality, and hence need to support *two* ways to update CoreOS nodes, it will become an enormous burden.

Also relatedly, we would need to continue to support [openshift-ansible](https://github.com/openshift/openshift-ansible) for some time alongside the `once-from` functionality.  See also [this issue](https://github.com/openshift/machine-config-operator/issues/1592).

#### Versioning of e.g. kubelet

We will need to ensure that we detect and handle the case where core components e.g. the `kubelet` binary is coming from the wrong place, or is the wrong version.


#### Location of builds

Today, ideally nodes are isolated from each other.  A compromised node can in theory only affect pods which land on that node.
In particular we want to avoid a compromised worker node being able to easily escalate and compromise the control plane.

#### Registry availability

If implemented in the obvious way, OS updates would fail if the cluster-internal registry is down.

A strong mitigation for this is to use ostree's native ability to "stage" the update across all machines before starting any drain at all.  However, we should probably still be careful to only stage the update on one node at a time (or `maxUnavailable`) in order to avoid "thundering herd" problems, particularly for the control plane with etcd.

Another mitigation here may be to support peer-to-peer upgrades, or have the control plane host a "bootstrap registry" that just contains the pending OS update.

#### Manifest list support

We know we want heterogeneous clusters, right now that's not supported by the build and image stream APIs. Support for manifest lists by image streams and the image registry are getting tracked [here](https://issues.redhat.com/browse/IR-192).


#### openshift-install bootstrap node process

A key question here is whether we need the OpenShift build API as part of the bootstrap node or not.  One option is to do a `podman build` on the bootstrap node.

Another possibility is that we initially use CoreOS layering only for worker nodes.

##### Single Node bootstrap in place

Today [Single Node OpenShift](https://docs.openshift.com/container-platform/4.9/installing/installing_sno/install-sno-installing-sno.html) performs a "bootstrap in place" process that turns the bootstrap node into the combined controlplane/worker node without requiring a separate (virtual/physical) machine.

It may be that we need to support converting the built custom container image into a CoreOS metal image that would be directly writable to disk to shave an extra reboot.

## Design Details

### Open Questions

- Would we offer multiple base images, e.g. users could now choose to use RHEL 8.X "Z-streams" versus RHEL 8.$latest?

#### Debugging custom layers (arbitrary images)

In this proposal so far, we support an arbitrary `BuildConfig`
which can do anything, but would most likely be a `Dockerfile`.

Hence, we need to accept arbitrary images, but will have the 
equivalent of `podman history` that is exposed to the cluster administrator and us.

Having history for the final image provides clear separation about the content. For example, we will be able to tell whether a cluster is using any custom layer or just using default content shipped with OCP. Also, layering related information will be made available into must-gather so that debugging an existing cluster can be performed easily without having access to live cluster.

#### Exposing custom RPMs via butane (Ignition)

Right now we have extensions in MachineConfig; to support fully custom builds it might
suffice to expose yum/rpm-md repos and an arbitrary set of packages to
add.  

Note that Ignition is designed not to have distro-specific syntax. We'd need to either support RPM packages via Butane sugar, or think about a generic way to describe packages in the Ignition spec.

This would be a custom container builder tool that drops the files from the Ignition config
into a layer.

This could also be used in the underlying CoreOS layering proposal.

#### External images

This will need some design to make it work nicely to build images for a different target OCP version. The build cluster will need access to base images for multiple versions.  Further, the MCO today dynamically templates some content based on target platform, so the build process would need to support running the MCO's templating code to generate per-platform config at build time.

Further, we have per-cluster data such as certificates.

We may need to fall back to doing a minimal per-cluster build, just effectively supporting replacing the coreos image instead of replacing the `mco-base`.

</details>

### Test Plan

The FCOS / RHCOS base OCI containers will be published automatically by their respective CI pipelines. These pipelines already have a release promotion mechanism built into them. This means that early development versions of these containers are available (e.g. `:testing-devel` tags), hence other dependent systems, such as the MCO, can use them.

For example, the MCO’s tests can use those containers to get a signal on potentially breaking changes to rpm-ostree as well as the base container images.

MCO-specific tests will be run during each PR as part of the MCOs presubmits. In addition, the same tests which exercise the MCOs image building facilities could also be made part of the `openshift/origin` [test suite](https://github.com/openshift/origin), meaning they’ll also get run during periodic CI runs.

### Graduation Criteria

The main goal of tech preview (4.12) is to implement image build and application, to the point a newly installer cluster on the new format is functional. There will be no user-facing changes. The goal for GA(4.12 for hotfix, 4.13) is to ensure upgrade success and readiness for wider adoption by users.

#### Dev Preview -> Tech Preview

- All the below is available in 4.12 nightlies
- `machine-os-content` is in the OSTree native container format and the MCO knows how to handle it.
- A package not in the current extensions (such as kata?) can be added to a custom image along with configuration and rolled out via the MCO
- MachineConfig objects can still be created, but turn into an internal cluster image build that is rolled out instead of changed node per node.  This feeds into the “Final Pool Image” the same way as the other MCO templates.
- Installation of a new cluster is successful with the new format.
- A CI test verifies the installation of a new cluster in this mode and installing an extension.

#### Tech Preview -> GA

- A supported model to apply hotfix on 4.12 based cluster is available
- All the goals listed above are implemented.
- Any feedback from tech preview (e.g. installation of specific 3rd party software) is addressed

#### Removing a deprecated feature

Nothing will be deprecated as part of this.

### Upgrade / Downgrade Strategy

See above - this is a large risk.  Nontrivial work may need to land in the MCO to support transitioning nodes.

### Version Skew Strategy

Similar to above.

### Operational Aspects of API Extensions

The API extensions are still TBD.  There are potentially significant
operational implications to making use of custom builds.

#### Failure Modes

It is highly likely that some users will perform complex configuration
of the operating system.  This means that upgrades are at risk.

However, we will retain the current MCO approach of updating a single node
at once by default which limits the blast radius of misconfigured systems.

Another large mitigation is that some forms of failure will appear
as *build failures* and not affect running nodes at all.  We hope to encourage a CI testing flow for updates - testing the OS content *as a container* before the image gets applied to disk.

#### Support Procedures

We intend to make it extremely clear from a support perspective when
custom configuration is applied, and exactly what is there.

## Implementation History

There was a prior version of this proposal which was OpenShift specific and called for a custom build strategy.  Since then, the "CoreOS layering" effort has been initiated, and this proposal is now dedicated to the OpenShift-specific aspects of using this functionality, rather than also containing machinery to build custom images.

## Drawbacks

If we are successful; not many. If it turns out that e.g. upgrading existing RHCOS systems in place is difficult, that will be a problem.

## Alternatives

Continue as is - supporting both RHEL CoreOS and traditional RHEL (where it's more obvious how to make arbitrary changes at the cost of upgrade reliability), for example.
