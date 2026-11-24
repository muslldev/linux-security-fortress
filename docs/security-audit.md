# Отчёт об аудите

## Назначение и оговорка

Здесь фиксируются фактические результаты сохранённых проверок. Целевые требования описаны в [security-policy.md](security-policy.md).

Результаты относятся к моменту выполнения команд. Перед новым релизом аудита проверки нужно повторить и указать дату.

## Сводка

| Область | Зафиксированный результат |
|---|---|
| ОС | Rocky Linux 9.9 (Blue Onyx) |
| Ядро в одном из прогонов | `5.14.0-741.24.1.el9_9.x86_64` |
| SSH port | `2222` |
| Root login | Запрещён |
| Password authentication | Отключена |
| Public key authentication | Включена |
| Разрешённый SSH-пользователь | `protected` |
| firewalld zone / target | `public` / `DROP` |
| SSH allow rule | IPv4 control node → TCP/2222 |
| SELinux | `Enforcing` |
| SELinux SSH port type | `ssh_port_t` для TCP/2222 |
| Fail2ban | jail `sshd` активен |
| Ansible повторный запуск | `ok=31 changed=0 unreachable=0 failed=0` в сохранённом прогоне |

## SSH

Сохранённый вывод `sshd -T`:

```text
port 2222
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
allowusers protected
```

Подключение пользователем `protected` по ключу на TCP/2222 работало.

## firewalld

Зафиксированы зона `public`, target `DROP` и правило, разрешающее TCP/2222 с IPv4-адреса control node. В списке сервисов зоны присутствовали `cockpit` и `dhcpv6-client`.

Наличие `cockpit` в списке разрешённых сервисов не доказывает, что процесс слушает порт. В сохранённом выводе также присутствовал `forward: yes`; назначение требует отдельной оценки.

## Fail2ban

Сохранённые параметры jail `sshd`: backend `systemd`, port `2222`, `maxretry=5`, `findtime=10m`, `bantime=1h`, action `firewallcmd-rich-rules`.

В одном из статусов: failed `0`, banned `0`. `fail2ban-regex systemd-journal sshd` показал 4 lines, 1 ignored, 0 matched, 3 missed.

**Реальный ban/unban test не выполнялся.** Активность jail не равнозначна доказательству успешной блокировки источника.

## SELinux и Ansible

SELinux был в режиме `Enforcing`; для TCP/2222 назначен `ssh_port_t`. Syntax check playbook проходил успешно. Один повторный запуск завершился `changed=0`; это подтверждает отсутствие изменений в конкретном прогоне, но не все возможные сценарии.

## Обновления ОС: требуется повторная проверка

В сохранённом более позднем выводе от 22 ноября 2026 команда `dnf check-update --security` вернула `rc=100` и вывела доступные пакеты. Для `dnf check-update` код `100` означает, что обновления доступны. В другом выводе `dnf updateinfo list --security` также присутствовали security advisories.

Поэтому прежнее утверждение об отсутствии доступных security updates устарело. После установки обновлений следует повторить проверку и сохранить вывод с датой.

## Не выполнено или не подтверждено

- end-to-end Fail2ban ban/unban test;
- полноценный penetration test;
- полный аудит TCP/UDP listeners;
- завершённая оценка IPv6 firewall policy;
- оценка необходимости Cockpit;
- оценка `forward: yes`.

## Команды повторного аудита

```bash
sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers) '
firewall-cmd --get-active-zones
firewall-cmd --zone=public --list-all
fail2ban-client status sshd
getenforce
semanage port -l | grep ssh
ss -tulpen
cat /etc/rocky-release
uname -r
dnf check-update --security
dnf updateinfo list --security
```
