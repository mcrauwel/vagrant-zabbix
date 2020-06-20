# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'

machines = YAML.load(File.read("machines.yaml"))
Vagrant.configure("2") do |config|
  machines.each do |hostname, specs|
    config.vm.define "#{hostname}" do |node|
      node.vm.box = "#{specs['box']}"
      node.vm.hostname = "#{hostname}"
      node.vm.network "private_network", ip: "#{specs['ip']}"

      if specs.has_key?('exposed_ports')
        specs['exposed_ports'].each do |internal_port, external_port|
          node.vm.network "forwarded_port", guest: internal_port, host: external_port
        end # each
      end # if

      @hosts_entries = []
      machines.each do |h, s|
        @hosts_entries << "%s   %s" % [s['ip'], h]
      end # each machine
      hosts = @hosts_entries.join("\n")

      # provision hosts file
      node.vm.provision "shell", inline: <<-SHELL
        echo "hello from node #{hostname}"
        echo "127.0.0.1   localhost" > /etc/hosts
        echo "127.0.1.1   #{hostname}" >> /etc/hosts
        echo "#{hosts}" >> /etc/hosts

        if [[ -f /etc/selinux/config ]]
        then
          sed -i s/^SELINUX=.*$/SELINUX=permissive/ /etc/selinux/config
          setenforce permissive
        fi
      SHELL

      if specs.has_key?('type')
        # provision servers based on type
        case specs['type']
        when 'zabbix-server'
          case specs['zabbix_db']
          when 'postgres'
            node.vm.provision "shell", inline: <<-SHELL
              echo "installing Zabbix-Server with PostgreSQL backend on #{hostname}"

              apt-get update -y -qq
              apt-get install -y postgresql-client postgresql -qq

              sudo -u postgres psql -c "CREATE DATABASE zabbix;"
              sudo -u postgres psql -c "CREATE USER zabbix WITH PASSWORD 'zabbix';"
              sudo -u postgres psql -c "GRANT ALL ON DATABASE zabbix TO zabbix;"

              echo "127.0.0.1:5432:zabbix:zabbix:zabbix" > ~/.pgpass
              chmod 600 ~/.pgpass

              if [[ ! -f /root/zabbix-release.deb ]]
              then
                wget --quiet https://repo.zabbix.com/zabbix/5.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_5.0-1+bionic_all.deb -O /root/zabbix-release.deb
                dpkg -i /root/zabbix-release.deb
              fi

              apt-get update -y -qq
              apt-get install -y zabbix-server-pgsql zabbix-frontend-php zabbix-agent libapache2-mod-php php-gd php-xml php-pgsql -qq

              zcat /usr/share/doc/zabbix-server-pgsql*/create.sql.gz | psql -h 127.0.0.1 -U zabbix zabbix

              echo "DBHost=127.0.0.1
DBName=zabbix
DBSchema=public
DBUser=zabbix
DBPassword=zabbix
DBPort=5432
" >> /etc/zabbix/zabbix_server.conf

              echo 'post_max_size = 16M
max_execution_time = 300
max_input_time = 300
date.timezone = UTC' > /etc/php/7.2/apache2/conf.d/99-zabbix.ini

              systemctl restart apache2

              mv /var/www/html /var/www/html.old
              ln -s /usr/share/zabbix /var/www/html

              echo '<?php
// Zabbix GUI configuration file
global $DB;

$DB["TYPE"]     = "POSTGRESQL";
$DB["SERVER"]   = "127.0.0.1";
$DB["PORT"]     = "0";
$DB["DATABASE"] = "zabbix";
$DB["USER"]     = "zabbix";
$DB["PASSWORD"] = "zabbix";

// SCHEMA is relevant only for IBM_DB2 database
$DB["SCHEMA"] = "";

$ZBX_SERVER      = "localhost";
$ZBX_SERVER_PORT = "10051";
$ZBX_SERVER_NAME = "Zabbix";

$IMAGE_FORMAT_DEFAULT = IMAGE_FORMAT_PNG;
' > /var/www/html/conf/zabbix.conf.php

            systemctl start zabbix-server.service

            SHELL
          else
            node.vm.provision "shell", inline: <<-SHELL
              echo "installing Zabbix-Server with MySQL backend on #{hostname}"

              apt-get update -y -qq
              apt-get install -y mysql-client mysql-server -qq

              mysql -e "CREATE DATABASE IF NOT EXISTS zabbix DEFAULT CHARACTER SET = UTF8 COLLATE = utf8_bin;"
              mysql -e "CREATE USER IF NOT EXISTS zabbix@localhost IDENTIFIED WITH mysql_native_password BY 'zabbix';"
              mysql -e "GRANT ALL ON zabbix.* TO zabbix@localhost;"

              if [[ ! -f /root/zabbix-release.deb ]]
              then
                wget --quiet https://repo.zabbix.com/zabbix/5.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_5.0-1+bionic_all.deb -O /root/zabbix-release.deb
                dpkg -i /root/zabbix-release.deb
              fi

              apt-get update -y -qq
              apt-get install -y zabbix-server-mysql zabbix-frontend-php zabbix-agent libapache2-mod-php php-gd php-xml php-mysql -qq

              zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -h localhost -uzabbix -pzabbix zabbix --init-command="SET NAMES utf8 COLLATE utf8_bin;"

              echo "DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix
DBPort=3306
" >> /etc/zabbix/zabbix_server.conf

              echo 'post_max_size = 16M
max_execution_time = 300
max_input_time = 300
date.timezone = UTC' > /etc/php/7.2/apache2/conf.d/99-zabbix.ini

              systemctl restart apache2

              mv /var/www/html /var/www/html.old
              ln -s /usr/share/zabbix /var/www/html

              echo '<?php
// Zabbix GUI configuration file
global $DB;

$DB["TYPE"]     = "MYSQL";
$DB["SERVER"]   = "localhost";
$DB["PORT"]     = "0";
$DB["DATABASE"] = "zabbix";
$DB["USER"]     = "zabbix";
$DB["PASSWORD"] = "zabbix";

// SCHEMA is relevant only for IBM_DB2 database
$DB["SCHEMA"] = "";

$ZBX_SERVER      = "localhost";
$ZBX_SERVER_PORT = "10051";
$ZBX_SERVER_NAME = "Zabbix";

$IMAGE_FORMAT_DEFAULT = IMAGE_FORMAT_PNG;
' > /var/www/html/conf/zabbix.conf.php

            systemctl restart zabbix-server.service

            SHELL
          end # case zabbix-db
        when 'mysql-server'
          node.vm.provision "shell", inline: <<-SHELL
            echo "installing MySQL (flavour: #{specs['mysql_flavour']}, version: #{specs['mysql_version']}) on #{hostname}"

            if [[ ! -f /root/zabbix-release.deb ]]
            then
              wget --quiet https://repo.zabbix.com/zabbix/5.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_5.0-1+bionic_all.deb -O /root/zabbix-release.deb
              dpkg -i /root/zabbix-release.deb
            fi


            apt-get update -y -qq
            apt-get install -y mysql-server zabbix-agent -qq

            echo "Server=#{machines['zabbix']['ip']}
ServerActive=#{machines['zabbix']['ip']}
Hostname=#{hostname}
" >> /etc/zabbix/zabbix_agentd.conf


            mysql -e "CREATE USER zbx_monitor@localhost IDENTIFIED WITH mysql_native_password BY 'monitor';"
            mysql -e "GRANT USAGE,REPLICATION CLIENT,PROCESS,SHOW DATABASES,SHOW VIEW ON *.* TO zbx_monitor@localhost;"

            mkdir -p /var/lib/zabbix
            chown zabbix:zabbix /var/lib/zabbix

            echo "[client]
user = 'zbx_monitor'
password = 'monitor'" > /var/lib/zabbix/.my.cnf
            chown zabbix:zabbix /var/lib/zabbix/.my.cnf
            chmod 600 /var/lib/zabbix/.my.cnf

            if [[ -f /vagrant/mysql/template_db_mysql.conf ]]
            then
              cp /vagrant/mysql/template_db_mysql.conf /etc/zabbix/zabbix_agentd.d/mysql.conf
            fi

            systemctl restart zabbix-agent.service

        SHELL
        when 'postgresql-server'
          node.vm.provision "shell", inline: <<-SHELL
            echo "installing PostgreSQL on #{hostname}"

            if [[ ! -f /root/zabbix-release.deb ]]
            then
              wget --quiet https://repo.zabbix.com/zabbix/5.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_5.0-1+bionic_all.deb -O /root/zabbix-release.deb
              dpkg -i /root/zabbix-release.deb
            fi

            apt-get update -y -qq
            apt-get install -y postgresql zabbix-agent -qq

            echo "Server=#{machines['zabbix']['ip']}
ServerActive=#{machines['zabbix']['ip']}
Hostname=#{hostname}
" >> /etc/zabbix/zabbix_agentd.conf

            sudo -u postgres psql -c "CREATE USER zbx_monitor WITH PASSWORD 'monitor' INHERIT;"
            sudo -u postgres psql -c "GRANT pg_monitor TO zbx_monitor;"

            mkdir -p /var/lib/zabbix
            chown zabbix:zabbix /var/lib/zabbix

            echo "*:5432:postgres:zbx_monitor:monitor" > /var/lib/zabbix/.pgpass
            chown zabbix:zabbix /var/lib/zabbix/.pgpass
            chmod 600 /var/lib/zabbix/.pgpass

            echo "host all zbx_monitor 127.0.0.1/32 trust
host all zbx_monitor 0.0.0.0/0 md5
host all zbx_monitor ::0/0 md5" >> /etc/postgresql/10/main/pg_hba.conf

            sudo -u postgres pg_ctlcluster 10 main reload

            if [[ -f /vagrant/postgresql/template_db_postgresql.conf ]]
            then
              cp /vagrant/postgresql/template_db_postgresql.conf /etc/zabbix/zabbix_agentd.d/postgresql.conf
              cp -Rpv /vagrant/postgresql/postgresql /var/lib/zabbix
            fi

            systemctl restart zabbix-agent.service

          SHELL
        end # case server-type
      end # spec type defined
    end # machine define
  end # machine loop
end # Vagrant.configure
