# Kustomization for Deploying OpenShift Data Foundation

This kustomization makes use of [odf-operator](https://github.com/openshift/odf-operator) to deploy OpenShift Data Foundation.

Red Hat OpenShift Data Foundation product documentation can be found [here](https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage).

This kustomization was tested on:
* VMware vSphere
* Amazon AWS

## Prerequisites

* OpenShift cluster with a minimum of 3 ODF worker nodes.
* Each worker node needs to have a minimum of 16 CPUs and 64 GB memory available.
* Worker nodes must be labeled with a label `cluster.ocs.openshift.io/openshift-storage=`
  * To label a node, issue the command:
    ```
    $ oc label nodes <node> cluster.ocs.openshift.io/openshift-storage=
    ```
  * To verify labeling, you can list the ODF worker nodes with:
    ```
    $ oc get node --selector cluster.ocs.openshift.io/openshift-storage=
    ```
    You should see at least three nodes listed in the output.

* If you plan to use the ODF worker nodes exlusively for ODF services, you can avoid double charges of both OpenShift and OpenShift Data Foundation for ODF worker nodes, see also [here](https://access.redhat.com/solutions/4827161). Refer to the following documentation on how to create infra nodes in your OpenShift cluster:

  * [Creating an Infra Node in OpenShift v4](https://access.redhat.com/solutions/4287111)
  * [Openshift 4 create infra machines ](https://access.redhat.com/solutions/4342791)
  * [Custom pools](https://github.com/openshift/machine-config-operator/blob/master/docs/custom-pools.md)
  
  The creation of infra nodes has also been implemented by the [openshift-post-install](https://github.com/noseka1/openshift-post-install) project.

* It is recommended that you apply a taint to the nodes to mark them for exclusive OpenShift Data Foundation use:
  ```
  $ oc adm taint nodes <node names> node.ocs.openshift.io/storage=true:NoSchedule
  ```
* The default storage class is set to the appropriate storage class for your infrastructure provider.
  * On AWS, the default storage class must be `gp3-csi`.
  * On VMware vSphere, the default storage class must be `thin`.
  
  For example, on AWS you can verify the default storage class setting with:
  ```
  $ oc get storageclass
  NAME                PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
  gp2-csi             ebs.csi.aws.com   Delete          WaitForFirstConsumer   true                   20h
  gp3-csi (default)   ebs.csi.aws.com   Delete          WaitForFirstConsumer   true                   20h
  ```
    
## Installing OpenShift Data Foundation

The commands in this section must be issued by an OpenShift user with a *cluster-admin* role.

### Installing odf-operator

Review the [odf-operator/base](odf-operator/base) kustomization and modify it to suit your needs. Notably, set the version of the odf-operator you want to deploy in [odf-operator/base/odf-operator-sub.yaml](odf-operator/base/odf-operator-sub.yaml).

To deploy an odf-operator, issue the command:
```
$ oc apply --kustomize odf-operator/base
```
Wait until the odf-operator csvs are fully deployed. You can watch their deployment status with:
```
$ oc get csv --namespace openshift-storage
NAME                                     DISPLAY                            VERSION    REPLACES                            PHASE
mcg-operator.v4.9.0                      NooBaa Operator                    4.9.0                                          Succeeded
ocs-operator.v4.9.0                      OpenShift Container Storage        4.9.0                                          Succeeded
odf-operator.v4.9.0                      OpenShift Data Foundation          4.9.0                                          Succeeded
```
All csvs must reach the phase `Succeeded`. Note that you must wait until the csvs are fully deployed before creating an ODF instance!

### Creating ODF instance

Review the [odf-instance/base](odf-instance/base) kustomization and modify it to suit your needs. Make sure that the `storageClassName` set in the *ocs-storagecluster-storagecluster.yaml* manifests is appropriate storage class for your infrastructure provider (use `gp3-csi` for AWS, `thin` for vSphere).

To deploy an ODF instance on AWS, issue the command:

```
$ oc apply --kustomize odf-instance/overlays/aws
```

To deploy an ODF instance on vSphere, issue the command:

```
$ oc apply --kustomize odf-instance/overlays/vsphere
```

Watch the status conditions of the storagecluster resource while the ODF instance is being deployed:

```
$ oc get storagecluster --namespace openshift-storage --output yaml
```

When the ODF instance deployment is complete, the status of the `Available` condition changes to `True` like this:

```
$ oc get storagecluster \
  --namespace openshift-storage \
  --output jsonpath='{.items[0].status.conditions[?(@.type=="Available")].status}' 
True
```
Congrats on completing the installation of OpenShift Data Foundation!

### Changing the default StorageClass to ODF

Mark the current StorageClass as non-default:

```
$ oc patch storageclass gp3-csi \
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
$ oc patch \
    --namespace openshift-storage \
    --type json \
    --patch  '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]' \
    OCSInitialization ocsinit
```

Obtain the rook-ceph-tools pod:

```
$ TOOLS_POD=$(oc get pods --namespace openshift-storage --selector app=rook-ceph-tools --output name)
```

Run Ceph commands:

```
$ oc rsh --namespace openshift-storage $TOOLS_POD ceph status
$ oc rsh --namespace openshift-storage $TOOLS_POD ceph osd status
$ oc rsh --namespace openshift-storage $TOOLS_POD ceph osd tree
$ oc rsh --namespace openshift-storage $TOOLS_POD ceph df
$ oc rsh --namespace openshift-storage $TOOLS_POD rados df
```

### Accessing Ceph dashboard

Enable dashboard (remember to set the TOOLS_POD variable as shown above before running this command):

```
$ oc rsh --namespace openshift-storage $TOOLS_POD ceph mgr module enable dashboard
```

Create user admin with the role administrator. Replace <password> with your own admin password:

```
$ echo <password> | oc rsh --namespace openshift-storage $TOOLS_POD ceph dashboard ac-user-create admin administrator -i -
```

Forward local port 8443 to the dashboard endpoint:

```
$ oc port-forward deploy/rook-ceph-mgr-a 8443
```

Browse to https://localhost:8443 to access the dashboard.

## Exercising the ODF cluster

You can create test volumes by issuing the commands:

```
$ oc apply -f docs/samples/test-rwo-pvc.yaml
```

```
$ oc apply -f docs/samples/test-rwx-pvc.yaml
```

```
$ oc apply -f docs/samples/test-bucket-obc.yaml
```

To further exercise the ODF cluster, you can follow the [OCS-training](https://github.com/red-hat-storage/ocs-training) hands-on workshop.

## Troubleshooting

Collect debugging data about the currently running Openshift cluster:

```
$ oc adm must-gather
```

Collect debugging information specific to OpenShift Data Foundation:

```
$ oc adm must-gather --image registry.redhat.io/ocs4/ocs-must-gather-rhel8:v4.9
```
