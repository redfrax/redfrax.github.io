---
layout: post
title:  "*DRAFT* GitOps Using RHACM, Policy Generator and Kustomize"
date:   2023-01-30 16:50:00 +0100
toc: true
categories: [OpenShift,GitOps]
tags: [OpenShift,GitOps,RHACM,kustomize,policygenerator]
---

## Introduction

In this article I'm just going to share one of the possible implementation for **GitOps** using **RHACM**.
The purpose is to highlight some issues that could arise while trying to use both [Kustomize](https://kustomize.io/) and the [Policy Generator](https://github.com/stolostron/policy-generator-plugin) at the same time, and to provide a possible mitigation.

**DISCLAIMER**: This is just a mean to show a possible GitOps directory hierarchy concept while using a combination of Kustomize and Policy Generator.  
So, using it in a production environment with out proper fitting and test is discouraged and at your own risk.

![RHACM GitOps](https://tadviser.com/images/thumb/1/16/Red_Hat_Advanced_Cluster_Management_for_Kubernetes_2.3.png/840px-Red_Hat_Advanced_Cluster_Management_for_Kubernetes_2.3.png)

### <a name="Hreq"></a>Requirements

To test this configuration you need a RHACM Installation and to generate manifests locally you need to have the [kustomize tool](https://kubectl.docs.kubernetes.io/installation/kustomize/) installed on your local machine.  
Furthermore a basic knowledge of [RHACM Governance Policy engine](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html-single/governance/index) is advisable.

## <a name="Hpolstruct"></a>Structure of a goverance policy

Let's start by analyze the basic structure of a very simple governance policy:
```yaml
##
## PlacementRule defines target clusters using labels as cluster selector
## In this case clusters having the label environment=devel
##
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: placement-devel-test
  namespace: policy-test
spec:
  clusterConditions:
  - status: "True"
    type: ManagedClusterConditionAvailable
  clusterSelector:
    matchExpressions:
    - key: environment
      operator: In
      values:
      - devel
---
##
## PlacementBinding binds the above placement rule with the policy definition
##
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-pol-ingr-router-devel
  namespace: policy-test
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placement-devel-test
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: pol-ingr-router-devel
---
##
## The policy itself, defines the remediation action, the compliance type, the severity
## and, of course, the configuration template needs to be applied
##
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  annotations:
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
    policy.open-cluster-management.io/standards: NIST SP 800-53
  name: pol-ingr-router-devel
  namespace: policy-test
spec:
  disabled: false
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: pol-ingr-router-devel
      spec:
        object-templates:
        - complianceType: musthave
          objectDefinition:
            ##
            ## Actual configuration applied to the target cluster, the real payload
            ##
            apiVersion: operator.openshift.io/v1
            kind: IngressController
            metadata:
              name: default
              namespace: openshift-ingress-operator
            spec:
              replicas: 3
              tuningOptions:
                threadCount: 4
            ##
            ##  End of payload
            ##
        remediationAction: enforce
        severity: low
```
The one above is a simple governance policy used to enforce the replicas and thread count configuration of the ingress routers. As you can see, mainly, a policy is composed by three parts, a placement rule, a placement binding and the policy itself where the configuration template to apply is wrapped.  
Out of 62 lines, only 8 lines are related to the actual cluster configuration.  
It would be great to have a tool that let us focus just on the payload development with out having to care about all the wrapping part.  
Here the kustomize policy generator plug-in comes helping us.

## Using the Policy Generator Plug-In
By using the policy generator plug-in for kustomize you can focus on the configuration manifests.  
In this case, just the ingress controller manifest is needed:
```yaml
# File ingress-router-conf-payload.yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  replicas: 3
  tuningOptions:
    threadCount: 4
```
After having defined the configuration manifests, you just have to describe the spec of the needed policy using the *PolicyGenerator* configuration yaml as reported in the following example:
```yaml
# File policy-generator-config.yaml
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  name: test-devel-pol-generator
## Policies default spec values
policyDefaults:
  # Namespace where the policies are going to be created in
  namespace: policy-test
  # Default remediation action for the policies
  remediationAction: enforce
  placement:
    # Default placement rule with label definition for proper clusters selection
    name: placement-devel-test
    clusterSelectors:
      environment: devel
placementBindingDefaults:
  name: "binding-devel-test"
# List of policies to be generated
policies:
    # name of the policy
  - name: pol-ingr-router-devel
    manifests:
      # reference to the manifests used as configuration templates 
      - path: ingress-router-conf-payload.yaml
```
Inside the base or overlay directory the file *kustomization.yaml* with a reference to the *PolicyGenerator* configuration, must exists:
```yaml
# File kustomization.yaml
generators: 
  - policy-generator-config.yaml
```
Following an example of a base directory to test the policy generation:
```bash
tree
.
├── sample-policy-generator-test
    ├── ingress-router-conf-payload.yaml
    ├── kustomization.yaml
    └── policy-generator-config.yaml
```
If the [requirements](#Hreq)