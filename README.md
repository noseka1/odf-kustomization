# Kustomization for Deploying OpenShift Container Storage

This kustomization makes use of [ocs-operator](https://github.com/openshift/ocs-operator) to deploy OpenShift Container Storage.

Red Hat OpenShift Container Storage product documentation can be found [here](https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage).

## Prerequisites

* OpenShift cluster with a minimum of 3 OCS worker nodes.
* Each worker node needs to have a minimum of 16 CPUs and 64 GB memory available.
* Worker nodes must be labeled with a label `cluster.ocs.openshift.io/openshift-storage=`
  * To label a node, issue the command:
    ```
    $ oc label nodes <node> cluster.ocs.openshift.io/openshift-storage=
    ```
  * To verify labeling, you can list the OCS worker nodes with:
    ```
    $ oc get node --selector cluster.ocs.openshift.io/openshift-storage=
    ```
    You should see at least three nodes listed in the output.

* If you plan to use the OCS worker nodes exlusively for OCS services, you can avoid double charges of both OpenShift and OpenShift Container Storage for OCS worker nodes, see also [here](https://access.redhat.com/solutions/4827161) . In order to achieve that you must set the role of OCS worker nodes to `infra`:

  * To remove the default `worker` role from OCS worker nodes, issue the command:
    ```
    $ oc label nodes <node> node-role.kubernetes.io/worker-
    ```
  * To add the infra role to OCS worker nodes, issue the command:
    ```
    $ oc label nodes <node> node-role.kubernetes.io/infra=
    ```
  * Configure the OpenShift scheduler to place workloads on regular worker nodes by default and not on OCS worker nodes:
    ```
    $ oc patch schedulers.config.openshift.io cluster \
      --type merge \
      --patch '{"spec":{"defaultNodeSelector": "node-role.kubernetes.io/worker="}}'
    ```

* See [BZ #1801008](https://bugzilla.redhat.com/show_bug.cgi?id=1801008) before you choose to follow this advice: It is recommended that you apply a taint to the nodes to mark them for exclusive OpenShift Container Storage use:
  ```
  $ oc adm taint nodes <node names> node.ocs.openshift.io/storage=true:NoSchedule
  ```
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

The commands in this section must be issued by an OpenShift user with a *cluster-admin* role.

### Installing ocs-operator

Review the [ocs-operator/base](ocs-operator/base) kustomization and modify it to suit your needs. Notably, set the version of the ocs-operator you want to deploy in [ocs-operator/base/openshift-storage-subscription.yaml](ocs-operator/base/openshift-storage-subscription.yaml).

To deploy an ocs-operator, issue the command:
```
$ oc apply --kustomize ocs-operator/base
```
Wait until the ocs-operator csvs are fully deployed. You can watch their deployment status with:
```
$ oc get csv --namespace openshift-storage
NAME                  DISPLAY                       VERSION   REPLACES              PHASE
awss3operator.1.0.1   AWS S3 Operator               1.0.1     awss3operator.1.0.0   Succeeded
ocs-operator.v4.2.1   OpenShift Container Storage   4.2.1                           Succeeded
```
All csvs must reach the phase `Succeeded`. Note that you must wait until the csvs are fully deployed before creating an OCS instance!

### Creating OCS instance

Review the [ocs-instance/base](ocs-instance/base) kustomization and modify it to suit your needs. Make sure that the `storageClassName` set in the [ocs-instance/base/ocs-storagecluster-storagecluster.yaml](ocs-instance/base/ocs-storagecluster-storagecluster.yaml) manifests is appropriate storage class for your infrastructure provider.

To deploy an OCS instance, issue the command:

```
$ oc apply --kustomize ocs-instance/base
```

Watch the status conditions of the storagecluster resource while the OCS instance is being deployed:

```
$ oc get storagecluster --namespace openshift-storage --output yaml
```

When the OCS instance deployment is complete, the status of the `Available` condition changes to `True` like this:

```
$ oc get storagecluster \
  --namespace openshift-storage \
  --output jsonpath='{.items[0].status.conditions[?(@.type=="Available")].status}' 
True
```
Congrats on completing the installation of OpenShift Container Storage!

### Changing the default StorageClass to OCS

Mark the current StorageClass as non-default:

```
$ oc patch storageclass gp2 \
    -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class": "false"}}}'
```
Configure ceph rbd as your default StorageClass:

```
$ oc patch storageclass ocs-storagecluster-ceph-rbd \
    -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class": "true"}}}'
```

### Deploying rook-ceph-tools

Optionally, you can deploy a Ceph toolbox:

```
$ oc apply --kustomize rook-ceph-tools/base
```

## Exercising the OCS cluster

To exercise the OCS cluster, you can follow the [OCS-training](https://github.com/red-hat-storage/ocs-training) hands-on workshop.
