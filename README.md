# NebulOuS helmfile

This repository tracks the **NebulOuS helmfile**. It is the **recommended** method of **installation** for **NebulOuS evaluators**.

## Hardware requirements

The target NebulOuS installation environment (in here, a Kubernetes cluster) should have **at least** that much **free** resources:

* 2 cores (currently only x86_64 arch is built and tested on the NebulOuS control plane side)
* 6 GiB RAM
* 20 GiB SSD-grade disk space

This environment will host the **NebulOuS control plane** and is independent of the environments that NebulOuS will manage.

## Software requirements

This method of installation requires the following:

- an existing (or new but doing this production-grade is beyond the scope of this README) **Kubernetes cluster**
  - minikube and kind will do just fine for evaluation purposes; we show an example of setting up kind in a compatible way
  - the cluster needs to be capable of exposing certain TCP- and HTTP-based services  (including **ingress**); the details (including ports) are given further on
- **Helmfile** v1: https://github.com/helmfile/helmfile (tested with v1.0.0-rc.1 up to v1.0.0-rc.3 -- any v1 should work as well), and its dependencies:
  - Helm v3: https://github.com/helm/helm (tested with v3.15.1 up to v3.15.3 -- any later should work as well -- limited to Helmfile support)
  - helm-diff v3: https://github.com/databus23/helm-diff (tested with v3.9.7 up to v3.9.9 -- any later should work as well -- limited to Helmfile support)
- (optional) git: https://git-scm.com/ (any typically available version nowadays will do)
  - this can be avoided by downloading the tarball of this repo but it is best practice to keep versioning any custom changes in git

### External dependency of NebulOuS - ProActive Scheduler

NebulOuS depends on having access to an installation of [ProActive Scheduler](https://github.com/ow2-proactive/scheduling) (PWS). It is used by the Scheduling Abstraction Layer (SAL) component of NebulOuS.

To set it up, please follow the instructions at https://github.com/eu-nebulous/nebulous/wiki/1.1-Installation-Walk%E2%80%90trough-for-Development-&-Evaluation#proactive-scheduler

## Exposed services (and their ports)

Certain services need to be typically reachable from outside the cluster. These are:

- Ingress-based (port TCP 80 for HTTP and TCP 443 port HTTPS [if enabled]):
  - [GUI](https://github.com/eu-nebulous/gui)
  - [GUI controller](https://github.com/eu-nebulous/gui-controller) (GUI's backend)
  - [Cloud-Fog Service Broker (CFSB)](https://github.com/eu-nebulous/cloud-fog-service-broker) (frontend + backend)
  - [Overlay Network Manager (ONM)](https://github.com/eu-nebulous/overlay-network-manager)

- TCP-Service-based:
  - [ActiveMQ](https://github.com/eu-nebulous/activemq) AMQP (default of port TCP 30000)
  - [Monitoring (EMS)](https://github.com/eu-nebulous/monitoring)
    - Web GUI over HTTPS (default of port TCP 30111)
    - ActiveMQ OpenWire TLS (default of port TCP 31617)
    - Baguette SSH (default of port TCP 32222)

Three services have to be reachable from the to-be-NebulOuS-managed systems: ActiveMQ AMQP, EMS (3 subservices) and ONM.

## (optional) Setting up kind

If you want to use kind for the first time (including downloading), please consult its docs: https://kind.sigs.k8s.io/

Example commands to set up kind with the necessary open ports and enabled services:

```
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 30000
    hostPort: 30000
    protocol: TCP
  - containerPort: 30111
    hostPort: 30111
    protocol: TCP
  - containerPort: 31617
    hostPort: 31617
    protocol: TCP
  - containerPort: 32222
    hostPort: 32222
    protocol: TCP
EOF

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

## Configuring the deployment

Any necessary customisations can be achieved by modifying the `helmfile.yaml` and other included `*-patches`. This section describes the bare minimum of configuration that needs to be adapted per environment.

Notably, every working installation will need changes in `config-patches` and `ingress-patches`. Please note that without modifications the helmfile is successfully deployablable but offers very limited functionality as all the possible access is from the local (to deployment) host only. The example patches use certain placeholders for ease of modification and depend on [sslip.io](https://sslip.io/) for subdomain HTTP service multiplexing:

* 127.0.0.1 denotes an address that must be reachable by NebulOuS users
  * its associated example domain is 127.0.0.1.sslip.io
* 127.0.0.129, 130, and 131 denote addresses that must be reachable from to-be-NebulOuS-managed systems
  * their associated domains are 127.0.0.129.sslip.io, 130, and 131 analogously

Often all the above can be replaced by the same, public IP address exposing the services of the Kubernetes cluster NebulOuS is installed on. If no domain name is available, the sslip.io variants can be kept (on the note that a globally-valid HTTPS certificate would not be possible).

Additionally, 127.0.1.1 denotes the address of ProActive Scheduler (see requirements above).

## Deploying = applying helmfile

If everything is set up, the (re)deployment is only a matter of running:

```
helmfile apply
```

And waiting for services to come online.

If no changes were made, it is idempotent. New changes can be deployed by making them in the files and rerunning the above command.
