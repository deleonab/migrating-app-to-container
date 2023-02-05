Deploying Application Using Docker

MySQL in container
Step 1: Pull MySQL Docker Image from Docker Hub Registry
We shall pull the mysql image from the dockerhub repository(public) into our local server machine.
```
[sudo] password for deleonab: 
latest: Pulling from mysql/mysql-server
Digest: sha256:d6c8301b7834c5b9c2b733b10b7e630f441af7bc917c74dba379f24eeeb6a313
Status: Image is up to date for mysql/mysql-server:latest
docker.io/mysql/mysql-server:latest
```
We will run a container from this image and setup the mysqldb environment variables

```
deleonab@DESKTOP-PURLK18:/mnt/c/Users/deles/Documents/migrating-to-the-cloud-with-containerisation$ sudo docker run --name mysqlbackend -e MYSQL_ROOT_PASSWORD=password -d mysql/mysql-server:latest
ebdaa40430d85b0a1095dc329d03481c4ecfcda937570ca80c6b8e1a668dd124
```

CONNECTING TO THE MYSQL DOCKER CONTAINER
We can either connect directly to the container running the MySQL server or use a second container as a MySQL client. 

Method 1
```
deleonab@DESKTOP-PURLK18:/mnt/c/Users/deles/Documents/migrating-to-the-cloud-with-containerisation$ sudo docker exec -it mysqlbackend mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 20
Server version: 8.0.32 MySQL Community Server - GPL

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>

```

Method 2
At this stage, we will need to add a network. 
```

deleonab@DESKTOP-PURLK18:/mnt/c/Users/deles/Documents/migrating-to-the-cloud-with-containerisation$ docker network create --subnet=172.18.0.0/24 tooling_app_network
```

Now we shall run the MySQL Server container using the created network.

First, let us create an environment variable to store the root password:

$ export MYSQL_PW=password

Then, pull the image and run the container
```
deleonab@DESKTOP-PURLK18:/mnt/c/Users/deles/Documents/migrating-to-the-cloud-with-containerisation$ sudo docker run --network tooling_app_network -h mysqlserverhost --name=mysql-server -e MYSQL_ROOT_PASSWORD=$MYSQL_PW  -d mysql/mysql-server:latest
ae7ba02a33f6705139e5fe17b3211bce59765f40929a9b05142789d1ffcad152
```
This spins up a mysql client container and connects to the mysql-server via the mysqlserverhost connection

By executing into the client mysql container and run


As you already know, it is best practice not to connect to the MySQL server remotely using the root user. We will create an SQL script that will create a user that can connect remotely.

rWe shall create a file create_user.sql and add the following code into the file:

$ CREATE USER 'dele'@'%' IDENTIFIED BY ''; GRANT ALL PRIVILEGES ON * . * TO 'dele'@'%';

```
deleonab@DESKTOP-PURLK18:/mnt/c/Users/deles/Documents/migrating-to-the-cloud-with-containerisation$ sudo docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < create_user.sql
mysql: [Warning] Using a password on the command line interface can be insecure.
``` 


The next task is to prepare database schema that our tooling application could connect to

We shall clone the Tooling-app repository from here
```
$ git clone https://github.com/darey-devops/tooling.git
```
We shall create an envoronment variable of the path to the database schema tooling_db_schema.sql
```
$ export tooling_db_schema=tooling/html/tooling_db_schema.sql 

```


Next we use the SQL script to create the database and prepare the schema. 
```
$ docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < $tooling_db_schema 
```


Let's update the .env file with connection details to the database The .env file is located in the html tooling/html/.env folder but not visible in terminal. you can use vi or nano
``
sudo vi .env
```
```
MYSQL_IP=mysqlserverhost MYSQL_USER=username MYSQL_PASS=client-secrete-password MYSQL_DBNAME=toolingdb
```

Let's run the tooling app using a dockerfile

```

FROM php:7-apache

ENV MYSQL_IP=$MYSQL_IP
ENV MYSQL_USER=$MYSQL_USER
ENV MYSQL_PASS=$MYSQL_PASS
ENV MYSQL_DBNAME=$MYSQL_DBNAME

RUN docker-php-ext-install mysqli
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
COPY apache-config.conf /etc/apache2/sites-available/000-default.conf
COPY start-apache /usr/local/bin
RUN a2enmod rewrite

# Copy application source
COPY html /var/www
RUN chown -R www-data:www-data /var/www

CMD ["start-apache"]

```

```
sudo docker build -t tooling:0.0.1 .
```

```
sudo docker run --network tooling_app_network -p 8085:80 -it tooling:0.0.1
```

![tooling login](Tooling-login.png)