# Стенд и его настройка

## Компоненты

- Wazuh 4.14.9 all-in-one на Ubuntu Server: Manager, Indexer (OpenSearch), Dashboard, Filebeat.
- Агент на Windows 10 (`Win10-Host`, ID 001). В списке агентов у него адрес `127.0.0.1`, так как подключение идёт через проброс портов NAT.
- Dashboard открывается в браузере хоста по `https://localhost:8443`.
- VirtualBox на Windows 10 работает поверх включённого гипервизора Windows (режим NEM), поэтому ВМ работает медленнее, чем на «чистом» VT-x.

## Проверка стенда

1. Агент `Win10-Host` появляется в Endpoints в статусе `pending`, затем `active` ([pending](screenshots/01-agent-pending.png), [active](screenshots/02-agent-active.png)).
2. В Discover (индекс `wazuh-alerts-*`) приходят свежие события ([скриншот](screenshots/03-events-arriving.png)).
3. Dashboard запущен как служба `wazuh-dashboard` на порту 443, первый запуск 19.10.2026 ([терминал и страница входа](screenshots/04-first-login-terminal.png)).

## Проблемы при установке и эксплуатации

| Симптом | Причина | Что сделал |
|---|---|---|
| `wazuh-manager` не запускался: таймаут systemd | ВМ медленная, запуск не укладывался в стандартный таймаут | Увеличил `TimeoutStartSec` через override юнита (до 300 с) |
| Менеджер не мог подключиться к Indexer | В keystore менеджера не было учётных данных Indexer | Добавил их в keystore (`wazuh-keystore`) |
| Dashboard: «Something went wrong», `timeout of 20000ms exceeded` | В ядре ВМ срабатывал `watchdog: BUG: soft lockup` (потоки `iou-sqp`, `G1 Refine`): диск хоста не успевал, API менеджера отвечало долго | Рассматривал: включить Host I/O Cache и 4 vCPU в VirtualBox, отключить `io_uring` и зафиксировать heap OpenSearch (`-Xms/-Xmx 1536m`), поднять `timeout` для API и `opensearch.requestTimeout` до 90000, поднять `kernel.watchdog_thresh` |

## Что вынес из установки

- Читать `journalctl -u <служба>` и логи `/var/ossec/logs/ossec.log`, `/var/log/wazuh-indexer/`, а не гадать.
- Ошибки Dashboard часто вторичны: настоящая причина в Manager, Indexer или нехватке ресурсов ВМ.
- Для проверки цепочки доставки алертов: `alerts.json` на сервере → Filebeat → Indexer → Dashboard. Если алерт есть в `alerts.json`, но нет в Dashboard, сбой между менеджером и панелью.
