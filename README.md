# Kubernetes Credentials

[Homepage](https://manifold.co) |
[Twitter](https://twitter.com/manifoldco) |
[Code of Conduct](./.github/CONDUCT.md) |
[Contribution Guidelines](./.github/CONTRIBUTING.md)

[![Build Status](https://travis-ci.com/manifoldco/kubernetes-credentials.svg?token=SbTMbCYMT5HWVmmTnBoj&branch=master)](https://travis-ci.com/manifoldco/kubernetes-credentials)
[![Go Report Card](https://goreportcard.com/badge/github.com/manifoldco/kubernetes-credentials)](https://goreportcard.com/report/github.com/manifoldco/kubernetes-credentials)
[![License](https://img.shields.io/badge/license-BSD-blue.svg)](./LICENSE)

This package allows you to load [Manifold](https://www.manifold.co/) credentials
into your Kubernetes cluster. These credentials will be stored as a Kubernetes
secrets so you can use them as such in your deployments.

## Usage

We've utilised [Kubernetes' CRD](https://kubernetes.io/docs/concepts/api-extension/custom-resources/) implementation to allow you to define fine
grained control over your credentials. As with our [Terraform provider](https://github.com/manifoldco/terraform-provider-manifold/), we
allow you to filter out projects, resources and specific credentials you want to
load.

### Defining credentials

Defining credentials happens through our Custom Resource Definition. We have
2 ways of defining how you would like to get credentials.

**Note:** The minimum requirement to define a specific credential is it's key.
If you provide a name, this name will be used as a key reference in the k8s
secret. A default value can also be provided. If a default value is provided for
a key that does not exist in the Manifold credentials list, this default value
will be used to populate the credential.

#### Project

You can load multiple credentials in one go for a specific project, you can do
this as follows:

```yaml
apiVersion: manifold.co/v1
kind: Project
metadata:
  name: manifold-terraform-project # required; this will be the name of the secret we'll write to and which you can use to reference
spec:
  project: manifold-terraform # required; project label
  team: manifold # optional; the team to load the credential from
  resources: # optional; load all resources by default, just like with terraform
    - resource: custom-resource1 # this one loads the specific credentials listed below
      credentials:
        - key: TOKEN_ID
    - resource: custom-resource2 # this one loads all credentials for this resource
```

#### Resource

If you only want to get the credentials from a specific resource, you can do
this as follows:

```yaml
apiVersion: manifold.co/v1
kind: Resource
metadata:
  name: manifold-terraform-resource # required; this will be the name of the secret we'll write to and which you can use to reference
spec:
  resource: custom-resource1 # required; resource label
  project: manifold-terraform # optional; project label
  team: manifold # optional; team label
  credentials:
    - key: TOKEN_ID
    - key: TOKEN_SECRET # alias the name to alias-name which we can use later on
      name: alias-name
    - key: NON_EXISTING # set a default value for a non existing credential
      default: "my-default-value"
```

### Referencing the credentials

Once you've set up the controller (see [setting up the controller](#setting-up-the-controller)),
the controller will start looking for the resources defined earlier and write
the values from Manifold to the respective Kubernetes secret. This means that
when a credential changes, the secret will also be updated automatically with
the new value.

By using exsiting Kubernetes secrets, we allow you to use the Manifold
credentials as secrets:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 2
  spec:
    nodeSelector: {}
    containers:
      - name: my-service
        image: manifoldco/my-service:latest
        env:
          - name: CUSTOM_TOKEN_ID
            valueFrom:
              secretKeyRef:
                name: secret-manifold-project
                key: TOKEN_ID
          - name: RESOURCE2_USERNAME
            valueFrom:
              secretKeyRef:
                name: secret-manifold-project
                key: USERNAME
          - name: CUSTOM_TOKEN_SECRET
            valueFrom:
              secretKeyRef:
                name: secret-manifold-resource
                key: alias-name
          - name: NON_EXISTING_TOKEN
            valueFrom:
              secretKeyRef:
                name: secret-manifold-resource
                key: NON_EXISTING
```

## Installation

Installing the Custom Resource Definition and Controller exists out of a few
steps which are listed below.

### Define the CRD in your cluster

To install this Custom Resource into your cluster, make sure that you have
access to your cluster through `kubectl`. Double check that you're connected to
the correct cluster by running `kubectl cluster-info`. You can then run the
following command to install the custom resource:

```
$ go run install.go -kubeconf=$HOME/.kube/config
```

### Setting up the Manifold Auth Token to retrieve the credentials

First, you'll need to create a new Auth Token:

```
$ manifold tokens create
```

Once you have the token, you'll want to create a new Kubernetes Secret:

```
$ kubectl create secret generic manifold-secrets --from-literal=auth_token=<AUTH_TOKEN> --from-literal=team=<MANIFOLD_TEAM>
```

**Note:** The team value is optional. If a team is provided in the controller
(see below), only resources that define this team will be picked up and used
to load the credentials. If no team is defined, this is ignored.

### Setting up the controller

The last thing you'll need to do is set up the controller. The controller takes
care of monitoring your Resource Definitions and populating the correct
Kubernetes Secrets with Manifold Credentials. Without it, nothing will happen.

```
$ kubectl create -f credentials-controller.yml
```

**Note:** You can customise this credentials-controller file. This is a general
purpose ReplicaSet. `K8S_NAMESPACE` and `MANIFOLD_AUTH_TOKEN` are required
environment variables.
