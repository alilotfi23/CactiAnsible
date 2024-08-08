# CactiAnsible

CactiAnsible is an Ansible playbook designed to automate the installation and configuration of Cacti, an open-source network monitoring and graphing tool, on Ubuntu servers. This project is modular and organized into roles for better reusability and maintainability.

## File Structure

```plaintext
CactiAnsible/
├── ansible.cfg               # Configuration file for Ansible
├── inventory
│   └── hosts                 # Inventory file containing the server details
├── playbooks
│   └── site.yml              # Main playbook that includes all roles
└── roles
    ├── apache
    │   ├── tasks
    │   │   └── main.yml      # Tasks to install and configure Apache
    │   └── templates
    │       └── apache.conf.j2 # Template for Apache virtual host configuration
    ├── cacti
    │   ├── tasks
    │   │   └── main.yml      # Tasks to install and configure Cacti
    │   └── templates
    │       └── cacti.sql.j2  # Template for Cacti database configuration
    ├── mysql
    │   ├── tasks
    │   │   └── main.yml      # Tasks to install and configure MySQL
    │   └── templates
    │       └── my.cnf.j2     # Template for MySQL configuration
    └── php
        ├── tasks
        │   └── main.yml      # Tasks to install and configure PHP
        └── templates
            └── php.ini.j2    # Template for PHP configuration
```

## Prerequisites

- Ansible installed on your control machine.
- Access to the target Ubuntu server(s) with appropriate permissions.

## Configuration

### ansible.cfg

Defines configuration settings for Ansible, including the inventory path and roles path.

```ini
[defaults]
inventory = inventory/hosts
roles_path = roles
```

### inventory/hosts

Lists the target Ubuntu server(s) where Cacti will be installed. Replace placeholders with actual server details.

```ini
[ubuntu_servers]
your_server_ip ansible_user=your_user ansible_password=your_password
```

## Playbooks

### playbooks/site.yml

Main playbook that includes all roles required for installing and configuring Cacti.

```yaml
---
- hosts: ubuntu_servers
  become: yes
  roles:
    - apache
    - mysql
    - php
    - cacti
```

## Roles

### Apache

Tasks for installing and configuring Apache.

#### roles/apache/tasks/main.yml

```yaml
---
- name: Install Apache
  apt:
    name: apache2
    state: present

- name: Start and enable Apache
  systemd:
    name: apache2
    enabled: yes
    state: started

- name: Deploy Apache configuration
  template:
    src: apache.conf.j2
    dest: /etc/apache2/sites-available/000-default.conf

- name: Enable Apache mod_rewrite
  command: a2enmod rewrite

- name: Restart Apache
  systemd:
    name: apache2
    state: restarted
```

#### roles/apache/templates/apache.conf.j2

Template for Apache virtual host configuration.

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### MySQL

Tasks for installing and configuring MySQL.

#### roles/mysql/tasks/main.yml

```yaml
---
- name: Install MySQL
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - mysql-server
    - mysql-client

- name: Start and enable MySQL
  systemd:
    name: mysql
    enabled: yes
    state: started

- name: Deploy MySQL configuration
  template:
    src: my.cnf.j2
    dest: /etc/mysql/my.cnf

- name: Restart MySQL
  systemd:
    name: mysql
    state: restarted

- name: Create Cacti database and user
  mysql_db:
    name: cacti
    state: present

- name: Create Cacti user
  mysql_user:
    name: cactiuser
    password: cactiuserpassword
    priv: 'cacti.*:ALL'
    state: present
```

#### roles/mysql/templates/my.cnf.j2

Template for MySQL configuration.

```ini
[mysqld]
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
```

### PHP

Tasks for installing and configuring PHP.

#### roles/php/tasks/main.yml

```yaml
---
- name: Install PHP and extensions
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - php
    - php-mysql
    - php-gd
    - libapache2-mod-php
    - php-xml
    - php-mbstring
    - php-snmp

- name: Deploy PHP configuration
  template:
    src: php.ini.j2
    dest: /etc/php/7.4/apache2/php.ini

- name: Restart Apache
  systemd:
    name: apache2
    state: restarted
```

#### roles/php/templates/php.ini.j2

Template for PHP configuration.

```ini
memory_limit = 512M
upload_max_filesize = 16M
post_max_size = 16M
max_execution_time = 300
```

### Cacti

Tasks for installing and configuring Cacti.

#### roles/cacti/tasks/main.yml

```yaml
---
- name: Install Cacti
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - cacti
    - snmp
    - snmpd
    - rrdtool

- name: Import Cacti database
  mysql_db:
    name: cacti
    state: import
    target: /usr/share/doc/cacti/cacti.sql

- name: Configure Cacti database settings
  template:
    src: cacti.sql.j2
    dest: /usr/share/cacti/include/config.php

- name: Set up Cacti cron job
  cron:
    name: "Cacti cron job"
    minute: "*/5"
    user: www-data
    job: "/usr/bin/php /usr/share/cacti/poller.php > /dev/null 2>&1"

- name: Restart Apache
  systemd:
    name: apache2
    state: restarted
```

#### roles/cacti/templates/cacti.sql.j2

Template for Cacti database configuration.

```php
<?php
$database_type = 'mysql';
$database_default = 'cacti';
$database_hostname = 'localhost';
$database_username = 'cactiuser';
$database_password = 'cactiuserpassword';
$database_port = '3306';
$database_ssl = false;
?>
```

## Usage

To deploy Cacti on your Ubuntu server(s), run the following command:

```bash
git clone https://github.com/alilotfi23/CactiAnsible.git
cd CactiAnsible/playbooks
ansible-playbook playbooks/site.yml
```

Ensure that you have replaced the placeholders in the `inventory/hosts` file with your actual server details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
