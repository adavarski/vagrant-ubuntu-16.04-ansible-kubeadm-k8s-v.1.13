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
  
### Manual install helm charts if tiller pod is not yet available during ansible play
```

??? helm install stable/nginx-ingress -f /etc/kubernetes/helm-values/nginx-ingress-external.yml --tls --tls-ca-cert /etc/kubernetes/pki/helm/prod/tiller-prod-ca.pem --tls-cert /etc/kubernetes/pki/helm/prod/helm-prod-cert.pem --tls-key /etc/kubernetes/pki/helm/prod/helm-prod-key.pem --tiller-namespace=production --namespace=kube-system

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

```
$ kubectl get all --all-namespaces
NAMESPACE     NAME                                                                  READY   STATUS    RESTARTS   AGE
development   pod/tiller-deploy-8d8676d44-8fzfb                                     1/1     Running   1          51m
kube-system   pod/coredns-86c58d9df4-7wpd2                                          1/1     Running   0          52m
kube-system   pod/coredns-86c58d9df4-xq4jw                                          1/1     Running   0          52m
kube-system   pod/etcd-kube-master-01                                               1/1     Running   0          53m
kube-system   pod/grafana-8579d65cc6-t5ktb                                          1/1     Running   0          21m
kube-system   pod/kube-apiserver-kube-master-01                                     1/1     Running   0          53m
kube-system   pod/kube-controller-manager-kube-master-01                            1/1     Running   0          53m
kube-system   pod/kube-proxy-4s9ks                                                  1/1     Running   0          52m
kube-system   pod/kube-proxy-qgzk5                                                  1/1     Running   0          52m
kube-system   pod/kube-scheduler-kube-master-01                                     1/1     Running   0          53m
kube-system   pod/kubernetes-dashboard-79ff88449c-mrbt8                             1/1     Running   0          52m
kube-system   pod/prometheus-alertmanager-5bbd595756-ml29q                          2/2     Running   0          24m
kube-system   pod/prometheus-kube-state-metrics-b4d7c746-pn5dj                      1/1     Running   0          24m
kube-system   pod/prometheus-node-exporter-cltk2                                    1/1     Running   0          24m
kube-system   pod/prometheus-pushgateway-695c7c958b-59b4j                           1/1     Running   0          24m
kube-system   pod/prometheus-server-7f59447d54-vp6qp                                2/2     Running   0          24m
kube-system   pod/rafting-greyhound-nginx-ingress-controller-6b584bf574-wbzw6       0/1     Running   0          5m49s
kube-system   pod/rafting-greyhound-nginx-ingress-default-backend-74897cd46ffl6jg   1/1     Running   0          5m49s
kube-system   pod/weave-net-hlwfd                                                   2/2     Running   0          52m
kube-system   pod/weave-net-sdvmf                                                   2/2     Running   0          52m
production    pod/tiller-deploy-77d75bcbf6-4m575                                    1/1     Running   1          51m
staging       pod/tiller-deploy-57b8499c56-gfqtl                                    1/1     Running   0          51m

NAMESPACE     NAME                                                         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
default       service/kubernetes                                           ClusterIP      10.96.0.1        <none>        443/TCP                      53m
development   service/tiller-deploy                                        ClusterIP      10.103.68.208    <none>        44134/TCP                    51m
kube-system   service/grafana                                              ClusterIP      10.110.63.209    <none>        80/TCP                       21m
kube-system   service/kube-dns                                             ClusterIP      10.96.0.10       <none>        53/UDP,53/TCP                53m
kube-system   service/kubernetes-dashboard                                 ClusterIP      10.110.90.35     <none>        443/TCP                      53m
kube-system   service/prometheus-alertmanager                              ClusterIP      10.109.40.210    <none>        80/TCP                       24m
kube-system   service/prometheus-kube-state-metrics                        ClusterIP      None             <none>        80/TCP                       24m
kube-system   service/prometheus-node-exporter                             ClusterIP      None             <none>        9100/TCP                     24m
kube-system   service/prometheus-pushgateway                               ClusterIP      10.109.8.70      <none>        9091/TCP                     24m
kube-system   service/prometheus-server                                    ClusterIP      10.101.67.5      <none>        80/TCP                       24m
kube-system   service/rafting-greyhound-nginx-ingress-controller           LoadBalancer   10.99.185.58     <pending>     80:30462/TCP,443:32467/TCP   5m50s
kube-system   service/rafting-greyhound-nginx-ingress-controller-metrics   ClusterIP      10.97.252.68     <none>        9913/TCP                     5m51s
kube-system   service/rafting-greyhound-nginx-ingress-controller-stats     ClusterIP      10.100.215.232   <none>        18080/TCP                    5m50s
kube-system   service/rafting-greyhound-nginx-ingress-default-backend      ClusterIP      10.110.194.252   <none>        80/TCP                       5m50s
production    service/tiller-deploy                                        ClusterIP      10.104.142.26    <none>        44134/TCP                    51m
staging       service/tiller-deploy                                        ClusterIP      10.96.186.210    <none>        44134/TCP                    51m

NAMESPACE     NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-system   daemonset.apps/kube-proxy                 2         2         2       2            2           <none>          53m
kube-system   daemonset.apps/prometheus-node-exporter   1         1         1       1            1           <none>          24m
kube-system   daemonset.apps/weave-net                  2         2         2       2            2           <none>          52m

NAMESPACE     NAME                                                              READY   UP-TO-DATE   AVAILABLE   AGE
development   deployment.apps/tiller-deploy                                     1/1     1            1           51m
kube-system   deployment.apps/coredns                                           2/2     2            2           53m
kube-system   deployment.apps/grafana                                           1/1     1            1           21m
kube-system   deployment.apps/kubernetes-dashboard                              1/1     1            1           53m
kube-system   deployment.apps/prometheus-alertmanager                           1/1     1            1           24m
kube-system   deployment.apps/prometheus-kube-state-metrics                     1/1     1            1           24m
kube-system   deployment.apps/prometheus-pushgateway                            1/1     1            1           24m
kube-system   deployment.apps/prometheus-server                                 1/1     1            1           24m
kube-system   deployment.apps/rafting-greyhound-nginx-ingress-controller        0/1     1            0           5m50s
kube-system   deployment.apps/rafting-greyhound-nginx-ingress-default-backend   1/1     1            1           5m49s
production    deployment.apps/tiller-deploy                                     1/1     1            1           51m
staging       deployment.apps/tiller-deploy                                     1/1     1            1           51m

NAMESPACE     NAME                                                                         DESIRED   CURRENT   READY   AGE
development   replicaset.apps/tiller-deploy-8d8676d44                                      1         1         1       51m
kube-system   replicaset.apps/coredns-86c58d9df4                                           2         2         2       52m
kube-system   replicaset.apps/grafana-8579d65cc6                                           1         1         1       21m
kube-system   replicaset.apps/kubernetes-dashboard-79ff88449c                              1         1         1       52m
kube-system   replicaset.apps/prometheus-alertmanager-5bbd595756                           1         1         1       24m
kube-system   replicaset.apps/prometheus-kube-state-metrics-b4d7c746                       1         1         1       24m
kube-system   replicaset.apps/prometheus-pushgateway-695c7c958b                            1         1         1       24m
kube-system   replicaset.apps/prometheus-server-7f59447d54                                 1         1         1       24m
kube-system   replicaset.apps/rafting-greyhound-nginx-ingress-controller-6b584bf574        1         1         0       5m49s
kube-system   replicaset.apps/rafting-greyhound-nginx-ingress-default-backend-74897cd46f   1         1         1       5m49s
production    replicaset.apps/tiller-deploy-77d75bcbf6                                     1         1         1       51m
staging       replicaset.apps/tiller-deploy-57b8499c56                                     1         1         1       51m
```

