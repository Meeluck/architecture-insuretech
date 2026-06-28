# Задание 2. Динамическое масштабирование контейнеров

## Что сделано

Для тестового приложения `scaletestapp` подготовлены Kubernetes-манифесты:

- `scaletestapp-deployment.yaml`
- `scaletestapp-service.yaml`
- `scaletestapp-hpa.yaml`

Deployment запускает приложение в одной начальной реплике и задаёт лимит памяти `30Mi`.

```yaml
replicas: 1
resources:
  requests:
    memory: "20Mi"
    cpu: "50m"
  limits:
    memory: "30Mi"
    cpu: "250m"
```

Так как при скачивании образа из `ghcr.io` возникала ошибка `ErrImagePull`, использован локально собранный образ:

```yaml
image: scaletestapp:local
imagePullPolicy: Never
```

Перед применением Deployment образ был загружен в Minikube командой:

```bash
minikube image load scaletestapp:local
```

Service создан с типом `NodePort` и направляет трафик на порт приложения `8080`.

HPA настроен на масштабирование Deployment `scaletestapp` по утилизации памяти:

```yaml
minReplicas: 1
maxReplicas: 10
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## Как применялась конфигурация

Кластер Minikube был запущен локально:

```bash
minikube start --driver=docker --cpus=2 --memory=4096
```

Для работы HPA был включён `metrics-server`:

```bash
minikube addons enable metrics-server
```

Манифесты были применены командами:

```bash
kubectl apply -f Task2/scaletestapp-deployment.yaml
kubectl apply -f Task2/scaletestapp-service.yaml
kubectl apply -f Task2/scaletestapp-hpa.yaml
```

Проверка доступности приложения выполнялась через Service:

```bash
minikube service scaletestapp --url
```

На macOS с Docker driver команда `minikube service scaletestapp --url` должна оставаться запущенной в отдельном терминале. `kubectl port-forward` использовался только для быстрой проверки доступности, так как он не подходит для проверки балансировки между репликами.

## Нагрузочное тестирование

Для генерации нагрузки использовался Locust. Сценарий находится в файле:

```text
Task2/locustfile.py
```

Содержимое сценария:

```python
from locust import HttpUser, between, task


class WebsiteUser(HttpUser):
    wait_time = between(0.1, 1)

    @task
    def index(self):
        self.client.get("/")
```

Locust отправлял запросы на URL, полученный через `minikube service scaletestapp --url`.

## Результаты до нагрузки

Перед нагрузочным тестом состояние HPA было сохранено в файл:

```text
Task2/scaling-before-hpa.txt
```

Вывод:

```text
NAME           REFERENCE                 TARGETS           MINPODS   MAXPODS   REPLICAS   AGE
scaletestapp   Deployment/scaletestapp   memory: 23%/80%   1         10        1          4m53s
```

Состояние Deployment было сохранено в файл:

```text
Task2/scaling-before-deployment.txt
```

Вывод:

```text
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
scaletestapp   1/1     1            1           7m43s
```

Состояние подов было сохранено в файл:

```text
Task2/scaling-before-pods.txt
```

Вывод:

```text
NAME                           READY   STATUS    RESTARTS   AGE
scaletestapp-b45f9d786-s9r9q   1/1     Running   0          7m43s
```

До нагрузки приложение работало в одной реплике.

## Результаты после нагрузки

После запуска Locust состояние HPA было сохранено в файл:

```text
Task2/scaling-after-hpa.txt
```

Вывод:

```text
NAME           REFERENCE                 TARGETS           MINPODS   MAXPODS   REPLICAS   AGE
scaletestapp   Deployment/scaletestapp   memory: 71%/80%   1         10        3          10m
```

Состояние Deployment было сохранено в файл:

```text
Task2/scaling-after-deployment.txt
```

Вывод:

```text
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
scaletestapp   3/3     3            3           13m
```

Состояние подов было сохранено в файл:

```text
Task2/scaling-after-pods.txt
```

Вывод:

```text
NAME                           READY   STATUS    RESTARTS        AGE
scaletestapp-b45f9d786-b5j6p   1/1     Running   0               2m27s
scaletestapp-b45f9d786-pkgnz   1/1     Running   0               3m27s
scaletestapp-b45f9d786-s9r9q   1/1     Running   1 (2m22s ago)   13m
```

После нагрузки приложение масштабировалось с одной реплики до трёх реплик.

## Подтверждение работы HPA

Подробное описание HPA сохранено в файл:

```text
Task2/scaletestapp-hpa-describe.txt
```

Ключевые строки из вывода:

```text
Metrics:
  resource memory on pods  (as a percentage of request):  71% (14964053333m) / 80%
Min replicas:                                             1
Max replicas:                                             10
Deployment pods:                                          3 current / 3 desired
ScalingActive   True    ValidMetricFound
ScalingLimited  False   DesiredWithinRange
Normal  SuccessfulRescale  New size: 2
Normal  SuccessfulRescale  New size: 3
```

Статус `ScalingActive True` подтверждает, что HPA получает метрики памяти.

Статус `DesiredWithinRange` означает, что рассчитанное количество реплик находится между `minReplicas` и `maxReplicas`.

События `SuccessfulRescale` подтверждают, что HPA выполнил масштабирование сначала до двух реплик, затем до трёх реплик.

## Итог

Задание выполнено:

- Deployment создан для тестового приложения.
- Service создан для доступа к приложению.
- Metrics Server включён.
- HPA создан и настроен на целевую утилизацию памяти `80%`.
- Максимальное количество реплик ограничено значением `10`.
- Под нагрузкой приложение масштабировалось с `1` до `3` реплик.
- Логи с состоянием до и после нагрузки сохранены в директории `Task2`.
