# Install Postgresql cluster ( etcd, patroni, pgbounce, haproxy, keepalived).
# Ubuntu 22.04.2 LTS
Для автоматизации развертывания используются роли:
- [etc_hosts](#etc_hosts)  
- [etcd](#etcd)
- [postgres](#postgres)  
- [patroni](#patroni)
- [create_user_and_db](#create_user)
- [pgbounce](#images)  

- [Haproxy node](#haproxy_node)
- [add_haproxy](#add_haproxy)
- [keepalive](#keepalive)

<a name="headers"><h2>etc_hosts</h2></a>
В данной роли мы добавляем все хосты которые будем использовать в /etc/hosts. Хосты указаны в /vars/main.yml -> в переменной etc_hosts


<a name="headers"><h2>etcd</h2></a>
- устанавливаем etcd
- создаем директорию по etcd_data с правами и владельцем
- используем шаблон который находиться в templates/etcd.conf.j2 для заполнения /etc/default/etcd. В данном шаблоне используются:
  - vars: etcd_data_dir , etcd_cluster_name
  - gather_facts: ansible_default_ipv4.address , ansible_hostname
  - special variable: invetory_hostname
- Перезапускаем сервис etcd и делаем его enabled

<a name="headers"><h2>postgres</h2></a>
- Добавляем ключ репозитория postgres
- Добавляем сам репозиторий
- Устанавливаем пакет acl
- Останавливаем Postgresql, с последующим использованием handler (удаление кластера)

<a name="headers"><h2>postgres</h2></a>
- Создаем директорию /etc/patroni
- Устанавливаем python
- Устанавливаем lib python psycopg2-binary (понадобиться для взаимодействия ansible c postgres), patroni, python-etcd
- Создаем data dir для patroni c правами и владельцем
- Создаем файл конфиг patroni в /etc/patroni. В данном шаблоне используются:
  - vars: {{ patroni_cluster_name }}, {{ patroni_namespace }}, {{ db_user }}, {{ replicator }}, {{ replicator_passwd }}, {{ superuser }}, {{ superuser_passwd }}
  - gather_facts: {{ ansible_host }}, {{ ansible_default_ipv4.address }},
  - Циклом for заполняется etcd хосты
- Создаем сервис patroni в /etc/systemd/system, используя шаблон patroni.sevice.j2
- Запускаем сервис и включаем автозапуск службы

<a name="headers"><h2>create_user_and_db</h2></a>
- Вызывается данная роль из role/pgbounce/meta/main. Она будет выполняться перед запуском роли pgbounce, т.к. указана в dependencies
- C помощью шела выясняем кто на данный момент являеться лидером и ложим в result
- # Следующий таск, если лидер не определен то ожидает 40 сек а потом повторяет запрос в шеле
- Создается БД с названием переменной {{ db_name }}. C помощью when указываеться что данный таск необходимо исполнить на leader
- Создается user с именем и паролем из данных переменных {{ db_user }}, {{ db_password }}. C помощью when указываеться что данный таск необходимо исполнить на leader
- Выдача прав до БД {{ db_name }} пользователю {{ db_user }}, C помощью when указываеться что данный таск необходимо исполнить на leader

<a name="headers"><h2>pgbounce</h2></a>
- Установка pgbounce
- Изменения конфигурационного файла ./etc/pgbounce/pgbouncer.ini из шаблона pgbouncer.ini.j2. В данном шаблоне используются:
  - vars: {{ db_name }}
  - В database прописывается БД postgres и {{ db_name }}, пул мод session
- C помощью SQL запроса достаем SCRAM-SHA-256 для пользователя superuser. Сохраняем в super_scrum_secret
- Добавляем в файл /etc/pgbouncer/userlist.txt запись c переменными {{ superuser }} {{ super_scrum_secret }}
- C помощью SQL запроса достаем SCRAM-SHA-256 для пользователя {{ db_user }} . Сохраняем в db_user_scrum
- Добавляем в файл /etc/pgbouncer/userlist.txt запись c переменными {{ superuser }} {{ db_user_scrum }}
- Добавляем pgbounce в автозагрузку
- Перезапускаем pgbounce

<a name="headers"><h2>haproxy_node</h2></a>

<a name="headers"><h2>add_haproxy</h2></a>
- Устанавливаем haproxy
- Собираем gather facts с группы хостов cluster. Для последующего их использорвания в haproxy.cfg.
- Изменяем конфигурационный файл /etc/haproxy/haproxy.cfg используем шаблон haproxy.cfg.j2 . В данном шаблоне используются:
- gather_facts: ansible_hostname ( которые мы собрали ранее)
- special_vars: inventory_hostname
- Используеться цикл for для заполнения хостов на которые будет перенаправлять при обращении к 5000 и 5001 порту. 5000 направляет на мастер ноду, 5001 направляет на реплик ноды
- Добавляем haproxy в автозагрузуку
- Перезагружаем haproxy

  <a name="headers"><h2>pgbounce</h2></a>
  - Устанавливаем keepalive
  - Включаем поддержку 2-х ip адресов в /etc/sysctl.conf
  - Сохраняем изменения
  - Создаем конфигурационный файл /etc/keepalived/keepalived.conf с использованием шаблона keepalived.conf.j2. В данном шаблоне используются:
    - vars: {{ keepalived_ip  }}
  - Добавляем haproxy в автозагрузуку и запускаем сервис keepalived
