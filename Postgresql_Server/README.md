# Postgresql Database
Walkthrough of installing Postgresql database. 

## Table of Contents

- [Step 1 - Installing Postgresql](#step-1---installing-postgresql)
- [Step 2 - Create user and database.](#step-2---create-user-and-database)
- [Step 3 - Configure postgresql](#step-3---configure-postgresql)
- [Step 4 - Configure access](#step-3---configure-access)



### Step 1 - Installing Postgresql
`sudo apt-get update`<br>
`sudo apt-get -y install postgresql`<br>


### Step 2 - Create User and Database
`sudo adduser --system --shell /bin/bash --group --disabled-password --home /home/username username`<br>
`su -c "psql" - postgres`<br>
Change postgresql password.<br>
`ALTER USER postgres WITH PASSWORD 'new_password';`<br>
Create app user etc.<br>
`CREATE ROLE username WITH LOGIN PASSWORD 'userpass';`<br>
`CREATE DATABASE userdatabase WITH OWNER username TEMPLATE template0 ENCODING UTF8 LC_COLLATE 'en_US.UTF-8' LC_CTYPE 'en_US.UTF-8';`<br>


### Step 3 - Configure Postgresql
Next, You need to switch to 'SCRAM-SHA-256' scheme from md5 encryption scheme for better security. If you want to connect to PostgreSQL remotely, then you need to allow your IP address in the PostgreSQL configuration file.<br>

[/etc/postgresql/versionNum/main/postgresql.conf](./resources/postgresql.conf)<br>

Next, change the following variables as per your requirement:<br>
`listen_addresses = 'localhost, IF_REMOTEIP'`<br>
`password_encryption = scram-sha-256`<br>
`sudo systemctl restart postgresql`<br>

### Step 4 - Configure Access 
At this point, your PostgreSQL setup and ready for your app user, verify authentication settings in /etc/postgresql/14/main/pg_hba.conf file. 

PostgreSQL accepts all local connections by defaults.<br>
[/etc/postgresql/versionNum/main/pg_hba.conf](./resources/pg_hba.conf)<br>
![Example](./resources/Example.webp)<br>



