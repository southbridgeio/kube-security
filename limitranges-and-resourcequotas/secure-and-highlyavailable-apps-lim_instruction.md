## Все работы все еще проводим с первого мастера

Переходим в директорию с практикой.
```bash
cd ~/slurm/practice/4.secure-and-highlyavailable-apps/limitranges-and-resourcequotas
```

Создаем нэймспэйс
```bash
kubectl create ns test
```

Создаем в нем limitrange и resourcequota
```bash
kubectl apply -f limitrange.yaml -f resourcequota.yaml -n test
```

Смотрим что получилось
```bash
kubectl describe ns test
```

Запускаем deployment

```bash
kubectl apply -f deployment.yaml -n test
```

И еще раз заглядываем в описание ns.
Видим что количество доступных ресурсов начало уменьшаться.

Правим deployment.
Теперь в лимитах у него будет 1 cpu.
Применяем изменения.
Сотрим в дескрайб последнего репликасета.
Там что то типа такого
```bash
  Warning  FailedCreate  20s (x4 over 38s)  replicaset-controller  (combined from similar events): Error creating: pods "nginx-6579b4dfb7-n7wmd" is forbidden: [maximum cpu usage per Container is 1, but limit is 2., cpu max limit to request ratio per Pod is 2, but provided ratio is 20.000000.]
```

Чистим за собой кластер
```bash
kubectl delete ns test
```
