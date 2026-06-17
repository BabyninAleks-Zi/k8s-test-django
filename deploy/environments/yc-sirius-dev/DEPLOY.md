# YC Sirius Dev Deploy

## Как собрать и опубликовать Docker image

В dev окружении образ версионируется hash-ем git-коммита, а не номером версии
вроде `v0.0.1`. В dev попадает много промежуточных сборок, и далеко не каждая
из них должна становиться релизной версией. Hash коммита уже уникален, связан с
конкретным состоянием кода и позволяет скачать старый образ, не затирая новые.

Перед сборкой задайте переменные:

```shell
DOCKERHUB_USERNAME="<your-dockerhub-username>"
IMAGE_NAME="k8s-test-django"
IMAGE="$DOCKERHUB_USERNAME/$IMAGE_NAME"
COMMIT_HASH="$(git log -1 --format=%H)"
```

В Docker Hub заранее создайте repository с таким же именем, как `IMAGE_NAME`,
или замените `IMAGE_NAME` на имя уже созданного repository.

Проверьте tag, который будет опубликован:

```shell
echo "$IMAGE:$COMMIT_HASH"
```

Соберите образ Django Site:

```shell
docker build \
  --tag "$IMAGE:$COMMIT_HASH" \
  ./backend_main_django
```

Авторизуйтесь в Docker Hub:

```shell
docker login
```

Опубликуйте образ:

```shell
docker push "$IMAGE:$COMMIT_HASH"
```

Проверьте, что образ можно скачать из Docker Hub:

```shell
docker pull "$IMAGE:$COMMIT_HASH"
```

Чтобы опубликовать образ для старого коммита, соберите код из checkout-а этого
коммита и используйте тот же принцип тегирования:

```shell
COMMIT_HASH="$(git log -1 --format=%H)"
docker build --tag "$IMAGE:$COMMIT_HASH" ./backend_main_django
docker push "$IMAGE:$COMMIT_HASH"
```

Старые образы не затираются, потому что каждый commit hash даёт отдельный
Docker tag.

## Как задеплоить код

Dev окружение в Yandex Cloud уже подготовлено Девманом. Для текущего
окружения используйте Ваш namespace, который Вам выдали в Yandex Cloud и примените его везде где увидите ниже your-namespace:

```shell
edu-your-namespace
```

Перед деплоем проверьте, что `kubectl` подключён к нужному кластеру:

```shell
kubectl config current-context
```

Ожидаемый контекст:

```text
yc-sirius-dev
```

Рабочие манифесты тестового Nginx лежат в каталоге
`deploy/environments/yc-sirius-dev/kubernetes/`.

Примените ConfigMap, Service и Deployment:

```shell
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/main-nginx-configmap.yaml
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/main-nginx-service.yaml
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/main-nginx-deployment.yaml
```

Проверьте, что Deployment успешно обновился:

```shell
kubectl rollout status deployment/main-nginx -n edu-your-namespace
```

Проверьте ресурсы в namespace:

```shell
kubectl get pods,svc,deploy,configmap -n edu-your-namespace
```

Проверьте сайт по HTTPS:

```shell
curl -I https://edu-your-namespace.yc-sirius-dev.pelid.team/
```

Если открыть сайт по HTTP, Application Load Balancer должен перенаправить
запрос на HTTPS:

```shell
curl -I http://edu-your-namespace.yc-sirius-dev.pelid.team/
```

Свежие HTTP-запросы можно увидеть в логах Nginx:

```shell
kubectl logs deployment/main-nginx -n edu-your-namespace --tail=30
```

## Как подготовить dev окружение

Namespace:

```shell
edu-your-namespace
```

Проверьте доступ к namespace:

```shell
kubectl get secrets -n edu-your-namespace
```

В окружении уже должен быть Secret `postgres` с параметрами подключения к
Managed PostgreSQL.

### Создать Secret с SSL-сертификатом PostgreSQL

Сертификат для проверки TLS-подключения хранится в ключе `root.crt` секрета
`postgres`. Для монтирования сертификата отдельным файлом создаём отдельный
Secret `postgres-root-cert`:

```shell
kubectl get secret postgres -n edu-your-namespace -o json \
  | jq '{
      apiVersion: "v1",
      kind: "Secret",
      metadata: {
        name: "postgres-root-cert",
        namespace: "edu-your-namespace"
      },
      type: "Opaque",
      data: {
        "root.crt": .data["root.crt"]
      }
    }' \
  | kubectl apply -f -
```

Проверьте, что Secret появился:

```shell
kubectl get secret postgres-root-cert -n edu-your-namespace
```

Секреты, пароли, DSN и содержимое сертификата не сохраняем в git.

### Проверить автоматическое монтирование сертификата

Создайте тестовый Pod:

```shell
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/psql-client-pod.yaml
kubectl wait pod/psql-client -n edu-your-namespace --for=condition=Ready --timeout=120s
```

Проверьте, что файл сертификата появился внутри контейнера:

```shell
kubectl exec -n edu-your-namespace psql-client -- ls -l /root/.postgresql/root.crt
```

Установите `psql` внутри тестового Pod:

```shell
kubectl exec -it -n edu-your-namespace psql-client -- bash
apt-get update
apt-get install -y postgresql-client ca-certificates
```

Проверьте подключение к PostgreSQL:

```shell
PGPASSWORD="$password" psql "host=$host port=$port dbname=$name user=$username sslmode=verify-full sslrootcert=/root/.postgresql/root.crt target_session_attrs=read-write"
```

Внутри `psql`:

```sql
SELECT current_database(), current_user, version();
\q
```

После проверки удалите тестовый Pod:

```shell
kubectl delete pod psql-client -n edu-your-namespace
```

Смена SSL-сертификата не требует пересборки Docker-образа: достаточно обновить
Secret `postgres-root-cert` и пересоздать Pod, который монтирует этот Secret.
