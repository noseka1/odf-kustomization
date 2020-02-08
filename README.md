# Kustomization for Deploying OpenShift Container Storage

This kustomization makes use of [ocs-operator](https://github.com/openshift/ocs-operator) to deploy OpenShift Container Storage.

Red Hat OpenShift Container Storage product documentation can be found [here](https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage).

## Prerequisites

* OpenShift cluster with a minimum of 3 OCS worker nodes.
* Each worker node needs to have a minimum of 16 CPUs and 64 GB memory available.
* Worker nodes must be labeled with a label `cluster.ocs.openshift.io/openshift-storage=`
  * To label a node, issue the command:  
    `$ oc label nodes <node> cluster.ocs.openshift.io/openshift-storage=`    
  * To verify labeling, you can list the OCS worker nodes with:  
    `$ oc get node --selector cluster.ocs.openshift.io/openshift-storage=`
    
    You should see at least three nodes listed in the output.
* The default storage class is set to the appropriate storage class for your infrastructure provider.
  * On AWS, the default storage class must be `gp2`.
  * On VMware vSphere, the default storage class must be `thin`.
  
  For example, on AWS you can verify the default storage class setting with:
  ```
  $ oc get storageclass
  NAME            PROVISIONER             AGE
  gp2 (default)   kubernetes.io/aws-ebs   27m
  ```
    
## Installing OpenShift Container Storage
