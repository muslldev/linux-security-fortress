# Linux Security Fortress

**Автоматизация базового hardening Linux-сервера с помощью Ansible**

Linux Security Fortress — учебный инфраструктурный проект по защите Linux-сервера. Он демонстрирует настройку SSH, firewalld, Fail2ban и SELinux, автоматизацию конфигурации и проверки критичных параметров с помощью Ansible.

> Проект предназначен для лабораторной среды и демонстрации практических навыков. Перед использованием в production необходимо адаптировать политику безопасности под конкретную инфраструктуру и провести полноценное тестирование.

## Содержание

- [Цели проекта](#цели-проекта)
- [Архитектура](#архитектура)
- [Реализованные меры защиты](#реализованные-меры-защиты)
- [Структура репозитория](#структура-репозитория)
- [Требования](#требования)
- [Подготовка и запуск](#подготовка-и-запуск)
- [Проверка результата](#проверка-результата)
- [Результаты аудита](#результаты-аудита)
- [Ограничения и развитие](#ограничения-и-развитие)

## Цели проекта

- Уменьшить поверхность атаки Linux-сервера.
- Перейти от ручной настройки к воспроизводимой конфигурации Ansible.
- Ограничить SSH-доступ и отключить парольную аутентификацию.
- Настроить host-based firewall и защиту SSH от перебора.
- Использовать SELinux в режиме Enforcing.
- Автоматизировать проверки конфигурации.
- Документировать архитектуру, политику безопасности и результаты аудита.

## Архитектура

Лаборатория состоит из двух машин:

| Узел | Роль | ОС |
|---|---|---|
| Control node | Управление конфигурацией, запуск Ansible | Ubuntu |
| Managed node | Защищаемый сервер | Rocky Linux 9.9 |

Управление выполняется с Ubuntu Host на Rocky Linux по SSH. Ansible подключается под отдельной учётной записью `protected`, используя SSH-ключ и порт `2222`. Для привилегированных задач используется become.

```text
┌──────────────────────────┐       SSH / TCP 2222
│       Ubuntu Host        │ ─────────────────────────┐
│   Ansible Control Node   │                          │
└──────────────────────────┘                          ▼
                                            ┌──────────────────────┐
                                            │   Rocky Linux 9.9    │
                                            │   Managed Server    │
                                            │                      │
                                            │  sshd :2222          │
                                            │  firewalld           │
                                            │  Fail2ban            │
                                            │  SELinux             │
                                            └──────────────────────┘
```

В лабораторной конфигурации правило firewalld разрешает SSH TCP/2222 только с IPv4-адреса управляющего узла. При переносе проекта в другую сеть этот адрес необходимо заменить в переменных.

## Реализованные меры защиты

### SSH hardening

Ansible управляет файлом `/etc/ssh/sshd_config.d/99-security-hardening.conf`.

| Параметр | Значение | Назначение |
|---|---|---|
| `Port` | `2222` | Выделенный SSH-порт |
| `PermitRootLogin` | `no` | Запрет прямого входа root |
| `PasswordAuthentication` | `no` | Отключение парольной аутентификации |
| `PubkeyAuthentication` | `yes` | Аутентификация по SSH-ключу |
| `AllowUsers` | `protected` | Ограничение списка пользователей |

Перед применением выполняется `sshd -t`. Для SELinux-порта используется модуль `community.general.seport` с типом `ssh_port_t`.

### Firewall

Используется `firewalld`:

- активная зона — `public`;
- target зоны — `DROP`;
- SSH разрешён по TCP/2222 только с разрешённого IPv4-адреса;
- в зоне также присутствуют сервисы `cockpit` и `dhcpv6-client`.

Наличие сервиса в конфигурации зоны само по себе не доказывает, что соответствующий процесс слушает порт. Фактические сокеты проверяются отдельно.

### Fail2ban

Настроен jail `sshd`:

| Параметр | Значение |
|---|---|
| Порт | `2222` |
| Backend | `systemd` |
| `maxretry` | `5` |
| `findtime` | `10m` |
| `bantime` | `1h` |
| Действие | `firewallcmd-rich-rules` |

Fail2ban анализирует события SSH из systemd journal и использует firewalld rich rules для блокировки адресов, превысивших порог неудачных попыток.

**Ограничение проверки:** подтверждены активность jail и его конфигурация. Намеренный тест реальной блокировки с адреса управляющего узла не проводился, чтобы не потерять административный доступ.

### SELinux

SELinux работает в режиме `Enforcing`. Для SSH-порта `2222/tcp` настроен тип `ssh_port_t`.

### Ansible automation

Конфигурация разделена на роли:

- `ssh_hardening` — настройка и проверка SSH;
- `selinux` - установка и разрешение SSH-порта
- `firewall` — правила firewalld и политика зоны;
- `fail2ban` — установка и настройка Fail2ban;
- `security_tests` — автоматические проверки критичных параметров.

Используются `group_vars`, Jinja2 template, handlers и Ansible modules. Повторный запуск playbook после применения конфигурации завершился с `changed=0`, что подтверждает идемпотентность текущего набора задач в проверенной лаборатории.

## Структура репозитория

Целевая структура публикации:

```text
linux-security-fortress/
├── README.md
├── LICENSE
├── .gitignore
├── docs/
│   ├── architecture.md
│   ├── security-policy.md
│   ├── threat-model.md
│   ├── security-audit.md
│   └── test-plan.md
├── screenshots/
└── ansible/
    ├── ansible.cfg
    ├── inventory.ini.example
    ├── playbook.yaml
    ├── group_vars/
    │   └── servers.yml
    └── roles/
        ├── ssh_hardening/
        ├── firewall/
        ├── fail2ban/
        ├── selinux/
        └── security_tests/
```

## Требования

На управляющем узле:

- Ubuntu с установленным Ansible;
- SSH-клиент и сетевой доступ к Rocky Linux;
- приватный SSH-ключ, соответствующий публичному ключу пользователя `protected` на сервере.

На managed node:

- Rocky Linux 9.x;
- доступ по SSH;
- пользователь `protected` с правами sudo;
- работающий SSH daemon.

Необходимые Ansible collections:

```bash
ansible-galaxy collection install ansible.posix community.general
```

## Подготовка и запуск

1. Склонируйте репозиторий:

   ```bash
   git clone <URL-РЕПОЗИТОРИЯ>
   cd linux-security-fortress/ansible
   ```

2. Создайте локальный inventory:

   ```bash
   cp inventory.ini.example inventory.ini
   ```

3. Укажите адрес сервера, пользователя, порт и путь к приватному ключу. Пример:

   ```ini
   [servers]
   protected_server ansible_host=<IP-АДРЕС-СЕРВЕРА> ansible_user=protected ansible_port=2222 ansible_ssh_private_key_file=<ПУТЬ-К-ПРИВАТНОМУ-КЛЮЧУ> admin_ip=<IP-АДРЕС-АДМИНИСТРАТОРА>
   ```

4. Проверьте переменные в `group_vars/servers.yml`: `ssh_port`, `ssh_allowed_user`, `firewall_zone` и `firewall_target`. IP управляющего узла (`admin_ip`) задаётся отдельно для каждого сервера в `inventory.ini`.

5. Проверьте доступность узла:

   ```bash
   ansible servers -m ping
   ```

6. Проверьте синтаксис playbook:

   ```bash
   ansible-playbook playbook.yaml --syntax-check
   ```

7. Запустите конфигурацию:

   ```bash
   ansible-playbook playbook.yaml -K
   ```

`-K` запрашивает become password.

> **Предупреждение:** изменения SSH и firewall могут привести к потере удалённого доступа. Перед запуском убедитесь, что доступна консоль виртуальной машины или другой способ восстановления. Не закрывайте действующую административную сессию, пока не проверили новое SSH-подключение.

Повторный запуск playbook позволяет проверить идемпотентность. Если конфигурация не изменилась, ожидается `changed=0`.

## Проверка результата

Команды для ручной проверки на managed node:

```bash
# Итоговая конфигурация SSH
sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers) '

# Активные зоны firewalld
firewall-cmd --get-active-zones

# Правила зоны
firewall-cmd --zone=public --list-all

# Fail2ban jail
fail2ban-client status sshd

# SELinux
getenforce

# Слушающие TCP- и UDP-сокеты
ss -tulpen

# Версия ОС и запущенное ядро
cat /etc/rocky-release
uname -r
```

Роль `security_tests` также проверяет параметры SSH, режим SELinux, SELinux-порт SSH, target firewalld, состояние jail Fail2ban, ожидаемый TCP listener и права на SSH drop-in файл.

## Результаты аудита

Последняя зафиксированная проверка лаборатории показала:

| Проверка | Результат |
|---|---|
| ОС | Rocky Linux 9.9 (Blue Onyx) |
| Запущенное ядро | `5.14.0-741.24.1.el9_9.x86_64` |
| SSH port | `2222` |
| Root login | Запрещён |
| Password authentication | Отключена |
| Public key authentication | Включена |
| Разрешённый SSH-пользователь | `protected` |
| Firewalld zone | `public` |
| Firewalld target | `DROP` |
| SSH allow rule | IPv4 управляющего узла → TCP/2222 |
| Fail2ban jail | `sshd`, активен |
| SELinux | `Enforcing` |
| Ansible recap | `ok=31 changed=0 unreachable=0 failed=0` |
| Security update check | После обновления `dnf check-update --security` вернул `rc=0` |

`rc=0` у `dnf check-update --security` означает, что на момент проверки команда не обнаружила доступных security updates в подключённых репозиториях. Это не доказывает отсутствие всех уязвимостей.

## Ограничения и дальнейшее развитие

Текущая версия — лабораторный проект базового hardening, а не сертифицированный security baseline и не полноценный penetration test.

Известные границы проверки:

- реальный Fail2ban ban/unban test не выполнялся;
- полноценное сканирование поверхности атаки и penetration testing не проводились;
- сервис Cockpit присутствует в разрешённых сервисах firewalld; необходимость его использования требует отдельного решения;
- `forward: yes` в зоне `public` зафиксирован и требует отдельной оценки с учётом сетевой роли сервера;
- результат проверки security updates зависит от актуальности и доступности подключённых DNF-репозиториев.

Возможные улучшения:

- оформить threat model и test plan;
- добавить безопасные интеграционные тесты SSH и firewall;
- принять решение по Cockpit и IP forwarding;
- расширить аудит слушающих TCP/UDP-портов и системных служб;
- улучшить обработку результатов DNF security update check;
- добавить CI-проверки Ansible syntax, lint и YAML;
- подготовить обезличенные скриншоты.

---

## Об авторе проекта

Проект создан как практическая работа по Linux administration, Ansible automation и базовому Linux security hardening. Основной акцент — воспроизводимость конфигурации, минимизация доступных способов удалённого входа и проверяемость результата.
