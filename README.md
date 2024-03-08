<h1>Парсер конфигурации роутеров Mikrotik</h1>

![Mikrotik Logo](https://mikrotik.com/img/mtv2/newlogo.svg)

---

Данный проект предназначен для парсигна конфигурации роутеров Mikrotik и приведения их к используемому у нас DTO

---

# Источники данных

В качестве источника данных используется текстовый файл, содержащий вывод команды `> export`

> [!IMPORTANT]
> 
> Получение файла данных не входит в задачи данного проекта. 
> Используйте либо SSH, либо регулярное резервное копирование на самом устройтсве и получение 
> файла посредством FTP.
> 
> Использование SSH предпочтительнее, так как не создаёт лишнюю нагрузку на внутренний flash-накопитель.


## Название файла конфигурации

Имя файла должно быть в виде ip адреса управления устройтсва.

**Не следует включать в имя файла расширение или иную информацию.**


>configs\
>├── 10.1.0.2\
>├── 172.20.255.1\
>├── 172.20.255.1\
>├── ~~config-router-office1-gw-172.21.2.2.txt~~\
>└── 172.20.255.2


---

# Парсер

Пример конфигурации:

```
[admin@MikroTik] > export
# 1970-01-02 00:01:22 by RouterOS 7.13
# software id = Z5EL-5QE2
#
# model = RB931-2nD
# serial number = A8A60A980AF6
/interface bridge
add admin-mac=74:4D:28:60:AA:99 auto-mac=no comment=defconf name=bridge
/interface wireless
set [ find default-name=wlan1 ] band=2ghz-b/g/n channel-width=20/40mhz-XX disabled=no distance=indoors frequency=auto installation=indoor mode=ap-bridge ssid=MikroTik-60AA9C wireless-protocol=802.11
/interface list
add comment=defconf name=WAN
add comment=defconf name=LAN
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
/ip pool
add name=default-dhcp ranges=192.168.88.10-192.168.88.254
/ip dhcp-server
add address-pool=default-dhcp interface=bridge lease-time=10m name=defconf
/interface bridge port
add bridge=bridge comment=defconf interface=ether2
add bridge=bridge comment=defconf interface=ether3
add bridge=bridge comment=defconf interface=pwr-line1
add bridge=bridge comment=defconf interface=wlan1
/ip neighbor discovery-settings
set discover-interface-list=LAN
/interface list member
add comment=defconf interface=bridge list=LAN
add comment=defconf interface=ether1 list=WAN
/ip address
add address=192.168.88.1/24 comment=defconf interface=bridge network=192.168.88.0
/ip dhcp-client
add comment=defconf interface=ether1
/ip dhcp-server network
add address=192.168.88.0/24 comment=defconf dns-server=192.168.88.1 gateway=192.168.88.1
/ip dns
set allow-remote-requests=yes
/ip dns static
add address=192.168.88.1 comment=defconf name=router.lan
/ip firewall filter
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
add action=accept chain=input comment="defconf: accept ICMP" protocol=icmp
    <часть кода пропущена>
/ip firewall nat
add action=masquerade chain=srcnat comment="defconf: masquerade" ipsec-policy=out,none out-interface-list=WAN
/ipv6 firewall address-list
add address=::/128 comment="defconf: unspecified address" list=bad_ipv6
add address=::1/128 comment="defconf: lo" list=bad_ipv6
    <часть кода пропущена>
/ipv6 firewall filter
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
add action=accept chain=input comment="defconf: accept ICMPv6" protocol=icmpv6
    <часть кода пропущена>
/system note
set show-at-login=no
/tool mac-server
set allowed-interface-list=LAN
/tool mac-server mac-winbox
set allowed-interface-list=LAN
```

## Алгоритм работы

1. Разбиваем конфигурацию на секции;
2. Для каждой секции используем набор регулярных выражений[^1];
3. Формируем json конфигурации Mikrotik'а;
4. С помощью Pydantic'а[^2] создаём DTO Mikrotik'а;
5. Формируем наш внутренний DTO.

[^1]: Смотри `src/regexp/mikrotik/`
[^2]: [Pydantic-docs](https://docs.pydantic.dev/latest/)

### Секции конфигурационного файла

- common
- interface
- ip
- ipv6
- system
- tool

# Реализованный функционал парсера

- [x] common
- [ ] interface
  - [x] interface bridge
  - [ ] interface wireless
  - [ ] interface wireless security-profiles
  - [x] interface list
    - [ ] interface list member
  - [ ] interface bridge port
- [ ] ip
  - [x] ip pool
  - [x] ip dhcp-server
  - [ ] ip neighbor
    - [ ] ip neighbor discovery-settings
  - [x] ip address
  - [ ] ip dhcp-client
  - [ ] ip dhcp-server
  - [ ] ip dhcp-server network
  - [ ] ip dns
    - [ ] ip dns static
  - [ ] ip firewall
  - [ ] ip firewall filter
  - [ ] ip firewall nat
- [ ] ipv6
- [ ] ipv6 firewall
  - ipv6 firewall address-list
  - ipv6 firewall filter
- [ ] system
  - [ ] system note
- [ ] tool
  - [ ] tool mac-server
    - [ ] tool mac-server mac-winbox

# Дополнительный функционал

В каталоге 'src/scripts' лежат скрипты, написанные разными людьми в разное время, 
для обращения к оборудованию, парсинга определенных частей конфигурации и вывода результата в консоль.

На их основе будем собирать наш проект.

Если хотите узнать, что делает тот или иной скрипт - запустите:
```
src/scripts/<script-name> -h --help
```

Например, скрипт `show_interfaces_status.py` выведет табличное представление об интерфейсах маршрутизатора:

| Intefrace | Type   | Admin Status | Oper Status |
|-----------|--------|--------------|-------------|
| Ether1    | Copper | Up           | Up          |
| Ether2    | Copper | Up           | Down        |
| SFP1      | SFP    | Down         | Down        |
