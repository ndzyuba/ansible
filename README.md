# Install Postgresql cluster ( etcd, patroni, pgbounce, haproxy, keepalived)
Для автоматизации развертывания используются роли:
[etc_hosts](#etc_hosts)  
[etcd](#etcd)  
[postgres](#postgres)  
[patroni](#patroni)
[create_user_and_db](#create_user)
[pgbounce](#images)  

[add_haproxy]
[keepalive]

<a name="headers"><h2>etc_hosts</h2></a>
В данной роли мы добавляем все хосты которые будем использовать в /etc/hosts. Хосты указаны в /vars/main.yml -> в переменной etc_hosts

<a name="headers"><h2>etcd</h2></a>
- устанавливаем etcd
- создаем директорию по etcd_data с правами и владельцем
- используем шаблон который находиться в templates/etcd.conf.j2. В данном шаблоне используются:
  - vars: etcd_data_dir , etcd_cluster_name
  - gather_facts: ansible_default_ipv4.address , ansible_hostname
  - special variable: invetory_hostname
  - 
