# Effective Mobile DevOps Тестовое задание:
Написать скрипт на bash для мониторинга процесса test в среде linux. Скрипт должен отвечать следующим требованиям:

Запускаться при запуске системы (предпочтительно написать юнит systemd в дополнение к скрипту).
2. Отрабатывать каждую минуту.
3. Если процесс запущен, то стучаться (по https) на https://test.com/monitoring/test/api.
4. Если процесс был перезапущен, писать в лог /var/log/monitoring.log (если процесс не запущен, то ничего не делать) 
5. Если сервер мониторинга не доступен, так же писать в лог.

# Решение:

# Шаг 1: Bash-скрипт process-monitor.sh
Создаем файл и задаем ему права:
```
sudo nano /usr/local/bin/process-monitor.sh
sudo chmod +x /usr/local/bin/process-monitor.sh

```
# Содержимое скрипта:
```
#!/bin/bash

PROCESS_NAME="test"
MONITOR_URL="https://test.com/monitoring/test/api"
LOG_FILE="/var/log/monitoring.log"
PID_FILE="/var/run/${PROCESS_NAME}.pid"

# Проверка: работает ли процесс
PID=$(pidof "$PROCESS_NAME")

if [[ -n "$PID" ]]; then
    # Проверка перезапуска: если PID изменился
    if [[ -f "$PID_FILE" ]]; then
        OLD_PID=$(cat "$PID_FILE")
        if [[ "$OLD_PID" != "$PID" ]]; then
            echo "$(date) - Process $PROCESS_NAME restarted (old PID: $OLD_PID, new PID: $PID)" >> "$LOG_FILE"
        fi
    fi
    echo "$PID" > "$PID_FILE"

    # Попытка стучаться на мониторинг-сервер
    curl -s --connect-timeout 5 "$MONITOR_URL" > /dev/null
    if [[ $? -ne 0 ]]; then
        echo "$(date) - Monitoring server unreachable: $MONITOR_URL" >> "$LOG_FILE"
    fi
else
    # Процесс не запущен — ничего не делаем
    exit 0
fi

```
# Шаг 2: Создание systemd юнита
Создаем юнит-файл:
```
sudo nano /etc/systemd/system/process-monitor.service
```
Содержимое юнита:
```
[Unit]
Description=Monitor process test
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/process-monitor.sh

[Install]
WantedBy=multi-user.target

```
# Шаг 3: Настройка таймера systemd
Создаем таймер, чтобы запускать скрипт каждую минуту:
```
sudo nano /etc/systemd/system/process-monitor.timer
```

Содержимое:
```
[Unit]
Description=Run process-monitor every minute

[Timer]
OnBootSec=1min
OnUnitActiveSec=1min
Unit=process-monitor.service

[Install]
WantedBy=timers.target
```
# Шаг 4: Включение и запуск
```
sudo systemctl daemon-reexec
sudo systemctl daemon-reload

sudo systemctl enable process-monitor.timer
sudo systemctl start process-monitor.timer
```

Проверка:
```
systemctl status process-monitor.timer
journalctl -u process-monitor.service
```
# Подсмотрено решение:
(https://gitlab.vadimevgrafov.ru/learning/learning-projects/-/blob/main/test-task/test_bash.md)
# Протестированно:
![test](https://github.com/netgitups/test/blob/main/img/test.png)