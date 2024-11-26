---
layout: post
title: "Ковыряемся в Annet, часть 2, Исходники, методы, генераторы"
date: '2024/11/26|09:00'
categories: [Automation]
tags: [Python]
published: true
author: Artem Kovalchuk
---

<img src="https://woohung.github.io/assets/images/Annet/main_2.jpg">

- [Коммиты, коммиты](#commits)
- [Context Yaml-ович](#config)
- [Исходники](#source)
    - [Структура проекта](#scheme)
        - [gen](#gen)
        - [diff](#diff)
        - [patch](#patch)
        - [deploy](#deploy)
    - [Как же annet ходит на коробки!?](#gnetcli)
- [Превозмогая трудности, правим код](#edit-code)
- [Полезное](#useful)

Приветствую, друг!

На этом этапе у нас есть представление об annet и собранная лаба, все готово для погружения. ВСЕ НА ДНО!

![ДНО](/assets/images/Annet/sea.gif)

> Получилась еще одна объемная статья, пожалуйста, если вы ее дочитаете, дойдите с фитбэком в канал телеги (там будет соответствующий пост) и расскажите то, что посчитаете нужным. Тяжко ли читается? На каком моменте захотелось дропнуть? и т.п. Приятного чтения.

План на сегодня:

1. Смотрим, что накоммитили в annet за последние две недели;
3. Разбираем `context.yml`;
4. Смотрим структуру проекта;
5. Накидываем тестовый генератор;
6. Попробуем основной функционал annet: `gen/diff/patch/deploy`;
7. Опишем бегло какие фиксы были внесены, а подробнее о них, в следующих статьях.
8. ???
9. PROFIT!

# Коммиты, коммиты <a name="commits"></a>

За две недели, с момента, как я взялся писать про annet, ребята из Я. уже успели накоммитить полезного, пройдемся по наиболее заметным изменениям.

- Теперь минимальная версия Python - 3.10;
- В документацию добавили описания "языка" ACL и описание RPL (Routing policy list), как я понял, относится к конфигурированию BGP вендоронезависимо;
- Прикрутили валидацию запросов к Netbox по SSL, опция `insecure: bool = False` по умолчанию SSL проверять;
- пачка коммитов разной направленности в сторону фиксов по работе с различными блоками конфигов в рулбуках;
- появились [примеры лаб](https://github.com/annetutil/contribs/tree/main/labs) для первого знакомства с инстурментом.

Как вообще можно посмотреть, что и кто накоммитил? Для этого можно пойти на github и в нужное репе тыцнуть на `Commits`:

![Коммиты на github](/assets/images/Annet/commits.png)

Там можно найти и изучить все изменения (включая ролбэки):

![Коммиты на github, подробнее](/assets/images/Annet/commits_2.png)

Вот, например, коммит по добавлению RPL.  Слева будет список файлов в формате дерева, которые были затронуты коммитом, а в основном окне `+` будут обозначены добавленные/измененные строки, а `-` удаленные:

![Коммиты на github, подробнее про единичный](/assets/images/Annet/commits_3.png)

Утилита `git`, та, которой мы в прошлой статье делали `git clone` позволяет посмотреть  `git log`. Сам же `git log` не так прост, как кажется. По умолчанию, т.е без параметров, `git log` выглядит так:

![git log](/assets/images/Annet/git_log.png)

А вот так, если добавить немного магии:

```
git log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<`%an`>%Creset'
```

![Красивый git log](/assets/images/Annet/pp_git_log.png)

> В команде выше следует убрать одинарные кавычки в <`%an`> у меня этот кусок ломал весь .md файл по форматированию, они тут как костыль, не болле.

Важно в текущих коммитах, конечно - список лаб, которые теперь можно потыкать. И выглядят они сильно легче (в плане запуска и эксплуатации), чем предлагал я в предыдущей статье. Каждая лаба - сервис в докере. Единственное, что нужно сделать - достать образы в формате `.bin` для коробок внутри.
Для первой лабы предлагаемый в лабе образ я не нашел, но нашел [такой](https://disk.yandex.ru/d/_zLWan-p0oM8QA).
В остальном, достаточно следовать инструкциям в лабе.

> Образ стартует с `.bin`, но в настройках мапится `.image` т.е если образ не совпадает с тем, что предлагают ребята - нужно поменять текстовый файл. Например для `lab00` нужно поменять файл `lab.net` по пути `labs/topologies/lab00_basic_cisco/src/lab.net`, а `.bin` кладется по тому пути, который указан в лабе, тут без изменений.

Список лаб:

- [Labs](https://github.com/annetutil/contribs/tree/main/labs#labs)
    - Basic
        - [lab00. Cisco Base Scenario](https://github.com/annetutil/contribs/blob/main/labs/topologies/lab00_basic_cisco)
        - [lab01. FRR Base Scenario](https://github.com/annetutil/contribs/blob/main/labs/topologies/lab01_basic_frr)
    - DC
        - [lab10. Cisco DC Scenario](https://github.com/annetutil/contribs/blob/main/labs/topologies/lab10_dc_cisco)
        - [lab11. FRR DC Scenario](https://github.com/annetutil/contribs/blob/main/labs/topologies/lab11_dc_frr)
        - [lab12. Multivendor DC Scenario](https://github.com/annetutil/contribs/blob/main/labs/topologies/lab12_dc_arista_cisco_frr)

Вы можете их проделать самостоятельно, я протыкал самую первую, отлично работает (делал на macos) если что-то не получится, приносите в чат, разберем.

Возвращаясь к теме статьи, моя затея с разбором annet затевалась до появления лаб, поэтому я пойду своей дорогой. У нас будут свои сложности.

По части чтения исходников (что и как работает) весь код вычитывать в рамках одной статьи я не успею, да и зачем? Остановимся на общем представлении и некоторых особенностях работы.

Чтобы не повторять судьбу Винни-Пуха и не окунаться в болото с головой, помочим ножки в `context.yml`.

# Context Yaml-ович <a name="config"></a>

Сначала хорошо будет разобраться с `context.yml` что в нем есть и какова его цель. С этого файла все начинается. В нем мы указываем различные параметры для работы с annet. Вот так выглядит рабочий его вариант:

```yaml
fetcher:
  default:
    adapter: gnetcli
    params: &gnetcli_params
      dev_login: admin
      dev_password: admin123

deployer:
  default:
    adapter: gnetcli
    params:
      <<: *gnetcli_params

generators:
  default:
    - annet_generators.example

storage:
  default:
    adapter: file
    params:
      path: ./inventory.yml
  netbox:
    adapter: netbox
    params:
      url: http://192.168.2.126:8000
      token: 3b7d0083a93ee4064bae34f51927a906854d6631

context:
  default:
    fetcher: default
    deployer: default
    generators: default
    storage: netbox 

selected_context: default
```

Этот файл - или "контекст", в терминах annet, состоит из:

- fetcher
- deployer
- generators
- storages

*Fetcher* - отвечает за опрос устройств, используется когда мы используем diff, а так же deploy.

*Deployer* - дергается в момент, когда мы используем deploy и так же использует логику опроса устройств.

Как `adapter` в обоих случаях указывается некий `gnetcli` с параметрами:

```yaml
dev_login: admin
dev_password: admin123
```

Да, все верно, это логин пароль для наших устройств. Обязательно ли ему быть в открытом виде? Скорее всего нет, но у меня пока не хватит знаний ответить, как можно сюда прикрутить, например, Vault или dotenv.

> При вызове annet через CLI можно передать ключ `--ask-pass`

*Generators* - собирает список генераторов по указанному пути, в данном случае `annet_generators.example` там у нас лежит:

```
.
├── __init__.py
├── interface.py
├── lldp.py
└── vlan.py
```

`interface.py`,`lldp.py`, `vlan.py` - файлы с генераторами, а `__init__,py` собирает их вместе, если угодно. Вот как он выглядит:

```python
from typing import List

from annet.generators import BaseGenerator
from annet.storage import Storage

from . import lldp, vlan, interface


def get_generators(store: Storage) -> List[BaseGenerator]:
    return [
        lldp.Lldp(store),
        vlan.VlanGenerator(store),
        interface.IfaceMtu(store),
        interface.IfaceDescriptions(store),
        interface.IfaceAccessVlan(store),
    ]
```

`get_generators`мы затем дергаем в другом `__init__.py` который расположен в `annet/generators`. С помощью нее мы соберем все генераторы, которые мы объявили ранее.

> `__init__.py` может быть много

*Storages* - это то, откуда будет взята информация об устройствах. В нашем случае определен netbox, но, как выяснилось, можно прикрутить что угодно через entry_points, о них чуть дальше. Если угодно, это наше инвентори в терминах ансибла.

Вот и весь файл с настройками. В конце собираем все в один контекст:

```yaml
context:
  default:
    fetcher: default
    deployer: default
    generators: default
    storage: netbox 
```

Контекстов может быть несколько, между ними можно переключаться. А в сам контекст подставлять различные модули, изначально описывая их в файле как несколько разных, например, storages, где определен `file` и `netbox`.

Для выбора же контекста целиком есть строчка `selected_context: default`, а так же переменная окружения `ANN_SELECTED_CONTEXT`, чтобы переключать контекст "на лету".

Файл `context.yml` может быть определен в нескольких местах, но порядок, в котором его чекает annet, определен изначально:

- `ANN_CONTEXT_CONFIG_PATH` путь, который можно поместить в переменную окружения
- `~/.annet/context.yml` да, верно, в нашей домашней директории
- `annet/configs/context.yml` внутри самой annet

# Исходники <a name="source"></a>

Как же  вообще читать исходники? 

1. Хорошо бы понимать, что вообще делает программа, об этом была [часть 0](https://woohung.github.io/automation/2024/11/12/Ковыряемся-в-Annet-Часть-0-Обзорная.html);
2. Затем изучить всю доступную документацию, хотя бы по диагонали, чтобы понять основные возможности программы;
3. Уже после 1 и 2 пунктов изучить структуру;
4. Попробовать погрепать по коду и последить куда уходит детство...уходят наши функции.

> Всегда нужно следить за тем, чтобы не вчитываться каждую строчку, а найти общую логику и принципы. Невольно хочется это сделать т.к "как же так!? Я хочу разобраться"

## Структура проекта <a name="scheme"></a>

Что у annet со структурой? 

![Структура annet](/assets/images/Annet/structure.png)

В корне лежат различные вспомогательные вещи по типу README, .gitignore, requirements.txt и т.п.

В `annet_generators` складируются различные генераторы

В `test` очевидно, тесты (они тут на pytest)

В `docs` живет дока

А вот в `annet` находится вся основная логика программы:

![Структура annet, глубже](/assets/images/Annet/structure_2.png)

В `adapters` определяется логика подключаемых адаптеров по типу `netbox`, `fetcher` и т.п. Адаптеры, считай, плагины.

В `annlib` я не погружался, но там точно живет логика обработки `rulebooks` которые отвечают за логику работы с конфигами при использовании `gen/diff`.

В `api` разместили логику работы, например, `gen/diff,patch/deploy` и вспомогательных функций для них

В `generators` уложили логику по обработке генераторов

Я не уловил за что отвечает `mesh` но, как будто бы, без его понимания все работает.

`rpl` добавили совсем недавно, там кроется реализация обращения к BGP без привязки к вендору. Как мы можем выяснить историю конкретного файла? Для этого есть `git blame`:

![git blame на github](/assets/images/Annet/git_blame.png)

В CLI тоже это есть, но в менее наглядном варианте:

![git blame в cli](/assets/images/Annet/git_blame_1.png)

По идее его можно так же кастомизировать, но я в это не лез т.к пока крайне редко использую эту команду.

Ну и в `rulebooks` хранится логика по работе с конфигами разных вендоров. В `annlib/rulbooks` был обработчик, а тут сама логика.

Вот и вся структура, если коротко. Перейдем к частностям.

Пойдем в лоб и начнем с `annet.py` и дальше двинем вглубь, по ходу дела записывая что и за что может отвечать:

```python
#!/usr/bin/env python3

import annet.annet

annet.annet.main()
```

Я пишу код в nvim-е, а точнее в astronvim т.к в какой-то момент я писал в vim и собирал `.vimrc` под себя, потом переехал на nvim вместе с этим файлом, а после узнал, что нынче модно собирать на lua и вообще там больше интересного, но у меня не было времени изучать lua, поэтому я прост нашел сборку, которая меня утроила. Отвлекся. 

В astronvim есть два инструмента для быстрого передвижения по коду - telescope и gd (go to definition).

Вот так выглядит telescope, с помощью него и live grep можно найти в каких файлах используется функция `main()`:

![Поиск в telescope](/assets/images/Annet/telescoope_nvim.png)

То же самое можно сделать с помощью хоткея `gd`. В этом случае меня сразу кинет в файл, где присутствует данная функция:

![go to definition](/assets/images/Annet/gd.png)

Это условная стартовая точка программы, в ней принимаются аргументы `show/gen/diff/patch/deploy/file-diff/file-patch/context` и далее дергается та или иная логика, прописанная для каждого из аругментов. Сначала обратимся к странице в доке, которую я сознательно пропустил в самом начале - [CLI Usage](https://annetutil.github.io/annet/main/usage/cli.html#).

В ней описаны взаимодействия с `gen/diff/patch/deploy` пройдемся катком по каждому из них, с живыми примерами.

Annet не умеет работать с netbox выше 3.7, поэтому по пути мне пришлось подпереть костылями то необходимое, чтобы оно завелось на версии 4.x. Как это сделано, расскажу в конце, а пока сосредоточимся на основном функционале.

### gen <a name="gen"></a>

`gen` гхм, генерит конфигурацию, ни больше ни меньше. Как он это делает? Берет инфу об устройстве из стораджа на вход, прогоняет через генераторы и возвращает конфигруацию. Выглядит как-то так:

![gen](/assets/images/Annet/gen.png)

С помощью ключа `--log-level DEBUG` можно посмотреть что делается под капотом:

![gen debug](/assets/images/Annet/gen_debug.png)

Сначала идет секция по обращению к netbox, затем дергается `/annet/annet/generators/__init__.py` где уже реализована логика прогона частичного (Partial) конфига, сейчас мой генератор использует именно его.

Сперва, как я понял, подкидывается `InitialConfig`, который не подходит к моей коробке. А дальше уже чекается мой генератор `IfaceAccessVlan`. 

> `InitialConfig` используется для коробок, которые никогда ранее не настраивались т.е запускаются впервые (Zero Touch).

В Annet можно генерить либо частичные конфиги, либо полные. Partial и Entire соответственно.

Partial наследуется от класса `PartialGenerator` а Entire от класса `Entire`.

> Просто примем за факт, что в питоне есть ООП и наследование. В annet код очень часто использует обе этих концепции. Разберемся с ними в других статьях, я пока и сам слаб в этом.

#### Разбор генератора <a name="generator"></a>

Предлагаю сразу, пока мозги еще не утекли, окунуться в генераторы и понять, на основе чего будет работать вся остальная логика.

Генераторы лежат в `/annet/annet_generators` и путь до них указывается в файле `context.yml`.

В самой команде `./annet.py gen -g access msk-swd-01` ключ `-g` означает использовать конкретный генератор, а дальше указывается *тег* генератора, в данном случае `access`. Вот так выглядит мой генератор:

```python
VLAN = 1

class IfaceAccessVlan(PartialGenerator):
    """Partial generator class of access VLAN on interfaces"""

    TAGS = ["access"]

    def acl_cisco(self, _: Device):
        """ACL for Cisco devices"""
        return f"""
        interface */((LoopBack|Eth-Trunk|.*GE[^.]*|static|.*Ether[^.]*)[^.]\\d*$)/
            switchport access vlan
        """

    def run_cisco(self, device: Device):
        """Generator for Cisco devices"""
        for interface in device.interfaces:
            vlan_id = interface.custom_fields.get("vlan_id")
            if vlan_id:
                vid: int = vlan_id.get("vid", VLAN) 
            else:
                vid = VLAN
            if "Gi" in interface.name:
                with self.block(f"interface {interface.name}"):
                    yield f"switchport access vlan {vid}"
```

TAGS = ["access"] - и есть ключевое слово с тегом в команде. В `acl_cisco` прописывается ACL, т.е та часть конфига, которая должна подпадать под написанное условие (регулярки, как видите, поддерживаются), если конфиг, который будет сгенерировать в функции `run_cisco` не подпадет под ACL - код прекратит выполнение и вывалится с исключением:

```bash
ERROR MainProcess - /Users/woo_hung/python_projects/clear_annet/annet/annet/generators/__init__.py:239 - ACL error: generator is not allowed to yield this command: fail interface GigabitEthernet0/0 -- host='msk-swd-01' generator='annet_generators.example.interface.[IfaceAccessVlan]'
```

Тут я добавил перед `interface` непредсказуемое слово `fail` в функции `run_cisco`:

```python
with self.block(f"fail interface {interface.name}"):
				yield f"switchport access vlan {vid}"
```

Еще один интересный момент, наша функция `run_cisco` - функция-генератор, об этом говорит наличие оператора `yield`.

Если коротко, то обычная функция так или иначе возвращает что-то одно и завершает свою работу. Будь то return None, return <что-то> или исключение, функция все равно завершит свою работу и все, что было внутри нее - исчезнет. 

А каждый раз, когда внутри функции встречается `yield`, генератор приостанавливается и возвращает значение. При следующем запросе, генератор начинает работать с того же места, где он завершил работу в прошлый раз.

> [Примеры работы](https://advpyneng.readthedocs.io/ru/latest/book/14_generators/generator.html) `yield` и чуть более детальную теорию можно посмотреть в книге adv-pyneng.

Тяжко? А мы только `gen` посмотрели...

> Изучать чужой код - сложная задача. Тот факт, что вам сложно - **нормально**. 

Логичный вопрос - а как, черт возьми, acl_cisco связано с run_cisco? (а оно связано, иначе исключения выше мы бы не получали)

Снова все дело в наследовании и классах. Мой класс генератора наследуется от `PartialGenerator`, а в нем заложена интересная логика.

Наследование там примерно такое: IfaceAccessVlan -> PartialGenerator -> TreeGenerator -> BaseGenerator...

Душим дальше..."где-то там" это где? Логично предположить, что если есть методы в классе, то где-то должны быть экземпляры этого класса, с которыми совершается логика по сверке того что, предлагается генерировать, на сверку с написанным ACL.

Вооружившись fzf и telescope (а хорошо бы еще и ведьмачьим чутьем) идем исследовать.

Проваливаемся в корень через nvim и тычем в telescope название нашего класса `PartialGenerator`. Я просто искал визуально похожу логику, которая могла бы отвечать за эту часть:

![Погружаемся поиском в код](/assets/images/Annet/lookup_1.png)

И находим в `annet/generators/__init__.py` (опять этот инит!):

![Погружаемся глубже](/assets/images/Annet/lookup_2.png)

Нашли функцию `run_partial_generators` которая внутри себя дергает `_run_partial_generators` в которое и реализована данная логика.

Побудем немного процессом внутри роутинг таблицы и провернем еще один lookup через telescope, но уже для функции `run_partial_generators`:

![и глубже...](/assets/images/Annet/lookup_3.png)

Вот мы уже дошли до непосредственно `gen.py`. На этом предалагаю пока остановиться т.к, в целом, еще пару итераций и десяток часов изучения мы таки доберемся до момента, где и какую функцию, метод, декоратор мы используем...

![Brain fuck...](/assets/images/Annet/oh.png)

Но нам пора к следующему методу - `diff`. С ними будет легче т.к основные штуки мы рассмотрели в контексте `gen`.

### diff <a name="diff"></a>

Выдохнули! Разбавим картинками и ненапряжным комментированием картинок с выводами.

В команде ничего не поменялось кроме метода с `get` на `diff`. Уже видим отличие, появились `+` аля как в гите. Произошло тут несколько вещей:

1. Сходили в netbox;
2. Спросили генератор;
3. Сгенерировали конфиг;
4. Сходили на устройство;
5. Сроверили часть runing config с интерфейсами;
6. Сычислили diff;
7. Сернули вывод что будет убрано, что будет добавлено.

![diff](/assets/images/Annet/diff.png)

Пока что одни плюсы...все потому, что на моей коробке ничего не настроено. Я проверну за кулисами `deploy`и покажу разницу.

*проворачивает deploy....*

Видим, что gi0/0 из вывода пропал...

![diff на примененный deploy](/assets/images/Annet/diff_1.png)

Тут ничего страшного, просто изменения доехали и теперь annet понимает, что для этого интерфейса ей менять нечего. Почему для других интерфейсов все еще есть vlan 1? Потому что, думается мне, не отловлен момент, что vlan 1 в конфиге не отображается.

Поменяем в netbox vlan на gi0/0 и увидим, что `diff` снова видит gi0/0:

![diff с отличиями](/assets/images/Annet/diff_2.png)

Annet любезно снесет наш vlan 50 и добавит vlan 60. Важно понимать, если влана еще нет в конфиге, то его нужно создать и запушить заранее отдельным генератором. Сделаем это. 

Вланы я создаю в netbox тут:

![VLAN в netbox](/assets/images/Annet/vlan_nb.png)

Затем в custom_fields делаю связку VLAN -> Interfaces, чтобы иметь возможность прицепить на интерфейс vlan, аля acces vlan, о trunk тут пока речи не идет:

![доп. поле VLAN в Interfaces](/assets/images/Annet/custom_fields.png)

А тут вешаю VLAN на интерфейс:

![Применение VLAN на интерфейсе](/assets/images/Annet/vlan_on_intf.png)

Сейчас я повесил вланы на еще два интерфейса, посмотрим, как будет выглядеть diff для генератора вланов:

![diff для vlan](/assets/images/Annet/diff_3.png)

*проворачивает deploy....*

После деплоя видим, что вланы на месте:

![show vlan с коробки](/assets/images/Annet/show_vlan.png)

Самое время сделать диф для switchport access:

![diff на измененные интерфейсы с вланами](/assets/images/Annet/diff_4.png)

Все работает!

А вот генератор для vlan, так же основанный на `custom_field` netbox, который автоматически выцепит все вланы, созданные на интерфейсах конкретного девайс:

```python
from annet.generators import PartialGenerator
from annet.storage import Device

VLAN = 1

class VlanGenerator(PartialGenerator):
    TAGS = ["vlan"]

    def acl_cisco(self, device: Device):
        return """
        vlan
        """

    def run_cisco(self, device: Device):
        for interface in device.interfaces:
            vlan_id = interface.custom_fields.get("vlan_id")
            if vlan_id:
                vid: int = vlan_id.get("vid", VLAN) 
                yield f"vlan {vid}"
            else:
                vid = VLAN
```

В логику работы `diff` я не погружался, она определяется в руллбуках, а если точнее `rulebook/texts/VENDOR`. Оставим на потом.

Подитожим, что делает дифф? Делает `gen` и затем вычисляет дифф для собранного с устройства конфига.

Двинули к `patch`.

### patch <a name="patch"></a>

`patch` продолжает дело `diff`, т.е принимает результат `diff` на вход и возвращает список команд, которые поедут на устройство.

![patch](/assets/images/Annet/patch.png)

Хитрость тут в том, что для разных вендоров - разная логика работы с командами. Т.е в Cisco это `no` перед командой, в Huawei это `undo` перед командой и т.д. Все это описывается так же в `rulebook/texts/VENDOR`. В документации наглядно [описан](https://annetutil.github.io/annet/main/usage/cli.html#annet-patch) пример с acl на huawei.

> В логику работы roolbook я не лез ибо не нашлось достаточно времени, но будет интересно доразобраться уже после nexthop-а.
> 
> Если захотите разобраться сами, вот вам для стартовой точки - при вызове, например, `diff` можно дать ключ `--show-rule` и получить в выводе кусочек правила, что отвечал за генерацию того или иного куска:
> 
> ![Примененные rule](/assets/images/Annet/ruleset.png)

Итог работы `patch` - сделать `diff` и предоставить итоговый список команд для дальнейшего деплоя.

Поехали, наконец, к `deploy`.

### deploy <a name="deploy"></a>

Последняя остановка - `deploy`. Метод является вершиной карточного домика, а именно - дергает `patch`, а затем льет изменения на устройство. Развернем логику. Чтобы залить желаемый сгенерированный конфиг на устрйоство, нам нужно:

1. Вызвать `deploy`;
2. Деплою нужен итог работы `patch`, вызываем `patch`;
3. Патчу нужен итог работы `diff`, вызываем `diff`;
4. Дифу нужен сгенеренный конфиг, вызываем `gen`;
5. `gen` дергает непосредственно нужный генератор;
6. Возвращаемся наверх и отправляем `patch` на устройство.
7. PROFIT!

Что интересно, `deploy` не рвется по умолчанию сразу лить все на коробки, он сначала спросить ВАС и реализовано это в виде отдельного окна, где будет отражен `diff` и, если жмякнуть букву A, то `patch`. От вас требуется провалидировать и прожать либо Y, чтоы да, либо q, чтобы нет:

![Валидирующий этап deploy](/assets/images/Annet/deploy_1.png)

Вот и все. Никакого вывода в ответ не будет (это вам не ансибл!):

![sh run с коробки](/assets/images/Annet/sh_run_intf.png)

Можно дать ключ `--no-ask-deploy` если вы очень смелый! и тогда `deploy` сразу ломанется на коробку, а дальше как карта ляжет.

Так, окей, а если несколько коробок? Можно же передать несколько хостов? Можно!

![deploy на несколько коробок](/assets/images/Annet/deploy_2.png)

На месте вот этих строчек, предполагаю, мог бы быть некий прогресс бар заливки, но...его нет, а я дальше не копал (:

![Вывод deploy](/assets/images/Annet/out_deploy.png)

ФСЕ! Мы с вами прочитали сегодня ДОХЕРА БУКВ! Мы молодцы. Но это еще не конец...

![Сколько можно...](/assets/images/Annet/what_the.png)

## Как же annet ходит на коробки!? <a name="gnetcli"></a>

Вообще исходники annet это не только сама annet, но и целиком [annetutil](https://github.com/annetutil) где присутствуют: `gnetcli_adapter`, `annetbox`, `gnetcli` и `annet`.

Самое интересное тут - как же annet ходит на коробки? Делает она это с помощью gnetcli, который написан на Go тем же Яндексом.

Вот только вы не найдете в коде annet упоминание об импорте gnetcli_adapter. Как так? Я тоже задавал себе такой вопрос (и в доке даже лежит ответ, но я слепой и не увидел), но я дошел до гражданина @gescheit, одного из участников создания annet, на что мне был дан ответ - Entry Point. 

Что такое Entry Point?

Заваривай чаю, милый друг...это разговор еще на десять тыщ символов. Шучу. Попробую покороче, впереди еще баги разбирать...

Entry points позволяют пакетам и библиотекам находить и загружать код (например, классы, функции, или модули) из других пакетов.

Когда система (в данном случае, annet) ищет подключаемый компонент для своего интерфейса connectors.fetcher, она проверяет, что указано в entry points установленных пакетов.

> Начиная с python 3.10 этот функционал является частью стандартной библиотеки.

Как же их найти в коде? В Annet, как мы выяснили при рассмотрении `context.yml`, за обращение к устройствам отвечает некий Фетчер, а за раскатку драйвер деплоер.

Оба используют get_all() для поиска коннекторов

```python
def get_fetcher() -> Fetcher:
    connectors = fetcher_connector.get_all()
    fetcher, _ = get_connector_from_config("fetcher", connectors)
    return fetcher
```

get_all отсылает нас к `_entry_point`:

```python
def get_all(self) -> List[T]:
    if self._classes is None:
        self._classes = self._entry_point or [self._get_default()]

    return self._classes.copy()
```

В классе `Connector` есть метод _entry_point:

```python
def _entry_point(self) -> List[Type[T]]:
	ep = load_entry_point(self.ep_group, self.ep_name)
	if self.ep_by_group_only:
		ep.extend(load_entry_point_new(self.ep_by_group_only))
        return ep
```

Что же тут происходит? При установке пакетов с использованием функционала entry_point (далее EP) пакет может содержать `pyproject.toml` или `setup.py` или `setup.cfg`  в которых прописаны эти самые EP, вот [пример](https://github.com/annetutil/gnetcli_adapter/blob/main/pyproject.toml#L30) в `gnetcli_adapter`.

В чем смысл, например, вот этого?:

```toml
[project.entry-points."annet.connectors.fetcher"]
gnetcli = "gnetcli_adapter.gnetcli_adapter:GnetcliFetcher"
```

В вашем случае [project.entry-points."annet.connectors.fetcher"] сообщает annet, что она может использовать класс GnetcliFetcher из модуля gnetcli_adapter.gnetcli_adapter как реализацию fetcher.

Как annet узнает о них? Воспользуемся все тем же поиском и немножко документацией по entry_point. Из документации нам важно выцепить вот что: 

> The recommended approach for loading and importing entry points is the [`importlib.metadata`](https://docs.python.org/3.11/library/importlib.metadata.html#module-importlib.metadata "(in Python v3.11)") module, which is a part of the standard library since Python 3.8 and is non-provisional since Python 3.10.

Те в поиске нужно искать импорты из [importlib.metadata](https://docs.python.org/3.11/library/importlib.metadata.html#module-importlib.metadata) и да, вот он, один единственный:

![Импорт EP](/assets/images/Annet/import_ep.png)

> `importlib.metadata` operates on third-party _distribution packages_ installed into Python’s `site-packages`directory via tools such as [pip](https://pypi.org/project/pip/). Specifically, it works with distributions with discoverable `dist-info`

И действительно, для пакета `gnetcli_adapter` есть `~/venv/test_annet/lib/python3.12/site-packages/gnetcli_adapter-1.0.9.dist-info/entry_points.txt`, где лежат наши EP:

```
[annet.connectors.deployer]
gnetcli=gnetcli_adapter.gnetcli_adapter:GnetcliDeployer

[annet.connectors.fetcher]
gnetcli=gnetcli_adapter.gnetcli_adapter:GnetcliFetcher
```

В `annet/connectors.py` и стартует логика по подключению `gnetcli` и дальнейшая работа самой библиотеки `gentcli_adapter`, которая, по факту, является для annet плагином и ей вовсе не обязательно знать о ней заранее.

> entry_points.txt создаётся автоматически при установке пакета  через pip

Так же когда мы в `context.yml` указывает `adapter: gnetcli` (да, `gnetcli` указывается точно так же, как он есть в `entry_points.txt`).

На этом мы почти закончили.

Я бы мог пройтись по методам `show` и `context`, но уже и так слишком много всего...оставлю на попозже. Сейчас быстренько пройдемся по багам и расход...

# Превозмогая трудности, правим код <a name="edit-code"></a>

Исправлений пришлось делать не так много, но кое-чего подкрутить пришлось. Первое, конечно же, версия нетбокса.

> Удаленные строки буду показывать `# -`, добавленные `# +`

Меняем в проверке версии нетбокса 3-ку, на 4-ку иначе annet вообще не стартует:

```python
@ annet/adapters/netbox/provider.py:23 @ def storage_factory(opts: NetboxStorageOpts) -> Storage:
            # old version do not support status reqeust
            return NetboxStorageV24(opts)
        raise
    if status.netbox_version.startswith("3."): # -
    if status.netbox_version.startswith("4."): # +
        return NetboxStorageV37(opts)
    else:
        raise ValueError(f"Unsupported version: {status.netbox_version}")
```

В 4-ке поменялось отображение device_role, теперь оно просто role на что, естественно, ругнется нетбокс из за несоответствия с моделями. Запилил отдельный класс для DeviceRole, который "обернут" декоратором `@dataclass`:

```python
@ annet/adapters/netbox/common/models.py:39 @ class DeviceType:

@dataclass # +
class DeviceRole: # +
    id: int # +
    url: str # +
```

> Декораторы в питоне отдельная большая и интересная тема, здесь мы их подробно не рассматриваем, постараюсь осветить в будущем.

Затем прикрутил его в модель NetboxDevice:

```python
@ annet/adapters/netbox/common/models.py:171 @ class NetboxDevice(Entity):
    display: str
    device_type: DeviceType
    device_role: Entity # -
    role: DeviceRole # +
    tenant: Optional[Entity]
    platform: Optional[Entity]
    serial: str
```

Очередное изменение, в 4-ке ip-family стал словарем (а был обычным int-ом), добавил по аналогии с DeviceRole класс IPFamily и прикрутил его вместо `int`:

```python
@dataclass # +
class IpFamily: # +
    value: int # +
    label: str # +

@dataclass
class DeviceIp(DumpableView):
    id: int
    display: str
    address: str
    family: int # -
    family: IpFamily # +

```

То же, что и выше, сделать в annetbox по пути `~/venv/test_annet/lib/python3.12/site-packages/annetbox/v37/models.py`

В Я. юзают FQDN в именах устройств, поэтому annet подпихивает точку в имя устройства, нам оно тут нинада...:

```python
@ annet/adapters/netbox/v37/storage.py:268 @ def _hostname_dot_hack(netbox_query: NetboxQuery) -> NetboxQuery:
    # so we would not receive devices with a common name prefix
    def add_dot(raw_query: Any) -> Any:
        if isinstance(raw_query, str) and "." not in raw_query:
            raw_query = raw_query + "." # -
            raw_query = raw_query # +
        return raw_query

    raw_query = netbox_query.query
```


Что касается собственных генераторов, созданные классы нужно добавить в  `__init__`, который лежит `annet_generators/example/`, там же лежат мои генераторы:

```python
@ annet_generators/example/__init__.py:6 @ from annet.generators import BaseGenerator
from annet.storage import Storage

from . import lldp, vlan, interface


def get_generators(store: Storage) -> List[BaseGenerator]:
    return [
        lldp.Lldp(store),
        vlan.VlanGenerator(store),
        interface.IfaceMtu(store),
        interface.IfaceDescriptions(store),
        interface.IfaceAccessVlan(store),
    ]
```

Генератор у меня с `custom_field` поэтому пришлось поправить и это...:

```python
@ annet/storage.py:107 @ class Interface(Protocol):
    def add_addr(self, address_mask: str, vrf: Optional[str]) -> None:
        raise NotImplementedError

    # @property
    @abc.abstractmethod # +
    def custom_fields(self) -> Dict[str, Any]: # +
        """Custom fields from NetBox.""" # +
        raise NotImplementedError # +

@ annet/adapters/netbox/common/models.py:136 @ class Interface(Entity):
    lag: Entity | None = None
    lag_min_links: int | None = None

    custom_fields: Dict[str, Any] = field(default_factory=dict) # +

    def add_addr(self, address_mask: str, vrf: str | None) -> None:
        addr = ip_interface(address_mask)
        if vrf is None:
```
А еще потом тесты сломались...пришлось докинуть `custom_field` в FakeInterface, который наследуется от Interface:

```python
@ tests/annet/test_mesh/fakes.py:2 @
from typing import Any, Optional, Sequence
from typing import Any, Dict, Optional, Sequence # +
import abc # +

from annet.mesh.executor import Device
from annet.storage import Storage, Interface
@ tests/annet/test_mesh/fakes.py:22 @ class FakeInterface(Interface):
    def add_addr(self, address_mask: str, vrf: Optional[str]) -> None:
        self.addrs.append((address_mask, vrf))

    def custom_fields(self): # +
        pass # +
```
А это фикс бага в `file-diff/file-patch`:

```python
def _read_device_config(path, hw):
    _logger = get_logger()
    _logger.debug("Reading %r ...", path)
    score = 1

    with open(path.split(",")[0]) as cfgdump_file: # +-
        text = cfgdump_file.read()
    try:
        if not hw:
            hw, score = guess_hw(text)
        config = tabparser.parse_to_tree(
            text=text,
            splitter=tabparser.make_formatter(hw).split,
        )
        return config, hw, score
    except tabparser.ParserError:
        _logger.exception("Parser error: %r", path)
        raise
```
Так же пришлось править путь до gnetcli_server по пути `~/venv/test_annet/lib/python3.12/site-packages/gnetcli_adapter/gnetcli_adapter.py`

```bash
DEFAULT_GNETCLI_SERVER_PATH = "/Users/woo_hung/go/bin/gnetcli_server"
```

А так же пришлось установить gnetcli_server как пакет go:

```bash
go install github.com/annetutil/gnetcli/cmd/gnetcli_server@latest
```

Вроде бы на этом всё. Я бы выложил форк с исправлениями, да только это все костыли и проще обождать, пока допилят поддержку нетбокса 4.X.

Вы можете легко вгрузить изменения в файлы и оно будет работать, если будут вопросы, велком в комменты.

> Если дочитали до конца - вы герой! Спасибо. Мне было весьма полезным разобраться во всем вышенаписанном, надеюсь и вам что-то пригодилось.

Все, можно идти дышать свежим воздухом.

Я тут хотел еще продолж....

![end...now](/assets/images/Annet/get-out-of-here-throw.gif)

# Полезное <a name="usefull"></a>

- [Репозиторий annet](https://github.com/annetutil/annet)
- [Документация annet](https://annetutil.github.io/annet/main/index.html)
- [Лабораторные по annet](https://github.com/annetutil/contribs/tree/main/labs)
- [Репозиторий annetutil](https://github.com/annetutil)
- [Документация по entry point](https://setuptools.pypa.io/en/latest/userguide/entry_point.html)
- [PDF с cheat-sheet по annet](https://github.com/annetutil/contribs/blob/main/annet-cheat-sheet.pdf)

<p></p>
<hr>
<h2>Хочешь обсудить тему?</h2>
С вопросами, комментариями и/или замечаниями, приходи в [чат](https://t.me/netautomationarea) или подписывайся на [Telegram-канал](https://t.me/+Jeoaxn2kby4zMWUy).

