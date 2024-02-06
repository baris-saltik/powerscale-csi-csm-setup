# This file contains the steps to deploy Dell CSM. It follows manual installation without Operator Lifecycle Management (OLM) method and does not include CSM Replication steps. 

## 1. CSM Operator Installation
### a. Install Snapshot CRDs

https://dell.github.io/csm-docs/docs/snapshots/#installation-example

```bash
git clone -b release-6.2 https://github.com/kubernetes-csi/external-snapshotter
cd ./external-snapshotter
kubectl kustomize client/config/crd | kubectl create -f -
kubectl -n kube-system kustomize deploy/kubernetes/snapshot-controller | kubectl create -f -
```


### b. Install CSM Operator

https://dell.github.io/csm-docs/docs/deployment/csmoperator/#manual-installation-on-a-cluster-without-olm

```bash
git clone -b v1.4.1 https://github.com/dell/csm-operator.git
cd csm-operator
bash scripts/install.sh
```

Verify that CSM Operator pod is running.
```bash
kubectl get pods -n dell-csm-operator
```

### b. Install PowerScale CSI Driver Using CSM

https://dell.github.io/csm-docs/docs/deployment/csmoperator/drivers/powerscale/

To see the list of installed CSI drivers:  
```bash
kubectl get csm --all-namespaces
```

Create a new namespace.
```bash
kubectl create namespace isilon
```

Create a secret.yaml file with the same content from "powerscale_csi_driver/secret.yaml" file in this repository. Customize the file to reflect the values of your environment.

Then create "isilon-creds" secret using this file:

```bash
kubectl create secret generic isilon-creds -n isilon --from-file=config=secret.yaml
```

If certificate validation is skipped, create an empty secret using the file "powerscale_csi_driver/empty-secret.yaml".

Then create the secret. Secret name in the yaml file above is "isilon-certs-0".

```bash
kubectl create -f empty-secret.yaml
```

Verify that both secrets are created.

> kubectl get secrets --namespace isilon  
> NAME             TYPE     DATA   AGE  
> isilon-certs-0   Opaque   1      7s  
> isilon-creds     Opaque   1      6m35s  

Copy the following sample PowerScale CSI installation file that is meant to work with CS Operator from the following directory into your working directory. Edit the file to reflect values from your environment.

```bash
cd *working_dir*
cp *clone_csm_operator_github*/csm-operator/samples/storage_csm_powerscale_v290.yaml .
```
Or you can use "powerscale_csi_driver/storage_csm_powerscale_v290.yaml" in this repostory and make your changes on that. Following are the changes from defaults in that file.

> ...  
> spec:   
> &nbsp;&nbsp;driver:  
> &nbsp;&nbsp;&nbsp;&nbsp;**replicas:1**  
> ...  
> &nbsp;&nbsp;driver:modules:  
> &nbsp;&nbsp;&nbsp;&nbsp;- name: observability  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**enabled: true**  
> ...   
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;components:  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- name: topology  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**enabled: true**  
> ...  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- name: otel-collector  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**enabled: true**  
> ...  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- name: cert-manager  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**enabled: true**  
> ...  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- name: metrics-powerscale  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**enabled: true**  
> ...       

Create a namespace named karavi and install CSI driver by runnning the following.
```bash
kubectl create namespace karavi
kubectl create -f storage_csm_powerscale_v290.yaml
```



