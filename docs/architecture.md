# Архитектура

Документ описывает состав лаборатории и связи между компонентами. Правила hardening находятся в [security-policy.md](security-policy.md), результаты проверок — в [security-audit.md](security-audit.md).

## Узлы

| Узел | Роль | ОС |
|---|---|---|
| Control node | Управляет конфигурацией, запускает Ansible | Ubuntu |
| Managed node | Получает настройки hardening | Rocky Linux 9.9 |

## Схема

```text
┌──────────────────────────────┐
│ Ubuntu Control Node          │
│ Ansible / Inventory          │
│ SSH private key              │
└──────────────┬───────────────┘
               │ SSH / TCP 2222
               │ user: protected
               ▼
┌──────────────────────────────┐
│ Rocky Linux Managed Node     │
│ OpenSSH                      │
│ firewalld                    │
│ Fail2ban                     │
│ SELinux                      │
│ Ansible-managed configuration│
└──────────────────────────────┘
```

## Поток управления

1. Оператор запускает playbook на Ubuntu.
2. Ansible подключается к Rocky Linux по SSH.
3. Привилегированные задачи выполняются через become.
4. Роли управляют конфигурацией security-компонентов.
5. Handlers применяют необходимые изменения служб.
6. Роль `security_tests` проверяет выбранные параметры.

## Компоненты

- **OpenSSH** — удалённое административное подключение.
- **firewalld** — host-based firewall.
- **Fail2ban** — реагирование на повторяющиеся неудачные попытки SSH-аутентификации.
- **SELinux** — обязательный контроль доступа (MAC).
- **Ansible** — управление конфигурацией и автоматизированные проверки.

## Организация Ansible

Playbook использует роли `selinux`, `ssh_hardening`, `firewall`, `fail2ban`, `security_tests`. Переменные находятся в `group_vars/servers.yml`; параметры SSH формируются из Jinja2 template. Рабочий `inventory.ini` не должен попадать в Git.

## Границы

Лаборатория описывает один control node и один managed node. Централизованный сбор логов, SIEM, IDS/IPS, HA, резервирование и управление секретами в текущую архитектуру не входят.
