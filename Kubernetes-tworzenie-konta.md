# Instrukcja tworzenia konta

Przykład dla ShinyProxy. Przetestowany na klastrze zainstalowanym za pomocą Kubespray.

Polecenia należy wydać jako użytkownik `root` na tym węźle klastra, na którym istnieje `/etc/kubernetes/ssl`.  
Być może będzie to jest jeden w węzłów master. Nie zawsze musi to być pierwszy z węzłów klastra.

```shell
$ cd "${HOME}"
$ mkdir -p kubernetes-accounts/shinyproxy1user1
$ cd kubernetes-accounts/shinyproxy1user1
$ openssl genrsa -out shinyproxy1user1.key 2048
$ openssl req -new -key shinyproxy1user1.key -out shinyproxy1user1.csr -subj "/CN=shinyproxy1user1/O=shinyproxy1"
$ openssl x509 -req -in "${HOME}/kubernetes-accounts/shinyproxy1user1/shinyproxy1user1.csr" -CA /etc/kubernetes/ssl/ca.crt -CAkey /etc/kubernetes/ssl/ca.key -CAcreateserial -out "${HOME}/kubernetes-accounts/shinyproxy1user1/shinyproxy1user1.crt" -days 4900
```

Robimy backup pliku `~/.kube/config`.
```shell
$ cp -v ~/.kube/config ~/.kube/config.$(date +%Y%m%d-%H%M%S).backup
```

Pobieramy klucz i certyfikat na maszynę nadzorcy (kubespray + Vagrant):
```shell
jkowalski@localhost:/kubespray$ vagrant ssh-config > ssh-config
jkowalski@localhost:/kubespray$ ssh -F ./ssh-config vagrant@k8s-1 sudo tar -C /root -c kubernetes-accounts | tar -C "${HOME}" -vx
kubernetes-accounts/
kubernetes-accounts/shinyproxy1user1/
kubernetes-accounts/shinyproxy1user1/shinyproxy1user1.crt
kubernetes-accounts/shinyproxy1user1/shinyproxy1user1.csr
kubernetes-accounts/shinyproxy1user1/shinyproxy1user1.key
```

Dodajemy nowego użytkownika do `~/.kube/config`:
```shell
$ kubectl config set-credentials shinyproxy1user1 --client-certificate="${HOME}/kubernetes-accounts/shinyproxy1user1/shinyproxy1user1.crt" --client-key="${HOME}/kubernetes-accounts/shinyproxy1user1/shinyproxy1user1.key"
$ kubectl config set-context shinyproxy1user1-context --cluster=kubernetes --namespace=shinyproxy1 --user=shinyproxy1user1
```

```shell
$ kubectl --context=shinyproxy1user1-context get pods
Error from server (Forbidden): pods is forbidden: User "shinyproxy1user1" cannot list resource "pods" in API group "" in the namespace "shinyproxy1"
```

Tak, powyższe polecenie powinno zwrócić błąd, bo nie ustawiliśmy jeszcze uprawnień.
