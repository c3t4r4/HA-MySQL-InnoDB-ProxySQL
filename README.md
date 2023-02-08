# Tutorial para criar uma solução de alta disponibilidade com MySQL, InnoDB e ProxySQL no Debian 11

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)
___
## Pré-requisitos
- Três servidores Debian 11 com acesso à internet.
- Sendo um para o ProxySQL e os demais como Master-Slave do MySQL.
- Uma conta de superusuário (root) nos servidores.
___
## Passo a passo

0. Preparação:
   - Instale o Debian 11 em pelo menos três servidores diferentes. Estes servidores serão os membros do seu cluster.
   - Atualize as informações de pacote e instale o software-properties-common para habilitar o repositório do MySQL:
   ```sh
    sudo apt update
    sudo apt install software-properties-common
    ```

   - Adicione o repositório do MySQL ao sistema:
   ```sh
    sudo apt-key adv --keyserver keys.gnupg.net --recv-keys 8C718D3B5072E1F5
    sudo add-apt-repository 'deb https://repo.mysql.com/apt/debian/ bullseye mysql-8.0'
    sudo apt update
    ```
___
1. Instalação do MySQL:
   - Instale o MySQL no Debian 11:
    ```sh
    sudo apt install mysql-server
    ```

    - Inicie o MySQL:
    ```sh
    sudo systemctl start mysql
    ```

    - Verifique se o MySQL está rodando:
    ```sh
    sudo systemctl status mysql
    ```
___
2. Configuração do MySQL:
    - Instale o pacote de gerenciamento de configuração do MySQL:
    ```sh
    sudo apt install mysql-community-server-advanced
    ```

    - Acesse o console do MySQL:
    ```sh
    mysql -u root -p
    ```

    - Configure o usuário e a senha para o acesso remoto ao MySQL:
    ```sh
    mysql> GRANT ALL PRIVILEGES ON . TO 'usuario'@'%' IDENTIFIED BY 'senha' WITH GRANT OPTION;
    mysql> FLUSH PRIVILEGES;
    ```

    - Configure o arquivo de configuração do MySQL para permitir o acesso remoto:
    ```sh
    sudo nano /etc/mysql/my.cnf
    ```

    - Adicione as seguintes linhas:
    ```nano
    [mysqld]
    bind-address = 0.0.0.0
    ```

    - Reinicie o MySQL:
    ```sh
    sudo systemctl restart mysql
    ```
___
3. Replicação do MySQL:
    - Acesse o console do MySQL no servidor principal:
    ```sh
    mysql -u root -p
    ```

    - Configure o servidor principal para permitir a replicação:
    ```sh
    mysql> CHANGE MASTER TO MASTER_HOST='<endereço IP do servidor secundário>', MASTER_USER='usuario', MASTER_PASSWORD='senha', MASTER_PORT=3306;
    ```

    - Inicie a replicação:
    ```sh
    mysql> START SLAVE;
    ```

    - Verifique o status da replicação:
    ```sh
    mysql> SHOW SLAVE STATUS\G
    ```
___
4. Configuração do InnoDB:
    - Configure o arquivo de configuração do MySQL para usar o InnoDB
    ```sh
    sudo nano /etc/mysql/my.cnf
    ```

    - Adicione as seguintes linhas:
    ```nano
    [mysqld]
    default-storage-engine = InnoDB
    ```

    - Reinicie o MySQL:
    ```sh
    sudo systemctl restart mysql
    ```
___
5. Instalação do ProxySQL:
    - Adicione o repositório do ProxySQL ao seu sistema:
    ```sh
    sudo echo "deb https://repo.proxysql.com/ProxySQL/proxysql-2.0.x/Debian/ buster main" >> /etc/apt/sources.list
    sudo apt-key adv --keyserver keys.gnupg.net --recv-keys 5072E1F5
    ```

    - Atualize a lista de pacotes:
    ```sh
    sudo apt update
    ```

    - Instale o ProxySQL:
    ```sh
    sudo apt install proxysql
    ```
___
6. Configure o ProxySQL: 
    - Configure o ProxySQL para apontar para o seu servidor principal e secundário:
    ```sh
    sudo nano /etc/proxysql.cnf
    ```

    - Adicione as seguintes linhas:
    ```nano
    [mysql_servers]
    hostgroup_id=10
    hostname=<endereço IP do servidor principal>
    port=3306
    status=ON
    weight=1

    hostgroup_id=20
    hostname=<endereço IP do servidor secundário>
    port=3306
    status=ON
    weight=2
    ```

    - Reinicie o ProxySQL:
    ```sh
    sudo systemctl restart proxysql
    ```
___
7. Verificação da configuração:
    - Verifique o status do ProxySQL:
    ```sh
    sudo systemctl status proxysql
    ```

    - Acesse o console do ProxySQL:
    ```sh
    mysql -u admin -padmin -h 127.0.0.1 -P6032
    ```

    - Verifique se o ProxySQL está apontando para o servidor principal e secundário:
    ```sh
    SELECT hostgroup_id, hostname, port, status, weight FROM mysql_servers;
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
