---
assignees:
- bprashanth
- enisoc
- erictune
- foxish
- janetkuo
- kow3ns
- smarterclayton
---

{% capture overview %}
**StatefulSets are a beta feature in 1.5. This feature replaces the 
PetSets feature from 1.4. Users of PetSets are referred to the 1.5 
[Upgrade Guide](/docs/tasks/stateful-set/upgrade-from-petsets-to-stateful-sets/)
for further information on how to upgrade existing PetSets to StatefulSets.**

A StatefulSet is a Controller that provides a unique identity to its Pods. It provides
guarantees about the ordering of deployment and scaling.
{% endcapture %}

{% capture body %}

### When to Use a StatefulSet
StatefulSets are valuable for applications that require one or more of the 
following.

* Stable, unique network identifiers.
* Stable, persistent storage.
* Ordered, graceful deployment and scaling.
* Ordered, graceful deletion and termination.

In the above, stable is synonymous with persistent across Pod (re) schedulings.
If an application doesn't require any stable identifiers or ordered deployment, 
deletion, or scaling, you should deploy your application with a controller that 
provides a set of stateless replicas. Such controllers, such as 
[Deployment](/docs/user-guide/deployments/) or 
[ReplicaSet](/docs/user-guide/replicasets/) may be better suited to your needs.

### Limitations
* StatefulSet is a beta resource, not available in any Kubernetes release prior to 1.5.
* As with all alpha/beta resources, you can disable StatefulSet through the `--runtime-config` option passed to the apiserver.
* The storage for a given Pod must either be provisioned by a [PersistentVolume Provisioner](http://releases.k8s.io/{{page.githubbranch}}/examples/experimental/persistent-volume-provisioning/README.md) based on the requested `storage class`, or pre-provisioned by an admin.
* Deleting and/or scaling a StatefulSet down will *not* delete the volumes associated with the StatefulSet. This is done to ensure data safety, which is generally more valuable than an automatic purge of all related StatefulSet resources.
* StatefulSets currently require a [Headless Service](/docs/user-guide/services/#headless-services) to be responsible for the network identity of the Pods. You are responsible for creating this Service.
* Updating an existing StatefulSet is currently a manual process.

### Components
The example below demonstrates the components of a StatefulSet. 

* A Headless Service, named nginx, is used to control the network domain. 
* The StatefulSet, named web, has a Spec that indicates that 3 replicas of the nginx container will be launched in unique Pods.
* The volumeClaimTemplates, will provide stable storage using [PersistentVolumes](/docs/user-guide/volumes/) provisioned by a 
 PersistentVolume Provisioner.

```yaml
---
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
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: gcr.io/google_containers/nginx-slim:0.8
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
      resources:
        requests:
          storage: 1Gi
```

### Pod Identity
StatefulSet Pods have a unique identity that is comprised of an ordinal, a 
stable network identity, and stable storage. The identity sticks to the Pod, 
regardless of which node it's (re) scheduled on.

__Ordinal Index__

For a StatefulSet with N replicas, each Pod in the StatefulSet will be 
assigned an integer ordinal, in the range [0,N), that is unique over the Set. 

__Stable Network ID__

Each Pod in a StatefulSet derives its hostname from the name of the StatefulSet 
and the ordinal of the Pod. The pattern for the constructed hostname 
is `$(statefulset name)-$(ordinal)`. The example above will create three Pods 
named `web-0,web-1,web-2`.
A StatefulSet can use a [Headless Service](/docs/user-guide/services/#headless-services)
to control the domain of its Pods. The domain managed by this Service takes the form: 
`$(service name).$(namespace).svc.cluster.local`, where "cluster.local" 
is the [cluster domain](http://releases.k8s.io/{{page.githubbranch}}/build/kube-dns/README.md#how-do-i-configure-it). 
As each Pod is created, it gets a matching DNS subdomain, taking the form: 
`$(podname).$(governing service domain)`, where the governing service is defined 
by the `serviceName` field on the StatefulSet.

Here are some examples of choices for Cluster Domain, Service name, 
StatefulSet name, and how that affects the DNS names for the StatefulSet's Pods.

Cluster Domain | Service (ns/name) | StatefulSet (ns/name)  | StatefulSet Domain  | Pod DNS | Pod Hostname |
-------------- | ----------------- | ----------------- | -------------- | ------- | ------------ |
 cluster.local | default/nginx     | default/web       | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0..N-1} |
 cluster.local | foo/nginx         | foo/web           | nginx.foo.svc.cluster.local     | web-{0..N-1}.nginx.foo.svc.cluster.local     | web-{0..N-1} |
 kube.local    | foo/nginx         | foo/web           | nginx.foo.svc.kube.local        | web-{0..N-1}.nginx.foo.svc.kube.local        | web-{0..N-1} |

Note that Cluster Domain will be set to `cluster.local` unless 
[otherwise configured](http://releases.k8s.io/{{page.githubbranch}}/build/kube-dns/README.md#how-do-i-configure-it).

__Stable Storage__

Kubernetes creates one [PersistentVolumes](/docs/user-guide/volumes/) for each 
VolumeClaimTemplate, as specified in the StatefulSet's volumeClaimTemplates field
In the example above, each Pod will receive a single PersistentVolume with a storage 
class of `anything` and 1 Gib of provisioned storage. When a Pod is (re) scheduled onto
a node, its `volumeMounts` mount the PersistentVolumes associated with its 
PersistentVolume Claims. Note that, the PersistentVolumes associated with the 
Pods' PersistentVolume  Claims are not deleted when the Pods, or StatefulSet are deleted. 
This must be done manually.

### Deployment and Scaling Guarantee

* For a StatefulSet with N replicas, when Pods are being deployed, they are created sequentially, in order from {0..N-1}. 
* When Pods are being deleted, they are terminated in reverse order, from {N-1..0}.
* Before a scaling operation is applied to a Pod, all of its predecessors must be Running and Ready. 
* Before a Pod is terminated, all of its successors must be completely shutdown.

The StatefulSet should not specify a `pod.Spec.TerminationGracePeriodSeconds` of 0. The practice of setting a `pod.Spec.TerminationGracePeriodSeconds` of 0 seconds is unsafe and strongly discouraged. For further explanation, please refer to [force deleting StatefulSet Pods](/docs/tasks/manage-stateful-set/delete-pods/#deleting-pods).

When the web example above is created, three Pods will be deployed in the order 
web-0, web-1, web-2. web-1 will not be deployed before web-0 is 
[Running and Ready](/docs/user-guide/pod-states), and web-2 will not be deployed until 
web-1 is Running and Ready. If web-0 should fail, after web-1 is Running and Ready, but before 
web-2 is launched, web-2 will not be launched until web-0 is successfully relaunched and 
becomes Running and Ready. 

If a user were to scale the deployed example by patching the StatefulSet such that
`replicas=1`, web-2 would be terminated first. web-1 would not be terminated until web-2 
is fully shutdown and deleted. If web-0 were to fail after web-2 has been terminated and 
is completely shutdown, but prior to web-1's termination, web-1 would not be terminated 
until web-0 is Running and Ready.
{% endcapture %}
{% include templates/concept.md %}