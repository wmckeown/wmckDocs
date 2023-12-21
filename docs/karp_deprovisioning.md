# Karpenter Deprovisioning

Karpenter sets a Kubernetes finaliser on each node it provisions. The finaliser specifies additional actions the Karpenter controller will take in response to a node deletion request. Thes include:

* Marking the node as unschedulable, so no further pods can be scheduled there.
* Evicting all pods other than daemonsets on the node
* Terminating the instance from the cloud provider
* Deleting the node from the Kubernetes cluster

## How Karpenter nodes are deprovisioned
There are both automated and manual ways of deprovisioning nodes provisioned by Karpeneter

* Provisioner Deletion: Nodes are considered to be "owned" by the Provisioner that launched them. Karpenter will gracefully terminate nodes when a provisioner is deleted. Nodes may be reparented to another provisioner by modifying their labels. For example: `kubectl label node -l karpenter.sh/provisioner-name=source-provisioner-name karpenter.sh/provisioner-name=destination-provisioner-name --overwrite`
* Node empty: Karpenter workloads when the last workload (non-daemonset) pod stops running on a node. From that point, Karpenter waits the number of seconds set by `ttlSecondsAfterEmpty` in the provisioner, then Karpenter requests to delete the node. This feature can keep costs down by removing nodes that are no longer being used for workloads
* Node expired: Karpenter requests to delete the node after a set number of seconds, based on the provisioner `ttlSecondsUntilExpired` value, from the time the node was provisioned. One use case for node expiry is to handle node upgrades. Old nodes (with a potentially outated Kubernetes version or operating system) are deleted, and replaced with nodes on the current version (assuming that you requested the latest version rather than a specific version).
* Consolidation: Karpenter works to actively reduce the cluster cost by indentifying when nodes can be removed as their workloads will run on other nodes in cluster and when nodes can be replaced with cheaper variants due to a change in the workloads
* Interruption: If enabled, Karpenter will watch for upcoming involuntary interruption events that could affect your nodes (health events, spot interruption etc.) and will cordon, drain and terminate the node(s) ahead of the event to reduce workload disruption.

!!! Note
        * Automated Deprovisioning is configured through the ProvisionerSpec `.ttlSecondsAfterEmpty`, `.ttlSecondsUntilExpired` and `.consolidation.enabled` fields. If these are not configured,Karpenter will not default values for them and will not terminate nodes for that purpose
        * Keep in mind that a small NodeExpiry results in a higher churn in cluster activity. So. for example if a cluster brings up all nodes at once, all the pods on those nodes would fall into the same batching window on expiration
        * Pods without an ownerRef (also called "controllerless" or "naked" pods) will be evicted during voluntary node disruption, such as expiration or consolidation. A pod with the annotation `karpenter.sh/do-not-evict: true` will cause its node to be opted out from voluntary node disruption workflows

* Node deleted: You could use `kubectl` to manually remove a single Karpenter node:

```bash
# Delete a specific node
kubectl delete node $NODE_NAME

# Delete all nodes owned by any provisioner
kubectl delete nodes -l karpenter.sh/provisioner-name

# Delete all nodes owned by a specific provisioner
kubectl delete nodes -l karpenter.sh/provisioner-name=$PROVISIONER_NAME
```

Whether through node expiry or manual deletion, Karpenter seeks to follow graceful termination procedures as described in Kubernetes Graceful node shutdown documentation. If the Karpenter controller is removed or fails, the finalisers on the nodes are orphaned and will require manual removal.

!!! Note

        By adding the finalizer, Karpenter improves the default Kubernetes process of node deletion, When you run `kubectl delete node` on a node without a finalizer, thenode is deleted without riggering the finalization logic. The instance will continue running in EC2, even though there is no longer a node object for it. The kubelet isn't watching for its own existence, so if a node is deleted the kubelet doesn't terminate itself. All the pod objects get deleted by a garbage collection process later, because the pods' node is gone.

## Consolidation
Karpenter has two mechanisms for cluster consolidation:

* Deletion - A node is eligible for deletion if all of its pods can run on free capacity of other nodes in the cluster
* Replace - A node can be replaced if all of its pods can run on a combination of free capacity of the other nodes in the cluster and a single, cheaper replacement node

When there are multiple nodes that could be potentially deleted or replaced, Karpenter will choose to consolidate the node that overall disrupts your workloads the least by preferring to terminate:

* Nodes running few pods
* Nodes that will expire soon
* Nodes with lower priority pods

!!! Note
        For spot nodes, Karpenter only uses the deletion consolidation mechanism. It will not replace a spot node with a cheaper spot node. Spot instance types are selected with the `price-capacity-optimized` strategy and often the cheapest spot istance type is not laucnched due to the chance of interruption. Consolidation would then replace the spot instance with a cheaper instance negating the `price-capacity-optimized` strategy and increasing the interruption rate.

## Interruption
If interruption-handling is enabled, Karpenter will watch for upcoming involuntary interruption events that would cause disruption to your workloads. These interruption events include:

* Spot Interruption Warnings
* Scheduled Change Health Events (Maintenance Events)
* Instance Terminating Events
* Instance Stopping Events

When Karpenter detects one of these events will occur to your nodes, it automatically cordons, drains and terminates the node(s) ahead of the interruption event to give the maximum amount of time