
# Policy-Based Routing

Разделяют три вида роутинга:

- **Destination-routing (DR)** - то, что мы обычно понимаем под роутингом, когда выбор пути происходит на основе адреса назначения пакета.
- **Policy-based routing (PBR)** - роутинг на основе правил, которые завязаны на те или иные атрибуты пакета (порты, протокол и т.д.).
- **Source-routing (SR)** - роутинг на основе адреса источника пакета, в Linux считается частным случаем PBR.

PBR используют когда нужно отправлять определенные пакеты через определенные интерфейсы, руководствуясь не адресом назначения, или по крайней мере - не только им.

Для управления PBR испольуется пакет утилит iproute2.

Чтобы включить поддержку PBR ядром Linux, должны быть включены опции:

- **CONFIG\_IP\_ADVANCED\_ROUTER**
- **CONFIG\_IP\_MULTIPLE\_TABLES**

## ip route

Для настройки роутинга используется команда `ip route`. Без параметров она показывает список текущих правил маршрутизации.

Пусть у нас есть интерфейс eth0 с адресом 192.168.12.10/24 и шлюзом по умолчанию 192.16.12.1, тогда таблица будет выглядеть следующим образом:
```bash
#ip route
192.168.12.0/24 dev eth0 proto kernel scope link src 192.168.12.101
default via 192.168.12.1 dev eth0
```

- **proto kernel** - означает, что маршрут был создан ядром автоматически при назначении адреса интерфейсу.
- **scope link** - означает, что эта запись действительна только для интерфейса (eth0).
- **src** - задает адрес отправителя для пакетов, подпадающих под это правило.

## Таблицы маршрутизации

На самом деле таблиц маршрутизации несколько, можно создавать собственные. Изначально определены таблицы:

- **local** - записи локальных и широковещательных адресов, для того чтобы такой трафик оставался локальным.
- **main** - является основной, именно ее мы видим при `ip route` без параметров.
- **default** - пустая.

Разберем содержимое local:
```bash
# ip route show table local
broadcast 127.255.255.255 dev lo  proto kernel  scope link  src 127.0.0.1
broadcast 192.168.12.255 dev eth0  proto kernel  scope link  src 192.168.12.101
broadcast 192.168.12.0 dev eth0  proto kernel  scope link  src 192.168.12.101
local 192.168.12.101 dev eth0  proto kernel  scope host  src 192.168.12.101
broadcast 127.0.0.0 dev lo  proto kernel  scope link  src 127.0.0.1
local 127.0.0.1 dev lo  proto kernel  scope host  src 127.0.0.1
local 127.0.0.0/8 dev lo  proto kernel  scope host  src 127.0.0.1
```

- **broadcast и local** - типы записей, для broadcast-пакетов и локальных соответственно.
- **scope host** - указывает, что эта запись действует только для хоста.

Для просмотра конкретной таблицы: `ip route show TABLE_NAME`.

Для просмотра всех таблиц: `ip route show all/0/unspec`.

## ip rule

Для того, чтобы выбрать какую именно таблицу маршрутизации для пакета выбрать, ядро пользуется правилами:
```bash
# ip rule
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default
```

- **число в начале** - идентификатор правила.
- **from all** - условие выбора правила.
- **lookup** - указывает какую таблицу выбрать.

> Пакет проходит по правилам сверху-вниз (от меньшего id к большему) и останавливается как только находит валидную запись в соответсвующей таблице маршутизации.

Возможные условия:

- **from** - проверка адреса отправителя
- **to** - проверка адреса назначения
- **iif** - имя интерфейса, на который пришел пакет
- **oif** - имя интерфейса, с которого ушел пакет (работает только для пакетов, исходящих из локальных сокетов)
- **tos** - поле TOS IP-пакета
- **fwmark** - метка FWMARK пакета, которая может быть присвоена в iptables.

> Условия можно комбинировать: from 10.10.10.0/24 to 10.10.20.0/24 и использовать отрицание при помощи not.

## Примеры

- Есть некий шлюз, на который приходят пакеты с адреса 192.168.1.20. Пакеты с него нужно отправлять на шлюз 10.1.0.1:
```bash
ip route add default via 10.1.0.1 table 120
ip rule add from 192.168.1.20 table 120
```

- Доступность сервера с двух ISP. Пусть у нас есть веб-сервер, подключенный к двум провайдерами, причем на одного из них настроен шлюз по умолчанию. В таком случае сервер будет доступен только из сети, на которую смотрит этот шлюз, так как пакеты из сети другого провайдера будут доходить до сервера, но уходить будут через тот самый шлюз по умолчанию:
```bash
ip route add default via 11.22.33.1 table 101
ip route add default via 55.66.77.1 table 102
ip rule add from 11.22.33.44 table 101
ip rule add from 55.66.77.88 table 102
```

## Балансировка трафика

Реализуется при помощи весов шлюзов и замене существущих в main записях для них:
```bash
ip route replace default scope global \
nexthop via 11.22.33.1 dev eth0 weight 1\
nexthop via 55.66.77.1 dev eth1 weight 1 \
```

Таким образом трафик будет распределяться 50 на 50, если бы мы выставили 7 и 3 - распределяляся бы как 70% на 30%.

> Маршруты кэшируются и висят в таблице некоторое время после обращения к ним. Иногда кэш не успевает обновляться из-за большого объема трафика. Чтобы исправить проблему: `ip route flush cache`.

## Маркировка пакетов в iptables

```bash
iptables -t mangle -A OUTPUT -p tcp -m tcp --dport 80 -j MARK --set-mark 0x2
ip route add default via 11.22.33.1 dev eth0 table 102
ip rule add fwmark 0x2/0x2 lookup 12
```

Существует модуль CONNMARK, который позволяет отслеживать и маркировать все пакеты, относящиеся к определенному соединению. Например, мы маркируем пакеты в цепочке INPUT, а затем автоматически маркируем пакеты этих соединений уже в цепочке OUTPUT:
```bash
iptables -t mangle -A INPUT -i eth0 -j CONNMARK --set-mark 0x2
iptables -t mange -A INPUT -i eth1 -j CONNMARK --set-mark 0x4
iptables -t mangle -A OUTPUT -j CONNMARK --restore-mark
```
