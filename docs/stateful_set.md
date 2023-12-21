# Understanding StatefulSets

Taken from: [https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

StatefulSet is the workload API object used to manage stateful applications

It manages the deployment and scaling of a set of pods, and provides guarantees about the ordering and uniqueness of these Pods.

Like a Deployment, a StatefulSet manages Pods that are based on an identical container spec. Unlike a deployment, a StatefulSet maintains a sticky identity for each of their pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.

If you wnt to use storage volumes to provide persistence for your workload, you can use a StatefulSet as part of the solution. Although individual Pods in a StatefulSet are susceptible to failure, the persistent Pod identifiers make it easier to match existing volumes to the new Pods that replace any that have failed.

## Using StatefulSets

StatefulSets are valuable for applications that require one or more of the following.

* Stable, unique network identifiers
* Stable, persistent storage
* Ordered, graceful deployment and scaling
* Ordered, automated rolling updates

In the above, stable is synonymous with persistence across Pod (re)scheduling. If an application doesn't require any stable identifiers or ordered deployemnt, deletion or scaling, you should deploy your application using a workload object that provides a set of stateless replicas. Deployment or ReplicaSet may be better suited to your stateless needs.

## Limitations

* The storage for a given Pod must be provisioned by a PersistentVolume Provisioner based on the requested `storage class`, or pre-provisioned by an admin
* Deleting and/or scaling a StatefulSet down *will not* delete the volumes associated with the StatefulSet. This is done to ensure data safety, which is generally more valuable that an automatic purge of all related StatefulSet resources
* StatefulSets currently require a HeadlessService to be be responsible for the network identity of the Pods. You are responsible for creating this Service
* StatefulSets do not provide any guarantees on the termination of pods when a StatefulSet is deleted. To achieve ordered and graceful termination of the pods in the StatefulSet, it is possible to scale the StatefulSet down to 0 prior to deletion.
* When using RollingUpdates with the default PodManagementPolicy (`OrderedReady`), it's possible to get into a broken state that requires manual intervention to repair.

## Components
The example below demonstrates the components of a StatefulSet.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  minReadySeconds: 10 # by default is 0
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi

```

In the above example:

* A headless service, named `nginx`, is used to control the network domain.
* The StatefulSet, named `web` has a Spec that indicates that 3 replicas of the nginx container will be launched in unique Pods.
* The `volumeClaimTemplates` will provide stable storage using PersistentVolumes provisioned by a PersistentVolume Provisioner

The name of a StatefulSet object must be a valid DNS subdomain name

### Pod Selector
You must set the `.spec.selector` field of a StatefulSet to match the labels of its `.spec.template.metadata.labels`. Failing to specify a matching Pod Selector will result in a validation error during StatefulSet creation.

### Volume Claim Templates
You can set the `.spec.volumeClaimTemplates` which can provide stable storage using `PersistentVolumes` provisioned by a PersistentVolume Provisioner

### Minimum ready seconds
Kubernetes v1.25 FEATURE 

`.spec.minReadySeconds` is an optional field that specifies the minimum number of seconds for which a newly created Pod should be running and ready without any of its containers crashing, for it to be considered available. This is used to check progression of a rollout when using a Rolling Update strategy. This field defaults to 0 (the Pod will be considered available as soon as it is ready)

## Pod Identity
StatefulSet Pods have a unique indentity that consists of an ordinal, a stable network identity, and stable storage. The identity sticks to the Pod, regardless of which node it is (re)scheduled on.

### Ordinal Index
For a StatefulSet with N replicas, each pod in the StatefulSet will be assigned an integer ordinal, that is unique over the Set. By default, pods will be assigned ordinals from 0 up through N-1.

### Stable Network ID
Each Pod in a StatefulSet derives its hostname from the name of the StatefulSet and the ordinal of the Pod. The patternf ro the constructed hostname is `$(statefulset name)-$(ordinal)`. The example above will create three pods name `web-0, web-1, web-2`. A StatefulSet can use a Headless Service to control the domain of its Pods. the domain managed by this Service takes the form: `$(service name).$(namespace).svc.cluster.local`, where "cluster.local" is the cluster domain. As each Pod is created, it gets a matching DNS subdomain, taking the form: `$(podname).$(governing service domain)`, where the governing service is defined by the `serviceName` field on the StatefulSet.

Depending on how DNS is configured in your cluster, you may not be able to look up the DNS name for a newly-run Pod immediately. This behaviour can occur when other clients in the cluster have already sent queries for the hostname of the Pod before it was created. negative caching (normal in DNS) means the results of previous failed lookups are remembered and reused, even after the Pod is running, for at least a few seconds.

If you need to discover Pods promptly after they are created, you have a few options:

* Query the Kubernetes API directly (for example using a watch) rather than relying on DNS lookups
* Decrease the time of caching in your Kubernetes DNS provider (typically this mean editing the config map for CoreDNS, which currently caches for 30 seconds)

As mentioned in the limitations section, you are responsible for creating the Headless Service responsible for the network identity of the pods.

Here are some examples of choices for Cluster Domain, Service name, StatefulSet name, and how that affecrs the DNS names for the StatefulSet's Pods.

| Cluster Domain | Service (ns/name) | StatefulSet (ns/name) | StatefulSet Domain              | Pod DNS                                      | Pod Hostname |
| -------------- | ----------------- | --------------------- | ------------------------------- | -------------------------------------------- | ------------ |
| cluster.local  | default/nginx     | default/web           | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0..N-1} |
| cluster.local  | foo/nginx         | foo/web               | nginx.foo,svc.cluster.local     | web-{0..N-1}.nginx.foo.svc.cluster.local     | web-{0..N-1} |
| kube.local     | foo/nginx         | foo/web               | nginx.foo.svc.kube.local        | web-{0..N-1}.nginx.foo.svc.kube.local        | web-{0..N-1} |

!!! Note
        Cluster Domain will be set to `cluster.local` unless configured otherwise

### Stable Storage
For each VolumeClaimTemplate entry defined in a StatefulSet, each Pod recieves one PersistentVolumeClaim. In the nginx example above, each pod recieves a single PersistentVolume with a StorageClass of `my-storage-class` and 1 Gib of proivisioned storage. If no StorageClass is specified, then the defaultStorageClass will be used. When a Pod is (rescheduled) onto a node, its `volumeMounts` mount the PersistentVolumes associated with its PersistentVolume Claims. Note that, the PersistentVolumes associated with the Pods' PersistentVolumeClaims are not deleted when the Pods or StatefulSet are deleted. This is done manually.

### Pod Name Label
When the StatefulSet controller creates a Pod, it adds a label, `statefulset/kubernetes.io/pod-name`, that is set to the name of the Pod. This label allows you to attach a Service to a specific Pod in the StatefulSet.

## Deployment and Scaling Guarantees

* For a StatefulSet with N replicas, when Pods are being deployed, they are created sequentially, in order from {0..N-1}.
* When Pods are being deleted, they are terminated in reverse order, from {N-1..0}
* Before a scaling operation is applied to a Pod, all of its predecessors must be Running and Ready
* Before a Pod is terminated, all of its successors must be completely shutdown
  
The StatefulSet should not specify a `pod.Spec.TerminationGracePeriodSeconds` of 0, This practice is unsafe and strongly discouraged.

When the nginx example above is created, three Pods will be deployed in the order web-0, web-1, web-2. web-1 will not be deployed before web-0 is Running and Ready, and web-2 will not be deployed until web-1 is Running and Ready. If web-0 should fail, after web-1 is Running and Ready, but before web-2 is launched, web-2 will not be launched until web-0 is successfully relaunched and becomes Running and Ready.

If a user were to scale the deployed example by patching the StatefulSet such that `replicas=1`, web-2 would be terminated first. web-1 would not be terminated until web-2 is fully shutdown and deleted. If web-0 were to fail after web-2 has been completely deleted and shutdown, but prior to web-1's termination, web-1 would not be terminated until web-0 is Running and Ready.

### Pod Management Policies
StatefulSet allows you to relax its ordering guarantees while preserving its uniqueness and identity guarantees via its `.spec.podManagementPolicy` field.

#### OrderedReady Pod Management
`OrderedReady` pod management is the default for StatefulSets. It implemenets the behaviour described above.

#### Parallel Pod Management
`Parallel` pod management tells the StatefulSet controller to launch or terminate all Pods in parallel, and not to wait for Pods to become Running and Ready or completely terminated prior to launching or terminating another Pod. This option only affects the behaviour for scaling operations. Updates are not affected.

## Update Strategies
A StatefulSet's `.spec.updateStrategy` field allows you to configure and disable automated rolling updates for containers, labels, resource request/limits and annotations for the Pods in a StatefulSet. There are two possible values:

###### On Delete
When a StatefulSet's `.spec.updateStrategy.type` is set to `OnDelete`, the StatefulSet controller will not automatically update the Pods in a StatefulSet. Users must manually delete Pods to cause the controller to create new Pods that reflect modifications made to a StatefulSet's `.spec.template`.

###### Rolling Update
The `RollingUpdate` update strategy implements automated, rolling updates for the Pods in a StatefulSet. This is the default update strategy.

## Rolling Updates
When a StatefulSet's `.spec.updateStrategy.type` is set to `RollingUpdate`, the StatefulSet controller will delete an recreate each Pod in the StatefulSet. It will proceed in the same order as Pod Termination (from the largest ordinal to the smallest), updating each Pod one at a time.

The Kubernetes control plan waits until an updated Pod is Running and Ready prior to updating its predecessor. If you have set `.spec.minReadySeconds`, the control plane additionally waits that amount of time after the Pod turns ready, before moving on.

### Partitioned rolling updates
The `RollingUpdate` update strategy can be partitioned, by specifying a `.spec.updateStrategy.rollingUpdate.partition`. If a partition is specified, all Pods with an ordinal that is greater than or equal to the partition will be updated when the StatefulSet's `.spec.template` is updated. All Pods with an ordinal that is less than the partition will not be updated, and, even if they are deleted, they will be recreated at the previous version. If a StatefulSet's `.spec.updateStrategy.rollingUpdate.partition` is greater than its `.spec.replicas`, updates to its `.spec.template` will not be propagated to its Pods. In most cases you will not need to use a partition, but they are useful if you want to stage an update, roll out a canary, or perform a phased roll out.

### Maximum Unavailable Pods
**FEATURE STATE:** `Kubernetes 1.24 [alpha]`

You can control the maximum number of Pods that can be unavailable during an update by specifying the `.spec.updateStrategy.rollingUpdate.maxUnavailable` field. The value can be an absolute number (for example, `5`) or a percentage of desired Pods (for example, `10%`). Absolute number is calculated from the percentage value by rounding it up. This field cannot be 0. The default setting is 1.

This field applies to all Pods in the range `0` to `replicas - 1`. If there is any unavailable Pod in the range `0` to `replicas - 1` it will be counted towards `maxUnavailable`.

!!! Note
        The `maxUnavailable` field is in Alpha stage and is honoured only by API servers that are running with the `MaxUnavailableStatefulSet` feature gate enabled

### Forced rollback
When using Rolling Updates with the default Pod Management Policy (`OrderedReady`) it's possible to get into a broken state that requires manual intervention to repair.

If you update the Pod template to a configuration that never becomes Running and Ready (for example, due to a bad binary or application-level configuration error), StatefulSet will stop the rollout and wait.

In this state, it is not enough to revert the Pod template to a good configuration. Due to a known issue, StatefulSet will continue to wait for the broken Pod to become Ready (which never happens) before it will attempt to revert it back to the working configuration.

After reverting the template, you must also delete any Pods that StatefulSet had already attempted to run with the bad configuration. StatefulSet will then begin to recreate the Pods using the reverted template.

### Replicas
`.spec.replicas` is an optional field that specifies the number of desired Pods. It defaults to 1.

Should you manually scale a deployment, example via `kubectl scale statefulset statefulset --replicas=X`, and then you update that StatefulSet based on a manifest (for example: by running `kubectl apply -f statefulset.yaml`), then applying that manifest overwrites the manual scaling that you previously did.

If a HorizontalPodAutoscaler (or any similar API for horizontal scaling) is managing scaling for a Statefulset, don't set `.spec.replicas`. Instead, allow the Kubernetes control plane to manage the `.spec.replicas` field automatically.











