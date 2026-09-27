# Наши изменения относительно upstream 3x-ui-pro

Актуальная рабочая ветка: `agent/restore-auto-domain`

Эта ветка сохраняет поведение старой установки, которым мы пользовались раньше, но адаптирует его под современную 3x-ui.

## Что восстановлено

### 1. Автоматические домены

Возвращён параметр:

```bash
-auto_domain y
```

При его использовании домены формируются автоматически из публичного IPv4 сервера:

```text
Панель / подписка: <IPv4>.cdn-one.org
REALITY:           <IPv4-с-дефисами>.cdn-one.org
```

Пример:

```text
13.140.11.224.cdn-one.org
13-140-11-224.cdn-one.org
```

Установка из тестовой ветки:

```bash
wget -qO x-ui-latest.sh \
https://raw.githubusercontent.com/lnquisitorS/3x-ui-pro/agent/restore-auto-domain/x-ui-latest.sh

bash x-ui-latest.sh -install y -auto_domain y
```

## 2. Исправление 404 у ссылки подписки

На современной 3x-ui при первом запуске база сама создаёт случайные значения:

```text
subPath
subJsonPath
subClashPath
```

Старый установщик затем добавлял свои `subPath` и `subJsonPath` через отдельные INSERT.
Из-за этого в таблице `settings` появлялись дубли.

Реально воспроизведённый пример:

```text
subPath|/cj23tg9m5mff0fbh/
subPath|/pEz4DYNlJf/

subJsonPath|/afs1px4pu0ki1e7r/
subJsonPath|/R7gSJmtdz8/
```

При этом nginx использовал один путь, а subscription server 3x-ui мог зарегистрировать другой.
Результат в браузере:

```text
404 page not found
```

В нашей ветке перед записью installer-managed значений seed-записи `subPath` и
`subJsonPath` удаляются, после чего создаётся ровно по одной записи.

После ручного исправления на тестовом сервере подписка сразу начала открываться.

Проверка дублей:

```bash
sqlite3 /etc/x-ui/x-ui.db \
"SELECT key,COUNT(*) FROM settings GROUP BY key HAVING COUNT(*) > 1 ORDER BY key;"
```

На корректной установке команда должна вернуть пустой результат.

## 3. Автоматический клиент `first`

Старая установка, которой мы пользовались, после развёртывания сразу имела клиента:

```text
Email:  first
Sub ID: first
```

В актуальной версии установщика такого клиента больше не было.

В нашей ветке автоматическое создание `first` возвращено.

Клиент создаётся через официальный Panel API самой установленной 3x-ui, а не прямыми
INSERT в новые first-class таблицы клиентов. Это важно, потому что современная 3x-ui
сама корректно создаёт:

- запись в `clients`;
- привязки в `client_inbounds`;
- traffic/stat record;
- UUID для VLESS;
- password для Trojan;
- protocol-specific defaults.

Клиент `first` автоматически привязывается ко всем четырём создаваемым установщиком inbound:

```text
VLESS + REALITY
VLESS + WebSocket
VLESS + XHTTP
Trojan + gRPC
```

После установки ожидаемая ссылка подписки:

```text
https://<domain>/<subPath>/first
```

Установщик также выводит эту ссылку в финальном отчёте.

## 4. hosts.group_id

Сохранена актуальная логика `group_id` для автоматически создаваемых hosts.

На современных версиях 3x-ui каждому host назначается отдельный 16-символьный
`group_id`, благодаря чему host можно редактировать и удалять через интерфейс панели.

Для старых версий 3x-ui, где колонки `hosts.group_id` ещё нет, используется проверка схемы.

## Что уже проверено

Проверено на чистом Ubuntu 26.04:

- установка с `-auto_domain y` проходит;
- автоматические домены работают;
- nginx конфигурация валидна;
- панель открывается;
- причина `404 page not found` по подписке воспроизведена;
- дубли `subPath/subJsonPath` подтверждены;
- после удаления дублей подписка открылась в браузере.

## Что осталось проверить перед merge в main

Последняя правка — автоматическое создание клиента `first` через Panel API —
должна быть проверена ещё одной чистой установкой.

После установки проверить:

```bash
sqlite3 /etc/x-ui/x-ui.db "
SELECT id,email,sub_id,uuid,password,enable
FROM clients;

SELECT ci.client_id,ci.inbound_id,i.remark,i.protocol
FROM client_inbounds ci
JOIN inbounds i ON i.id=ci.inbound_id
ORDER BY ci.client_id,ci.inbound_id;

SELECT key,value
FROM settings
WHERE key IN ('subPath','subJsonPath','subURI')
ORDER BY key;
"
```

Ожидается:

- один клиент `first`;
- `sub_id = first`;
- UUID заполнен;
- Trojan password заполнен;
- четыре записи в `client_inbounds`;
- по одному `subPath` и `subJsonPath`;
- `https://<domain>/<subPath>/first` открывается без ручного создания клиента.

## Статус

PR #1 остаётся draft до завершения последней чистой проверки.

Не сливать в `main` вслепую: сначала подтвердить автоматическое создание `first`
на новой установке.
