---
layout: post
title: "Воскрешаем Dynamic VPN на Juniper SRX100H2"
date: '2023/02/09|15:00'
categories: [Work routine]
tags: [Некромантия]
published: true
author: Artem Kovalchuk
---

<img src="https://woohung.github.io/assets/images/necromancer.jpg">

Приветствую, друг!
Это статья-заметка, отражающая рутину сетевого инженера. Не всегда, знаете ли, приходится запускать интересные проекты :)  
Тут я добрался, наконец, до Juniper. Жаль, что не что-то современное и актуальное, но что поделать, чем больше работаю в сетях, тем больше убеждаюсь, некромантия - это неизбежно :)  
В любом случае, я пострадал и разобрался, вдруг кому пригодится результат.  

**Пациент:** SRX100H2 (H2 - 2G DRAM памяти) с древним софтом (12.1X44-D15.5) и к тому же EoL.  
Конфиг можно считать стандартным: базовый NAT, зоны, правила и dynamic vpn по официальной доке.  
**Проблема:** Dynamic VPN не работает от слова совсем. Подключение реализуется через Junos Pulse. На этапе, когда "все работало" клиент использовал Junos Pulse 2.0  
**Дано:**  
- идентичная железка, на которой можно поэкспериментировать (что замечательно)
- конфиг-файл с железки в проде
- пакет Junos Pulse 2.0

![stp vlan](/assets/images/juniper_srx100.jpg)  

Немного информации о работе с Junos OS для тех, кто, так же как я, впервые столкнулся с ней.  
Junos собрана на основе FreeBSD. После логина, вы попадаете в Unix shell, `%` подскажет вам об этом.  
Чтобы перейти в более менее классический cli, нужно использовать команду `cli`, после чего, приветствие с `%` изменится на `>` - рабочий режим или *operational mode*.  
Чтобы перейти в режим конфигурации, вводим команду `configure`, приветствие с `>` изменится на `#`.  
Как выглядят разные режимы в консоли:  

```
root@srx100h2% cli    <- Unix shell
root@srx100h2> configure    <- Operational mode
Entering configuration mode
[edit]
root@srx100h2#    <- Configuration mode
```

Дальнейшая работа в cli мало чем отличается от остальных вендоров. Из особенностей:  
- `commit/rollback` конфигуарции как часть процесса конфигуарции
- `set` задать что-либо
- `delete` удалить что-либо
- `request` попросить систему сделать что-либо (например отправить в ребут `request system reboot`)
- `show` в operational mode работает предсказуемо - показывает инфу о чем-либо, в режиме configuration mode работает как вьювер json файла конфигурации - показывает структуру запрашиваемого куска конфига в виде json.

Вроде все, поехали разбираться.  

## Заливаем конфиг
Предварительно ознакомившись с конфигом в текстовом редакторе, закинем его на устройство. В Junos конфиг записывается в формате `json`, если не знакомы, [документ](https://higherlogicdownload.s3.amazonaws.com/JUNIPER/MigratedAttachments/BD2B2715-FA41-4B55-93C4-B6F775D18EE0-2-JSON-Whitepaper.pdf) с примерами.  
Мы можем скопировать конфиг прямо в терминал, используя команду: (не построчно, а весь файл целиком) `load merge terminal` Выполняется из режима `#`.  
Кроме необходимого нам `merge` у нее есть несколько опций, о которых, в том числе, можно почитать [тут](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-config-files-loading.html).  

После ввода команды, увидим приглашение ко вводу. Открываем файл в блокноте, CTRL+A и копируем в терминал:  
```
[edit]  
root@srx100h2# load merge terminal  
[Type ^D at a new line to end input]  
>    <- Вставляем конфиг сюда
```

В конце жмем ENTER, чобы перейти на следующую строку и далее CTRL+D, чтобы закончить ввод.  

>Командой `show` можно посмотреть конфиг целиком, прежде чем его закоммитить.

Теперь используем команду `commit` и готово, конфиг применен.    

## Заливаем софт
Ранее я уже возился с воскрешением ASA5505, поэтому мы сразу попробуем обновить софт.  
Для начала нужно определиться, какой софт подходит. Для этого [смотрим](https://supportportal.juniper.net/s/article/Junos-Software-Versions-Suggested-Releases-to-Consider-and-Evaluate?language=en_US), какой последний релиз на SRX100.  

Видим, нужная нам версия: `Junos 12.3X48-D105`. Раздобыть прошивки не так сложно, но в текущих реалиях нужно таки немного постараться. Используемые мной прошивки для решения данной задачи, я закинул на [ЯД](https://disk.yandex.ru/d/P6nZwqq4GFeBwA), либо можно поглядеть, например [тут](https://archive.org/download/junos-srxsme).  

При попытке накатить софт, доходим до того, что просто так прыгнуть на последнюю версию не выйдет. Выхватываем ошибку формата `WARNING: Package 12.3X48-Dxx is not compatible with this hardware`, [тут  подробнее](https://community.juniper.net/communities/community-home/digestviewer/viewthread?MID=67555).  

Наша версия очень старая, поэтому необходимо сначала обновиться до промежуточной, а затем до последней. Качаем нужный софт, далее заливаем в железку. Я делал с помощью WinSCP по FTP, предварительно включив его командой: (не забудьте выключить после)  
```
root@srx100h2> confgigure
root@srx100h2# set system services ftp
root@srx100h2# commit
```

Подключаемся по FTP, используя адрес железки внутри локальной сети и пароль администратора.  
Кладем софт (в формате `.tgz`) прямиком в /var/tmp/.  
Переходим в терминал и проверяем, что софт на месте:  
```
root@srx100h2> file list /var/tmp/
/var/tmp/:
cleanup-pkgs.log
eedebug_bin_file
gksdchk.log
gres-tp/
idp_license_info
install/
junos-srxsme-12.1X46-D25.7-domestic.tgz <- промежуточный
junos-srxsme-12.3X48-D105.4-domestic.tgz <- последний для SRX100H2
kmdchk.log
krt_gencfg_filter.txt
nsd_restart
pics/
policy_status
rtsdb/
sec-download/
spu_kmd_init
vi.recover/
vpn_tunnel-_orig.id
vpn_tunnel_orig.id
```

Осталось запустить процесс установки (отдельно для каждой версии) из `>` режима консоли.  

> Используем доп. опцию `no-validate` т.к нам ни к чему проверять файл прошивки и `reboot` по окончанию всех операций.

```
root@srx100h2> request system software add /var/tmp/junos-srxsme-12.1X46-D25.7-domestic.tgz no-validate reboot

```

После промежуточного обновления, рекомендую проверить `show version`, если все встало, проворачиваем то же самое для следующего файла.  

### Полезные мелочи
Осталось обновить BIOS, обновить резервный образ и сохранить рабочий конфиг в резервный.  

Проверяем версию BIOS (с новой прошивкой будет и новая версия BIOS)  
```
root@srx100h2> show system firmware
```

![Check BIOS](/assets/images/SRX100/BIOS_upd.png)  

Тут все просто, если Current совпадает с Available - ничего делать не нужно, если нет - нужно.  

Обновим сразу и основной и бэкап:  
```
root@srx100h2> request system firmware upgrade re bios
root@srx100h2> request system firmware upgrade re bios backup
```

Статус обновления можно смотреть в выводе предыдущей команды `show`. Сначала будет состояние - PROGRAMMING, а после успешного завершения - UPGRADED SUCCESSFULLY.  

![Update BIOS](/assets/images/SRX100/BIOS_upd_success.png)  

На скриншоте показано успешное обноление BIOS, после которого можно отправлять железку в ребут командой:  
```
root@srx100h2> request system reboot
Reboot the system ? [yes,no] (no) yes 
```

Теперь обновим резервный образ, который используется, если с основным что-то случилось. Обычно, Junos OS предупреждает об этом следующим сообщением:  

![Failed image](/assets/images/SRX100/failed_image.png)  
Такой баннер вы точно не пропустите :)  

Обновление производится в одну команду:  
```
root@srx100h2# request system snapshot slice alternate 
Formatting alternate root (/dev/da0s2a)...
Copying '/dev/da0s1a' to '/dev/da0s2a' .. (this may take a few minutes)
The following filesystems were archived: /
```

Можно посмотреть текущее состояние резервной копии. Видим, что есть primary и backup, версии должны совпадать:  
```
root@srx100h2> show system snapshot media internal 
Information for snapshot on       internal (/dev/da0s1a) (primary)
Creation date: Feb 8 15:20:35 2023
JUNOS version on snapshot:
  junos  : 12.3X48-D105.4-domestic
Information for snapshot on       internal (/dev/da0s2a) (backup)
Creation date: Feb 8 16:25:58 2023
JUNOS version on snapshot:
  junos  : 12.3X48-D105.4-domestic
```

Осталось обновить резервный конфиг, делается командой:  
```
request system configuration rescue save
```

Теперь мы сможем легко вернуться на резервный конфиг:  
```
rollback rescue
commit check
commit
```

## В чем же проблема
В SRX из коробки есть две бесплатные лицензии dynamic vpn, их клиенту хватает, в конфиге всего два пользователя. Проблему с лицензиями отметаем.  

![Check license](/assets/images/SRX100/lic.png)  

Так же обновления софта проблема не решилась, при попытке подключиться через Junos Pulse 2.0, получаем бесконечные реконекты.  

Первым делом я нашел актуальную версию Junos Pulse, положил так же на [ЯД](https://disk.yandex.ru/d/KonN78vE7DiEUw), если кому пригодится.  

Подключиться к VPN с Junos Pulse 9.1 так же не удалось.  

![Failed connect Pulse](/assets/images/SRX100/failed_connect_pulse.png)  

Изучив конфигурацию подробнее, я заметил один момент, в `gateway dyn-vpn-local-gw`-> `external-interface` сконфигурирован как физический `fe-0/0/0`, а основной внешний интерфейс на железке - логический `fe-0/0/0.0`.  
```
gateway dyn-vpn-local-gw {
	ike-policy ike-dyn-vpn-policy;
	dynamic {
		hostname dynvpn;
		connections-limit 10;
		ike-user-type group-ike-id;
	}
	external-interface fe-0/0/0.0;    <- тут
	xauth access-profile dyn-vpn-access-profile;
}
```

Т.к остальной конфиг подозрений не вызывает, поправим это досадное упущение с интерфейсом:  
```
root@srx100h2# set security ike gateway dyn-vpn-local-gw external-interface fe-0/0/0.0
commit
```

Проверяем и действительно, теперь все заработало как надо.  

![Success connect Pulse](/assets/images/SRX100/success_connect_pulse.png)  

В URL указываете IP или hostname адрес внешнего интерфейса.  

Проверяем ike sa и ipsec sa:  

![ike/ipsec sa](/assets/images/SRX100/ike_ipsec_sa.png)  

## SHA1? AES128? серьезно?
Как можно заметить, алгоритмы у нас sha1 и aes-128-cbs, не густо для 2023...  
Забегая вперед, далее в конфиге мы установили все параметры вручную, но если вы обратитесь к документации, рекомендуемый proposal-set для pre-shared ключей следующий:  

> Другие proposal-set требуют сертификат вместо общего ключа, а в нашем случае поддерживается только общий ключ.

```
### IKE ###
Sec-level basic: preshared key, g1, des, sha1
Sec-level compatible: preshared key, g2, 3des, sha1
Sec-level standard: preshared key, g2, aes128, sha1
    
### IPSEC ###
Sec-level basic: esp, no pfs (if not configured) or groupx (if configured), des, sha1
Sec-level compatible: esp, no pfs (if not configured) or groupx (if configured), 3des, sha1
Sec-level standard: esp, g2 (if not configured) or groupx (if configured), aes128, sha1
```

А вот так выглядит кусок конфига с custom proposals, вместо proposal-set:  
```
### IKE ###
proposal ike-proposal {
    authentication-method pre-shared-keys;
    dh-group group2;
    authentication-algorithm sha1;
    encryption-algorithm aes-128-cbc;
    lifetime-seconds 28800;
}
policy ike-dyn-vpn-policy {
    mode aggressive;
    proposals ike-proposal;
    pre-shared-key ascii-text ## SECRET-DATA
}
gateway dyn-vpn-local-gw {
    ike-policy ike-dyn-vpn-policy;
    dynamic {
        hostname dynvpn;
        connections-limit 10;
        ike-user-type group-ike-id;
    }
    external-interface fe-0/0/0.0;
    xauth access-profile dyn-vpn-access-profile;
}

### IPSEC ###
proposal ipsec-proposal {
    protocol esp;
    authentication-algorithm hmac-sha1-96;
    encryption-algorithm aes-128-cbc;
    lifetime-seconds 3600;
}
policy ipsec-dyn-vpn-policy {
    proposals ipsec-proposal;
}
vpn dyn-vpn {
    ike {
        gateway dyn-vpn-local-gw;
        ipsec-policy ipsec-dyn-vpn-policy;
    }
}
```

Можем ли мы улучшить ситуацию? Совсем немного...  
После череды успешных и не очень экспериментов, получились следующие результаты:  

**IKE (Только IKEv1)**
- aes-256-cbs
- sha1
- dh group2

**IPSEC**
- esp
- aes-256-cbs
- hmac-sha1-96

Т.е нам доступно только повышение до AES-256-cbs, изменение остаьных параметров пиводит к неуспешному подключению. Поправим конфиг в соответствии с данными:  
```
set security ike proposals ike-proposal encryption-algorithm aes-256-cbc
set security ipsec proposals ipsec-proposal encryption-algorithm aes-256-cbc
commit
```

Наблюдаем aes-256-cbs, это максимум, на что способен SRX100H2.  

![ike-256](/assets/images/SRX100/ike_256.png)  

![ipsec-256](/assets/images/SRX100/ipsec_256.png)  

Ближайшая замена, рекомендуемая Juniper - [SRX300](https://apps.juniper.net/home/srx300/overview).   

У меня не было цели разбирать возможности железки целиком или настройку Dynamic VPN целиком. Моя конфигурация была стандартной. Оставлю пару ссылок по теме Dynamic VPN на SRX.  

Список источников: 
- [Описание proposal-set (Security IKE)](https://www.juniper.net/documentation/us/en/software/junos/vpn-ipsec/topics/ref/statement/security-edit-proposal-set-ike.html)
- [Описание Dynamic VPNs через Pulse Secure](https://www.juniper.net/documentation/us/en/software/junos/vpn-ipsec/topics/topic-map/security-dynamic-vpns-with-pulse-secure-clients.html#id-dynamic-vpn-overview__d86539e396)
- [Пример конфигурации Dynamic VPN](https://www.juniper.net/documentation/en_US/junos12.1x47/topics/example/vpn-security-dynamic-example-configuring.html)
- [Гайд по настройке Dynamic VPN на SRX](https://www.juniper.net/documentation/en_US/junos12.1x47/information-products/pathway-pages/security/security-vpn-dynamic.pdf)
- [Гайд по ролбэку на резервный конфиг](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade-evo/topics/topic-map/evo-backup-recover-configuration.html)

<p></p>
<hr>
<h2>Хочешь обсудить тему?</h2>
С вопросами, комментариями и/или замечаниями, приходи в [чат](https://t.me/netautomationarea) или подписывайся на [канал](https://t.me/netautomation).