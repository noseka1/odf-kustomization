# Deployment Options

Node roles:

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
