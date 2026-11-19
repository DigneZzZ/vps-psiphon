# Gemini через Psiphon на ноде Xray

Установщик рассчитан на Linux с systemd и локальным Docker Engine. На VPS нужны `curl`, `jq`, `ip`, `ss` и `flock`. Установку запускай от root. Для установки через `git clone` также нужен Git. Настройку маршрута выполняют отдельно в Xray или Remnawave. Для Claude см. [руководство Anthropic](anthropic.md).

## Выбери адрес SOCKS

Начни с проверки Gemini через существующий VPN-профиль. Для сравнения с SOCKS можно измерить доступность страницы напрямую с VPS.

```bash
curl --noproxy '*' -sS --connect-timeout 8 --max-time 25 -I https://gemini.google.com/
```

На хосте или в контейнере с `network_mode: host` используй `127.0.0.1`. Если Xray работает в отдельном Docker network namespace, посмотри его сети и их IPv4-шлюзы. Замени `remnanode` фактическим именем контейнера.

```bash
xray_container=remnanode
sudo docker inspect -f '{{.HostConfig.NetworkMode}}' "$xray_container"
sudo docker inspect -f '{{range $name, $net := .NetworkSettings.Networks}}{{println $name $net.Gateway}}{{end}}' "$xray_container"
sudo ss -ltnp 'sport = :1080'
```

Для bridge выбери сеть, из которой Xray может подключаться к прокси. Именованная сеть Compose обычно имеет свой gateway. При нескольких сетях проверь выбранную из namespace Xray. Если сеть без IPv4, macvlan, overlay или иной режим не даёт достичь адреса, выбери существующий изолированный путь до хоста. SOCKS работает без пароля и должен быть доступен только от нужных клиентов. Привязка к gateway задаёт адрес публикации. При маршрутизации между интерфейсами фильтрация доступа к этому адресу зависит от конфигурации Docker и хоста. Проверь доступ из namespace Xray и с внешней точки. Если к прокси могут подключиться посторонние, ограничь доступ действующим межсетевым экраном до применения маршрута. [Публикация портов Docker](https://docs.docker.com/engine/network/port-publishing/).

## Скачай и установи

Скачай исходник версии `v1.4.0-geolog.1` в новый каталог. Перед запуском проверь команды и выбранный образ. Запиши SHA коммита и digest образа той версии, которую проверил. [Форк vps-psiphon](https://github.com/DigneZzZ/vps-psiphon).

```bash
git clone --branch v1.4.0-geolog.1 --depth 1 https://github.com/DigneZzZ/vps-psiphon.git
cd vps-psiphon
git rev-parse HEAD
bash -n psiphon_install.sh
bash psiphon_install.sh --help
```

Начни со страны VPS, если её предлагает Psiphon. Для Германии укажи `DE`, для Великобритании `GB`, для Франции `FR`, для Нидерландов `NL`. Установщик приводит известные коды `AT AU BE BR CA CH CZ DE DK ES FR GB ID IE IN IT JP NL NO PL RS SE SG US`. Список зависит от серверов, известных клиенту Psiphon. После запуска ищи `AvailableEgressRegions` в `sudo vps-psiphon logs`. Параметр `--region` задаёт запрашиваемую страну выхода, а фактический IP и пригодность для Gemini проверяются после подключения.

Для Xray на хосте или с host network выполни установку с loopback.

```bash
sudo bash ./psiphon_install.sh --region DE --bind-loopback
```

Для Xray в Docker bridge укажи имя его сети.

```bash
xray_network=bridge
sudo bash ./psiphon_install.sh --region DE --bind-docker "$xray_network"
```

Подставь страну своей ноды и имя её bridge-сети. Установщик по умолчанию использует IPv4-шлюз стандартной Docker-сети `bridge`, при его отсутствии выбирает loopback. Для известного сетевого режима задай привязку явно. При новой установке включён только SOCKS. HTTP-прокси доступен по флагу `--http`. Для трафика с лимитом доступен `--bulk-test-mb 0`, который отключает объёмную контрольную загрузку watchdog.

Для уже установленного экземпляра проверь состояние.

```bash
sudo vps-psiphon status
sudo systemctl is-active vps-psiphon.service vps-psiphon-watchdog.timer
```

Перед обновлением сохрани в закрытом каталоге настройки `/etc/default/vps-psiphon`, файлы экземпляра из `/opt/vps-psiphon`, его скрипты и юниты. Зафиксируй ID или digest образа, состояние таймера и серверный профиль. Суффикс `-2` и далее соответствует `--instance N`. Команда ниже обновляет первый экземпляр.

```bash
sudo bash ./psiphon_install.sh --instance 1
```

При повторном запуске установщик сохраняет привязку, регион, порты, состояние HTTP-прокси и автозапуск watchdog, когда их не переопределяют. Если HTTP больше никому не нужен, закрой его через `--no-http`. Смена адреса или порта требует согласованного изменения профиля Xray и прерывает активные соединения через этот SOCKS.

## Проверь SOCKS до изменения Xray

```bash
sudo vps-psiphon status
sudo vps-psiphon check
sudo vps-psiphon outbound
```

`check` выполняет контрольный HTTPS-запрос через SOCKS и отправляет короткий тестовый запрос модели через веб-интерфейс Gemini без входа в аккаунт. Проверка может занять несколько минут. Код выхода `0` означает полученный ответ, `1` — распознанный региональный отказ, `2` — неопределённый результат, `3` — обнаруженную CAPTCHA в режиме имитации браузера, `4` — сбой контрольного HTTPS через SOCKS. Изменения внутреннего формата ответа Google могут повлиять на результат. `outbound` печатает JSON с фактическим адресом, портом и тегом. Используй имя CLI своего экземпляра.

Повтори HTTP-проверку из сети Xray. Для Xray в Docker на Linux можно воспользоваться namespace контейнера, даже если внутри образа отсутствует `curl`.

```bash
socks_endpoint=127.0.0.1:1080
xray_pid=$(sudo docker inspect -f '{{.State.Pid}}' "$xray_container")
sudo nsenter -t "$xray_pid" -n curl -sS --noproxy '' --socks5-hostname "$socks_endpoint" \
  --connect-timeout 8 --max-time 25 -I https://gemini.google.com/
```

Подставь адрес и порт из `outbound`. Для Xray на хосте запусти тот же `curl` без `nsenter`. `--socks5-hostname` оставляет разрешение имени на стороне SOCKS. Код HTTP удостоверяет, что запрос достиг веб-сервера. Для проверки генерации понадобится отдельный запрос модели.

## Добавь маршрут Gemini

В `outbounds` добавь объект из `vps-psiphon outbound`. Оставь первый outbound на месте, поскольку Xray использует его для трафика вне заданных правил. Фрагмент ниже подходит для актуального Xray.

```json
{
  "tag": "psiphon-out",
  "protocol": "socks",
  "settings": {
    "address": "127.0.0.1",
    "port": 1080
  }
}
```

Если сборка Xray поддерживает только старый формат, используй `"settings": {"servers": [{"address": "127.0.0.1", "port": 1080}]}`. В обоих вариантах подставь адрес и порт установленного экземпляра. [Текущая схема SOCKS Xray](https://xtls.github.io/en/config/outbounds/socks.html), [код SOCKS в Xray 1.8.24](https://github.com/XTLS/Xray-core/blob/v1.8.24/infra/conf/socks.go).

В `routing.rules` добавь объект с тем же `outboundTag`.

```json
{
  "type": "field",
  "ruleTag": "GEMINI_PSIPHON",
  "network": "tcp",
  "domain": [
    "domain:gemini.google.com",
    "domain:generativelanguage.googleapis.com",
    "domain:ai.google.dev",
    "domain:aistudio.google.com",
    "domain:alkalimakersuite-pa.clients6.google.com",
    "geosite:google-deepmind"
  ],
  "outboundTag": "psiphon-out"
}
```

`geosite:google-deepmind` добавляет другие сервисы Google AI. Используй эту строку, если такой охват подходит и группа есть в активном `geosite.dat`. Для узкого маршрута или сборки без группы оставь явные домены. При необходимости дополни их по результатам проверки. [Состав группы google-deepmind](https://github.com/v2fly/domain-list-community/blob/master/data/google-deepmind).

Xray применяет первое совпавшее правило. Поставь правило Gemini перед общим DIRECT, сохрани приоритет блокировок и остальных ограничений. Если клиент передаёт IP назначения, проверь распознавание домена на используемом inbound. При необходимости включи `sniffing` с `routeOnly: true` и подходящими `destOverride` в этом inbound. SOCKS Psiphon обслуживает TCP. UDP, включая QUIC браузера, обрабатывается остальными правилами. Если QUIC мешает проверке маршрута, повтори её с клиентом без HTTP/3. При необходимости ограничь только QUIC нужных доменов до общего правила маршрутизации. [Правила маршрутизации Xray](https://xtls.github.io/en/config/routing.html), [пример vps-psiphon](https://github.com/DigneZzZ/vps-psiphon/blob/v1.4.0-geolog.1/xray-outbound.json).

## Примени и проверь с клиента

В Remnawave меняй конфигурацию серверного профиля через панель или её API. Если профиль общий, сначала проверь доступность одного и того же SOCKS-адреса со всех его нод. Для теста собери конфиг с фактическими snippets, геоданными и путями к сертификатам. Команду выполняй тем же бинарником и с теми же ресурсами, что у активной ноды.

```bash
xray run -test -config test-config.json
```

Перед записью перечитай профиль. Обновление может перезапустить Xray на всех связанных нодах, поэтому примени изменение с учётом активных подключений. При ошибке запуска восстанови рабочий профиль.

С действующим клиентским VPN-профилем выполни запрос Gemini и проверь, что модель ответила. Для авторизованных проверок используй аккаунт или API-ключ, предоставленный для этой задачи. В журнале маршрутов проверь выход `psiphon-out`. Обычный HTTPS-запрос должен сохранить прежний маршрут. Для каждой ноды зафиксируй, на каком уровне получен результат.

- `check` и HTTPS из namespace Xray подтверждают серверный маршрут до доступных целей. Результат `inconclusive` оставляет доступность модели неопределённой.
- Ответ модели через VPN подтверждает работу проверенного клиентского маршрута. Для выводов по конкретному аккаунту нужен запрос с этим аккаунтом.

## Если выход не подходит

Сравни запрошенный регион, фактический IP и результат Gemini. Если через SOCKS стабильно воспроизводится отказ, попробуй штатную команду `vps-psiphon rotate`. Она прерывает соединения текущего экземпляра. Если задан пул стран, выход может перейти к следующему региону. Управляемая смена региона выполняется через `vps-psiphon region DE` с нужным кодом. Она пересоздаёт конфиг туннеля из настроек экземпляра. Перед этим сохрани ручные правки. После смены снова проверь IP и работу модели. При повторении ошибки проверь корректность самого теста и доступ через VPN.

## Откат

Восстанови прежний маршрут в панели или Xray. При параллельных изменениях верни только добавленное правило и изменённый outbound. Перед остановкой SOCKS проверь, что у него больше нет потребителей.

Для новой установки используй `sudo vps-psiphon uninstall` или соответствующий CLI экземпляра. Для обновления останови service и timer своего экземпляра, восстанови сохранённые скрипты, юниты, окружение и нужную версию образа. Предыдущую конфигурацию туннеля восстанавливай из закрытого снимка. Выполни `sudo systemctl daemon-reload`, верни исходное состояние service и timer и повтори серверную и клиентскую проверки.
