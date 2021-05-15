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
