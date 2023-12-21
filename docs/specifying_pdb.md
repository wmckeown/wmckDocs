# Specifying a Disruption Budget for your App
Taken from [https://kubernetes.io/docs/tasks/run-application/configure-pdb/](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)

This page shows how to limit the number of concurrent disruptions that your application experiences, allowing for higher availability while permitting the cluster adminstrator to manage the cluster's nodes.

## Protecting an Application with a PodDisruptionBudget

1. Identify what application you want to proctect with a PodDisruptionBudget (PDB)
2. Think about how your application reacts to disruptions
3. Create a PDB definition as a YAML file
4. Create the PDB object from the YAML file

## Identify an application to Protect

The most common use case when you want to protect an application specified by one of the built-in Kubernetes controllers:

* Deployment
* ReplicationController
* ReplicaSet
* StatefulSet

In this case, make a note of the controller's `.spec.selector`; the same selector goes into the PDBs `.spec.selector`.

From v1.15 PDBs support custom controlles where the scale subresource is enabled.

You can also use PDBs with Pods which are not controlled by one of the above controllers, or arbitrary groups of Pods, but these are some restrictions defined in Arbitrary Controllers and Selectors.

## Think about how your application reacts to disruptions

Decide how many instances can be down at the same time for a short period of time due to voluntary disruption.

* Stateless frontends:
    * Concern: don't reduce serving capacity by more than 10%
        * Solution: use PDB with minAvailable 90% for example
* Single-Instance Stateful Application:
    * Concern: do not terminate this application without talking to me.
        * Possible Solution 1: Do not use a PDB and tolerate ocassional downtime
        * Possible Solution 2: Set PDB with `maxUnavailable=0`. Have an understanding (outside of Kubernetes) that the cluster operator needs to consult you before termination. When the cluster operator contacts you, prepare for downtime, and then delete the PDB to indicate readiness for disruption. Recreate afterwards.
* Multiple-instance Stateful application such as Consul, ZooKeeper or etcd:
    * Concern: do not reduce the number of instances below quroum, otherwise writes fail.
        * Possible Solution 1: Set `maxUnavailable=1` (works with varying scale of application).
        * 


## Specifying a PodDisruptionBudget

A `podDisruptionBudget` has three fields:

* A label selector `.spec.selector` to be speicify the set of pods to which it applies. This field is required.
* `.spec.minAvailable` which is a description of the number of pods from that set that must still be available after the eviction, even in the absence of the evicted pod. `minAvailable` can either be an absolute number or a percentage.

