# Interrogating Operators on Openshift
As an OpenShift Technical Account Manager, helping Red Hat customers migrate from OpenShift 3 to OpenShift 4 has become a primary job function.  There are new terms, new objects, and generally an all around more automated experience.  The key part of this automation is the "Operator" and if you're reading this, we'll assume that you have a passing knowledge of Operators.  If you don't, a good primer on "Operators" can be found [here](https://www.redhat.com/en/topics/containers/what-is-a-kubernetes-operator).

If you're following along on your own cluster, you'll need a user with `cluster-admin` permissions and `oc` in your `$PATH`.  We'll also use Go templates so if you're not familiar with those, a self-led workshop is available [here](https://github.com/brandisher/openshift-and-gotemplates-workshop).  One last thing to note; we're using OpenShift 4.5 as the reference so there may be minor variations depending on your OpenShift version and installation method.  Now that we have the basics covered, let's get started!

## Finding all the Operators
The first thing we'll want to identify is what operators are part of the platform versus which operators are installed after the cluster is running.  OpenShift has a convenient API resource named `clusteroperators` which provides you with the list of native OpenShift operators.

```
$ oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.4     True        False         False      42h
cloud-credential                           4.5.4     True        False         False      43h
cluster-autoscaler                         4.5.4     True        False         False      42h
config-operator                            4.5.4     True        False         False      42h
console                                    4.5.4     True        False         False      42h
...
```
This one-liner will show that all of these operators are the same `Kind`.
```
$ oc get co -o go-template='{{ range .items }}{{ print .metadata.name " => " .kind "\n" }}{{ end }}'
authentication => ClusterOperator
cloud-credential => ClusterOperator
cluster-autoscaler => ClusterOperator
config-operator => ClusterOperator
console => ClusterOperator
...
```
That's easy enough!  But what about operators that aren't `ClusterOperators`? You'll notice that if you run `oc api-resources | grep -i operator` there are no canonical `operator` api resources.  In other words, we can't do `oc get operators` to see every operator on the cluster.  The reason we can do `oc get clusteroperators` is because the set of operators under that resource are all managed by the `cluster-version-operator` which provides a custom resource defintion (CRD) that describes a `ClusterOperator`.  Each of the `ClusterOperators` also provide their own CRD(s) for additional CRs that they create and manage.  You can think of this like a family tree where the   `cluster-version-operator` is the parent of the `ClusterOperator` instances, and each `ClusterOperator` instance can in turn create its own children of a specific type (e.g. etcd-operator creates and manages etcd instances).  The tree structure is what allows the OpenShift operators to be managed as a unit and also why they have their own resource.

For demonstration purposes, we'll use the Node Feature Discovery (NFD) operator as our example operator that's been installed independently on the cluster.  The NFD operator has been installed using the Operator Lifecycle Manager (OLM) and its installed in all namespaces. Operators installed through OLM get a `ClusterServiceVersion` that we can use to see where they're installed and what version is installed.
```
$ oc project default
Now using project "default".

$ oc get csv
NAME                        DISPLAY                  VERSION   REPLACES                    PHASE
nfd.4.5.0-202008100413.p0   Node Feature Discovery   4.5.0     nfd.4.5.0-202007281827.p0   Succeeded
```
That confirms that our operator has been installed in the `default` namespace.  We can use the value of the name column to confirm its been installed in all of the other namespaces as well.
```
$ oc get csv --no-headers --all-namespaces | grep 'nfd.4.5.0-202008100413.p0' | wc -l
57

$ oc get projects --no-headers | wc -l
57
```
Its worth noting that we're pivoting on a specific version of the NFD operator.  If we ran this command while a new version was being deployed, the count of the first command may not match the number of projects as the new version rolls out.  Likewise, if we only used `grep 'nfd'` in our command, then we'll have a count higher than the number of projects since the old version will stick around until the new version is successfully deployed.

## Untangling the Operator
* From finding the operators to mapping all of their components/resources
* What version of the operator am I running?
* What images does the operator use?
* What objects does the operator manage?
* What is managing the operator?

## Leveraging Go templates for a hollistic view
* Template for single operator view
* Template for all operators
* Templates for special cases

