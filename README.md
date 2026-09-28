# Final task: Запуск кластера с нуля

Здесь описано, как поднять всё заново на чистом minikube, если кластер удалён: GitLab + Registry, раннер, два репозитория, CI/CD, ArgoCD, web-app и мониторинг.

Все команды выполняются из корня этой папки (`final_task/`), если не сказано иное.

```
final_task/
├── app_repo/                 # -> GitLab: rymond044/final_k8s_app_repo  (код + .gitlab-ci.yml)
└── helm_repo/                # -> GitLab: rymond044/final_k8s_helm_repo (корень репозитория = содержимое этой папки)
    └── helm_apps/
        ├── charts/web-app/   # helm-chart приложения (его деплоит ArgoCD)
        └── infra/
            ├── gitlab/       # values для gitlab/postgres/redis + манифесты minio и секретов
            ├── argocd/       # ingress, params, AppProject, Application, ClusterIssuer
            └── monitoring/   # values для kube-prometheus-stack и loki-stack + CRD-манифесты
```

| Компонент | Способ установки | Namespace | Имя релиза |
| --- | --- | --- | --- |
| PostgreSQL | helm `bitnami/postgresql` | `gitlab` | `gitlab-postgres` |
| Redis | helm `bitnami/redis` | `gitlab` | `gitlab-redis` |
| MinIO (object storage для GitLab) | манифест `gitlab-minio.yaml` | `gitlab` | – |
| GitLab + Registry + Runner (+ cert-manager из чарта) | helm `gitlab/gitlab` | `gitlab` | `gitlab` |
| ArgoCD | официальный `install.yaml` + манифесты | `argocd` | – |
| Web-app | helm-chart, деплоит ArgoCD | `app` | `web-app` |
| Prometheus + Grafana + Alertmanager | helm `prometheus-community/kube-prometheus-stack` | `monitoring` | `monitoring-stack` |
| Loki + Promtail | helm `grafana/loki-stack` | `logging` | `loki` |

Версии, которые работали на момент написания: minikube k8s `v1.35.1`, чарт `gitlab-10.4.0` (GitLab `v19.4.0`), `postgresql-18.11.3`, `redis-28.2.1`, ArgoCD `v3.5.3`.

---

## 0. Требования к хосту

- docker, minikube, kubectl, helm, git, ssh-keygen
- ~8 GiB RAM под minikube. Всё сразу (GitLab + мониторинг) в 8 GiB не влезает, см. раздел 8.

### /etc/hosts (обязательно!)

```
192.168.49.2 gitlab.minikube.local registry.minikube.local app.minikube.local argocd.minikube.local grafana.minikube.local prometheus.minikube.local alertmanager.minikube.local
```

> Это нужно не только браузеру. В CoreDNS **нет** своих записей для `*.minikube.local`: поды (ArgoCD repo-server, CI-джобы, dind) и docker внутри ноды minikube резолвят эти имена через DNS хоста (CoreDNS → docker DNS `192.168.49.1` → systemd-resolved хоста → `/etc/hosts`).
> Если minikube выдаст другой IP (`minikube ip`), поправь `/etc/hosts` и `global.hosts.externalIP` в `helm_repo/helm_apps/infra/gitlab/values/gitlab-values.yaml`.

---

## 1. Minikube

```bash
minikube start --memory=8192 --cpus=4 --cni=flannel --insecure-registry="registry.minikube.local"
minikube addons enable ingress
minikube ip   # должен быть 192.168.49.2
```

`--insecure-registry` обязателен: registry GitLab работает по HTTP (TLS выключен), и без этого kubelet не сможет стянуть образ web-app.

Helm-репозитории:

```bash
helm repo add gitlab https://charts.gitlab.io/
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## 2. GitLab (postgres, redis, minio, gitlab)

Порядок важен: сначала секреты и MinIO, потом БД и Redis, и только потом GitLab.

```bash
kubectl create namespace gitlab

# секреты (пароли postgres/redis, подключение к minio, конфиг хранилища registry) + minio + job создания бакетов
kubectl apply -f helm_repo/helm_apps/infra/gitlab/manifests/

helm install gitlab-postgres bitnami/postgresql -n gitlab -f helm_repo/helm_apps/infra/gitlab/values/postgresql-values.yaml
helm install gitlab-redis    bitnami/redis      -n gitlab -f helm_repo/helm_apps/infra/gitlab/values/redis-values.yaml

kubectl -n gitlab rollout status statefulset/gitlab-postgres-postgresql
kubectl -n gitlab rollout status statefulset/gitlab-redis-master
kubectl -n gitlab wait --for=condition=complete job/create-minio-buckets --timeout=300s

helm install gitlab gitlab/gitlab -n gitlab -f helm_repo/helm_apps/infra/gitlab/values/gitlab-values.yaml --timeout 20m
```

Пароли в `gitlab-secrets.yaml` должны совпадать с values:

- `gitlab-postgres-password.mainDB-password` = `auth.password` в `postgresql-values.yaml` (`gitlab-password`)
- `gitlab-redis-password.secret` = `auth.password` в `redis-values.yaml` (`redis-password`)
- ключи MinIO в `gitlab-minio-secret` = `MINIO_ROOT_USER/PASSWORD` в `gitlab-minio.yaml`

Поднимается долго (10–20 минут, `gitlab-migrations` и `webservice` тяжёлые). Следить так:

```bash
kubectl -n gitlab get pods -w
```

Что ставит чарт, кроме самого GitLab:

- **gitlab-runner** (kubernetes executor, privileged для docker-in-docker) регистрируется сам через `gitlab-gitlab-runner-secret`. Проверить можно в *Admin → CI/CD → Runners*.
- **cert-manager** (`gitlab-certmanager*`). Он же потом выдаёт self-signed серт для ingress ArgoCD.
- **envoy gateway** (`envoy-gitlab-gitlab-gw-*`, Service `LoadBalancer` с `externalIPs: 192.168.49.2`): через него идёт **SSH на порт 22** (`git@gitlab.minikube.local`). HTTP идёт через ingress-nginx.

Пароль root:

```bash
kubectl -n gitlab get secret gitlab-gitlab-initial-root-password -o jsonpath='{.data.password}' | base64 -d; echo
```

UI: <http://gitlab.minikube.local> (логин `root`).

> После пересоздания кластера SSH host key GitLab меняется, поэтому удали старый из known_hosts:
> `ssh-keygen -R gitlab.minikube.local; ssh-keygen -R registry.minikube.local`

---

## 3. Пользователь, репозитории и доступы в GitLab

### 3.1 Пользователь

Под `root` создай пользователя **`rymond044`** (Admin → Users → New user), задай пароль и залогинься им.
Путь `rymond044/...` захардкожен в `.gitlab-ci.yml`, `values.yaml` (`image.repository`) и `application.yaml`. Если namespace будет другой, поправь во всех трёх местах.

Добавь свой публичный SSH-ключ пользователю (*User settings → SSH Keys*), чтобы пушить с ноутбука.

### 3.2 Репозитории

Создай два **пустых** проекта (без README), видимость любая:

| Проект | Содержимое | Default branch |
| --- | --- | --- |
| `rymond044/final_k8s_app_repo` | содержимое `app_repo/` | `main` |
| `rymond044/final_k8s_helm_repo` | содержимое `helm_repo/` (в корне репо лежит `helm_apps/`) | `main` |

У `final_k8s_app_repo` должен быть включён Container Registry (*Settings → General → Visibility → Container registry*). По умолчанию он включён.

Залить код:

```bash
cd helm_repo
git init -b main && git add . && git commit -m "init helm repo"
git remote add origin git@gitlab.minikube.local:rymond044/final_k8s_helm_repo.git
git push -u origin main
cd ..

cd app_repo
git init -b main && git add . && git commit -m "init app repo"
git remote add origin git@gitlab.minikube.local:rymond044/final_k8s_app_repo.git
git push -u origin main
cd ..
```

(Если в папках уже есть `.git` со старым remote, просто `git remote set-url origin ...` и `git push`.)

Сначала заливай helm-репо, потом app-репо. Тег `dev-v*` пушить только после настройки переменных из п. 3.3.

### 3.3 SSH-ключ для CI и ArgoCD (доступ к helm-репо)

Одна пара ключей используется в двух местах: CI пушит новый тег в helm-репо, ArgoCD читает helm-репо.

```bash
ssh-keygen -t ed25519 -C "gitlab-ci-deploy-key" -f ./id_ed25519_helm -N ""
```

(Приватный ключ в git не коммитить.)

**В `final_k8s_helm_repo`** → *Settings → Repository → Deploy keys → Add new key*:

- Key: содержимое `id_ed25519_helm.pub`
- ✅ **Grant write permissions to this key**

**Разрешить deploy key пушить в защищённую `main`:** *Settings → Repository → Protected branches → main → Allowed to push and merge* добавь этот deploy key (или убери защиту с `main`). Иначе `update_helm_tag` упадёт на `git push` с ошибкой `You are not allowed to push code to protected branches`.

### 3.4 CI/CD переменные в `final_k8s_app_repo`

*Settings → CI/CD → Variables → Add variable*:

| Key | Value | Type | Protected | Masked | Зачем |
| --- | --- | --- | --- | --- | --- |
| `HELM_REPO_SSH_KEY` | **весь** приватный ключ `id_ed25519_helm` (с `-----BEGIN…` / `-----END…` и переводом строки в конце) | Variable | ❌ | ❌ (многострочное не маскируется) | `update_helm_tag` клонирует и пушит helm-репо по SSH |
| `HELM_REPO_PAT` | *(опционально)* Project Access Token helm-репо с `write_repository` | Variable | ❌ | ✅ | используется только в `HELM_REPO_URL`, который пайплайн сейчас **не вызывает**. Можно не создавать |

> **Protected = выключено**, это важно. Пайплайн запускается на **теги** `dev-v*`, а protected-переменные доступны только на protected-ветках/тегах. Если хочешь включить Protected, добавь правило *Settings → Repository → Protected tags*: `dev-v*`.

Остальное берётся из предопределённых переменных GitLab: `CI_REGISTRY`, `CI_REGISTRY_USER`, `CI_JOB_TOKEN`, `CI_PROJECT_PATH`, `CI_COMMIT_TAG`. Настраивать их не нужно.

### 3.5 Deploy token для pull образа из registry

Kubernetes тянет образ `registry.minikube.local/rymond044/final_k8s_app_repo` через `imagePullSecret` `gitlab-registry-key`, а тот собирается чартом из `values.yaml`.

**В `final_k8s_app_repo`** → *Settings → Repository → Deploy tokens → Add token*:

- Name: любое (например `k8s-pull`)
- Scopes: ✅ `read_registry`

После пересоздания GitLab старый токен недействителен, поэтому впиши новые данные в `helm_repo/helm_apps/charts/web-app/values.yaml`:

```yaml
registry:
  username: "gitlab+deploy-token-N"   # что показал GitLab
  password: "gldt-...."
```

и запушь в helm-репо.

---

## 4. Первая сборка образа

```bash
cd app_repo
git tag dev-v1.0.0
git push origin dev-v1.0.0
```

Пайплайн (`.gitlab-ci.yml`):

1. `build_image` (docker:24.0.7 + dind с `--insecure-registry=registry.minikube.local`) собирает и пушит `registry.minikube.local/rymond044/final_k8s_app_repo:dev-v1.0.0`.
2. `update_helm_tag` по SSH клонирует helm-репо, через `sed` меняет `tag:` в `helm_apps/charts/web-app/values.yaml`, коммитит с `[skip ci]` и пушит в `main`.

Запускается **только по тегам** `dev-v*`. Обычные коммиты в ветки пайплайн не триггерят.

Проверка: *Deploy → Container Registry* в app-репо и новый коммит `chore(ci): update image tag…` в helm-репо.

> ⚠️ Хранилище registry сейчас `filesystem: /tmp/registry` (секрет `gitlab-registry-storage`), то есть **внутри пода**. После рестарта `gitlab-registry` образы пропадают, и под web-app словит `ImagePullBackOff`. Лечится повторным пушем тега (новым, например `dev-v1.0.1`). Бакет `registry-storage` в MinIO создан, но не используется.

---

## 5. ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# (если stable сломается: .../argo-cd/v3.5.3/manifests/install.yaml)

kubectl -n argocd rollout status deploy/argocd-server
```

Манифесты из helm-репо (`server.insecure`, ingress, ClusterIssuer для cert-manager, AppProject, Application):

```bash
kubectl apply -f helm_repo/helm_apps/infra/argocd/argocd-params.yaml \
              -f helm_repo/helm_apps/infra/argocd/cert-manager-issuer.yaml \
              -f helm_repo/helm_apps/infra/argocd/argocd-ingress.yaml \
              -f helm_repo/helm_apps/infra/argocd/app-project.yaml
kubectl -n argocd rollout restart deployment argocd-server
```

> `cert-manager-issuer.yaml` требует CRD cert-manager, а они приходят с чартом GitLab (п. 2). Поэтому ArgoCD ставится после GitLab.

### 5.1 Подключение helm-репо к ArgoCD

Нужен тот же приватный ключ `id_ed25519_helm` из п. 3.3. Декларативно:

```bash
kubectl -n argocd create secret generic web-app-helm-repo \
  --from-literal=type=git \
  --from-literal=name=web-app-helm-repo \
  --from-literal=project=web-app-project \
  --from-literal=url=git@gitlab.minikube.local:rymond044/final_k8s_helm_repo.git \
  --from-literal=insecure=true \
  --from-file=sshPrivateKey=./id_ed25519_helm
kubectl -n argocd label secret web-app-helm-repo argocd.argoproj.io/secret-type=repository
```

(То же самое можно сделать в UI: *Settings → Repositories → Connect repo → SSH*, project `web-app-project`, ✅ *Skip server verification*.)

`insecure=true` отключает проверку host key: после каждого пересоздания GitLab он новый.

### 5.2 Application

```bash
kubectl apply -f helm_repo/helm_apps/infra/argocd/application.yaml
```

Application `web-app` смотрит на `helm_apps/charts/web-app` (HEAD), деплоит в namespace `app`, `automated` + `prune` + `selfHeal`.

> Namespace `app` ArgoCD сам **не создаёт** (в `syncPolicy` нет `CreateNamespace=true`). Создай его заранее: `kubectl create namespace app`.

Пароль `admin` для ArgoCD:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

UI: <https://argocd.minikube.local> (self-signed серт, браузер будет ругаться).

После синка: <http://app.minikube.local> покажет страницу с секретом `APP_SECRET` (из `secret.appSecretValue` в values → k8s Secret `web-app-secret` → env контейнера).

### Проверка всей цепочки CI/CD

```bash
cd app_repo
# поменять что-нибудь в app.py
git commit -am "change" && git push
git tag dev-v1.0.1 && git push origin dev-v1.0.1
```

GitLab собирает образ, CI меняет `tag` в helm-репо, ArgoCD (опрашивает git раз в ~3 мин, или жми *Refresh*) передеплоит под с новым образом.

---

## 6. Мониторинг (Prometheus + Grafana + Alertmanager + Loki)

```bash
kubectl create namespace monitoring
kubectl create namespace logging

# секрет с конфигом alertmanager (telegram) должен существовать ДО установки стека
kubectl apply -f helm_repo/helm_apps/infra/monitoring/manifests/alertmanager-secret.yaml

helm install loki grafana/loki-stack -n logging \
  -f helm_repo/helm_apps/infra/monitoring/values/loki-values.yaml

helm install monitoring-stack prometheus-community/kube-prometheus-stack -n monitoring \
  -f helm_repo/helm_apps/infra/monitoring/values/prom-grafana-stack.yaml

# PrometheusRule + ServiceMonitor (нужны CRD из kube-prometheus-stack)
kubectl apply -f helm_repo/helm_apps/infra/monitoring/manifests/alert-rule.yaml \
              -f helm_repo/helm_apps/infra/monitoring/manifests/service-monitor.yaml
```

Имя релиза **обязательно `monitoring-stack`**: `PrometheusRule` и `ServiceMonitor` помечены `release: monitoring-stack`, и Prometheus подхватывает только объекты с этим label.

- Grafana: <http://grafana.minikube.local>, логин `admin`, пароль:
  `kubectl -n monitoring get secret monitoring-stack-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo`
- Prometheus: <http://prometheus.minikube.local>
- Alertmanager: <http://alertmanager.minikube.local>

Loki прописан как datasource через `grafana.additionalDataSources`, то есть через provisioning. Поэтому после рестарта пода Grafana он остаётся.

### Telegram

Токен бота и `chat_id` лежат в `alertmanager-secret.yaml`. Если бот/чат новые:

- токен выдаёт @BotFather;
- `chat_id` группы (отрицательное число) можно взять из `https://api.telegram.org/bot<TOKEN>/getUpdates` после сообщения в группе, где есть бот.

После изменения секрета: `kubectl apply -f ...alertmanager-secret.yaml`. Оператор перечитает конфиг сам.

### Известные недоделки мониторинга

- **Grafana не достучалась до Loki.** Скорее всего дело в том, что `grafana/loki-stack` (чарт deprecated) ставит старый Loki `2.6.x`, а health-check datasource в новых версиях Grafana его не проходит. Варианты: поднять образ Loki в `loki-values.yaml` (`loki.image.tag: 2.9.x`) или перейти на чарт `grafana/loki` (single binary) + `grafana/promtail`. URL `http://loki.logging.svc.cluster.local:3100` правильный для релиза `loki` в `logging`. Проверить из пода Grafana: `curl http://loki.logging.svc.cluster.local:3100/ready`.
- В `loki-values.yaml` опечатка `promtail.enables` вместо `enabled`. Сейчас это ни на что не влияет (promtail в loki-stack и так включён по умолчанию), но лучше поправить.
- `ServiceMonitor` ищет Service с label `app: web-app` и портом `http` по пути `/metrics`. У Service в чарте web-app **нет labels**, а во Flask **нет `/metrics`**. Поэтому метрики приложения пока берутся только из cAdvisor/kube-state-metrics, то есть CPU/RAM контейнеров.
- В `alert-rule.yaml` `WebAppHighMemoryUsage` фильтрует `container="web-app"`, а контейнеры в поде называются `app` и `nginx`. Правило никогда не сработает, нужно `container=~"app|nginx"` или просто `pod=~"web-app-.*"`.

---

## 7. Сводка секретов и откуда они берутся

| Что | Где хранится | Где используется |
| --- | --- | --- |
| Пароль root GitLab | генерируется чартом, `gitlab-gitlab-initial-root-password` | UI GitLab |
| Пароли postgres/redis | `gitlab-secrets.yaml` + values postgres/redis | GitLab ↔ БД |
| Ключи MinIO | `gitlab-minio.yaml`, `gitlab-secrets.yaml` (`gitlab-minio-secret`) | object storage GitLab |
| SSH-ключ `id_ed25519_helm` | pub → deploy key (write) в helm-репо; priv → CI var `HELM_REPO_SSH_KEY` в app-репо **и** secret repo в ArgoCD | CI пушит тег; ArgoCD читает чарт |
| Deploy token `read_registry` | создаётся в app-репо, прописывается в `web-app/values.yaml` → `registry.*` | imagePullSecret `gitlab-registry-key` |
| `APP_SECRET` | `web-app/values.yaml` → `secret.appSecretValue` | выводится на главной странице |
| Telegram bot token / chat_id | `alertmanager-secret.yaml` | Alertmanager |
| Пароль admin ArgoCD | `argocd-initial-admin-secret` | UI ArgoCD |
| Пароль admin Grafana | `monitoring-stack-grafana` | UI Grafana |

---

## 8. Нехватка памяти

GitLab (webservice ~1.2–2 GiB, sidekiq до 1.8 GiB, postgres до 1.5 GiB, gitaly, registry, runner + dind-джобы) почти целиком съедает 8 GiB minikube. Мониторинг вместе с ним не помещается. Что можно сделать:

- поднимать по очереди: сначала GitLab + ArgoCD + web-app, прогнать CI/CD, потом
  `kubectl -n gitlab scale deploy --all --replicas=0` (кроме registry, если нужен pull образа) и ставить мониторинг;
- или стартовать minikube с большей памятью (`--memory=12288`), если хватает RAM хоста.

---

## 9. Быстрый чек-лист

1. `/etc/hosts` → `192.168.49.2 *.minikube.local`
2. `minikube start --memory=8192 --cpus=4 --cni=flannel --insecure-registry=registry.minikube.local` + `addons enable ingress`
3. `kubectl apply -f helm_repo/helm_apps/infra/gitlab/manifests/` → postgres → redis → gitlab
4. Пользователь `rymond044`, проекты `final_k8s_app_repo` / `final_k8s_helm_repo`, пуш кода
5. `ssh-keygen` → deploy key (write + push в protected `main`) в helm-репо, `HELM_REPO_SSH_KEY` (не protected) в app-репо
6. Deploy token `read_registry` в app-репо → `registry.username/password` в `web-app/values.yaml` → пуш
7. ArgoCD `install.yaml` → `infra/argocd/*` → repo secret с тем же ключом → `kubectl create ns app` → `application.yaml`
8. `git tag dev-vX.Y.Z && git push origin dev-vX.Y.Z` → проверить, что ArgoCD передеплоил
9. (по памяти) мониторинг: `alertmanager-secret` → `loki-stack` → `kube-prometheus-stack` (релиз `monitoring-stack`) → rule + servicemonitor
