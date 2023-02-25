---
layout: post
title:  "GitOps Using Red Hat Advanced Cluster Management, Policy Generator and Kustomize"
date:   2023-01-30 16:50:00 +0100
toc: true
categories: [OpenShift,GitOps]
tags: [OpenShift,GitOps,RHACM,kustomize,policygenerator]
---

## Introduction

In this article, I'm just going to share one of the possible implementations for **GitOps** using [Red Hat Advanced Cluster Management for Kubernetes](https://www.redhat.com/en/technologies/management/advanced-cluster-management) (**RHACM**).
The purpose is to highlight some issues that could arise while trying to use both [Kustomize](https://kustomize.io/) and the [Policy Generator](https://github.com/stolostron/policy-generator-plugin) at the same time and to provide a possible mitigation.

**DISCLAIMER**: This is just a means to show a possible GitOps directory hierarchy concept while using a combination of Kustomize and Policy Generator.  
So, using it in a production environment with out proper fitting and testing is discouraged and at your own risk.

![RHACM GitOps](https://raw.githubusercontent.com/redfrax/post-gitops-rhacm-kustomize-polgen/main/post_image.png)

### <a name="Hreq"></a>Requirements

To test this configuration, you need a RHACM installation, and to generate manifests locally, you need to have the [Kustomize tool](https://kubectl.docs.kubernetes.io/installation/kustomize/) installed on your local machine.  
To use the policy generator plug-in, this one has to be installed as well, following [this procedure](https://github.com/stolostron/policy-generator-plugin#installation).  
Furthermore, a basic knowledge of the [RHACM Governance Policy Engine](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html-single/governance/index) and **Kustomize overlays** is advisable.

## <a name="Hpolstruct"></a>Structure of a Governance Policy

Let's start by analyzing the basic structure of a very simple governance policy.  
The policy reported below is used to enforce the replicas and thread count configuration of the ingress routers. As you can see, a policy is composed of mainly three parts: a placement rule, a placement binding, and the policy itself, where the configuration template to apply is wrapped.  
Out of 62 lines, only 8 lines are related to the actual cluster configuration.  
It would be great to have a tool that lets us focus just on the payload development without having to worry about all the wrapping.  
Here the Kustomize policy generator plug-in comes in handy.

```yaml
##
## PlacementRule defines target clusters using labels as cluster selectors
## In this case, clusters having the label environment=devel
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
## The policy itself defines the remediation action, the compliance type, the severity
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
*Sample file [here](https://github.com/redfrax/post-gitops-rhacm-kustomize-polgen/blob/main/sample-standalone-policy/pol-ingr-router-devel.yaml)*

## <a name="Hpolgen"></a>Using the Policy Generator Plug-In
By using the policy generator plug-in for Kustomize, you can focus on the configuration manifests.  
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
After having defined the configuration manifests, you just have to describe the specification of the needed policy using the *PolicyGenerator* configuration yaml, as reported in the following example:
```yaml
# File policy-generator-config.yaml
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  name: test-devel-pol-generator
## Policies default specification values
policyDefaults:
  # Namespace where the policies are going to be created in
  namespace: policy-test
  # Default remediation action for the policies
  remediationAction: enforce
  placement:
    # Default placement rule with label definition for proper cluster selection
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
Inside the base and overlay directories, the file *kustomization.yaml* with a reference to the *PolicyGenerator* configuration must exist:
```yaml
# File kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

generators: 
  - policy-generator-config.yaml
```
Following is an example of a base directory to test the generation of a policy:
```bash
tree
.
├── sample-policy-generator-test
    ├── ingress-router-conf-payload.yaml
    ├── kustomization.yaml
    └── policy-generator-config.yaml
```
*Sample files [here](https://github.com/redfrax/post-gitops-rhacm-kustomize-polgen/tree/main/sample-policy-generator-test)*

If the [requirements](#Hreq) are met, by executing the command
```bash
kustomize build --enable-alpha-plugins <your-path-to-dir>/sample-policy-generator-test
```
the same governance policy structure that we analyzed [before](#Hpolstruct), should have been created.

## Bringing Kustomize overlays into the game (here comes the issue)
In real-world use cases, we're going to have several environments and cluster types to manage. So we have the need to leverage the use of kustomize overlays to customize the configuration manifests depending on which cluster type or environments we are targeting.
Let's customize our ingress controller configuration in the case of a production cluster, raising replicas to 6 and the thread count for the single router to 8. 

To do that we try to use a classic base/overlays Kustomize directories structure organized as follow:
```bash
tree
.
├── bases
│   ├── ingress-router-conf-payload.yaml
│   ├── kustomization.yaml
│   └── policy-generator-config.yaml
└── overlays
    ├── devel
    │   ├── ingress-router-conf-payload.yaml
    │   └── kustomization.yaml
    └── prod
        ├── ingress-router-conf-payload.yaml
        └── kustomization.yaml
```
*Sample files [here](https://github.com/redfrax/post-gitops-rhacm-kustomize-polgen/tree/main/sample-base-overlays)*

Inside the "*bases*" directory, we have a generalized version of the policy generator configuration we have seen [before](#Hpolgen).  
Then we have the overlay directories for development and production environments; in both cases, we added the reference to the "*bases*" directory, an environment specific name suffix, and a file to patch the policy for each environment.
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
# Reference to bases dir
resources: 
  - ../../bases
# Suffix to add to metadata names
nameSuffix: -prod
# File used for patching the policy
patchesStrategicMerge:
  - ingress-router-conf-payload.yaml
```
**BUT**, if we go to check the patching file for the overlay, we'll get a bad surprise
```yaml
# Policy patching
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: pol-ingr-router
  namespace: policy-test
...
...
            kind: IngressController
            metadata:
              name: default
              namespace: openshift-ingress-operator
            spec:
              replicas: 6
              tuningOptions:
                threadCount: 8
...
...
---
# Placement rule patching
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: placement-test
  namespace: policy-test
spec:
  clusterSelector:
    matchExpressions:
    - key: environment
      operator: In
      values:
      - prod
---
# Placement binding patching
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-pol-ingr-router
  namespace: policy-test
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placement-test-prod
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: pol-ingr-router-prod
```
As you can see, patching **ALL** the policy parts has been needed due to a couple of reasons:
- The "*nameSuffix*" directive adds the suffix only to the *name* field under the *metadata*; this changes the objects names, but does **NOT** change their references inside the placement binding, resulting in a broken governance policy for RHACM.
- Policy creation takes effect before the patching step, so you need to patch the resulting policy template and not the original payload manifest for the ingress controller. 
- If you had patched the original payload manifest, Kustomize would not have managed to match the customized resource, and you would have gotten the following error:
```bash
Error: no matches for Id IngressController.v1.operator.openshift.io/default.openshift-ingress-operator; failed to find unique target for patch IngressController.v1.operator.openshift.io/default.openshift-ingress-operator
```
**As a result, using this kind of strategy forces us to take a step back and work again with the wrapping part of the policy instead of just paying attention to the configuration payload. It's prone to error, hard to maintain, hard to automate, and definitely not advisable**.

## The Two-Stage Approach
If you want to keep focusing just on the configuration manifests, one of the possible solutions is to use a two-stage approach, such as the one implemented by the following directory tree:
```bash
tree
.
├── bases
│   ├── ingress-router-conf-payload.yaml
│   └── kustomization.yaml
├── overlays
│   ├── devel
│   │   └── kustomization.yaml
│   └── prod
│       ├── ingress-router-conf-payload.yaml
│       └── kustomization.yaml
├── policies-generators
│   ├── devel
│   │   ├── customized-config-manifest.yaml
│   │   ├── kustomization.yaml
│   │   └── policy-generator-config.yaml
│   └── prod
│       ├── customized-config-manifest.yaml
│       ├── kustomization.yaml
│       └── policy-generator-config.yaml
└── simple-manifest-refreshing-script.sh
```
*Sample files [here](https://github.com/redfrax/post-gitops-rhacm-kustomize-polgen/tree/main/sample-two-stage)*

The very first two directories (*bases, overlays*) implement only the classic Kustomize bases/overlays logic, but working just with the pure configuration manifests, inside those directories there's nothing at all related to policy generation.
```yaml
## File overlays/prod/ingress-router-conf-payload.yaml
## just patching the production configuration replicas and threadcount
##
 apiVersion: operator.openshift.io/v1
 kind: IngressController
 metadata:
   name: default
   namespace: openshift-ingress-operator
 spec:
   replicas: 6
   tuningOptions:
     threadCount: 8
```
```yaml
## File overlays/prod/kustomization.yaml
## Just patching the default base yaml with the one reported above specific to prod
##
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources: 
  - ../../bases

patchesStrategicMerge:
  - ingress-router-conf-payload.yaml
```
The script *simple-manifest-refreshing-script.sh* represents the automation bridging the two stages; in this particular case, it just refreshes (using Kustomize itself) the environment specific configuration manifests contained inside the directories *policies-generators/devel and policies-generators/prod*. So, after you have developed the needed configuration files inside the *bases* and *overlays* directories, you just have to run this script to refresh the manifests used by the policy generator during the second step.  
Even if the script is very simple in this case, it can become complex at will by accommodating all the features needed by your specific use case. For example, it could take a configuration file to customize the policy specs inside the policy generator file.
With this process, the environment specific files are generated inside the related directories, *policies-generators/devel and policies-generators/prod*, where they are referenced in each environment specific policy generator.
Those directories are going to be the natural targets for RHACM *subscriptions* that will take care of generating the governance policies using the built-in Kustomize plug-in and applying them to the right clusters based on their placement selectors.
```yaml
## File policies-generators/devel/policy-generator-config.yaml
## Defines the specs for development environments and references the proper manifest
##
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  name: test-devel-pol-generator
#policy default specs
policyDefaults:
  namespace: policy-test
  remediationAction: enforce
  placement:
    name: placement-devel-test
    clusterSelectors:
      environment: devel
placementBindingDefaults:
  name: "binding-devel-test"
policies:
    #policy for development environments
  - name: conf-pol-devel
    manifests:
        # reference to the specific development manifest refreshed by the script
      - path: customized-config-manifest.yaml
```
## Conclusion
We have seen a couple of ways to implement GitOps using RHACM and the *Kustomize Policy Generator*, in particular, we found that using a single-step approach leads to some collaterals that can be avoided by using a more advisable two-stage process, which lets us:
- focusing just on the configuration payload during the development
- having a less error-prone development process
- keeping the process easier to automate and scale with the needed granularity
- having a more maintainable directory hierarchy
