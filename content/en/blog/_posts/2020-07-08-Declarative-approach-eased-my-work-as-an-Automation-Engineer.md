---
layout: blog
title: Declarative-approach-eased-my-work-as-an-Automation-Engineer
date: 2020-07-08
slug: Declarative-approach-eased-my-work-as-an-Automation-Engineer
---

**Authors:** [Amit Bhatt](https://twitter.com/amitbhatt818) [(MayaData)](https://twitter.com/MayaData) [Core Contributer @ [LitmusChaos](https://github.com/litmuschaos/litmus), [OpenEBS](https://github.com/openEBS)]


I have been using **Ansible** for a while to automate everything related to Kubernetes. One of the irritants for me was the mix of bash, awk, sed, loops, conditions, etc along with Ansible YAML to get my job automated. This is when I decided to invest some time to learn more about the pure declarative specifications that Kubernetes operator pattern has been offering. This post is more like a summary of my experience during the exercise.

### **So what is this declarative approach?**

This definitely is not something that has its origins with Kubernetes. Infact if you have written any Jenkinsfile (i.e. groovy scripts) or Terraform or used YAMLs to express configurations of an application then you have already experienced the declarative way. However, being declarative the Kubernetes way needs to be grasped with a pinch of salt. Kubernetes has been advocating the use of custom resources and backing controllers to let teams handle the infrastructure. One can think of html & backing javascript as an analogy where we are often advised to have clear separation of responsibilities & avoid mix and match of user interface style with logic. If done right, html defines the look and feel of the web page while javascript handles the behavioral part. Alas few care about the advice and the best practices. We often find loads of javascript embedded inside the html specifications. The same can be said about other declarative approaches where logic finds its way into the declarations. Kubernetes on the other hand ensures these declarations remain pure. Hence one does not find templating or loops inside Pod specifications.

In the world of Kubernetes, custom resources are used to express the declarative intentions that are free of side effects (read logic). It does not end here. These specifications define the desired state and backing controller handles the responsibility to make this desired state a reality. Since we are not here to dig deep into Kubernetes control loops, let's get straight to the point. All we need when dealing with Kubernetes is its CLI and a bunch of YAMLs that convey our intentions to the Kubernetes cluster.


### **Introduction to d-operators**

[D-operators](https://github.com/mayadata-io/d-operators) define various declarative patterns to write Kubernetes controllers. This uses the [metacontroller](https://github.com/AmitKumarDas/metac) SDK under the hood. Users can create, delete, update, assert, patch, clone, & schedule one or more Kubernetes resources (native as well as custom) using a yaml file. D-operators expose a bunch of Kubernetes custom resources that provide the building blocks to implement a higher order controller.

It follows a pure intent based approach to write specifications instead of having to deal with yamls that are cluttered with scripts, kubectl, loops, conditions, templating and so on.


### **Show me the ~~code~~ YAML**

```
apiVersion: metacontroller.app/v1
# Do not confuse kind: Job with the native Job 
# that Kubernetes offers out of box.
#
# This will be changed to something else very shortly
kind: Job
metadata:
  name: crud-ops-on-pod
  namespace: d-testing
  labels:
    d-testing.metacontroller.app/enabled: "true"
spec:
  tasks:                       ### -----> Add multiple operations via tasks
  - name: apply-a-namespace    ### ----> This applies a Namespace
    apply: 
      state: 
        kind: Namespace
        apiVersion: v1
        metadata:
          name: my-ns
  - name: create-a-pod         ### -----> This creates a Pod
    create: 
      state: 
        kind: Pod
        apiVersion: v1
        metadata:
          name: my-pod
          namespace: my-ns
        spec:
          containers:
          - name: web
            image: nginx
  - name: delete-the-pod      ### -----> This deletes the Pod
    delete: 
      state: 
        kind: Pod
        apiVersion: v1
        metadata:
          name: my-pod
          namespace: my-ns
  - name: delete-the-namespace ### ----> This deletes the Namespace
    delete: 
      state: 
        kind: Namespace
        apiVersion: v1
        metadata:
          name: my-ns

---
```

### **Result**

You can see the result of the above yaml in the description of the job itself.

```
Name:         crud-ops-on-pod
Namespace:    d-testing
Labels:       d-testing.metacontroller.app/enabled=true
              job.dope.metacontroller.io/phase=Completed
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"metacontroller.app/v1","kind":"Job","metadata":{"annotations":{},"labels":{"d-testing.metacontroller.app/enabled":"true"},"...
API Version:  metacontroller.app/v1
Kind:         Job
Metadata:
  Creation Timestamp:  2020-06-07T07:14:41Z
  Generation:          1
  Resource Version:    18111945
  Self Link:           /apis/metacontroller.app/v1/namespaces/d-testing/jobs/crud-ops-on-pod
  UID:                 b4b3b2ce-8ebd-47f0-b438-aa1166742dbf
Spec:
  Tasks:
    Apply:
      State:
        API Version:  v1
        Kind:         Namespace
        Metadata:
          Name:  my-ns
    Name:        apply-a-namespace
    Create:
      State:
        API Version:  v1
        Kind:         Pod
        Metadata:
          Name:       my-pod
          Namespace:  my-ns
        Spec:
          Containers:
            Image:  nginx
            Name:   web
    Name:           create-a-pod
    Delete:
      State:
        API Version:  v1
        Kind:         Pod
        Metadata:
          Name:       my-pod
          Namespace:  my-ns
    Name:             delete-the-pod
    Delete:
      State:
        API Version:  v1
        Kind:         Namespace
        Metadata:
          Name:  my-ns
    Name:        delete-the-namespace
Status:
  Failed Task Count:  0
  Message:            
  Phase:              Completed
  Reason:             
  Task Count:         4
  Task List Status:
    Apply - A - Namespace:
      Message:  Create resource  my-ns: GVK /v1, Kind=Namespace
      Phase:    Passed
      Step:     1
    Create - A - Pod:
      Message:  Create action: Resource my-ns my-pod: GVK /v1, Kind=Pod: TaskName create-a-pod
      Phase:    Passed
      Step:     2
    Crud - Ops - On - Pod - Lock:
      Internal:  true
      Message:   Create: Lock d-testing crud-ops-on-pod-lock: GVK /v1, Kind=ConfigMap
      Phase:     Passed
      Step:      0
    Crud - Ops - On - Pod - Unlock:
      Internal:  true
      Message:   Locked forever
      Phase:     Passed
      Step:      6
    Delete - The - Namespace:
      Message:  Delete: Resource  my-ns: GVK /v1, Kind=Namespace
      Phase:    Passed
      Step:     4
    Delete - The - Pod:
      Message:  Delete: Resource my-ns my-pod: GVK /v1, Kind=Pod
      Phase:    Passed
      Step:     3
    Job - Elapsed - Time:
      Elapsed Time In Seconds:  1.048863253
      Internal:                 true
      Phase:                    Passed
      Step:                     5
Events:                         <none>
```
### **Conclution**

In this example, we saw how we can convert a bunch of Kubernetes operations into a declarative specification. We also were able to avoid kubectl and other imperative logic inside our yaml. While there is more to what can be achieved via d-operators this article shows an approach to re-imagine handling infrastructure using Kubernetes. In other words, it pays to think like Kubernetes (read pure declarations) when operating on Kubernetes.