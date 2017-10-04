# oscm-helm

Helm charts for provisioning OSCM on Kubernetes.

## oscm-demo-helm
This repository contains templates for the deployment of OSCM as a platform for managing Kubernetes applications using Helm. 
The `oscm-demo-helm` directory contains:
- `kubernetes-templates` - Kubernetes templates for deployng OSCM applications, Kafka and Rudder REST Proxy for Helm.
- `oscm-service` - Sample OSCM techncial service definition for installing WordPress on Kubernetes with OSCM.

The same cluster is used for deploying OSCM with all necessary applications. It is also used as a target cluster for applications managed by OSCM. The deployment can be adapted as needed. For example, you can set up different clusters for OSCM and for the managed applications, or define different namespaces.  
The image below shows the deployment in different clusters.

![OSCM Helm Provisioning](oscm-demo-helm/img/Demo.jpg)

**TODO**
- [ ] Simplify the installation using Helm charts.
- [ ] Solve [issue](https://github.com/servicecatalog/provisioning-service/issues/8)

## Deployment on Google Cloud Platform (GCP)

Prerequisites:
- gcloud SDK with kubectl on your system (alternatively, you can use the Kubernetes Dashboard). 
- Kubernetes cluster with 2 nodes type "n1-standard-2" (2 CPU, 7.5 GB) in Google Container Engine
- Gmail account for OSCM email notification allowing for access of less secure applications (or other mail server solution)

Set the correct target cluster from the client using commands shown in GCP. To get the commands, use the "connect" button on your cluster in Container Engine/Container clusters.

Assuming that the kube configuration has the target cluster in the current context, execute the following installation steps in the given order:

### Install Helm

Helm has two parts: a client (`helm`) and a server (`tiller`). Tiller runs inside of your Kubernetes cluster and manages releases (installations) of your charts. 
1. Download the binaries for your system [here](https://github.com/kubernetes/helm/releases). 
2. The command `helm init` will install the `tiller` server in your cluster. 
3. Once you have the client installed, upgrade Tiller with `helm init --upgrade`.

### Install Rudder

Rudder Proxy interfaces the Helm Chart repositories and the Helm Tiller server. The public [Kubernetes charts](https://github.com/kubernetes/charts) repository is configured in the `rudder-repositories.yaml` file. You can add your own Helm chart repository to this file. The `rudder-repositories.yaml` is saved as a Kubernetes secret.

1. `kubectl create generic rudder-repositories --from-file rudder-repositories.yaml`
2. Extract the IP address and port of the `tiller` pod and adapt it in the `rudder.yaml` file via`kubectl get pod --all-namespaces`
3. `kubectl create -f rudder.yaml`

### Install Kafka

1. `kubectl create -f zookeeper.yaml`
2. `kubectl create -f kafka.yaml`

### Install OSCM Applications


1. `kubectl create -f helm-provisioning.yaml`
2. `kubectl create -f oscm-db.yaml`
3. `kubectl create -f oscm-initdb-jms.yaml`
4. `kubectl create -f oscm-bes-svc.yaml`
5. Extract the external IP address of the bes service and adapt the BASE_URL and BASE_URL_HTTPS in the `oscm-initdb-bes.yaml` file. Fill out REPORT settings and all SSO settings using any valid URL, even though these are currently not used. Use `kubectl get svc --all-namespaces` to find the IP address.
6. `kubectl create -f oscm-initdb-bes.yaml`. Wait until this job is finished (check in the Kubernetes Dashboard)
7. `kubectl create -f oscm-bes-pod.yaml`. Wait until this job is finished (check in the Kubernetes Dashboard) before trying to log in to the OSCM portal.

### OSCM Service Definition for Helm Charts

In order to manage the Kubernetes applications with OSCM, they must be represented in OSCM by corresponding service definitions (technical services in OSCM). The OSCM service definition describes:  
- The target Kubernetes cluster (URL of the Rudder Proxy) where the application will be deployed (`target` parameter);
- The target Kubernetes namespace for deployment (`namespace` parameter)
- The provisioning template (parameters with the `template.` prefix for the chart repository, name and version);
- Kubernetes label as identifying attribute for the deployment (`labels.release` parameter)
- Application parameters (parameters with the `parameters.` prefix which correspond to the values in the `values.yaml` file of the chart).

A sample service definition for the [`WordPress`](https://github.com/kubernetes/charts/tree/master/stable/wordpress) chart from the [kubeapps repository](https://github.com/kubernetes/charts) can be found [here](https://github.com/servicecatalog/oscm-helm/blob/master/oscm-demo-helm/oscm-service/TechnicalServicesHelmWordPress.xml).


### Getting Started

To start working with OSCM, please see the [Getting Started](https://github.com/servicecatalog/development/wiki/Getting-Started) guide.

*Note: To log in to the OSCM portal, use the external IP address and port. Use `kubectl get svc --all-namespaces` to find them out.

If you use the sample [WordPress service](https://github.com/servicecatalog/oscm-helm/blob/master/oscm-demo-helm/oscm-service/TechnicalServicesHelmWordPress.xml), you can define some settings in the technical service definition. For example: 

- Let the user configure the WordPress version, admin credentials, blog name, and cluster resources when subscribing
- Define prices for cluster resources (CPU, memory, storage)

*Note: When the subscription is ready for use, it is planned to see the access information (URL or other) in OSCM ([issue](https://github.com/servicecatalog/provisioning-service/issues/8)). As long as the issue is not resolved, the access information can be seen in the `Kubernetes Dashboard` or extracted using the `kubectl` command*.

The WordPress application can be managed by modifying the corresponding OSCM subscription:
- Upgrade/downgrade to different WordPress versions
- Scale up/down by using different cluster resources
- Delete the application by terminating the OSCM subscription


Enjoy!
