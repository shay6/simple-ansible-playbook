# Sonar Deployment
How to deploy new SonarQube and restore from backup.

SonarQube has two main components: database and UI (including API).
SonarQube supports many databases for backend but we use
PostgreSQL. The UI is a stateless program and the only important part of it
is the server side configuration including database connection settings.
The production environment has the two components separated into
different hosts.

Our automation allows you to deploy both single host and separated host
deployment. A single host is more suited for testing and development.
The automation is Ansible based, developed for RHEL, and maintained on GitHub:

 - [ansible-sonar-deployment][4]
 - [ansible-role-sonar][5]

The production SonarQube is deployed on RHEV-CI using DevOps Satellite:
 - [cci-prod-sonar-db][1]
 - [cci-prod-sonar-ui][2]

Connecting to the SonarQube operating system can be done with SSH, ask
@Alexander Braverman Masis to add your SSH public key to gain access.

On production UI server there is reverse proxy Nginx
[installed manually][3]. With the following configuration
(/etc/nginx/nginx.conf):

```
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;

    # CID-3279 - Can't push large reports to SonarQube server
    # 413 Request Entity Too Large
    client_max_body_size 100M;
}
```

## How to run automation

Requirements:

 - Ansible
 - RHEL server 7.5
 - Valid subscription manager credentials

Steps:

1. Setup:

  ```bash
  git clone https://github.com/abraverm/ansible-sonar-deployment.git
  cd ansible-sonar-deployment
  ansible-galaxy install -r requirements.yml -p roles
  ```

2. Edit 'inventory' file:

  - Separated UI and database:

    ```
    db ansible_host=10.8.242.105 ansible_ssh_user=root
    ui ansible_host=10.8.245.242 ansible_ssh_user=root
    ```

  - Single host:


    ```
    db ansible_host=10.8.242.105 ansible_ssh_user=root
    ui ansible_host=10.8.242.105 ansible_ssh_user=root
    ```

3. Run automation:
  - Clean deployment:

    ```bash
    ansible-playbook -i inventory playbook.yml
    ```
  - Restore from backup:

    ```bash
    ansible-playbook -i inventory playbook.yml --extra-vars "sonar_db_backup=</path/to/sonar/db/dump>"
    ```

Notes:

 - The `ansible-playbook` will prompt for RHN credentials which only needed on
   the first run.
 - When the automation is finished, you can browse to the UI on port 9000.
 - To have SonarQube listen on port 80, you will need to setup reverse
   proxy, for example, Nginx.
 - If you restored SonarQube from backup and don't know the admin password
   then SSH to database system, and then:

   ```
   su - postgres
   psql
   update users set crypted_password = '88c991e39bb88b94178123a849606905ebf440f5', salt='6522f3c5007ae910ad690bb1bdbf264a34884c6d' where login = 'admin';
   ```

 - SonarQube settings are found on the UI server at `/usr/local/sonar/conf`.
   You will need it if you have issues with database connection or plugin
   configuration.
 - LDAP settings:

   ```
   sonar.security.realm=LDAP
   sonar.security.savePassword=False
   ldap.url=ldaps://ldap.corp.redhat.com
   ldap.authentication=simple
   ldap.bindDn=uid=sonar,ou=serviceaccounts,dc=redhat,dc=com
   ldap.bindPassword=< ask abraverm >
   ldap.user.baseDn=dc=redhat,dc=com
   ldap.user.request=(uid={login})
   ```

 - To configure backup, open a ticket for DevOps to setup BareOS on the
   database host.

# For Ansible-Tower deployment

 - Log to Central CI Ansible Tower server at "https://tower-lb-01.host.prod.eng.rdu2.redhat.com/#/login#followAnchor"
 - Note: you need that your DB and UI servers will have the same root password.
 - Select "TEMPLATES" Tab

    - On "Deploy Sonar Test" click on the first icon that looks like a rocket. 
    - Choose "Root & Password" for CREDENTIAL then Enter your root password and click NEXT (to use SSH-KEY see the instructions below).
    - Enter your RHN User and Password in the SURVEY.
    - Enter your DB and UI addresses Respectively in the SURVEY and click LAUNCH. That's it!!

 - If you want to use CREDENTIAL with SSH-KEY you need to copy the Ansible Tower public key to your DB and UI servers and put it in /root/.ssh/authorized_keys file.
 - The Public Key is:
 ```
 ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC77WAIgJVpRdJTqNEhGwWCSq6YCTTPiKqzRZ0L5prJIBBqTMWBMVKTI12WW9LxJYvebibTyHF9K671Slje3hAevXFWvuYpkv4y+wX9ZbU3G1PstJP42sD0Tp6oZR8ReVE2k2pLenux6NbYHPJ98kiuWnYREnTjENVQQ33anfOFNsxCsQ08x/xnfpliQY1Bq4KAbU5NmhIXPeRdkSccis04Le4OCPTjMe7cE1SzPkeADhCxWz9kytK25flJbxJ/prQ0AXXMpleXtHIGL1v2rxnGkw5dmRbkW4YzzHYmfIws/fDdGe7lGvXN6A5OoCo3UdqgzO4+KVCoJnX1aT21n6Cz root@tower-lb-01.host.prod.eng.rdu2.redhat.com
 ```

[1]: https://satellite6.corp.redhat.com/hosts/cci-prod-sonar-db.rhev-ci-vms.eng.rdu2.redhat.com
[2]: https://satellite6.corp.redhat.com/hosts/cci-prod-sonar-ui.rhev-ci-vms.eng.rdu2.redhat.com
[3]: https://www.cyberciti.biz/faq/how-to-install-and-use-nginx-on-centos-7-rhel-7/
[4]: https://github.com/abraverm/ansible-sonar-deployment
[5]: https://github.com/abraverm/ansible-role-sonar/tree/postgresql
