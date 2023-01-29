# Домашнее задание к занятию "08.02 Работа с Playbook"

## Подготовка к выполнению

1. (Необязательно) Изучите, что такое [clickhouse](https://www.youtube.com/watch?v=fjTNS2zkeBs) и [vector](https://www.youtube.com/watch?v=CgEhyffisLY)
2. Создайте свой собственный (или используйте старый) публичный репозиторий на github с произвольным именем.
3. Скачайте playbook из репозитория с домашним заданием и перенесите его в свой репозиторий.
4. Подготовьте хосты в соответствии с группами из предподготовленного playbook. Создал виртуальную машину Centos 7 на 192.168.1.74

## Основная часть

1. Приготовьте свой собственный inventory файл `prod.yml`.
```bash
---
clickhouse:
  hosts:
    clickhouse-01:
      ansible_host: 192.168.1.74

vector:
  hosts:
    vector-01:
      ansible_host: 192.168.1.74

```
2. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает vector.
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
    - name: Create database
      ansible.builtin.command: "clickhouse-client -q 'create database logs;'"
      register: create_db
      failed_when: create_db.rc != 0 and create_db.rc !=82
      changed_when: create_db.rc == 0
- name: Install Vector
  hosts: vector
  handlers:
  - name: Start Vector service
    become: true
    ansible.builtin.service:
      name: vector
      state: restarted
  tasks:
    - name: Get Vector distrib
      ansible.builtin.get_url:
        url: "https://packages.timber.io/vector/0.21.1/vector-0.21.1-1.{{ ansible_architecture }}.rpm"
        dest: "./vector-0.21.1-1.{{ ansible_architecture }}.rpm"
    - name: Install Vector packages
      become: true
      ansible.builtin.yum:
        name: vector-0.21.1-1.{{ ansible_architecture }}.rpm
      notify: Start Vector service
    - name: Deploy config Vector
      template:
        src: vector.j2
        dest: /etc/vector/vector.toml
        mode: 0755
      notify: Start Vector service
```
3. При создании tasks рекомендую использовать модули: `get_url`, `template`, `unarchive`, `file`.

4. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, установить vector.

5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
```bash
    ~/git/devops-netology/MNT-21/08-ansible-02-playbook/playbook    main !3    ansible-lint site.yml                                                                                                                                                          ✔ 
WARNING  Overriding detected file kind 'yaml' with 'playbook' for given positional argument: site.yml
WARNING  Listing 8 violation(s) that are fatal
name[missing]: All tasks should be named.
site.yml:11 Task/Handler: block/always/rescue 

risky-file-permissions: File permissions unset or incorrect. (warning)
site.yml:12 Task/Handler: Get clickhouse distrib

risky-file-permissions: File permissions unset or incorrect. (warning)
site.yml:18 Task/Handler: Get clickhouse distrib

jinja[spacing]: Jinja2 spacing could be improved: create_db.rc != 0 and create_db.rc !=82 -> create_db.rc != 0 and create_db.rc != 82 (warning)
site.yml:30 Jinja2 template rewrite recommendation: `create_db.rc != 0 and create_db.rc != 82`.

yaml[indentation]: Wrong indentation: expected 4 but found 2
site.yml:38

risky-file-permissions: File permissions unset or incorrect. (warning)
site.yml:44 Task/Handler: Get Vector distrib

fqcn[action-core]: Use FQCN for builtin module actions (template).
site.yml:53 Use `ansible.builtin.template` or `ansible.legacy.template` instead.

yaml[new-line-at-end-of-file]: No new line character at the end of file
site.yml:58

You can skip specific rules or tags by adding them to your configuration file:
# .config/ansible-lint.yml
warn_list:  # or 'skip_list' to silence them completely
  - experimental  # all rules tagged as experimental
  - fqcn[action-core]  # Use FQCN for builtin actions.
  - name[missing]  # Rule for checking task and play names.
  - yaml[indentation]  # Violations reported by yamllint.
  - yaml[new-line-at-end-of-file]  # Violations reported by yamllint.

                                 Rule Violation Summary                                  
 count tag                           profile    rule associated tags                     
     1 jinja[spacing]                basic      formatting (warning)                     
     1 name[missing]                 basic      idiom                                    
     1 yaml[indentation]             basic      formatting, yaml                         
     1 yaml[new-line-at-end-of-file] basic      formatting, yaml                         
     3 risky-file-permissions        safety     unpredictability, experimental (warning) 
     1 fqcn[action-core]             production formatting                               

Failed after min profile: 4 failure(s), 4 warning(s) on 1 files.

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
10. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-02-playbook` на фиксирующий коммит, в ответ предоставьте ссылку на него.
* измененый [playbook](./playbook/)

---
