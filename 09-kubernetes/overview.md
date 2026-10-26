# Kubernetes: краткая справка

## От контейнера к оркестратору

Контейнер упаковывает приложение и его userspace-зависимости, но один container runtime не решает размещение по хостам, автоматическое восстановление, rollout, service discovery, секреты и масштабирование. `docker compose` управляет группой контейнеров на одной машине; Kubernetes согласует состояние приложений в кластере машин.

Пользователь описывает **желаемое состояние** объектами API. Контроллеры сравнивают его с фактическим и выполняют reconciliation: создают, удаляют или заменяют ресурсы, пока состояние не приблизится к требуемому. Это не одноразовый скрипт, а постоянный цикл управления.

## Архитектура кластера

Кластер состоит из control plane и worker nodes.

| Компонент | Роль |
|---|---|
| `kube-apiserver` | Центральный REST API, аутентификация, авторизация и валидация запросов |
| `etcd` | Распределённое key-value хранилище состояния кластера; его backup критичен |
| `kube-controller-manager` | Запускает controllers, которые согласуют desired и actual state |
| `kube-scheduler` | Выбирает node для ещё не назначенного Pod с учётом ресурсов и ограничений |
| `kubelet` | Агент на node: получает PodSpec и обеспечивает запуск контейнеров через runtime |
| `kube-proxy` | Поддерживает сетевые правила для Service; конкретная реализация зависит от кластера |
| cloud controller manager | Связывает Kubernetes с API облака: load balancers, routes, nodes, volumes |

Упрощённый путь создания Pod:

1. Клиент отправляет объект в `kube-apiserver`.
2. API server проверяет запрос и сохраняет состояние в `etcd`.
3. Controller создаёт недостающий Pod для Deployment/ReplicaSet.
4. Scheduler выбирает node и записывает binding.
5. Kubelet выбранной node просит container runtime скачать image и запустить containers.
6. Kubelet публикует status; controllers продолжают следить за отклонениями.

Компоненты не передают друг другу длинную императивную команду: большинство взаимодействий идёт через наблюдение объектов API.

## Pod

Pod — минимальная единица развёртывания. Его контейнеры:

- планируются на одну node;
- разделяют network namespace и IP;
- обращаются друг к другу через `localhost`;
- могут разделять volumes;
- создаются и удаляются как единое целое.

Обычно в Pod один основной контейнер. Дополнительный sidecar оправдан, когда процесс тесно связан с жизненным циклом приложения, например локальный proxy или сборщик. Размещать в одном Pod произвольные сервисы ради экономии нельзя: они потеряют независимое масштабирование и выпуск.

Pod эфемерен. После пересоздания у него новое имя/IP, поэтому к Pod обращаются через Service, а долговечные данные выносят во внешнее хранилище или PersistentVolume.

## Controllers и workloads

### ReplicaSet и Deployment

ReplicaSet поддерживает заданное число одинаковых Pod. На практике им управляет Deployment, который добавляет декларативный rollout и rollback через версии ReplicaSet.

Deployment подходит для stateless-приложений: HTTP API, workers, consumers. Изменение Pod template создаёт новый rollout; простое изменение числа replicas — нет.

### StatefulSet

StatefulSet нужен экземплярам с устойчивой идентичностью, порядком запуска/остановки и отдельными persistent volumes. Pod получают предсказуемые имена вроде `db-0`, `db-1`. StatefulSet помогает управлять stateful workload, но не создаёт репликацию данных или корректный failover внутри самой БД.

### DaemonSet

DaemonSet обеспечивает Pod на каждой подходящей node. Типичные примеры: log collector, node metrics agent, сетевой компонент. Его масштаб задаётся числом nodes, а не полем `replicas`.

### Job и CronJob

Job запускает конечную задачу до успешного завершения и умеет повторять неуспешные Pod. CronJob создаёт Job по расписанию. Обработчик должен учитывать повторы: из-за сбоев одна логическая задача может стартовать больше одного раза.

## Манифест и связь объектов

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: app
          image: example/orders:1.4.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
          livenessProbe:
            httpGet:
              path: /live
              port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: orders
spec:
  selector:
    app: orders
  ports:
    - port: 80
      targetPort: 8080
```

Ключевая связь — selector Service совпадает с labels Pod. `containerPort` документирует порт контейнера, но сам по себе не публикует приложение.

## Service, DNS и внешний трафик

Service даёт стабильное виртуальное имя и адрес для динамического набора Pod. Endpoints формируются по labels и readiness.

- `ClusterIP` — доступ только внутри кластера, тип по умолчанию;
- `NodePort` — порт на nodes, обычно промежуточный механизм;
- `LoadBalancer` — запрашивает внешний балансировщик у интеграции с облаком;
- headless Service (`clusterIP: None`) — не даёт общий virtual IP, DNS возвращает адреса отдельных endpoints.

Внутренний клиент обычно обращается к `orders.<namespace>.svc.cluster.local`. Cluster DNS разрешает имя, а сетевой dataplane направляет соединение на один ready endpoint.

Для HTTP-трафика снаружи применяют Ingress или более выразительный Gateway API. `Gateway` описывает точку входа и listeners, `HTTPRoute` — правила сопоставления host/path и backend Service. Ресурсы работают только при установленном controller.

## ConfigMap и Secret

ConfigMap хранит несекретную конфигурацию, Secret — чувствительные значения. Оба можно передать как environment variables или смонтированные файлы.

Secret не становится безопасным только из-за имени: данные в обычном YAML/base64 не зашифрованы. Нужны RBAC, encryption at rest, ограничение доступа, ротация и, при необходимости, внешний secrets manager. Секреты нельзя коммитить в репозиторий.

Обновление ConfigMap/Secret не всегда автоматически перезапускает приложение. Environment variables не меняются в уже запущенном процессе; файл volume может обновиться с задержкой, но приложение ещё должно перечитать его. Часто rollout запускают явно.

## Volumes, PV и PVC

- `emptyDir` создаётся вместе с Pod и живёт до удаления Pod; переживает перезапуск отдельного container, но не Pod;
- `hostPath` связывает данные с конкретной node и опасен для переносимости и безопасности;
- PersistentVolume (PV) представляет storage в кластере;
- PersistentVolumeClaim (PVC) — запрос workload на размер и свойства storage;
- StorageClass описывает способ динамического provisioning.

PVC отделяет потребность приложения от конкретного диска. Но Kubernetes не заменяет backup: удаление или повреждение данных может распространиться на persistent storage.

## Requests, limits и scheduling

`requests` используются scheduler для размещения и резервирования ёмкости. `limits` ограничивают потребление.

- при превышении memory limit процесс обычно получает OOM kill;
- CPU limit приводит к throttling, а не к убийству;
- отсутствие requests мешает планированию ёмкости;
- слишком маленькие requests создают перегрузку nodes, слишком большие — плохую утилизацию и `Pending` Pod.

QoS class (`Guaranteed`, `Burstable`, `BestEffort`) влияет на поведение при дефиците ресурсов. Для высокой доступности replicas следует распределять по nodes/zones через topology spread constraints или anti-affinity.

## Probes и graceful lifecycle

- readiness probe отвечает: можно ли направлять новый трафик;
- liveness probe отвечает: нужно ли перезапустить container;
- startup probe даёт медленно стартующему приложению время до включения остальных probes.

Readiness не должна проверять каждую далёкую зависимость так, чтобы краткий сбой выключил все replicas одновременно. Liveness особенно опасна: неверная проверка создаёт restart loop под нагрузкой.

При завершении Pod получает `SIGTERM`, начинается grace period, endpoint исключается из готовых, затем при необходимости приходит `SIGKILL`. Приложение должно прекратить приём новой работы, закончить допустимые запросы, закрыть consumers/connections и уложиться в `terminationGracePeriodSeconds`.

## Rollout и autoscaling

RollingUpdate постепенно заменяет старые Pod новыми. `maxUnavailable` ограничивает временно недоступные replicas, `maxSurge` — дополнительные. Успешный запуск процесса ещё не означает готовность: rollout зависит от readiness.

Horizontal Pod Autoscaler меняет replicas по метрикам. Он не помогает, если:

- узкое место — общая БД;
- Pod долго стартуют, а пик короче старта;
- метрика запаздывает или плохо связана с нагрузкой;
- нет свободных nodes и cluster autoscaler не добавляет capacity.

Для очередей полезнее масштабирование по lag/длине очереди, чем только по CPU.

## Диагностика

Базовая последовательность:

```bash
kubectl get pods -n shop -o wide
kubectl describe pod orders-... -n shop
kubectl logs orders-... -n shop --previous
kubectl get events -n shop --sort-by=.lastTimestamp
kubectl get svc,endpoints -n shop
kubectl rollout status deployment/orders -n shop
kubectl top pod -n shop
```

Частые состояния:

| Симптом | Что проверять |
|---|---|
| `Pending` | resources, taints/tolerations, affinity, PVC, события scheduler |
| `ImagePullBackOff` | имя/tag image, registry credentials, сеть registry |
| `CrashLoopBackOff` | текущие и `--previous` logs, command/args, config, probes, OOM |
| Pod `Running`, но не `Ready` | readiness probe и зависимости |
| Service не отвечает | selector/labels, endpoints, targetPort, NetworkPolicy, приложение слушает не только localhost |
| Rollout завис | readiness, image, quota, `maxUnavailable`/`maxSurge`, capacity |

Не начинать с удаления Pod: сначала сохранить status, events и logs, иначе важное доказательство исчезнет.

## Безопасность и эксплуатационный минимум

- выдавать ServiceAccount минимальные RBAC-права;
- запускать контейнер не от root, запретить privilege escalation, сделать root filesystem read-only где возможно;
- фиксировать image digest или неизменяемый tag;
- задавать resources и probes осмысленно;
- использовать NetworkPolicy при поддержке CNI;
- собирать logs, metrics, events и audit trail;
- иметь PodDisruptionBudget, но помнить: он влияет на добровольные disruptions, а не спасает от падения node;
- проверять backup/restore stateful-компонентов.

Высокая доступность достигается совместно: несколько replicas, распределение по failure domains, готовые probes, достаточная capacity, устойчивые зависимости, корректный rollout и наблюдаемость.
