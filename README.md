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

Now that we know how to find all of the Operators that comprise our OpenShift cluster and all of the Operators that have been installed separately from the cluster operators, let's start untangling the pieces and demonstrate how they fit together.

## Untangling the Operator
Since `ClusterOperator`s will (for the most part) always be the same, we're going to use them as a starting point and then compare/contrast against a non-`ClusterOperator`.

We'll use the authentication `ClusterOperator` for this section and to start, let's get a baseline of the Operator's stats.
```
$ oc get co/authentication
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication   4.5.3     True        False         False      3d2h
```
The output is pretty self explanatory so we'll dive into the details a bit more. First, we'll grab the YAML for the Operator and look at the `managedFields` section.
```
$ oc get co/authentication -o yaml
...
  managedFields:
  - apiVersion: config.openshift.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:exclude.release.openshift.io/internal-openshift-hosted: {}
      f:spec: {}
      f:status:
        .: {}
        f:extension: {}
    manager: cluster-version-operator
    operation: Update
    time: "2020-08-28T16:56:59Z"
  - apiVersion: config.openshift.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        f:conditions: {}
        f:relatedObjects: {}
        f:versions: {}
    manager: authentication-operator
    operation: Update
    time: "2020-08-28T22:19:27Z"
```
This looks a lot more complicated than it is.  Luckily, we can use OpenShift's built-in documentation to figure out what they mean! A snippet is below this explains what we're looking at.
```
$ oc explain clusteroperator.metadata.managedFields
KIND:     ClusterOperator
VERSION:  config.openshift.io/v1

RESOURCE: managedFields <[]Object>

DESCRIPTION:
     ManagedFields maps workflow-id and version to the set of fields that are managed by that workflow. This is mostly for internal housekeeping, and users typically shouldn't need to set or understand this field. A workflow can be the user's name, a controller's name, or the name of a specific apply path like "ci-cd". The set of fields is always in the version that the workflow used when modifying the object.

FIELDS:
   fieldsV1     <map[string]>
     FieldsV1 holds the first JSON version format as described in the "FieldsV1" type.

   manager      <string>
     Manager is an identifier of the workflow managing these fields.
```
To put it simply, `fieldsV1` holds the format, and `manager` holds the object that manages the format.  In the case of the authentication operator, that means that the fields listed in `managedFields[0]` are managed by the `cluster-version-operator` and the fields in `managedFields[1]` are managed by the `authentication-operator`.

Let's continue with our YAML analysis with the `relatedObjects` section. What you see here is a list of resources which may or may not be custom resources, along with the API group they belong to, and their name.  All of the items on this list are instantiated objects on the cluster so we can query any of them like a normal resource.
```
  relatedObjects:
  - group: operator.openshift.io
    name: cluster
    resource: authentications
  - group: config.openshift.io
    name: cluster
    resource: authentications
  - group: config.openshift.io
    name: cluster
    resource: infrastructures
  - group: config.openshift.io
    name: cluster
    resource: oauths
  - group: route.openshift.io
    name: oauth-openshift
    namespace: openshift-authentication
    resource: routes
  - group: ""
    name: oauth-openshift
    namespace: openshift-authentication
    resource: services
  - group: ""
    name: openshift-config
    resource: namespaces
  - group: ""
    name: openshift-config-managed
    resource: namespaces
  - group: ""
    name: openshift-authentication
    resource: namespaces
  - group: ""
    name: openshift-authentication-operator
    resource: namespaces
  - group: ""
    name: openshift-ingress
    resource: namespaces
```
For an example, let's confirm that the authentication operator pod is running, and that there are two authentication instances running (as noted in `relatedObjects[0]` and `relatedObjects[1]`).
```
$ oc get po -n openshift-authentication-operator
NAME                                      READY   STATUS    RESTARTS   AGE
authentication-operator-6d74f68f8-6qz92   1/1     Running   1          3d2h

$ oc get authentications
NAME      AGE
cluster   3d3h
```
That's interesting...there's only one `authentications` custom resource.  Why are there two listed in the `relatedObjects` array for our authentication Operator? If you're readying closely, you may have caught that each instance of `authentications` is in a different API group so when we run the above command, we're only getting it from one API group, not both. If we run the above command and include the group, we'll be able to see two disctint resources.
```
$ oc get authentications.operator.openshift.io -o name
authentication.operator.openshift.io/cluster

$ oc get authentications.config.openshift.io -o name
authentication.config.openshift.io/cluster
```
One last note on this topic; you'll want to pay special attention to entries in `relatedObjects` that have a `namespace` field as that will help direct you to where that resource lives in the event that you need to do some troubleshooting.  For example, if I'm having trouble with authentication, I know that the authentication Operator has a route in the `openshift-authentication` namespace which fronts a service for two pods.
```
$ oc get route,service,pod -n openshift-authentication -o name
route.route.openshift.io/oauth-openshift
service/oauth-openshift
pod/oauth-openshift-6d59b47f7-c4lsj
pod/oauth-openshift-6d59b47f7-pt82j
```
Before we move on to a non-`ClusterOperator`, let's take a moment to recap what we've covered so far. We have...
* Picked a `ClusterOperator` to interrogate.
* Run a command to get basic Operator info like it's version and state.
* Learned how to identify pieces of a `ClusterOperator` and what objects manage each piece.
* Learned how to find objects related to an Operator.
* Learned that multiple resources of the same type and name can exist under different API groups.

Next, we'll follow an abridged version of the above process to get the details for a non-`ClusterOperator` on the cluster.

## Leveraging Go templates for a hollistic view
* Template for single operator view
* Template for all operators
* Templates for special cases

