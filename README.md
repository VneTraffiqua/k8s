# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Разертывание с помощью Minikube

Перед началом работы убедитесь что у вас установлены следующие инструменты:

- Инструмент для управления кластерами Kubernetes: [Kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/).
- [Minikube](https://minikube.sigs.k8s.io/docs/)

Linux:

```commandline
 minikube start --driver=virtualbox --insecure-registry true image-pull-policy Never --force
```

Для Windows необходимо установить гипервизор. Например, [virtualbox](https://www.virtualbox.org/wiki/Downloads). Windows:

```commandline
minikube start --driver=virtualbox
```

## Настройка и запуск базы данных

Создайте файл `values.yaml` в директории проекта и заполните его по шаблону:

```commandline
auth:
  enablePostgresUser: true
  postgresPassword: {your database password}
  username: {your username}
  password: {your user`s password}
  database: {your database`s name}
```

Установите пакетный менеджер для Kubernetes - [Helm](https://helm.sh/) Выполните следующие команды:
```
helm install postgre-release -f values.yaml bitnami/postgresql
```

Helm установит и запустит postgresql, а также создаст пользователя по параметрам, указанным в values.yaml.

## Переменные окружения

Создайте .env файл с переменными:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Настройка и запуск Django

Создайте образ Django-приложения в кластере:

```
minikube image build -t django-app backend_main_django/
```

Создайте объект Secret в ноде, выполнив команду:
```commandline
 kubectl create secret generic django-k8s-env --from-env-file=<path_to_env_file>
```

Запустите деплой:

```
kubectl apply -f deploy-manifest.yml
```

Запустите сервис:

```
kubectl apply -f svc-manifest.yml
```

Заупстите миграции:
```
kubectl apply -f django-migrate.yml
```

Создайте запланированное задание CronJob:
```commandline
kubectl apply -f django-clearsessions-once.yml
```

## Hosts

В папке `/etc/hosts` добавьте строку:

```
<minikube ip> star-burger.test www.star-burger.test
```
