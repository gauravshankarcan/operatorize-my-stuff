# Enabling self-serve the operator way.
This article presents an approach for utilizing operator-SDK to transform a collection of Yamls into a Custom resource. It outlines the steps for creating a distinctive resource ```CustomResource``` / ```CustomResourceDefinition``` that can be employed to deploy a predefined set of resources, such as a particular policy, governance, or a deployment.

The primary aim of this article is to demonstrate how operators can serve as governance tools in a multi-tenant setting. By delegating access controls to tenant teams, platform administrators can enable self-service capabilities within their organizations without jeopardizing coexistence among tenants.

### Prerequisites
1. You have an understanding the basics of kubernetes and you are the admin of the cluster where you are deploying. ( local machine instance is good )
2. Understanding the basics of ansible
3. docker repo to published compiled images 

### Scenario:  

For the purposes of this article, let's create a mock scenario or a problem statement we are trying to solve.

#### Request / Problem statement we are trying to solve:
Tenants requests the ability to create /delete a namespace in a multi-tenant environment.

#### Problems in answering this request:
- There is a single resource type ```kind: Namespace```, providing access to this (create/ delete) will potentially grant access to delete namespaces belonging to another tenant in the same cluster.
- Tenants can create namespaces without restrictions i.e without mapping to resource quotas / LDAP bindings to namespace etc 

Now let's explore on how to can solve this problem and make this scenario a self-serve capability for tenants using operators.  

## Let's get started
### Step1: Installing the dependent libraries

The dependencies required in your PATH variables are

1) Kubectl
2) podman or docker
3) operator_sdk from [here](https://sdk.operatorframework.io/docs/installation/#install-from-github-release)
   
### Step2: Environment Setup
 
 - login to openshift as kubeadmin ( prefer a local instance )
 - clone a an empty git repo to your local machine ( This is needed to store generated code  ) and cd into the folder cloned .

Assume I have 2 tenants. 
- Tenant 1 is named Acme
- Tenant 2 is named Umbrella
   
In this article, I aim to create a resource ```Kind: AcmeNamespace``` and ```Kind: UmbrellaNamespace```  which can create namespace attach to appropriate custer resource quota and perform other actions . once the permissions are mapped, tenants in Acme cannot interfere with namespaces belonging in Umbrella 

1.  Initiate a operator project
```yaml
operator-sdk init --domain mytenant.com --plugins ansible
```

1. Create a custom resource definition for your resource
```yaml
operator-sdk create api --group tenants --version v1alpha1 --kind AcmeNamespace --generate-role
operator-sdk create api --group tenants --version v1alpha1 --kind UmbrellaNamespace --generate-role
```

1. Open the generated codebase in an IDE 

### Step3: Understanding the generated codebase by the operator_sdk

For the pupose of this article, we will go through a simplistic MVP approach .

Files you need to worry about
- config/crd   --> Contains two CRDS  . one for each type ```kind: AcmeNamespace``` and ```kind UmbrellaNamespace```
  - A CRD is a definition of a custom resource which is tracked by an operator, when the operator receives a resource or yaml instance  belonging to this CRD, it performs an action. A CRD contains the OpenAPI Schema spec of the yaml which defines the sttucture of the yaml.
  > In this article we will use a free form OpenAPI spec, notice key  ```x-kubernetes-preserve-unknown-fields: true``` which defines it can accept any format 
- roles/``api name```  --> This is the ansible role which gets executed when an instance(Custom resource) of the above CRD is encountered


### Step4: Building the Playbook

Under ```roles/acmenamespace/tasks``` create a file caled main.yaml . The main.yaml is the entrypoint for this playbook

```yaml
---
- name: create namespace
  kubernetes.core.k8s:
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: "{{ ansible_operator_meta.name }}""
        labels:
          tenant: acme
```
> add additional ldap bindings as necessary
> 
> add additional scc / limit ranges /labels or any other yamls which you consider default tenant settings



Do the same for Umrella tenants as well but with a different label ```tenant: Umbrella``` with the file ```roles/umbrellanamespace/tasks/main.yaml```

> Note: For the purpose of showcasing that custom tenant specific configuration is possible I dropped in a network policy 

```yaml
---
- name: create namespace
  kubernetes.core.k8s:
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: "{{ ansible_operator_meta.name }}"
        labels:
          tenant: umbrella
          
- name: create default network policy to allow networking between namespaces owned by tenant umbrella.  
  kubernetes.core.k8s:
    definition:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: allow ingress from all namespaces under umbrella tenant
          namespace: "{{ ansible_operator_meta.name }}"
        spec:
          podSelector: null
          ingress:
              - from:
                  - namespaceSelector:
                      matchLabels:
                      tenant: umbrella
```

> add additional ldap bindings as necessary
> 
> add additional scc / limit ranges /labels or any other yamls which you consider default tenant settings

### Step 5: Prepping the cluster to accept these new namespaces to be mapped into resource quotas

create a resource quota mapping to the labels above 

```Shell
oc create clusterquota acme \
     --project-label-selector tenant=acme \
     --hard pods=10 \
     --hard secrets=20

oc create clusterquota umbrella \
     --project-label-selector tenant=umbrella \
     --hard requests.cpu=4

```

Since I use podman in my environment , I need to  replace occurances of docker to podman
```
sed -i 's/docker/podman/g' Makefile
```

Publish the operator you built to a repo

Modify the ```MakeFile``` which is in the root folder .
Search for the the ```IMG``` variable

```Shell
IMG ?= controller:latest

to
## To your in-house repo
IMG ?= docker.io/datatruckerio/operatorize-tenancy:latest 
``` 


Execute the operator to the cluster
```Shell
make podman-build podman-push
make deploy
```

The namespace is already created by the operators to ```<project-name>-system```

> Notes: for the service account created for the operator you need to provide rbac . however to keep the article simple lets add the serviceaccount to cluster admin . You can definately focus a clusterrole / rolebinding to be granular to whats provisioned by the anisble play
>
> So to add cluster-admin ```oc adm policy add-cluster-role-to-user cluster-admin -z operatorize-my-stuff-controller-manager```
>
> Alternatively you can add the rbac in config/rbac folder 


Push your code to the git repo for storage , you can re-use this repo to add more custom resources / upgrade the operator etc , this helps in having 1 operator for several custom resources

### Step6 : Lets go Test

Lets create the the yamls

```yaml
apiVersion: tenants.mytenant.com/v1alpha1
kind: AcmeNamespace
metadata:
  name: acmenamespace-sample
```