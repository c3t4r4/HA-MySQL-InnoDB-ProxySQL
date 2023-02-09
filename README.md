# Tutorial para criar uma solução de alta disponibilidade com MySQL, InnoDB e ProxySQL no Debian 11

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
        > ```sh
        >apt install mysql-client -y
        >```

    - Adicione o repositório do ProxySQL ao seu sistema:
    ```sh
    sudo wget -O - 'https://repo.proxysql.com/ProxySQL/proxysql-2.4.x/repo_pub_key' | apt-key add - && sudo echo "deb https://repo.proxysql.com/ProxySQL/proxysql-2.4.x/$(lsb_release -sc)/ ./" | tee /etc/apt/sources.list.d/proxysql.list && sudo apt update
    ```
    > Proxmox use isso:
    > ```sh
    > wget -O - 'https://repo.proxysql.com/ProxySQL/proxysql-2.4.x/repo_pub_key' | apt-key add - && echo "deb https://repo.proxysql.com/ProxySQL/proxysql-2.4.x/$(lsb_release -sc)/ ./" | tee /etc/apt/sources.list.d/proxysql.list && apt update
    >```

    - Instale o ProxySQL:
    ```sh
    sudo apt install proxysql
    ```
    > Proxmox use isso:
    > ```sh
    >apt install proxysql
    >```
___

5. Criando Usuários do ProxySQL no MySQL Master (Criando no Master, será replicado nos Slaves):
    - Criando os usuários:
        - Acesse o console do MySQL:
        ```sh
        mysql -u root -p
        ```
        - Configurando usuário slave:
        ```sh
        mysql> CREATE USER 'sysbench'@'12.10.10.%' IDENTIFIED WITH mysql_native_password BY 'senha-sysbench';

        mysql> CREATE USER 'monitor'@'12.10.10.%' IDENTIFIED WITH mysql_native_password BY 'senha-monitor';

        mysql> GRANT ALL PRIVILEGES on *.* TO 'sysbench'@'12.10.10.%' WITH GRANT OPTION;

        mysql> GRANT ALL PRIVILEGES on *.* TO 'monitor'@'12.10.10.%' WITH GRANT OPTION;

        mysql> FLUSH PRIVILEGES;
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
        mysql -u admin -padmin -h 127.0.0.1 -P6032
        ```

    > Nessa Configuração usaremos apenas 2 grupos, sendo 0 para gravação e 1 para leitura.

    - Conectando Servidores nos seu grupos:
        ```sh
        mysql> INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight) VALUES (0,'12.10.10.209',3306, 1000);

        mysql> INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight) VALUES (1,'12.10.10.209',3306, 200);

        mysql> INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight) VALUES (1,'12.10.10.216',3306, 1000);

        mysql> INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight) VALUES (1,'12.10.10.229',3306, 1000);

        mysql> INSERT INTO  mysql_replication_hostgroups VALUES (0,1,'production');

        mysql> SELECT * FROM mysql_replication_hostgroups;
        ```

    - Saida:
        ```sh
        +------------------+------------------+------------+------------+
        | writer_hostgroup | reader_hostgroup | check_type | comment    |
        +------------------+------------------+------------+------------+
        | 0                | 1                | read_only  | production |
        +------------------+------------------+------------+------------+
        1 row in set (0.00 sec)
        ```
    
    - Continuando:
        ```sh
        mysql> LOAD MYSQL SERVERS TO RUNTIME; 
        mysql> SAVE MYSQL SERVERS TO DISK;

        mysql> SELECT hostgroup_id,hostname,port,status,weight FROM mysql_servers;
        ```

    - Saida:
        ```sh
        +--------------+--------------+------+--------+--------+
        | hostgroup_id | hostname     | port | status | weight |
        +--------------+--------------+------+--------+--------+
        | 0            | 12.10.10.209 | 3306 | ONLINE | 1000   |
        | 1            | 12.10.10.209 | 3306 | ONLINE | 200    |
        | 1            | 12.10.10.216 | 3306 | ONLINE | 1000   |
        | 1            | 12.10.10.229 | 3306 | ONLINE | 1000   |
        +--------------+--------------+------+--------+--------+
        4 rows in set (0.00 sec)
        ```

    - Continuando:
        ```sh
        mysql> UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_password';

        mysql> LOAD MYSQL VARIABLES TO RUNTIME;

        mysql> SAVE MYSQL VARIABLES TO DISK;

        mysql> INSERT INTO mysql_users(username,password,default_hostgroup) VALUES ('sysbench','senha-sysbench',1);

        mysql> SELECT username,password,active,default_hostgroup,default_schema,max_connections,max_connections FROM mysql_users;
        ```

    - Saida:
        ```sh
        +----------+-----------------------+--------+-------------------+----------------+-----------------+-----------------+
        | username | password              | active | default_hostgroup | default_schema | max_connections | max_connections |
        +----------+-----------------------+--------+-------------------+----------------+-----------------+-----------------+
        | sysbench | senha-sysbench | 1      | 1                 | NULL           | 10000           | 10000           |
        +----------+-----------------------+--------+-------------------+----------------+-----------------+-----------------+
        1 row in set (0.00 sec)
        ```

    - Continuando:
        ```sh
        mysql> LOAD MYSQL USERS TO RUNTIME;
        mysql> SAVE MYSQL USERS TO DISK;

        mysql> UPDATE global_variables SET variable_value=2000 WHERE variable_name IN ('mysql-monitor_connect_interval','mysql-monitor_ping_interval','mysql-monitor_read_only_interval');

        mysql> UPDATE global_variables SET variable_value = 1000 where variable_name = 'mysql-monitor_connect_timeout';

        mysql> UPDATE global_variables SET variable_value = 500 where variable_name = 'mysql-monitor_ping_timeout';

        mysql> LOAD MYSQL VARIABLES TO RUNTIME;
        mysql> SAVE MYSQL VARIABLES TO DISK;

        mysql> UPDATE mysql_servers SET max_replication_lag=60;

        mysql> INSERT INTO mysql_query_rules (active, match_digest, destination_hostgroup, apply) VALUES (1, '^SELECT.*', 1, 0);

        mysql> INSERT INTO mysql_query_rules (active, match_digest, destination_hostgroup, apply) VALUES (1, '^SELECT.*FOR UPDATE', 0, 1);

        mysql> SELECT active, match_pattern, destination_hostgroup, apply FROM mysql_query_rules;

        mysql> SELECT rule_id, match_digest,destination_hostgroup hg_id, apply FROM mysql_query_rules WHERE active=1;
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
        mysql> LOAD MYSQL QUERY RULES TO RUNTIME;
        mysql> SAVE MYSQL QUERY RULES TO DISK;
        ```
___

>>> VALIDADO ATE AQUI

7. Validando DB Connection: 
    - Iniciando o ProxySQL:
        ```sh
        mysql -u sysbench -p sysbench -h 127.0.0.1 -P6033 -e "SELECT @@server_id"
        ```

___

> Com esses passos, você deve ter uma solução de alta disponibilidade com MySQL, InnoDB e ProxySQL configurada no seu Debian 11. Certifique-se de testar sua configuração antes de colocá-la em produção.
>
>A conexão com o banco de dados é feita através do ProxySQL da mesma forma que seria feita com um banco de dados diretamente. Você precisa especificar o endereço IP e a porta do ProxySQL como seu host de banco de dados e a porta 3306 (padrão para o MySQL). Além disso, você precisa fornecer o nome de usuário e a senha para autenticação no banco de dados.
___

>Aqui está um exemplo de como realizar a conexão ao banco de dados através do ProxySQL em PHP:
```php
    <?php
    $host = "127.0.0.1";
    $port = 3306;
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
