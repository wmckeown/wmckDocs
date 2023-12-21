# Kubernetes Pod Disruption

## Voluntary and Involuntary disruptions
Pods do not disappear until someone (a person or controller) destroys them, or there is an unavoidable hardware or system software error.

These unvoidable cases are called *involuntary disruptions* to an application. Examples are:

* A hardware failure of the physical machine backing the node
* Cluster admin deletes VM (instance) by mistake
* Cloud provider or hypervisor makes the VM disappear
* A kernel panic
* The node disappears from the cluster due to cluster network partition
* Eviction of a pod due to the node being out of resources

Other cases are called *voluntary disruptions*. These include actions initiated by the application owner and those initiated by a Cluster Administrator. Typical application owner actions include:

* Deleting the deployment or other controller that manages the pod
* Updating a deployment's pod template causing a restart
* Directly deleting a pod (e.g. by accident)

Cluster administrator actions include:

* Draining a node for repair or upgrade
* Draining a node from a cluster to scale the cluster down
* Removing a pod from a node to permit something else to fit on that node

These actions may be taken directly by the cluster administrator, or by the automation run by the cluster admin, or by your cluster hosting provider.

!!! Caution
    Not all voluntary disruptions are constrained by Pod Disruption Budgets. For example, deleting deployments or pods bypasses Pod Disruption Budgets
    
## Pod Disruption Budgets

Kubernetes offers features that help us run highly-available applications even when we introduce frequent voluntary disruptions.

As an application owner, you an create a PodDisruptionBudget (PDB) for each application. A PDB limits the numnber of Pods of a replicated application that are down simultaneously from voluntary disruptions. For example, a quorum-based application would like to ensure that the number of replicas running is never brought down below the number needed for a quorum. A web front-end might want to ensure that the number of replicas serving load never falls below a certain percentage of the total.

Cluster managers and hosting providers should use tools which respect PodDisruptionBudgets by calling the Eviction API instead of directly deleting pods or deployments.

For example, the `kubectl drain` subcommand lets you mark a node as going out of service. When you run `kubectl drain`, the tool tries to evict all the Pods on the Node you are taking out of service. The eviction request that `kubectl` submits on your behalf my be temporarily rejected, so the tool periodically retries all failed requests until all Pods on the target node are terminated, or until a configurable timeout is reached.

A PDB specifies the number of replicas that an application can tolerate having, relative to how many it is intended to have. For example, a Deployment which has a `.spec.replicas: 5` is supposed to have 5 pods at any given time. If its PDB allows for there to be 4 at a time, then the Eviction API will allow voluntary disruption of one (but not two) pods at a time.

The group of Pods that comprise the application is specified using a label selector, the same as the one used by applications controller (deployment, stateful-set etc.)

The "intended" number of pods is computed from the `.spec.replicas` of the workload resource that is managing those pods. The control plane discovers the owning workload resource by examining the `.metadata.ownerReferences` of the Pod.

Involuntary disruptions cannot be prevented by PDBs but they do count against the budget.

Pods which are deleted or unavailable due to a rolling upgrade do count against the disruption budget, but workload resources such (such as Deployment and StatefulSet) are not limited by PDBs when doing rolling upgrades. Instead, the handling of failures during application updates is configured in the spec for the specific workload resource.

When a pod is evicted using the Eviction API, it is gracefully terminated, honoring the `terminationGracePeriodSeconds` setting in its PodSpec.

## PodDisruptionBudget example
Consider a cluster with 3 nodes, `node-1` through `node-3`. The cluster is running several applications. One of them has 3 replicas intially called `pod-a`, `pod-b` and `pod-c`. Another, unrelated pod without a PDB, called `pod-x`, is also shown. Intially, the pods are laid out as follows:

| node-1          | node-2          | node-3          |
| --------------- | --------------- | --------------- |
| pod-a available | pod-b available | pod-c available |
| pod-x available |                 |                 |

All 3 pods are part of a deployment, and they collecively have a PDB which requires there to be at least 2 of the 3 pods to be available at all times.

For example, assume the cluster admin wants to reboot into a new kernel version to fix a bug in the kernel. The cluster admin first tries to drain `node-1` using the `kubectl drain` command. That tool tries to evict `pod-a` and `pod-x`. This succeeds immediately. Both pods go into the `terminating` state at the same time. This puts the cluster in this state:

| node-1 draining   | node-2          | node-3          |
| ----------------- | --------------- | --------------- |
| pod-a terminating | pod-b available | pod-c available |
| pod-z terminating |                 |                 |

The deployment notices that one of the pods is terminating, so it creates a replacement called `pod-d`. Since `node-1` is cordoned, it lands on another node. Something has also created `pod-y` as a replacement for `pod-x`.

!!! Note
        For a StatefulSet, `pod-a` which would be called something like `pod-0`, would need to terminate completely before its replacement, which is also called `pod-0` but has a different UID, could be created. Otherwise, the above example applies to a StatefulSet as well.

Now the cluster is in this state.

| node-1 draining   | node-2          | node-3          |
| ----------------- | --------------- | --------------- |
| pod-a terminating | pod-b available | pod-c available |
| pod-z terminating | pod-d starting  | pod-y           |

At some point, the pods terminate, and the cluster looks like this:


| node-1 draining | node-2          | node-3          |
| --------------- | --------------- | --------------- |
|                 | pod-b available | pod-c available |
|                 | pod-d starting  | pod-y           |

At this point, if an impatient cluster admin tries to drain `node-2` or `node-3`, the drain command will block, because there are only 2 available pods for the deployment, and its PDB requires at least 2. After some time passes, `pod-d` becomes available.

The cluster state now looks like this:

| node-1 draining | node-2          | node-3          |
| --------------- | --------------- | --------------- |
|                 | pod-b available | pod-c available |
|                 | pod-d available | pod-y           |

Now, the cluster admin tries to drain `node-2`. The drain command will try to evict two pods in the same order, say `pod-b` first and then `pod-d`. It will succeed in evicting `pod-b`. But, when it tries to evict `pod-d` it will be refused because that would leave only one pod available for the deployment.

The deployment creates a replacement for `pod-b` called `pod-e`. Because there are not enough resources in the cluster to schedule `pod-e`, the drain will again block. The cluster may then end up in this state.

| node-1 draining | node-2            | node-3          | no node       |
| --------------- | ----------------- | --------------- | ------------- |
|                 | pod-b terminating | pod-c available | pod-e pending |
|                 | pod-d available   | pod-y           |               |

At this point, the cluster admin needs to add a node back to the cluster to proceed with the upgrade.

You can see how Kubernetes varies the rate at which disruptions can happen, according to:

* How many replicas an application needs
* How long it takes to gracefull shut down an instance
* How long it takes a new instanvce to start up
* The type of controller
* The cluster's resource capacity


