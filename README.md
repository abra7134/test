## Инструкция по добавлению новых выделенных IP адресов

Итак, возникла задача добавить на виртуальный сервер новых IP адресов выделенных провайдером.  
Например провайдер нам выдал следующие настройки:
* ip: 85.67.34.1/30, шлюз 85.67.34.2
* ip: 85.67.38.17/30, шлюз 85.67.38.18
 
> В ОСах семейства Debian настройка сетевых интерфейсов описывается в конфигурационном файле **/etc/network/interfaces**.
Также конфигурация может быть разделена на части и размещаться в каталоге **/etc/network/interfaces.d** в отдельных файлах. Как правило для каждого интерфейса оформляется отдельный файл, что в некоторых ситуациях упрощает процедуру добавления/удаления настроек, т.к. достаточно просто добавить или удалить файл, а не редактировать его.

Открываем файл **/etc/network/interfaces** и смотрим содержимое:
```
# The eth0 network interface
auto eth0
iface eth0 inet static
    address 35.67.35.5/30
    gateway 35.67.35.6
```

Из файла видно, что в системе единственный сетевой интерфейс **eth0**, что ОС при загрузке будет его автоматически поднимать **auto eth0**, что используется ipv4 адресация и прописан статический адрес **inet static**.  

Добавляем новые IP адреса подобным образом, но внимательно смотрим на схему наименования сетевых интерфейсов в этом случае:
```
# The eth0 network interface
auto eth0
iface eth0 inet static
    address 35.67.35.5/30
    gateway 35.67.35.6

# Additional addresses
auto eth0:1
iface eth0:1 inet static
    address 85.67.34.1/30
auto eth0:2
iface eth0:2 inet static
    address 85.67.38.17/30
```

Как видим, для указания дополнительных адресов (алисов) интерфейса его имя указывается в нотации: **интерфейса:номер_алиаса**.  
Если конфигурацию оставим в таком виде, что все пакетики пришедшие на новые IP адреса обратно будут уходить на маршрут по-умолчанию (для алиасов невозможно указать свои шлюзы). Если нам необходимо обратно отправлять пакетики на тот же шлюз с которого они пришли, то необходимо будет использовать маршрутизацию по политикам. Для этого создадим дополнительные таблицы маршрутизации с указанием необходимых шлюзов.

Для начала посмотрим, какие таблицы уже существуют:
```sh
$ cat /etc/iproute2/rt_tables 
#
# reserved values
#
255	local
254	main
253	default
0	unspec
#
# local
#
#1	inr.ruhep
```

Как видно таблички с номерами 0,253,254,255 зарезервированы, соответственно нам можно использовать любые другие числа.
Добавляем две новые таблицы с номерами 1 и 2 (создаётся 1 таблица на один новый IP адрес):
```sh
$ echo 1 gateway1 >> /etc/iproute2/rt_tables
$ echo 2 gateway2 >> /etc/iproute2/rt_tables
```
И теперь в конфигурационный файл **/etc/network/interfaces** добавляем следующие строки:
```
# The eth0 network interface
auto eth0
iface eth0 inet static
    address 35.67.35.5/30
    gateway 35.67.35.6

# Additional addresses
auto eth0:1
iface eth0:1 inet static
    address 85.67.34.1/30
    up ip rule add from 85.67.34.1 table gateway1
    up ip route add default via 85.67.34.2 table gateway1
    down ip route del default table gateway1
    down ip rule del from 85.67.34.1 table gateway1
auto eth0:2
iface eth0:2 inet static
    address 85.67.38.17/30
    up ip rule add from 85.67.38.17 table gateway2
    up ip route add default via 85.67.38.18 table gateway2
    down ip route del default table gateway2
    down ip rule del from 85.67.38.17 table gateway2
```

Ключевые слова **up** и **down** указывают системе что запустить при поднятии и отключении интерфейса соотвественно. Мы сначала добавляем новую политику маршрутизации (**ip rule add**), указывающую, что все исходящие пакетики с конкретного IP адреса (** from x.x.x.x**) будут просматривать таблицу (**table xxx**), в которой мы указываем свой шлюз по-умолчанию (**ip route add default via x.x.x.x table xxx**).

Далее осталось запустить новые интерфейсы, делаем это командами:
```sh
$ ifup eth0:1
$ ifup eth0:2
```
После чего новые адреса заработают. Останется проверить их доступность извне (например просто пропинговать).
> Если сетевые сервисы на сервере привязаны к адресу 0.0.0.0, то перезапуск не требуется.