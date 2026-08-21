# RB5009 + WireGuard: разбор настройки и диагностики

**Устройство:** MikroTik RB5009UG+S+IN
**Провайдер:** A1 (Беларусь), динамический публичный IP
**Дата разбора:** 21 августа 2026

---

## Исходное состояние конфигурации

| Параметр | Значение |
|---|---|
| WG-интерфейс | `wg-srv`, порт `51999`, MTU 1420 |
| Адрес в туннеле | `10.9.8.1/24` |
| Пиры | `phone` → `10.9.8.2/32`, `pc` → `10.9.8.3/32` |
| WAN | `vlan-A1` (VLAN 1 на `bridgeLocal`), публичный `93.191.102.200/22` по DHCP |
| DDNS | `hkb0aqf04v4.sn.mynetname.net` (IP Cloud) |
| VLAN | 1 — WAN, 10 — `vlan-mgmt` (`192.168.10.1`), 20 — `vlan-clients` (`192.168.20.1`) |
| Бридж | `bridgeLocal`, `vlan-filtering=yes`, `ingress-filtering=yes`, `pvid=1` |

---

## Часть 1. Почему WireGuard не подключался

### В: `/interface/wireguard/print detail` не показывает `private-key`. Ключ потерян?

**О:** Нет. Публичный ключ математически выводится из приватного — если RouterOS показывает `public-key`, то приватный существует. Новые версии просто не печатают его в `print detail`.

Проверить явно:

```
/interface/wireguard/get [find name="wg-srv"] private-key
```

Если действительно пусто — сгенерировать новую пару:

```
/interface/wireguard/set [find name="wg-srv"] private-key=""
```

⚠️ При этом меняется и публичный ключ — придётся переписать его во всех клиентах.

---

### В: Флаг `R` (RUNNING) у WG-интерфейса означает, что туннель работает?

**О:** Нет. WireGuard-интерфейс в RouterOS получает статус «running» сразу после создания — без пиров и без трафика. Работоспособность видна только по `last-handshake`:

```
/interface/wireguard/peers/print stats
```

---

### В: Как читать счётчики пиров при диагностике?

| Что видно | Что значит |
|---|---|
| `last-handshake` идёт, tx/rx растут | туннель работает; проблема в маршрутах/NAT/DNS |
| tx > 0, rx = 0 | пакеты уходят, ответа нет → неверный endpoint/порт, firewall, CGNAT |
| tx = 0, rx = 0 | клиент вообще не отправляет — ошибка в конфиге клиента |
| rx растёт, handshake нет | не совпадают ключи или preshared-key |

---

### В: Настоящая причина неработающего туннеля?

**О:** Пустое поле `client-endpoint` у обоих пиров.

```
0  name="phone" ... client-endpoint="" client-allowed-address=::/0
1  name="pc"    ... client-endpoint="" client-allowed-address=""
```

Поля `client-*` нужны роутеру для генерации конфига и QR-кода клиента. С пустым `client-endpoint` в сгенерированном конфиге отсутствует строка `Endpoint=` — клиенту физически некуда стучаться. Отсюда `rx=0, tx=0` у обоих пиров.

Вторая ошибка: у `phone` стояло `client-allowed-address=::/0` — только IPv6. Весь IPv4-трафик пошёл бы мимо туннеля.

**Исправление:**

```
/interface/wireguard/peers/set [find name="phone"] \
    client-endpoint="hkb0aqf04v4.sn.mynetname.net:51999" \
    client-allowed-address=0.0.0.0/0,::/0

/interface/wireguard/peers/set [find name="pc"] \
    client-address=10.9.8.3/32 client-dns=1.1.1.1 \
    client-endpoint="hkb0aqf04v4.sn.mynetname.net:51999" \
    client-allowed-address=0.0.0.0/0
```

После этого в WinBox/WebFig в свойствах пира появятся готовый конфиг и QR-код.

---

### В: Что должно быть в `allowed-address` на стороне роутера?

**О:** Только адрес клиента в туннеле — `10.9.8.2/32`. `0.0.0.0/0` ставится **только в конфиге клиента**. Один ключ = один клиент, два устройства с одним ключом работать не будут.

---

### В: Обязателен ли `persistent-keepalive`?

**О:** Да, если клиент за NAT. Значение `25s` — стандартное. У обоих пиров уже выставлено.

---

## Часть 2. Динамический IP провайдера

### В: Как не менять конфиги клиентов при смене IP провайдером?

**О:** MikroTik IP Cloud DDNS:

```
/ip/cloud/set ddns-enabled=yes
/ip/cloud/print
```

Получаем имя `hkb0aqf04v4.sn.mynetname.net`. В клиенте вместо IP указываем `hkb0aqf04v4.sn.mynetname.net:51999`. Роутер сам обновляет запись при смене адреса.

**Нюанс:** клиенты WireGuard разрешают DNS-имя при подключении. Мобильные приложения и официальный клиент Windows перерезолвят его при неудачных хендшейках, а `wg-quick` на Linux — только при старте. После смены IP достаточно переподключиться.

Для надёжности можно включить принудительный интервал обновления:

```
/ip/cloud/set ddns-update-interval=10m
```

---

### В: Как проверить, что я не за CGNAT?

**О:** Сравнить вывод двух команд:

```
/ip/cloud/print
/ip/address/print where interface=<WAN>
```

Признаки CGNAT: адрес на WAN из `100.64.0.0/10`, либо предупреждение «DDNS server received request from IP X which does not match IP Y», либо `public-address` не совпадает с адресом на интерфейсе.

В нашем случае `93.191.102.200` висит прямо на `vlan-A1`, `status: updated` — CGNAT нет, входящие подключения возможны, проброс портов не нужен.

---

### В: Как настроить site-to-site, если у второй стороны тоже динамический адрес?

**О:** Сторона с динамическим адресом инициирует соединение и в `endpoint` не нуждается вовсе. Достаточно, чтобы адрес был известен у одной стороны.

Если нужно указать DNS-имя в `endpoint-address` пира, RouterOS резолвит его периодически, но исторически с этим бывают заминки. Надёжнее скрипт:

```
/system/script/add name=wg-reresolve source={
  :local h "hkb0aqf04v4.sn.mynetname.net"
  :local new [:resolve $h]
  :local cur [/interface/wireguard/peers/get [find comment="hq"] endpoint-address]
  :if ($new != $cur) do={
    /interface/wireguard/peers/set [find comment="hq"] endpoint-address=$new
    :log info "WG endpoint updated to $new"
  }
}
/system/scheduler/add name=wg-reresolve interval=2m on-event=wg-reresolve
```

---

## Часть 3. Потеря доступа к роутеру — разбор ошибки

### В: Выполнил `/ip/service/set www disabled=yes` и потерял браузерный доступ. Что делать?

**О:** `www` — это и есть WebFig на порту 80. Отключение сервиса убирает браузерный доступ целиком.

**Правильный подход:** не отключать сервис, а ограничить по адресу:

```
/ip/service/set www disabled=no address=192.168.10.0/24,192.168.20.0/24,10.9.8.0/24
```

Отключать целиком (`disabled=yes`) стоит только то, что не используется: `telnet`, `ftp`, `api`, `api-ssl`.

---

### В: Чем отличаются ошибки Winbox и что каждая означает?

| Ошибка | Что произошло |
|---|---|
| `syn timeout` / зависание | пакет отброшен (`action=drop` в firewall) или не дошёл |
| `The remote host closed the connection` | TCP установился, приложение закрыло сессию — ограничение `/ip/service address=`, несовместимость версий, либо перехватчик на клиенте |
| `Connection refused` | сервис выключен, порт не слушает |
| `MacConnection syn timeout` | MAC-Winbox запрещён на интерфейсе (`allowed-interface-list`) или нет L2-связности |

---

### В: `nc -vz` показывает `succeeded` на любом порту, даже несуществующем. Почему?

**О:** Корпоративный агент или VPN-клиент на машине перехватывает исходящие TCP-соединения и отвечает SYN/ACK за удалённый хост. Реальные пакеты до роутера не доходят.

**Контрольный тест:**

```
nc -vz 192.0.2.1 9999
```

`192.0.2.1` — зарезервированный адрес из RFC 5737, его не существует. Если и он `succeeded` — перехват локальный, 100%.

Пока агент активен, любая сетевая диагностика врёт: невозможно отличить «дропнуто firewall» от «сервис закрыл соединение». Выключить на время настройки.

---

### В: Winbox видит роутер в Neighbors, но подключиться не может. Почему?

**О:** Neighbors работает через MNDP — широковещательный протокол на L2, показывающий **все** IP-адреса роутера независимо от того, в какой из его сетей находится клиент. Видимость в Neighbors не означает наличия IP-маршрута до конкретного адреса.

Проверить реальную L2-достижимость:

```
arp -n 192.168.10.1
```

`no entry` = сеть не on-link, ARP-запрос не отправлялся.

---

### В: Итоговая причина потери доступа в нашем случае?

**О:** На USB-Ethernet адаптере MacBook (`en7`) стоял адрес `192.168.10.22` с маской **`/32`** вместо `/24`.

```
en7: 192.168.10.22 netmask 0xffffffff
192.168.10.22/32   link#25   UC   en7      # только host-route на себя
```

Маршрута в сеть `192.168.10.0/24` не существовало. Пакеты на `192.168.10.1` уходили через default → `en0` (`192.168.20.41`) → роутер маршрутизировал их → source-адрес становился `192.168.20.41` → правило firewall «Drop Winbox access from others» их отбрасывало.

**Исправление:**

```
sudo ifconfig en7 192.168.10.22 netmask 255.255.255.0
```

Постоянно: *System Settings → Network → USB-адаптер → Details → TCP/IP → Configure IPv4: Manually*, маска `255.255.255.0`.

---

### В: Какие аварийные входы имеет смысл держать?

| Способ | Надёжность | Комментарий |
|---|---|---|
| Консоль micro-USB | максимальная | не зависит от VLAN, firewall, сервисов и сети клиента |
| Winbox через WireGuard (`10.9.8.1`) | высокая | второй независимый путь, но требует рабочего туннеля |
| Winbox из mgmt-VLAN | средняя | зависит от корректной настройки клиента |
| MAC-Winbox | не рекомендуется | см. ниже |

Консоль:

```
ls /dev/tty.usb*
screen /dev/tty.usbmodem1101 115200
```

Выход из `screen`: `Ctrl+A`, затем `K`.

---

### В: Почему MAC-Winbox не подходит как аварийный вход в этой конфигурации?

**О:** При `vlan-filtering=yes` единственный интерфейс, на котором его можно разрешить, — `bridgeLocal`, а в него входит и VLAN 1 с сегментом провайдера. Разрешив MAC-Winbox, вы открываете вход по MAC для L2-сегмента A1, минуя весь IP-firewall. Ограничить его по VLAN нельзя — он работает на уровне интерфейса.

```
/tool/mac-server/set allowed-interface-list=none
/tool/mac-server/mac-winbox/set allowed-interface-list=none
/tool/mac-server/ping/set enabled=no
```

Роль аварийного входа берёт на себя консоль micro-USB. Плюс — не нужно выводить порт из бриджа и терять hw-offload.

---

## Часть 4. Аудит firewall

### В: Что было не так в цепочках?

**Ключевая проблема — подход «чёрный список».** В обеих цепочках отсутствует финальное `drop`, то есть политика по умолчанию `accept`. Всё, что не перечислено явно в drop-правилах, открыто из интернета: WebFig 80/443, DNS-резолвер 53, MNDP 5678, IPsec-порты.

Конкретные дефекты:

| Правило | Проблема |
|---|---|
| 0, 1, 14 | три идентичных `accept` для UDP 51999 |
| 3, 4 | флаг `I` — интерфейса `zerotier1` не существует, правила мертвы |
| 5 → 6 | **ошибка порядка:** drop API стоит перед allow для `trusted_api` — разрешающее правило недостижимо |
| 5, 7 | полный дубль друг друга |
| 8, 9 | Telnet и SSH дропаются со **всех** интерфейсов, включая LAN |
| 11 | `in-interface=vlan-A1 out-interface=vlan-clients` без `connection-state` — любой неинициированный входящий трафик проходит в клиентскую сеть |
| 15, 16 | перекрыты правилом 17 (`src-address=10.9.8.0/24` без ограничений) |

---

### В: Почему правила с `in-interface-list=WAN` не работали?

**О:** В списке `WAN` были `ether1` и `sfp-sfpplus1`, а публичный IP — на `vlan-A1`. Трафик приходит на VLAN-интерфейс, физический порт в input-цепочке не участвует. Все правила, привязанные к списку, к реальному WAN не применялись.

```
/interface/list/member/add list=WAN interface=vlan-A1 comment="A1 tagged"
/interface/list/member/add list=LAN interface=wg-srv
```

Без `wg-srv` в `LAN` клиенты подключатся, но не смогут ни в локальную сеть, ни в интернет — трафик умрёт в цепочке forward.

---

### В: Что чистить в NAT?

```
0  chain=srcnat action=masquerade out-interface=vlan-A1
1  ;;; WG Masquerade
   chain=srcnat action=masquerade src-address=10.9.8.0/24 out-interface-list=WAN
```

Правило 0 уже покрывает всё, включая `10.9.8.0/24`. Правило 1 привязано к списку `WAN`, куда `vlan-A1` не входит — никогда не срабатывало.

```
/ip/firewall/nat/remove [find comment="WG Masquerade"]
```

---

### В: Как закрыть сервисы правильно?

Половина правил не нужна, если ограничить сами сервисы:

```
/ip/service/set telnet,ftp,api,api-ssl disabled=yes
/ip/service/set winbox address=10.9.8.0/24,192.168.10.0/24
/ip/service/set www    address=10.9.8.0/24,192.168.10.0/24
/ip/service/set ssh    address=10.9.8.0/24,192.168.10.0/24

/ip/neighbor/discovery-settings/set discover-interface-list=LAN
/tool/mac-server/set allowed-interface-list=none
/tool/mac-server/mac-winbox/set allowed-interface-list=none
/ip/dns/set allow-remote-requests=no
```

Последняя строка важна: при политике `accept` по умолчанию открытый DNS-резолвер доступен из интернета и используется для DNS-amplification атак.

⚠️ **Порядок критичен.** Не ограничивать `winbox` до того, как проверен доступ через WireGuard — иначе гарантированная потеря доступа.

---

### В: Как должна выглядеть цепочка input в подходе «белый список»?

```
/ip/firewall/filter
add chain=input action=accept connection-state=established,related,untracked
add chain=input action=drop   connection-state=invalid
add chain=input action=accept protocol=icmp
add chain=input action=accept protocol=udp dst-port=51999 in-interface=vlan-A1 comment="WireGuard"
add chain=input action=accept src-address=10.9.8.0/24 comment="from WG clients"
add chain=input action=accept in-interface=vlan-mgmt comment="from mgmt VLAN"
add chain=input action=drop   comment="drop everything else"
```

Финальное `drop` добавлять **последним** и обязательно в safe mode (`Ctrl+X` в терминале) — при потере связи конфиг откатится автоматически.

---

## Часть 5. Аудит L2 / VLAN

### В: Влияет ли membership в бридже на скорость порта?

**О:** Нет. Порт согласует свою скорость независимо (`ether2`–`ether8` — 1G, `ether1` — 2.5G, `sfp-sfpplus1` — 10G). Проверить:

```
/interface/ethernet/monitor [find] once
```

Бридж даёт **hw-offload** — коммутацию между портами через switch-чип, минуя CPU (флаг `H`).

**Важный нюанс:** для WAN-порта выигрыш нулевой. Трафик в интернет маршрутизируется на L3, а маршрутизация всегда идёт через CPU — hardware offload на неё не распространяется. Держа WAN в бридже, вы не получаете производительности.

Для management-порта offload тоже не важен: Winbox — это десятки килобит.

---

### В: Опасно ли, что WAN находится в бридже вместе с LAN?

**О:** В данной конфигурации — нет. `vlan-filtering=yes` и `ingress-filtering=yes` изолируют VLAN на уровне switch-чипа. WAN живёт в VLAN 1, клиенты в VLAN 20, управление в VLAN 10 — общего broadcast-домена нет.

---

### В: Что реально требовало исправления в L2?

**1. `frame-types=admit-all` на access-портах.** Защита от VLAN hopping держалась только на `ingress-filtering=yes`. Стоит кому-то добавить порт в таблицу VLAN — появится возможность попасть в чужой VLAN.

```
:foreach i in={"ether1";"ether3";"ether4";"ether5";"ether6";"ether7";"ether8"} do={
  /interface/bridge/port/set [find interface=$i] \
      frame-types=admit-only-untagged-and-priority-tagged
}
```

**2. `pvid=1` у бриджа при WAN на VLAN 1.** Любой новый порт по умолчанию получает `pvid=1` и оказывается в WAN-домене.

```
/interface/bridge/set bridgeLocal pvid=999
```

VLAN 999 остаётся пустой ловушкой — забытый порт не попадает никуда.

**3. `vlan-provider2` построен напрямую на `sfp-sfpplus1`,** который является портом бриджа с `vlan-filtering=yes`. MikroTik это прямо не рекомендует: обработка тегов становится непредсказуемой. Либо вынести порт из бриджа, либо перенести VLAN-интерфейс на бридж.

**4. Все записи в `bridge/vlan` динамические** — выведены из `pvid`. Работает, но конфигурация неявная, и тегированное членство динамически не создаётся вообще.

---

### В: Как настроить `sfp-sfpplus1` как trunk между двумя MikroTik?

```
/interface/bridge/port/set [find interface=sfp-sfpplus1] \
    frame-types=admit-only-vlan-tagged ingress-filtering=yes pvid=999
```

Статическое членство в VLAN (тегированное динамически не создаётся):

```
/interface/bridge/vlan
add bridge=bridgeLocal vlan-ids=10 tagged=bridgeLocal,sfp-sfpplus1 untagged=ether2
add bridge=bridgeLocal vlan-ids=20 tagged=bridgeLocal,sfp-sfpplus1 \
    untagged=ether3,ether4,ether5,ether6,ether7,ether8
add bridge=bridgeLocal vlan-ids=1  tagged=bridgeLocal untagged=ether1
```

⚠️ **`sfp-sfpplus1` не должен входить в VLAN 1** — иначе сегмент провайдера растянется до второго роутера.

`vlan-filtering` на RB5009 обрабатывается switch-чипом, флаг `H` на trunk-порту сохранится — 10G не пострадают. `protocol-mode=rstp` уже включён, защитит от петли.

На втором роутере — зеркальный trunk, те же VLAN тегированными, `vlan-mgmt` для управления. IP-адреса шлюзов и DHCP-серверы остаются только на RB5009; второй роутер работает как коммутатор доступа.

---

## Чеклист незавершённых работ

### Приоритет 1 — поднять туннель

- [ ] Заполнить `client-endpoint` и `client-allowed-address` у пиров `phone` и `pc`
- [ ] Перегенерировать конфиги клиентов из WinBox (свойства пира → конфиг/QR)
- [ ] Проверить `last-handshake` и рост `rx` в `/interface/wireguard/peers/print stats`
- [ ] Добавить `vlan-A1` в список `WAN`, `wg-srv` в список `LAN`
- [ ] Проверить выход клиентов WG в интернет и в `192.168.20.0/24`

### Приоритет 2 — зафиксировать доступ

- [ ] `/system/backup/save` + `/export` перед любыми изменениями firewall
- [ ] Проверить Winbox через туннель на `10.9.8.1`
- [ ] Закрепить маску `/24` на `en7` в настройках macOS
- [ ] Проверить работу консоли micro-USB (кабель USB-C → micro-USB держать рядом)

### Приоритет 3 — чистка firewall

- [ ] Удалить дубли: правила для UDP 51999 (оставить одно), правила `zerotier1`, `WG Masquerade` в NAT
- [ ] Исправить порядок правил API либо отключить сервисы `api`, `api-ssl`, `telnet`, `ftp`
- [ ] Ограничить drop Telnet/SSH через `in-interface-list=WAN` вместо всех интерфейсов
- [ ] Добавить `connection-state=established,related` в правило «NAT <---»
- [ ] Добавить `accept established,related,untracked` и `drop invalid` в начало input
- [ ] Отключить `allow-remote-requests` в DNS либо закрыть 53 из интернета
- [ ] Добавить финальное `drop` в input и forward — **в safe mode, последним шагом**

### Приоритет 4 — L2

- [ ] `frame-types=admit-only-untagged-and-priority-tagged` на всех access-портах
- [ ] `pvid=999` на бридже
- [ ] Перевести `bridge/vlan` на статические записи
- [ ] Решить судьбу `vlan-provider2` (удалить или перенести на бридж)
- [ ] Настроить `sfp-sfpplus1` как trunk перед подключением второго роутера

---

## Полезные команды диагностики

```
# WireGuard
/interface/wireguard/peers/print stats
/interface/wireguard/get [find name="wg-srv"] private-key

# Доходят ли пакеты до WAN
/tool/sniffer/quick interface=vlan-A1 port=51999

# Компактный вид портов бриджа
/interface/bridge/port/print proplist=interface,pvid,frame-types
/interface/bridge/vlan/print

# Проверка публичного адреса
/ip/cloud/print
/ip/address/print

# Скорость линков
/interface/ethernet/monitor [find] once

# Сервисы и их ограничения
/ip/service/print detail
```

Со стороны клиента (macOS):

```
ifconfig | awk '/^[a-z]/{i=$1} /inet /{print i, $2}'
netstat -rn -f inet
arp -n 192.168.10.1
nc -vz 192.0.2.1 9999      # тест на локальный перехват TCP
```

---

## Выводы, которые стоит запомнить

1. **`rx=0` и `tx=0` у пира — клиент не отправлял пакетов.** Причина на стороне клиента или в его конфиге, а не в firewall роутера.
2. **Проверяйте политику цепочки перед тем, как искать блокирующее правило.** При отсутствии финального `drop` цепочка работает в режиме `accept`, и никакое правило ничего не блокирует.
3. **Списки интерфейсов должны содержать те интерфейсы, на которых реально живёт трафик.** VLAN-интерфейс и его физический порт — разные объекты для firewall.
4. **`timeout` и `connection closed` — разные диагнозы.** Первое — firewall, второе — приложение или ограничение сервиса.
5. **Перехватчик TCP на клиенте делает всю сетевую диагностику бессмысленной.** Проверять его наличие нужно до, а не после.
6. **Ограничивайте сервисы через `address=`, а не через `disabled=yes`.** Второе отрезает доступ вместе с угрозой.
7. **Всегда имейте out-of-band доступ.** Консольный кабель дешевле, чем поездка к роутеру.
