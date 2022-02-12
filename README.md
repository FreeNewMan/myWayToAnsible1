# Домашнее задание к занятию "08.02 Работа с Playbook"
```
- name: Install Java #Установка Java
  hosts: all #на всех хостах
  tasks:
    - name: Set facts for Java 11 vars 
      set_fact:
        java_home: "/opt/jdk/{{ java_jdk_version }}" #настройка значения системной переменной окружения JAVA_HOME
      tags: java
    - name: Upload .tar.gz file containing binaries from local storage
      copy:
        src: "{{ java_oracle_jdk_package }}" #Загрузка на целевой хост архива с JAVA (откуда)
        dest: "/tmp/jdk-{{ java_jdk_version }}.tar.gz" #(куда, на настраиваемый хост)
      register: download_java_binaries
      until: download_java_binaries is succeeded #Ожидание окончание загрузки файла на хост перед тем как идти дальше
      tags: java
    - name: Ensure installation dir exists #Проверка существования директории по пути JAVA_HOME
      become: true
      file:
        state: directory
        path: "{{ java_home }}"
      tags: java
    - name: Extract java in the installation directory #Распаковка архива с JAVA 
      become: true
      unarchive:
        copy: false
        src: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"
        dest: "{{ java_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ java_home }}/bin/java"
      tags:
        - java
    - name: Export environment variables #Добавление переменных окружения в профиль пользователя 
      become: true
      template:
        src: jdk.sh.j2
        dest: /etc/profile.d/jdk.sh
      tags: java
- name: Install Elasticsearch #Установка Elastic
  hosts: elasticsearch
  tasks:
    - name: Upload tar.gz Elasticsearch from remote URL
      get_url:
        url: "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        dest: "/tmp/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        mode: 0755 #Права доступа на папку
        timeout: 60 #Перерыв между запроса на скачивание при неудачной попытке 
        force: true #Всегда скачивать
        validate_certs: false  # не проверять сертификаты
      register: get_elastic #результат скачивания записать в переменную get_elastic
      until: get_elastic is succeeded #ждать пока не скачается
      tags: elastic
    - name: Create directrory for Elasticsearch #Создание директории для эластика
      become: true
      file:
        state: directory
        path: "{{ elastic_home }}"
        owner: "{{ ansible_user }}" #установка владельца
        group: "{{ ansible_user }}" #установка группы
        recurse: true
      tags: elastic
    - name: Extract Elasticsearch in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        dest: "{{ elastic_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ elastic_home }}/bin/elasticsearch" #Если уже существует файл elasticsearch, то не расыпаковывать
      tags:
        - elastic
    - name: Set environment Elastic #Запись переменных окружения в настройки профиль пользователя
      become: true
      template:
        src: templates/elk.sh.j2
        dest: /etc/profile.d/elk.sh
      tags: elastic
    - name: Set config for Elastic #Настройка конфигурационного файла для Elasticsearch
      become: true
      template:
        src: elasticsearch.yml.j2
        dest: "{{ elastic_config }}/elasticsearch.yml"
      tags: elastic        
- name: Install Kibana #Установка Kibana
  hosts: kibana
  tasks:
    - name: Upload tar.gz Kibana from remote URL #Скачивание архива и загрузка на целевой хост
      get_url:
        url: "https://artifacts.elastic.co/downloads/kibana/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        dest: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        mode: 0755
        timeout: 60
        force: true
        validate_certs: false
      register: get_kibana
      until: get_kibana is succeeded #Дождаться скачивания
      tags: kibana
    - name: Create directrory for Kibana #Создание каталога для Kibana c назначением владельца и группы
      become: true
      file:
        state: directory
        path: "{{ kibana_home }}"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        recurse: true
    - name: Create data directrory for Kibana #Создание каталога для данных с указанием владельца и его группы
      become: true
      file:
        state: directory
        path: "{{ kibana_home }}/data"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        recurse: true
      tags: kibana
    - name: Extract kibana in the installation directory #Распаковка архива
      become: true
      unarchive:
        copy: false
        src: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        dest: "{{ kibana_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ kibana_home }}/bin/kibana"
      tags:
        - kibana
    - name: Set environment Kibana #Настройка системных переменных в профиле пользователя
      become: true
      template:
        src: templates/kib.sh.j2
        dest: /etc/profile.d/kib.sh
      tags: kibana
    - name: Set config for Kibana #Настройка для Kibana
      become: true
      template:
        src: kibana.yml.j2
        dest: "{{ kibana_config }}/kibana.yml"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"            
      tags:
        - kibana        
- name: Install Logstash
  hosts: logstash
  tasks:
    - name: Upload tar.gz Logstash from remote URL
      get_url:
        url: "https://artifacts.elastic.co/downloads/logstash/logstash-{{ logstash_version }}-linux-x86_64.tar.gz"
        dest: "/tmp/logstash-{{ logstash_version }}-linux-x86_64.tar.gz"
        mode: 0755
        timeout: 60
        force: true
        validate_certs: false
      register: get_logstash
      until: get_logstash is succeeded
      tags: logstash
    - name: Create directrory for logstash
      become: true
      file:
        state: directory
        path: "{{ logstash_home }}"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        recurse: true
    - name: Create data directrory for logstash
      become: true
      file:
        state: directory
        path: "{{ logstash_home }}/data"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: 0755
        recurse: true
      tags: logstash
    - name: Extract logstash in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/logstash-{{ logstash_version }}-linux-x86_64.tar.gz"
        dest: "{{ logstash_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ logstash_home }}/bin/logstash"
      tags:
        - logstash
    - name: Set environment logstash
      become: true
      template:
        src: templates/logst.sh.j2
        dest: /etc/profile.d/logst.sh
      tags: logstash
    - name: Set config for logstash
      become: true
      template:
        src:  myconf.conf.j2
        dest: "{{ logstash_config }}/myconf.conf"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"        
      tags:
        - logstash
        - logstash_cfg        
```


---
