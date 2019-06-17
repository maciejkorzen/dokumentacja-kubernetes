# Wstęp

Instrukcja instalacji klastra Kubernetes za pomocą Kubespray

Procedura została przygotowana i przetestowana na Ubuntu 16.04. Użyłem Vagranta z libvirtd.  
Jeśli w instrukcji nie zaznaczono inaczej, to polecenia wykonywałem jako zwykły użytkownik systemowy (nie-`root`).

# Instrukcja

Jako `root`:
```shell
sudo apt install build-essential zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev libssl-dev libsqlite{0,3}-dev make wget curl llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev
cd /opt
umask 002
git clone https://github.com/yyuu/pyenv.git
cat << EOF > "/opt/pyenv1/sh-source"
export PYENV_ROOT="/opt/pyenv1"
export PATH="\$PYENV_ROOT/bin:\$PATH"
eval "\$(pyenv init -)"
EOF
echo "source /opt/pyenv1/sh-source" >> /etc/bash.bashrc
```
Uruchomić nową powłokę.
W nowej powłoce jako `root`:
```shell
pyenv install 3.7.3
pyenv rehash
pyenv global 3.7.3
sudo apt install python-pip python-paramiko python-yaml python-jinja2 python-httplib2 python-six libyaml-dev python-virtualenv
```

Jako zwykły użytkownik:
```shell
cd "${HOME}"
mkdir venv
cd venv
python3 -m venv ansible-2.7.10
cd ansible-2.7.10/bin
source activate
python -m pip install --upgrade pip wheel setuptools paramiko netaddr
pip install ansible==2.7.10.0
cd "${HOME}"
git clone https://github.com/kubernetes-sigs/kubespray
cd kubespray
pip install -r requirements.txt
cp -rfp inventory/sample inventory/mycluster
```

Wprowadzamy modyfikacje w pliku `Vagrantfile`:
```diff
jkowalski@localhost:~/kubespray% diff -uNr Vagrantfile Vagrantfile.org
--- Vagrantfile 2019-04-15 09:43:18.677202241 +0000
+++ Vagrantfile.org     2019-04-12 16:03:31.754225231 +0000
@@ -28,11 +28,11 @@
 }

 # Defaults for config options defined in CONFIG
-$num_instances = 4
+$num_instances = 3
 $instance_name_prefix = "k8s"
 $vm_gui = false
-$vm_memory = 16384
-$vm_cpus = 8
+$vm_memory = 2048
+$vm_cpus = 1
 $shared_folders = {}
 $forwarded_ports = {}
 $subnet = "172.17.8"
@@ -41,17 +41,17 @@
 # Setting multi_networking to true will install Multus: https://github.com/intel/multus-cni
 $multi_networking = false
 # The first three nodes are etcd servers
-$etcd_instances = 3
+$etcd_instances = $num_instances
 # The first two nodes are kube masters
-$kube_master_instances = 2
+$kube_master_instances = $num_instances == 1 ? $num_instances : ($num_instances - 1)
 # All nodes are kube nodes
 $kube_node_instances = $num_instances
 # The following only works when using the libvirt provider
 $kube_node_instances_with_disks = false
-$kube_node_instances_with_disks_size = "40G"
+$kube_node_instances_with_disks_size = "20G"
 $kube_node_instances_with_disks_number = 2
 $override_disk_size = false
-$disk_size = "40GB"
+$disk_size = "20GB"
 $local_path_provisioner_enabled = false
 $local_path_provisioner_claim_root = "/opt/local-path-provisioner/"

@@ -65,7 +65,7 @@

 $box = SUPPORTED_OS[$os][:box]
 # if $inventory is not set, try to use example
-$inventory = "inventory/mycluster" if ! $inventory
+$inventory = "inventory/sample" if ! $inventory
 $inventory = File.absolute_path($inventory, File.dirname(__FILE__))

 # if $inventory has a hosts.ini file use it, otherwise copy over
@@ -202,8 +202,8 @@
           ansible.host_vars = host_vars
           #ansible.tags = ['download']
           ansible.groups = {
-            "etcd" => ["#{$instance_name_prefix}-[1:3]"],
-            "kube-master" => ["#{$instance_name_prefix}-[1:2]"],
+            "etcd" => ["#{$instance_name_prefix}-[1:#{$etcd_instances}]"],
+            "kube-master" => ["#{$instance_name_prefix}-[1:#{$kube_master_instances}]"],
             "kube-node" => ["#{$instance_name_prefix}-[1:#{$kube_node_instances}]"],
             "k8s-cluster:children" => ["kube-master", "kube-node"],
           }
```

W pliku `inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml` ustawiamy:
```
kubeconfig_localhost: true
```

Jeśli maszyny wirtualne będą miały dwie karty sieciowe i druga z nich będzie używana przez klaster Kubernetes (VirtualBox i libvirtd domyślnie tak mają), to w pliku `inventory/mycluster/group_vars/k8s-cluster/k8s-net-flannel.yml` ustawiamy:
```
flannel_interface: 'eth1'
```

W najlepszym wypadku wystarczy teraz zrobić `vagrant up` i na maszynach wirtualnych zostanie zainstalowany i skonfigurowany Kubernetes.  
Ale czasami pojawiają się różne błędy (niezależne od nas) podczas uruchamiania playbooka Ansible. Np. problemy sieciowe podczas pobierania plików lub pakietów. Więc praktycznej jest wydać poniższe polecenie i pójść na obiad. :-)
```
vagrant up; for i in {1..9}; do vagrant provision; sleep 60; done
```
Jak wrócimy z obiadu, to Kubernetes powinien już działać.

Poniższy komunikat błędu pojawił mi się gdy ilość RAM-u, którą sumarycznie przydzieliłem maszynom wirtualnym, była za duża:
```
==> k8s-6: An error occurred. The error will be shown after all tasks complete.
==> k8s-2: An error occurred. The error will be shown after all tasks complete.
An error occurred while executing multiple actions in parallel.
Any errors that occurred are shown below.
An error occurred while executing the action on the 'k8s-2'
machine. Please handle this error then try again:
There was an error talking to Libvirt. The error message is shown
below:
Call to virDomainCreateWithFlags failed: internal error: process exited while connecting to monitor: warning: host doesn't support requested feature: CPUID.01H:EDX.ds [bit 21]
warning: host doesn't support requested feature: CPUID.01H:EDX.acpi [bit 22]
warning: host doesn't support requested feature: CPUID.01H:EDX.ht [bit 28]
warning: host doesn't support requested feature: CPUID.01H:EDX.tm [bit 29]
warning: host doesn't support requested feature: CPUID.01H:EDX.pbe [bit 31]
warning: host doesn't support requested feature: CPUID.01H:ECX.dtes64 [bit 2]
warning: host doesn't support requested feature: CPUID.01H:ECX.monitor [bit 3]
warning: host doesn't support requested feature: CPUID.01H:ECX.ds_cpl [bit 4]
warning: host doesn't support requested feature: CPUID.01H:ECX.smx [bit 6]
warning: host doesn't support requested feature: CPUID.01H:ECX.est [bit 7]
warning: host doesn't support requested feature: CPUID.01H:ECX.tm2 [bit 8]
warning: host doesn't support requested feature: CPUID.01H:ECX.xtpr [bit 14]
warning: host doesn't support requested feature: CPUID.01H:ECX.
An error occurred while executing the action on the 'k8s-6'
machine. Please handle this error then try again:
There was an error talking to Libvirt. The error message is shown
below:
Call to virDomainCreateWithFlags failed: internal error: process exited while connecting to monitor: warning: host doesn't support requested feature: CPUID.01H:EDX.ds [bit 21]
warning: host doesn't support requested feature: CPUID.01H:EDX.acpi [bit 22]
warning: host doesn't support requested feature: CPUID.01H:EDX.ht [bit 28]
warning: host doesn't support requested feature: CPUID.01H:EDX.tm [bit 29]
warning: host doesn't support requested feature: CPUID.01H:EDX.pbe [bit 31]
warning: host doesn't support requested feature: CPUID.01H:ECX.dtes64 [bit 2]
warning: host doesn't support requested feature: CPUID.01H:ECX.monitor [bit 3]
warning: host doesn't support requested feature: CPUID.01H:ECX.ds_cpl [bit 4]
warning: host doesn't support requested feature: CPUID.01H:ECX.smx [bit 6]
warning: host doesn't support requested feature: CPUID.01H:ECX.est [bit 7]
warning: host doesn't support requested feature: CPUID.01H:ECX.tm2 [bit 8]
warning: host doesn't support requested feature: CPUID.01H:ECX.xtpr [bit 14]
warning: host doesn't support requested feature: CPUID.01H:ECX.
```

Teraz trzeba zainstalować `kubectl`. Instrukcja: [https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl).

Następnie:
```shell
mkdir ~/.kube
cp ./inventory/mycluster/artifacts/admin.conf ~/.kube/config
kubectl get pods --all-namespaces
```

Powinniśmy zobaczyć listę podów. Przykład:
```
jkowalski@localhost:~/kubespray% kubectl get pods --all-namespaces
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE
kube-system   coredns-d5b697bd8-cn8zj                 1/1     Running   0          17h
kube-system   coredns-d5b697bd8-hqn4w                 1/1     Running   0          17h
kube-system   dns-autoscaler-8666ffc6c6-dbbgl         1/1     Running   0          17h
kube-system   haproxy-k8s-3                           1/1     Running   0          17h
kube-system   haproxy-k8s-4                           1/1     Running   0          17h
kube-system   kube-apiserver-k8s-1                    1/1     Running   0          17h
kube-system   kube-apiserver-k8s-2                    1/1     Running   0          17h
kube-system   kube-controller-manager-k8s-1           1/1     Running   0          17h
kube-system   kube-controller-manager-k8s-2           1/1     Running   0          17h
kube-system   kube-flannel-7xn9l                      2/2     Running   0          17h
kube-system   kube-flannel-qjs4c                      2/2     Running   0          17h
kube-system   kube-flannel-srzzd                      2/2     Running   0          17h
kube-system   kube-flannel-x7jdj                      2/2     Running   0          17h
kube-system   kube-proxy-9668f                        1/1     Running   0          17h
kube-system   kube-proxy-n7d4n                        1/1     Running   0          17h
kube-system   kube-proxy-w5knl                        1/1     Running   0          17h
kube-system   kube-proxy-wwvx4                        1/1     Running   0          17h
kube-system   kube-scheduler-k8s-1                    1/1     Running   0          17h
kube-system   kube-scheduler-k8s-2                    1/1     Running   0          17h
kube-system   kubernetes-dashboard-7f8b6c9b7d-5qsqd   1/1     Running   0          17h
kube-system   nodelocaldns-6fz5d                      1/1     Running   0          17h
kube-system   nodelocaldns-6s24r                      1/1     Running   0          17h
kube-system   nodelocaldns-b6258                      1/1     Running   0          17h
kube-system   nodelocaldns-j8hxc                      1/1     Running   0          17h
```

Wszystkie pody powinny być w stanie `Running`. Jeśli nie są, to znaczy, że coś poszło nie tak.

Żeby zalogować się do jednej z maszyn wirtualnych, należy wydać polecenie:
```
vagrant ssh k8s-X
```

Gdzie `X` to numer maszyny wirtualnej.
