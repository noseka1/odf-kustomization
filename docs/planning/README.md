# Deployment Options

Node roles:

* Helper Host = Linux host used to run the OCP installer
* Bootstrap = ephemeral node needed for installation only
* Master = OpenShift control plane
* Infra = OpenShift router, logging, monitoring, and integrated image registry
* App = Application node
* OCS = Ceph control plane + data plane

## Simple Deployment

![Deployment Diagram](ocs_on_vsphere_simple_deployment.svg "Deployment Diagram")

## Optimized Deployment

![Deployment Diagram](ocs_on_vsphere_optimized_deployment.svg "Deployment Diagram")

## Scaled-out Deployment

![Deployment Diagram](ocs_on_vsphere_scaled_out_deployment.svg "Deployment Diagram")

# Node Sizing

 Role | Count | vCPUs | Memory (GB)| Storage (GB)
--------- | ----- | ---- | ------ | -------
Helper Host | 1 | 2 | 8 | 40
Bootstrap | 1 | 4 | 16 | 120
Master | 3 | 8 | 24 | 120
Infra | 3 | 8 | 32 | 120
App | 3+ | 8+ | 32+ | 120
OCS | 3+ | 16 | 96 | 120
