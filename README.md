## Ansible-Clickhouse
=========

#### It is highly recommended that you do not directly access this repository from your CI/CD jobs. Make a fork in your project's SCM instead. By doing so, you make sure that your cluster will not be broken in case one of the new commits appear to have a bug. You can keep the fork synchronized with the origin manually

#### Components

* `Yandex ClickHoise` (column-oriented relational database service)

#### Works with

* `Apache Zookeeper` (a server for coordinating data replication between ClickHouse nodes)

## Intro

> {{ playbook_dir }} is a default Ansible variable. This is where you keep your playbooks, variables, config files and templates

> Ansible roles you install with Ansible-Galaxy can be stored elsewhere. I used this variable to give you a better understanding of where all these files should be stored

## Config example with a minimum set of parameters required to set up a ClickHouse cluster with Zookeeper

```
* `FILE: {{ playbook_dir }}/inventory-clickhouse-staging`
```

```
[clickhouse]
ch-stg-01-01 ansible_host=10.226.10.110 ansible_user=jenkins-deploy ansible_port=22 ch_shard=01 ch_replica=01 gce_zone=us-east4-a
ch-stg-01-02 ansible_host=10.226.10.111 ansible_user=jenkins-deploy ansible_port=22 ch_shard=01 ch_replica=02 gce_zone=us-east4-b
ch-stg-02-01 ansible_host=10.226.10.112 ansible_user=jenkins-deploy ansible_port=22 ch_shard=02 ch_replica=01 gce_zone=us-east4-c
ch-stg-02-02 ansible_host=10.226.10.113 ansible_user=jenkins-deploy ansible_port=22 ch_shard=02 ch_replica=02 gce_zone=us-east4-a
ch-stg-03-01 ansible_host=10.226.10.114 ansible_user=jenkins-deploy ansible_port=22 ch_shard=03 ch_replica=01 gce_zone=us-east4-b
ch-stg-03-02 ansible_host=10.226.10.115 ansible_user=jenkins-deploy ansible_port=22 ch_shard=03 ch_replica=02 gce_zone=us-east4-c

[zookeeper]
zk-ch-stg-1 zookeeper_myid=1 ansible_host=10.226.10.116 ansible_user=jenkins-deploy ansible_port=22 gce_zone=us-east4-a
zk-ch-stg-2 zookeeper_myid=2 ansible_host=10.226.10.117 ansible_user=jenkins-deploy ansible_port=22 gce_zone=us-east4-b
zk-ch-stg-3 zookeeper_myid=3 ansible_host=10.226.10.118 ansible_user=jenkins-deploy ansible_port=22 gce_zone=us-east4-c
```

### THE FOLLOWING HOSTVARS ARE OBLIGATORY: 'ch_shard', 'ch_replica', 'zookeeper_myid'
### You may disregard the 'gce_' variables since they are related to Google Cloud Platform


* `FILE: {{ playbook_dir }}/requirements/clickhouse-cluster-requirements.yml`

```
---
# Basic server setup
- name: ansible-basic-server-setup
  ### Original repo
  src: git@github.com:arsenii-stefanov/ansible-basic-server-setup.git
# Set up Python
- name: ansible-python
  ### Original repo
  src: git@github.com:arsenii-stefanov/ansible-python.git
# Set up Docker
- name: ansible-docker
  ### Original repo
  src: git@github.com:arsenii-stefanov/ansible-docker.git
# Set up ClickHouse
- name: ansible-clickhouse
  ### Original repo
  src: git@github.com:arsenii-stefanov/ansible-clickhouse.git
# Set up Zookeeper
- name: ansible-zookeeper
  ### Original repo
  src: git@github.com:arsenii-stefanov/ansible-zookeeper.git
```

```
$ ansible-galaxy -vvvv install -f -r requirements/clickhouse-cluster-requirements.yml
```

#### Below is a playbook for setting up 2 ClickHouse clusters (primary and its failover replica, this approach presupposes that you write data to both clusters simultaneously)

* `FILE: {{ playbook_dir }}/playbook.yml`

```
---

- name: Install Zookeeper
  hosts: zookeeper
  connection: ssh
  become: true
  become_user: root
  become_method: sudo
  any_errors_fatal: true
  gather_facts: true

  pre_tasks:
    - name: Include Variables
      include_vars: "{{ item }}"
      with_items:
        - "zookeeper-{{ env_name }}.yml"
        - "zookeeper-{{ env_name }}-secrets.yml"
      when:
        - zookeeper_failover is defined
        - zookeeper_failover == false
      tags: [always]

    - name: Include Variables
      include_vars: "{{ item }}"
      with_items:
        - "zookeeper-{{ env_name }}-failover.yml"
        - "zookeepere-{{ env_name }}-failover-secrets.yml"
      when:
        - zookeeper_failover is defined
        - zookeeper_failover == true
      tags: [always]

  roles:
    - { role: "ansible-basic-server-setup", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - { role: "ansible-python", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - { role: "ansible-docker", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - ansible-zookeeper

- name: Install ClickHouse
  hosts: clickhouse
  connection: ssh
  become: true
  become_user: root
  become_method: sudo
  any_errors_fatal: true
  gather_facts: true

  pre_tasks:
    - name: Include Variables
      include_vars: "{{ item }}"
      with_items:
        - "clickhouse-{{ env_name }}.yml"
        - "clickhouse-{{ env_name }}-secrets.yml"
      when:
        - clickhouse_failover is defined
        - clickhouse_failover == false
      tags: [always]

    - name: Include Variables
      include_vars: "{{ item }}"
      with_items:
        - "clickhouse-{{ env_name }}-failover.yml"
        - "clickhouse-{{ env_name }}-failover-secrets.yml"
      when:
        - clickhouse_failover is defined
        - clickhouse_failover == true
      tags: [always]

  roles:
    - { role: "ansible-basic-server-setup", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - { role: "ansible-python", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
### In case you would like to install ClickHouse in Docker. This approach has several drawbacks. For example, by default you are not able to see the real IP
### inside the Docker container (you will see the Docker network adapter's gateway IP instead). And ClickHouse has a nice feature that allows you to limit direct access to
### its nodes by IP by user (in some cases it may be more convenient than a regular firewall). Although there is a workaround - make the Docker container use the host's
### network, running ClickHouse is Docker makes more sens if you use an orchestrator, such as Kubernetes. Anyway, if you wish to try, make sure you uncommen 
### the 'ansible-docker' role below
#    - { role: "ansible-docker", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - ansible-clickhouse
```

* `FILE: {{ playbook_dir }}/vars/clickhouse-staging-secrets.yml`
 
```
---

clickhouse_root_ca_cert_base64: <YOUR BASE64 ENCODED ROOT CERTIFICATE GOES HERE>
clickhouse_root_ca_key_base64: <YOUR BASE64 ENCODED ROOT KEY GOES HERE>

### Looks like the Interserver Secret feature is still being developed: https://github.com/ClickHouse/ClickHouse/pull/13156
#clickhouse_interserver_secret: <YOUR INTERSERVER SECRET GOES HERE>

clickhouse_zookeeper_user: "YOUR ZOOKEEPER USER GOES HERE"
clickhouse_zookeeper_password: "YOUR ZOOKEEPER PASSWORD GOES HERE"

clickhouse_users_default:
  - name: "default"
    password: ""
    networks: "{{ clickhouse_networks_default }}"
#    profile: "default"
    profile: "readonly"
    quota: "default"
    ### If no database is specified, it means that the user is allowed to access all databases. It doesn't seem possible to restrict access to the 'system' database at this time
#    dbs:
#      - "default"
#      - "system"
    comment: "Default user for login if user not defined"

clickhouse_users_custom:
### System Users
  - name: "interserver-user"    # we use this user since 'clickhouse_interserver_secret' still is not properly implemented
    password: "SOME PASSWORD"
    networks: "{{ clickhouse_networks_custom['stg-net'] }}"
    profile: "readonly"
    quota: "default"
    comment: "Interserver user with read-only access for running distributed queries"
    ### the 'system_user' options allows your user to connect from one CH node to another by adding the CH nodes' IPs to <networks></networks>
    system_user: true
  - name: "monitoring"
    password: ""
    networks: "{{ clickhouse_networks_default }}"
    profile: "readonly"
    quota: "default"
    comment: "Monitoring user with read only access"
### Programmatic users
  - name: "app-user"
    password: "SOME PASSWORD"
    networks: "{{ clickhouse_networks_custom['stg-net'] }}"
    profile: "readerbot"
    quota: "default"
    comment: "Bot with read only access"

```

* `FILE: {{ playbook_dir }}/vars/clickhouse-staging.yml`

```
---

ansible_python_interpreter: /usr/bin/python3

####################################
###     BASIC SERVER CONFIGS     ###
####################################
server_basic_fix_resolv_conf: true

python_modules_general:
  - name: "pip"
    state: "latest"
    extra_args: "--user"
    executable: "pip3"
python_modules_docker:
### Python module for communicating with Docker API
  - name: "docker"
    state: "present"
    extra_args: "--user"
    executable: "pip3"

docker_packages_ubuntu: [ "docker-ce" ]

#############################
###     SYSTEM CONFIGS    ###
#############################
clickhouse_sysctl_config: [
  { name: "vm.max_map_count", value: "262144", state: "present" }
]
clickhouse_limits_docker:
  - "nofile:500000:500000"    # Default value set by ClickHouse

##################################
###     CLICKHOUSE CONFIGS     ###
##################################

clickhouse_installation_type: "host"        # Available options: host, docker

clickhouse_cluster_name: "clickhouse-staging"

#clickhouse_docker_network_mode: "host"

#clickhouse_version: "20.3.15"        # Docker tag
clickhouse_version: "20.3.15.133"    # Ubuntu APT package

clickhouse_server_config:
  max_connections: 2048
  keep_alive_timeout: 3
  max_concurrent_queries: 100
  uncompressed_cache_size: 8589934592
  mark_cache_size: 5368709120
  builtin_dictionaries_reload_interval: 3600
  max_session_timeout: 3600
  default_session_timeout: 60

clickhouse_docker_log_options:
  max-file: 10
  max-size: "100m"

clickhouse_cluster:
  shards:
    - shard:
      - host: "ch-stg-01-01.{{ hostvars['ch-stg-01-01'].gce_zone }}.c.{{ gcp_project_id }}.internal"
        priority: 1
      - host: "ch-stg-01-02.{{ hostvars['ch-stg-01-02'].gce_zone }}.c.{{ gcp_project_id }}.internal"
        priority: 1
    - shard:
      - host: "ch-stg-02-01.{{ hostvars['ch-stg-02-01'].gce_zone }}.c.{{ gcp_project_id }}.internal"
        priority: 1
      - host: "ch-stg-02-02.{{ hostvars['ch-stg-02-02'].gce_zone }}.c.{{ gcp_project_id }}.internal"
        priority: 1
    - shard:
      - host: "ch-stg-03-01.{{ hostvars['ch-stg-03-01'].gce_zone }}.c.{{ gcp_project_id }}.internal"
        priority: 1
      - host: "ch-stg-03-02.{{ hostvars['ch-stg-03-02'].gce_zone }}.c.{{ gcp_project_id }}.internal"
        priority: 1

clickhouse_macro:
  shard: "{{ hostvars[inventory_hostname].ch_shard }}"
  replica: "{{ hostvars[inventory_hostname].ch_replica }}"

clickhouse_zookeeper_nodes:
  - host: "zk-ch-stg-1.{{ hostvars['zk-ch-stg-1'].gce_zone }}.c.{{ gcp_project_id }}.internal"
    port: 2181
#    secure: 1    # You may enable secure connection to each Zookeeper node if it has SSL configured
  - host: "zk-ch-stg-2.{{ hostvars['zk-ch-stg-2'].gce_zone }}.c.{{ gcp_project_id }}.internal"
    port: 2181
  - host: "zk-ch-stg-3.{{ hostvars['zk-ch-stg-3'].gce_zone }}.c.{{ gcp_project_id }}.internal"
    port: 2181

### User+password authentication
clickhouse_zookeeper_auth: true

###################################
###     CLICKHOUSE SSL CERTS    ###
###################################

clickhouse_ssl:
### Default HTTP interface
  http_interface_ssl_enable: false
  http_interface_ssl_only: false
### Native interface
  native_interface_ssl_enable: true
  native_interface_ssl_only: true
### Same Native interface but for dictributed queries
  distributed_query_ssl_enable: true
### Communication between replicas. Used for data exchange (apparently, if there is no Zookeeper)
  interserver_repl_ssl_enable: false    # if enabled, you must set the `clickhouse_interserver_https_host` too

clickhouse_generate_self_signed_ssl: true

### Shell command for generating a Root CA and key: openssl req -x509 -newkey rsa:4096 -keyout key.pem -out root-ca.pem -days 3650 -nodes -subj "/C=US/ST=VA/L=McLean/O=MyCompany/OU=IT/CN=clickhouse-staging/emailAddress=admin@clickhouse.local"

clickhouse_openssl_csr:
  common_name: "clickhouse-staging"
  country_name: "US"
  state_or_province_name: "VA"
  locality_name: "McLean"
  organizational_unit_name: "IT"
  organization_name: "MyCompany"
  email_address: "admin@clickhouse.local"
  subject_alt_name:
    - "DNS:{{ clickhouse_node_hostname }}"
    - "IP:{{ ansible_host }}"


###########################################
###     CLICKHOUSE ADDITIONAL CONFIGS   ###
###########################################
clickhouse_logger:
  server-log:
    level: trace
    log: "{{ clickhouse_default_log_dir }}/clickhouse-server.log"
    errorlog: "{{ clickhouse_default_log_dir }}/clickhouse-server.err.log"
    size: 1000M
    count: 10
  query-log:
    database: "system"
    table: "query_log"
    partition_by: "toYYYYMM(event_date)"
    flush_interval_milliseconds: 7500

clickhouse_zookeeper_config:
  resharding:
    task_queue_path: "/clickhouse/task_queue"
  ### Allow to execute distributed DDL queries (CREATE, DROP, ALTER, RENAME) on cluster. Works only if ZooKeeper is enabled. Comment it if such functionality isn't required  
  distributed_ddl:
    path: "/clickhouse/task_queue/ddl"

########################################################################
###     CLICKHOUSE REMOTE SERVERS (clickhouse-remote-servers.xml)    ###
########################################################################
### If a user is not set, and '<secret></secret>' is defined, then the initial_user (the user who initiated the query) should be used
clickhouse_interserver_user: "interserver-user"

####################################################################
###     CLICKHOUSE IPs/SUBNETS TO ALLOW USERS TO CONNECT FROM    ###
####################################################################
clickhouse_networks_default:
  - "127.0.0.1"

### If you want to restrict direct access to ClickHouse nodes and only allow access via your Load Balancer, then whitelist your Load Balancer IP/subnet
clickhouse_networks_custom:
  local:
    - "127.0.0.1"
  gcp-internal:
    - "10.128.0.0/9"
  docker:
    - "172.16.0.0/12"
  stg-net:
    - "10.226.0.0/20"        # staging network
    - "10.222.0.10/32"       # VPN

##################################
###     CLICKHOUSE PROFILES    ###
##################################
clickhouse_profiles_default:
  default:
    max_memory_usage: 10000000000
    use_uncompressed_cache: 0
    load_balancing: random
  readonly:
    readonly: 1

##########################################
###     CLICKHOUSE USERS AND QUOTAS    ###
##########################################
clickhouse_quotas_intervals_default: [
  { duration: 3600, queries: 0, errors: 0,result_rows: 0,read_rows: 0,execution_time: 0 }
]

clickhouse_quotas_default: [
  { name: "default", intervals: "{{ clickhouse_quotas_intervals_default }}", comment: "Default quota - count only" }
]

clickhouse_quotas_custom: []

clickhouse_profiles_custom:
  writer:
    readonly: "0" # write permission
    allow_ddl: "0" # allow DDL queries, such as: CREATE, ALTER, RENAME, ATTACH, DETACH, DROP, TRUNCATE
    log_queries: "1" # log queries to the database 'system'
  writerbot:
    readonly: "0" # write permission
    allow_ddl: "0" # allow DDL queries, such as: CREATE, ALTER, RENAME, ATTACH, DETACH, DROP, TRUNCATE
    log_queries: "1" # log queries to the database 'system'
  developer:
    readonly: "1" # read permission
    log_queries: "1" # log queries to the database 'system'
  developerexp:
    readonly: "2" # read permission + change session parameters permission 
    log_queries: "1" # log queries to the database 'system'
  readerbot:
    readonly: "1" # read permission
    max_query_size: "1000000"
    log_queries: "1" # log queries to the database 'system'
  readerbotexp:
    readonly: "2" # read permission + change session parameters permission
    max_query_size: "1000000"
    log_queries: "1" # log queries to the database 'system'
  bigqueryreaderbot:
    readonly: "1" # read permission
    max_query_size: "1000000"
    log_queries: "1" # log queries to the database 'system'
    max_memory_usage: "100000000000" # Max RAM per query, bytes
    prefer_localhost_replica: "0" # execute query on all available replicas
    max_parallel_replicas: "2" # works only for the tables created with SAMPLE clause. CH splits the query by sample section and executes part of the query on N (max_parallel_replicas) replicas.
  bigquerydeveloper:
    readonly: "1" # read permission
    max_query_size: "1000000"
    log_queries: "1" # log queries to the database 'system'
    max_memory_usage: "100000000000" # Max RAM per query, bytes
    prefer_localhost_replica: "0" # execute query on all available replicas
    max_parallel_replicas: "2" # works only for the tables created with SAMPLE clause. CH splits the query by sample section and executes part of the query on N (max_parallel_replicas) replicas.
  bigquerydeveloperexp:
    readonly: "2" # read permission + change session parameters permission
    max_query_size: "1000000"
    log_queries: "1" # log queries to the database 'system'
    max_memory_usage: "100000000000" # Max RAM per query, bytes
    prefer_localhost_replica: "0" # execute query on all available replicas
    max_parallel_replicas: "2" # works only for the tables created with SAMPLE clause. CH splits the query by sample section and executes part of the query on N (max_parallel_replicas) replicas.
  bigquerywriter:
    readonly: "0" # write permission
    allow_ddl: "1" # allow DDL queries, such as: CREATE, ALTER, RENAME, ATTACH, DETACH, DROP, TRUNCATE
    log_queries: "1" # log queries to the database 'system'
    max_memory_usage: "100000000000" # Max RAM per query, bytes
    prefer_localhost_replica: "0" # execute query on all available replicas
    max_parallel_replicas: "2" # works only for the tables created with SAMPLE clause. CH splits the query by sample section and executes part of the query on N (max_parallel_replicas) replicas.

################################################
###     CLICKHOUSE POST INSTALLATION TASKS   ###
################################################

### DATABASES
clickhouse_dbs_default: []
clickhouse_dbs_custom: []

clickhouse_dbs: "{{ clickhouse_dbs_default + clickhouse_dbs_custom }}"

### CUSTOM SQL SCRIPTS
clickhouse_custom_sql_scripts:
  path_local: "{{ playbook_dir }}/sql-scripts"
  ### If 'path_remote' is set, then Ansible will copy your SQL scripts FROM the local machine TO one of the remote ClickHouse nodes for further execution of
  ### SQL queries one the 'ClickHouse localhost'
  path_remote: "/tmp/custom_sql_scripts"
  search_patterns: "*.sql"    # comma separated
  search_recursively: true    # search for SQL scripts in the directory and its subdirectories
  ### if 'git' is set, then Ansible will attempt to clone the git repo with your SQL scripts to the local directory
  git:
    repo: "git@github.com:myteam/my-repo.git"
    dest: "{{ playbook_dir }}/sql-scripts"
    version: "master"
    accept_hostkey: true
    key_file: "{{ git_ssh_key_path | default(omit) }}"
```


* `FILE: {{ playbook_dir }}/vars/zookeeper-staging-secrets.yml`

```
---

zookeeper_users:
- user: "YOUR ZOOKEEPER USER GOES HERE"
  password: "YOUR ZOOKEEPER PASSWORD GOES HERE"
```

* `FILE: {{ playbook_dir }}/vars/zookeeper-staging.yml`

```
---

ansible_python_interpreter: /usr/bin/python3

####################################
###     BASIC SERVER CONFIGS     ###
####################################
server_basic_fix_resolv_conf: true

python_modules_general:
  - name: "pip"
    state: "latest"
    extra_args: "--user"
    executable: "pip3"
python_modules_docker:
### Python module for communicating with Docker API
  - name: "docker"
    state: "present"
    extra_args: "--user"
    executable: "pip3"

docker_packages_ubuntu: [ "docker-ce" ]

########################################
###     ZOOKEEPER SERVER CONFIGS     ###
########################################

zookeeper_version: "3.6.2"

zookeeper_docker_network: "zookeeper"

zookeeper_docker_env_vars:
  ZOO_MY_ID: "{{ hostvars[inventory_hostname]['zookeeper_myid'] }}"
  ZOO_SERVERS: "{{ zookeeper_servers }}"
  ZOO_TICK_TIME: "2000"
  #### The length of a single tick, which is the basic time unit used by ZooKeeper, as measured in milliseconds. It is used to regulate heartbeats, and timeouts.
  ### For example, the minimum session timeout will be two ticks
  ZOO_INIT_LIMIT: "30000"
  #### Amount of time, in ticks (see tickTime), to allow followers to connect and sync to a leader. Increased this value as needed, if the amount of data managed
  ### by ZooKeeper is large.
  ZOO_SYNC_LIMIT: "10"
  # Amount of time, in ticks (see tickTime), to allow followers to sync with ZooKeeper. If followers fall too far behind a leader, they will be dropped.
  ZOO_MAX_CLIENT_CNXNS: "2000"
  # Limits the number of concurrent connections (at the socket level) that a single client, identified by IP address, may make to a single member of the ZooKeeper ensemble.
  ZOO_AUTOPURGE_PURGEINTERVAL: "12"
  # The time interval in hours for which the purge task has to be triggered. Set to a positive integer (1 and above) to enable the auto purging. Defaults to 0.
  ZOO_AUTOPURGE_SNAPRETAINCOUNT: "100"
  ### When enabled, ZooKeeper auto purge feature retains the autopurge.snapRetainCount most recent snapshots and the corresponding transaction logs in the dataDir
  ### and dataLogDir respectively and deletes the rest. Defaults to 3. Minimum value is 3.
  ZOO_STANDALONE_ENABLED: "false"
  ### Prior to 3.5.0, one could run ZooKeeper in Standalone mode or in a Distributed mode. These are separate implementation stacks, and switching between them during
  ### run time is not possible. By default (for backward compatibility) standaloneEnabled is set to true. The consequence of using this default is that if started
  ### with a single server the ensemble will not be allowed to grow, and if started with more than one server it will not be allowed to shrink to contain fewer than
  ### two participants.
  ZOO_CFG_EXTRA: "maxSessionTimeout=60000000 preAllocSize=131072 snapCount=3000000 metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider metricsProvider.httpPort={{ zookeeper_metrics_port | default(7070) }}"
  JVMFLAGS: "-Xms128M -Xmx2G -verbose:gc -XX:+PrintGCDetails -XX:+PrintSafepointStatistics -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -Dzookeeper.authProvider.sasl=org.apache.zookeeper.server.auth.SASLAuthenticationProvider -Djava.security.auth.login.config={{ zookeeper_auth_dir_container }}/{{ zookeeper_auth_config_file }}"

zookeeper_docker_log_options:
  max-file: 10
  max-size: "100m"

```

### Set up the primary cluster

```
ansible-playbook clickhouse-cluster.yml -i inventory-clickhouse-staging -e env_name=staging -e "{'clickhouse_failover': false}" -e "{'zookeeper_failover': false}" -e "{'skip_basic_server_setup': false}"
```

### Set up the failover cluster

```
ansible-playbook clickhouse-cluster.yml -i inventory-clickhouse-staging-failover -e env_name=staging -e "{'clickhouse_failover': true}" -e "{'zookeeper_failover': true}" -e "{'skip_basic_server_setup': false}"
```