# MetalLB

Instrukcja instalacji MetalLB na Kubernetesie.

Dzięki MetalLB możemy tworzyć w Kubernetesie usługi typu `LoadBalancer` i MetalLB przydzieli im dostępne z zewnątrz klastra adresy IP należące do podanego przez nas zakresu.

```shell
$ kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.7.3/manifests/metallb.yaml
namespace/metallb-system created
serviceaccount/controller created
serviceaccount/speaker created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
role.rbac.authorization.k8s.io/config-watcher created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/config-watcher created
daemonset.apps/speaker created
deployment.apps/controller created
```

```shell
$ kubectl get pods -n metallb-system
NAME                          READY   STATUS    RESTARTS   AGE
controller-7cc9c87cfb-fj4rm   1/1     Running   0          26s
speaker-8pqrg                 1/1     Running   0          26s
speaker-fk5gj                 1/1     Running   0          26s
speaker-jcsbx                 1/1     Running   0          26s
```

Trzeba określić jaki jest zakres adresów IP, które mogą być przydzielane przez MetalLB:
```shell
$ cat << EOF > metallb-config-map-1.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 10.11.11.160-10.11.11.169
EOF
$ kubectl apply -f metallb-config-map-1.yaml
```

```shell
kubectl logs -l component=speaker -n metallb-system
```

# Dokumentacja

* [https://metallb.universe.tf/tutorial/layer2/](https://metallb.universe.tf/tutorial/layer2/)
* [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/)
* [https://metallb.universe.tf/configuration/](https://metallb.universe.tf/configuration/)
