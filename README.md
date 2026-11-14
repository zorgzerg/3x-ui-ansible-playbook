# Ansible: Deploy 3x-ui on Ubuntu

Этот плейбук разворачивает на Ubuntu-хосте `3x-ui` и подготавливает его для работы `VLESS`, `Hysteria2` и `AmneziaWG` прокси-протоколов.

## Файлы

```bash
playbook.yml                   legacy entry point для всех hosts
relay.yml                      безопасный запуск только для xui_relay
foreign.yml                    запуск только для xui_foreign
playbooks/
  00_bootstrap.yml             создание пользователя (первый запуск)
  01_base.yml                  обновление ОС + SSH hardening + assertions
  02_docker.yml                Docker CE + Compose plugin
  03_xui.yml                   3x-ui контейнер
  04_nginx.yml                 Nginx + TLS (install, certs, site configs)
  05_firewall.yml              UFW
group_vars/all.yml             все переменные
group_vars/xui_relay.yml       значения по умолчанию для relay
group_vars/xui_foreign.yml     роль foreign
templates/
  compose.yml.j2               docker compose (условный по режиму)
  nginx_panel.conf.j2          nginx: HTTP→HTTPS + reverse proxy
  nginx_sni_router.conf.j2     nginx stream: SNI-роутер
inventory.example.yml          пример инвентаря
requirements.yml               community.general
```

## Быстрый старт

```bash
# 1. Установить коллекцию
ansible-galaxy collection install -r requirements.yml

# 2. Подготовить инвентарь
cp inventory.example.yml inventory.yml
# указать FQDN своего хоста

# 3. Проверить/изменить переменные
#    (см. group_vars/all.yml)

# 4. Запустить выполнение
ansible-playbook -i inventory.yml playbook.yml
```

На первом запуске при подключении `root` playbook может автоматически создать
`xui_admin_user` (по умолчанию `xuiadmin`), выдать ему `sudo` и скопировать
`/root/.ssh/authorized_keys` из `root`.

## Relay и foreign

Для двухузловой схемы inventory разделён на две группы:

```yaml
all:
  children:
    xui_foreign:
      hosts:
        foreign.example.com:
          ansible_host: 203.0.113.10
          xui_panel_domain: panel-foreign.example.com
          xui_hysteria_domain: hyst-foreign.example.com
          xui_vpn_domain: www.example.com
          xui_protected: true
    xui_relay:
      hosts:
        relay.example.com:
          ansible_host: 198.51.100.20
          xui_panel_domain: panel-relay.example.com
          xui_hysteria_domain: hyst-relay.example.com
          xui_vpn_domain: www.example.org
```

Существующий production foreign должен оставаться с `xui_protected=true`.
Все playbook завершаются на таком хосте до первой изменяющей задачи. Relay
разворачивается отдельной точкой входа:

```bash
# Сначала убедиться, что в выборке только relay.
ansible-playbook -i inventory.yml relay.yml --list-hosts

# Развернуть новый российский relay.
ansible-playbook -i inventory.yml relay.yml
```

`relay.yml` выбирает только группу `xui_relay`, поэтому foreign не участвует в
play вообще. `foreign.yml` предназначен для новых foreign-серверов. Чтобы
намеренно применить его к защищённому существующему серверу, нужны одновременно
явный entry point и снятие защиты:

```bash
ansible-playbook -i inventory.yml foreign.yml \
  -e xui_allow_protected_changes=true
```

Без этого параметра запуск остановится с ошибкой. Обычный `playbook.yml` оставлен
для обратной совместимости с одноузловым inventory, но защита
`xui_protected=true` действует и в нём.

Основной плейбук управляет ОС, Docker, 3x-ui, Nginx, TLS и UFW.

## Режимы (`xui_mode`)

### `basic`

```bash
Клиент → server:2053  (панель)
Клиент → server:443   (VLESS+Reality)
```

Дополнительных переменных не требует.

### `nginx_simple`

```bash
Браузер    → server:443   → Nginx → 127.0.0.1:2053 (панель)
VPN-клиент → server:8443  → Xray напрямую
```

Обязательные переменные:

```yaml
xui_mode: nginx_simple
xui_panel_domain: panel.example.com
xui_use_letsencrypt: true
xui_certbot_email: you@example.com
```

### `nginx_sni` (по умолчанию)

```bash
Любой клиент → server:443

                    ↓ Nginx читает SNI
    ┌───────────────────────────────────────────┐
    │ panel.example.com → nginx-http → панель   │
    │ vpn.example.com   → Xray (TLS не тронут)  │
    │ неизвестный SNI   → www.yahoo.com (маска) │
    └───────────────────────────────────────────┘
```

Обязательные переменные:

```yaml
xui_mode: nginx_sni
xui_panel_domain: panel.example.com
xui_vpn_domain: vpn.example.com
xui_use_letsencrypt: true
xui_certbot_email: you@example.com
```

Для нескольких VLESS-инбаундов задайте список в `host_vars` или inventory YAML:

```yaml
xui_vless_inbounds:
  - sni: www.example.com
    host_port: 8443
    container_port: 443
  - sni: www.example.org
    host_port: 8444
    container_port: 444
```

Все клиенты подключаются к TCP-порту 443 сервера. Nginx выбирает маршрут по
SNI, а Docker передаёт соединение на указанный порт контейнера. В 3x-ui нужно
самостоятельно создать отдельный VLESS-инбаунд на каждом `container_port`
с адресом прослушивания `0.0.0.0` или пустым (внутри контейнера).
Для Reality его `serverNames`, клиентский SNI и маршрут должны совпадать;
Reality target и параметры клиента настраиваются в панели. SNI панели и всех
инбаундов должны различаться без учёта регистра, порты — быть уникальными.

Пустой список сохраняет прежнее поведение: `xui_vpn_domain` направляется на
`xui_xray_backend_port` и порт 443 контейнера. Непустой список полностью
заменяет этот маршрут. Внутренние TCP-порты публикуются только на localhost.
После изменения списка примените вместе слои `xui` и `nginx`; пересоздание
контейнера может ненадолго прервать соединения:

```bash
ansible-playbook -i inventory.yml playbook.yml --tags xui,nginx
```

Для VLESS+Reality отдельный сертификат на SNI не выпускается: Reality использует
сертификат настроенного целевого сайта.

## Let's Encrypt в режиме webroot

В nginx-режимах сертификаты выпускаются через `certbot --webroot`.

- Nginx не останавливается для HTTP-01 challenge.
- Challenge-файлы обслуживаются из `xui_acme_webroot`.
- Можно выпускать отдельный сертификат для домена Hysteria2 (`xui_hysteria_domain`).
- Состав SAN сверяется с inventory; сертификат перевыпускается при отсутствующих или лишних доменах.

## Hysteria2 + VLESS TCP Reality

Важно: это **два разных inbound**. Один inbound не может одновременно быть `hysteria2` и `vless+reality`.

- `VLESS TCP + Reality` работает по TCP (обычно через 443) и использует отдельные поля Reality.
- `Hysteria2` работает по UDP/QUIC и требует отдельного UDP-порта (например `8443/udp`).

- Настраивает серверную основу для VLESS+Reality (через ваш текущий режим `xui_mode`).
- Открывает UDP-порт для Hysteria2 через UFW:

Обязательные переменные:

```yaml
xui_hysteria_enabled: true
xui_hysteria_port: 8443
xui_hysteria_domain: v.example.com
```

При `xui_use_letsencrypt: true` playbook также выпускает отдельный сертификат для `xui_hysteria_domain`.
Пути по умолчанию:

- `/etc/letsencrypt/live/<xui_hysteria_domain>/fullchain.pem`
- `/etc/letsencrypt/live/<xui_hysteria_domain>/privkey.pem`

После изменения примените конфигурацию контейнера и firewall:

```bash
ansible-playbook -i inventory.yml playbook.yml --tags xui,firewall
```

### Что нужно сделать в 3x-ui UI

1. Создать inbound `vless`:

    - Network: `tcp`
    - Security: `reality`
    - Port: 443 в контейнере для настройки по умолчанию; для списка — `container_port`

2. Создать отдельный inbound `hysteria`:

    - Port: `xui_hysteria_port` (например 8443)
    - Transport: UDP/QUIC

3. Для клиентов использовать разные профили/ссылки:

    - профиль VLESS+Reality
    - профиль Hysteria2

## AmneziaWG

Для созданного в 3x-ui inbound плейбук пробрасывает UDP-порт контейнера и
открывает его в UFW:

```yaml
xui_amneziawg_enabled: true
xui_amneziawg_port: 25204
```

Применить конфигурацию контейнера и firewall:

```bash
ansible-playbook -i inventory.yml playbook.yml --tags xui,firewall
```

## Ключевые переменные

| Переменная | По умолчанию | Назначение |
| --- | --- | --- |
| `xui_mode` | `nginx_sni` | режим: `basic`, `nginx_simple`, `nginx_sni` |
| `xui_admin_user` | `xuiadmin` | пользователь, добавляемый в группу docker |
| `xui_image_repository` | `ghcr.io/mhsanaei/3x-ui` | репозиторий Docker-образа 3x-ui |
| `xui_version` | `v3.11.0` | фиксированная версия 3x-ui |
| `xui_server_role` | `standalone` | роль: `standalone`, `relay`, `foreign` |
| `xui_protected` | `false` | запрет любых изменений хоста без явного override |
| `xui_ssh_port` | `1337` | порт SSH |
| `xui_panel_port` | `2053` | внутренний порт панели |
| `xui_sub_port` | `2096` | локальный backend-порт подписок для nginx `/sub/` |
| `xui_vless_port` | `443` | внешний порт Xray (только basic) |
| `xui_xray_backend_port` | `8443` | локальный порт Xray (nginx режимы) |
| `xui_hysteria_enabled` | `true` | включить проброс и UFW-правило для Hysteria2 UDP |
| `xui_hysteria_port` | `443` | внешний и контейнерный UDP-порт Hysteria2 |
| `xui_amneziawg_enabled` | `true` | включить проброс и UFW-правило для AmneziaWG UDP |
| `xui_amneziawg_port` | `25204` | внешний и контейнерный UDP-порт AmneziaWG |
| `xui_panel_domain` | `""` | домен панели (nginx режимы) |
| `xui_vpn_domain` | `""` | SNI одного VPN-инбаунда при пустом списке |
| `xui_vless_inbounds` | `[]` | список SNI, локальных портов и портов контейнера для `nginx_sni` |
| `xui_nginx_stream_map_hash_bucket_size` | `128` | размер hash bucket для `stream map` SNI-доменов |
| `xui_panel_path` | `"/"` | URL-префикс панели |
| `xui_use_letsencrypt` | `true` | Let's Encrypt или самоподписанный |
| `xui_certbot_email` | задан в `group_vars/all.yml` | email для certbot |
| `xui_acme_webroot` | `/var/www/certbot` | webroot для HTTP-01 challenge |
| `xui_manage_ssh_hardening` | `false` | применять hardening sshd |
| `xui_hysteria_domain` | `""` | домен для отдельного LE-сертификата Hysteria2 |
| `xui_firewall_extra_tcp_ports` | `[]` | дополнительные TCP-порты, которые UFW должен сохранить |
| `xui_firewall_extra_udp_ports` | `[]` | дополнительные UDP-порты, которые UFW должен сохранить |

## Варианты запуска по тегам

```bash
# Только Docker + контейнер
ansible-playbook -i inventory.yml playbook.yml --tags docker,xui

# Только nginx + сертификат
ansible-playbook -i inventory.yml playbook.yml --tags nginx,cert

# Только файрволл
ansible-playbook -i inventory.yml playbook.yml --tags firewall

# Запуск отдельного sub-playbook напрямую (без playbook.yml)
ansible-playbook -i inventory.yml playbooks/03_xui.yml
```

## Замечания по безопасности

- На используемом foreign всегда задавайте `xui_protected=true` в inventory.
- Перед запуском relay проверяйте выборку командой `relay.yml --list-hosts`.
- `xui_manage_ssh_hardening: false` по умолчанию — включайте только после подтверждения входа по ключу.
- При смене SSH-порта anti-lockout правило UFW автоматически сохраняет текущий порт до тех пор, пока не будет подтверждено подключение по новому.
- После первого запуска сохраните сгенерированные 3x-ui учётные данные и смените их при необходимости.
