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

Change ansible_host for your master and replica in file hosts:
```
[master_servers]
master_1    ansible_host=10.92.5.140
[replica_servers]
replica_1    ansible_host=10.92.5.9
```
Change the variables for asynchronous replication in vars of role config:

!!!change master and replica hosts is necessary!!!
```
--hosts
master_host: 10.92.5.140
replica_host: 10.92.5.9

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


# Still to do

- Add support for centos
- Add the ability to use different versions of postgresql
