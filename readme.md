## Мониторим события PortSecurity коммутаторов Cisco в Zabbix

> *В качестве документации - статья на [Хабре](https://habr.com/en/post/481658/)*   
> *update:добавлена поддержка трапа cpsTrunkSecureMacAddrViolation (iso.3.6.1.4.1.9.9.315.0.0.2)*

В этой статье речь пойдет о мониторинге событий стандартного (для многих вендоров) механизма защиты от несанкционированного подключения устройств к сети, - механизма *PortSecurity*.  

Решение изначально построено для коммутаторов от компании *Cisco*, но при желании легко допиливается под любой коммутатор и под любые события, основанные на SNMP-трапах.

Если интересно, добро пожаловать под кут...
<cut>
Краткий экскурс о чем вообще речь.  

Технология *PortSecurity* работает на основе мак-адресов. Порт доступа коммутатора изучает заданное количество (по умолчанию один) маков для входящего трафика и при появлении нового мак-адреса активизирует защиту сети, блокируя порт. Так же коммутатор может послать *SNMP*-трап на хост указанный в настройках интерфейса.  

Режим блокировки бывает трех типов:

1. *shutdown* - выключение порта + *snmp-trap*
2. *restrict* - ограничении входящего трафика с неизвестного мак-а + *snmp-trap*
3. *protected* - ограничении входящего трафика с неизвестного мак-а молча, без *trap*-а. 

В более-менее крупной сети события *PortSecurity* происходят постоянно и поэтому их весьма полезно мониторить. Система мониторинга, как следует из заголовка - *Zabbix*.  

Поддержка трапов в *Zabbix* вроде как есть, но пользоваться этим я так и не научился. В итоге сделал свое решение, которое меня полностью устраивает. Собственно все решение - это достаточно простой скрипт-обработчик (*trap handler*) для пары конкретных *SNMP*-трапов.  Обработчик написан конечно же на *python* и вызывается стандартным демоном *snmptrapd*. Код обработчика выложен на [github](https://github.com/IDQDD/zabbix-recipes/tree/master/cisco-errdisable-traphandler).

Краткий теоретический экскурс закончен, переходим к конкретике.

Механизм мониторинга выстроен и работает по следующей цепочке:  
##### [1. Коммутатор cisco] → [2. демон snmptrapd] → [3. the script] → [4. Zabbix] 

В такой же последовательности и пойдет дальнейшее повествование  

#### 1. Cisco

На коммутаторах настраиваем *host* который будет принимать трапы и запускать скрипт

```bash
snmp-server enable traps port-security
snmp-server enable traps errdisable
snmp-server host 10.1.0.1 version 2c public
```

Тут же, отдельно для каждого порта настраиваем *PortSecurity*.  

В примере ниже *PortSecurity* настроен в режиме для гибридного порта *компьютер+телефон*. Поэтому указано максимальное количество маков равное двум. Режим блокировки *restricted*

```bash
 switchport port-security maximum 2
 switchport port-security
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 1111.11co.ffee vlan access
 switchport port-security mac-address sticky 0000.0000.beef vlan voice
```

Еще на коммутаторе будет полезным включить механизм *autorecovery*, для того чтобы при исчезновении причины блокировки блокировка лечилась сама без стороннего вмешательства. В случае, если причина не исчезла, после каждой попытки автовосстановления будет посылаться новый трап.

```bash
errdisable recovery cause psecure-violation
errdisable recovery interval 300
```

На этом про коммутатор все.  

Подробней как настраивать PortSecutity можно прочитать по ссылкам 
* [Cisco IOS Software Configuration Guide: Configuring Port Security](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/12-2/25ew/configuration/guide/conf/port_sec.html)  
* [Port security на xgu.ru](http://xgu.ru/wiki/Port_security)


#### 2. snmptrapd

*snmptrapd* - это стандартный сервис для обработки *snmp*-трапов. В Ubuntu ставится командой 
```bash 
> sudo apt install snmptrapd
```
конфигурация настраивается в файле */etc/snmp/snmptrapd.conf*

Каждый тип трапа уникален как для разных вендоров так и разных типов блокировки (restricted|shutdown).  

Мы сосредоточимся на двух конкретных:

1. CISCO-ERR-DISABLE-MIB::cErrDisableInterfaceEvent (1.3.6.1.4.1.9.9.548.0.1.1) - трап посылаемый в режиме shutdown
2. ciscoPortSecurityMIB::cpsSecureMacAddrViolation (1.3.6.1.4.1.9.9.315.0.0.1) - трап посылаемый в режиме restrict

В конфигурационном файле демона snmptrapd /etc/snmp/snmptrapd.conf пропишем такие строки:
```bash
authCommunity   log,execute,net public
traphandle .1.3.6.1.4.1.9.9.315.0.0.1 /etc/zabbix/externalscripts/traphandlers/cisco-psec-traphandler.py
traphandle .1.3.6.1.4.1.9.9.548.0.1.1 /etc/zabbix/externalscripts/traphandlers/cisco-psec-traphandler.py
```

Далеее, для краткости, эти трапы буду называть по номерами **315** и **548**.

#### 3. the script

Здесь я буду последовательно описывать логику написания скрипта, руководствуясь которой можно будет по аналогии писать другие обработчики (трапхэндлеры) для других видов трапов и/или устройств. Кому не сильно интересно что творится под капотом, тот может сразу переходить в следующую главу. Правда, возможно, предварительно имеет смысл немного пробежаться по этому разделу с тем чтобы понимать зачем нужен файл конфигурации скрипта *config.ini*.

Кстати, наверное, c сonfig.ini и начнем. Для простоты сразу приведу его содержимое
```bash
[snmp]
community = public

[api]
zabbix_url = https://zabbix.acme.loc
zabbix_user = api_ro
zabbix_passwd = neskazhu

[zabbix]
server = 10.1.1.1
port = 10051
zabbix_sender = /usr/bin/zabbix_sender

#the predefined keyname of an item that has to be created for a given host in Zabbix
trapkeyname_disable = ErrDisable
trapkeyname_restrict = ErrRestrict

[logging]
logfile = /var/log/cisco-errdisable-traphandler.log
loglevel = INFO
```
Как мне кажется конфиг достаточно прозрачен для понимания. Здесь мы задаем snmp-community, уровень логирования и параметры доступа к серверу *Zabbix* через *Zabbix_API* и *zabbix_sender*.  

Единственный, возможно непонятный момент - это параметры  *trapkeyname_disable* и *trapkeyname_restrict*.  

Так вот эти параметры соответствуют трапам **548** и **315** и определяют имена ключей для *Items* (элементов данных) в самом *Zabbix*.

Идем дальше. Как было сказано выше, наш обработчик принимает на вход два вида трапов: **315** и **548**.  

В коде они различаются вот таким элементарным условием:
```python
if "548.0.1.1" in trapstr:
    mode = "disable"
    trapkeyname = "trapkeyname_disable"

elif "315.0.0.1" in trapstr:
    mode = "restrict"
    trapkeyname = "trapkeyname_restrict"

else:
    logging.error("Unknown trap. Discarding ...")
    exit(1)
```

Далее скрипт обращается в *Zabbix* используя *Zabbix_API* для того, чтобы по имени или адресу коммутатора узнать мониторим ли мы в принципе трапы от этого коммутатора. Здесь используется модуль *ZabbixAPI* и пару методов *host.get* и *item.get*. Ничего сложного.

Переходим к парсингу трапов.  

Интересно, что структура наших двух трапов кардинально различается. Вот смотрите  
Пример трапа 548:
```bash
switch-20
UDP: [0.0.0.0]->[192.168.99.20]:-2039
DISMAN-EVENT-MIB::sysUpTimeInstance 338:5:51:38.08
SNMPv2-MIB::snmpTrapOID.0 CISCO-ERR-DISABLE-MIB::cErrDisableInterfaceEvent
cErrDisableIfStatusCause.10640.0 9
```

А это типовой трап 315:  
```bash
switch-27
UDP: [0.0.0.0]->[192.168.99.27]:-13209
DISMAN-EVENT-MIB::sysUpTimeInstance 342:22:17:16.63
SNMPv2-MIB::snmpTrapOID.0 CISCO-PORT-SECURITY-MIB::cpsSecureMacAddrViolation
IF-MIB::ifIndex.10028 10028
IF-MIB::ifName.10028 FastEthernet0/28
CISCO-PORT-SECURITY-MIB::cpsIfSecureLastMacAddress.10028 0:22:55:88:ee:dd
```
Кстати, приведенные выше трапы я изобразил в человекочитаемом формате. На самом деле на вход скрипта трапы попадают в формате *ASN.1*, который выглядит совсем по другому. И именно с этим сырым форматом мы и будем работать.

Вот так те же самые трапы выглядят в сыром виде:

Трап 548:
```bash
iso.3.6.1.2.1.1.3.0 21:19:06.72
iso.3.6.1.6.3.1.1.4.1.0 iso.3.6.1.4.1.9.9.548.0.1.1
iso.3.6.1.4.1.9.9.548.1.3.1.1.2.10640.0 9
```

Трап 315:
```bash
iso.3.6.1.2.1.1.3.0 342:22:17:16.63
iso.3.6.1.6.3.1.1.4.1.0 iso.3.6.1.4.1.9.9.315.0.0.1
iso.3.6.1.2.1.2.2.1.1.10028 10028
iso.3.6.1.2.1.31.1.1.1.1.10028 "FastEthernet0/28"
iso.3.6.1.4.1.9.9.315.1.2.1.1.10.10028 "00 22 55 88 EE DD "
```

Формат *ASN.1*, хоть и относится к структурированным, но работать с ним далеко не так удобно как с *json* или *xml*. Нельзя так просто вытащить нужную информацию по ключу. Нужно изучать отдельно каждый трап и затем считать на пальцах в каком слове и букве прячется нужное значение. Не очень современно конечно, но да ладно, *snmp* это давно легаси. Возможно *gNMI* нас всех спасет. Тогда и будем делать красиво. Возращаемся к нашим баранам.

По приведенным трапам видно, что в случае трапа **315** мы легко можем вытащить номер порта и даже мак-адрес, который вызвал срабатывание *port-security*. И конечно мы это сделаем. 
   
А вот с **548**-м все сложнее. Здесь нам доступны только название коммутатора (switch-20) (кстати оно не сохранится если переслать трап на другой хост используя инструкцию *forward* в *snmptrapd*), а вместо названия заблокированного интерфейса в нашем распоряжении есть только его индекс - *SNMP ifIndex*.  

Индекс содержится в последней строке нашего трапа *iso.3.6.1.4.1.9.9.548.1.3.1.1.2.10640.0 9* и равен числу **10640**  

Для того, чтобы из *ifIndex* получить название порта коммутатора, необходимо обратиться к самому коммутатору по *SNMP*. Этим в скрипте занимается отдельная функция ***find_ifDesc_from_ifIndex()***. И конечно коммутатор должен быть настроен на то чтобы принимать *SNMP* от нашего хоста причем с тем snmp-community, которое прописывается все в том же *config.ini*.  

Итоговый код парсинга наших трапов выглядит следующим образом:

```python
if mode is "disable":
    trapvalue = traplist[-2]
    ifIndex = trapvalue.split(".")[-2]
    ifName = find_ifDesc_from_ifIndex(ip, ifIndex, snmp_config['community'])

elif mode is "restrict":
    ifName = traplist[7].strip('"')
    mac = ':'.join(traplist[-7:-1]).strip('"')

```

Итак в результате у нас есть один из двух наборов данных
1. (имя коммутатора, имя интерфейса, тип трапа)
2. (имя коммутатора, имя интерфейса, мак-адрес, тип трапа)

И эти данные необходимо передать *Zabbix*-у.   

По какой то причине архитекторы *Zabbix* не позволяют инжектить данные в систему используя *API*. Единственный (на момент написания скрипта, а написал я его уже давно) способ сунуть туда произвольные данные - использовать ***zabbix_sender***.  

***zabbix_sender*** для нужного *hostname* передает *key:value* пару где *value* - это те самые данные которые мы подготовили, а в качестве ключа необходимо указать предопределенное имя ключа для элементов данных в *Zabbix*. И ровно это самое имя нужно прописать в *config.ini* для параметров *trapkeyname_х*. В качестве *value* передается имя интерфейса или строка, состоящая из имени интерфейса и мак-адреса.

##### Установка скрипта #####
Скрипт может находиться где угодно. Запускается он демоном *snmptrapd* и к *Zabbix* никак не привязан. Но лично я держу его в */etc/zabbix/external-scripts* просто потому что "а почему бы и нет".  

Для функционирования скрипта требуется ряд модулей, которые перечислены в файле requirements.txt.

Для установки необходимых модулей достаточно запустить команду:
```bash
sudo -H pip install -r requirements.txt
```

Так же необходимо наличие утилиты *zabbix_sender*.  

В моей любимой *Ubuntu* она идет отдельным пакетом, который так и называется *zabbix-sender*, правда с дефисом вместо нижнего подчеркивания. Ну т.е. *sudo apt install zabbix-sender*

#### 4. Zabbix

В *Zabbix* для каждого коммутатора, с которого мы хотим получать трапы, нужно создать элементы данных (Items) и по одному триггеру на каждый трап. Как уже говорилось выше, имена ключей для этих элементов данных должны быть прописаны в файле *config.ini*. Но можно не заморачиваться. Готовый шаблон с этими компонентами уже лежит в репозитарии вместе с кодом. 

Так же в Zabbix необходимо создать специального пользователя от имени которого будет происходить взаимодействие скрипта с *Zabbix* через *API*. Сгодится простой пользователь с правами read-only для группы с коммутаторами и без доступа к Frontend.

Результат будет выглядеть как то так:  

![](https://habrastorage.org/webt/ge/46/mj/ge46mjnp2x-jzyxmi5hehswifr4.png)

т.е. четко видно, что на коммутаторе с именем *catalyst100* заблокировался порт *Fa0/2* левым мак-адресом *00:22:55:D4:3F:51*  

Тут, кстати, пригодится один грязный хак. По неведомой причине, в фронтенде *Zabbix*, в виджете Problems для одноименного поля стоит ограничение в 20 символов на длину строки для значения тригера. Но один только мак-адрес занимает 17 символов, а с названием интерфейса как минимум 23. В общем для полной красоты это ограничение надо поменять. Находится оно в файле:
```bash
$ZABBIX_FRONTEND_HOME/include/items.inc.php
```
Искать вот такой фрагмент:

```php
        switch ($item['value_type']) {
                case ITEM_VALUE_TYPE_STR:
                        $mapping = getMappedValue($value, $item['valuemapid']);
                // break; is not missing here
                case ITEM_VALUE_TYPE_TEXT:
                case ITEM_VALUE_TYPE_LOG:
                        if ($trim && mb_strlen($value) > 20) {
                                $value = mb_substr($value, 0, 20).'...';
                        }

```
И затем тюнить обе 20-ки. Я поставил 30. Теперь выглядит красиво. Вот наверное и все на этом. Готов ответить на вопросы в комментариях.

p.s. Хочу поделиться одной специфичной для Заббикс фишкой под названием *tags*. Наверное многие в курсе, а многим просто не интересно поэтому спрятал:
<spoiler title="zabbix tags">
Смотрите как я настраиваю Actions:  
![](https://habrastorage.org/webt/gh/ua/ne/ghuanea2za4pvyjm2b2vlmsgjyk.png)

tag *portsecurity* прописан в шаблоне для каждого триггера и теперь условия в *Actions* 
можно записать одной строчкой. Или двумя. Удобнейшая вещь, которой мне раньше сильно не хватало.  
</spoiler>
