---
layout: post
title: "Дизайн на коленке №1 - Резервирование L2 каналов по BGP, между M-Lag CE Huawei и Eltex ESR кластером"
date: '2025/10/04|09:00'
categories: [Routine]
tags: [Дизайн]
published: true
author: Artem Kovalchuk
---

<img src="https://woohung.github.io/assets/images/design-1/design.jpg">

Приветствую, друг!

Нашел время оформить одну из рабочих задач. Частенько кажется, что это "и так понянтно, зачем об этом писать". Задача действительтно вполне обычная - организовать отказоустойчивый доступ к кластеру с виртуалками в локации "ДЦ" из локации "Штаб-квартира", через два L2VPN канала от разных провайдеров.

> Я не архитектор, я только учусь. С дизайном сталкиваюсь редко и это далеко не EVPN VXLAN или какой-нибудь SRv6 или, прости хоспаде, драгонфлай. Но и таких задач (как в этой заметке) много и их тоже нужно решать.  
> Перефразируя классику: *Это моя задача. Таких задач много, но эта - моя.*

В наличии: два ESR-а, софт 1.34.4 (свежак!), два коммутатора Huawei CE и два арендованных L2 канала, принятых VLAN-ами.

Если нет наработанной базы решенных кейсов, часто даже "типовая" задача в руках одного инженера, превращается в твореческие муки "выдумывания" у другого. Второе - это про меня. Есть у этого еще одна особенность - подводные камни. Один из таких камней в нашей задаче - кластер из ESR-ов, производства Eltex, второй - незнакомый мне M-LAG на коммутаторах Huawei CE. 

А теперь по порядку.

Стартовые условия важны. И в реальной жизни они редко бывают идеальными.

Сразу определимся с отношением к "кластеру" у ESR. Да, оно сырое. Да, по определенным причинам оно оказалось в проде. Да, оно работает. Да, баги есть. Да, **не всё** работает.

На площадке "Штаб-квартира" имеем кластер ESR, с софтом 1.34.4, подключенные в VPC-пару aka ядро сети. На каждый юнит приходится по VPC-группе, со стороны ESR собран LACP, поверх собранного port-chanel навешаны sub-интерфейсы...

Все это могло быть так, но кластер ESR пока не поддерживает юнитизацию port-channel (человеческую)...поэтому, у нас все по хардкору - на бриджах.

Так же излишним будет затевать холивар на тему "Что лучше: стек или млаг". В выборе технологий все решают требования.

В нашем случае было требование - максимально обеспечить режим работы сетевого оборудования в режиме 24/7. 

Это значит, что даже обновление коробок должно происходит без потерь трафика.

Стекирование такой роскоши не позволяет, поэтому было решено собирать M-LAG.

> Да, у Huawei в его реализации стэка есть фича "Smooth Upgrade" - позволяет разбить коробки на зоны и провести апдейт без перезагрузки всего стэка (как это обычно с ним и бывает)
> ![Smooth Upgrade](/assets/images/design-1/smooth_upgrade.png)  
> Но на наш YunShan ее довезли в V600R024, а инсталляция собиралась годом ранее.

На площадке "ДЦ" имеем m-lag из двух Huawei CE, с ОС YunShan. 

> У Huawei есть две ОС для сетевого оборудования. Одна из них проприетарная VRP, другая YunShan - с открытым исходным кодом. В них есть отличия, местами прям кардинальные. 
> Если есть возможность выбирать - лучше брать VRP - более стабильно. 
> YunShan можно распознать по названию модели, в ней будет "V2", например S6730-V2.

M-LAG - накладывает свои...гхм...особенности в части конфигурации, например, протоколов динамической маршрутизации или работы с BUM трафиком между нодами.

Если коротко, недостаточно знать "куда жмать", важно хотяб примерно понимать, что будет, когда уже "нажмал".

Дополнительно к этому веселью добавляются VRF-ы, они же vpn-instance на Huawei. Откуда они тут? Да очень просто. Я не придумал ничего лучше, кроме как разбить таблицы маршрутизации, по одной на задачу. Есть VRF с сетями площадки "ДЦ", VRF в котором будет жить некий транзитный трафик, и, наконец, VRF для нашего L2 в сторону "Штаб-квартиры"

Внутрь L2 мы положим транзитные сетки /29, чтобы поверх них накрутить то, что будем наиболее удобным для перетаскивания маршрутов между локациями.

Как говорил один знакомый дядька-сетевик: "Не понимаешь - рисуй схемку". Начнем с нее.

![Набросок схемы](/assets/images/design-1/first_scheme.png)  

Сразу стоит ответить почему /29. Ответ прост - потому что кластер и потому что M-LAG. Если у вас был опыт и есть варианты еще - делитесь в комментариях в телеграм-канале :)

Если коротко, нам нужно обеспечить хождение трафика от сетей "Штаб-квартиры" до сетей "ДЦ" вне зависимости от жизнеспособности каналов или железок.

Даже если будет так...

![Вариант отказа системы](/assets/images/design-1/scheme_fail.png) 

...все должно работать.

Условия понятны, схемку накидали - время ~~делать КОРМУШКИ!~~...лабить. Если, конечно, есть на чем.

![КОРМУШКИ](/assets/images/design-1/red21.png)

У Eltex есть виртуальный vESR - запрашиваем его напрямую у завода, [собираем образ](https://github.com/alekho/EVE-NG_vESR) для EVE-NG/Pnetlab. 

> Виртуальный [vESR](https://disk.yandex.ru/d/P50DWdPAvzT1JA) с особенностями...одна из них - обязательно указывать уникальный первичный mac-адрес перед первым запуском ноды иначе перезапуск ноды вывалится в ошибку. Проблема известна, есть шансы, что починять в 1.37. На остальных мы останавливаться пока не будет.

С Huawei плясок будет не меньше, в еве у него определенные проблемы с L2 функционалом т.е по чти все, что связано с VLAN - работать не будет или будет делать это некорректно. А вот control plane собрать можно.

Ах да, M-LAG в лабе собрать не получится. Но поверьте на слово - он легко собирается по документации. Главное не забыть про аутентификацию в dfs-group:

```
dfs-group 1
 authentication-mode hmac-sha256 password
```
иначе рискуете влипнуть в...

![WHY?](/assets/images/design-1/why.png)

Возвращаясь к образу, для "изучения синтаксиса" вполне подойдет [CE6800](https://disk.yandex.ru/d/apot8Ts4nYnSAA), это VRP, но синтаксис по нашей задаче от YanShan отличается не сильно. Разве что некоторые специфические фичи доезжают позднее, чем в VRP.

> Можно рядом собрать ENSP - виртуальная среда для Huawei, там все работает исправно и просто прокинуть доступы между двумя эмуляторами. Однажды я такое делал - работает. Если захотите повторить - [ловите "рассыпуху"](https://disk.yandex.ru/d/ACUh_QDCCU1azg) по eNSP.

Промежуточными свичами можно взять условные cisco vios.

По используемым сетям:

| Подсеть          | Назначение                     | VLAN |
|------------------|--------------------------------|------|
| 10.10.20.16/29   | L2VPN #1                       | 200  |
| 10.10.20.24/29   | L2VPN #2                       | 201  |
| 172.20.1.24/29   | Синхронизация кластера ESR     | 500  |
| 192.168.1.0/24   | Сеть в локации "Штаб-квартира" | 10   |
| 192.168.100.0/24 | Сеть в локации "ДЦ"            | 110  |

Перед конфигом кластера нужно сделать пару вещей.
В образе vESR системный MAC будет одинаковым. Кластер в таком виде вы собрать не сможете. 

Перед конфигом нужно:
- изменить серийный номер, от него зависит системный MAC, командой `set serial-number VESRXXXXXXX`. Например, это может выглядеть так `set serial-number VESR0000001`, а у второй ноды, соответственно `set serial-number VESR0000002`
- назначить юниты командой `set unit id <num>`

Будет как-то так:

![vESR show system](/assets/images/design-1/vesr_system.png)

[Собираем](https://docs.eltex-co.ru/pages/viewpage.action?pageId=560463907#id-%D0%A3%D0%BF%D1%80%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5%D0%BA%D0%BB%D0%B0%D1%81%D1%82%D0%B5%D1%80%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D0%B5%D0%B9-%D0%9D%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B0%D0%BA%D0%BB%D0%B0%D1%81%D1%82%D0%B5%D1%80%D0%B0) кластер ESR, базово конфигурим фаервол (или вообще лепим в глобале `ip firewall disable`, настраиваем интерфейсы и бриджи. Получится что-то такое:

```
cluster
  cluster-interface bridge 500
  unit 1
    mac-address a2:00:00:00:10:00
  exit
  unit 2
    mac-address a2:00:00:00:20:00
  exit
  enable
exit

hostname srt-01 unit 1
hostname srt-02 unit 2

interface gigabitethernet 1/0/1
  mode switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,200,201
exit
interface gigabitethernet 1/0/2
  description "Cluster Sync Link"
  mode switchport
  switchport access vlan 500
exit
interface gigabitethernet 2/0/1
  mode switchport
  switchport mode trunk
  switchport trunk allowed vlan add 10,200,201
exit
interface gigabitethernet 2/0/2
  description "Cluster Sync Link"
  mode switchport
  switchport access vlan 500
exit

bridge 127
  description "L2VPN #1"
  vlan 200
  security-zone trusted
  ip address 10.10.10.17/29 unit 1
  ip address 10.10.10.18/29 unit 2
  vrrp 127
    ip address 10.10.10.21/29
    group 1
    enable
  exit
  no spanning-tree
  enable
exit
bridge 128
  description "L2VPN #2"
  vlan 201
  security-zone trusted
  ip address 10.10.10.25/29 unit 1
  ip address 10.10.10.26/29 unit 2
  vrrp 128
    ip address 10.10.10.29/29
    group 1
    enable
  exit
  no spanning-tree
  enable
exit

bridge 500
  vlan 500
  security-zone cluster_sync
  ip address 172.20.1.26/29 unit 1
  ip address 172.20.1.27/29 unit 2
  vrrp 20
    ip address 172.20.1.28/29
    group 1
    enable
  exit
  enable
exit

```

> **Несколько нюансов кластера ESR:**
> - на текущий момент, интерфейс синхронизации кластера можно собрать только на бриджах
> - даже при условии, что VRRP в своей сути позволяет выбрать VIP-адресом один из настоящих IP-адресов, кластер этого не потерпит. **VIP-адрес должен быть уникальным и точка**. Вот зачем мы выбрали сетку для, по сути, ptp интерфейса /29, а не /30 или /31.
> - **ни в коем случае** не суйте VLAN синхронизации в даунлинки
> - **ни в коем случае** не суйте сетку синхронизации в анонсы куда либо

Так же расправляемся и с Huawei. Нстраиваем vpn-instance, конфигурим L3 интерфейсы (помним про нерабочий L2) на проде, соответственно, VLANIF (аля SVI у Cisco), пример будет чуть ниже. С L3 будет выглядеть так:

```
ip vpn-instance HQ-L2
 ipv4-family
  route-distinguisher 65550:102
  vpn-target 65550:102 export-extcommunity
  vpn-target 65550:102 import-extcommunity
  vpn-target 65550:101 import-extcommunity
#
ip vpn-instance LAN
 ipv4-family
  route-distinguisher 65550:101
  vpn-target 65550:101 export-extcommunity
  vpn-target 65550:101 import-extcommunity
  vpn-target 65550:102 import-extcommunity
#

interface GE1/0/2
 undo portswitch
 undo shutdown  
 ip binding vpn-instance HQ-L2
 ip address 10.10.10.22 255.255.255.248
#
interface GE1/0/3
 undo portswitch
 undo shutdown
 ip binding vpn-instance HQ-L2
 ip address 10.10.10.30 255.255.255.248
#

```

Основу собрали. Теперь можно думать "а как же собрать резерв". Но желательно, чтобы каждая сторона +- одновременно понимала, что другая сторона или канал - мертвы и переключалась на резерв. А когда все вернется в строй - возвращалась обратно на основной путь.

Про MPLS и VXLAN тут не будет, извините.

На связку "статик-роуты + SLA + Track" мы не смотрим т.к сетей с обеих сторон уже 10+ и может стать больше.

На мой скромный опыт, с учетом того, что VLAN-ны таскать не требуется, очевидный выбор - протокол динамической маршрутизации. Но какой?

OSPF выглядел вполне логичным. Собираем соседства "каждый с каждым", накидываем косты на интерфейсы, чтобы раскидать приоритеты, анонсим сетки - готово! 

Конфигов приводить не буду т.к тут меня ждало розочарование...OSPF в кластере ESR работает, мягко скажем, плохо. Соседства собираются, но не на той ноде (хотелось бы на active, а не на backup), что-то нужно придумать с соседством между двух нод кластера т.к каждый видит каждого, разобраться с DR/BDR и зонами...в общем, одновременно сражаясь с желанием не лезть в BGP на Huawei...все таки был выбран BGP.

Мало того, что BGP позволит нам определить соседей самостоятельно, дык он еще и в кластере ESR более менее поддержан! И мне даже доводилось его конфигурить. Но вот Huawei...да в разных vpn-instance...предстояло основательно покурить доку.

![Курим доку](/assets/images/design-1/smoke_doc.png)

Сначала нужно разобраться - как слить маршруты из одного vpn-instance в другой? В cisco это можно сделать КУЧЕЙ способов, а что у Huawei?

Можно нарисовать статик-роуты между инстансами, а можно через BGP! Конечно же. Больше BGP.

```
bgp 65550
 private-4-byte-as enable
 #
 ipv4-family vpn-instance HQ-L2
 #
 ipv4-family vpn-instance LAN
  import-route direct
 #
```

Этого конфига, в связке с настроенными таргетами vpn-instance в начале, будет достаточно, чтобы слить маршруты из LAN в HQ-L2.

Следующий вопрос - как будем получать маршруты от HQ? Ответ мы уже знаем - BGP.

![Роутим!](/assets/images/design-1/routing_classic.png)

Начнем с ESR. BGP будем строить с VRRP адреса, т.к нас вполне устраивает "строить соседства только с активной ноды кластера". 

Нужно будет собрать 4 соседа, по два на каждый L2VPN:

- 10.10.10.21 - ESR VRRP L2VPN #1
- 10.10.10.29 - ESR VRRP L2VPN #2

Конфиг будет следующий:
```
router bgp 65560
  router-id loopback 1
  neighbor 10.10.10.19
    remote-as 65550
    weight 120
    update-source 10.10.10.21
    fall-over bfd
    address-family ipv4 unicast
      route-map bgp-in in
      route-map bgp-out out
      default-originate
      enable
    exit
    enable
  exit
  neighbor 10.10.10.20
    remote-as 65550
    weight 20
    update-source 10.10.10.21
    fall-over bfd
    address-family ipv4 unicast
      route-map bgp-in in
      route-map bgp-out out
      default-originate
      enable
    exit
    enable
  exit
  neighbor 10.10.10.27
    remote-as 65550
    weight 110
    update-source 10.10.10.29
    fall-over bfd
    address-family ipv4 unicast
      route-map bgp-in in
      route-map bgp-out out
      default-originate
      enable
    exit
    enable
  exit
  neighbor 10.10.10.28
    remote-as 65550
    weight 10
    update-source 10.10.10.29
    fall-over bfd
    address-family ipv4 unicast
      route-map bgp-in in
      route-map bgp-out out
      default-originate
      enable
    exit
    enable
  exit
  address-family ipv4 unicast
    redistribute connected route-map rd-to-DC
      exit
  enable
exit
```

`update-source <ip>` - строим соседства с конкретного сорс-адреса - VRRP адреса

Роут мапа bgp-in - для контроля получаемых маршрутов, тут по желанию, а вот bgp-out обязателен т.к по умолчанию в ESR анонс [EBGP без route-map запрещен](https://docs.eltex-co.ru/pages/viewpage.action?pageId=47022191) и важно не забыть выкинуть из анонса сетку для синхронизации кластера:

```
route-map bgp-out
  rule 1
    match ip address 172.20.1.24/29
    action deny
  exit
  rule 2
  exit
exit
```

`redistribute connected` - заанонсим редистрибьютом connected сетки, с привязанной к route-map, для выбора только необходимых.

`fall-over bfd` - запустим bfd

`weight <value>` - разрулим выбор best-path с помощью локального атрибута weight

После commit мы должны увидеть в `sh bgp summary` сессии в состоянии `Connect`.


```
srt-01# sh bgp summary 
2025-10-03 00:35:01
  BGP router identifier 10.20.31.1, local AS number 65560
  BGP activity 0/0 prefixes
  Neighbor                 AS              MsgRcvd      MsgSent      Up/Down      St/PfxRcd
                                                                     (d,h:m:s)                
  ----------------------   -------------   ----------   ----------   ----------   ------------
  10.10.10.20             65555           0            0            00,11:58:45   Connect     
  10.10.10.19             65555           0            0            00,11:58:45   Connect     
  10.10.10.28             65555           0            0            00,11:58:45   Connect     
  10.10.10.27             65555           0            0            00,11:58:45   Connect    
```

Теперь давайте разберемся, откуда со стороны huawei аж 4 соседа?

Тут стоит добавить, что сервисы приняты VLAN-ом на другой паре коммутаторов, которые подключены в M-LAG пару. В таком случае документация Huawei предлагает всего один вариант - собирать протокол динамической маршрутизации ["over M-LAG"](https://support.huawei.com/enterprise/en/doc/EDOC1100304980/6fbe4d29/configuring-bgp-or-bgp4-over-m-lag). Так и поступим.

Сам M-LAG собран в режиме dual-active gateway, это тот же VRRP (на каждом из VLANIF нужно указать mac-address 0000-5e00-01xx, где XX это VRID), но все же ближе к anycast-gateway т.к трафик может попасть как на первую ноду, так и на вторую и будет успешно обработан.

Ключевым поинтом в настройке будет `m-lag ip address` и `mac-address` на каждом VLANIF с нашими транспортными сетками. Адреса должны быть уникальными на каждой коробке. И тут нас снова выручает выбранная /29.

Выглядеть все будет как-то так, пример уже с прод. конфига на VLANIF интерфейсах:

```
### CE #1
interface Vlanif201
 ip binding vpn-instance HQ-L2
 ip address 10.10.10.30 255.255.255.248
 arp proxy enable
 mac-address 0000-5e00-0102
 m-lag ip address 10.10.10.27 255.255.255.248
#
interface Vlanif200
 ip binding vpn-instance HQ-L2
 ip address 10.10.10.22 255.255.255.248
 arp proxy enable
 mac-address 0000-5e00-0101
 m-lag ip address 10.10.10.19 255.255.255.248

### CE #2
interface Vlanif201
 ip binding vpn-instance HQ-L2
 ip address 10.10.10.30 255.255.255.248
 arp proxy enable
 mac-address 0000-5e00-0102
 m-lag ip address 10.10.10.28 255.255.255.248
#
interface Vlanif200
 ip binding vpn-instance HQ-L2
 ip address 10.10.10.22 255.255.255.248
 arp proxy enable
 mac-address 0000-5e00-0101
 m-lag ip address 10.10.10.20 255.255.255.248
#
```

В конечном итоге, каждый VLANIF будет с уникальным адресом, от которого необходимо будет построить самодостаточные BGP-сессии.

Не забываем, что BGP у нас уже есть, он занимается раскидыванием роутов между vpn-instance. Куда пихать конфиг соседей?

В отдельный vpn-instance того же BGP процесса, конечно.

> Изначально, правда, казалось, что потребуется другой BGP процесс, помещенный в свой vpn-instance (и так даже можно сделать!). Успешно потратил некоторое время на попытки выяснить, что мне этот вариант не подходит.

Вот как выглядит BGP конфиг на обе ноды (в добавок к тому, что уже собрали выше):

```
### CE #1
bgp 65550  
 #
 vpn-instance HQ-L2
  peer 10.10.10.21 as-number 65560
  peer 10.10.10.21 connect-interface Vlanif200 10.10.10.19
  peer 10.10.10.21 bfd enable
  peer 10.10.10.29 as-number 65560
  peer 10.10.10.29 connect-interface Vlanif201 10.10.10.27
  peer 10.10.10.29 bfd enable
 #
 ipv4-family vpn-instance HQ-L2
  network 192.168.100.0 255.255.255.0
  peer 10.10.10.21 enable
  peer 10.10.10.21 preferred-value 120
  peer 10.10.10.29 enable
  peer 10.10.10.29 preferred-value 110
 #
### CE #2
bgp 65550
 #
 vpn-instance HQ-L2
  peer 10.10.10.21 as-number 65560
  peer 10.10.10.21 connect-interface Vlanif200 10.10.10.20
  peer 10.10.10.21 bfd enable
  peer 10.10.10.29 as-number 65560
  peer 10.10.10.29 connect-interface Vlanif201 10.10.10.28
  peer 10.10.10.29 bfd enable
 #
 ipv4-family vpn-instance HQ-L2
  network 192.168.100.0 255.255.255.0
  peer 10.10.10.21 enable
  peer 10.10.10.21 preferred-value 120
  peer 10.10.10.29 enable
  peer 10.10.10.29 preferred-value 110
 #
```

`vpn-instance HQ-L2 `- внутри нужного vpn-instance указываем соседей

`peer <nbr ip> bfd enable` - включаем bfd

`peer <nbr ip>  connect-interface Vlanif200 <update source ip>` - указываем наш m-lag сорс адрес. С него будет устанавливаться BGP-сессия

В `ipv4-family vpn-instance HQ-L2` активируем соседей `peer <nbr ip> enable` и раскидаем приоритет роутов через `preferred-value`

Проверим со стороны huawei, что с BGP соседями все ок:

```
[swc-01]dis bgp vpnv4 vpn-instance HQ-L2 peer 
 Status codes: * - Dynamic
 BGP local router ID : 10.20.30.11
 Local AS number : 1.14

 VPN-Instance HQ-L2, Router ID 10.20.30.11:
 Total number of peers      : 2
 Peers in established state : 2
 Total number of dynamic peers : 0

  Peer                             V          AS  MsgRcvd  MsgSent  OutQ  Up/Down       State  PrefRcv
  10.10.10.21                     4        1.24    10136    10180     0 0147h32m Established        4
  10.10.10.29                     4        1.24    10120    10182     0 0147h32m Established        4
```

Кстати, нужные маршруты, полученные в vpn-instance HQ-L2, до vpn-instance LAN доедут самостоятельно т.к уже изучены по BGP

```
[swc-01]dis ip routing-table vpn-instance LAN 
Proto: Protocol        Pre: Preference
Route Flags: R - relay, D - download to fib, T - to vpn-instance, B - black hole route
------------------------------------------------------------------------------
Routing Table : LAN
         Destinations : 67       Routes : 67        

Destination/Mask    Proto   Pre  Cost        Flags NextHop                                  Interface

       192.168.1.0/24  EBGP    255  0             RD  10.10.10.21                             Vlanif200
```

Со стороны кластера ESR в роутах тоже все в порядке
```
srt-01# sh ip ro
Codes: C - connected, S - static, R - RIP derived,
       O - OSPF derived, IA - OSPF inter area route,
       E1 - OSPF external type 1 route, E2 - OSPF external type 2 route,
       B - BGP derived, D - DHCP derived, K - kernel route, V - VRRP route,
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area,
       H - NHRP, * - FIB route
       
B     * 192.168.100.0/24     [170]             via 10.10.10.19 on br127         [bgp65560 2025-09-25] (AS65550?)
```

Какие могут быть нюансы? M-LAG huawei, по сути, позволяет ходить трафику наверх как с первой ноды, так и со второй.

Исходя из документации Huawei о том, как работает dual-active:

> *North-south unicast traffic from the access device is load balanced to M-LAG master and backup devices through Eth-Trunks. After receiving the traffic, the two devices forward it to the network side based on their respective routing table.*

Например, гипотетически возможен вот такой сценарий...

![Сценарий отказа](/assets/images/design-1/fail.png)

Будет ли такой ответ расценен ESR как асинхронный роут? :) (ESR не церимонится с асинхронной маршрутизацией и сразу дропает такие пакеты)

Как будто бы не должно, ибо запрос и ответ уходят/приходят (если все в норме) с одного интерфейса - br127, а в какую корзинку LACP попадет трафик со стороны мемберов M-LAG Huawei(учитывая dual-gateway) должно быть не сильно важно - unicast трафик раскидается куда нужно.

Еще, возможно имеет смысл докинуть ECMP для BGP со стороны ESR, дабы пускать трафик в оба CE в рамках одного L2, т.к dual-gateway все равно пережует трафик с двух направлений. Но т.к прод не жалуется, пока остановились на таком варианте итоговой схемы:

![Итоговая схема](/assets/images/design-1/final_scheme.png)

У нас выгрызть окно на работы - та еще задача, поэтому проверено только переключение роутов в случае отвала одной из услуг. В будущем запланирована проверка с "отказом" M-LAG пары и одной из нод кластера. BGP наверняка отработает ожидаемо, а все сюрпризы, как это наверняка было у многих с экзаменационными билетами - выпадет тот, который "не учил".

<p></p>
<hr>
<h2>Хочешь обсудить тему?</h2>
С вопросами, комментариями и/или замечаниями, приходи в [чат](https://t.me/netautomationarea) или подписывайся на [Telegram-канал](https://t.me/+Jeoaxn2kby4zMWUy).

