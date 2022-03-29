# oracle-docker
a quick guide to connect Spring Boot to an Oracle 12.2.0.1-slim Docker Image


So, you wanna run a simple local Oracle DB to use with your Spring Boot service? Maybe even with a Flyway to run your scripts up? Here, hold my beer!

* Step 1 - DockerHub: 
Search for this image store/oracle/database-enterprise:12.2.0.1 - but you are going to save some configs and storage gigabytes by useing version 12.2.0.1-slim (we will come to it later at docker-compose).
You will need to checkout and buy it. No worries, it is free (now I am writeing in March/2022). YOU NEED TO DO SO to be able to download the image, and also to generate an important Manual.
You are here to skip the steps, but I advise you to read it anyway... it isn't that big.

* Step 2 - Docker Compose YAML: 

NOTICE: all prompt commands are as run on Windows Power Shell with Administrator privileges, and the Docker is Windows Docker Desktop version.

Create a docker-compose.yml file, and notice that we are reaching for a slim version for memory sake. Also, it has a few less configs, so ports are already configured as noted below

------
version: '2'

services:
  db:
    image: store/oracle/database-enterprise:12.2.0.1-slim
    container_name: oracle12
    ports:
      - '1521:1521'
      - '5500:5500'
    expose:
      - '1521'
      - '5500'
      
 ------
 
* Step 3 - Start Oracle DB
> docker-compose up

There is a huge wait here. Time for a coffee while you wait!


* Step 4 - Connect with DBEAVER / Is it Ready Yet?

Why DBEAVER? Do I require a DBeaver? Answer is no, but I find it pretty easy to work it, but please, your kitchen, your rules

- Create a connection: for those that went with DBeaver (6.3.0) > Menu: Database > new Database Connection > select Oracle > settings: 
-- connection type: BASIC
-- host: 127.0.0.1
-- database: ORCLCDB >> and side combo must be at SID
-- username: sys >> and side combo must be at SYSDBA
-- password: Oradoc_db1

-- OPTIONAL: check is connection driver is OK at the button: Edit Driver Settings > Class Name: oracle.jdbc.driver.OracleDriver 

Now, spam the "Test Connection" button until it says "Connected"... this means that the docker image is all set up! Click OK to save the connection.

Also, on Edit Driver Setting button (OPTIONAL), the field "Libraries" should have the JAR list that your Spring Boot project downloaded as OJDBC (click "Add Folder" and point to it - nice official guides about it at https://www.oracle.com/database/technologies/oracledb-and-spring-springboot.html ) (see more about it a little below at the Gradle Step).

Common doubts here: can I change the database name, user, password, port, and so on? Not really much... there are no easy ways to do this on the slim version, but it is all ready (if you got the Connected message) to go by now. The full version has more about customization, but you will need help from a Database Mage, I am but a Java Sorcerer.


* Step 5 - Mind about Gradle

Not necessary, but a good reminder (Maven users can relate and translate): you OJDBC library (the Oracle connection driver from your project) must be OK on your project.  

The build.gradle needs a repository to grab the OJDBC library, and a proper implementarion should be called. Also, possibly a Flyway startup on booRun call.

Here is an example. Mind your own users, passwords and versions:

----------

repositories {
	mavenCentral()
	maven { url 'https://repo.spring.io/milestone' }
	maven {
		url "https://www.oracle.com/content/secure/maven/content"
		name "maven.oracle.com"
		credentials {
			username = 'some-registered-email-on-oracle@whatever.com'
			password = 'some-registered-password-on-oracle'
		}
	}
}

ext {
	set('springCloudVersion', 'Greenwich.RELEASE')
	flywayVersion = '5.2.4'
	ojdbc8Version = '18.3.0.0'
}

dependencies {
  {{ ...other libraries and configs... }}
  implementation "com.oracle.jdbc:ojdbc8:${ojdbc8Version}"
  implementation "org.flywaydb:flyway-core:${flywayVersion}"
}

----------

* Step 6 - Create your SCHEMA

After connecting on the Database, you will need to create schemas so you can start adding tables and data manually or with later with Spring Boot Flyway plugin.

Since the SID on this Oracle version is fixed as ORCLCDB, you can create SCHEMAs by creating users on it. Just run the follow SQLs (one by one, not all together, because they do clash):

---------

example: SCHEMA will be named MYPRETTYDB

alter session set "_ORACLE_SCRIPT"=true;
create user MYPRETTYDB identified by MYPRETTYDB;
ALTER SESSION SET CURRENT_SCHEMA=MYPRETTYDB;
GRANT UNLIMITED TABLESPACE TO "MYPRETTYDB" WITH ADMIN OPTION;

--------

Also, if you want to destroy the schema so you can recreate from the scratch (not needing to delete a docker volume and so on), all your need to run is:

---------

example: SCHEMA will be named MYPRETTYDB

alter session set "_ORACLE_SCRIPT"=true;
DROP USER MYPRETTYDB CASCADE;

--------

You are ready to create tables and querying by now, just select the MYPRETTYDB Schema (sql hint: ALTER SESSION SET CURRENT_SCHEMA=MYPRETTYDB)

* Step 7 - Spring Boot

Over your application.yml (or bootstrap.yml), we will need these configs under spring tree:

---

example: SCHEMA will be named MYPRETTYDB

spring:
  datasource:
    url: jdbc:oracle:thin:@127.0.0.1:1521:ORCLCDB
    username: sys AS SYSDBA
    password: Oradoc_db1
    driver:
      class: oracle.jdbc.driver.OracleDriver
    initialize: false
    hikari:
      pool-name: custom
      connection-timeout: 60000
      minimum-idle: 3
      maximum-pool-size: 100
      leak-detection-threshold: 60000
      connection-init-sql: ALTER SESSION SET CURRENT_SCHEMA=MYPRETTYDB      
  jpa:
    database-platform: org.hibernate.dialect.Oracle12cDialect
    hibernate:
      ddl-auto: none
    properties:
      dialect: org.hibernate.dialect.Oracle12cDialect


NOTICE: some OJDBC library versions may differ about the right place of SYSDBA

spring:
  datasource:
    url: jdbc:oracle:thin:@127.0.0.1:1521:ORCLCDB
    username: sys AS SYSDBA
    
  - OR - 

spring:
  datasource:
    url: jdbc:oracle:thin:@127.0.0.1:1521:ORCLCDB?internal_logon=sysdba
    username: sys

---

Also, you may test the spring.datasource.url as it is (if you are using the same  OJDBC driver on DBeaver as your are using on your Spring Boot config) on DBeaver > Create or Edit Connection > General > Connection Type: Custom > JDBC URL > {{paste the spring config url}} > try the TEST CONNECTION button

All set! Should be working by now. Just run > gradle bootRun or run it on your IDE and start developing things up.


* Step 8 - Flyway Config

If you are going to use Flyway, on your application.yml (or bootstrap.yml), we will need these configs under spring tree as well:

NOTICE: there are a lot of FLyway specific properties, so please check the plugin documentation, but the basics are in here! Also, "spring.flyway.locations" are an example where to save your flyway sql scripts. 

---

example: SCHEMA will be named MYPRETTYDB

example: the true filepath (Windows) in here would be like this 
C:\whatever-workspace-path\whatever-project-path\src\main\resources\db\migration\oracle\V1__init_script.sql
>> referenced on the config at "locations: classpath:db/migration/oracle"


spring:
  flyway:
    enabled: true
    baselineOnMigrate: true
    baselineVersion: 0
    url: ${spring.datasource.url}
    user: ${spring.datasource.username}
    password: ${spring.datasource.password}
    schemas:
    - MYPRETTYDB
    init-sqls:
    - ${spring.datasource.hikari.connection-init-sql}
    locations: classpath:db/migration/oracle

---

* Post Step: Re-Starting things 

Now you can start/stop the Oracle with:

{{ at the directory of your target docker-compose.yml file}} > docker-compose up
... and you can stop gracefully by pressing CTRL+C (Windows Power Shell with Administrator privileges)


* Post Steps: Cleaning Up

Sometimes you wanna save space, or just wanna destroy the whole thing to recreate, and there is a few catches for those still getting used to a Docker environment.

Docker Compose will tie a chunk of your memory as a docker volume. If you don't have any other images using persistent volumes, or is ready to destroy all volumes for a clean re-create, just go with:

{{ at the directory of your target docker-compose.yml file}} > docker-compose down -v
> docker volume prune

This stops the containers and release the volumes (docker help manual will say that the volumes are now "orphans"). Second command will delete them.


Now, the long version (also, go for it if docker volume prune isn't working for you). We will do the same thing, but we are going to destroy a specific volume.

> docker container ls
> docker inspect {{ insert oracle container id here }}
> docker volume ls
> docker volume rm {{ insert volume name-value here }}

These commands will do the following:

> will list the containers, copy the CONTAINER_ID for later use (something like 0f2b33881ebd)
> A huge JSON will show up. Search for "Mounts". Inside its clauses will be a "Type": "volume", and a "name" (probably a "source" too), with a huge hash, Copy this name-value (something like 7d69db92f6fd0d6fb6d797afa4e69f86c7a72b58824b8ffa81afa1398c8fb4bf)
> will list your docker volumes. Make sure the name-value you just coppied is in there

There you go. Now, just go to the directory of your target docker-compose.yml file and start it with (docker-compose will cry a bit on the log, but will recreate the new volume - no need to wait for a huge Oracle image download)
> docker-compose up --remove-orphans


* COPY PASTE OF THE DOCKER HUB ORACLE IMAGE MANUAL (as generated after I "bought" it)

Oracle Database Enterprise Edition | Fri May 22 2020
Oracle Database Server Docker Image Documentation
Oracle Database Server 12c R2 is an industry leading relational database server. The Oracle Database Server Docker Image contains the Oracle Database Server 12.2.0.1 Enterprise Edition running on Oracle Linux 7. This image contains a default database in a multitenant configuration with one pdb.

For more information on Oracle Database Server 12c R2 refer to http://docs.oracle.com/en/database/

Using this image
Accepting the terms of service
From the store.docker.com website accept Terms of Service for Oracle Database Enterprise Edition.

Login to Docker Store
Login to Docker Store with your credentials

$ docker login

Starting an Oracle Database Server instance
Starting an Oracle database server instance is as simple as executing

$ docker run -d -it --name <Oracle-DB> store/oracle/database-enterprise:12.2.0.1

where <Oracle-DB> is the name of the container and 12.2.0.1 is the Docker image tag.

The database server is ready to use when the STATUS field shows (healthy) in the output of docker ps.

Connecting to the Database Server Container
The default password to connect to the database with sys user is Oradoc_db1.

Connecting from within the container
The database server can be connected to by executing SQL*Plus,

$ docker exec -it <Oracle-DB> bash -c "source /home/oracle/.bashrc; sqlplus /nolog"

Connecting from outside the container
The database server exposes port 1521 for Oracle client connections over SQLNet protocol and port 5500 for Oracle XML DB. SQLPlus or any JDBC client can be used to connect to the database server from outside the container.

To connect from outside the container start the container with -P or -p option as,

$ docker run -d -it --name <Oracle-DB> -P store/oracle/database-enterprise:12.2.0.1

option -P indicates the ports are allocated by Docker. The mapped port can be discovered by executing

$ docker port <Oracle-DB> 1521/tcp -> 0.0.0.0:<mapped host port>

Using this <mapped host port> and <ip-address of host> create tnsnames.ora in the directory pointed to by environment variable TNS_ADMIN.

ORCLCDB=(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=<ip-address of host>)(PORT=<mapped host port>))
    (CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=ORCLCDB.localdomain)))
ORCLPDB1=(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=<ip-address> of host)(PORT=<mapped host port>))
    (CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=ORCLPDB1.localdomain)))
To connect from outside the container using SQL*Plus,

$ sqlplus sys/Oradoc_db1@ORCLCDB as sysdba

Custom Configurations
The Oracle database server container also provides custom configuration parameters for starting up the container. All the custom configuration parameters are optional. The following list of custom configuration parameters can be provided in the ENV file (ora.conf).

DB_SID
This parameter changes the ORACLE_SID of the database. The default value is set to ORCLCDB.

DB_PDB
This parameter modifies the name of the PDB. The default value is set to ORCLPDB1.

DB_MEMORY
This parameter sets the memory requirement for the Oracle server. This value determines the amount of memory to be allocated for SGA and PGA. The default value is set to 2GB.

DB_DOMAIN
This parameter sets the domain to be used for database server. The default value is localdomain.

To start an Oracle database server with custom configuration parameters

$ docker run -d -it --name <Oracle-DB> -P --env-file ora.conf store/oracle/database-enterprise:12.2.0.1

Ensure custom values for DB_SID, DB_PDB and DB_DOMAIN are updated in the tnsnames.ora.

Caveats
This Docker image has the following restrictions.

Supports a single instance database.
Dataguard is not supported.
Database options and patching are not supported.
Changing default password for SYS user
The Oracle database server is started with a default password Oradoc_db1. The password used during the container creation is not secure and should be changed. To change the password connect to the database with SQL*Plus and execute

alter user sys identified by <new-password>;

Resource Requirements
The minimum requirements for the container is 8GB of disk space and 2GB of memory.

Database Logs
The database alert log can be viewed with

$ docker logs <Oracle-DB>

where

Reusing existing database
This Oracle database server image uses Docker data volumes to store data files, redo logs, audit logs, alert logs and trace files. The data volume is mounted inside the container at /ORCL. To start a database with a data volume using docker run command,

$ docker run -d -it --name <Oracle-DB> -v OracleDBData:/ORCL store/oracle/database-enterprise:12.2.0.1

OracleDBData is the data volume that is created by Docker and mounted inside the container at /ORCL. The persisted data files can be reused with another container by reusing the OracleDBData data volume.

Using host system directory for data volume
To use a directory on the host system for the data volume,

$ docker run -d -it --name <Oracle-DB> -v /data/OracleDBData:/ORCL store/oracle/database-enterprise:12.2.0.1

where /data/OracleDBData is a directory in the host system.

Oracle Database Server 12.2.0.1 Enterprise Edition Slim Variant
The Slim Variant (12.2.0.1-slim tag) of EE has reduced disk space (4GB) requirements and a quicker container startup. This image does not support the following features - Analytics, Oracle R, Oracle Label Security, Oracle Text, Oracle Application Express and Oracle DataVault. To use the slim variant

$ docker run -d -it --name <Oracle-DB> store/oracle/database-enterprise:12.2.0.1-slim

where <Oracle-DB> is the name of the container and 12.2.0.1-slim is the Docker image tag.
