# This file includes the steps for Kubernetes version 1.28.6-1.1.

1) Disable ufw on all nodes.

systemctl stop ufw
systemctl disable ufw

2) Edit /etc/hostst to include all the nodes on all nodes.

3) Setup DNS for all the nodes.

4) Configure passwordless SSH from master nodes to worker nodes.

5) Verify the init system (initd or SystemV) for all the nodes. Accordingly the kubelet configuration is going to be modified soon. If it systemd then cgroup driver should be systemd. There are two possibilities; cgroupfs and systemd.

root@drizzt:~# cat /proc/1/comm
systemd
root@drizzt:~#
root@drizzt:~# ls -l /sbin/init
lrwxrwxrwx 1 root root 20 Apr  7  2022 /sbin/init -> /lib/systemd/systemd
root@drizzt:~#
root@drizzt:~# systemctl --version
systemd 249 (249.11-0ubuntu3)
+PAM +AUDIT +SELINUX +APPARMOR +IMA +SMACK +SECCOMP +GCRYPT +GNUTLS -OPENSSL +ACL +BLKID +CURL +ELFUTILS -FIDO2 +IDN2 -IDN +IPTC +KMOD +LIBCRYPTSETUP -LIBFDISK +PCRE2 -PWQUALITY -P11KIT -QRENCODE +BZIP2 +LZ4 +XZ +ZLIB +ZSTD -XKBCOMMON +UTMP +SYSVINIT default-hierarchy=unified
root@drizzt:~#

6) Install containerd on all nodes.

    a) Install overlay and br_filter kernel modules.

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

    b) Enable IP forwarding and bridging.

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot.
sudo sysctl --system

    c) Install contanerd

https://github.com/containerd/containerd/blob/main/docs/getting-started.md

# Follow Option 1.

# Note that init.d service configuration files are stored in /lib/systemd/system/.

    d) Install runc.

    e) Install CRI plugins.

    f) Create the default containerd configuration file.

mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml

    g) Configure the systemd cgroup driver. 

# To use the systemd cgroup driver in /etc/containerd/config.toml with runc, set the following in the file:

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

# Restart containerd.
systemctl restart containerd

7) Remember to set systemd init service in Kubelet configuration section in the kubeadm init file.

# kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.21.0
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd

9) Install Kubernetes repo on all nodes.

https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

10 ) Install kubeadm, kubelet, kubectl on all nodes.

11 ) Prepare kubeadm init file on the master node.

https://v1-28.docs.kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3/#kubeadm-k8s-io-v1beta3-ClusterConfiguration

# Example init file for both "kubeadm init" and "kubeadm join":

apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
localAPIEndpoint:
  advertiseAddress: "192.168.44.10"
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "11.11.11.0/24"
  dnsDomain: "realm.com"
clusterName: "fr"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock

12) Run "kubeadm init" on the master node.

# You had better take a snapshot of the master node if it is a virtual machine.

# First run it in dry-mode to see if there are any issues.
kubeadm init --config kubeadm_init_custom.yaml --dry-run

# If there are no issues run it for real on the master node.
kubeadm init --config kubeadm_init_custom.yaml

13) Export KUBECONFIG pointing at /etc/kubernetes/admin.conf at boot time.

cat << EOF > /etc/profile.d/kubernetes_config.sh
> export KUBECONFIG=/etc/kubernetes/admin.conf
> EOF
chmod 755 /etc/profile.d/kubernetes_config.sh
source /etc/profile

14) Enable kubectl completion

echo "source <(kubectl completion bash)" >> /etc/profile.d/kubernetes_config.sh
source /etc/profile

15) Join the worker nodes.

kubeadm join 192.168.44.10:6443 --cri-socket unix:///var/run/containerd/containerd.sock --token zx24m3.lwu8hu3kzxgzs4go --discovery-token-ca-cert-hash sha256:fdd5ecc3cffc1145f05534109a5fe035b3e0331c7a714b803b40848f3483a147

16) Install networking. Example Weavenet file is provided in the repository as "weave-daemonset-k8s-1.11.yaml".

kubectl apply -f weave-daemonset-k8s-1.11.yaml

17) Verify the cluster is ready schedule nodes.

kubectl cluster-info
kubectl get nodes --namespace kube-system

# Wait till "Status" turns to "Ready".

18) Untaint the master node if you want to schedule pods on that node too.

kubectl describe nodes drizzt  | grep -i taint
> Taints:             node-role.kubernetes.io/control-plane:NoSchedule

kubectl taint node drizzt node-role.kubernetes.io/control-plane-
node/drizzt untainted

19) Test the cluster by running a pod.

kubectl run --image nginx nginx
pod/nginx created

kubectl get pods
NAME    READY   STATUS              RESTARTS   AGE
nginx   0/1     ContainerCreating   0          4s

kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          15s

# Run a hostname command from withing pod.
kubectl exec --tty --stdin nginx /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@nginx:/#
root@nginx:/# hostname
nginx
root@nginx:/#exit
exit

# Delete the pod.
kubectl delete pod nginx
pod "nginx" deleted




