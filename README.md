## On Premise setup of Kubernetes
###### Can be adapted to baremetal servers


#### Install steps kubernetes on virtualbox

*Pre-requisits, understanding of networking, routing and virtualbox networking, bridging, linked cloning and snapshots*

1. Install vbox additions
1. apt-get install build-essential
1. mount /dev/cdrom /media/cdrom
1. cd /media/cdrom
1. apt-get install build-essential
1. ./VboxLinuxAdditions.run
1. Shutdown server
1. Add linux shared folder and boot (with these files)
1. *If using seperate static network for workers*

    **Modify:** `/etc/netplan/01-<press tab>`
    NOTE: gateway4 for enp0s3 should be 10.0.0.1 for workers.

    ```yaml
    enp0s3
    addresses: [10.0.0.1]
    gateway4: 192.168.1.1
    nameservers:
      addresses: [192.168.1.1, 8.8.8.8, 8.8.4.4]
    dhcp4: no

    enp0s8
    dhcp4: yes
    nameservers:
      search: [scriptninjas.se]
      addresses: [192.168.1.1, 8.8.8.8, 8.8.4.4]
    ```
1. ***If using seperate static network you need routing***
    ```bash
    cp firewall.service /etc/systemd/system/
    echo "1" > /proc/sys/net/ipv4/ip_forward
    iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE
    iptables -A FORWARD -i enp0s8 -o enp0s3 -m state --state RELATED,ESTABLISHED -j ACCEPT
    iptables -A FORWARD -i enp0s3 -o enp0s8 -j ACCEPT
    iptables-save > /etc/firewall.rule
    ```
1. Make a file with the content: /etc/systemd/firewall.sh
    ```bash
    #!/bin/sh
    iptables-restore < /etc/firewall.rules
    ```
1. Run commands:
    ```bash
    chmod a+x /etc/systemd/firewall.sh
    sudo systemctl daemon-reload
    sudo systemctl enable firewall.service
    ```

##### Install Kubernetes
1. use ssh-keygen to generate new id on master, and ssh-copy-id to install it on all nodes for access.
2. Install commands

    ```bash
    apt-get update && apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" >> /etc/apt/sources.list.d/kubernetes.list
    apt-get update
    apt-get install -y kubelet kubeadm kubectl docker.io
    apt-mark hold kubelet kubeadm kubectl
    ```
  Files to check; /var/lib/kubelet/kubeadm-flags.env

##### VBOX notes

1. If using linked clones in vbox/kvm and systemd
   Delete `/etc/machine-id`
   **run**: *systemd-machine-id-setup*



##### Standup kubernetes

1. Standup Kubernetes master - Swap needs to be turned off (`swapoff -a`).
    ```bash
    kubeadm config images pull
    kubeadm  init --pod-network-cidr=10.100.0.0/16
    #Optional if needed for example static subnet or custom dns.
    --apiserver-advertise-address=10.0.0.1
    --service-cidr=10.42.0.0/16
    --service-dns-domain=scriptninjas.local
    ```

1. **WAIT** for pods to deploy on primary node (`master`)
This creates the network overlay
    ```bash
    kubectl apply -f setup/kubernetes/networking/rbac-kdd.yaml
    kubectl apply -f setup/kubernetes/networking/calico.yaml
    ```

##### Install helm

Command: `snap install helm --classic`

Helm needs a custom user in kubeadm environment and a RBAC rules for clusterroles.
*Note: this creates a service account for helm in the kube-system space.*
`kubectl apply -f setup/kubernetes/helm-service.yaml`

Now create the necessary helm client and tiller service.
Command: `helm init --service-account helm`

To remove/redo Helm setup you need to run these commands:
```bash
kubectl delete deployment tiller-deploy --namespace=kube-system
kubectl delete service tiller-deploy --namespace=kube-system
rm -rf ~/.helm/
```



##### Install cert-manager

Prepare cert-manager install

```bash
git clone https://github.com/helm/charts.git
cd charts/stable/cert-manager
helm dep build
cd ../../
```

Install cert-manager into your kubes cluster by running:
` helm install --name cert-manager --namespace kube-system stable/cert-manager`

*Note:* This requires your cluster to have RBAC enabled.

##### Install certissuer

Run command for certissuer. `kubectl apply -f setup/kubernetes/certissuer.yaml`

##### Install ingress controller

Run commands:
```bash
git clone https://github.com/nginxinc/kubernetes-ingress/
git checkout v1.4.3
cd kubernetes-ingress/deployments/helm-chart
```
To actually install the ingress controller run command: `helm install --name nginx-ingress .`

Apply the "on premise" patch for an ingress controller, which is exposing the controller thru a nodeport.
command: `kubectl apply -f setup/kubernetes/ingress-nodeport.yaml`

##### Install your service for testing

And now deploy the sample pod and nodeport service: `kubectl apply -f pods/nginx-service.yaml`

##### Install kubernetes dashboard - *work in progress*

```bash
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
kubectl proxy --address='0.0.0.0' --accept-hosts='.*'
http://ip:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```


###### URLs
URLS to note:
1. https://kubernetes.io/docs/concepts/
1. https://kubernetes.io/docs/setup/independent/install-kubeadm/
1. https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#tear-down
1. https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/installation.md
1. https://github.com/containous/traefik
1. https://github.com/jcmoraisjr/haproxy-ingress
1. https://blog.heptio.com/declarative-configuration-for-kubeadm-heptioprotip-7304eb32641
1. https://netplan.io/examples
1. https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/
1. https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/
1. https://kubernetes.io/blog/2015/10/some-things-you-didnt-know-about-kubectl_28/
1. https://akomljen.com/kubernetes-nginx-ingress-controller/
1. https://labs.play-with-k8s.com/
