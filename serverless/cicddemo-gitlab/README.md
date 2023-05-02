# Демонстрация использования СI/CD для разворачивания serverless-приложений в Yandex Cloud

В этом примере демонстрируется тестовое приложение, которое разворачивается в Yandex Cloud на serverless-компонентах. Процесс сборки, тестирования и деплоя приложения полностью автоматизирован с помощью GitLab CI/CD.

## Используемые сервисы и технологии

* [GitLab Runner](https://docs.gitlab.com/runner/) — для автоматизации процессов.
* [Docker](https://www.docker.com/) — для контейнеризации приложения.
* [Yandex Container Registry](https://cloud.yandex.ru/services/container-registry) — для хранения Docker-образа приложения.
* [Yandex Serverless Containers](https://cloud.yandex.ru/services/serverless-containers) — для развертывания приложения в бессерверном режиме.
* [Yandex API Gateway](https://cloud.yandex.ru/services/api-gateway) — отвечает за прием трафика в приложении.
* [Yandex Managed Service for YDB](https://cloud.yandex.ru/services/ydb) в режиме serverless Document API — для хранения данных приложения.
* [Yandex Lockbox](https://cloud.yandex.ru/services/lockbox) — для хранения и доставки секретов приложения.

## Как развернуть демо-приложение у себя

Чтобы самостоятельно развернуть этот пример, вам потребуется репозиторий в GitLab и аккаунт в Yandex Cloud.

Локально должны быть установлены и настроены приложения и утилиты:
* [Интерфейс командной строки Yandex Cloud](https://cloud.yandex.ru/docs/cli/).
* [Утилита потоковой обработки JSON-файлов `jq`](https://stedolan.github.io/jq/).
* [Утилита потоковой обработки YAML-файлов `yq`](https://github.com/mikefarah/yq).
* [Python версии 3.8 или выше](https://www.python.org/).
* Библиотеки Python, перечисленные в [application/requirements.txt](application/requirements.txt).

Также обратите внимание, что сервис Yandex Lockbox находится на стадии [Preview](https://cloud.yandex.ru/docs/overview/concepts/launch-stages). Доступ предоставляется по запросу. Для использования сервиса в своем облаке [запросите доступ](https://cloud.yandex.ru/services/lockbox#preview-form).

## Первоначальная настройка

1. Скопируйте содержимое этой директории в корень своего репозитория GitLab.
1. Для развертывания базовой инфраструктуры в корне репозитория выполните команду:

   ```bash
   YC_CLOUD_ID=<идентификатор вашего облака> ./bootstrap.sh
   ```

В конце выполнения скрипт выведет значения переменных окружения для GitLab. Сохраните их, они потребуются в дальнейшем.

## О приложении

В этом демонстрационном проекте реализовано простое web-приложение на [Django](https://www.djangoproject.com/), имитирующее корзину товаров e-commerce сервиса.

В базе данных хранятся описания товаров, БД наполняется тестовыми данными из bootstrap-скрипта. Состояние корзины товаров сервис хранит в сессии пользователя.

Django-приложение разворачивается в serverless-контейнере, доставка секретов в приложение осуществляется безопасно с помощью сервиса Yandex Lockbox. API Gateway принимает запросы от пользователей и перенаправляет их в контейнер приложения.

## Окружения

В этом примере мы используем два окружения для деплоя нашего приложения:
* **prod** — продакшн окружение, доступное пользователям.
* **testing** — тестовое окружение, используется для проверки приложения перед релизом в **prod**.

Для каждого из окружений в bootstrap-скрипте создается отдельный каталог в Yandex Cloud, а так же отдельный набор статических ресурсов — БД, сервисные аккаунты и т. д. Таким образом все окружения изолированы друг от друга на уровне настроек [Yandex Identity and Access Management](https://cloud.yandex.ru/services/iam).

Окружения **prod** и **testing** содержат в себе по одному развернутому стенду.

Дополнительно в проекте используется общий каталог infra — в него публикуются все собранные Docker-образы приложения. Сервисные аккаунты окружений prod и testing имеют ограниченные права в infra-каталоге, которых достаточно для того, чтобы скачивать образы из infra-каталога. Публикация образов в infra-каталог осуществляется от отдельного сервисного аккаунта builder.

## Автоматизированные процессы

После добавления файла конфигурации сценария CI `.gitlab-ci.yml` запустится сценарий сборки.

Он выглядит следующим образом:
* Сборка образа из ветки и загрузка в реестр Container Registry.
* Тестовое развертывание приложения в окружение `testing`.
* Тестирование приложения — на данный момент командой `curl` мы проверяем доступность главной страницы приложения.
* Удаление тестового приложения.
* Развертывание приложения в продакшн. Дополнительно используется опция с окружениями, которая позволяет сохранить развертывание и восстановить его при необходимости.

При публикации изменений в ветку `main` (при влитии PR или прямом пуше в main) автоматически снова запустится сценарий сборки. Таким образом изменения в PR автоматически докатываются в развернутый стенд.