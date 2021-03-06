# spring-xd-phd3-sandbox
Demo sandbox with with an Ambari management vm and PHD3 with 3 data nodes (single node per vm). Integrates Spring XD hdfs and GPDB in this example. Also integrates with GemFire, Redis, etc. Instructions shown are on OSX using Homwbrew where possible.

## Installation of Ambari and PHD3

Installation instructions are here:

http://blog.tzolov.net/2015/06/leverage-vagrant-and-ambari-blueprint.html
```
User          Password
-----------------------
root          vagrant
vagrant       vagrant
gpadmin       gpadmin
```
Installation has been verified verbatim on OSX.  No issues noted. However, sleep tends to bounce HBase -- maintenance mode is therefore recommended.
```
The Ambari interface is found here: http://10.211.55.100:8080 (username: admin, password: admin)
```
## Install Spring XD on OSX Host with Homebrew

http://docs.spring.io/spring-xd/docs/current/reference/html/

The only caveat with the Ambari/PHD3 installation is the Spring XD config -- you'll need to bind to namenode host adapter on eth1 (10.211.55.101), not the NAT adapter (Ambari only shows the NAT adapter on eth0). 

## Configure Spring XD on OSX Host

1) Add the following to
/usr/local/Cellar/springxd/1.2.0.RELEASE/libexec/xd/config/hadoop.properties:
```
fs.default.name=hdfs://10.211.55.101:8020 
```
2) Add or uncomment the following to
/usr/local/Cellar/springxd/1.2.0.RELEASE/libexec/xd/config/servers.yml:
```
---
# Hadoop properties
spring:
  hadoop:
  fsUri: hdfs://10.211.55.101:8020
```
3) Spin up the Spring XD service
```
xd/bin> ./xd-singlenode --hadoopDistro phd30
```
4) Spawn the Spring XD shell
```
xd/bin> ./xd-shell --hadoopDistro phd30
```
5) Configure the hdfs namenode in the Spring XD shell
```
xd:> hadoop config fs --namenode hdfs://10.211.55.101:8020
```
6) Verify hdfs from the Spring XD shell
```
Xd:> hadoop fs ls /
```
You should see something like:
```
Found 11 items
drwxrwxrwx   - yarn      hadoop           0 2015-08-19 14:15 /app-logs
drwxr-xr-x   - hdfs      hdfs             0 2015-08-19 14:18 /apps
drwxr-xr-x   - gpadmin   gpadmin          0 2015-08-19 14:17 /hawq_data
drwxr-xr-x   - mapred    hdfs             0 2015-08-19 14:15 /mapred
drwxr-xr-x   - hdfs      hdfs             0 2015-08-19 14:15 /mr-history
drwxr-xr-x   - hdfs      hdfs             0 2015-08-19 14:15 /phd
drwxr-xr-x   - hdfs      hdfs             0 2015-08-19 16:49 /retail_demo
drwxr-xr-x   - hdfs      hdfs             0 2015-08-19 14:15 /system
drwxrwxrwx   - hdfs      hdfs             0 2015-08-19 14:20 /tmp
drwxr-xr-x   - hdfs      hdfs             0 2015-08-19 14:21 /user
drwxrwxrwx   - spring-xd hdfs             0 2015-08-19 14:18 /xd
```
# Stream Data to HDFS
Recreate the the ticktock example (http://docs.spring.io/spring-xd/docs/current/reference/html/), but sink to hdfs
```
Xd:> stream create --name ticktockhdfs --definition "time | hdfs" --deploy
```
Leave it running a few seconds, then destroy the stream
```
Xd:> stream destroy --name ticktockhdfs
```
View the small file that will have been generated in hdfs
```
Xd:> hadoop fs ls /xd/ticktockhdfs  

Found 1 items -rwxr-xr-x   3 root hdfs        420 2013-10-12 17:18 /xd/ticktockhdfs/ticktockhdfs-0.log
```
Quickly examine the file
```
Xd:> hadoop fs cat /xd/ticktockhdfs/ticktockhdfs-0.log  
2013-10-12 17:18:09 2013-10-12 17:18:10 2013-10-12 17:18:11 2013-10-12 17:18:12 2013-10-12 17:18:13 2013-10-12 17:18:14
```
###### HDFS Sink Partioning
https://github.com/spring-projects/spring-xd-samples/tree/master/hdfs-partitioning

# Stream Data to GPDB using HAWQ and gpfdist

The following is generally based on the gfdist example shown here:
https://github.com/spring-projects/spring-xd/blob/master/src/docs/asciidoc/Sinks.asciidoc#gpfdist

1) On the HAWQ master (node phd3), add or edit /data/hawq/master/gpseg-1/pg_hba.conf to accept remote connections from vboxnet0 host (10.211.55.1)
```
host  all  gpadmin  10.211.55.1/32  trust
```
2) Login to phd3 (the HAWQ master) as gpadmin

3) Reload the config
```
[gpadmin@phd3 ~]$ gpstop –u
```
4) Verify psql
```
[gpadmin@phd3 ~]$ psql
psql (8.2.15)
Type "help" for help.

gpadmin=#
```
5) Use psql to create table 'xdsink'
```
gpadmin=# create table xdsink (pid integer, time timestamp, tid integer, sample real, sequence integer) distributed randomly;
```
6) Build the gpfdist load generator in this repository
###### Building with Maven
```
$ mvn package
```
7) Load the module
```
xd:>module upload --type source --name load-generator-gpfdist --file /<path-to-module>/target/load-generator-gpfdist-source-1.0.0.BUILD-SNAPSHOT.jar
```
Verify the module is loaded correctly
```
xd:>module info source:load-generator-gpfdist
Information about source module 'load-generator-gpfdist':

  Option Name      Description                                                  Default    Type
  ---------------  -----------------------------------------------------------  ---------  --------
  sleepCount       the record count after sleep                                 0          int
  recordType       the type of records in message to send, [counter,timestamp]  timestamp  String
  sleepTime        the time in millis to sleep                                  0          long
  producers        the number of producers                                      1          int
  recordDelimiter  the delimiter of records in message to send                  \t         String
  messageCount     the number of messages to send per producer                  100        int
  recordCount      the count of records in message to send                      1          int
  outputType       how this module should emit messages it produces             <none>     MimeType
```
8) Create and deploy a stream for the source module "load-generator-gpfdist"
```
xd:>stream create --name gpfdiststream --definition "load-generator-gpfdist --messageCount=20 --producers=5 --recordType=counter  | gpfdist --dbHost=10.211.55.103 --table=xdsink --batchTimeout=5 --batchCount=10 --batchPeriod=0 --flushCount=200 --flushTime=2 --rateInterval=1000000" --deploy
```
Verify i/o in the terminal the Spring XD service is running in.

9) Run for a few seconds, then undeploy
```
xd:> stream undeploy --name gpfdiststream
```
10) Verify results exist in table "xdsink" with psql
```
gpadmin=# select * from xdsink;

 pid |          time           | tid |   sample   | sequence 
-----+-------------------------+-----+------------+----------
   4 | 2015-09-16 19:21:05.516 | 149 |   0.414842 |        0
   2 | 2015-09-16 19:21:05.515 | 147 |   0.963706 |        0
   1 | 2015-09-16 19:21:05.515 | 146 |   0.893071 |        0
   3 | 2015-09-16 19:21:05.517 | 148 |   0.730467 |        1
   2 | 2015-09-16 19:21:05.517 | 147 |   0.294622 |        1
   1 | 2015-09-16 19:21:05.517 | 146 |  0.0944971 |        1
   0 | 2015-09-16 19:21:05.517 | 145 |   0.665101 |        1
   2 | 2015-09-16 19:21:05.517 | 147 |   0.142871 |        2
   3 | 2015-09-16 19:21:05.517 | 148 |   0.929922 |        2
   0 | 2015-09-16 19:21:05.517 | 145 |  0.0841604 |        2
   1 | 2015-09-16 19:21:05.517 | 146 |   0.521306 |        2
   2 | 2015-09-16 19:21:05.517 | 147 |   0.382493 |        3
   3 | 2015-09-16 19:21:05.517 | 148 |   0.504931 |        3
   2 | 2015-09-16 19:21:05.517 | 147 |   0.693706 |        4
   1 | 2015-09-16 19:21:05.517 | 146 |   0.855409 |        3
   3 | 2015-09-16 19:21:05.517 | 148 |   0.166996 |        4
   0 | 2015-09-16 19:21:05.517 | 145 |   0.173563 |        3
   2 | 2015-09-16 19:21:05.517 | 147 |   0.173133 |        5
   3 | 2015-09-16 19:21:05.517 | 148 |   0.835099 |        5
   2 | 2015-09-16 19:21:05.517 | 147 |   0.862009 |        6
   0 | 2015-09-16 19:21:05.517 | 145 |   0.138441 |        4
```
