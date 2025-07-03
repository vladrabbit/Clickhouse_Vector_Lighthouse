# Домашнее задание к занятию 4 «Работа с roles»

Данный плейбук предназначен для установки `Clickhouse`, `Vector` и `Lighthouse` на хосты, которые необходио указать в файле  `./inventory/hosts.yaml`.

## group_vars

| Файл  | Переменная  | Назначение  |
|:---|:---|:---|
| `clickhouse.yml` | `clickhouse_listen_host` | адрес на котором будет запускаться сервис Clickhouse |
| `clickhouse.yml` | `clickhouse_users_custom` | параметры пользователя с правами доступа к БД |
| `clickhouse.yml` | `clickhouse_users_custom` var `name` | имя пользователя с правами доступа к БД |
| `clickhouse.yml` | `clickhouse_users_custom` var `password` | пароль пользователя с правами доступа к БД |
| `clickhouse.yml` | `clickhouse_users_custom` var `networks` | подсети откуда разрешен доступ к БД определен переменной конфигурации Clickhouse `clickhouse_networks_default`  |
| `clickhouse.yml` | `clickhouse_users_custom` var `profile` | профиль пользователя |
| `clickhouse.yml` | `clickhouse_users_custom` var `quota` | квота пользователя |
| `lighthouse.yml` | `lighthouse_nginx_user` | пользователь под которым будет запускаться сервис `Nginx` |
| `vector.yml` | `vector_config` var `sinks`  | параметры синхронизации vector с clickhouse,  описание в документации [Vector](https://vector.dev/docs/reference/configuration/sinks/clickhouse/) |
| `vector.yml` | `vector_config` var `sources`  | параметры настройки источника логов ,  описание в документации [Vector](https://vector.dev/docs/reference/configuration/sources/file/) |


## Inventory файл находится по пути ./inventory/hosts.yaml

Группа "clickhouse" состоит из 1 хоста `clickhouse-server`

Группа "vector" состоит из 1 хоста `vector-server`

Группа "vector" состоит из 1 хоста `lighthouse-server`

## Файл requirements.yaml с перечнем источников для установки ролей

Используются 3 роли:

1. clickhouse

```yaml
  - src: git@github.com:AlexeySetevoi/ansible-clickhouse.git
    scm: git
    version: "1.11.0"
    name: clickhouse
```

2. vector-role

```yaml
  - name: vector-role
    src: git@github.com:aragastmatb/vector_role.git
    scm: git
    version: "main"
```

3. lighthouse
   
```yaml
  - name: lighthouse
    src: git@github.com:vladrabbit/lighthouse.git
    scm: git
    version: "1.0.0"
```

## Файл playbook site.yaml

Playbook состоит из 3 `play`.

Play "Assert clickhouse role" применяется на группу хостов "Clickhouse" и предназначен для установки и запуска `Clickhouse` 

Данный `play` запускает роль `сlickhouse`

---

Play "Assert vector role" применяется на группу хостов "Vector" и предназначен для установки и запуска `Vector`

Данный `play` запускает роль `vector-role`

---

Play "Install lighthouse" применяется на группу хостов "lighthouse" и предназначен для установки и запуска `lighthouse`

Используется роль `Lighthouse`
Объявляем `handler` для перезапуска `Nginx`.

```yaml
 handlers:
    - name: Nginx reload
      become: true
      ansible.builtin.service:
        name: nginx
        state: restarted
```
Устанавливаем `GIT` и `NGINX`. Производим первоначальную настройку веб-сервера

| Имя pretask | Описание |
|--------------|---------|
| `Lighthouse \| Install git` | Устанавливаем `git` |
| `Lighhouse \| Install nginx` | Устанавливаем `Nginx` |
| `Lighthouse \| Apply nginx config` | Применяем конфиг `Nginx`. |

Применяем конфиг `Lighthouse` для `NGINX`

| Имя posttask | Описание |
|--------------|---------|
| `Lighthouse \| Apply config` | Применяем конфиг `Nginx` для `lighthouse`. После этого перезапускаем `nginx` для применения изменений |

## Шаблоны конфигурации ./templates

Шаблон "nginx.conf.j2" используется для первичной настройки `nginx`. Мы задаем пользователя для работы `nginx` и удаляем настройки root директории по умолчанию.

Шаблон "lighthouse_nginx.conf.j2" настраивает `nginx` на работу с `lighthouse`. В нем прописываем порт 80, root директорию и index страницу.