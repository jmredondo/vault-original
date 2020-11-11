# HCL Commerce Vault and Consul Deployment

## Introduction
Vault is used to store commerce environment configuration and secret, and is also used as  certification authority (CA) to issue certificate to commerce servers to allow them communicate with SSL.

This chart deploys Vault and Consul for HCL Commerce V9 as the remote configuration center, which stores environment data and acts as the certification agent to issue certificate to each HCL Commerce app server based on unique service name.  Vault and Consul are required before you deploy HCL Commerce V9 application if you plan to use the `Vault` configuration mode.  It needs to be deployed only once and the service can be available to multiple Commerce environments. After the vault is started, it can run scripts to enable mount the tenant secret path and load commerce environment configuration data as you specified in values.yaml, and also it can enable and configure PKI backend to issue certificate if you enable the `vaultConsul.vaultLoadData` in values.yaml.

## Chart Details
This Helm Chart is only for testing and development purpose. It does not persist data or handle vault root token securely. However, for development and testing purpose, the data stored in the values.yaml can be used as a master copy and re-load the data every time when vault is redeployed or restarted.  Please refer [guide](
https://www.vaultproject.io/guides/operations/vault-ha-consul.html) of Vault to setup cluster to support production environment.

## Prerequisites
This Helm Chart attempts to pull Vault and Consul Docker Image from DockerHub. If you use the default settings,
ensure that your environment can connect to internet. Or you can pull those Docker images to a private Docker Image Repository

Attribute Name |   Docker Image | Usage
------------- | -------------| -------------
vaultConsul.vaultImageTag | docker.io/vault:1.3.3 | Vault Docker Image provides Vault service
vaultConsul.consulImageTag | docker pull docker.io/consul:1.7.1  | Consul Docker Image provides Consul service
test.image |  docker.io/centos:latest | Helm Test command uses Centos Docker Image

> **Note**
vault:1.3.3  / consul:1.7.1 have been tested.  There is no guarantee that other tags for those docker images will work as expected.

## Resources Required
The resource limitation is not defined for Vault-Consul, since it's used as the associate service to support HCL Commerce Version 9 deployment and will not handle high traffic volumes.

## Plan and Configure Values for vault deployment
Before deploying vault, you need to plan how you are going to deploy commerce, and then modify the data accordingly as the data specified in your values yaml file will be loaded to the vault as postStart action. In vault, a tenant name will be used as a secret mount path, this secret path will contain one or more environments, and each environment contains the key-value (KV) pairs. The environment level KVs will be under environment path directly, while auth and live instance specific KVs are under auth or live. E.g
```
	/Demo                                                        # tenant path
		/qa                                                      # env path
			internalDomainName: commerce.svc.cluster.local       # environment level properties
			…
			/auth                                                # auth instance path
				dbHost: myDb.com                                 # auth instance level properties
				…
			/live                                                # live instance path
				dbHost: myLiveDb.com                             # live instance level properties
```

It is strongly recommended to not modify the default [values.yaml](./values.yaml) file for your deployment, but instead copying it to your customized values file, e.g `my-values.yaml` file, to maintain your customized values for your future deployment and upgrade.

### Modify configuration in my-values.yaml file
#### common:
1. Change the tenant if you want to name it differently. Note, if you change the tenant name here, you will also need to change the tenant in commerce helm chart values to match the same name.
1. By default it does not create ingress for vault service. If you want to create an ingress to access vault ui, set `enableIngress` to `true`
1. As part of the vault deployment, it will create a vault token secret in the `commerce` namespace, so that the commerce application can get the vault token from that secret. It requires the commerce namespace existed before you can deploy this vault. If `commerce` namespace has not been created, you can create it now with `kubectl create ns commerce`. If you plan to deploy commerce in other name spaces, you need to create those names spaces now (kubectl create ns <namespace>), and list all of the namespaces in commerceNameSpaces. E.g if I want to deploy 2 commerce environments 'dev' and 'qa' in 'commerce-dev' and 'commerce-qa' name spaces, you would need to:
	1. kubectl create ns commerce-dev
	1. kubectl create ns commerce-qa
	1. Config `commerceNameSpaces` to following:
		```
		commerceNameSpaces: 
			- commerce-dev
			- commerce-qa
		```
#### vaultConsul:
1. Update `consulImageTag` and `vaultImageTag` if you want to test different images
1. Update `vaultToken` to the one you want to use. 
1. If you change the `vaultToken` value, you will need to run `echo -n new_token | base64` and update `vaultTokenBase64` with this value
1. Update the data under `vaultData`. E.g update the db information. See above [Plan and Configure Values for vault deployment](#Plan-and-Configure-Values-for-vault-deployment) for details of the data hierarchy in vault.


## Installing the Chart
It is recommended to deploy vault in a separate namespace, such as `vault` and serve for all commerce environments. If you don't `vault` namespace, you can create it by `kubectl create ns vault`. Following command is to use helm (v3) to deploy this chart in `vault` namespace.
```
$ helm install vault-consul ./hcl-commerce-vaultconsul -f my-values.yaml -n vault
```

Note: `vault-consul` is the release name, and `./hcl-commerce-vaultconsul` is the local path to the chart, and `vault` is the namespace where this chart deploys to.

Once vault is deployed, run "kubectl get pods -n vault" to make sure vault-consul-xxxx has 2/2 in READY column

Also list the secret in your commerce namespace to make sure the secret has been created.
```
$ kubectl get secret vault-token-secret -n commerce
NAME                 TYPE     DATA   AGE
vault-token-secret   Opaque   1      7m44s
```

## Upgrade the deployment
When you need to update vault values, you can update configuration values in vaultData, and then re-deploy vault using a command similar to following
```
helm upgrade vault-consul ./hcl-commerce-vaultconsul -f my-values.yaml -n vault
```

## Verifying the Chart
use the helm test command: `helm test vault-consul --cleanup=true`

## Uninstalling the Chart
```
$ helm delete vault-consul -n vault
```
## Documentation
* [Learn more about Vault](https://www.vaultproject.io/)
* [Learm more about PKI in Vault](https://www.vaultproject.io/docs/secrets/pki/index.html)
* [Learn more about Key-Value management in Vault](https://www.vaultproject.io/docs/secrets/kv/index.html)
* [How to setup Vault/Consul HA](https://www.vaultproject.io/guides/operations/vault-ha-consul.html)