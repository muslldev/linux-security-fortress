# План тестирования

## Назначение

План содержит повторяемые проверки hardening. Статусы отражают только сохранённые результаты; после изменения конфигурации тесты следует запускать заново.

## Матрица тестов

| ID | Проверка | Метод | Ожидаемый результат | Сохранённый статус |
|---|---|---|---|---|
| T-01 | Ansible connectivity | `ansible servers -m ping` | Узел доступен | Повторить перед новым прогоном |
| T-02 | Playbook syntax | `ansible-playbook playbook.yaml --syntax-check` | Syntax OK | Выполнено успешно |
| T-03 | SSH effective config | `sshd -T` | Порт 2222, root/password off, key auth on, user protected | Подтверждено |
| T-04 | SSH key login | `ssh protected@HOST -p 2222` | Вход по ключу успешен | Подтверждено |
| T-05 | firewalld zone | `firewall-cmd --get-active-zones`, `--list-all` | Ожидаемые зона и правила | Основные параметры подтверждены |
| T-06 | SELinux mode | `getenforce` | `Enforcing` | Подтверждено |
| T-07 | SELinux SSH port | `semanage port -l \| grep ssh` | TCP/2222 → `ssh_port_t` | Подтверждено |
| T-08 | Fail2ban jail | `fail2ban-client status sshd` | Jail активен | Конфигурация/активность подтверждены |
| T-09 | Fail2ban ban/unban | Тестовые неудачные входы с отдельного IP | Тестовый IP блокируется и разблокируется | NOT RUN |
| T-10 | Listener inventory | `ss -tulpen` | Все listeners идентифицированы | INCOMPLETE |
| T-11 | IPv6 firewall review | Проверка IPv6 rules и тест из IPv6-среды | Политика соответствует требованиям | INCOMPLETE |
| T-12 | Idempotence | Повторный playbook | Нет неожиданных изменений | Один прогон `changed=0` |
| T-13 | Security updates | `dnf check-update --security` | Состояние обновлений зафиксировано | Последний сохранённый вывод показывал обновления; повторить после patching |

## Базовый прогон

На control node:

```bash
cd ansible
ansible servers -m ping
ansible-playbook playbook.yaml --syntax-check
ansible-playbook playbook.yaml --check --diff
```

После snapshot и проверки доступа:

```bash
ansible-playbook playbook.yaml -b -K
ansible-playbook playbook.yaml -b -K
```

На managed node:

```bash
sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers) '
firewall-cmd --get-active-zones
firewall-cmd --zone=public --list-all
fail2ban-client status sshd
getenforce
semanage port -l | grep ssh
ss -tulpen
dnf check-update --security
dnf updateinfo list --security
```

## Безопасность выполнения

- До изменений SSH/firewall создай snapshot VM.
- Убедись, что доступна консоль VM.
- Не закрывай текущую административную сессию до проверки нового SSH-входа.
- Не проводи Fail2ban ban-тест с IP управляющего узла `admin_ip`.
- Не очищай nftables ruleset и не отключай firewalld.
- Не меняй SSH/firewall удалённо без проверенного способа восстановления.

## Критерии завершения

Тест считается пройденным только после получения фактического результата и его записи. Состояние сервиса не заменяет функциональный тест. Не выполненные проверки должны оставаться `NOT RUN` или `INCOMPLETE`, а не обозначаться как успешные.
