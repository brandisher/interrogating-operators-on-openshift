# Interrogating Operators on Openshift
As an OpenShift Technical Account Manager, helping Red Hat customers migrate from OpenShift 3 to OpenShift 4 has become a primary job function.  There are new terms, new objects, and generally an all around more automated experience.  The key part of this automation is the "Operator" and if you're reading this, we'll assume that you have a passing knowledge of Operators.  If you don't, a good primer on "Operators" can be found [here](https://www.redhat.com/en/topics/containers/what-is-a-kubernetes-operator).

If you're following along on your own cluster, you'll need a user with `cluster-admin` permissions and `oc` in your `$PATH`.  We'll also use Go templates so if you're not familiar with those, a self-led workshop is available [here](https://github.com/brandisher/openshift-and-gotemplates-workshop).  One last thing to note; we're using OpenShift 4.5 as the reference so there may be minor variations depending on your OpenShift version and installation method.  Now that we have the basics covered, let's get started!

## Finding all of the Operators
* List cluster operators
* List all operators

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

