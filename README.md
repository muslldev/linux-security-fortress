# Linux Security Fortress

**Воспроизводимый базовый hardening Rocky Linux с помощью Ansible.**

Linux Security Fortress — учебный DevSecOps-проект по автоматизации базовой защиты Linux-сервера. Он демонстрирует настройку SSH, firewalld, Fail2ban и SELinux, а также автоматизированную проверку конфигурации.

> Это лабораторный проект, а не сертифицированный security baseline и не доказательство защищённости production-системы.

## Что реализовано

- SSH hardening: key-based authentication, запрет root login и парольной аутентификации.
- Ограничение SSH-доступа через firewalld по адресу источника.
- Fail2ban jail для SSH.
- SELinux в режиме `Enforcing`; TCP/2222 имеет тип `ssh_port_t`.
- Ansible-роли для применения и проверки конфигурации.
- Проверка идемпотентности повторным запуском playbook.

## Лаборатория

| Узел | Назначение | ОС |
|---|---|---|
| Control node | Запуск Ansible | Ubuntu |
| Managed node | Защищаемый сервер | Rocky Linux 9.9 |

Ansible подключается к managed node по SSH на TCP/2222 под пользователем `protected`. Адрес control node для firewall задаётся переменной `admin_ip`.

## Технологии

`Rocky Linux 9` · `Ubuntu` · `Ansible` · `OpenSSH` · `firewalld` · `Fail2ban` · `SELinux`

## Структура репозитория

```text
.
├── README.md
├── LICENSE
├── .gitignore
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini.example
│   ├── playbook.yaml
│   ├── group_vars/servers.yml
│   └── roles/
│       ├── selinux/
│       ├── ssh_hardening/
│       ├── firewall/
│       ├── fail2ban/
│       └── security_tests/
└── docs/
    ├── architecture.md
    ├── security-policy.md
    ├── threat-model.md
    ├── security-audit.md
    └── test-plan.md
```

## Быстрый запуск

### Требования

**Control node:** Ubuntu, Ansible, SSH client, сетевой доступ и приватный ключ пользователя `protected`.

**Managed node:** Rocky Linux 9.x, доступный SSH daemon, пользователь `protected` с sudo и доступ к консоли VM для восстановления.

Установи collections:

```bash
ansible-galaxy collection install ansible.posix community.general
```

### Подготовка

```bash
git clone <REPOSITORY_URL>
cd linux-security-fortress/ansible
cp inventory.ini.example inventory.ini
```

В `inventory.ini` укажи адрес managed node, пользователя, порт и путь к приватному ключу. Для каждого сервера задай `admin_ip` — IPv4-адрес control node, которому разрешён SSH-доступ.

Проверь переменные в `group_vars/servers.yml`: `ssh_port`, `ssh_allowed_user`, `firewall_zone`, `firewall_target`.

### Проверка и применение

```bash
ansible servers -m ping
ansible-playbook playbook.yaml --syntax-check
ansible-playbook playbook.yaml --check --diff
ansible-playbook playbook.yaml -b -K
```

После применения повтори запуск. Если конфигурация не изменилась, ожидается `changed=0`.

> **Важно:** playbook меняет SSH и firewall. Перед запуском создай snapshot VM, проверь доступ к консоли и не закрывай текущую административную SSH-сессию, пока не проверишь новое подключение. Не проводи Fail2ban ban-тест с адреса control node.

## Документация

- [Архитектура](docs/architecture.md) — узлы, компоненты и потоки управления.
- [Политика безопасности](docs/security-policy.md) — целевые правила hardening.
- [Модель угроз](docs/threat-model.md) — активы, угрозы и остаточные риски.
- [Аудит](docs/security-audit.md) — фактические результаты и ограничения проверки.
- [План тестирования](docs/test-plan.md) — повторяемые проверки и критерии прохождения.

## Текущий статус

В сохранённых результатах лаборатории подтверждены SSH hardening, SELinux `Enforcing`, основные параметры firewalld, активность Fail2ban jail `sshd` и повторный запуск Ansible с `changed=0`.

Реальный Fail2ban ban/unban test и полноценный penetration test не проводились. Последний сохранённый вывод DNF показывал доступные security updates, поэтому состояние обновлений нужно перепроверить после patching. Подробности и ограничения — в [отчёте аудита](docs/security-audit.md).

## Безопасность репозитория

Не коммить рабочий `inventory.ini`, приватные ключи, пароли, токены или vault password files. Публикуй только обезличенный `inventory.ini.example`; перед публикацией логов проверяй их на IP-адреса, usernames, hostnames и секреты.
