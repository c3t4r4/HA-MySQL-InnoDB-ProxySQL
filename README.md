# Tutorial para criar uma solução escalável com MySQL e ProxySQL no Debian 11

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

## Diagrama Usado:
![Diagrama](diagrama1.png "Diagrama")
___
## Pré-requisitos
- 04 servidores Debian 11 com acesso à internet.
- Sendo:
    - 01 = ProxySQL
    - 01 = MySQL Master (Escrita no Banco)
    - 02 = MySQL Slave (Leitura no Banco)
___
## Passo a passo

0. Preparação:
    - Instale o Debian 11 nos servidores diferentes.
    - Atualize as informações de pacote:
        ```sh
        sudo apt update && sudo timedatectl set-timezone America/Sao_Paulo && sudo apt update && sudo apt upgrade -y && sudo apt install curl net-tools software-properties-common acl unzip htop ncdu locales locales-all -y install software-properties-common wget gnupg ufw -y
        ```
        > Proxmox use isso:
        > ```sh
        >apt update && timedatectl set-timezone America/Sao_Paulo && apt update && apt upgrade -y && apt install curl net-tools software-properties-common acl unzip htop ncdu locales locales-all wget gnupg ufw -y
        >```

1. Configurando o MASTER:

    - Após Executar o Passo 0.

    - Configurando UFW:
        - Adicione regras UFW:
            ```sh
            ufw enable && ufw allow from 12.10.10.0/24 && ufw status
            ```

    - Instalação do MySQL:
        - Adicione o repositório do MySQL ao sistema:
            ```sh
            wget https://repo.mysql.com//mysql-apt-config_0.8.24-1_all.deb && sudo dpkg -i mysql-apt-config_0.8.24-1_all.deb && sudo apt update
            ```
            > Proxmox use isso:
            > ```sh
            >wget https://repo.mysql.com//mysql-apt-config_0.8.24-1_all.deb && dpkg -i mysql-apt-config_0.8.24-1_all.deb && apt update
            >```

        - Configuração Padrão:
        ![Config MySQL](imagem1.png "Config MySQL")
        - Instale o MySQL no Debian 11:
            ```sh
            sudo apt install mysql-server -y && mysql_secure_installation
            ```
            > Proxmox use isso:
            > ```sh
            >apt install mysql-server -y && mysql_secure_installation
            >```

        - Verifique se o MySQL está rodando:
            ```sh
            sudo systemctl status mysql
            ```
            > Proxmox use isso:
            > ```sh
            >systemctl status mysql
            >```
        
    - Configuração do MySQL:

        - Configure o arquivo de configuração do MySQL para permitir o acesso remoto:
            ```sh
            sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
            ```
            > Proxmox use isso:
            > ```sh
            >nano /etc/mysql/mysql.conf.d/mysqld.cnf
            >```

            - Adicione as seguintes linhas:
                ```nano
                [mysqld]

                default-storage-engine = InnoDB
                bind-address = 0.0.0.0
                server-id = 1
                log_bin = /var/log/mysql/mysql-bin.log
                max_binlog_size = 500M
                ```

            - Reinicie o MySQL:
                ```sh
                sudo systemctl restart mysql && sudo systemctl status mysql
                ```
                > Proxmox use isso:
                > ```sh
                >systemctl restart mysql && systemctl status mysql
                >```

        - Criando os usuários:
            - Acesse o console do MySQL:
                ```sh
                mysql -u root -p
                ```
            - Configurando usuário slave:
                ```sh
                mysql> create user 'user-slave-1'@'12.10.10.216' IDENTIFIED WITH mysql_native_password by 'strong-pass';

                mysql> create user 'user-slave-2'@'12.10.10.229' IDENTIFIED WITH mysql_native_password by 'strong-pass';

                mysql> GRANT REPLICATION SLAVE ON *.* TO 'user-slave-1'@'12.10.10.216';

                mysql> GRANT REPLICATION SLAVE ON *.* TO 'user-slave-2'@'12.10.10.229';
                mysql> FLUSH PRIVILEGES;
                ```
___

2. Configurando SLAVE 1:

    - Após Executar o Passo 0.

    - Configurando UFW:
        - Adicione regras UFW:
            ```sh
            ufw enable && ufw allow from 12.10.10.0/24 && ufw status
            ```
    - Instalação do MySQL:

        - Adicione o repositório do MySQL ao sistema:
            ```sh
            wget https://repo.mysql.com//mysql-apt-config_0.8.24-1_all.deb && sudo dpkg -i mysql-apt-config_0.8.24-1_all.deb && sudo apt update
            ```
            > Proxmox use isso:
            > ```sh
            >wget https://repo.mysql.com//mysql-apt-config_0.8.24-1_all.deb && dpkg -i mysql-apt-config_0.8.24-1_all.deb && apt update
            >```

        - Configuração Padrão:
        ![Config MySQL](imagem1.png "Config MySQL")
        - Instale o MySQL no Debian 11:
            ```sh
            sudo apt install mysql-server -y && mysql_secure_installation
            ```
            > Proxmox use isso:
            > ```sh
            >apt install mysql-server -y && mysql_secure_installation
            >```

        - Verifique se o MySQL está rodando:
            ```sh
            sudo systemctl status mysql
            ```
            > Proxmox use isso:
            > ```sh
            >systemctl status mysql
            >```

    - Configure o arquivo de configuração do MySQL slave 1:
        ```sh
        sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
        ```
        - Adicione as seguintes linhas:
            ```nano
            [mysqld]

            default-storage-engine = InnoDB
            bind-address = 0.0.0.0
            server-id = 2
            read_only = 1
            log_bin = /var/log/mysql/mysql-bin.log
            max_binlog_size = 500M
            ```

        - Reinicie o MySQL:
            ```sh
            sudo systemctl restart mysql && sudo systemctl status mysql
            ```
            > Proxmox use isso:
            > ```sh
            >systemctl restart mysql && systemctl status mysql
            >```

    - Configure replicação:

         - Acesse o console do MySQL:
            ```sh
            mysql -u root -p
            ```

        - Coletando Info do Server:
            ```sh
            mysql> SHOW MASTER STATUS;
            ```

        - Saida:
            ```sh
            *************************** 1. row ***************************
            File: mysql-bin.000002
            Position: 1047
            Binlog_Do_DB: 
            Binlog_Ignore_DB: 
            Executed_Gtid_Set: 
            1 row in set (0.00 sec)
            ```

        - Configure o usuário e a senha para o acesso remoto ao MySQL:
            ```sh
            mysql> CHANGE MASTER TO
                MASTER_HOST='12.10.10.209',
                MASTER_USER='user-slave-1',
                MASTER_PASSWORD='passwd-user-slave-1',
                MASTER_PORT=3306,
                MASTER_LOG_FILE='mysql-bin.000002',
                MASTER_LOG_POS=1047,
                MASTER_CONNECT_RETRY=10;
            ```

        - Iniciando Copia para o Slave:
            ```sh
            mysql> START SLAVE;
            ```

        - Verificando Status:
            ```sh
            mysql> SHOW SLAVE STATUS\G;
            ```

        - Saida:
            ```sh
            *************************** 1. row ***************************
                    Slave_IO_State: Waiting for master to send event
                        Master_Host: 12.10.10.209
                        Master_User: user-slave-1
                        Master_Port: 3306
                    Connect_Retry: 60
                    Master_Log_File: mysql-bin.000002
                Read_Master_Log_Pos: 1047
                    Relay_Log_File: slave-relay-bin.000002
                    Relay_Log_Pos: 324
            Relay_Master_Log_File: mysql-bin.000002
                    Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
            ```
___

3. Configurando SLAVE 2:

    - Após Executar o Passo 0.

    - Configurando UFW:
        - Adicione regras UFW:
            ```sh
            ufw enable && ufw allow from 12.10.10.0/24 && ufw status
            ```
    - Instalação do MySQL:

        - Adicione o repositório do MySQL ao sistema:
            ```sh
            wget https://repo.mysql.com//mysql-apt-config_0.8.24-1_all.deb && sudo dpkg -i mysql-apt-config_0.8.24-1_all.deb && sudo apt update
            ```
            > Proxmox use isso:
            > ```sh
            >wget https://repo.mysql.com//mysql-apt-config_0.8.24-1_all.deb && dpkg -i mysql-apt-config_0.8.24-1_all.deb && apt update
            >```

        - Configuração Padrão:
        ![Config MySQL](imagem1.png "Config MySQL")
        - Instale o MySQL no Debian 11:
            ```sh
            sudo apt install mysql-server -y && mysql_secure_installation
            ```
            > Proxmox use isso:
            > ```sh
            >apt install mysql-server -y && mysql_secure_installation
            >```

        - Verifique se o MySQL está rodando:
            ```sh
            sudo systemctl status mysql
            ```
            > Proxmox use isso:
            > ```sh
            >systemctl status mysql
            >```

    - Configure o arquivo de configuração do MySQL slave 1:
        ```sh
        sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
        ```
        - Adicione as seguintes linhas:
            ```nano
            [mysqld]

            default-storage-engine = InnoDB
            bind-address = 0.0.0.0
            server-id = 3
            read_only = 1
            log_bin = /var/log/mysql/mysql-bin.log
            max_binlog_size = 500M
            ```

        - Reinicie o MySQL:
            ```sh
            sudo systemctl restart mysql && sudo systemctl status mysql
            ```
            > Proxmox use isso:
            > ```sh
            >systemctl restart mysql && systemctl status mysql
            >```

    - Configure replicação:

         - Acesse o console do MySQL:
            ```sh
            mysql -u root -p
            ```

        - Coletando Info do Server:
            ```sh
            mysql> SHOW MASTER STATUS;
            ```

        - Saida:
            ```sh
            *************************** 1. row ***************************
            File: mysql-bin.000003
            Position: 115
            Binlog_Do_DB: 
            Binlog_Ignore_DB: 
            Executed_Gtid_Set: 
            1 row in set (0.00 sec)
            ```

        - Configure o usuário e a senha para o acesso remoto ao MySQL:
            ```sh
            mysql> CHANGE MASTER TO
                MASTER_HOST='12.10.10.209',
                MASTER_USER='user-slave-2',
                MASTER_PASSWORD='passwd-user-slave-2',
                MASTER_PORT=3306,
                MASTER_LOG_FILE='mysql-bin.000003',
                MASTER_LOG_POS=115,
                MASTER_CONNECT_RETRY=10;
            ```

        - Iniciando Copia para o Slave:
            ```sh
            mysql> START SLAVE;
            ```

        - Verificando Status:
            ```sh
            mysql> SHOW SLAVE STATUS\G;
            ```

        - Saida:
            ```sh
            *************************** 1. row ***************************
                    Slave_IO_State: Waiting for master to send event
                        Master_Host: 12.10.10.209
                        Master_User: user-slave-2
                        Master_Port: 3306
                    Connect_Retry: 60
                    Master_Log_File: mysql-bin.000003
                Read_Master_Log_Pos: 115
                    Relay_Log_File: slave-relay-bin.000003
                    Relay_Log_Pos: 324
            Relay_Master_Log_File: mysql-bin.000003
                    Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
            ```
___

4. Configurando o ProxySQL:

    - Após Executar o Passo 0.

    - Configurando UFW:
        - Adicione regras UFW:
            ```sh
            ufw enable && ufw allow from 12.10.10.0/24 && ufw status

    - Instalação do MySQL Client:

        - Adicione o repositório do MySQL ao sistema:
            ```sh
            wget https://repo.mysql.com//mysql-apt-config_0.8.24-1_all.deb && sudo dpkg -i mysql-apt-config_0.8.24-1_all.deb && sudo apt update
            ```
            > Proxmox use isso:
            > ```sh
            >wget https://repo.mysql.com//mysql-apt-config_0.8.24-1_all.deb && dpkg -i mysql-apt-config_0.8.24-1_all.deb && apt update
            >```

        - Configuração Padrão:
        ![Config MySQL](imagem1.png "Config MySQL")
        - Instale o MySQL no Debian 11:
            ```sh
            sudo apt install mysql-client -y
            ```
        > Proxmox use isso:
        >   ```sh
        >   apt install mysql-client -y
        >   ```

    - Adicione o repositório do ProxySQL ao seu sistema:
        ```sh
        sudo wget -O - 'https://repo.proxysql.com/ProxySQL/proxysql-2.4.x/repo_pub_key' | apt-key add - && sudo echo "deb https://repo.proxysql.com/ProxySQL/proxysql-2.4.x/$(lsb_release -sc)/ ./" | tee /etc/apt/sources.list.d/proxysql.list && sudo apt update
        ```
        > Proxmox use isso:
        >   ```sh
        >   wget -O - 'https://repo.proxysql.com/ProxySQL/proxysql-2.4.x/repo_pub_key' | apt-key add - && echo "deb https://repo.proxysql.com/ProxySQL/proxysql-2.4.x/$(lsb_release -sc)/ ./" | tee /etc/apt/sources.list.d/proxysql.list && apt update
        >   ```

    - Instale o ProxySQL:
        ```sh
        sudo apt install proxysql
        ```
        > Proxmox use isso:
        >   ```sh
        >   apt install proxysql
        >   ```
___

5. Criando Usuários do ProxySQL no MySQL Master (Criando no Master, será replicado nos Slaves):
    - Criando os usuários:
        - Acesse o console do MySQL:
            ```sh
            mysql -u root -p
            ```
        - Configurando usuários do ProxySQL:
            ```sh
            mysql> CREATE USER 'sysbench'@'12.10.10.%' IDENTIFIED WITH mysql_native_password BY 'senha-sysbench';

            mysql> CREATE USER 'monitor'@'12.10.10.%' IDENTIFIED WITH mysql_native_password BY 'senha-monitor';

            mysql> GRANT ALL PRIVILEGES on *.* TO 'sysbench'@'12.10.10.%' WITH GRANT OPTION;

            mysql> GRANT ALL PRIVILEGES on *.* TO 'monitor'@'12.10.10.%' WITH GRANT OPTION;

            mysql> FLUSH PRIVILEGES;
            ```
___

6. Configure o ProxySQL: 
    - Iniciando o ProxySQL:
        ```sh
        sudo systemctl start proxysql && sudo systemctl status proxysql
        ```
        > Proxmox use isso:
        > ```sh
        >systemctl start proxysql && systemctl status proxysql
        >```

    - Conectando com o admin do ProxySQL:
        ```sh
        mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt 'ProxySQL Admin> '
        ```

    > Nessa Configuração usaremos apenas 2 grupos, sendo 0 para gravação e 1 para leitura.

    - Conectando Servidores nos seu grupos:
        - Consultando dados:
            ```sh
            ProxySQL Admin> SELECT * FROM mysql_servers;
            ```

        - Saída:
            ```sh
            Empty set (0.00 sec)
            ```

        - Consultando dados:
            ```sh
            ProxySQL Admin> SELECT * from mysql_replication_hostgroups;
            ```

        - Saída:
            ```sh
            Empty set (0.00 sec)
            ```

        - Consultando dados:
            ```sh
            ProxySQL Admin> SELECT * from mysql_query_rules;
            ```

        - Saída:
            ```sh
            Empty set (0.00 sec)
            ```

        - Inserindo os Servidores:
            ```sh
            ProxySQL Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight,max_replication_lag) VALUES (0,'12.10.10.209',3306, 1000, 60);

            ProxySQL Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight,max_replication_lag) VALUES (1,'12.10.10.209',3306, 200, 60);

            ProxySQL Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight,max_replication_lag) VALUES (1,'12.10.10.216',3306, 1000, 60);

            ProxySQL Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight,max_replication_lag) VALUES (1,'12.10.10.229',3306, 1000, 60);

            ProxySQL Admin> SELECT hostgroup_id,hostname,port,weight,max_replication_lag FROM mysql_servers;
            ```

        - Saída:
            ```sh
            +--------------+--------------+------+--------+---------------------+
            | hostgroup_id | hostname     | port | weight | max_replication_lag |
            +--------------+--------------+------+--------+---------------------+
            | 0            | 12.10.10.209 | 3306 | 1000   | 60                  |
            | 1            | 12.10.10.209 | 3306 | 200    | 60                  |
            | 1            | 12.10.10.216 | 3306 | 1000   | 60                  |
            | 1            | 12.10.10.229 | 3306 | 1000   | 60                  |
            +--------------+--------------+------+--------+---------------------+
            4 rows in set (0.00 sec)
            ```
    - Configurando Monitor - (Esse monitor é o usuário criado no passo 5):

        - Atualizando a senha do monitor:
            ```sh
            ProxySQL Admin> UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_username';

            ProxySQL Admin> UPDATE global_variables SET variable_value='senha-monitor' WHERE variable_name='mysql-monitor_password';
            ```

        - Atualizando Intervalos:
            ```sh
            ProxySQL Admin> UPDATE global_variables SET variable_value='2000' WHERE variable_name IN ('mysql-monitor_connect_interval','mysql-monitor_ping_interval','mysql-monitor_read_only_interval');

            ProxySQL Admin> SELECT * FROM global_variables WHERE variable_name LIKE 'mysql-monitor_%';
            ```

        - Saída:
            ```sh
            +----------------------------------------------------------------------+----------------------+
            | variable_name                                                        | variable_value       |
            +----------------------------------------------------------------------+----------------------+
            | mysql-monitor_enabled                                                | true                 |
            | mysql-monitor_connect_timeout                                        | 600                  |
            | mysql-monitor_ping_max_failures                                      | 3                    |
            | mysql-monitor_ping_timeout                                           | 1000                 |
            | mysql-monitor_read_only_max_timeout_count                            | 3                    |
            | mysql-monitor_replication_lag_group_by_host                          | false                |
            | mysql-monitor_replication_lag_interval                               | 10000                |
            | mysql-monitor_replication_lag_timeout                                | 1000                 |
            | mysql-monitor_replication_lag_count                                  | 1                    |
            | mysql-monitor_groupreplication_healthcheck_interval                  | 5000                 |
            | mysql-monitor_groupreplication_healthcheck_timeout                   | 800                  |
            | mysql-monitor_groupreplication_healthcheck_max_timeout_count         | 3                    |
            | mysql-monitor_groupreplication_max_transactions_behind_count         | 3                    |
            | mysql-monitor_groupreplication_max_transactions_behind_for_read_only | 1                    |
            | mysql-monitor_galera_healthcheck_interval                            | 5000                 |
            | mysql-monitor_galera_healthcheck_timeout                             | 800                  |
            | mysql-monitor_galera_healthcheck_max_timeout_count                   | 3                    |
            | mysql-monitor_replication_lag_use_percona_heartbeat                  |                      |
            | mysql-monitor_query_interval                                         | 60000                |
            | mysql-monitor_query_timeout                                          | 100                  |
            | mysql-monitor_slave_lag_when_null                                    | 60                   |
            | mysql-monitor_threads_min                                            | 8                    |
            | mysql-monitor_threads_max                                            | 128                  |
            | mysql-monitor_threads_queue_maxsize                                  | 128                  |
            | mysql-monitor_local_dns_cache_ttl                                    | 300000               |
            | mysql-monitor_local_dns_cache_refresh_interval                       | 60000                |
            | mysql-monitor_local_dns_resolver_queue_maxsize                       | 128                  |
            | mysql-monitor_wait_timeout                                           | true                 |
            | mysql-monitor_writer_is_also_reader                                  | true                 |
            | mysql-monitor_username                                               | monitor              |
            | mysql-monitor_password                                               | senha-monitor        |
            | mysql-monitor_history                                                | 600000               |
            | mysql-monitor_connect_interval                                       | 2000                 |
            | mysql-monitor_ping_interval                                          | 2000                 |
            | mysql-monitor_read_only_interval                                     | 2000                 |
            | mysql-monitor_read_only_timeout                                      | 500                  |
            +----------------------------------------------------------------------+----------------------+
            36 rows in set (0.00 sec)
            ```

        - Atualizando as variaveis:
            ```sh
            ProxySQL Admin> LOAD MYSQL VARIABLES TO RUNTIME;

            ProxySQL Admin> SAVE MYSQL VARIABLES TO DISK;
            
            ProxySQL Admin> LOAD MYSQL SERVERS TO RUNTIME;
            ```

         - Consultando a Saude dos Servidores:
            ```sh
            ProxySQL Admin> SHOW TABLES FROM monitor;
            ```

        - Saída (Essas são as tabelas que vão guardar os dados dos servidores):
            ```sh
            +--------------------------------------+
            | tables                               |
            +--------------------------------------+
            | mysql_server_aws_aurora_check_status |
            | mysql_server_aws_aurora_failovers    |
            | mysql_server_aws_aurora_log          |
            | mysql_server_connect_log             |
            | mysql_server_galera_log              |
            | mysql_server_group_replication_log   |
            | mysql_server_ping_log                |
            | mysql_server_read_only_log           |
            | mysql_server_replication_lag_log     |
            +--------------------------------------+
            9 rows in set (0.00 sec)
            ```

        - Pode consultar com:
            ```sh
            ProxySQL Admin> SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC LIMIT 3;
            ```

        - Saída (Essas são as tabelas que vão guardar os dados dos servidores):
            ```sh
            +--------------+------+------------------+----------------------+------------+
            | hostname     | port | time_start_us    | ping_success_time_us | ping_error |
            +--------------+------+------------------+----------------------+------------+
            | 12.10.10.216 | 3306 | 1676120901730305 | 152                  | NULL       |
            | 12.10.10.229 | 3306 | 1676120901713324 | 117                  | NULL       |
            | 12.10.10.209 | 3306 | 1676120901696322 | 153                  | NULL       |
            +--------------+------+------------------+----------------------+------------+
            3 rows in set (0.00 sec)
            ```

        - Pode consultar com:
            ```sh
            ProxySQL Admin> SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 3;
            ```

        - Saída (Essas são as tabelas que vão guardar os dados dos servidores):
            ```sh
            +--------------+------+------------------+-------------------------+---------------+
            | hostname     | port | time_start_us    | connect_success_time_us | connect_error |
            +--------------+------+------------------+-------------------------+---------------+
            | 12.10.10.216 | 3306 | 1676120937689249 | 43998                   | NULL          |
            | 12.10.10.209 | 3306 | 1676120937666540 | 24975                   | NULL          |
            | 12.10.10.229 | 3306 | 1676120937643810 | 47824                   | NULL          |
            +--------------+------+------------------+-------------------------+---------------+
            3 rows in set (0.00 sec)
            ```

     - Configurando os Grupos de Replicação:

        - Atualizando a senha do monitor:
            ```sh
            ProxySQL Admin> INSERT INTO mysql_replication_hostgroups (writer_hostgroup,reader_hostgroup,comment) VALUES (0,1,'production');

            ProxySQL Admin> SELECT * FROM mysql_replication_hostgroups;
            ```

        - Saída:
            ```sh
            +------------------+------------------+------------+------------+
            | writer_hostgroup | reader_hostgroup | check_type | comment    |
            +------------------+------------------+------------+------------+
            | 0                | 1                | read_only  | production |
            +------------------+------------------+------------+------------+
            1 row in set (0.00 sec)
            ```

        - Para ativar essa auteração use:
            ```sh
            ProxySQL Admin> LOAD MYSQL SERVERS TO RUNTIME;

            ProxySQL Admin> SELECT * FROM monitor.mysql_server_read_only_log ORDER BY time_start_us DESC LIMIT 3;
            ```

        - Saída:
            ```sh
            +--------------+------+------------------+-----------------+-----------+-------+
            | hostname     | port | time_start_us    | success_time_us | read_only | error |
            +--------------+------+------------------+-----------------+-----------+-------+
            | 12.10.10.216 | 3306 | 1676121199751273 | 228             | 1         | NULL  |
            | 12.10.10.229 | 3306 | 1676121199733612 | 218             | 1         | NULL  |
            | 12.10.10.209 | 3306 | 1676121199716018 | 219             | 0         | NULL  |
            +--------------+------+------------------+-----------------+-----------+-------+
            3 rows in set (0.00 sec)
            ```

         - Agora salve os dados:
            ```sh
            ProxySQL Admin> SAVE MYSQL SERVERS TO DISK;
            ProxySQL Admin> SAVE MYSQL VARIABLES TO DISK;
            ```

    - Adicionando Usuários no ProxySQL (Lembrando que os mesmos usuários deve existir no servidor Master):
        - Para adicionar:
            ```sh
            ProxySQL Admin> INSERT INTO mysql_users(username,password,default_hostgroup) VALUES ('root','',0);

            ProxySQL Admin> INSERT INTO mysql_users(username,password,default_hostgroup) VALUES ('syscench','senha-sysbench',0);

            ProxySQL Admin> SELECT * FROM mysql_users;
            ```

        - Ativar Usuários:
            ```sh
            ProxySQL Admin> LOAD MYSQL USERS TO RUNTIME;

            ProxySQL Admin> SAVE MYSQL USERS TO DISK;
            ```
    
    - Configurando algumas regras:
        
        - Para inserir regras:
            ```sh
            ProxySQL Admin> INSERT INTO mysql_query_rules (active, match_digest, destination_hostgroup, apply) VALUES (1, '^SELECT.*', 1, 0);

            ProxySQL Admin> INSERT INTO mysql_query_rules (active, match_digest, destination_hostgroup, apply) VALUES (1, '^SELECT.*FOR UPDATE', 0, 1);

            ProxySQL Admin> SELECT active, match_pattern, destination_hostgroup, apply FROM mysql_query_rules;

            ProxySQL Admin> SELECT rule_id, match_digest,destination_hostgroup hg_id, apply FROM mysql_query_rules WHERE active=1;
            ```
    
        - Saida:
            ```sh
            +---------+---------------------+-------+-------+
            | rule_id | match_digest        | hg_id | apply |
            +---------+---------------------+-------+-------+
            | 1       | ^SELECT.*           | 1     | 0     |
            | 2       | ^SELECT.*FOR UPDATE | 0     | 1     |
            +---------+---------------------+-------+-------+
            2 rows in set (0.00 sec)
            ```

    - Continuando:
        ```sh
        ProxySQL Admin> LOAD MYSQL QUERY RULES TO RUNTIME;
        ProxySQL Admin> SAVE MYSQL QUERY RULES TO DISK;
        ```

        > Se algum de seus servidores ficar inacessível de qualquer grupo de host, o status será alterado de ONLINE para SHUNNED.
        > Isso significa que o ProxySQL não enviará nenhuma consulta a esse host até que ele volte para ONLINE.
        > Também podemos colocar qualquer servidor offline para manutenção. Para desativar um servidor de back-end, é necessário alterar seu status para OFFLINE_SOFT (Desativando corretamente um servidor de back-end) ou OFFLINE_HARD(Desativando imediatamente um servidor de back-end).

    - Caso queira Desabilitar um servidor:
        - Use o seguinte comando:
            ```sh
            ProxySQL Admin> UPDATE mysql_servers SET status='OFFLINE_SOFT' WHERE hostname='12.10.10.229';
            ```

    - Agora pode testar a conexão de outra maquina na range permitida:

        - Consultando a Porta:
            ```sh
            myuser> mysql -u sysbench -p -h 12.10.10.230 -P6033 -e"SELECT @@port"
            ```

        - Saída:
            ```sh
            +--------+
            | @@port |
            +--------+
            |   3306 |
            +--------+
            ```

        - Consultando o ID do Server:
            ```sh
            myuser> mysql -u sybcench -p -h 12.10.10.230 -P6033 -e"SELECT @@server_id"
            ```

        - Saída:
            ```sh
            +-------------+
            | @@server_id |
            +-------------+
            |           1 |
            +-------------+
            ```
___

7. Teste Funcional (Geralmente rodado em sua maquina mesmo, não no servidor ProxySQL):
    - Instalando o Script Sysbech:
        ```sh
        curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash && sudo apt -y install sysbench
        ```

    - Para Rodar o Teste Precisamos criar um schema e uma tabela:
        ```sh
        myuser> mysql -u syscench -p -h 12.10.10.230 -P6033 --prompt 'ProxySQL Client> '

        ProxySQL Client> CREATE SCHEMA sbtest;

        ProxySQL Client> USE sbtest;

        ProxySQL Client> CREATE TABLE `sbtest1` (
        `id` int(11) NOT NULL AUTO_INCREMENT,
        `k` int(11) NOT NULL DEFAULT '0',
        `c` char(120) NOT NULL DEFAULT '',
        `pad` char(60) NOT NULL DEFAULT '',
        PRIMARY KEY (`id`,`k`)
        )
        PARTITION BY RANGE (k) (
            PARTITION p1 VALUES LESS THAN (499999),
            PARTITION p2 VALUES LESS THAN MAXVALUE
        );

        ProxySQL Client> quit;
        ```

    - Rodando o Teste:
        ```sh
        myuser> sysbench /usr/share/sysbench/oltp_insert.lua --report-interval=2 --threads=4 --rate=20 --time=9999 --db-driver=mysql --mysql-host=12.10.10.230 --mysql-port=6033 --mysql-user=sysbench --mysql-db=sbtest --mysql-password='senha-sysbench' --tables=1 --table-size=1000000 run
        ```
    - Saída:
        ```sh
        [ 170s ] thds: 4 tps: 37.49 qps: 37.49 (r/w/o: 0.00/37.49/0.00) lat (ms,95%): 2009.23 err/s: 0.00 reconn/s: 0.00
        [ 170s ] queue length: 0, concurrency: 0
        [ 172s ] thds: 4 tps: 13.50 qps: 13.50 (r/w/o: 0.00/13.50/0.00) lat (ms,95%): 4.49 err/s: 0.00 reconn/s: 0.00
        [ 172s ] queue length: 11, concurrency: 4
        [ 174s ] thds: 4 tps: 0.00 qps: 0.00 (r/w/o: 0.00/0.00/0.00) lat (ms,95%): 0.00 err/s: 0.00 reconn/s: 0.00
        [ 174s ] queue length: 52, concurrency: 4
        [ 176s ] thds: 4 tps: 46.49 qps: 46.49 (r/w/o: 0.00/46.49/0.00) lat (ms,95%): 3639.94 err/s: 0.00 reconn/s: 0.00
        [ 176s ] queue length: 0, concurrency: 0
        [ 178s ] thds: 4 tps: 21.01 qps: 21.01 (r/w/o: 0.00/21.01/0.00) lat (ms,95%): 8.74 err/s: 0.00 reconn/s: 0.00
        [ 178s ] queue length: 0, concurrency: 0
        [ 180s ] thds: 4 tps: 11.00 qps: 11.00 (r/w/o: 0.00/11.00/0.00) lat (ms,95%): 4.65 err/s: 0.00 reconn/s: 0.00
        [ 180s ] queue length: 1, concurrency: 4
        [ 182s ] thds: 4 tps: 18.00 qps: 18.00 (r/w/o: 0.00/18.00/0.00) lat (ms,95%): 404.61 err/s: 0.00 reconn/s: 0.00
        [ 182s ] queue length: 0, concurrency: 0
        [ 184s ] thds: 4 tps: 24.01 qps: 24.01 (r/w/o: 0.00/24.01/0.00) lat (ms,95%): 26.68 err/s: 0.00 reconn/s: 0.00
        [ 184s ] queue length: 0, concurrency: 0
        [ 186s ] thds: 4 tps: 17.00 qps: 17.00 (r/w/o: 0.00/17.00/0.00) lat (ms,95%): 4.49 err/s: 0.00 reconn/s: 0.00
        [ 186s ] queue length: 0, concurrency: 0
        ```

    - Com isso é possível ver a fila de escrita.
___

- Considerações:
    > Com esses passos, você deve ter uma solução escalável com MySQL e ProxySQL configurada no seu Debian 11. Certifique-se de testar sua configuração antes de colocá-la em produção.
    >
    >A conexão com o banco de dados é feita através do ProxySQL da mesma forma que seria feita com um banco de dados diretamente. Você precisa especificar o endereço IP e a porta do ProxySQL como seu host de banco de dados e a porta 6033, padrão para o ProxySQL e não mais a porta 3306 padrão do MySQL. Além disso, você precisa fornecer o nome de usuário e a senha para autenticação no banco de dados.
___

- Exemplo de Conexão no ProxySQL:

    >Aqui está um exemplo de como realizar a conexão ao banco de dados através do ProxySQL em PHP:
    ```php
        <?php
        $host = "12.10.10.230";
        $port = 6033;
        $user = "username";
        $password = "password";
        $dbname = "database_name";

        // Cria uma nova conexão
        $conn = new mysqli($host, $user, $password, $dbname, $port);

        // Verifica se a conexão foi bem-sucedida
        if ($conn->connect_error) {
            die("A conexão falhou: " . $conn->connect_error);
        }
        echo "Conexão realizada com sucesso.";
        ?>
    ```
