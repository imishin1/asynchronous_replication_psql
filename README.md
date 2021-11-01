# Appointment

Deploys postgresql version 12 and sets up asynchronous replication for two hosts (master and slave)

# Requirements

- Access to repositories
- Connection between two nodes via internal ip using ssh keys is configured
- Debian (CentOS will be added later)
- Ansible

# Usage

File tree
```
.
├── ansible.cfg
├── group_vars
│   └── all_servers.yml
├── hosts
├── main.yml
├── README.md
├── roles
│   ├── config
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   ├── main.yml
│   │   │   ├── pg_basebackup.yml
│   │   │   ├── postgresql_conf.yml
│   │   │   ├── postgres_user.yml
│   │   │   ├── replica_user.yml
│   │   │   └── restart.yml
│   │   ├── templates
│   │   │   └── postgresql_conf.j2
│   │   └── vars
│   │       └── main.yml
│   ├── postgresql
│   │   └── tasks
│   │       ├── install.yml
│   │       ├── main.yml
│   │       └── start.yml
│   └── python_modules
│       └── tasks
│           ├── install.yml
│           └── main.yml
└── vars
    └── main.yml
```
Change ansible_user and ansible_ssh_private_key in ./group_vars/all_servers:
```
---
ansible_user : ubuntu
ansible_ssh_private_key : /home/ubuntu/.ssh/id_rsa
```

Change ansible_host for your master and replica in file hosts:
```
[master_servers]
master_1    ansible_host=10.236.30.178

[replica_servers]
replica_1    ansible_host=10.236.30.64
```
Change the variables for asynchronous replication in vars of role config:

!!!change master and replica hosts is necessary!!!
```
--hosts
master_host: 10.236.30.178
replica_host: 10.236.30.64

--Variables for replication user
db_user: replica_user
db_password: password
db_name: postgres

--The list of postgres directory
pg_hba_dir: /etc/postgresql/12/main/pg_hba.conf
postgresql_conf_dir: /etc/postgresql/12/main/postgresql.conf
postgresql_data_dir: /var/lib/postgresql/12/main
```
You can start the deployment and configuration process as follows:
```
ansible-playbook main.yml -i hosts
```

Сheck that everything works correctly:

On master:
```
select * from pg_stat_replication;
```
On slave:
```
select * from pg_stat_wal_receiver;
```

# Still to do

- Add support for centos
- Add the ability to use different versions of postgresql

# Recommendations (RUS)

1) Для мониторинга работспобности БД в первую очередь рекомендуется настроить следующие метрики:
С точки зрения ОС:
- Мониторинг LoadAVG
- Мониторинг CPU
- Мониторинг Memory
- Мониторинг Disk space
- Мониторинг IO
С точки зрения БД:
- Мониторинг статуса сегментов и базы в целом (up/down)
- Мониторинг числа подключений для предотвращения инцидентов с переполнением пулла коннектов
- Мониторинг блокировок

Для сбора метрик можно использовать какие-то из следующих инструментов:
- Graphite
- Zabbix
- Prometheus
Для визуализации можно использовать в т.ч. Grafana (например, в связке с Graphite)
Лично я бы использовал связку Graphite/Grafana, так как имею опыт работы с этими инструментами, и со своми функциями они справляются хорошо. В целом, любое из предложенных решений хорошо подходит для поставленной задачи, к тому же это OpenSource.

2) Если базу необходимо скофигурировать под OLAP, то рекомендуется следующее:
- Разрешить небольшое число подключений
- Не включать отстрел сессий из-за долгого выполнения, ибо для OLAP характерны большие запросы, которые могут выполняться довольно длительное время
- Увеличить значение shared_buffers. Цель состоит в том, чтобы хранить в кэше как можно больше данных, чтобы не подвергаться воздействию дорогостоящих операций ввода-вывода на диске. Вместе с ним стоит увеличить также max_wal_size.
- Установить высокий уровень work_mem, так как необходимо выделять как можно больше памяти каждому сеансу для слияния и сортировки
- Со стороны Hardware рекомендуется использование массива твердотельных накопителей для быстрого извлечения данных (можно использовать RAID10). Такая рекомендация возниакает из-за того, что запросы, скорее всего, не уместятся в памяти и необходимо использование дисков с высокой скоростью работы. Также рекомендуется выделить как можно больше ОЗУ, чтобы помочь сохранить данные в памяти и избежать дорогостоящего ввода-вывода
- Использование партиционирования, например, по времени, что позволит увеличить скорость выполнения запросов, отфильтрованных по времени. Более того, партиционирование даст возможность избавляться от ненужных данных посредством удаления старых партиций

3) Если базу необходимо скофигурировать под OLTP, то рекомендуется следующее:
- Разрешить большое число подключений
- Уменьшить объем памяти, выделяемый для каждой отдельной транзакции
- Настоить схему отсрела сессий, которые выполняются длительное время
- Использование индексов для быстрого доступа к строкам
- PostgreSQL не очень хорошо справляется с большим количеством соединений. Так, при увеличении большого числа пользователей можно столкнуться с рядом проблем, например, заполнение пулла коннектов или невозможность выполнить запросы определенного пользователя из-за чрезмерной загруженности со стороны других пользователей. Да и сама производительнсть базы страдает, так как открытие новых сессий - довольно дорогостоящая операция. Решить эту проблему можно, например, использоавнием pgbouncer, который будет держать подключения к БД постоянно открытыми, что позволит не затрачивать ресурсы на открытие новых сессий. С помощью pgbouncer можно также выставить ограничения по подключениям как к базе данных или для определенных пользователей, так и к кластеру постгреса в целом. Также имеет смысл выделить некоторое количество сессий, зарезервированных для админа, чтобы был доступ к БД в моменты высоких нагрузок.

4) Для обеспечения отказоустойчивости кластера в пределах одного дата центра рекомендуется следующее:
