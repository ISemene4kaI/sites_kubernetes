# Argo CD: как он устроен и как с ним работать в этом репозитории

## Что такое Argo CD и зачем он здесь нужен

Argo CD — это GitOps-инструмент для Kubernetes. Он следит за тем, чтобы состояние в кластере совпадало с тем, что описано в Git.

Если говорить совсем просто, логика такая:

1. В репозитории лежат манифесты или Helm chart.
2. Argo CD читает Git-репозиторий и путь, который ему указан в `Application`.
3. Argo CD сравнивает желаемое состояние из Git с тем, что реально работает в кластере.
4. Если есть расхождение, приложение получает статус `OutOfSync`.
5. Если включён `automated sync`, Argo CD сам применяет изменения.
6. После успешного применения статус становится `Synced`.

То есть в этой схеме деплой делается не через `kubectl apply` на каждый чих, а через коммит и пуш в GitOps-репозиторий.

---

## Где Argo CD лежит в этом проекте

В этом репозитории Argo CD описан отдельным манифестом `Application`:

```text
argocd/semka-informatics.yaml
```

Именно этот файл говорит Argo CD:

- как называется приложение;
- в каком namespace находится сам объект `Application`;
- из какого Git-репозитория брать конфигурацию;
- какую ветку отслеживать;
- какой путь внутри репозитория использовать;
- в какой namespace кластера деплоить ресурсы;
- включён ли авто-синк.

На момент написания документа файл выглядит по смыслу так:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: semka-informatics
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ISemene4kaI/sites_kubernetes
    targetRevision: main
    path: apps/semka-informatics
  destination:
    server: https://kubernetes.default.svc
    namespace: semka-informatics
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Как Argo CD читает именно этот репозиторий

Для этого проекта цепочка такая:

1. Argo CD читает файл `argocd/semka-informatics.yaml`.
2. Из него берёт `repoURL`, `targetRevision` и `path`.
3. Переходит в репозиторий `sites_kubernetes`.
4. Открывает путь `apps/semka-informatics`.
5. Рендерит Helm chart из этого каталога.
6. Сравнивает результат с тем, что уже есть в namespace `semka-informatics`.
7. Если есть изменения, применяет их в кластер.

Важно понимать одну вещь: **источником истины для Argo CD является Git**, а не локальная папка на сервере. Если поменять что-то локально, но не сделать `git push`, Argo CD этого не увидит.

---

## Разбор файла `argocd/semka-informatics.yaml` по параметрам

### `apiVersion: argoproj.io/v1alpha1`
Версия Argo CD для объекта `Application`.

### `kind: Application`
Говорит Kubernetes, что это ресурс Argo CD, который описывает одно приложение.

### `metadata.name: semka-informatics`
Имя приложения внутри Argo CD.

Именно это имя потом используется в командах вроде:

```bash
kubectl get application semka-informatics -n argocd
argocd app get semka-informatics
argocd app sync semka-informatics
```

### `metadata.namespace: argocd`
Namespace, где живёт сам объект `Application`.

Обычно это `argocd`, потому что по умолчанию Argo CD ставится именно туда.

### `spec.project: default`
Проект Argo CD, к которому относится приложение.

### `spec.source.repoURL`
Git-репозиторий, который отслеживает Argo CD.

### `spec.source.targetRevision`
Ветка, тег или commit SHA, которые должен читать Argo CD.

### `spec.source.path`
Путь внутри репозитория, где лежит приложение.

### `spec.destination.server`
Куда деплоить ресурсы.

`https://kubernetes.default.svc` означает текущий кластер, в котором работает сам Argo CD.

### `spec.destination.namespace`
Namespace, куда попадут ресурсы приложения.

### `spec.syncPolicy.automated`
Включает автоматический синк.

То есть после `git push` Argo CD не ждёт ручной кнопки Sync, а сам начинает приводить кластер к состоянию из Git.

### `prune: true`
Если ресурс раньше был в Git, а потом его удалили из репозитория, Argo CD удалит его и из кластера.

### `selfHeal: true`
Если кто-то руками поменял ресурс прямо в кластере, Argo CD попытается вернуть его к состоянию из Git.

---

## Установка ArgoCD в кластер

### Вариант 1. Официальный install manifest

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

После этого проверить:

```bash
kubectl get pods -n argocd
```

Ожидаемо должны подняться основные компоненты:

- `argocd-server`
- `argocd-repo-server`
- `argocd-application-controller`
- `argocd-dex-server` или другие вспомогательные pod'ы, в зависимости от версии и конфигурации.

### Вариант 2. Через Helm

Если нужен Helm-способ, лучше брать актуальную схему из официального chart-репозитория Argo CD.

---

## Как получить доступ к Argo CD

### Через port-forward

Самый простой и безопасный вариант для локальной работы:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

После этого UI будет доступен на:

```text
https://localhost:8080
```

### Через Ingress

Можно вынести Argo CD наружу через Ingress, но это уже надо делать аккуратно: с TLS и нормальной аутентификацией. Для базовой работы port-forward обычно удобнее и безопаснее.

---

## Как залогиниться в Argo CD

Сначала получить initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Потом вход через CLI:

```bash
argocd login localhost:8080
```

Если используется port-forward и self-signed cert на локальном доступе, иногда нужен флаг:

```bash
argocd login localhost:8080 --insecure
```

---

## Как добавить `Application` в кластер

Если файл уже есть в репозитории, применяешь его один раз:

```bash
kubectl apply -f argocd/semka-informatics.yaml
```

Проверка:

```bash
kubectl get applications -n argocd
kubectl describe application semka-informatics -n argocd
```

После этого Argo CD начинает следить за этим приложением.

Важно: сам `Application` можно хранить в Git и обновлять declarative-способом через `kubectl apply`. Но уже дальнейший деплой самого приложения в namespace идёт не через ручной `kubectl apply`, а через Argo CD.

---

## Как смотреть состояние приложения

### Базовая проверка

```bash
kubectl get applications -n argocd
```

Должно быть что-то по типу:

- `SYNC STATUS`: `Synced` / `OutOfSync`
- `HEALTH STATUS`: `Healthy` / `Progressing` / `Degraded`

### Подробно по одному приложению

```bash
kubectl describe application semka-informatics -n argocd
```

Там важнее всего смотреть:

- `Source`
- `Destination`
- `Sync Policy`
- `Operation State`
- `History`
- `Events`
- `Summary.Images`

### Через CLI Argo CD

```bash
argocd app get semka-informatics
```

Полезные команды:

```bash
argocd app list
argocd app get semka-informatics
argocd app history semka-informatics
argocd app diff semka-informatics
argocd app manifests semka-informatics
argocd app resources semka-informatics
argocd app logs semka-informatics
```

---

## Как понять, что auto-sync реально работает

Проверка очень простая:

1. Меняем что-то в `apps/semka-informatics`.
2. Делаем `git commit` и `git push`.
3. Смотрим статус приложения.

Команды:

```bash
kubectl get application semka-informatics -n argocd -w
```

или:

```bash
argocd app get semka-informatics --refresh
```

Если всё настроено правильно, будет видно:

- новая revision из Git;
- краткий переход `Synced -> OutOfSync -> Synced`;
- затем обновление `Health` до `Healthy`.

---

## Как вручную запускать sync

Если auto-sync выключен или нужно руками форсировать применение:

```bash
argocd app sync semka-informatics
```

Если надо обновить данные из Git перед просмотром статуса:

```bash
argocd app get semka-informatics --refresh
```

Если нужно откатиться по истории, сначала посмотреть историю:

```bash
argocd app history semka-informatics
```

Потом уже работать с конкретной revision или deployment history через UI/CLI в зависимости от сценария.

---

## Как смотреть, что именно Argo CD задеплоил

Argo CD управляет обычными Kubernetes-ресурсами, поэтому смотреть их можно и через `kubectl`.

### Что развернуло приложение

```bash
kubectl get all -n semka-informatics
kubectl get ingress -n semka-informatics
kubectl get configmap -n semka-informatics
kubectl get secret -n semka-informatics
kubectl get pvc -n semka-informatics
```

### Deployment

```bash
kubectl get deployment -n semka-informatics
kubectl describe deployment semka-informatics -n semka-informatics
kubectl rollout status deployment/semka-informatics -n semka-informatics
```

### Pod'ы

```bash
kubectl get pods -n semka-informatics
kubectl describe pod -n semka-informatics <pod-name>
kubectl logs -n semka-informatics <pod-name>
```

### Service

```bash
kubectl get svc -n semka-informatics
kubectl describe svc semka-informatics -n semka-informatics
```

### Ingress

```bash
kubectl get ingress -n semka-informatics
kubectl describe ingress semka-informatics -n semka-informatics
```

Идея простая: Argo CD управляет ресурсами, но диагностировать их всё равно часто будем через `kubectl`.

---

## Как устроен обычный рабочий цикл

Для этого репозитория нормальная схема такая:

1. Правим Helm chart в `apps/semka-informatics`.
2. Проверям локально:

```bash
helm template semka-informatics ./apps/semka-informatics > /tmp/semka-rendered.yaml
```

3. Коммитим и пушим:

```bash
git add .
git commit -m "Update semka-informatics"
git push origin main
```

4. Argo CD видит новую ревизию и сам применяет изменения.
5. Проверяем:

```bash
kubectl get application semka-informatics -n argocd
kubectl rollout status deployment/semka-informatics -n semka-informatics
```

---

## Что важно помнить про Argo CD

### 1. Источник истины — Git
Если ты поменял что-то руками в кластере, а в Git этого нет, при `selfHeal: true` Argo CD со временем вернёт состояние назад.

### 2. `prune: true` удаляет то, что удалено из Git
Если ресурс пропал из chart, Argo CD может удалить его из кластера.

### 3. Локальный `git pull` на сервере не равен sync
Argo CD следит за **удалённым репозиторием**, а не за локальной папкой на диске.

### 4. `Synced` не всегда значит, что приложение реально работает
`Synced` означает, что манифесты в кластере совпадают с Git. Но приложение может быть `Degraded`, если pod'ы падают или сервис не работает.

---

## Проблемы и диагностика

### Проблема 1. `Application` есть, но Argo CD не подтягивает изменения

Проверяем:

```bash
kubectl describe application semka-informatics -n argocd
```

Смотри:

- `Repo URL`
- `Target Revision`
- `Path`
- `Reconciled At`
- `History`
- `Events`

Что чаще всего бывает:

- изменения не запушены в remote;
- не та ветка в `targetRevision`;
- не тот путь в `spec.source.path`;
- приложение смотрит не в тот репозиторий.

---

### Проблема 2. `OutOfSync`

Это значит, что то, что в кластере, отличается от того, что в Git.

Проверка:

```bash
argocd app diff semka-informatics
kubectl describe application semka-informatics -n argocd
```

Причины обычно такие:

- поменяли Git, но sync ещё не завершился;
- кто-то руками изменил ресурс в кластере;
- chart рендерится не так, как ожидается;
- появился новый ресурс, которого раньше не было.

---

### Проблема 3. `Synced`, но `Degraded`

Это одна из самых частых ситуаций.

Она означает: манифесты совпали, но приложение в кластере работает плохо.

Смотреть надо уже не только Argo CD, но и сами ресурсы:

```bash
kubectl get pods -n semka-informatics
kubectl logs -n semka-informatics deployment/semka-informatics --tail=100
kubectl describe pod -n semka-informatics <pod-name>
kubectl rollout status deployment/semka-informatics -n semka-informatics
```

Типичные причины:

- контейнер падает;
- неправильные env-переменные;
- не найден файл или каталог;
- неправильный image tag;
- PVC не примонтировался;
- readiness probe не проходит.

---

### Проблема 4. `Progressing` висит слишком долго

Смотри rollout и pod'ы:

```bash
kubectl rollout status deployment/semka-informatics -n semka-informatics
kubectl get pods -n semka-informatics
kubectl describe deployment semka-informatics -n semka-informatics
```

Если rollout завис, Argo CD сам по себе тут не всегда виноват — часто проблема уже в Deployment, probes, image или volume.

---

### Проблема 5. Argo CD не может прочитать Git

Смотри `Events` у `Application` и логи компонентов Argo CD:

```bash
kubectl logs -n argocd deploy/argocd-repo-server --tail=100
kubectl logs -n argocd deploy/argocd-application-controller --tail=100
```

Типичные причины:

- неверный `repoURL`;
- приватный репозиторий без настроенного доступа;
- сетевые проблемы;
- ошибка в пути.

---

### Проблема 6. Argo CD применил манифесты, но ресурс не появился

Смотри:

```bash
kubectl describe application semka-informatics -n argocd
argocd app resources semka-informatics
argocd app manifests semka-informatics
```

Тут часто оказывается, что:

- chart не рендерит ресурс;
- условие в Helm `if` не выполнилось;
- значение в `values.yaml` выключает нужный объект;
- путь в `path` указывает не на тот chart.

---

### Проблема 7. Приложение работает, но Argo CD показывает не тот image

Проверяй:

```bash
kubectl get deployment semka-informatics -n semka-informatics -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get pods -n semka-informatics -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.containers[0].image}{"\n"}{end}'
```

И сравнивай это с тем, что реально записано в `values.yaml` и что видно в `Summary.Images` у `Application`.

---

## Какие команды полезно знать наизусть

### Argo CD через kubectl

```bash
kubectl get applications -n argocd
kubectl describe application semka-informatics -n argocd
kubectl get pods -n argocd
kubectl logs -n argocd deploy/argocd-application-controller --tail=100
kubectl logs -n argocd deploy/argocd-repo-server --tail=100
kubectl logs -n argocd deploy/argocd-server --tail=100
```

### Argo CD через CLI

```bash
argocd app list
argocd app get semka-informatics
argocd app diff semka-informatics
argocd app sync semka-informatics
argocd app history semka-informatics
argocd app resources semka-informatics
argocd app manifests semka-informatics
argocd app logs semka-informatics
```

### Проверка самого приложения, которым управляет Argo CD

```bash
kubectl get all -n semka-informatics
kubectl get ingress,configmap,secret,pvc -n semka-informatics
kubectl rollout status deployment/semka-informatics -n semka-informatics
kubectl logs -n semka-informatics deployment/semka-informatics --tail=100
```