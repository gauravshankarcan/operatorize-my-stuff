# Enabling self-serve the operator way.
This article presents a straightforward approach for utilizing operator-SDK to transform a collection of Yamls into a Custom resource. It outlines the steps for creating a distinctive resource named MyCustomResource that can be employed to deploy a predefined set of resources, such as a particular policy, governance, or deployment.

The primary aim of this article is to demonstrate how operators can serve as governance tools in a multi-tenant setting. By delegating access controls to tenant teams, platform administrators can enable self-service capabilities within their organizations without jeopardizing coexistence among tenants.

### Prerequisites
1. Understanding the basics of kubernetes and you are the admin of the cluster where you are deploying.
2. Understanding the basics of Ansible

### Scenario:  

For the purposes of this article, let's create a mock scenario or a problem statement we are trying to solve.

#### Request:
Tenants requests the ability to create /delete a namespace in a multi-tenant environment.

#### Problems in answering this request:
- There is a single resource type ```kind: Namespace``` providing access to this (create/ delete) will potentially grant access to delete namespaces belonging to another tenant in the same cluster.
- Tenants can create namespaces without restrictions like not mapping to resource quotas / LDAP bindings to namespace etc 

Now let's explore on how to can solve this problem and make this scenario a self-serve capability for tenants using operators.  

## Let's get started
#### Step1: Installing the dependent libraries

The dependencies required in your PATH variables are

1) Kubectl
2) podman or docker
3) operator_sdk from [here](https://sdk.operatorframework.io/docs/installation/#install-from-github-release)

#### Step2: Environment Setup
 
 login to openshift 

#### Step3: