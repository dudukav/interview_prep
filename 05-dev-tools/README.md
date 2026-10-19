# Docker, Git и Bash

- [Краткая справка](overview.md) — Docker, Git, Bash и путь изменения до выпуска.
- [Вопросы и ответы](theory.md) — часть тем СОА по контейнеризации и CI/CD.

## `docker/`

Теория: image/container, layers, registry, volumes, networks, ports, environment, healthcheck, Compose, multi-stage build.

Практика:

- написать multi-stage Dockerfile для Go-сервиса;
- запускать процесс не от root;
- добавить healthcheck и graceful shutdown;
- связать сервис с PostgreSQL и Kafka через Compose;
- исследовать логи, процессы, сеть и файловую систему контейнера;
- объяснить разницу image, container и volume.

## `git/`

Теория: working tree, index, commit, branch, merge, rebase, remote, HEAD, конфликт.

Практика:

- создать feature branch и осмысленные commits;
- разрешить конфликт;
- сравнить merge и rebase на тестовых ветках;
- найти изменение через `log`, `show`, `diff`, `blame` и `bisect`;
- отменить локальное изменение безопасным способом.

## `bash/`

Теория: stdin/stdout/stderr, exit code, pipes, redirects, variables, quoting, processes, signals.

Практика:

- отфильтровать структурированные логи;
- найти процесс и корректно отправить сигнал;
- написать скрипт проверки готовности HTTP-сервиса;
- обработать аргументы и ошибки с понятным exit code;
- объяснить, почему неправильные кавычки и неосторожные glob-шаблоны опасны.
