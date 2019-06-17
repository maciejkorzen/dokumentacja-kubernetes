# Instalacja Kubernetesa

Najpierw instalujemy Kubernetesa. Np. za pomocą minikube lub kubespray.

Rzeczy do sprawdzenia po instalacji

* Czy wszystkie pody mają stan `Running`?
* `dmesg`
* Logi systemowe.

Należy zwrócić uwagę czy w logach systemowych nie ma komunikatu podobnego do poniższego.
```
Apr 15 02:30:04 k8s-4 kubelet[1481]: E0415 02:30:04.585583    1481 dns.go:132] Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 4.2.2.1 4.2.2.2 208.67.220.220
```

Jeśli jest, to możemy zmienić konfigurację `resolved`, aby zawierała tylko jeden resolver DNS w następujący sposób:
```shell
jkowalski@localhost:~/kubespray% vagrant ssh-config > ssh-config
jkowalski@localhost:~/kubespray% egrep '^Host' ssh-config | awk '{ print $2; }' | sort | while read i; do echo "=== $i ==="; ssh -F ssh-config "${i}" "sudo sed -i -r -e 's,^DNS=.*,DNS=9.9.9.9,g' /etc/systemd/resolved.conf; sudo systemctl restart systemd-resolved.service" < /dev/null; echo; done
```

Po wykonaniu tych czynności trzeba zrestartować maszyny klastra.

# Przykładowe/testowe aplikacje

```shell
kubectl run kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1 --port=8080
```

