# Ansible: Deploy 3x-ui on Ubuntu

Этот плейбук разворачивает на Ubuntu-хосте `3x-ui` и подготавливает его для работы `VLESS`, `Hysteria2` и `AmneziaWG` прокси-протоколов.

## Файлы

```bash
playbook.yml                   главная точка входа (import_playbook)
playbooks/
  00_bootstrap.yml             создание пользователя (первый запуск)
  01_base.yml                  обновление ОС + SSH hardening + assertions
  02_docker.yml                Docker CE + Compose plugin
  03_xui.yml                   3x-ui контейнер
  04_nginx.yml                 Nginx + TLS (install, certs, site configs)
  05_firewall.yml              UFW
group_vars/all.yml             все переменные
templates/
  compose.yml.j2               docker compose (условный по режиму)
  nginx_panel.conf.j2          nginx: HTTP→HTTPS + reverse proxy
  nginx_sni_router.conf.j2     nginx stream: SNI-роутер
inventory.example.ini          пример инвентаря
requirements.yml               community.general
```

## Быстрый старт

```bash
# 1. Установить коллекцию
ansible-galaxy collection install -r requirements.yml

# 2. Подготовить инвентарь
cp inventory.example.ini inventory.ini
# указать FQDN своего хоста

# 3. Проверить/изменить переменные
#    (см. group_vars/all.yml)

# 4. Запустить выполнение
ansible-playbook -i inventory.ini playbook.yml
```

На первом запуске при подключении `root` playbook может автоматически создать
`xui_admin_user` (по умолчанию `pilot`), выдать ему `sudo` и скопировать
`/root/.ssh/authorized_keys` из `root`.

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

Сертификат на VPN-домен **не нужен** — Reality использует чужой сертификат (yahoo.com).

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
ansible-playbook -i inventory.ini playbook.yml --tags xui,firewall
```

### Что нужно сделать в 3x-ui UI

1. Создать inbound `vless`:

    - Network: `tcp`
    - Security: `reality`
    - Port: обычно 443

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
ansible-playbook -i inventory.ini playbook.yml --tags xui,firewall
```

## Ключевые переменные

| Переменная | По умолчанию | Назначение |
| --- | --- | --- |
| `xui_mode` | `basic` | режим: `basic`, `nginx_simple`, `nginx_sni` |
| `xui_admin_user` | `pilot` | пользователь, добавляемый в группу docker |
| `xui_image_repository` | `ghcr.io/mhsanaei/3x-ui` | репозиторий Docker-образа 3x-ui |
| `xui_version` | `v3.7.0` | фиксированная версия 3x-ui |
| `xui_ssh_port` | `1337` | порт SSH |
| `xui_panel_port` | `2053` | внутренний порт панели |
| `xui_sub_port` | `2096` | локальный backend-порт подписок для nginx `/sub/` |
| `xui_vless_port` | `443` | внешний порт Xray (только basic) |
| `xui_xray_backend_port` | `8443` | локальный порт Xray (nginx режимы) |
| `xui_hysteria_enabled` | `true` | включить проброс и UFW-правило для Hysteria2 UDP |
| `xui_hysteria_port` | `8443` | внешний и контейнерный UDP-порт Hysteria2 |
| `xui_amneziawg_enabled` | `true` | включить проброс и UFW-правило для AmneziaWG UDP |
| `xui_amneziawg_port` | `25204` | внешний и контейнерный UDP-порт AmneziaWG |
| `xui_panel_domain` | `""` | домен панели (nginx режимы) |
| `xui_vpn_domain` | `""` | домен VPN (только nginx_sni) |
| `xui_nginx_stream_map_hash_bucket_size` | `128` | размер hash bucket для `stream map` SNI-доменов |
| `xui_panel_path` | `"/panel/"` | URL-префикс панели |
| `xui_use_letsencrypt` | `false` | Let's Encrypt или самоподписанный |
| `xui_certbot_email` | `""` | email для certbot |
| `xui_acme_webroot` | `/var/www/certbot` | webroot для HTTP-01 challenge |
| `xui_manage_ssh_hardening` | `false` | применять hardening sshd |
| `xui_hysteria_domain` | `""` | домен для отдельного LE-сертификата Hysteria2 |

## Варианты запуска по тегам

```bash
# Только Docker + контейнер
ansible-playbook -i inventory.ini playbook.yml --tags docker,xui

# Только nginx + сертификат
ansible-playbook -i inventory.ini playbook.yml --tags nginx,cert

# Только файрволл
ansible-playbook -i inventory.ini playbook.yml --tags firewall

# Запуск отдельного sub-playbook напрямую (без playbook.yml)
ansible-playbook -i inventory.ini playbooks/03_xui.yml
```

## Замечания по безопасности

- `xui_manage_ssh_hardening: false` по умолчанию — включайте только после подтверждения входа по ключу.
- При смене SSH-порта anti-lockout правило UFW автоматически сохраняет текущий порт до тех пор, пока не будет подтверждено подключение по новому.
- После первого запуска смените логин/пароль `admin/admin` в панели `3x-ui`.
