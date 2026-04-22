# CI, контейнерные образы и запуск в Kubernetes

## О чём этот документ

Этот документ описывает, как в отдельных репозиториях приложений должен быть устроен процесс сборки контейнеров в CI, куда эти контейнеры публикуются и по какому адресу Kubernetes затем запускает нужный образ.

Здесь разбирается именно связка:

- репозиторий приложения;
- GitHub Actions как CI;
- GitHub Container Registry как хранилище образов;
- GitOps-репозиторий с Helm chart;
- Kubernetes и Argo CD, которые подтягивают и запускают нужную версию контейнера.

Документ не уходит глубоко в устройство самого приложения. Основная цель — зафиксировать единый рабочий подход для новых репозиториев.

---

## Общая логика работы

Схема состоит из двух репозиториев и одного реестра образов.

### 1. Репозиторий приложения

В репозитории приложения хранится код самого сервиса:

- исходники;
- `Dockerfile`;
- тесты;
- workflow GitHub Actions, который запускает CI.

Именно этот репозиторий отвечает за то, чтобы:

1. прогнать тесты;
2. собрать Docker-образ;
3. отправить его в контейнерный реестр.

### 2. Контейнерный реестр

После успешной сборки CI публикует образ в GitHub Container Registry.

Для текущей схемы адрес выглядит так:

```text
ghcr.io/isemene4kai/<имя-образа>:<тег>
```

Пример:

```text
ghcr.io/isemene4kai/semka-informatics:887aded8c7c00c8f983c7c651ccc04992a06ff04
```

Именно **этот адрес образа** потом использует Kubernetes. Не ссылка на GitHub-репозиторий, не ссылка на workflow, не ссылка на Dockerfile, а именно ссылка на контейнерный образ в registry.

### 3. GitOps-репозиторий

Во втором репозитории хранится Kubernetes-описание приложения:

- Helm chart;
- `values.yaml`;
- манифест `Application` для Argo CD.

В этом репозитории задаётся:

- какой образ должен быть запущен;
- какой у него тег;
- какие ресурсы, переменные окружения, PVC, ingress и прочие параметры нужно применить.

### 4. Argo CD

Argo CD следит **не за репозиторием приложения**, а за GitOps-репозиторием.

Когда в GitOps-репозитории меняется:

- `image.repository`;
- `image.tag`;
- или любая другая часть Helm chart,

Argo CD видит изменение, синхронизирует приложение и обновляет Deployment в Kubernetes.

### 5. Kubernetes

После применения нового Deployment Kubernetes запускает контейнер по полю:

```yaml
image: "<registry>/<image>:<tag>"
```

Например:

```yaml
image: "ghcr.io/isemene4kai/semka-informatics:887aded8c7c00c8f983c7c651ccc04992a06ff04"
```

То есть Kubernetes запускает контейнер **по ссылке на образ в registry**, а не по ссылке на GitHub-репозиторий.

---

## Как должен выглядеть процесс для нового репозитория приложения

Для нового приложения рекомендуется придерживаться одного и того же шаблона.

### В репозитории приложения должны быть

Минимальный набор:

```text
.github/workflows/ci.yml
Dockerfile
requirements.txt или аналогичный файл зависимостей
исходный код приложения
тесты
```

### Что должен делать workflow CI

Минимальная логика CI:

1. забрать код;
2. поднять окружение;
3. установить зависимости;
4. прогнать тесты;
5. залогиниться в GHCR;
6. собрать Docker-образ;
7. отправить образ в GHCR с нормальными тегами.

### Рекомендуемая логика тегов

Практика, которая уже используется и хорошо работает:

- `latest` — для быстрого просмотра последней версии;
- `${{ github.sha }}` — для точной привязки образа к конкретному коммиту.

Основной рабочий тег для Kubernetes — это именно **SHA коммита**, потому что он однозначный и позволяет точно понимать, какая версия образа запущена.

`latest` допустим как вспомогательный тег, но не как основной способ деплоя в Kubernetes.

---

## Пример workflow для нового репозитория

Ниже — рабочий пример CI по той же логике, которая уже используется.

```yaml
name: CI

on:
  push:
    branches:
      - main
      - develop
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run tests
        run: |
          python -m tests.test_health

  docker:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push'
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ghcr.io/isemene4kai/NEW-IMAGE-NAME:latest
            ghcr.io/isemene4kai/NEW-IMAGE-NAME:${{ github.sha }}
```

---

## Что в этом workflow делает каждая часть

### `actions/checkout@v4`
Забирает содержимое репозитория в раннер GitHub Actions.

### `actions/setup-python@v5`
Поднимает Python нужной версии для тестов.

### `needs: test`
Запрещает запуск сборки Docker-образа, пока не пройдёт job с тестами.

### `if: github.event_name == 'push'`
Позволяет не пушить образы при каждом `pull_request`.

### `docker/login-action@v3`
Логинится в GitHub Container Registry.

### `docker/build-push-action@v6`
Собирает образ по `Dockerfile` и отправляет его в registry.

### `${{ github.sha }}`
Подставляет SHA коммита, который запустил workflow. Именно этот тег удобнее всего использовать в Kubernetes.

---

## Какой адрес образа потом использует Kubernetes

Kubernetes использует **строку из поля `image` в Deployment**.

Эта строка собирается из двух частей:

```yaml
image:
  repository: ghcr.io/isemene4kai/semka-informatics
  tag: "887aded8c7c00c8f983c7c651ccc04992a06ff04"
```

После рендера Helm получается:

```yaml
image: "ghcr.io/isemene4kai/semka-informatics:887aded8c7c00c8f983c7c651ccc04992a06ff04"
```

Это и есть точный адрес, по которому kubelet на ноде будет тянуть контейнерный образ.

### Важный момент

Kubernetes **не умеет запускать приложение прямо из GitHub-репозитория**.

Он не работает так:

```text
https://github.com/...
```

Он работает только так:

```text
ghcr.io/<owner>/<image>:<tag>
```

Или через другой container registry в таком же формате.

---

## Где эта ссылка задаётся в текущей схеме

### В приложенческом репозитории

В CI задаётся, **куда пушить образ**. То есть именно там определяется registry и имя образа:

```yaml
tags: |
  ghcr.io/isemene4kai/semka-informatics:latest
  ghcr.io/isemene4kai/semka-informatics:${{ github.sha }}
```

### В GitOps-репозитории

В `values.yaml` задаётся, **какую именно версию образа должен запускать Kubernetes**:

```yaml
image:
  repository: ghcr.io/isemene4kai/semka-informatics
  tag: "887aded8c7c00c8f983c7c651ccc04992a06ff04"
```

### В шаблоне Deployment

Helm подставляет эти значения в Deployment:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Именно после этого Kubernetes получает финальный адрес образа.

---

## Как должен выглядеть процесс обновления контейнера

При выпуске новой версии логика должна быть такой.

### Шаг 1. Изменения попадают в репозиторий приложения

В репозитории приложения коммитится новый код.

### Шаг 2. CI прогоняет тесты

Если тесты не прошли, Docker-образ не должен публиковаться.

### Шаг 3. CI собирает и пушит образ

После успешных тестов публикуется новый образ в GHCR.

Например:

```text
ghcr.io/isemene4kai/semka-informatics:9b8a7c6d5e4f...
```

### Шаг 4. В GitOps-репозитории обновляется `image.tag`

В `values.yaml` меняется старый SHA на новый.

### Шаг 5. Argo CD видит изменение

Argo CD отслеживает GitOps-репозиторий и синхронизирует новое состояние.

### Шаг 6. Kubernetes делает rollout

Deployment получает новый `image`, создаёт новый ReplicaSet и запускает pod уже с новым контейнером.

---

## Почему лучше использовать SHA, а не `latest`

`latest` удобно для быстрого теста, но для рабочей схемы у него есть минусы:

- не видно, какой именно коммит сейчас запущен;
- сложнее откатываться;
- сложнее разбирать инциденты;
- можно запутаться, если registry уже обновился, а manifest нет.

Тег по SHA решает эти проблемы:

- каждая версия уникальна;
- легко понять, какой коммит в кластере;
- откат — это просто возврат к предыдущему SHA;
- проще сопоставлять CI, Git и состояние Deployment.

---

## Как смотреть, какой образ сейчас реально запущен в Kubernetes

### Какой image прописан в Deployment

```bash
kubectl get deployment semka-informatics -n semka-informatics \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

### Какие образы стоят в pod'ах

```bash
kubectl get pods -n semka-informatics \
  -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.containers[0].image}{"\n"}{end}'
```

### История rollout

```bash
kubectl rollout history deployment/semka-informatics -n semka-informatics
```

### Текущий rollout

```bash
kubectl rollout status deployment/semka-informatics -n semka-informatics
```

---

## Как смотреть, что образ вообще собрался в CI

### Через GitHub Actions

Проверяется последний успешный workflow run в репозитории приложения.

Важно смотреть именно job, где был `docker/build-push-action`.

### Через список тегов в workflow

Если в workflow есть:

```yaml
ghcr.io/isemene4kai/IMAGE:${{ github.sha }}
```

то тегом образа становится SHA того коммита, который запустил workflow.

### Через локальный git

Если известен коммит, можно проверить его SHA:

```bash
git rev-parse HEAD
```

Но для деплоя лучше ориентироваться на **последний успешный workflow run**, а не просто на локальный коммит.

---

## Как проверить связку целиком

Ниже — базовая цепочка проверки.

### 1. Проверить, что workflow отработал успешно

В GitHub Actions должен быть зелёный run.

### 2. Проверить, что в GitOps-репозитории обновлён `image.tag`

```bash
grep -n "tag:" apps/semka-informatics/values.yaml
```

### 3. Проверить, что Argo CD синхронизировал изменения

```bash
kubectl get application semka-informatics -n argocd
kubectl describe application semka-informatics -n argocd
```

### 4. Проверить, что Deployment уже смотрит на новый образ

```bash
kubectl get deployment semka-informatics -n semka-informatics \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

### 5. Проверить rollout

```bash
kubectl rollout status deployment/semka-informatics -n semka-informatics
```

### 6. Проверить pod'ы

```bash
kubectl get pods -n semka-informatics
```

---

## Что нужно предусматривать в новых репозиториях

Чтобы новый сервис легко подключался к общей схеме, в его репозитории стоит держать одинаковый подход.

### Обязательно

- нормальный `Dockerfile`;
- тесты, которые можно запускать в CI;
- GitHub Actions workflow;
- публикацию в GHCR;
- SHA-теги для образов;
- понятное и стабильное имя образа.

### Желательно

- отдельный health endpoint;
- минимальные зависимости в образе;
- понятный entrypoint;
- `.dockerignore`, чтобы не тащить мусор в build context.

---

## Типовые ошибки

### 1. Образ не пушится в GHCR

Частые причины:

- не выданы `packages: write` permissions;
- нет логина в GHCR;
- имя образа содержит заглавные буквы;
- сборка падает ещё до шага push.

Что проверить:

```bash
# в workflow должен быть такой блок
permissions:
  contents: read
  packages: write
```

И логин:

```yaml
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

### 2. В Kubernetes всё ещё крутится старый образ

Частые причины:

- в `values.yaml` не обновился `image.tag`;
- обновился только `latest`, а Deployment закреплён на старом SHA;
- Argo CD ещё не сделал sync;
- rollout завис.

Что проверить:

```bash
kubectl get application semka-informatics -n argocd
kubectl get deployment semka-informatics -n semka-informatics \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl rollout status deployment/semka-informatics -n semka-informatics
```

### 3. Образ собирается, но pod падает

Это уже не проблема registry, а проблема самого контейнера.

Что смотреть:

```bash
kubectl logs -n semka-informatics deployment/semka-informatics --tail=100
kubectl describe pod -n semka-informatics <имя-pod>
```

### 4. Артефакты в приложении не переживают пересоздание pod

Если приложение пишет состояние в файловую систему контейнера, данные пропадут после пересоздания pod.

Для таких файлов нужен PVC и volume mount, а не запись прямо в слой контейнера.

### 5. CI публикует образ, но Kubernetes не может его вытянуть

Частые причины:

- неверный `repository`;
- неверный `tag`;
- образ приватный, а `imagePullSecrets` не настроены;
- опечатка в имени registry.

Проверять надо итоговую строку `image` в Deployment.

---

## Практическое правило для новых сервисов

Если коротко, то для нового репозитория процесс должен выглядеть так:

1. В репозитории приложения настраивается CI.
2. CI после тестов собирает Docker-образ.
3. Образ пушится в GHCR.
4. В GitOps-репозитории обновляется `image.tag` на новый SHA.
5. Argo CD видит изменение.
6. Kubernetes запускает контейнер по адресу вида:

```text
ghcr.io/isemene4kai/<image-name>:<sha>
```

Именно эта строка и является ответом на вопрос, **по какой ссылке Kubernetes запускает контейнер**.

---

## Короткая памятка

### Где собирается контейнер
В CI репозитория приложения.

### Куда пушится контейнер
В GHCR.

### Где хранится адрес нужного контейнера для кластера
В GitOps-репозитории, в `values.yaml`.

### Кто применяет изменения в кластер
Argo CD.

### По чему именно kubelet тянет контейнер
По строке вида:

```text
ghcr.io/isemene4kai/<image>:<tag>
```

### Какой тег использовать для боевого деплоя
SHA коммита, а не `latest`.