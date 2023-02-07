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
      roles:
    - role: lighthouse-role
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

```
6. Попробуйте запустить playbook на этом окружении с флагом `--check`.
```bash

```
7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
```bash

```
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
```bash

```
9. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
* описание playbook [README.md](./playbook/README.md)
10. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-03-yandex` на фиксирующий коммит, в ответ предоставьте ссылку на него.
* измененый [playbook](./playbook/) 

---
