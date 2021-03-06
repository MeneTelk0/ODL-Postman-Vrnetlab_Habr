
# Автоматизация сетей или как создать свою виртуальную лабораторию при помощи OpenDaylight, Postman и Vrnetlab
![](https://habrastorage.org/webt/mm/ee/mu/mmeemunxzfcaodkl5ro0x7_mr-w.png)

В этой статье я расскажу, как настроить *OpenDaylight* для работы с сетевым оборудованием, а также покажу, как с помощью *Postman* и простых *RESTCONF* запросов этим оборудованием можно управлять. Работать с железом мы не будем, а вместо этого развернем небольшую виртуальную лабораторию из одного-единственного роутера с помощью *Vrnetlab* поверх *Ubuntu 20.04 LTS*.
<cut/>
Подробную настройку я покажу на примере роутера *Juniper vMX 20.1R1.11*, а затем мы сравним ее с настройкой *Cisco xRV9000 7.0.2*.
## Содержание
 - __Необходимые знания__
 - __Часть 1__: кратко обсуждаем *OpenDaylight (ODL)*, *Postman* и *Vrnetlab* и зачем они нам потребуются
 - __Часть 2__: описание виртуальной лаборатории
 - __Часть 3__: настраиваем *OpenDaylight*
 - __Часть 4__: настраиваем *Vrnetlab*
 - __Часть 5__: с помощью *Postman* подключаем виртуальный роутер (*Juniper vMX*) к *ODL*
 - __Часть 6__: получаем и изменяем конфигурацию роутера с помощью *Postman* и *ODL*
 - __Часть 7__: добавляем Cisco xRV9000
 - __Заключение__
 - __Список Литературы__
 - __P.S.__

## Необходимые знания
Для того, чтобы статья не превратилась в простыню, некоторые технические подробности я опустил (со ссылками на литературу, где про них можно почитать).
В связи с чем, предлагаю вам темы, которые хорошо бы (но почти не обязательно) знать перед прочтением:
- [NETCONF](https://habr.com/ru/post/135259/), [RESTCONF](https://tools.ietf.org/html/rfc8040)
- [XML](https://habr.com/ru/company/mailru/blog/475474/) / [JSON](https://gist.github.com/ermakovpetr/4c9f56d48e49d822705a)
- [YANG](https://tools.ietf.org/html/rfc6020)
- [Базовое понимание Docker'a](https://habr.com/ru/post/346634/)

## Часть 1: немного теории
![](https://habrastorage.org/webt/o7/vr/bp/o7vrbpyvgsueqrvfrozuipc5hxy.png)
- Открытая SDN платформа для управления и автоматизации всевозможных сетей, поддерживаемая *Linux Foundation*
- Java inside 
- Основан на Model-Driven Service Abstraction Level (MD-SAL)
- Использует YANG модели для автоматического создания RESTCONF API сетевых устройств

Основной модуль для управления сетью. Именно через него мы будем общаться с подключенными устройствами. Управляется через свой собственный API. 
Более подробно про OpenDaylight можно прочитать [здесь](https://www.opendaylight.org/what-we-do/odl-platform-overview).

![](https://habrastorage.org/webt/sg/cq/gs/sgcqgsqsybafjfkwab74prftt7s.png)
- Инструмент для тестирования API
- Простой и удобный для использования интерфейс

В нашем случае он нам интересен как средство для отправки REST запросов на API OpenDaylight'а. Можно, конечно, и вручную запросы отправлять, но в Postman все выглядит очень наглядно и для наших целей подходит как нельзя лучше.
Для желающих покопаться: по нему написано много обучающих материалов ([например](https://habr.com/ru/company/kolesa/blog/351250/)). 

![](https://habrastorage.org/webt/zs/wt/py/zswtpyxzib9vjdmcdvutk3dynq0.png)
- Инструмент для развертывания виртуальных роутеров в Docker'е
- Поддерживает: Cisco XRv, Juniper vMX, Arista vEOS, Nokia VSR и др.
- Open Source

Очень интересный, но малоизвестный инструмент. В нашем случае с его помощью мы запустим Juniper vMX и Cisco xRV9000 на обычной Ubuntu 20.04 LTS. 
Прочитать подробнее о нем можно на [странице проекта](https://github.com/plajjan/vrnetlab).


## Часть 2: лабораторная работа
В рамках этого туториала мы будем настраиваить следующую систему:
![](https://habrastorage.org/webt/cj/ad/eu/cjadeubfejqrbohczcgypggqots.png)
#### Как это работает 
- *Juniper vMX* поднимается в *Docker* контейнере (средствами *Vrnetlab*) и функционирует как самый обычный виртуальный роутер.
- *ODL* подключен к роутеру и позволяет управлять им.
- *Postman* запущен на отдельной машине и через него мы отправляем команды *ODL*: на подключение/удаление роутера, изменение конфигурации и тп.

##### Комментарий к устройству системы
*Juniper vMX* и *ODL* требуют довольно много ресурсов для своей стабильной работы. Один только *vMX* просит 6 Gb оперативной памяти и 4 ядра. Поэтому было принято решение вынести всех "тяжеловесов" на отдельную машину (*Heulett Packard Enterprise MicroServer ProLiant Gen8, Ubuntu 20.04 LTS*). Роутер, конечно, на ней не "летает", но для небольших экспериментов производительности хватает.

## Часть 3: настраиваем OpenDaylight
![](https://habrastorage.org/webt/x_/e1/be/x_e1beuyefiadpvdhusg1uvdiga.png)

*Актуальная версия ODL на момент написания статьи - Magnesium SR1*
1) Устанавливаем *Java OpenJDK 11* (за более подробной установкой [сюда](https://linuxize.com/post/install-java-on-ubuntu-18-04/))
````
ubuntu:~$ sudo apt install default-jdk
````
2) Находим и скачиваем свежую сборку *ODL* [отсюда](http://www.opendaylight.org/software/downloads)
3) Разархивируем скачанный архив
4) Переходим в полученную директорию
5) Запускаем `./bin/karaf`

На этом шаге *ODL* должен запуститься и мы окажемся в консоли (Для доступа извне используется порт 8181, чем мы воспользуемся далее).

Далее устанавливаем *ODL Features*, предназначенные для работы с протоколами *NETCONF* и *RESTCONF*. Для этого в консоли *ODL* выполняем:
````
opendaylight-user@root> feature:install odl-netconf-topology odl-restconf-all
````
На этом простейшая настройка *ODL* завершена. (Более подробно можно прочитать [здесь](https://docs.opendaylight.org/en/stable-magnesium/getting-started-guide/installing_opendaylight.html)).

## Часть 4: настраиваем Vrnetlab
![](https://habrastorage.org/webt/dw/4j/mf/dw4jmfbsvwaa_8cda7-yayf5nys.png)
#### Подготовка системы
Перед установкой *Vrnetlab* необходимо поставить требуемые для его работы пакеты. Такие как [Docker](https://docs.docker.com/engine/install/ubuntu/), [git](https://git-scm.com/doc), [sshpass](https://www.tecmint.com/sshpass-non-interactive-ssh-login-shell-script-ssh-password/):
````
ubuntu:~$ sudo apt update
ubuntu:~$ sudo apt -y install python3-bs4 sshpass make
ubuntu:~$ sudo apt -y install git
ubuntu:~$ sudo apt install -y \
    apt-transport-https ca-certificates \
    curl gnupg-agent software-properties-common
ubuntu:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
ubuntu:~$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
ubuntu:~$ sudo apt update
ubuntu:~$ sudo apt install -y docker-ce docker-ce-cli containerd.io
````
#### Установка Vrnetlab
Для установки *Vrnetlab* клонируем соответствующий репозиторий с github:
````
ubuntu:~$ cd ~
ubuntu:~$ git clone https://github.com/plajjan/vrnetlab.git
````
Переходим в директорию *vrnetlab*:
````
ubuntu:~$ cd ~/vrnetlab
````
Здесь можно увидеть все скрипты, необходимые для запуска. Обратите внимание, что для каждого типа роутера сделана соответствующая директория:
````
ubuntu:~/vrnetlab$ ls
CODE_OF_CONDUCT.md  config-engine-lite        openwrt           vr-bgp
CONTRIBUTING.md     csr                       routeros          vr-xcon
LICENSE             git-lfs-repo.sh           sros              vrnetlab.sh
Makefile            makefile-install.include  topology-machine  vrp
README.md           makefile-sanity.include   veos              vsr1000
ci-builder-image    makefile.include          vmx               xrv
common              nxos                      vqfx              xrv9k
````
#### Создаем image роутера
Каждый роутер, который поддерживается *Vrnetlab*, имеет свою уникальную процедуру настройки. В случае *Juniper vMX* нам достаточно закинуть .tgz архив с роутером (скачать его можно с [официального сайта](https://support.juniper.net/support/downloads/?p=vmx#sw)) в директорию vmx и выполнить команду `make`:
````
ubuntu:~$ cd ~/vrnetlab/vmx
ubuntu:~$ # Копируем в эту директорию .tgz архив с роутером
ubuntu:~$ sudo make
````
Сборка образа *vMX* займет порядка 10-20 минут. Самое время сходить заварить кофе! 


<spoiler title="Почему же так долго, спросите вы?">
Перевод [ответа](https://github.com/plajjan/vrnetlab/tree/master/vmx) автора на этот вопрос:

"Это связано с тем, что при первом запуске VCP (Control Plane) считывает файл конфигурации, который определяет, будет ли он работать в качестве VRR VCP в vMX. Ранее этот запуск выполнялся во время запуска Docker, но это означало, что VCP всегда перезапускался один раз, прежде чем виртуальный маршрутизатор становился доступным, что приводило к длительному времени загрузки (около 5 минут). Теперь первый запуск VCP выполняется во время сборки образа Docker, и поскольку сборка Docker не может быть запущена с параметром --privileged, это означает, что qemu работает без аппаратного ускорения KVM и, таким образом, сборка занимает очень много времени. Во время этого процесса выводится много логов, так что, по крайней мере, вы сможете увидеть, что происходит. Я думаю, что длительная сборка не так страшна, потому что образ мы создаем один раз, а запускаем множество."
 </spoiler>
После  можно будет увидеть image нашего роутера в *Docker*:
````
ubuntu:~$ sudo docker image list
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
vrnetlab/vr-vmx     20.1R1.11           b1b2369b453c        3 weeks ago         4.43GB
debian              stretch             614bb74b620e        7 weeks ago         101MB
````
#### Запускаем контейнер vr-vmx
Запускаем командой:
````
ubuntu:~$ sudo docker run -d --privileged --name jun01 b1b2369b453c
````
Далее можем посмотреть информацию об активных контейнерах:
````
ubuntu:~$ sudo docker container list
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS                                                 NAMES
120f882c8712        b1b2369b453c        "/launch.py"        2 minutes ago       Up 2 minutes (unhealthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp   jun01
````
#### Подключаемся к роутеру
IP-адрес сетевого интерфейса роутера можно получить следующей командой:
```` 
ubuntu:~$ sudo docker inspect --format '{{.NetworkSettings.IPAddress}}' jun01
172.17.0.2
````
По умолчанию, *Vrnetlab* создает у роутера пользователя __vrnetlab/VR-netlab9__. Подключаемся с помощью `ssh`:
````
ubuntu:~$ ssh vrnetlab@172.17.0.2
The authenticity of host '172.17.0.2 (172.17.0.2)' can't be established.
ECDSA key fingerprint is SHA256:g9Sfg/k5qGBTOX96WiCWyoJJO9FxjzXYspRoDPv+C0Y.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.17.0.2' (ECDSA) to the list of known hosts.
Password:
--- JUNOS 20.1R1.11 Kernel 64-bit  JNPR-11.0-20200219.fb120e7_buil
vrnetlab> show version
Model: vmx
Junos: 20.1R1.11
````
На этом настройка роутера завершена. 
Рекомендации по установке для роутеров различных вендоров можно найти на [github проекта](https://github.com/plajjan/vrnetlab) в соответствующих директориях.

## Часть 5: Postman - подключаем роутер к OpenDaylight
#### Установка Postman 
Для установки достаточно скачать приложение [отсюда](https://www.postman.com/downloads/).
#### Подключение роутера к ODL
Создадим __PUT__ запрос:
![](https://habrastorage.org/webt/wt/tf/0-/wttf0--8yp9aa6gi_fneqne1hg0.png)

1.    Строка запроса:
````
PUT http://10.132.1.202:8181/restconf/config/network-topology:network-topology/topology/topology-netconf/node/jun01
````
2.    Тело запроса (вкладка Body):
````xml
<node xmlns="urn:TBD:params:xml:ns:yang:network-topology">
<node-id>jun01</node-id>
<host xmlns="urn:opendaylight:netconf-node-topology">172.17.0.2</host>
<port xmlns="urn:opendaylight:netconf-node-topology">22</port>
<username xmlns="urn:opendaylight:netconf-node-topology">vrnetlab</username>
<password xmlns="urn:opendaylight:netconf-node-topology">VR-netlab9</password>
<tcp-only xmlns="urn:opendaylight:netconf-node-topology">false</tcp-only>
<schema-cache-directory xmlns="urn:opendaylight:netconf-node-topology">jun01_cache</schema-cache-directory>
</node>
````
3.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin. Это необходимо для доступа к ODL:
![](https://habrastorage.org/webt/fo/vq/uz/fovquziwqi0hpjemf2jevyq845o.png)
4.    На вкладке Headers необходимо добавить два заголовка:
- Accept application/xml
- Content-Type application/xml

Наш запрос сформирован. Отправляем. Если все было настроено правильно, то нам должен вернуться статус "201 Created":
![](https://habrastorage.org/webt/k0/xy/z3/k0xyz3gaeqyfgprvsp-47o7pv7c.png)

<spoiler title="Что делает этот запрос?">
Мы создаем node внутри *ODL* с параметрами реального роутера, к которому мы хотим получить доступ.
````
xmlns="urn:TBD:params:xml:ns:yang:network-topology"
xmlns="urn:opendaylight:netconf-node-topology"
````
Это внутренние пространства имен *XML* (*XML namespace*) для *ODL* в соответствии с которыми он создает node.
Далее, соответственно, имя роутера - это *node-id*, адрес роутера - *host* и тд. 
Самая интересная строчка - последняя. *Schema-cache-directory* создает директорию, в которую выкачиваются все файлы *YANG Schema* подключенного роутера. Найти их можно в `$ODL_ROOT/cache/jun01_cache`. 
</spoiler>

#### Проверяем подключение роутера
Создадим __GET__ запрос:
1.    Строка запроса:
````
GET http://10.132.1.202:8181/restconf/operational/network-topology:network-topology/topology/topology-netconf/
````
2.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.

Отправляем. Должны получить статус "200 OK" и список всех поддерживаемых устройством *YANG Schema*:
![](https://habrastorage.org/webt/rp/9n/iv/rp9nivb4m4_ukztr-ulfrxiqlao.png)

__Комментарий__: Чтобы увидеть последнее, в моем случае необходимо было подождать порядка 10 минут после выполнения __PUT__, пока все *YANG sсhema* выгрузятся на *ODL*. До этого момента при выполнении данного __GET__ запроса будет выведено следующее:
![](https://habrastorage.org/webt/vd/yn/jr/vdynjrtgsycrqvfnwzy0nwdqcm4.png)

#### Удаляем роутер 
Создадим __DELETE__ запрос:
1.    Строка запроса:
````
DELETE http://10.132.1.202:8181/restconf/config/network-topology:network-topology/topology/topology-netconf/node/jun01
````
2.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.

## Часть 6: Изменяем конфигурацию роутера
#### Получаем конфигурацию
Создадим __GET__ запрос:
1.    Строка запроса:
````
GET http://10.132.1.202:8181/restconf/config/network-topology:network-topology/topology/topology-netconf/node/jun01/yang-ext:mount/
````
2.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.

Отправляем. Должны получить статус "200 OK" и конфигурацию роутера:
![](https://habrastorage.org/webt/rq/zz/ls/rqzzlsee-xzsqk42ex1cuagln2u.png)

#### Создаем конфигурацию 
В качестве примера создадим следующую конфигурацию и поизменяем ее:
````
protocols {
    bgp {
        disable;
        shutdown;
    }
}
````
Создадим __POST__ запрос:
1.    Строка запроса:
````
POST http://10.132.1.202:8181/restconf/config/network-topology:network-topology/topology/topology-netconf/node/jun01/yang-ext:mount/junos-conf-root:configuration/junos-conf-protocols:protocols
````
2.    Тело запроса (вкладка Body):
````xml
<bgp xmlns="http://yang.juniper.net/junos/conf/protocols">
    <disable/>
    <shutdown>
    </shutdown>
</bgp>
````
3.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.
4.    На вкладке Headers необходимо добавить два заголовка:
- Accept application/xml
- Content-Type application/xml

После отправки должны получить статус "204 No Content"

Чтобы проверить, что конфигурация изменилась, можно воспользоваться предыдущим запросом. Но для примера мы создадим еще один, который выведет нам информацию только о сконфигурированных на роутере протоколах. 
Создадим __GET__ запрос:
1.    Строка запроса:
````
GET http://10.132.1.202:8181/restconf/config/network-topology:network-topology/topology/topology-netconf/node/jun01/yang-ext:mount/junos-conf-root:configuration/junos-conf-protocols:protocols
````
2.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.

После выполнения запроса увидим следующее:
![](https://habrastorage.org/webt/kj/sp/rs/kjsprsgse4caaudejogyojh4gyu.png)
#### Изменяем конфигурацию
Изменим информацию о протоколе BGP. После наших действий она будет выглядеть следующим образом:
````
protocols {
    bgp {
        disable;
    }
}
````
Создадим __PUT__ запрос:
1.    Строка запроса:
````
PUT http://10.132.1.202:8181/restconf/config/network-topology:network-topology/topology/topology-netconf/node/jun01/yang-ext:mount/junos-conf-root:configuration/junos-conf-protocols:protocols
````
2.    Тело запроса (вкладка Body):
````xml
<protocols xmlns="http://yang.juniper.net/junos/conf/protocols">
    <bgp>
        <disable/>
    </bgp>
</protocols>
````
3.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.
4.    На вкладке Headers необходимо добавить два заголовка:
- Accept application/xml
- Content-Type application/xml

Используя предыдущий __GET__ запрос, видим изменения:
![](https://habrastorage.org/webt/uc/aq/eo/ucaqeoipjtnyzqj0n9o2yrdccnc.png)
#### Удаляем конфигурацию
Создадим __DELETE__ запрос:
1.    Строка запроса:
````
DELETE http://10.132.1.202:8181/restconf/config/network-topology:network-topology/topology/topology-netconf/node/jun01/yang-ext:mount/junos-conf-root:configuration/junos-conf-protocols:protocols
````
2.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.

При вызове __GET__ запроса с информацией о протоколах увидим следующее:
![](https://habrastorage.org/webt/ij/jh/uy/ijjhuyiesyxsfa-oyovwwn1yvmu.png)

#### Дополнение:
Для того, чтобы изменить конфигурацию, не обязательно отправлять тело запроса в формате *XML*. Это можно сделать и в формате *JSON*.
Для этого, например, в запросе __PUT__ на изменение конфигурации заменим тело запроса на: 
````json
{
    "junos-conf-protocols:protocols": {
        "bgp": {
            "description" : "Changed in postman" 
        }
    }
}
````
Не забудьте поменять на вкладке Headers заголовки на:
- Accept application/json
- Content-Type application/json

После отправки получим следующий результат (Ответ смотрим используя __GET__ запрос):
![](https://habrastorage.org/webt/5_/r-/me/5_r-mepekyu2maifbhahfcyebrw.png)

## Часть 7: добавляем Cisco xRV9000
Что мы все о Джунипере, да о Джунипере? Давайте о Cisco поговорим! 
У меня нашелся xRV9000 версии 7.0.2 (зверюга, которому нужны 8Gb RAM и 4 ядра) - его и запустим.
#### Запуск контейнера
Процесс создания Docker контейнера практически ничем не отличается от Juniper. Аналогично, закидываем .qcow2 файл с роутером в директорию, соответствующую его названию, (в данном случае xrv9k) и выполняем команду `make docker-image`.
Через несколько минут видим, что образ создался:
````
ubuntu:~$ sudo docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
vrnetlab/vr-xrv9k   7.0.2               54debc7973fc        4 hours ago         1.7GB
vrnetlab/vr-vmx     20.1R1.11           b1b2369b453c        4 weeks ago         4.43GB
debian              stretch             614bb74b620e        7 weeks ago         101MB
````
Производим запуск контейнера:
````
ubuntu:~$ sudo docker run -d --privileged --name xrv01 54debc7973fc
````
Через некоторое время смотрим, что контейнер запустился:
````
ubuntu:~$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                 PORTS                                                      NAMES
058c5ecddae3        54debc7973fc        "/launch.py"        4 hours ago         Up 4 hours (healthy)   22/tcp, 830/tcp, 5000-5003/tcp, 10000-10099/tcp, 161/udp   xrv01
````
Подключаемся по ssh:
````
ubuntu@ubuntu:~$ ssh vrnetlab@172.17.0.2
Password:


RP/0/RP0/CPU0:ios#show version
Mon Jul  6 12:19:28.036 UTC
Cisco IOS XR Software, Version 7.0.2
Copyright (c) 2013-2020 by Cisco Systems, Inc.

Build Information:
 Built By     : ahoang
 Built On     : Fri Mar 13 22:27:54 PDT 2020
 Built Host   : iox-ucs-029
 Workspace    : /auto/srcarchive15/prod/7.0.2/xrv9k/ws
 Version      : 7.0.2
 Location     : /opt/cisco/XR/packages/
 Label        : 7.0.2

cisco IOS-XRv 9000 () processor
System uptime is 3 hours 22 minutes
````
#### Подключаем роутер к OpenDaylight
Добавление происходит совершенно аналогичным с vMX образом. Нужно только названия поменять.
___PUT___ запрос:
![](https://habrastorage.org/webt/i2/cc/ug/i2ccuguu8o6n_aicbznq6g3l2p8.png)

Через некоторое время вызываем ___GET___ запрос, чтобы проверить, что все подключилось:
![](https://habrastorage.org/webt/dk/eg/zj/dkegzjm9orfrwfhfe9l8o7sz9f4.png)

#### Изменяем конфигурацию
Настроим следующую конфигурацию:
````
!
router ospf LAB
 mpls ldp auto-config
!
````

Создадим __POST__ запрос:
1.    Строка запроса:
````
POST http://10.132.1.202:8181/restconf/config/network-topology:network-topology/topology/topology-netconf/node/xrv01/yang-ext:mount/Cisco-IOS-XR-ipv4-ospf-cfg:ospf
````
2.    Тело запроса (вкладка Body):
````json
{
    "processes": {
        "process": [
            {
                "process-name": "LAB",
                "default-vrf": {
                    "process-scope": {
                        "ldp-auto-config": [
                            null
                        ]
                    }
                }
            }
        ]
    }
}
````
3.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.
4.    На вкладке Headers необходимо добавить два заголовка:
- Accept application/json
- Content-Type application/json

После его выполнения должны получить статус "204 No Content".

Проверим, что у нас получилось. 
Для этого создадим __GET__ запрос:
1.    Строка запроса:
````
GET http://10.132.1.202:8181/restconf/config/network-topology:network-topology/topology/topology-netconf/node/xrv01/yang-ext:mount/Cisco-IOS-XR-ipv4-ospf-cfg:ospf
````
2.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.

После выполнения должны увидеть следующее:
![](https://habrastorage.org/webt/ed/j3/lp/edj3lpulnqcbm0cyit5j1to1lyi.png)

Для удаления конфигурации используем __DELETE__:
1.    Строка запроса:
````
DELETE http://10.132.1.202:8181/restconf/config/network-topology:network-topology/topology/topology-netconf/node/xrv01/yang-ext:mount/Cisco-IOS-XR-ipv4-ospf-cfg:ospf
````
2.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.

## Заключение
Итого, как вы могли заметить, настройка OpenDaylight для работы с Cisco или Juniper не сильно отличается - это открывает довольно широкий простор для творчества. Начиная от управления конфигурациями всех сетевых компонентов и заканчивая балансировкой нагрузки с созданием собственных сетевых политик.
В этом туториале я привел простейшие примеры того, как можно взаимодействовать с сетевым оборудованием при помощи OpenDaylight. Без сомнения, запросы из приведенных примеров можно сделать сильно сложнее и настраивать целые сервисы одним кликом мыши - все ограничено только вашей фантазией*. 

## Список литературы
1.    [Vrnetlab: Emulate networks using KVM and Docker](https://www.brianlinkletter.com/vrnetlab-emulate-networks-using-kvm-and-docker/) / Brian Linkletter
2.    OpenDaylight Cookbook / Mathieu Lemay, Alexis de Talhouet, Et al
3.    Network Programmability with YANG / Benoît Claise, Loe Clarke, Jan Lindblad
4.    Learning XML, Second Edition / Erik T. Ray 
5.    Effective DevOps / Jennifer Davis, Ryn Daniels

## P.S.
Если вы вдруг все это уже знаете или, наоборот, прошли и вам запал в душу ODL, то рекомендую посмотреть в сторону разработки приложений на контроллере ODL. Начать можно [отсюда](https://docs.opendaylight.org/en/stable-magnesium/developer-guide/developing-apps-on-the-opendaylight-controller.html).
Успешных экспериментов!

