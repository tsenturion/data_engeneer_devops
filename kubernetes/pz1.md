## Практическое задание

### Тема:

Работа с Kubernetes Controllers: **DaemonSet, Job, CronJob, Taints & Tolerations, InitContainers**

## Цель

Научиться:

- управлять рабочими узлами Kubernetes-кластера;
- использовать **Taints и Tolerations** для управления размещением Pods;
- разворачивать системные сервисы с помощью **DaemonSet**;
- выполнять одноразовые задачи через **Job**;
- автоматизировать периодические задачи с помощью **CronJob**;
- использовать **PersistentVolumeClaim (PVC)** для хранения данных;
- применять **InitContainers** для предварительной подготовки среды выполнения.

## Задание (шаги)

### Часть 1. Подготовка к практике

### Шаг 1. Увеличение количества рабочих узлов

Добавьте **2 дополнительных worker-ноды** в группу узлов Kubernetes-кластера через консоль управления Yandex Cloud.

Проверьте результат:

```bash
kubectl get nodes
```

### Шаг 2. Настройка Taint

Выберите один из узлов и установите на него Taint:

```bash
kubectl taint nodes <node_name> special=true:NoSchedule
```

Проверьте наличие Taint:

```bash
kubectl describe node <node_name>
```

## Часть 2. Работа с DaemonSet

### Шаг 3. Развертывание Prometheus Node Exporter

Создайте DaemonSet для запуска Prometheus Node Exporter на всех узлах кластера.

Примените манифест:

```bash
kubectl apply -f daemonset.yaml
```

### Шаг 4. Проверка размещения Pods

Проверьте, на каких узлах запущены Pods:

```bash
kubectl get pods -o wide
```

Убедитесь, что Pod **не запущен** на узле с Taint.

### Шаг 5. Добавление NodeSelector и Tolerations

Измените манифест:

- добавьте `nodeSelector` для Linux-узлов;
- добавьте `tolerations`, разрешающие запуск на узлах с `NoSchedule`.

Повторно примените манифест:

```bash
kubectl apply -f daemonset.yaml
```

### Шаг 6. Повторная проверка

Снова выполните:

```bash
kubectl get pods -o wide
```

Убедитесь, что теперь Pod запущен и на узле с Taint.

## Часть 3. Создание Job

### Шаг 7. Создание PVC и Job

Создайте:

- PersistentVolumeClaim `exchange-rates-pvc`
- Job `get-cbr-exchange-rates`

Job должен:

- выполнить GET-запрос к API ЦБ РФ
- сохранить XML-файл в PVC

Примените манифест:

```bash
kubectl apply -f job.yaml
```

### Шаг 8. Проверка выполнения Job

Проверьте статус:

```bash
kubectl get jobs
```

Должно быть:

```text
COMPLETIONS: 1/1
```

### Шаг 9. Проверка содержимого PVC

Создайте временный Pod с использованием `override.json` и проверьте содержимое файла:

```bash
kubectl run --rm=true -i -t check-exchange-rates-pod \
--image=alpine/curl \
--restart=Never \
--overrides="$(cat override.json)"
```

Убедитесь, что XML-файл успешно сохранён.

## Часть 4. Создание CronJob

### Шаг 10. Создание CronJob

Создайте CronJob, который:

- выполняется каждую минуту;
- получает курсы валют;
- сохраняет новый XML-файл с timestamp в имени.

Примените манифест:

```bash
kubectl apply -f cronjob.yaml
```

### Шаг 11. Проверка создания файлов

Через несколько минут проверьте содержимое PVC:

```bash
ls -la /data/
```

Убедитесь, что новые файлы появляются каждую минуту.

## Часть 5. Использование InitContainers

### Шаг 12. Добавление InitContainer

Измените CronJob:

Добавьте `InitContainer`, который:

- удаляет старые XML-файлы;
- оставляет только последние 3 файла.

Повторно примените манифест.

### Шаг 13. Финальная проверка

Проверьте PVC ещё раз.

Убедитесь, что:

- старые файлы удаляются;
- остаются только последние 3 XML-файла.

### Шаг 14. Очистка ресурсов

Удалите:

- DaemonSet
- Job
- CronJob
- PVC
- временные Pods

И уменьшите количество worker-нод.

# Подсказки по ключевым частям

## DaemonSet

Используется для запуска **одного Pod на каждом узле**.

Подходит для:

- monitoring agents
- log collectors
- system services
- security agents

Пример:

```yaml
kind: DaemonSet
```

## Taints и Tolerations

### Taint

Запрещает запуск Pods на узлах:

```bash
special=true:NoSchedule
```

### Toleration

Разрешает Pod игнорировать Taint:

```yaml
tolerations:
  - effect: NoSchedule
    operator: Exists
```

## Job

Используется для:

- одноразовых задач
- backup
- import/export
- batch processing

## CronJob

Используется для:

- регулярных задач по расписанию

Пример:

```yaml
schedule: "*/1 * * * *"
```

## InitContainer

Выполняется **до запуска основного контейнера**.

Подходит для:

- очистки файлов
- подготовки данных
- ожидания зависимостей

# Что проверить перед отправкой (чек-лист)

## Обязательно проверьте:

### Кластер

- [ ] Добавлены 2 worker-ноды
- [ ] На одной ноде установлен Taint

### DaemonSet

- [ ] DaemonSet создан
- [ ] Pods сначала не запускались на tainted node
- [ ] После добавления toleration Pods запускаются на всех узлах

### Job

- [ ] PVC создан
- [ ] Job выполнен успешно
- [ ] XML-файл появился в хранилище

### CronJob

- [ ] CronJob создаёт новые файлы каждую минуту

### InitContainer

- [ ] Старые файлы удаляются
- [ ] Остаются только последние 3 файла

### Завершение работы

- [ ] Все ресурсы удалены
- [ ] Количество нод уменьшено

# Советы по улучшению работы

## 1. Используйте отдельные YAML-файлы

Лучше создать:

- daemonset.yaml
- job.yaml
- cronjob.yaml
- override.json

Это упростит отладку.

## 2. Проверяйте события Kubernetes

Если Pod не запускается:

```bash
kubectl describe pod <pod_name>
```

Это помогает быстро найти ошибки.

## 3. Следите за логами

Для проверки выполнения:

```bash
kubectl logs <pod_name>
```

## 4. Проверяйте YAML-отступы

Большинство ошибок возникает именно из-за неверных отступов.

## 5. Не забывайте про cleanup

Особенно в облаке — лишние ноды и PVC расходуют бюджет.
