# TAP Automation on AKS

A tekton pipeline that provision an AKS cluster and install TAP full profile with HTTPS (Let's Encrypt) enabled on the cluster.

## Prerequisites

* [`kubectl`](https://kubernetes.io/docs/tasks/tools/) CLI
* [`kind`](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) CLI
* [`az`](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) CLI
* [`tkn`](https://tekton.dev/docs/cli/) CLI
* Azure Account
* TanzuNet Account

## Create Service Principal

```
SP_NAME=tap-automation                                                                              
SP=$(az ad sp create-for-rbac --name ${SP_NAME} --years 10)
SP_APP_ID=$(echo ${SP} | jq -r '.appId')
SP_PASSWORD=$(echo ${SP} | jq -r '.password')
AZURE_TENANT_ID=$(echo ${SP} | jq -r '.tenant')
AZURE_SUBSCRIPTION_ID=$(az account show --query id --output tsv)

az role assignment create --assignee ${SP_APP_ID} --role "Contributor"
az role assignment create --assignee ${SP_APP_ID} --role "User Access Administrator"
```

## Create a Kind Cluster

```
kind create cluster --name tap-automation
```

## Install Tekton on the kind cluster

```
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply -f https://storage.googleapis.com/tekton-releases/dashboard/latest/tekton-dashboard-release.yaml
```

## Configure Secrets

```
kubectl create ns tap-automation
```

### Configure Service Principal

```
kubectl create secret generic azure-service-principal \
  --from-literal client_id=${SP_APP_ID} \
  --from-literal client_secret=${SP_PASSWORD} \
  --from-literal tenant_id=${AZURE_TENANT_ID} \
  --from-literal subscription_id=${AZURE_SUBSCRIPTION_ID} \
  --dry-run=client \
  -o yaml > azure-service-principal.yaml
kubectl apply -f azure-service-principal.yaml -n tap-automation
```

### Configure TanzuNet credentials

Prepare username, password and api token for TanzuNet

```
TANZUNET_API_TOKEN=...
TANZUNET_USERNAME=...
TANZUNET_PASSWORD=...

kubectl create secret generic tanzunet \
  --from-literal api_token="${TANZUNET_API_TOKEN}" \
  --from-literal username="${TANZUNET_USERNAME}" \
  --from-literal password="${TANZUNET_PASSWORD}" \
  --dry-run=client \
  -o yaml > tanzunet.yaml
kubectl apply -f tanzunet.yaml -n tap-automation
```

## Install the tap-automation pipeline

```
kubectl apply -f https://github.com/making/tap-automation-on-aks/raw/main/tap-automation.yaml -n tap-automation
```

```
$ kubectl get -n tap-automation pipeline,task
NAME                                        AGE
pipeline.tekton.dev/tap-automation-create   20s
pipeline.tekton.dev/tap-automation-delete   20s

NAME                                         AGE
task.tekton.dev/az-login                     20s
task.tekton.dev/create-acr                   20s
task.tekton.dev/create-aks                   20s
task.tekton.dev/create-envoy-ip              20s
task.tekton.dev/create-resource-group        20s
task.tekton.dev/delete-acr                   20s
task.tekton.dev/delete-aks                   20s
task.tekton.dev/delete-envoy-ip              20s
task.tekton.dev/delete-resource-group        20s
task.tekton.dev/delete-tap                   20s
task.tekton.dev/download-from-pivnet         20s
task.tekton.dev/install-cluster-essentials   20s
task.tekton.dev/install-tap                  20s
```

## Install TAP using the tap-automation pipeline

```
tkn pipeline start tap-automation-create --showlog -w name=config,claimName=tap-automation-config --use-param-defaults -n tap-automation
```

you can override the following parameters with `-p` option

<table>
<tr>
<th>
Name
</th>
<th>
Default
</th>
<tr>
<td>
<code>resource_group</code>
</td>
<td>
tap-rg
</td>
</tr>
<tr>
<td>
<code>location</code>
</td>
<td>
japaneast
</td>
</tr>
<tr>
<td>
<code>cluster_name</code>
</td>
<td>
tap-sandbox
</td>
</tr>
<tr>
<td>
<code>vm_size</code>
</td>
<td>
standard_f4s_v2
</td>
</tr>
<tr>
<td>
<code>node_count</code>
</td>
<td>
3 (subject to change by cluster-autoscaler)
</td>
</tr>
<tr>
<td>
<code>tap_version</code>
</td>
<td>
1.2.0
</td>
</tr>
<tr>
<td>
<code>tbs_version</code>
</td>
<td>
1.6.0
</td>
</tr>
<tr>
<td>
<code>cluster_essentials_version</code>
</td>
<td>
1.2.0
</td>
</tr>
</table>

[Log Example](https://gist.github.com/making/354fad9227a3298654e84e68d4ceba45)

After the installation, you can access the TAP GUI, The url should be `https://tap-gui.***-***-***-***.sslip.io` <<-- check the url in the log.

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/106908/182030737-aaaabf22-afcf-4eca-8dcb-144fb01aee30.png">

You can see the result of the pipelinerun on the tekton dashboard

```
kubectl port-forward -n tekton-pipelines service/tekton-dashboard 9097:9097
```

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/106908/182033773-6d5460e3-29af-442d-91fe-15cffa13998e.png">

## Deploy a workload

```
az aks get-credentials --resource-group tap-rg --name tap-sandbox --overwrite-existing --admin 
```

```
tanzu apps workload apply spring-music \
  --app spring-music \
  --git-repo https://github.com/scottfrederick/spring-music \
  --git-branch tanzu \
  --type web \
  --annotation autoscaling.knative.dev/minScale=1 \
  -n demo \
  -y
tanzu apps workload tail spring-music -n demo
```

```
tanzu apps workload get -n demo spring-music
```

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/106908/182030702-94dd3176-ca45-4ebc-b4db-5260d81dcd15.png">

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/106908/182030768-f28f0451-ee5e-4dc8-ac92-575d8672ebc3.png">

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/106908/182030984-c1247d72-db64-477f-b016-dafe8d9791aa.png">

## Uninstall TAP using the tap-automation pipeline

Don't forget to set the context back to the kind cluster

```
kubectl config use-context kind-tap-automation
```

```
tkn pipeline start tap-automation-delete --showlog -w name=config,claimName=tap-automation-config --use-param-defaults -n tap-automation
```

you can override the following parameters with `-p` option

<table>
<tr>
<th>
Name
</th>
<th>
Default
</th>
<tr>
<td>
<code>resource_group</code>
</td>
<td>
tap-rg
</td>
</tr>
<tr>
<td>
<code>cluster_name</code>
</td>
<td>
tap-sandbox
</td>
</tr>
<tr>
<td>
<code>acr_name</code>
</td>
<td>
auto
</td>
</tr>
</table>