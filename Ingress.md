# Ingress

## Spis treści

<!-- MarkdownTOC autolink="true" -->

- [Instalacja](#instalacja)
- [Konfiguracja](#konfiguracja)
- [Ingress z SSL/TLS](#ingress-z-ssltls)

<!-- /MarkdownTOC -->

## Instalacja

Instalacja Ingress Nginx.

Dokumentacja:

* [https://kubernetes.github.io/ingress-nginx/deploy/](https://kubernetes.github.io/ingress-nginx/deploy/)
* [https://kubernetes.github.io/ingress-nginx/deploy/baremetal/](https://kubernetes.github.io/ingress-nginx/deploy/baremetal/)
* [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)

```shell
% kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
```

```shell
% kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
```

W usłudze `ingress-nginx` trzeba zmienić domyślny typ `NodePort` na `LoadBalancer`:

```shell
$ cat << EOF > ingress-nginx.yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 31149
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    nodePort: 30480
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  sessionAffinity: None
  type: LoadBalancer
EOF
$ kubectl apply -f ingress-nginx.yml
```

Sprawdzamy czy się udało:

```shell
% kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx
NAMESPACE       NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx   nginx-ingress-controller-689498bc7c-5mp52   1/1     Running   0          105s
```

```shell
% kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-689498bc7c-5mp52   1/1     Running   0          4h34m
```

## Konfiguracja

Żeby można było użyć Ingressa, potrzebujemy najpierw działającego poda z jaką aplikacją udostepniającą HTTP.  
Jak już będziemy ją mieć, to to musimy utworzyć dla tej aplikacji (poda) usługę.  
`kubernetes-bootcamp-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-bootcamp
spec:
  selector:
    run: kubernetes-bootcamp
  ports:
   - protocol: TCP
     port: 8080
```

Tworzymy obiekt typu Ingress.

`myingressresource1.yml`:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  labels:
    app: myingressresource1
  name: myingressresource1
  namespace: default
spec:
  rules:
  - host: bootcamp.lan
    http:
      paths:
      - backend:
          serviceName: kubernetes-bootcamp
          servicePort: 8080
        path: /
```

Sprawdzamy stan poszczególnych obiektów:

```shell
$ kubectl -n ingress-nginx get svc
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx   LoadBalancer   10.233.54.42   192.168.121.201   80:31149/TCP,443:30480/TCP   29m
```

```shell
% kubectl get pods --all-namespaces -o wide
NAMESPACE        NAME                                        READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
default          kubernetes-bootcamp-6bf84cb898-7q4rm        1/1     Running   1          5h18m   10.233.66.9    k8s-3   <none>           <none>
ingress-nginx    nginx-ingress-controller-689498bc7c-5mp52   1/1     Running   0          5h6m    10.233.67.8    k8s-4   <none>           <none>
kube-system      coredns-d5b697bd8-cn8zj                     1/1     Running   1          22h     10.233.67.5    k8s-4   <none>           <none>
kube-system      coredns-d5b697bd8-hqn4w                     1/1     Running   2          22h     10.233.66.10   k8s-3   <none>           <none>
kube-system      dns-autoscaler-8666ffc6c6-dbbgl             1/1     Running   1          23h     10.233.67.6    k8s-4   <none>           <none>
kube-system      haproxy-k8s-3                               1/1     Running   2          23h     172.17.8.103   k8s-3   <none>           <none>
kube-system      haproxy-k8s-4                               1/1     Running   1          23h     172.17.8.104   k8s-4   <none>           <none>
kube-system      kube-apiserver-k8s-1                        1/1     Running   1          23h     172.17.8.101   k8s-1   <none>           <none>
kube-system      kube-apiserver-k8s-2                        1/1     Running   1          23h     172.17.8.102   k8s-2   <none>           <none>
kube-system      kube-controller-manager-k8s-1               1/1     Running   2          23h     172.17.8.101   k8s-1   <none>           <none>
kube-system      kube-controller-manager-k8s-2               1/1     Running   2          23h     172.17.8.102   k8s-2   <none>           <none>
kube-system      kube-flannel-7xn9l                          2/2     Running   3          23h     172.17.8.104   k8s-4   <none>           <none>
kube-system      kube-flannel-qjs4c                          2/2     Running   5          23h     172.17.8.103   k8s-3   <none>           <none>
kube-system      kube-flannel-srzzd                          2/2     Running   3          23h     172.17.8.102   k8s-2   <none>           <none>
kube-system      kube-flannel-x7jdj                          2/2     Running   2          23h     172.17.8.101   k8s-1   <none>           <none>
kube-system      kube-proxy-9668f                            1/1     Running   1          22h     172.17.8.102   k8s-2   <none>           <none>
kube-system      kube-proxy-n7d4n                            1/1     Running   1          22h     172.17.8.104   k8s-4   <none>           <none>
kube-system      kube-proxy-w5knl                            1/1     Running   2          22h     172.17.8.103   k8s-3   <none>           <none>
kube-system      kube-proxy-wwvx4                            1/1     Running   1          22h     172.17.8.101   k8s-1   <none>           <none>
kube-system      kube-scheduler-k8s-1                        1/1     Running   3          23h     172.17.8.101   k8s-1   <none>           <none>
kube-system      kube-scheduler-k8s-2                        1/1     Running   1          23h     172.17.8.102   k8s-2   <none>           <none>
kube-system      kubernetes-dashboard-7f8b6c9b7d-5qsqd       1/1     Running   2          23h     10.233.66.8    k8s-3   <none>           <none>
kube-system      nodelocaldns-6fz5d                          1/1     Running   1          23h     172.17.8.102   k8s-2   <none>           <none>
kube-system      nodelocaldns-6s24r                          1/1     Running   2          23h     172.17.8.103   k8s-3   <none>           <none>
kube-system      nodelocaldns-b6258                          1/1     Running   1          23h     172.17.8.104   k8s-4   <none>           <none>
kube-system      nodelocaldns-j8hxc                          1/1     Running   1          23h     172.17.8.101   k8s-1   <none>           <none>
metallb-system   controller-7cc9c87cfb-stv7d                 1/1     Running   0          5h17m   10.233.67.7    k8s-4   <none>           <none>
metallb-system   speaker-f4m68                               1/1     Running   0          5h17m   172.17.8.104   k8s-4   <none>           <none>
metallb-system   speaker-szmrr                               1/1     Running   1          5h17m   172.17.8.103   k8s-3   <none>           <none>
```

```shell
% kubectl get ingress --all-namespaces -o wide
NAMESPACE   NAME                 HOSTS          ADDRESS           PORTS   AGE
default     myingressresource1   bootcamp.lan   192.168.121.201   80      5h15m
```

```shell
$ kubectl get ingress myingressresource1 -o yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{},"labels":{"app":"myingressresource1"},"name":"myingressresource1","namespace":"default"},"spec":{"rules":[{"host":"bootcamp.lan","http":{"paths":[{"backend":{"serviceName":"kubernetes-bootcamp","servicePort":8080},"path":"/"}]}}]},"status":{"loadBalancer":{"ingress":[{"ip":"192.168.121.200"}]}}}
  creationTimestamp: "2019-04-16T08:33:24Z"
  generation: 1
  labels:
    app: myingressresource1
  name: myingressresource1
  namespace: default
  resourceVersion: "144823"
  selfLink: /apis/extensions/v1beta1/namespaces/default/ingresses/myingressresource1
  uid: 4cf64005-6022-11e9-9a9d-525400660ed6
spec:
  rules:
  - host: bootcamp.lan
    http:
      paths:
      - backend:
          serviceName: kubernetes-bootcamp
          servicePort: 8080
        path: /
status:
  loadBalancer:
    ingress:
    - ip: 192.168.121.201
```

```shell
% kubectl get services --all-namespaces
NAMESPACE       NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                      AGE
default         kubernetes             ClusterIP      10.233.0.1      <none>            443/TCP                      23h
default         kubernetes-bootcamp    ClusterIP      10.233.18.144   <none>            8080/TCP                     5h20m
ingress-nginx   ingress-nginx          LoadBalancer   10.233.54.42    192.168.121.201   80:31149/TCP,443:30480/TCP   34m
kube-system     coredns                ClusterIP      10.233.0.3      <none>            53/UDP,53/TCP,9153/TCP       23h
kube-system     kubernetes-dashboard   ClusterIP      10.233.10.220   <none>            443/TCP                      23h
```

```shell
% curl -D- http://192.168.121.201 -H 'Host: bootcamp.lan'
HTTP/1.1 200 OK
Server: nginx/1.15.10
Date: Tue, 16 Apr 2019 13:56:37 GMT
Content-Type: text/plain
Transfer-Encoding: chunked
Connection: keep-alive
Vary: Accept-Encoding

Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-6bf84cb898-7q4rm | v=1
```

Aktualną konfigurację Ingressa odczytałem sobie tak:

```shell
kubectl get ingress --all-namespaces -o yaml
```

Ostateczny plik konfiguracjyny Ingressa (Nginx) jest budowany na podstawie obiektów typu Ingress ze wszystkich namepsace.

## Ingress z SSL/TLS

Przykład:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    # shinyproxy wymaga timeoutu 20 dni
    ingress.kubernetes.io/proxy-read-timeout: 1728000
  labels:
    app: myingressresource1
  name: myingressresource1
  namespace: default
spec:
  rules:
  - host: nginx.kubernetes-test.example.com
    http:
      paths:
      - backend:
          serviceName: servicenginx
          servicePort: 80
        path: /
  - host: bootcamp.kubernetes-test.example.com
    http:
      paths:
      - backend:
          serviceName: servicebootcamp
          servicePort: 8080
        path: /
  tls:
  - hosts:
    - nginx.kubernetes-test.example.com
    - bootcamp.kubernetes-test.example.com
    secretName: mysslcert1
```
