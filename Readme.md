Vagrant-стенд c LDAP на базе FreeIPA

Цель домашнего задания
Научиться настраивать LDAP-сервер и подключать к нему LDAP-клиентов

Описание домашнего задания
1) Установить FreeIPA
2) Написать Ansible-playbook для конфигурации клиента

В данной лабораторной работе будет рассмотрена установка и настройка FreeIPA. FreeIPA — это готовое решение, включающее в себе:
Сервер LDAP на базе Novell 389 DS c предустановленными схемами
Сервер Kerberos
Предустановленный BIND с хранилищем зон в LDAP
Web-консоль управления

Создадим Vagrantfile, в котором будут указаны параметры наших ВМ:

```
Vagrant.configure("2") do |config|
    # Указываем ОС, версию, количество ядер и ОЗУ
    config.vm.box = "centos/7"

    config.vm.provider :virtualbox do |v|
      v.memory = 2048
      v.cpus = 1
    end

    # Указываем имена хостов и их IP-адреса
    boxes = [
      { :name => "ipa.otus.lan",
        :ip => "192.168.57.10",
      },
      { :name => "client1.otus.lan",
        :ip => "192.168.57.11",
      },
      { :name => "client2.otus.lan",
        :ip => "192.168.57.12",
      }
    ]
    # Цикл запуска виртуальных машин
    boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.network "private_network", ip: opts[:ip]
      end
    end
  end
```

После создания Vagrantfile, запустим виртуальные машины командой vagrant up. Будут созданы 3 виртуальных машины с ОС CentOS 8 Stream. Каждая ВМ будет иметь по 2ГБ ОЗУ и по одному ядру CPU. 

```
neva@Uneva:~/Otus_Kaneva_dz25$ vagrant status
Current machine states:

ipa.otus.lan              running (virtualbox)
client1.otus.lan          running (virtualbox)
client2.otus.lan          running (virtualbox)
```

1) Установка FreeIPA сервера

Для начала нам необходимо настроить FreeIPA-сервер. Подключимся к нему по SSH с помощью команды:

```
neva@Uneva:~/Otus_Kaneva_dz25$ vagrant ssh ipa.otus.lan
[vagrant@ipa ~]$ sudo -i
```

Начнём настройку FreeIPA-сервера: 
Установим часовой пояс:

```
[root@ipa ~]# timedatectl set-timezone Europe/Moscow
```

Установим утилиту chrony:

```
[root@ipa ~]# yum install -y chrony
```

Запустим chrony и добавим его в автозагрузку:

```
[root@ipa ~]# systemctl enable chronyd
```

Выключим Firewall: 

```
[root@ipa ~]# systemctl stop firewalld
```

Отключим автозапуск Firewalld:

```
[root@ipa ~]# systemctl disable firewalld
```

Остановим Selinux и отключим его на постоянной основе:

```
[root@ipa ~]# setenforce 0
[root@ipa ~]# vi /etc/selinux/config
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

Для дальнейшей настройки FreeIPA нам потребуется, чтобы DNS-сервер хранил запись о нашем LDAP-сервере. В рамках данной лабораторной работы мы не будем настраивать отдельный DNS-сервер и просто добавим запись в файл /etc/hosts

```
[root@ipa ~]# vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
127.0.1.1 ipa.otus.lan ipa
192.168.57.10 ipa.otus.lan ipa
```

Установим модуль DL1:

```
[root@ipa ~] yum install ipa-server -y
```

Запустим скрипт установки:

```
[root@ipa ~]# yum update nss
[root@ipa ~]# ipa-server-install
[root@ipa ~]# ipa-server-install

The log file for this installation can be found in /var/log/ipaserver-install.log
==============================================================================
This program will set up the IPA Server.

This includes:
  * Configure a stand-alone CA (dogtag) for certificate management
  * Configure the Network Time Daemon (ntpd)
  * Create and configure an instance of Directory Server
  * Create and configure a Kerberos Key Distribution Center (KDC)
  * Configure Apache (httpd)
  * Configure the KDC to enable PKINIT

To accept the default shown in brackets, press the Enter key.

WARNING: conflicting time&date synchronization service 'chronyd' will be disabled
in favor of ntpd

Do you want to configure integrated DNS (BIND)? [no]: no

Enter the fully qualified domain name of the computer
on which you're setting up server software. Using the form
<hostname>.<domainname>
Example: master.example.com.


Server host name [ipa.otus.lan]:

The domain name has been determined based on the host name.

Please confirm the domain name [otus.lan]:

The kerberos protocol requires a Realm name to be defined.
This is typically the domain name converted to uppercase.

Please provide a realm name [OTUS.LAN]:
Certain directory server operations require an administrative user.
This user is referred to as the Directory Manager and has full access
to the Directory for system management tasks and will be added to the
instance of directory server created for IPA.
The password must be at least 8 characters long.

Directory Manager password:
Password (confirm):

The IPA server requires an administrative user, named 'admin'.
This user is a regular system account used for IPA server administration.

IPA admin password:
Password (confirm):


The IPA Master Server will be configured with:
Hostname:       ipa.otus.lan
IP address(es): 192.168.57.10
Domain name:    otus.lan
Realm name:     OTUS.LAN

Continue to configure the system with these values? [no]: yes

The following operations may take some minutes to complete.
Please wait until the prompt is returned.


The ipa-client-install command was successful
```

После успешной установки FreeIPA, проверим, что сервер Kerberos может выдать нам билет: 

```
[root@ipa ~]# kinit admin
Password for admin@OTUS.LAN:
[root@ipa ~]# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: admin@OTUS.LAN

Valid starting     Expires            Service principal
09/25/23 17:27:15  09/26/23 17:27:09  krbtgt/OTUS.LAN@OTUS.LAN
```

Для удаление полученного билета воспользуемся командой:

```
[root@ipa ~]# kdestroy
```

Мы можем зайти в Web-интерфейс нашего FreeIPA-сервера, для этого на нашей хостой машине нужно прописать следующую строку в файле Hosts:
192.168.57.10 ipa.otus.lan

После добавления DNS-записи откроем c нашей хост-машины веб-страницу: https://ipa.otus.lan и увидим там консоль управления - [рис. 1](https://github.com/zoyqqyoz/Otus_Kaneva_dz25/blob/master/ipa.JPG).
В имени пользователя укажем admin, в пароле укажем наш IPA admin password и нажмём войти. 
На этом установка и настройка FreeIPA-сервера завершена.

2. Ansible playbook для конфигурации клиента

Настройка клиента похожа на настройку сервера. На хосте также нужно:
Настроить синхронизацию времени и часовой пояс
Настроить (или выключить) firewall
Настроить (или выключить) SElinux
В файле hosts должна быть указана запись с FreeIPA-сервером и хостом

Хостов, которые требуется добавить к серверу может быть много, для упращения нашей работы выполним настройки с помощью Ansible:

В каталоге с нашей лабораторной работой создадим каталог Ansible:

```
neva@Uneva:~/Otus_Kaneva_dz25$ mkdir ansible
```

В каталоге ansible создадим файл hosts со следующими параметрами:

```
[clients]
client1.otus.lan ansible_host=192.168.57.11 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/client1.otus.lan/virtualbox/private_key
client2.otus.lan ansible_host=192.168.57.12 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/client2.otus.lan/virtualbox/private_key
```

Далее создадим файл provision.yml в котором непосредственно будет выполняться настройка клиентов: 

```
- name: Base set up
  hosts: all
  #Выполнять действия от root-пользователя
  become: yes
  tasks:
  #Установка текстового редактора Vim и chrony
  - name: install softs on CentOS
    yum:
      name:
        - vim
        - chrony
      state: present
      update_cache: true
  
  #Отключение firewalld и удаление его из автозагрузки
  - name: disable firewalld
    service:
      name: firewalld
      state: stopped
      enabled: false
  
  #Отключение SElinux из автозагрузки
  #Будет применено после перезагрузки
  - name: disable SElinux
    selinux:
      state: disabled
  
  #Отключение SElinux до перезагрузки
  - name: disable SElinux now
    shell: setenforce 0

  #Установка временной зоны Европа/Москва    
  - name: Set up timezone
    timezone:
      name: "Europe/Moscow"
  
  #Запуск службы Chrony, добавление её в автозагрузку
  - name: enable chrony
    service:
      name: chronyd
      state: restarted
      enabled: true
  
  #Копирование файла /etc/hosts c правами root:root 0644
  - name: change /etc/hosts
    template:
      src: hosts.j2
      dest: /etc/hosts
      owner: root
      group: root
      mode: 0644
  
  #Установка клиента Freeipa
  - name: install module ipa-client
    yum:
      name:
        - freeipa-client
      state: present
      update_cache: true
  
  #Запуск скрипта добавления хоста к серверу
  - name: add host to ipa-server
    shell: echo -e "yes\nyes" | ipa-client-install --mkhomedir --domain=OTUS.LAN --server=ipa.otus.lan --no-ntp -p admin -w otus2022
```

Template файла /etc/hosts выглядит следующим образом:

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.57.10 ipa.otus.lan ipa
```
Почти все модули нам уже знакомы, давайте подробнее остановимся на последней команде echo -e "yes\nyes" | ipa-client-install --mkhomedir --domain=OTUS.LAN --server=ipa.otus.lan --no-ntp -p admin -w otus2022

При добавлении хоста к домену мы можем просто ввести команду ipa-client-install и следовать мастеру подключения к FreeIPA-серверу (как было в первом пункте).

Однако команда позволяет нам сразу задать требуемые нам параметры:
--domain — имя домена
--server — имя FreeIPA-сервера
--no-ntp — не настраивать дополнительно ntp (мы уже настроили chrony)
-p — имя админа домена
-w — пароль администратора домена (IPA password)
--mkhomedir — создать директории пользователей при их первом логине

Если мы сразу укажем все параметры, то можем добавить эту команду в Ansible и автоматизировать процесс добавления хостов в домен. 

Выполним команду ansible-playbook provision.yml  и проверим результат.

```
neva@Uneva:~/Otus_Kaneva_dz25/ansible$ ansible-playbook provision.yml
[DEPRECATION WARNING]: COMMAND_WARNINGS option, the command warnings feature is being removed. This feature will be removed from ansible-core in version 2.14. Deprecation warnings can be disabled by setting deprecation_warnings=False in
 ansible.cfg.

PLAY [Base set up] **************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************************************************************
ok: [client1.otus.lan]
ok: [client2.otus.lan]

TASK [install softs on CentOS] **************************************************************************************************************************************************************************************************************
ok: [client2.otus.lan]
ok: [client1.otus.lan]

TASK [disable firewalld] ********************************************************************************************************************************************************************************************************************
ok: [client1.otus.lan]
ok: [client2.otus.lan]

TASK [disable SElinux] **********************************************************************************************************************************************************************************************************************
[WARNING]: SELinux state change will take effect next reboot
ok: [client2.otus.lan]
ok: [client1.otus.lan]

TASK [disable SElinux now] ******************************************************************************************************************************************************************************************************************
changed: [client1.otus.lan]
changed: [client2.otus.lan]

TASK [Set up timezone] **********************************************************************************************************************************************************************************************************************
ok: [client1.otus.lan]
ok: [client2.otus.lan]

TASK [enable chrony] ************************************************************************************************************************************************************************************************************************
changed: [client2.otus.lan]
changed: [client1.otus.lan]

TASK [change /etc/hosts] ********************************************************************************************************************************************************************************************************************
ok: [client1.otus.lan]
ok: [client2.otus.lan]

TASK [install module ipa-client] ************************************************************************************************************************************************************************************************************
ok: [client1.otus.lan]
ok: [client2.otus.lan]

TASK [add host to ipa-server] ***************************************************************************************************************************************************************************************************************
changed: [client2.otus.lan]
changed: [client1.otus.lan]

PLAY RECAP **********************************************************************************************************************************************************************************************************************************
client1.otus.lan           : ok=10   changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
client2.otus.lan           : ok=10   changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Давайте проверим работу LDAP, для этого на сервере FreeIPA создадим пользователя и попробуем залогиниться к клиенту:

Авторизируемся на сервере и создадим пользователя otus-user:

```
[vagrant@ipa ~]$ sudo -i
[root@ipa ~]# kinit admin
Password for admin@OTUS.LAN:
[root@ipa ~]# ipa user-add otus-user --first=Otus --last=User --password
Password:
Enter Password again to verify:
----------------------
Added user "otus-user"
----------------------
  User login: otus-user
  First name: Otus
  Last name: User
  Full name: Otus User
  Display name: Otus User
  Initials: OU
  Home directory: /home/otus-user
  GECOS: Otus User
  Login shell: /bin/sh
  Principal name: otus-user@OTUS.LAN
  Principal alias: otus-user@OTUS.LAN
  User password expiration: 20230926082709Z
  Email address: otus-user@otus.lan
  UID: 1741000001
  GID: 1741000001
  Password: True
  Member of groups: ipausers
  Kerberos keys available: True
```

На хосте client1 или client2 выполним команду kinit otus-user

```
neva@Uneva:~/Otus_Kaneva_dz25/ansible$ vagrant ssh client1.otus.lan
Last login: Tue Sep 26 11:01:59 2023 from 192.168.57.1
[vagrant@client1 ~]$ kinit otus-user
Password for otus-user@OTUS.LAN:
Password expired.  You must change it now.
Enter new password:
Enter it again:
[vagrant@client1 ~]$
```

Система запросит у нас пароль и попросит ввести новый пароль. 

На этом процесс добавления хостов к FreeIPA-серверу завершен.

