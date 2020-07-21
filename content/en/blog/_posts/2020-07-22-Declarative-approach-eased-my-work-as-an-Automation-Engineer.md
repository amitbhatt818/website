---
layout: blog
title: Declarative-approach-eased-my-work-as-an-Automation-Engineer
date: 2020-07-22
slug: Declarative-approach-eased-my-work-as-an-Automation-Engineer
---

**Authors:** [Amit Bhatt](https://twitter.com/amitbhatt818) [(MayaData)](https://twitter.com/MayaData) [Core Contributor @ [LitmusChaos](https://github.com/litmuschaos/litmus), [OpenEBS](https://github.com/openEBS)]


I have been using **Ansible** for a while to automate everything related to Kubernetes. One of the irritants for me was the mix of bash, awk, sed, loops, conditions along with Ansible YAML to get my job automated. This is when I decided to invest some time to learn more about the pure declarative intents that Kubernetes operator pattern offer. This post is a summary of my experience of this investment.

### **So what is this declarative approach?**

This definitely is not something that has its origins within Kubernetes. In-fact if you have written any Jenkinsfile (i.e. groovy scripts) or Terraform or used YAMLs to express configurations of an application then you have already experienced the declarative nuances of programming. However, being declarative the Kubernetes way needs to be grasped with a pinch of salt. Kubernetes has been advocating the use of custom resources and backing controllers to let teams handle their infrastructure. One can think of html & backing javascript as an analogy where we are often advised to have clear separation of responsibilities & avoid mixing user interface styling with logic. If done right, html defines the look and feel of the web page while javascript handles the behavioral part. Alas few care about the advice and the best practices. We often find loads of javascript embedded inside the html specifications. The same can be said about other declarative approaches where logic finds its way into the declarations. Kubernetes on the other hand advocates that these declarations remain pure. Hence one does not find templating or loops inside Pod specifications.
 
In the world of Kubernetes, custom resources are used to express the declarative intents that are free from side effects (read logic). It does not end here. These specifications define the desired state and corresponding controllers have the responsibility to make this desired state a reality. Since we are not here to dig deep into Kubernetes control loops, let's get straight to the point. All we need when dealing with Kubernetes is its CLI and a bunch of YAMLs to convey our intentions to the Kubernetes cluster.


### **Introduction to d-operators**

I started experimenting with  [d-operators](https://github.com/mayadata-io/d-operators) to define declarative intents that look like imperative workflows to manage Kubernetes resources. On the whole, I wanted to avoid using kubectl and other command based invocations inside my YAMLs. D-operators make use of the [metacontroller](https://github.com/AmitKumarDas/metac) SDK under the hood. My intention is to enable yaml authors to create, delete, update, assert, patch, clone, schedule, etc. one or more Kubernetes resources (native as well as custom) using a yaml file. This yaml file is in turn governed by a backing Kubernetes controller that deals with reconciliation and other Kubernetes internals.
 
Needless to say, the project tries its best to follow a pure intent based approach to define specifications instead of having to deal with yamls that are cluttered with scripts, kubectl, loops, conditions, templating and so on.


### **Show me the ~~code~~ YAML**

```
apiVersion: dope.metacontroller.io/v1
kind: Recipe
metadata:
  name: crud-ops-on-pod
  namespace: d-testing
  labels:
    d-testing.dope.metacontroller.io/enabled: "true"
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
  - name: delete-the-pod       ### -----> This deletes the Pod
    delete: 
      state: 
        kind: Pod
        apiVersion: v1
        metadata:
          name: my-pod
          namespace: my-ns
  - name: delete-the-namespace  ### ----> This deletes the Namespace
    delete: 
      state: 
        kind: Namespace
        apiVersion: v1
        metadata:
          name: my-ns

  - name: assert-presence-of-pod ### ----> Check presence of Pod
    assert: 
      state: 
        kind: Pod
        apiVersion: v1
        metadata:
          name: nginx
          namespace: test

---
```

### **Result**

You can see the result of the above yaml in the description of the recipe itself.

```
Name:         crud-ops-on-pod
Namespace:    d-testing
Labels:       d-testing.dope.metacontroller.io/enabled=true
              job.dope.metacontroller.io/phase=Completed
Annotations:  <none>
API Version:  dope.metacontroller.io/v1
Kind:         Recipe
Metadata:
  Creation Timestamp:  2020-07-21T18:59:09Z
  Generation:          1
  Resource Version:    33101724
  Self Link:           /apis/metacontroller.app/v1/namespaces/d-testing/jobs/crud-ops-on-pod
  UID:                 2eb8f663-15cd-4836-bfd5-14b3ca4aefd3
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
    Assert:
      State:
        API Version:  v1
        Kind:         Pod
        Metadata:
          Name:       nginx
          Namespace:  test
    Name:             assert-presence-of-pod
Status:
  Failed Task Count:  0
  Message:            
  Phase:              Completed
  Reason:             
  Task Count:         5
  Task List Status:
    Apply - A - Namespace:
      Message:  Create resource  my-ns: GVK /v1, Kind=Namespace
      Phase:    Passed
      Step:     1
    Assert - Presence - Of - Pod:
      Message:  StateCheckEquals: Resource test nginx: GVK /v1, Kind=Pod: TaskName assert-presence-of-pod
      Phase:    Passed
      Step:     5
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
      Step:      7
    Delete - The - Namespace:
      Message:  Delete: Resource  my-ns: GVK /v1, Kind=Namespace
      Phase:    Passed
      Step:     4
    Delete - The - Pod:
      Message:  Delete: Resource my-ns my-pod: GVK /v1, Kind=Pod
      Phase:    Passed
      Step:     3
    Job - Elapsed - Time:
      Elapsed Time In Seconds:  0.239145302
      Internal:                 true
      Phase:                    Passed
      Step:                     6
Events:                         <none>

```
### **Conclution**

In this example, we saw how we can convert a bunch of Kubernetes operations into a declarative specification. We also were able to avoid kubectl and other imperative logic inside our yaml. While there is more to what can be achieved via d-operators this article shows an approach to re-imagine handling infrastructure using Kubernetes. In other words, it pays to think like Kubernetes (read pure declarations) when operating on Kubernetes.