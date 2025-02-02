# Домашняя работа 24

## Тема: DNS - настройка и обслуживание
---
## Домашнее задание:

  - 1. взять стенд https://github.com/erlong15/vagrant-bind 
    - добавить еще один сервер client2
    - завести в зоне dns.lab имена:
      - web1 - смотрит на клиент1
      - web2  смотрит на клиент2
    - завести еще одну зону newdns.lab
    - завести в ней запись
    - www - смотрит на обоих клиентов

  - 2. настроить split-dns
    - клиент1 - видит обе зоны, но в зоне dns.lab только web1
    - клиент2 видит только dns.lab

  - Дополнительное задание
    - настроить все без выключения selinux
---
## Введение:
<details>

DNS(Domain Name System, Служба доменных имён) -  это распределенная система, для получения информации о доменах. DNS используется для сопоставления IP-адресов и доменных имён.
Сопостовления IP-адресов и DNS-имён бывают двух видов: 
- Прямое (DNS-bмя в IP-адрес)
- Обратное (IP-адрес в DNS-имя)

Доменная структура DNS представляет собой древовидную иерархию, состоящую из узлов, зон, доменов, поддоменов и т.д. «Вершиной» доменной структуры является корневая зона. Корневая (root) зона обозначается точкой. Далее следуют домены первого уровня (.com, ,ru, .org и т. д.) и т д.

В DNS встречаются понятия зон и доменов:
- Зона — это любая часть дерева системы доменных имён, размещаемая как единое целое на некотором DNS-сервере. 
- Домен – определенный узел, включающий в себя все подчинённые узлы. 

Давайте разберем основное отличие зоны от домена. Возьмём для примера ресурс otus.ru — это может быть сразу и зона и домен, однако, при использовании зоны otus.ru мы можем сделать отдельную зону mail.otus.ru, которая будет управляться не нами. В случае домена так сделать нельзя...

FQDN (Fully Qualified Domain Name) - полностью указанное доменное имя, т.е. от корневого домена. Ключевой идентификатор FQDN - точка в конце имени. Максимальный размер FQDN — 255 байт, с ограничением в 63 байта на каждое имя домена. Пример FQDN: mail.otus.ru.

Вся информация о DNS-ресурсах хранится в ресурсных записях. Записи хранят следующие атрибуты:
- Имя (NAME) - доменное имя, к которому привязана или которому принадлежит данная ресурсная область, либо IP-адрес. При отсутствии данного поля, запись ресурса наследуется от предыдущей записи. 
- TTL (время жизни в кэше) - после указанного времени запись удаляется, данное поле может не указываться в индивидуальных записях ресурсов, но тогда оно должно быть указано в начале файла зоны и будет наследоваться всеми записями.
- Класс (CLASS) - определяет тип сети (в 99% используется IN - интернет)
- Тип (TYPE) - тип записи, синтаксис и назначение записи
- Значение (DATA)  

Типы рекурсивных записей:
- А (Address record) - отображают имя хоста (доменное имя) на адрес IPv4
- AAAA - отображает доменное имя на адрес IPv6
- CNAME (Canonical name record/псевдоним) - привязка алиаса к существующему доменному имени
- MX (mail exchange) - указывает хосты для отправки почты, адресованной домену. При этом поле NAME указывает домен назначения, а поле DATA приоритет и доменное имя хоста, ответственного за приём почты. Данные вводятся через пробел
- NS (name server) -  указывает на DNS-сервер, обслуживающий данный домен. 
- PTR (pointer) -  Отображает IP-адрес в доменное имя 
- SOA (Start of Authority/начальная запись зоны) - описывает основные начальные настройки зоны. 
- SRV (server selection) — указывает на сервера, обеспечивающие работу тех или иных служб в данном домене (например  Jabber и Active Directory).

Для работы с DNS (как клиенту) в linux используют утилиты dig, host и nslookup
Также в Linux есть следующие реализации DNS-серверов:
- bind
- powerdns (умеет хранить зоны в БД)
- unbound (реализация bind)
- dnsmasq 
- и т д. 

Split DNS (split-horizon или split-brain) — это конфигурация, позволяющая отдавать разные записи зон DNS в зависимости от подсети источника запроса. Данную функцию можно реализовать как с помощью одного DNS-сервера, так и с помощью нескольких DNS-серверов… 

</details>
## Выполнение задания:

### Настройка рабочего стенда
<details>
Выполнение домашнего задания предполагает, что на компьютере установлен Vagrant+VirtualBox   

Развернем Vagrant-стенд:
  - Создайте папку с проектом и зайдите в нее (например: /otus_dns):
```
mkdir -p otus_systemd ; cd ./otus_dns
```
  - Клонируете проект с Github, набрав команду:
```
apt update -y && apt install git -y ; git clone https://github.com/pahami/otus_dns.git
```
  - Запустите проект из папки, в которую склонировали проект (в нашем примере ./otus_dns):
```
vagrant up
```
Результатом выполнения команды `vagrant up` станут 4 виртуальных машины:
 - ns01 - основной DNS сервер (ip: 192.168.50.10)
 - ns02 - дублирующий DNS сервер (ip: 192.168.50.11)
 - client - первый хост, может идеть запись web1.dns.lab и не видеть запись web2.dns.lab (ip: 192.168.50.15)
 - client2 - второй хост, может видеть обе записи из домена dns.lab, но не должен видеть записи домена newdns.lab (ip: 192.168.50.16)

Требуемые нам файлы:
 - playbook.yml — это Ansible-playbook, в котором содержатся инструкции по настройке нашего стенда
 - client-motd — файл, содержимое которого будет появляться перед пользователем, который подключился по SSH
 - named.ddns.lab и named.dns.lab — файлы описания зон ddns.lab и dns.lab соответсвенно
 - master-named.conf и slave-named.conf — конфигурационные файлы, в которых хранятся настройки DNS-сервера
 - client-resolv.conf и servers-resolv.conf — файлы, в которых содержатся IP-адреса DNS-серверов
</summary>
#### Задание №1

  - Добавить еще один сервер client2
---
Добавляем в Vagrantfile запись о хосте client2
```
  config.vm.define "client2" do |client2|
    client2.vm.network "private_network", ip: "192.168.50.16", virtualbox__intnet: "dns"
    client2.vm.hostname = "client2"
  end
```
Настроили NTP-клиент Chrony, добавив запись в плейбук. Это нужно для синхронизации времени на всех хостах
```
- name: install packages
    yum: 
      name:
        - bind
        - bind-utils
        - vim
      state: latest
      update_cache: true      
  
  - name: start chronyd
    service: 
      name: chronyd
      state: restarted
      enabled: true

```
Нам нужно подкорректировать файл /etc/resolv.conf для DNS-серверов: на хосте ns01 указать nameserver 192.168.50.10, а на хосте ns02 — 192.168.50.11. В Ansible для этого можно воспользоваться шаблоном с Jinja. Изменим имя файла servers-resolv.conf на servers-resolv.conf.j2 и укажем там следующие условия:
```
domain dns.lab
search dns.lab
#Если имя сервера ns02, то указываем nameserver 192.168.50.11
{% if ansible_hostname == 'ns02' %}
nameserver 192.168.50.11
{% endif %}
#Если имя сервера ns01, то указываем nameserver 192.168.50.10
{% if ansible_hostname == 'ns01' %}
nameserver 192.168.50.10
{% endif %} 
```
После внесение изменений в файл, внесём измения в ansible-playbook:
Используем вместо модуля copy модуль template:
```
- name: copy resolv.conf to the servers
  template: src=servers-resolv.conf.j2 dest=/etc/resolv.conf owner=root group=root mode=0644
```
---
  - завести в зоне dns.lab имена (web1 - client, web2 - client2):
---
Нам необходимо добавить стоки с новыми именами в файл named.dns.lab
```
;Web
web1            IN      A       192.168.50.15
web2            IN      A       192.168.50.16
```
---
  - завести еще одну зону newdns.lab
---
Для создания зоны и добавления в неё записей, добавляем зону в файл /etc/named.conf на хостах ns01 и ns02, а также создаем файл named.newdns.lab, который далее отправим на сервер ns01.

Добавление записей в master-named.conf и slave-named.conf
```
// lab's newdns zone
zone "newdns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    allow-update { key "zonetransfer.key"; };
    file "/etc/named/named.newdns.lab";
```
Добавим в модуль copy наш файл named.newdns.lab:
```
- name: copy zones
    copy: src={{ item }} dest=/etc/named/ owner=root group=named mode=0660
    with_fileglob:
      - named.d*
      - named.newdns.lab
```
---

#### Задание №2

Создаём дополнительный файл зоны dns.lab, в котором будет прописана только одна запись named.dns.lab.client
---
```
$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201407 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11

;Web
web1            IN      A       192.168.50.15
```
---
Внести изменения в файл /etc/named.conf на хостах ns01 и ns02
---
- Сначала сгенерируем ключи для хостов client и client2, для этого на хосте ns01 запустим утилиту tsig-keygen (ключ может генериться 5 минут и более): 
```
[root@ns01 ~]# tsig-keygen
key "tsig-key" {
	algorithm hmac-sha256;
	secret "IxAIDfcewAtWxE5NnD54HBwXvX4EFThcW1o0DqL15oI=";
};
[root@ns01 ~]# 
```
- После их генерации добавим блок с access листами в конец файла /etc/named.conf
```
#Описание ключа для хоста client
key "client-key" {
    algorithm hmac-sha256;
    secret "IQg171Ht4mdGYcjjYKhI9gSc1fhoxzHZB+h2NMtyZWY=";
};
#Описание ключа для хоста client2
key "client2-key" {
    algorithm hmac-sha256;
    secret "m7r7SpZ9KBcA4kOl1JHQQnUiIlpQA1IJ9xkBHwdRAHc=";
};
#Описание access-листов
acl client { !key client2-key; key client-key; 192.168.50.15; };
acl client2 { !key client-key; key client2-key; 192.168.50.16; };
```
- Добавляем ACL и View внеся соответствующие пункты в master-named.conf и slave-named.conf
```
// Указание Access листов 
acl client { !key client2-key; key client-key; 192.168.50.15; };
acl client2 { !key client-key; key client2-key; 192.168.50.16; };
// Настройка первого view 
view "client" {
    // Кому из клиентов разрешено подключаться, нужно указать имя access-листа
    match-clients { client; };

    // Описание зоны dns.lab для client
    zone "dns.lab" {
        // Тип сервера — мастер
        type master;
        // Добавляем ссылку на файл зоны, который создали в прошлом пункте
        file "/etc/named/named.dns.lab.client";
        // Адрес хостов, которым будет отправлена информация об изменении зоны
        also-notify { 192.168.50.11 key client-key; };
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type master;
        file "/etc/named/named.newdns.lab";
        also-notify { 192.168.50.11 key client-key; };
    };
};

// Описание view для client2
view "client2" {
    match-clients { client2; };

    // dns.lab zone
    zone "dns.lab" {
        type master;
        file "/etc/named/named.dns.lab";
        also-notify { 192.168.50.11 key client2-key; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "/etc/named/named.dns.lab.rev";
        also-notify { 192.168.50.11 key client2-key; };
    };
};

// Зона any, указана в файле самой последней
view "default" {
    match-clients { any; };

```
---
Для работу dns c помощью ping на хостах client, client2
<details>
<summary> client2 </summary>
---
[vagrant@client2 ~]$ ping www.newdns.lab
ping: www.newdns.lab: Name or service not known
[vagrant@client2 ~]$ ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=1 ttl=64 time=0.457 ms
^C
--- web1.dns.lab ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.457/0.457/0.457/0.000 ms
[vagrant@client2 ~]$  ping web2.dns.lab
PING web2.dns.lab (192.168.50.16) 56(84) bytes of data.
64 bytes from client2 (192.168.50.16): icmp_seq=1 ttl=64 time=0.018 ms
64 bytes from client2 (192.168.50.16): icmp_seq=2 ttl=64 time=0.046 ms
^C
--- web2.dns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.018/0.032/0.046/0.014 ms
---
<summary> client </summary>
---
[vagrant@client ~]$ ping www.newdns.lab
PING www.newdns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.007 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.039 ms
^C
--- www.newdns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.007/0.023/0.039/0.016 ms
[vagrant@client ~]$ ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.020 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.038 ms
^C
--- web1.dns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.020/0.029/0.038/0.009 ms
[vagrant@client ~]$ ping web2.dns.lab
ping: web2.dns.lab: Name or service not known
---
</details>
