# Kubeadm Ansible Playbook
---

Quickly bootstrap a Kubernetes cluster via Kubeadm using this Playbook.

System Requirements:

* Required Ansible version: 2.5+ or 2.6+
* Passwordless SSH access for all nodes
* Currently targetted for Ubuntu 16.04 (can be adapted to other distros)
* 4GB RAM (6GB if running via Vagrant/VM) with a dual-core processor minimum recommended

Environment Info:

* Kubernetes version: 1.13
* CNI: Weave-Net
* Default number of nodes: 2

## Usage:

---

### Vagrant:

* Install [VirtualBox][0] and [Vagrant][1].

* Run `vagrant up`. This will automatically provision two VM with Kubernetes cluster installed with some addons (check below).

### Manual Installation

* Add the system-node information to the `inventory/hosts.yml` file. For example:

```YAML
all:
  children:

    kubernetes:
      children:

        kubernetes-masters:
          hosts:
            kube-master-01:
```

* Edit `group_vars/all.yml` and other role-specific variables (under `roles/<role-name>/default/main.yml`) as required, also you can setup for your needs `roles/kube-master/templates/kubeadm-config.yml.j2`.

* Run the `kubernetes-addons.yaml` playbook ansible 2.5+.  
  
  ```Bash
  ansible-playbook kubernetes-addons.yaml
  ```
  
* Run the `kubernetes-addons.yaml` playbook with privileges if using ansible 2.6+.  
  **Note**: Since Ansible 2.6+, it is required that the private_key must have permissions `400`, and the ansible-files should reside in a non-public directory (permissions `744`).

  ```Bash
  sudo ansible-playbook kubernetes-addons.yaml
  ```
  
  

#### Getting the Kubeconfig for Kubectl

* Copy file `/etc/kubernetes/admin.conf` to `~/.kube/config`. This will allow Kubectl to connect and operate on this node.

  Within the Vagrant VM, all this has already been setup.

### Namespaces

The role installs additional three namespaces by default:

* Development
* Staging
* Production

To not install these namespaces, disable the *Setup Namespaces* step in `roles/addons/tasks/main.yml`.
**Note**: A lot of addons depend on these namespaces to be present, so you'll have to make changes to addon-installation tasks too.

### Addons

The playbook install following addons by default:

* Helm/Tillar
* Nginx Ingress (Internal Services)
* Nginx Ingress (External Services)
* Prometheus
* Grafana (includes a dashboard, using Prometheus as DataSource)
* The default Kubeadm addons (such as Dashboard)

If you would prefer not to install these addons, comment-out/delete the relavant lines in `kubernetes-addons.yaml` file. For example, to disable addons:

```YAML
# - name: Setup Kubernetes addons
#   hosts: kube-master-01
#   gather_facts: true
#   become: true
#   roles:
#     - role: addons
#       tags: addons
```

### The Nginx-Ingress Addons

* By default, the internal Nginx-Ingress is exposed via NodePorts, while the external Nginx-Ingress uses HostNetwork. This can be configured in the addons role: `roles/addons/templates`

* Check the addon-templates to know more about how the Ingress services are created: `roles/addons/templates`.

### Security

This is intended to be just a basic setup, and as such, the security is very basic.  
With Kubernetes providing vast security options, the user is expected to go through the setup and customize options such as security (and especially security) as per requirements.

  [0]: https://www.virtualbox.org/
  [1]: https://www.vagrantup.com/
  
### Manual install helm charts if tiller pod is not yet available during ansible play
```
$ helm install stable/prometheus --name prometheus -f /etc/kubernetes/helm-values/prometheus.yml --tls --tls-ca-cert /etc/kubernetes/pki/helm/prod/tiller-prod-ca.pem --tls-cert /etc/kubernetes/pki/helm/prod/helm-prod-cert.pem --tls-key /etc/kubernetes/pki/helm/prod/helm-prod-key.pem --tiller-namespace=production --namespace=kube-system

$ helm install stable/grafana --name grafana -f /etc/kubernetes/helm-values/grafana.yml --tls --tls-ca-cert /etc/kubernetes/pki/helm/prod/tiller-prod-ca.pem --tls-cert /etc/kubernetes/pki/helm/prod/helm-prod-cert.pem --tls-key /etc/kubernetes/pki/helm/prod/helm-prod-key.pem --tiller-namespace=production --namespace=kube-system
```
### View grafana if nginx LB is not installed or working

```
$ kubectl get pods --namespace kube-system -l "app=grafana" -o jsonpath="{.items[0].metadata.name}"
grafana-8579d65cc6-t5ktb

$ kubectl --namespace kube-system port-forward grafana-8579d65cc6-t5ktb 3000

BROWSER: http://localhost:3000
```

