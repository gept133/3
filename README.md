# 3


```bash
#!/bin/bash
# Скрипт configure-cloudinfra.sh

export ANSIBLE_CONFIG=/home/altlinux/bin/ansible.cfg

# Создаем структуру каталогов Ansible
mkdir -p /home/altlinux/bin/ansible/{inventory,group_vars,roles}

# Создаем inventory файл
cat > /home/altlinux/bin/ansible/inventory/hosts << EOF
[haproxy]
Cloud-HA01 ansible_host=192.168.1.10
Cloud-HA02 ansible_host=192.168.1.11

[webservers]
Cloud-WEB01 ansible_host=192.168.1.20
Cloud-WEB02 ansible_host=192.168.1.21

[databases]
Cloud-DB01 ansible_host=192.168.1.30
Cloud-DB02 ansible_host=192.168.1.31
EOF

# Создаем переменные для HAProxy и Keepalived
mkdir -p /home/altlinux/bin/ansible/group_vars/haproxy
cat > /home/altlinux/bin/ansible/group_vars/haproxy/vars.yml << EOF
vip: 192.168.1.100
secret_key: "keepalived-secret"
web_servers:
  - Cloud-WEB01:80
  - Cloud-WEB02:80
EOF

# Playbook для HAProxy и Keepalived
cat > /home/altlinux/bin/ansible/ha.yml << EOF
---
- name: Configure HAProxy and Keepalived
  hosts: haproxy
  become: yes
  tasks:
    - name: Install HAProxy
      package:
        name: haproxy
        state: present

    - name: Configure HAProxy
      template:
        src: haproxy.cfg.j2
        dest: /etc/haproxy/haproxy.cfg
      notify: Restart HAProxy

    - name: Install Keepalived
      package:
        name: keepalived
        state: present

    - name: Configure Keepalived
      template:
        src: keepalived.conf.j2
        dest: /etc/keepalived/keepalived.conf
      notify: Restart Keepalived

  handlers:
    - name: Restart HAProxy
      service:
        name: haproxy
        state: restarted

    - name: Restart Keepalived
      service:
        name: keepalived
        state: restarted
EOF

# Шаблон HAProxy
cat > /home/altlinux/bin/ansible/roles/haproxy/templates/haproxy.cfg.j2 << EOF
global
    log /dev/log local0
    user haproxy
    group haproxy

defaults
    mode http
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/selfsigned.pem
    default_backend http_back

backend http_back
    balance roundrobin
    {% for server in web_servers %}
    server {{ server.split(':')[0] }} {{ server }} check
    {% endfor %}
EOF

# Шаблон Keepalived
cat > /home/altlinux/bin/ansible/roles/keepalived/templates/keepalived.conf.j2 << EOF
vrrp_instance VI_1 {
    state {% if inventory_hostname == "Cloud-HA01" %}MASTER{% else %}BACKUP{% endif %}
    interface eth0
    virtual_router_id 51
    priority {% if inventory_hostname == "Cloud-HA01" %}100{% else %}90{% endif %}
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass {{ secret_key }}
    }
    virtual_ipaddress {
        {{ vip }}
    }
}
EOF

# Playbook для веб-серверов
cat > /home/altlinux/bin/ansible/web.yml << EOF
---
- name: Configure Web Servers
  hosts: webservers
  become: yes
  tasks:
    - name: Install Apache and PHP
      package:
        name: "{{ item }}"
        state: present
      loop:
        - apache2
        - php
        - libapache2-mod-php

    - name: Create SSL certificate
      command: |
        openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout /etc/ssl/private/selfsigned.key \
        -out /etc/ssl/certs/selfsigned.pem \
        -subj "/CN=localhost"

    - name: Configure Apache Virtual Host
      template:
        src: default-ssl.conf.j2
        dest: /etc/apache2/sites-available/default-ssl.conf

    - name: Enable SSL module
      command: a2enmod ssl

    - name: Enable SSL site
      command: a2ensite default-ssl

    - name: Restart Apache
      service:
        name: apache2
        state: restarted
EOF

# Playbook для PostgreSQL
cat > /home/altlinux/bin/ansible/db.yml << EOF
---
- name: Configure PostgreSQL
  hosts: databases
  become: yes
  tasks:
    - name: Install PostgreSQL
      package:
        name: postgresql
        state: present

    - name: Configure replication
      template:
        src: postgresql.conf.j2
        dest: /etc/postgresql/14/main/postgresql.conf

    - name: Create replication user
      postgresql_user:
        name: replicator
        password: replicatepass
        role: REPLICATION

    - name: Create test database and user
      postgresql_db:
        name: testdb
      postgresql_user:
        name: testuser
        password: testpassword
        db: testdb
        priv: ALL
EOF

# Настройка Ansible
cat > /home/altlinux/bin/ansible.cfg << EOF
[defaults]
inventory = /home/altlinux/bin/ansible/inventory/hosts
roles_path = /home/altlinux/bin/ansible/roles
EOF

# Установка Ansible
if ! command -v ansible &> /dev/null
then
    sudo apt-get update
    sudo apt-get install -y ansible
fi

# Выполнение playbook'ов
ansible-playbook /home/altlinux/bin/ansible/ha.yml
ansible-playbook /home/altlinux/bin/ansible/web.yml
ansible-playbook /home/altlinux/bin/ansible/db.yml

echo "Настройка инфраструктуры завершена!"
```

Для использования этого решения:

1. Разместите скрипт в `/home/altlinux/bin/configure-cloudinfra.sh`
2. Дайте права на выполнение: `chmod +x /home/altlinux/bin/configure-cloudinfra.sh`
3. Добавьте путь в `.bashrc` или создайте симлинк:
   ```bash
   echo 'export PATH=$PATH:/home/altlinux/bin' >> ~/.bashrc
   source ~/.bashrc
   ```

Скрипт автоматически:
- Установит необходимые зависимости
- Создаст структуру каталогов Ansible
- Настроит балансировку нагрузки и отказоустойчивость
- Развернет веб-серверы с SSL
- Настроит PostgreSQL кластер с репликацией

Перед выполнением убедитесь:
1. На всех целевых узлах настроен SSH доступ с Cloud-ADM
2. В файле inventory указаны правильные IP-адреса
3. Пакетные менеджеры настроены для установки ПО
4. Сетевые настройки разрешают необходимое взаимодействие между узлами
