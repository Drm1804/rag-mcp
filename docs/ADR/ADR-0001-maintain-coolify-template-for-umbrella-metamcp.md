# ADR-0001: Maintain a Coolify service template for Umbrella MetaMCP

- Status: Accepted
- Date: 2026-09-07
- Implemented: 2026-09-07 — [`docker-compose.yaml`](../../docker-compose.yaml)
- Revised: 2026-09-08 — Cloudflare Tunnel added as the only public access path; application build
  pinned to Umbrella source after the published GHCR image proved inaccessible and amd64-only
- Tags: `mcp`, `metamcp`, `umbrella`, `coolify`, `docker-compose`, `cloudflare-tunnel`

## Context

Coolify уже распространяет MetaMCP через собственный service template:
[`templates/compose/metamcp.yaml`](https://github.com/coollabsio/coolify/blob/main/templates/compose/metamcp.yaml).
Этот шаблон представляет собой один Docker Compose-файл и описывает весь deploy stack:

- контейнер MetaMCP;
- отдельный PostgreSQL с persistent volume;
- переменные окружения и генерируемые Coolify секреты;
- публичный URL приложения через magic variables Coolify;
- healthchecks и порядок запуска сервисов.

Официальный шаблон использует образ upstream MetaMCP от `metatool-ai`. Нам нужна та же модель
деплоя, но для дистрибутива
[`Umbrella-IT-Group/metamcp`](https://github.com/Umbrella-IT-Group/metamcp).

Публичный доступ к нашему экземпляру выполняется через remotely-managed Cloudflare Tunnel, а не
через Coolify proxy. Поэтому connector `cloudflared` должен быть частью того же Compose stack и
обращаться к MetaMCP по внутреннему имени сервиса.

Для этого не требуется форкать исходный код MetaMCP, собирать отдельную deploy-платформу или
переносить в этот ADR конфигурацию MCP-серверов, namespace'ов и endpoint'ов. Нужен только свой
вариант Coolify template, который можно развивать вместе с требованиями нашего деплоя.

## Decision

Репозиторий `RAG-mcp` будет содержать и развивать **собственный Docker Compose template для
деплоя Umbrella MetaMCP в Coolify**.

За основу берётся структура официального MetaMCP template из каталога Coolify. Наш шаблон должен
сохранять тот же минимальный состав и соглашения:

1. Сервис приложения собирается из публичного репозитория Umbrella MetaMCP его собственным
   `Dockerfile`. Build context закрепляется на точном commit SHA из ветки `umbrella`; mutable branch
   name и тег `latest` не используются.
2. PostgreSQL входит в тот же Compose stack и хранит данные в именованном persistent volume.
3. Подключение к PostgreSQL, URL приложения и секрет аутентификации задаются через переменные
   окружения Coolify; секреты не коммитятся в git.
4. Публичный origin задаётся обязательной переменной `METAMCP_PUBLIC_URL` и передаётся в
   `APP_URL` и `NEXT_PUBLIC_APP_URL`. Coolify proxy для сервиса приложения не используется.
5. `cloudflared` запускается из официального образа с remotely-managed tunnel token из secret
   variable `CLOUDFLARE_TUNNEL_TOKEN` и зависит от успешного healthcheck приложения.
6. В Cloudflare tunnel route направляется на `http://app:12008`; host port не публикуется.
7. Для приложения и PostgreSQL определяются healthchecks, а приложение зависит от готовности БД.
8. Docker Compose-файл является source of truth для состава stack, переменных, volumes и
   healthchecks. Coolify разворачивает его напрямую из `RAG-mcp`.

Шаблон не подключается к каталогу Coolify во время выполнения и не копируется из него
автоматически. Официальный `metamcp.yaml` служит референсом. Изменения в нём периодически
просматриваются и при необходимости переносятся в наш template осознанным commit/PR.

Допустимая дельта относительно официального шаблона ограничена требованиями Umbrella MetaMCP и
нашего окружения: source build вместо недоступного amd64-only image, Cloudflare Tunnel,
дополнительные поддерживаемые переменные окружения и необходимые настройки Coolify.

## Scope

В рамках ADR:

- собственный Coolify-compatible Docker Compose template;
- нативная сборка закреплённого commit Umbrella MetaMCP его собственным Dockerfile;
- сервис Umbrella MetaMCP, его PostgreSQL и connector `cloudflared`;
- Coolify variables, secrets, volume, healthchecks и dependency wiring;
- внутренний origin Cloudflare Tunnel и публичный URL приложения;
- процедура сопровождения template относительно официального каталога Coolify и релизов
  Umbrella MetaMCP.

Вне рамок ADR:

- форк или изменение исходного кода Umbrella MetaMCP;
- собственный Dockerfile, патчи исходного кода или публикация нашего образа MetaMCP;
- декларативное создание MCP-серверов, namespace'ов, endpoint'ов, пользователей и API-ключей;
- конфигурация подключаемого `rag-gateway`;
- backup policy, Cloudflare Access policy, DNS lifecycle и выбор конкретной версии образа.

Эти вопросы могут быть зафиксированы отдельными ADR, если для них появятся самостоятельные
архитектурные решения.

## Consequences

Положительные:

- деплой Umbrella MetaMCP воспроизводится из git;
- структура остаётся знакомой и совместимой с моделью service templates Coolify;
- изменения Umbrella-специфичного деплоя не зависят от принятия правок в официальный каталог;
- обновление template и образа проходит через обычный review в `RAG-mcp`.
- приложение не требует входящего host port: `cloudflared` устанавливает исходящее соединение.
- сборка выполняется нативно на Docker host и не требует доступа к приватному Umbrella GHCR image.

Отрицательные:

- мы сами отвечаем за актуальность template;
- исправления официального MetaMCP template не попадут к нам автоматически;
- расхождения с официальным шаблоном необходимо удерживать минимальными и документированными.
- tunnel token и published application route необходимо создать и сопровождать в Cloudflare.
- первый деплой и обновления MetaMCP требуют полной source build и занимают больше времени, чем
  pull готового образа.

## Alternatives considered

- **Использовать официальный MetaMCP template без изменений.** Отклонено: он запускает upstream
  MetaMCP, а не Umbrella MetaMCP.
- **Поддерживать форк Coolify ради одного service template.** Отклонено как несоразмерная стоимость
  сопровождения.
- **Форкнуть Umbrella MetaMCP и хранить deploy-конфигурацию вместе с кодом приложения.**
  Отклонено: задача ограничена конфигурацией деплоя, изменений приложения не требуется.
