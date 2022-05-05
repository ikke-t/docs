---
layout: default
title: LDAP Authentication
grand_parent: Patterns
parent: Multicloud GitOps
nav_order: 2
---

# Controlling cluster config and secrets, LDAP auth example

{: .no_toc }

## Table of contents

{: .no_toc .text-delta }

1. TOC
{:toc}


## Allow ACM to configure managed clusters to do LDAP authentication for users

This example demonstrates how managed clusters can be configured in GitOps
way. LDAP authentication is an example of common features that need to be
set to each cluster. It demonstrates how to create and maintain configuration
changes, configmaps and secrets.

## Prerequisites

As a prerequisite, you should of course have the identity management set up.
This example uses Red Hat IDM product. It was setup by ansible, configuration
is in
[cool-lab repository](https://github.com/RedHatNordicsSA/cool-lab/blob/main/setup-idm.yml)
for your reference. Users and groups are also listed in
[ansible config](https://github.com/RedHatNordicsSA/cool-lab/blob/main/group_vars/ipaservers/users.yml).

You could use this setup for any other LDAP authentication source as well by
modifying the LDAP connection parameters slightly. Like MS AD.

## Configuration items

Setting up OpenShift Oauth to work with LDAP is
[described in OpenShift docs](https://docs.openshift.com/container-platform/4.10/authentication/identity_providers/configuring-ldap-identity-provider.html).
We also setup
[LDAP group syncronization](https://docs.openshift.com/container-platform/4.10/authentication/ldap-syncing.html)
to be able to control user groups.

For the job we need to setup:

* New namespace for ldap-sync
* Drop LDAP bind secret to ldap-sync and openshift-config namespaces
* Set configurations for LDAP connection parameters
* Store LDAP bind secret to Vault, and make ACM push and maintain it in all
  clusters in both the above namespaces
* Access configurations for secrets
* CronJob to sync groups regularly

This document will not describe how LDAP auth works, see the above links for
what needs to get done. This only describes how it's automized.

## Deploying OpenShift configurations

In this picture we see the steps the automatin does:

![ldap-sync](images/multicloud-gitops/ldap-sync.png "ldap-sync logical diagram")

First we need to put LDAP config items into files:

* [values-global.yaml](https://github.com/RedHatNordicsSA/multicloud-gitops/tree/cool-lab/values-global.yaml)
* [values-hub.yaml](https://github.com/RedHatNordicsSA/multicloud-gitops/tree/cool-lab/values-hub.yaml)
* [values-region-one.yaml](https://github.com/RedHatNordicsSA/multicloud-gitops/tree/cool-lab/values-region-one.yaml)

This defines the namespaces created to OpenShift:

```yaml
  namespaces:
  - ldap-sync
```yaml

This kinda parts define the projects created to argocd:

```yaml
  projects:
  - ldap-sync
```

Here we define the application for ACM:

```yaml
  applications:
  - name: ldap-sync
    namespace: ldap-sync
    project: ldap-sync
    path: charts/all/ldap-sync
```

The above path holds the
[helm charts](https://github.com/RedHatNordicsSA/multicloud-gitops/tree/cool-lab/charts/all/ldap-sync)
used to create all the configs for LDAP.

## LDAP bind Secret

We need to also tell OpenShift what is the password for LDAP authentication.
Such a secret can't be put into git, but is rather put into Hashicorp Vault
used in the demo. We define it in values-secret.yaml file like:

```yaml
secrets:
  ldap:
    bindPassword: veryverysecretldappassword
```

Later on
[kubernetes external secrets operator](https://github.com/external-secrets/kubernetes-external-secrets)
grabs it from vault, and puts it into ldap-secret secret in two of the projects.
This is how we tell ACM which secret to use, see line in
[ldap-sync-secrets.yml](https://github.com/RedHatNordicsSA/multicloud-gitops/tree/cool-lab/charts/all/ldap-sync/templates/ldap-sync-secrets.yml)
file:

```yaml
{% raw %}
{{ if .Values.clusterGroup.isHubCluster }}
---
apiVersion: "external-secrets.io/v1alpha1"
kind: ExternalSecret
metadata:
  name: ldap-secret
  namespace: ldap-sync
spec:
  refreshInterval: 15s
  secretStoreRef:
    name: {{ .Values.secretStore.name }}
    kind: {{ .Values.secretStore.kind }}
  target:
    name: ldap-secret
    template:
      type: Opaque
  dataFrom:
    - key: {{ .Values.ldapBindSecret.key }}
{{ end }}
{% endraw %}
```

Kubernetes operator sets the secret into ACM cluster's ldap-sync project with
name ldap-secret. That then is picked up by ACM and delivered to required
clusters via
[policy in this file](https://github.com/RedHatNordicsSA/multicloud-gitops/tree/cool-lab/charts/all/ldap-sync/templates/).


```yaml
{% raw %}
bindPassword: '{{ `{{hub (lookup "v1" "Secret" "ldap-sync" "ldap-secret").data.bindPassword hub}}` }}'
{% endraw %}
```

## Getting it applied

I assume you have at this stage followed the generic docs to setup the
multicloud-gitops pattern. What's left now to do is to label the clusters
correctly. At this point of time all the configs will go into place by
applying the region-one label to cluster. But while playing around with
external secrets, I left another label to be set to make it copy the secret.
You need to set "ldap-auth=true" label for each cluster too. Let's remove
this part at some stage.
