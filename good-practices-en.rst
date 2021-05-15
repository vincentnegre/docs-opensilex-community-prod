About this document
===================

This document aims to share the best practices followed during the deployement in a production environment of the opensilex application. OpenSilex has been deployed for the `Montpellier Plant Phenotyping Platforms <https://www6.montpellier.inrae.fr/lepse_eng/M3P>`_ in the framework of the `PHENOME <https://www.phenome-emphasis.fr/phenome_eng>`_ project. The application is available at https://phis.m3p.inrae.fr


| Copyright (c) 2021 INRAE.
| This document is licensed under a  `Creative Commons Attribution 4.0 International license <https://creativecommons.org/licenses/by/4.0/>`_

Setting up a production environment
===================================

Graphdb as a service
--------------------

systemd replaces the init system V daemon for Linux. 

Its purpose is to offer a better management of dependencies between services, as well as to allow parallel loading of services at startup.


- Give appropriate rights to the opensilex user on graphdb folder:

::

  sudo chown -R opensilex.opensilex /opt/graphdb-free-9.7.0/

- Create file **/etc/systemd/system/graphdb.service**:

::

  [Unit]
  Description=graphdb service

  [Service]
  Type=forking
  User=opensilex
  ExecStart=/opt/graphdb-free-9.7.0/bin/graphdb -d
  ExecStop=kill $(ps -aux | grep graphdb | grep java |  awk '{ print $2 }')

  [Install]
  WantedBy=multi-user.target

- To run the graphdb service at startup, use the enable command :

::

  sudo systemctl enable graphdb

- You can check if the service is correclty running :

::

  sudo systemctl status graphdb

Opensilex application as a service
----------------------------------

- Create file */etc/systemd/system/opensilex.service*:

::

 [Unit]
 Description= opensilex service
 Requires=graphdb.service
 After=graphdb.service

 [Service]
 Type=oneshot
 RemainAfterExit=yes
 User=opensilex
 ExecStart=<path_to_opensilex_application>/bin/opensilex.sh server start --adminPort=4081 --port=8081
 ExecStop=<path_to_opensilex_application>/bin/opensilex.sh server stop --port=8081
 
 [Install]
 WantedBy=multi-user.target
 
<path_to_opensilex_application> corresponds to the path where opensilex is installed.

- To start opensilex service at startup, use the enable command:

::

 sudo systemctl enable opensilex

- Verify that everything works by restarting the server:

::

 sudo shutdown -r now

- Once the server is restarted check if the opensilex application is running:

::

 systemctl status opensilex
 May 08 11:37:22 phisphenoarch opensilex.sh[935]: INFO: Starting ProtocolHandler ["http-nio-8081"]

You can also open a browser and go the opensilex application at http://<IP_address_of_your_server>:8081/app/

The tag <IP_address_of_your_server> refers to the address IP of your server.

Secure MongoDB
--------------

By default MongoDB runs without authentication. Starting in version 3.6 MongoDB listens to the local interface only (127.0.0.1) so the risk is limited. But it would be dangerous  if you want to allow remote connections. In this case you should filter machines allowed to connect (see binIp directive in the configuration file). In addition you can also activate authentication as follows.

- create a key file:

::

 sudo sh -c "openssl rand -base64 756 > /var/lib/mongodb/keyfile"
 sudo chmod 400 /var/lib/mongodb/keyfile
 sudo chown mongodb:mongodb /var/lib/mongodb/keyfile

- This step is **optionnal**. It's needed only if you have different mongodb instances. You should copy  keyfile to each replicat set member

::
 
 sync -av /var/lib/mongodb/keyfile root@ip-of-your-second-instance:/var/lib/mongodb
…..

Restart each member of the replica set with access control enabled.

- Stop mongodb :

::

 sudo service mongod stop

- Update the configuration file */etc/mongod.conf*. Uncomment out the security section, and enter: 

::

 security:
     keyFile: /var/lib/mongodb/keyfile

- Start mongodb :

::

 sudo service mongod start

Create users
~~~~~~~~~~~~

You **must** be connected to the primary to create users. Run rs.status() from the mongo shell to find out which instance is the primary. When you activate authentication, you must create an admin user otherwise you will not be able to create new users. 

- Run the mongo client

::

 mongo

- Create an super admin user for monogdb :

::

 use admin;
 admin.createUser(
   {
     user: "admin",
     pwd: "set_password_for_admin_user",
     roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
   }
 )

You should see "Successfully added user" as the response.

- Logout and login with login and password:

::

 mongo -u admin -p set_password_for_admin_user
 show dbs;

- Create an admin for the database hosting opensilex application:

::

 use admin
 db.createUser(
   {
     user: "opensilex",
     pwd: "set_password_for_opensilex_user",
     roles: [ { role: "dbOwner", db: "<db_name>" } ]
   }
 )

Replace the tag <db_name> with the name of your mongo database.
 

- Add the following lines in the opensilex config configuration file ~/config/opensilex.yml (mongodb section) :

:
            #MongoDB user name
            username: opensilex

            #MongoDB password
            password: set_password_for_admin_user

- Restart opensilex application:

::
 sudo systemctl restart opensilex
 


