# Kubernetes

- [Краткая справка](overview.md) — архитектура кластера, основные объекты, сеть, хранение и жизненный цикл Pod.
- [Вопросы и ответы](theory.md) — формулировки для собеседований и учебных защит.

Раздел опирается на семинар «Основы Kubernetes» и дополняет его эксплуатационным минимумом для backend-разработчика.

## Порядок изучения

1. Зачем после контейнеров нужен оркестратор.
2. Control plane и worker nodes: `kube-apiserver`, `etcd`, controller manager, scheduler, kubelet, kube-proxy.
3. Pod и controllers: ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob.
4. Service discovery и доступ: Service, DNS, headless Service, Gateway API.
5. ConfigMap, Secret, volumes, PV, PVC и StorageClass.
6. Requests/limits, probes, rollout, autoscaling и диагностика.

## Что нужно уметь на защите

- объяснить reconciliation loop и desired state;
- провести путь от `kubectl apply` до запущенного контейнера;
- выбрать Deployment, StatefulSet, DaemonSet или Job под задачу;
- объяснить, почему IP Pod нельзя использовать как постоянный адрес;
- отличить readiness от liveness и startup probe;
- прочитать простой YAML-манифест и найти связь `Service selector -> Pod labels`;
- диагностировать `Pending`, `CrashLoopBackOff`, `ImagePullBackOff` и недоступный Service;
- объяснить, почему несколько replicas не гарантируют высокую доступность сами по себе.

## Практический минимум

1. Запустить локальный кластер `minikube` или `kind`.
2. Создать Namespace, Deployment и Service из YAML.
3. Выполнить rollout новой версии и rollback.
4. Изменить ConfigMap и Secret, проверить способ их доставки в Pod.
5. Сломать readiness probe и увидеть, что Pod исчез из endpoints Service.
6. Ограничить CPU/memory, создать нагрузку и посмотреть `kubectl top`, events и logs.
7. Удалить Pod под управлением Deployment и объяснить, почему он появился снова.

Для каждого опыта сохранять манифест, команды, ожидаемое состояние, фактические events/logs и объяснение результата.
