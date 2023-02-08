# Домашнее задание к занятию "08.03 Использование Yandex Cloud"

## Подготовка к выполнению

1. (Необязательно) Познакомтесь с [lighthouse](https://youtu.be/ymlrNlaHzIY?t=929)
2. Подготовьте в Yandex Cloud три хоста: для `clickhouse`, для `vector` и для `lighthouse`.
![](/img/Yandex_cloud_for_08-ansible-03.png)
Ссылка на репозиторий LightHouse: https://github.com/VKCOM/lighthouse

## Основная часть

1. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает lighthouse.
```bash
---
- name: Install Clickhouse
  hosts: clickhouse
  handlers:
    - name: Start clickhouse service
      become: true
      ansible.builtin.service:
        name: clickhouse-server
        state: restarted
  tasks:
    - block:
        - name: Get clickhouse distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/{{ item }}-{{ clickhouse_version }}.noarch.rpm"
            dest: "./{{ item }}-{{ clickhouse_version }}.rpm"
          with_items: "{{ clickhouse_packages }}"
      rescue:
        - name: Get clickhouse distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-{{ clickhouse_version }}.x86_64.rpm"
            dest: "./clickhouse-common-static-{{ clickhouse_version }}.rpm"
    - name: Install clickhouse packages
      become: true
      ansible.builtin.yum:
        name:
          - clickhouse-common-static-{{ clickhouse_version }}.rpm
          - clickhouse-client-{{ clickhouse_version }}.rpm
          - clickhouse-server-{{ clickhouse_version }}.rpm
      notify: Start clickhouse service
    - name: Flush handlers
      meta: flush_handlers
    - name: Create database
      ansible.builtin.command: "clickhouse-client -q 'create database logs;'"
      register: create_db
      failed_when: create_db.rc != 0 and create_db.rc !=82
      changed_when: create_db.rc == 0
    - name: Get Vector distrib
      ansible.builtin.get_url:
        url: "https://packages.timber.io/vector/0.21.1/vector-0.21.1-1.{{ ansible_architecture }}.rpm"
        dest: "./vector-0.21.1-1.{{ ansible_architecture }}.rpm"
        mode: 0644
    - name: Install Vector packages
      become: true
      ansible.builtin.yum:
        name: vector-0.21.1-1.{{ ansible_architecture }}.rpm
        state: present
      notify: Start Vector service
    - name: Deploy config Vector
      ansible.builtin.template:
        src: vector.j2
        dest: "{{ vector_config_path }}"
        mode: 0644
        owner: "{{ ansible_user_id }}"
        group: "{{ ansible_user_gid }}"
        validate: vector validate --no-environment --config-yaml %s
      become: true
      notify: Start Vector service
    - name: Creates directory
      become: true
      file:
        path: /var/lib/vector/local_logs
        state: directory
        owner: "{{ ansible_user_id }}"
        group: "{{ ansible_user_gid }}"
        mode: 0644
    - name: Create systemd unit Vector
      become: true
      template:
        src: vector.service.j2
        dest: /etc/systemd/system/vector.service
        mode: 0644
        owner: "{{ ansible_user_id }}"
        group: "{{ ansible_user_gid }}"
    - name: Start Vector service
      become: true
      systemd:
        name: vector
        state: started
        daemon_reload: true
    - name: Install lighthouse
      hosts: lighthouse
      handlers:
    - name: Start nginx service
      become: true
      ansible.builtin.service:
        name: nginx
        state: restarted
    pre_tasks:
    - name: Install epel-release | Install Nginx
      become: true
      yum:
        name: epel-release
        state: present
    - name: Install Nginx | Install Nginx
      become: true
      yum:
        name: nginx
        state: present
      notify: Start nginx service
    - name: Create Nginx config | Install Nginx
      become: true
      template:
        src: nginx.j2
        dest: /etc/nginx/nginx.conf
        mode: 0644
      notify: Start nginx service
      post_tasks:
    - name: Stop FW
      become: true
      ansible.builtin.service:
        name: firewalld
        state: stopped
    - name: Show connect URL lighthouse
      debug:
        msg: "http://{{ ansible_host }}/#http://{{ hostvars['clickhouse-01'].ansible_host }}:8123/?user={{ clickhouse_user }}"    

```
2. При создании tasks рекомендую использовать модули: `get_url`, `template`, `yum`, `apt`.
3. Tasks должны: скачать статику lighthouse, установить nginx или любой другой webserver, настроить его конфиг для открытия lighthouse, запустить webserver.
4. Приготовьте свой собственный inventory файл `prod.yml`.
```bash
---
clickhouse:
  hosts:
    clickhouse-01:
      ansible_host: 51.250.7.18


vector:
  hosts:
    vector-01:
      ansible_host: 51.250.4.189


lighthouse:
  hosts:
    lighthouse-01:
      ansible_host: 51.250.64.210
```
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
```bash
alexg@alexg-PC:~/PycharmProjects/devops-netology/MNT-21/08-ansible-03-yandex/playbook$ ansible-lint site.yml
WARNING  Overriding detected file kind 'yaml' with 'playbook' for given positional argument: site.yml

```
6. Попробуйте запустить playbook на этом окружении с флагом `--check`.
```bash
alexg@alexg-PC:~/PycharmProjects/devops-netology/MNT-21/08-ansible-03-yandex/playbook$ ansible-playbook -i inventory/prod.yml site.yml --check

PLAY [Install clickhouse] ********************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************************
ok: [clickhouse-01]

PLAY [Install vector] ************************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************************
ok: [vector-01]

PLAY [Install lighthouse] ********************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************************
ok: [lighthouse-01]

TASK [Install epel-release | Install Nginx] **************************************************************************************************************************************************************
changed: [lighthouse-01]

TASK [Install Nginx | Install Nginx] *********************************************************************************************************************************************************************
ok: [lighthouse-01]

PLAY RECAP ***********************************************************************************************************************************************************************************************
clickhouse-01              : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
lighthouse-01              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
vector-01                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```
7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
```bash
alexg@alexg-PC:~/PycharmProjects/devops-netology/MNT-21/08-ansible-03-yandex/playbook$ ansible-playbook -i inventory/prod.yml site.yml --diff

PLAY [Install clickhouse] ********************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************************
ok: [clickhouse-01]

PLAY [Install vector] ************************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************************
ok: [vector-01]

PLAY [Install lighthouse] ********************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************************
ok: [lighthouse-01]

TASK [Install epel-release | Install Nginx] **************************************************************************************************************************************************************
changed: [lighthouse-01]

TASK [Install Nginx | Install Nginx] *********************************************************************************************************************************************************************
changed: [lighthouse-01]

TASK [Create Nginx config | Install Nginx] ***************************************************************************************************************************************************************
--- before: /etc/nginx/nginx.conf
+++ after: /home/alexg/.ansible/tmp/ansible-local-231414zzjfle4/tmp4bib1wk7/nginx.j2
@@ -1,84 +1,46 @@
-# For more information on configuration, see:
-#   * Official English Documentation: http://nginx.org/en/docs/
-#   * Official Russian Documentation: http://nginx.org/ru/docs/
+user  root;
+worker_processes  auto;
+worker_priority     -1;
 
-user nginx;
-worker_processes auto;
-error_log /var/log/nginx/error.log;
-pid /run/nginx.pid;
-
-# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
-include /usr/share/nginx/modules/*.conf;
+error_log  /var/log/nginx/error.log info;
+pid        /var/run/nginx.pid;
 
 events {
-    worker_connections 1024;
+    worker_connections  2048;
 }
 
 http {
-    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
+    include       /etc/nginx/mime.types;
+    default_type  application/octet-stream;
+
+    log_format  compression  '$remote_addr - $remote_user [$time_local] "$request" '
                       '$status $body_bytes_sent "$http_referer" '
                       '"$http_user_agent" "$http_x_forwarded_for"';
+    access_log  /var/log/nginx/access.log  combined;
 
-    access_log  /var/log/nginx/access.log  main;
+    sendfile on;
+    tcp_nopush on;
+    tcp_nodelay on;
+    keepalive_timeout  65;
+    reset_timedout_connection  on;
+    client_body_timeout        35;
+    send_timeout               30;
 
-    sendfile            on;
-    tcp_nopush          on;
-    tcp_nodelay         on;
-    keepalive_timeout   65;
-    types_hash_max_size 4096;
+    gzip on;
+    gzip_min_length     1000;
+    gzip_vary on;
+    gzip_proxied        expired no-cache no-store private auth;
+    gzip_types          text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript;
+    gzip_disable        "msie6";
 
-    include             /etc/nginx/mime.types;
-    default_type        application/octet-stream;
+    types_hash_max_size 2048;
+    client_max_body_size 512M;
+    proxy_buffer_size   64k;
+    proxy_buffers   4 64k;
+    proxy_busy_buffers_size   64k;
+    server_names_hash_bucket_size 64;
 
-    # Load modular configuration files from the /etc/nginx/conf.d directory.
-    # See http://nginx.org/en/docs/ngx_core_module.html#include
-    # for more information.
+    include /etc/nginx/modules-enabled/*.conf;
     include /etc/nginx/conf.d/*.conf;
-
-    server {
-        listen       80;
-        listen       [::]:80;
-        server_name  _;
-        root         /usr/share/nginx/html;
-
-        # Load configuration files for the default server block.
-        include /etc/nginx/default.d/*.conf;
-
-        error_page 404 /404.html;
-        location = /404.html {
-        }
-
-        error_page 500 502 503 504 /50x.html;
-        location = /50x.html {
-        }
-    }
-
-# Settings for a TLS enabled server.
-#
-#    server {
-#        listen       443 ssl http2;
-#        listen       [::]:443 ssl http2;
-#        server_name  _;
-#        root         /usr/share/nginx/html;
-#
-#        ssl_certificate "/etc/pki/nginx/server.crt";
-#        ssl_certificate_key "/etc/pki/nginx/private/server.key";
-#        ssl_session_cache shared:SSL:1m;
-#        ssl_session_timeout  10m;
-#        ssl_ciphers HIGH:!aNULL:!MD5;
-#        ssl_prefer_server_ciphers on;
-#
-#        # Load configuration files for the default server block.
-#        include /etc/nginx/default.d/*.conf;
-#
-#        error_page 404 /404.html;
-#            location = /40x.html {
-#        }
-#
-#        error_page 500 502 503 504 /50x.html;
-#            location = /50x.html {
-#        }
-#    }
-
-}
-
+    include /etc/nginx/sites-enabled/*;
+}
\ No newline at end of file

changed: [lighthouse-01]

RUNNING HANDLER [Start nginx service] ********************************************************************************************************************************************************************
changed: [lighthouse-01]

TASK [Stop FW] *******************************************************************************************************************************************************************************************
ok: [lighthouse-01]:

PLAY RECAP ***********************************************************************************************************************************************************************************************
clickhouse-01              : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
lighthouse-01              : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
vector-01                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
```bash
alexg@alexg-PC:~/PycharmProjects/devops-netology/MNT-21/08-ansible-03-yandex/playbook$ ansible-playbook -i inventory/prod.yml site.yml --diff

PLAY [Install clickhouse] ********************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************************
ok: [clickhouse-01]

PLAY [Install vector] ************************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************************
ok: [vector-01]

PLAY [Install lighthouse] ********************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************************
ok: [lighthouse-01]

TASK [Install epel-release | Install Nginx] **************************************************************************************************************************************************************
ok: [lighthouse-01]

TASK [Install Nginx | Install Nginx] *********************************************************************************************************************************************************************
ok: [lighthouse-01]

TASK [Create Nginx config | Install Nginx] ***************************************************************************************************************************************************************
ok: [lighthouse-01]

TASK [Stop FW] *******************************************************************************************************************************************************************************************
ok: [lighthouse-01]

PLAY RECAP ***********************************************************************************************************************************************************************************************
clickhouse-01              : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
lighthouse-01              : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
vector-01                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```
9. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
* описание playbook [README.md](./playbook/README.md)

10. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-03-yandex` на фиксирующий коммит, в ответ предоставьте ссылку на него.
* измененый [playbook](./playbook/) 

---
