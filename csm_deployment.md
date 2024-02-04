# This file contains the steps to deploy Dell CSM. It follows manual installation without Operator Lifecycle Management (OLM) method and does not include CSM Replication steps. 

## 1. CSM Operator Installation
### a. Install Snapshot CRDs

https://dell.github.io/csm-docs/docs/snapshots/#installation-example

### b. Install CSM Operator

https://dell.github.io/csm-docs/docs/deployment/csmoperator/#manual-installation-on-a-cluster-without-olm

```bash
git clone -b v1.4.1 https://github.com/dell/csm-operator.git
cd csm-operator
bash scripts/install.sh
```


