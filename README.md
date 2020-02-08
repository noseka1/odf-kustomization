# Kustomization for Deploying OpenShift Container Storage

This kustomization makes use of [ocs-operator](https://github.com/openshift/ocs-operator) to deploy OpenShift Container Storage.

## Prerequisites

* OpenShift cluster with a minimum of 3 OCS worker nodes.
* Each worker node needs to have a minimum of 16 CPUs and 64 GB memory available.
* Worker nodes must be labeled with a label `cluster.ocs.openshift.io/openshift-storage=`
  * To label a node, issue the command:  
    `oc label nodes <node> cluster.ocs.openshift.io/openshift-storage=`    
  * To verify labeling, you can list the OCS worker nodes with:  
    `oc get node --selector cluster.ocs.openshift.io/openshift-storage=`
    
    You should see at least three nodes listed in the output.
    
## Installing OpenShift Container Storage
