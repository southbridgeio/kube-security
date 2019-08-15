## Все работы все еще проводим с первого мастера

Переходим в директорию с практикой.
```bash
cd ~/slurm/practice/4.secure-and-highlyavailable-apps/psp
```

Cоздаем тестового пользователя
```bash
kubectl create serviceaccount user --namespace=default
kubectl create rolebinding user --clusterrole=edit --serviceaccount=default:user
```

Проверяем права созданного пользователя
```bash
kubectl get po --as=system:serviceaccount:default:user --all-namespaces
```

Куб должен запрещать просмотр всех подов во всем кластере

Пробуем хакнуть ограничения
Запускаем под хацкера, так же от имени пользователя user
```bash
kubectl create -f hackers-pod.yaml --as=system:serviceaccount:default:user
```

Проверяем под
```bash
kubectl get pod
```
Видим
```bash
NAME          READY   STATUS    RESTARTS   AGE
hackers-pod   1/1     Running   0          1m
```

Заходим внутрь и осматриваемся (все это делаем от имени пользователя без прав админа)
```bash
kubectl exec -t -i hackers-pod --as=system:serviceaccount:default:user bash
```

Например
```bash
cat /host/etc/kubernetes/admin.conf
```
Радуемся полученному доступу к сертификатам кластера

Удаляем хацкерский под
```bash
kubectl delete -f hackers-pod.yaml
```

Применяем psp и rbac
```bash
kubectl apply -f psp-system.yaml
kubectl apply -f rbac-system.yaml -n kube-system
kubectl apply -f psp-default.yaml -f rbac-default.yaml
```

Смотрим что получилось
```bash
kubectl get psp
```
Видим
```bash
NAME      PRIV    CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
default   false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            configMap,emptyDir,projected,secret,downwardAPI,persistentVolumeClaim
system    true           RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            *
```

Обновляем конфигурацию kube-api
> Инструкции преведены для кластеров установленных с помощью kubeadm.
> Для изменения флагов запуска kube-api для других инсталяций кластера обращайтесь к инструкциям своего установщика/провайдера.

```bash
kubectl edit configmap --namespace=kube-system kubeadm-config
```

И добавляем

```yaml
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
# вот отсюда
    enable-admission-plugins: PodSecurityPolicy
```

Далее на каждом из трех мастеров выполняем

```bash
kubeadm upgrade apply v1.14.1 -y
```

Пробуем еще раз создать хацкерский под
```bash
kubectl create -f hackers-pod.yaml --as=system:serviceaccount:default:user
```

В ответ должны увидеть что то типа
```bash
Error from server (Forbidden): error when creating "hackers-pod.yaml": pods "hackers-pod" is forbidden: unable to validate against any pod security policy: [spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used spec.securityContext.hostPID: Invalid value: true: Host PID is not allowed to be used spec.volumes[0]: Invalid value: "hostPath": hostPath volumes are not allowed to be used]
```

Чистим за собой кластер
Созданные PSP и RBAС НЕ удаляем!
```bash
kubectl delete -f hackers-pod.yaml
kubectl delete serviceaccount user
kubectl delete rolebinding user
```
> Вернет ошибку, если прошлая команда как и задумано не смогла создать под
