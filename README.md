# Django site in Minikube and in Yandex Cloud

Докеризированный сайт на Django для экспериментов с Kubernetes.

## Как запустить в Minikube

1. Перейдите в окружение Docker в Minikube:

    ```bash
    eval $(minikube docker-env)
    ```

2. Перейдите в каталог с Docker-образом:

    ```bash
    cd app/k8s-test-django/backend_main_django
    ```

3. Соберите Docker-образ:

    ```bash
    docker build -t django_app:v88 .
    ```

4. Проверьте, добавился ли образ в Minikube:

    ```bash
    minikube image ls
    ```

5. Создайте файл `.env` в корневой директории проекта и добавьте переменные окружения:

    - `DEBUG` — дебаг-режим
    - `DATABASE_URL` — Данные для базы данных PostgreSQL в формате postgres://USER:PASSWORD@HOST:PORT/NAME
    - `ALLOWED_HOSTS` — [см. документацию Django](https://docs.djangoproject.com/en/3.1/ref/settings/#allowed-hosts)

    Убедитесь, что в переменной `DATABASE_URL` используется хост `postgres.default.svc.cluster.local`.

6. Установите [Helm](https://helm.sh/)

7. Разверните PostgreSQL в кластере:

    ```bash
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm install postgres bitnami/postgresql
    ```

    В качестве хоста БД используйте:

    ```bash
    postgres.default.svc.cluster.local
    ```

    Добавьте его в `DATABASE_URL` файла `.env`.

8. Сохраните пароль в переменной окружения `POSTGRES_PASSWORD`:

    ```bash
    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
    ```

    Чтобы подключиться к вашей базе данных, выполните следующую команду:

    ```bash
    kubectl run postgres-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:16.1.0-debian-11-r20 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
          --command -- psql --host postgres-postgresql -U postgres -d postgres -p 5432
    ```

9. Создайте БД и пользователя в PostgreSQL:

    ```sql
    create database Имя_вашей_бд;
    create user ваш_логин with encrypted password 'пароль_пользователя';
    grant all privileges on database Имя_вашей_бд to ваш_логин;
    ALTER DATABASE Имя_вашей_бд OWNER TO ваш_логин;
    ```

    Замените `Имя_вашей_бд` и `ваш_логин` на соответствующие значения.

10. Получите IP-адрес из команды `minikube cluster-info` и добавьте его в файл `/etc/hosts`.

11. Создайте ConfigMap в Kubernetes:

    ```bash
    kubectl apply -f k8s/configmap.yaml
    ```

12. Создайте манифесты Deployment, Service, Ingress, Secrets, Jobs для очистки сессий и миграций:

    ```bash
    kubectl apply -f k8s/deployment.yaml -f k8s/django-service.yaml -f k8s/ingress.yaml -f django-secrets-env-file.yaml -f k8s/django-clearsessions.yaml -f k8s/migrate-job.yaml 
    ```

## ConfigMap и Secrets

- **ConfigMap:**

    ConfigMap включает в себя следующие переменные.

    Замените на ваши из .env файла

    ```
      DEBUG: "True"
      ALLOWED_HOSTS: "127.0.0.1,192.168.0.87,star-burger.test"
    ```

- **Secret:**

    Secret включает в себя base64 закодированные значения для следующих переменных:

    ```
      DJANGO_SECRET_KEY: <base64-encoded-secret-key>
      DATABASE_URL: <base64-encoded-database-url>
    ```
  Замените на ваши переменные из env файла и закодируйте их в base64
  ```
    echo -n "DJANGO_SECRET_KEY" | base64
    echo -n "DATABASE_URL" | base64
  ```

## Примечания

- После изменения переменных окружения необходимо перезапустить под:

    ```bash
    kubectl delete configmap django-config
    kubectl apply -f k8s/configmap.yaml
    kubectl rollout restart deployment django
    ```

## Развертывание приложения в Yandex Cloud

Установите интерфейс командной строки Yandex Cloud (CLI) следуя [инструкциям](https://cloud.yandex.com/en/docs/cli/quickstart). Подключитесь к кластеру.

## Как задеплоить код

### Разворачиваем тестовый Nginx
    
В файле dev-test-nginx/Service.yaml замените nodePort: 30341 на свое согласно настройкам ALB-роутера.


Примените манифесты Kubernetes:
   ```bash
   kubectl apply -f dev-test-nginx/Deployment.yaml
   kubectl apply -f dev-test-nginx/Service.yaml
   kubectl apply -f dev-test-nginx/Ingress.yaml
   ```
Проверьте перейдя по домену который вам был выдан.

### Загрузка DockerImage на DockerHub
Перейдите в каталог проекта local-docker-app и соберите докер-образ:

```
docker build -t <image-name>:<tag> .
```
Переименуйте образ:
```
docker tag <image-name>:<tag> YOUR-USERNAME/<image-name>:<tag>
```
Загрузите образ на DockerHub.
```
docker push YOUR-USERNAME/<image-name>:<tag>
```

### Как подготовить dev окружение
Все необходимые манифесты для деплоя находятся в директории deploy/yc-sirius/edu-hopeful-ritchie.

В файле configmap.yaml подставьте свои значения переменных окружения.
```
kubectl -n <namespace> apply -f configmap.yaml
```
Для подключения к базе данных скачайте [SSL-сертификат](https://cloud.yandex.ru/ru/docs/managed-postgresql/operations/connect#get-ssl-cert) для postgresql командой:
```
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
     --output-document ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt
```
Сертификат будет сохранен в файле ~/.postgresql/root.crt

Создайте Secret:
```
kubectl create secret generic postgres-ssl-cert -n <namespace> --from-file=/path_to/root.crt
```

### Запуск приложения
Разверните в кластере джанго-приложение:
```
kubectl -n <namespace> apply -f deployment.yaml -f configmap.yaml
```
В файле django-service.yaml змените значение nodePort: 30341 на свое согласно настройкам ALB-роутера. Запустите сервис:
```
kubectl -n <namespace> apply -f django-service.yaml
```
Примените миграции:

```
kubectl -n <namespace> apply -f migrate-job.yaml
```
Запустите очисту сессий:
```
kubectl -n <namespace> apply -f django-clearsessions.yaml
```
Сайт доступен по ссылке https://edu-hopeful-ritchie.sirius-k8s.dvmn.org/

