# How-To-Remotely-Sqlplus-Expdp-Impdp-Oracle-DB-In-Kubernetes
The way to remotely do Oracle DB tasks In Kubernetes
###  Requirement:
Sometimes we can't get into DB container, we need to test from apps side remotely sqlplus into Oracle DB docker container to check DB status...etc all kind of things for debugging. Also we need to use expdp impdp to migrate data. However apps base image won't have such tools like sqlplus installed as we mean to keen running  images as slim as possible. How can we sqlplus into the DB without such tools?
###  Solution:
 Create our own container with all tools we need and attach our container to network of apps container.

Here are some details
* docker run -itd --name debug oraclelinux:7-slim
* docker exec -it debug /bin/bash
* <debug container># yum install  oracle-instantclient18.3-sqlplus.x86_64  oracle-instantclient18.3-tools-18.3.0.0.0-1.x86_64
* Please refer Oracle blog how to yum install oracle instant client link
* Please set Oracle Sqlplus related enviroment variables in the .bash_profile.
```
export https_proxy=http://www-proxy.us.test.com:80/
export http_proxy=http://www-proxy.us.test.com:80/
export NLS_LANG=AMERICAN_AMERICA.UTF8
export LD_LIBRARY_PATH=/usr/lib/oracle/18.3/client64/lib:$LD_LIBRARY_PATH
export PATH=/usr/bin:/bin:/usr/lib/oracle/18.3/client64/bin:$PATH
export TNS_ADMIN=/root
```
* create tnsnames.ora  under /root as we define above and add tns entry for your DB
```
echo "altest=(DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = livesqlsb-db-service)(PORT = 1521))(CONNECT_DATA =(SERVER = DEDICATED)(SERVICE_NAME = testdb)))" >> /root/tnsnames.ora
```
* exit container
* docker commit debug henry-swiss-knife:v1
* later you can add more tools into your own container image.Then use this henry-swiss-knife to attach network stack of kubernetes
* use docker ps |grep apex   ( find out container id of K8S pod of apexords which is the example). In this case it is 44c780d348bd  ( the pod with  "/pause")
```
  [root@instance-cas-mt2 ~]# docker ps|grep apexords
340722fe6f77        4b39de352b36                                                         "/bin/sh -c $ORDS_HO…"   18 hours ago        Up 18 hours                                  k8s_apexords_apexords_default_8b06d971-cb89-11e8-a112-000017010a8f_0
44c780d348bd        container-registry.oracle.com/kubernetes_developer/pause-amd64:3.1   "/pause"                 18 hours ago        Up 18 hours                                  k8s_POD_apexords_default_8b06d971-cb89-11e8-a112-000017010a8f_0
```
* docker run -itd --name debug --net=container:44c780d348bd henry-swiss-knife:v1
```
[root@instance-cas-mt2 ~]# docker run -itd --name debug --net=container:44c780d348bd henry-swiss-knife:v1
904180885ae527b3fc4f34a319ab6dfae39af960e29b2dbf7a5902ead55684e8
```
* docker exec -it 90418 /bin/bash    (get into the debug container to debug K8S DB related)
* Inside the debug container ,you can use sqlplus ,expdp ,impdp to handle data...etc like doing normal DB related tasks:
```
<debug container># sqlplus testuser/password@altest   --- to check any data
<debug container># expdp testuser/password@altest     --- to export data
<debug container># impdp testuser/password@altest     --- to import data
```
