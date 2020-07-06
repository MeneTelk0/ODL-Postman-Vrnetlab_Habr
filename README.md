# Руководство по работе с сетевым оборудованием при помощи OpenDaylight, Postman и Vrnetlab

![](https://habrastorage.org/webt/mm/ee/mu/mmeemunxzfcaodkl5ro0x7_mr-w.png)

В этой статье я расскажу, как настроить *OpenDaylight* для работы с сетевым оборудованием, а также покажу, как с помощью *Postman* и простых *RESTCONF* запросов этим оборудованием можно управлять. Работать с железом мы не будем, а вместо этого развернем небольшую виртуальную лабораторию из одного роутера (для демонстрации нам этого будет достаточно) с помощью *Vrnetlab* поверх *Ubuntu 20.04 LTS*.
<cut/>
## Содержание
 - __Необходимые знания__
 - __Часть 1__: кратко обсуждаем *OpenDaylight (ODL)*, *Postman* и *Vrnetlab* и зачем они нам потребуются
 - __Часть 2__: описание лабораторной работы
 - __Часть 3__: настраиваем *OpenDaylight*
 - __Часть 4__: настраиваем *Vrnetlab*
 - __Часть 5__: с помощью *Postman* подключаем виртуальный роутер (*Juniper vMX*) к *ODL*
 - __Часть 6__: получаем и изменяем конфигурацию роутера с помощью *Postman* и *ODL*
 - __Заключение__
 - __Список Литературы__

## Необходимые знания
Для того, чтобы статья не слишком растягивалась, некоторые технические подробности я буду опускать (со ссылками на литературу, где про них можно почитать). 
Темы, которые хорошо бы (но совсем не обязательно) знать перед прочтением:
- NETCONF (RFC 6241), RESTCONF (RFC 8040)
- XML / JSON
- YANG (RFC 6020 + RFC 7950)
- Базовое понимание Docker

## Часть 1: немного теории
![](https://habrastorage.org/webt/o7/vr/bp/o7vrbpyvgsueqrvfrozuipc5hxy.png)
- Открытая SDN платформа для управления и автоматизации всевозможных сетей, поддерживаемая *Linux Foundation*
- Java inside 
- Основан на Model-Driven Service Abstraction Level (MD-SAL)
- Использует YANG модели для автоматического создания RESTCONF API сетевых устройств

Основной модуль для управления сетью. Именно через него мы будем общаться с подключенными устройствами. Управляется через свой собственный API.

![](https://habrastorage.org/webt/sg/cq/gs/sgcqgsqsybafjfkwab74prftt7s.png)
- Инструмент для тестирования API
- Простой и удобный для использования интерфейс

В нашем случае он нам интересен как средство для отправки REST запросов на API OpenDaylight'а. Можно, конечно, и вручную запросы отправлять, но в Postman все выглядит очень наглядно и для наших целей подходит как нельзя лучше.
Для желающих покопаться: по нему написано много обучающих материалов ([например](https://habr.com/ru/company/kolesa/blog/351250/)). 

![](https://habrastorage.org/webt/zs/wt/py/zswtpyxzib9vjdmcdvutk3dynq0.png)
- Инструмент для развертывания виртуальных роутеров в Docker'е
- Поддерживает: Cisco XRv, Juniper vMX, Arista vEOS, Nokia VSR и др.
- Open Source

Очень интересный, но малоизвестный инструмент. В нашем случае с его помощью мы запустим Juniper vMX на обычной Ubuntu 20.04 LTS.

## Часть 2: лабораторная работа
В рамках этого туториала мы будем настраиваить следующую систему:
![](https://habrastorage.org/webt/7k/kk/j-/7kkkj-fuyty6eyjkjj-qwx3s8oe.png)
#### Как это работает 
- *Juniper vMX* поднимается в *Docker* контейнере (средствами *Vrnetlab*) и функционирует как самый обычный виртуальный роутер.
- *ODL* подключен к роутеру и позволяет управлять им.
- *Postman* запущен на отдельной машине и через него мы отправляем команды *ODL*: на подключение/удаление роутера, изменение конфигурации и тп.

##### Комментарий к устройству системы
*Juniper vMX* и *ODL* требуют довольно много ресурсов для своей стабильной работы. Один только *vMX* просит 6 Gb оперативной памяти и 4 ядра. Поэтому было принято решение вынести всех "тяжеловесов" на отдельную машину (*Heulett Packard Enterprise ProLiant Gen8, Ubuntu 20.04 LTS*). Роутер, конечно, на ней не "летает", но для небольших экспериментов производительности достаточно.

## Часть 3: настраиваем OpenDaylight
![](https://habrastorage.org/webt/x_/e1/be/x_e1beuyefiadpvdhusg1uvdiga.png)

*Актуальная версия ODL на момент написания статьи - Magnesium SR1*
1) Устанавливаем *Java OpenJDK 11* (за более подробной установкой [сюда](https://linuxize.com/post/install-java-on-ubuntu-18-04/))
````
sudo apt install default-jdk
````
2) Находим и скачиваем свежую сборку *ODL* [отсюда](http://www.opendaylight.org/software/downloads)
3) Разархивируем скачанный архив
4) Переходим в полученную директорию
5) Запускаем `./bin/karaf`

На этом шаге *ODL* должен запуститься и мы окажемся в консоли (Для доступа извне используется порт 8181).

Далее устанавливаем *ODL Features*, предназначенные для работы с протоколами *NETCONF* и *RESTCONF*. Для этого в консоли *ODL* выполняем:
````
feature:install odl-netconf-topology odl-restconf-all
````
На этом простейшая настройка *ODL* завершена. (Более подробно можно прочитать [здесь](https://docs.opendaylight.org/en/stable-magnesium/getting-started-guide/installing_opendaylight.html))

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
Для установки *Vrnetlab* клонируем соответствующий репозиторий с github.com:
````
ubuntu:~$ cd ~
ubuntu:~$ git clone https://github.com/plajjan/vrnetlab.git
````
Переходим в директорию *vrnetlab*:
````
ubuntu:~$ cd ~/vrnetlab
````
Здесь можно увидеть все скрипты, неоходимые для запуска. Обратите внимание, что для каждого типа роутера сделана соответсвующая директория:
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
Каждый роутер, который поддерживается *Vrnetlab*, имеет свою уникальную процедуру настройки. В случае *Juniper vMX* нам достаточно лишь положить .tgz архив с роутером (скачать его можно с [официального сайта](https://support.juniper.net/support/downloads/?p=vmx#sw)) в директорию vmx и выполнить команду `make`:
````
ubuntu:~$ cd ~/vrnetlab/vmx
ubuntu:~$ # Копируем в эту директорию .tgz архив с роутером
ubuntu:~$ sudo make
````
Установка *vMX* займет порядка 10-20 минут. После можно будет увидеть image нашего роутера в *Docker*:
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
Рекомендации по установке для роутеров различных вендоров можно найти на [github проекта](https://github.com/plajjan/vrnetlab).

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

#### Проверяем подключение роутера
Создадим __GET__ запрос:
1.    Строка запроса:
````
GET http://10.132.1.202:8181/restconf/operational/network-topology:network-topology/topology/topology-netconf/
````
2.    На вкладке Authorization необходимо выставить параметр `Basic Auth` и логин/пароль: admin/admin.

Отправляем. Должны получить статус "200 OK" и список всех поддерживаемых устройством *YANG Schema*:
![](https://habrastorage.org/webt/rp/9n/iv/rp9nivb4m4_ukztr-ulfrxiqlao.png)

__Комментарий__: Чтобы увидеть последнее, в моем случае необходимо было подождать порядка 10 минут пока все YANG sсhema выгрузятся на ODL. До этого момента при выполнении данного GET запроса будет выведено следующее:
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

Чтобы проверить, что конфигурация изменилась, можно воспользоваться предыдущим запросом. Но для примера мы создадим еще один, который выведет нам информацию только о сконфигугированных на роутере протоколах: 
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
После отправки получим следующий результат (Используя __GET__ запрос):
![](https://habrastorage.org/webt/5_/r-/me/5_r-mepekyu2maifbhahfcyebrw.png)

## Заключение
Данный туториал даёт простейшие примеры того, как можно взаимодействовать с сетевым оборудованием при помощи OpenDaylight. Без сомнения, запросы из приведенных примеров можно сделать сильно сложнее и настраивать целые сервисы одним кликом мыши - все ограничено только вашей фантазией*. 

## Список литературы
1.    [Vrnetlab: Emulate networks using KVM and Docker](https://www.brianlinkletter.com/vrnetlab-emulate-networks-using-kvm-and-docker/) / Brian Linkletter
2.    OpenDaylight Cookbook / Mathieu Lemay, Alexis de Talhouet, Et al
3.    Network Programmability with YANG / Benoît Claise, Loe Clarke, Jan Lindblad
4.    Learning XML, Second Edition / Erik T. Ray 
5.    Effective DevOps / Jennifer Davis, Ryn Daniels


