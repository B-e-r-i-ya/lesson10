# lesson10

## Создаем Vagrantfile


Cоздаем `vagrantfile` по средствам которого создается 2 машины с именем `Host1` `Host2`.

```
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :host1 => {
        :box_name => "centos/7",
        :ip_addr => '192.168.11.150'
  },
  :host2 => {
        :box_name => "centos/7",
        :ip_addr => '192.168.11.151'
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s
          box.vm.network "private_network", ip: boxconfig[:ip_addr]
          box.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", "200"]
          end
..........
          box.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
            sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
            systemctl restart sshd
          SHELL
=begin.
          box.vm.provision "ansible" do |ansible|
            ansible.verbose = "vv"
            ansible.playbook = "provision/playbook.yml"
            ansible.become = "true"
          end
=end
      end
  end
end


```


##Запускаем стенд

```
#vagrant up
```

## Проверяем наличие на стенде web сервера

```
#curl http://192.168.11.150
#curl http://192.168.11.151
#curl http://192.168.11.150:8080
#curl http://192.168.11.151:8080

```

Убедившись что на хостах отсутствует утановленный веб сервер, приступаем к настройке.

#Ansible

Для удобства создаем дерикторию `Ansible`

```
#mkdir Ansible
#cd Ansible/
```

Внутри создаем еще 2 директории  `Playbook` и `roles`

переходим в каталог `roles`

```
#cd roles/
```

Создаем роль
```
#ansible-galaxy init nginx
```


```
/ansible/roles# tree
.
└── nginx
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── README.md
    ├── tasks
    │   └── main.yml
    ├── templates
    │   ├── index.html.j2
    │   └── nginx.conf.j2
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml

```

В каталоге Ansible создаем инвентори файл `ALL.yml`

```
---
all:
  children:
    nginx:
  vars:
    ansible_user: 'vagrant'


nginx:
 hosts:
  nginx1:
   ansible_host: 127.0.0.1
   ansible_port: 2222
   ansible_ssh_private_key_file: /root/otus/lesson10/.vagrant/machines/host1/virtualbox/private_key
  nginx2:
   ansible_host: 127.0.0.1
   ansible_port: 2200
   ansible_ssh_private_key_file: /root/otus/lesson10/.vagrant/machines/host2/virtualbox/private_key

```

В каталоге `Playbook` создаем файл nginx.yml

```
---
- name: Install Nginx
  hosts: nginx
  become: true
  roles:
   - ../roles/nginx


```

Далее идем настраивать роль:

переходим в каталог `roles/nginx` и по порядку заполняем файлы

## defaults - переменные в файле `main.yml `


```
---
# defaults file for nginx
nginx_listen_port: 8080

```

## filles -  каталог с файлами для копирования
 

## hanlers

```
---
# handlers file for nginx

- name: restart nginx
  systemd:
   name: nginx
   state: restarted
   enabled: yes

- name: reload nginx
  systemd:
   name: nginx
   state: reloaded

```

## meta - ???

## README.md - описываем нашу роль

## tasks - описываем наши таски (первоначально закоментируем изменение конфига веб сервера)

```
# tasks file for nginx
- name: NGINX | Install EPEL Repo package from standart repo
  yum:
   name: epel-release
   state: present
  tags:
   - epel-package
   - packages

- name: NGINX | Install NGINX package from EPEL Repo
  yum:
   name: nginx
   state: latest
  notify:
   - restart nginx
  tags:
   - nginx-package
   - packages

#- name: NGINX | Create NGINX config file from template
#  template:
#   src: nginx.conf.j2
#   dest: /etc/nginx/nginx.conf
#  notify:
#   - reload nginx
#  tags:
#   - nginx-configuration


```


## templates - складываем шаблоны файлов, в нашем случае nginx.conf.j2

```
events {
 worker_connections 1024;
}
http {
 server {
 listen {{ nginx_listen_port }} default_server;
 server_name default_server;
 root /usr/share/nginx/html;
 location / {
 }
 }
}

```


#Первый Запуск:

Переходим в каталог  `ansible`

```
/ansible# ansible-playbook -i ALL.yml ./Playbook/nginx.yml
```

после выполнения проверяем доступность веб сервера по 80 порту, после чего расскоментируем task 
по изменению конфига, запусим снова playbook и по окончании проверяем доступность по порту 8080

# УРА все работает