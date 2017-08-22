# Spark Notes
This document is very incomplete and draft-like.  It's meant as a rolling document for different ways to run Spark jobs on DC/OS.  It's mostly notes for myself, provided with no guarantees of anything.

Reminder: DC/OS essentially consists of many additional services packaged to interact with Apache Mesos, so all of these essentially boil down to different ways to run Spark on Apache Mesos, adjusted for the DC/OS environment.

This document currently focuses on using spark submit (i.e., not using an interactive shell.  Interactive shell may be added later).

This document indicates several different ways to run Spark jobs.

According to the Spark docs, there are two primary high-level ways of running Spark jobs against Mesos:

1. Client Mode: Start a Spark driver on the calling client machine.  We'll look at:

    1. Running Spark jobs from outside the cluster, in client mode
    2. Running Spark jobs from inside the cluster, in client mode, using hdfs
    3. Running Spark jobs from a Docker container in the cluster
    4. Running Spark jobs from a Marathon job, in client mode
    5. Running Spark jobs from a Marathon job as a Docker container, in client mode

2. Cluster mode: Start a Spark driver as a task hosted in the cluster.  This requires a dispatcher.  We'll look at:

    1. Running Spark jobs from the dcos command line using the `dcos spark run` command, which runs jobs in cluster mode
    2. Running Spark jobs from outside the cluster in cluster mode
    3. Running Spark jobs from inside the clsuter, in cluster mode, using hdfs
    4. Some other stuff.


First, we'll look at running Spark jobs in standalone/local mode (without Mesos), and then we'll address each of the above situations.

*This document assumes everything is run as root, for ease of use.  Maybe I'll add proper permissions and stuff later.*

***

# Local Mode: Run Spark jobs without Mesos (Spark Standalone)

This section is primarily meant to introduce the job that we'll be running throughout this document.

*Note that 'Standalone Mode' can be used to mean building out a Spark cluster of multiple nodes (no YARN or Mesos), and running jobs on this cluster.  This part of the document does **not** use a cluster like this; instead, we specify `--master local` to run jobs locally without a cluster.*


First, download and extract the Spark binaries and place them somewhere accessible (such as /opt)
```
# Download the Mesos-specific Spark binaries
curl -LO https://downloads.mesosphere.com/spark/assets/spark-2.2.0-bin-2.6.tgz
tar -xzvf spark-2.2.0-bin-2.6.tgz
mv spark-2.2.0-bin-2.6 /opt/spark
```

Add the /opt/spark/bin directory to your PATH, so that commands can be run directly
```
echo 'PATH=$PATH:/opt/spark/bin' >> ~/.bash_profile
sed -i '/export PATH/d' ~/.bash_profile 
echo "export PATH" >> ~/.bash_profile
```

Download the jar file that contains all of our examples
```
curl -LO https://downloads.mesosphere.com/spark/assets/spark-examples_2.11-2.0.1.jar
```

Submit it to the local 'cluster':
```
spark-submit \
    --master local \
    --class org.apache.spark.examples.SparkPi \
    spark-examples_2.11-2.0.1.jar \
    30
```

#### So what is this actually doing?

1. We're downloading the Spark binaries, and extracting them
2. We're setting up our local path to point to the spark bin directory
3. We're downloading the spark example jar file
4. We're running spark-submit, with the following settings:
    * Use 'local' as our master (don't send jobs elsewhere)
    * Run class `org.apache.spark.examples.SparkPi`
    * Provide the class with the local jar file
    * Provide the parameter '30' to our job (indicating that we should run the example with 30 slices)

What could we do differently, for local run / what errors may we run into?

* We can add additional parameters to the executor configuration
* We could try to auto-fetch the url from https, but Spark by default only supports local, S3, and hdfs:
* We can change the parameters (this is trivial and boring)

***

# Client Mode
The official [Apache Mesos documentation](https://spark.apache.org/docs/latest/running-on-mesos.html) says this: 

>In client mode, a Spark Mesos framework is launched directly on the client machine and waits for the driver output.

What this means is that on the machine that the `spark-submit` command is run, we are starting a mini service that registers with Apache Mesos as a framework.  This service will then receive and accept resource offers from Apache Mesos, and will schedule tasks (part of the job) on the Mesos agents based on those resources offers.

***
<!-- *** -->

## Client Mode: Running Spark jobs from outside the cluster, in client mode

#### The first several parts of this are identical to running Spark jobs in local standalone mode.

First, download and extract the Spark binaries and place them somewhere accessible (such as /opt)
```
curl -LO https://downloads.mesosphere.com/spark/assets/spark-2.2.0-bin-2.6.tgz
tar -xzvf spark-2.2.0-bin-2.6.tgz
mv spark-2.2.0-bin-2.6 /opt/spark
```

Add the /opt/spark/bin directory to your PATH, so that commands can be run directly
```
echo 'PATH=$PATH:/opt/spark/bin' >> ~/.bash_profile
sed -i '/export PATH/d' ~/.bash_profile 
echo "export PATH" >> ~/.bash_profile
```

Download the jar file that contains all of our examples
```
curl -LO https://downloads.mesosphere.com/spark/assets/spark-examples_2.11-2.0.1.jar
```

#### In order to run in Client mode, the following changes must be made:

You must obtain these files from somewhere (they can be copied from an existing Mesos node, where they live in `/opt/mesosphere/lib/`):

* libaprutil-1.so.0
* libcrypto.so.1.0.0
* libssl.so.1.0.0
* libapr-1.so.0
* libsasl2.so.2
* libsvn_subr-1.so.1
* libsvn_delta-1.so.1
* libmesos.so

Put them all in one directory (e.g. /usr/lib/mesos), then export the path to that directory as an environment variable:

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/mesos
```

Then, run the `spark-submit` command like this (you must specify the correct IP addresses for your mesos zookeeper instances):

```
spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://zk://10.10.0.136:2181,10.10.0.50:2181,10.10.0.50:2181/mesos \
  --conf spark.executor.uri=https://downloads.mesosphere.com/spark/assets/spark-2.2.0-bin-2.6.tgz \
  --deploy-mode client \
  spark-examples_2.11-2.0.1.jar 400
```

So what are we doing differently?
* We're pointing master at the Mesos zookeeper cluster (this requires that we set up and point to the mesos libraries)
* We're running in client mode
* Providing a mesos executor URI to download to each Mesos task (to run the stuff)

What else can we do?
* If we only have a single master, we could change `mesos://zk://10.10.0.136:2181,10.10.0.50:2181,10.10.0.50:2181/mesos` to `mesos://HOSTNAME:5050`
* If we had hdfs up and running, we could download the jar file from hdfs
* If we had s3 up and running, we could download the jar from s3

We can also run with a Docker image instead of providing an executor URI:

```
spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://zk://10.10.0.136:2181,10.10.0.50:2181,10.10.0.50:2181/mesos \
  --conf spark.mesos.executor.docker.image=mesosphere/spark:1.1.1-2.2.0-hadoop-2.7 \
  --conf spark.mesos.executor.home=/opt/spark/dist \
  --deploy-mode client \
  spark-examples_2.11-2.0.1.jar 400
```

And we can configure the task to run with the 'mesos' containerizer:
```
spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://zk://10.10.0.136:2181,10.10.0.50:2181,10.10.0.50:2181/mesos \
  --conf spark.mesos.executor.docker.image=mesosphere/spark:1.1.1-2.2.0-hadoop-2.7 \
  --conf spark.mesos.executor.home=/opt/spark/dist \
  --conf spark.mesos.containerizer=mesos \
  --deploy-mode client \
  spark-examples_2.11-2.0.1.jar 400
```

***

## Client Mode: Running Spark jobs from inside the cluster, in client mode, using hdfs

Next, we're gonna set up hdfs and run in client mode.  We're going to use the dcos hdfs package, and we don't have to worry about obtaining the libraries (although we will have to point to them).

So, first, set up hdfs (see: [Spark Env Setup](env.md))

Now, we should be able to directly access hdfs (this should result in an empty list):

```
hdfs dfs -ls /
```

Let's put the example jar file in hdfs:
```
curl -LO https://downloads.mesosphere.com/spark/assets/spark-examples_2.11-2.0.1.jar
hdfs dfs -put spark-examples_2.11-2.0.1.jar /
```

In order for Spark (client mode) to know how to access hdfs, you have to tell it where the hdfs configurations are:

```
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop 
```

And again, we have to tell Spark where the library files are (but this time, they already exist on the filesystem):

```
export LD_LIBRARY_PATH=/opt/mesosphere/lib
```

Now we can run our test command:

```
spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://zk://zk-1.zk:2181,zk-2.zk:2181,zk-3.zk:2181,zk-4.zk:2181,zk-5.zk:2181/mesos \
  --conf spark.executor.uri=https://downloads.mesosphere.com/spark/assets/spark-2.2.0-bin-2.6.tgz \
  --deploy-mode client \
  hdfs://hdfs/spark-examples_2.10-1.4.0-SNAPSHOT.jar 400
```

So what are we doing differently?
* We're pointing master at the Mesos zookeeper cluster using the zk DNS names (available within the cluster)
* We're accessing the jar file from hdfs, which additionally requires that:
    * The jar file has to exist on hdfs
    * We have the hdfs core-site and hdfs-site XML files
    * We tell Spark where to find the core-site.xml and hdfs-site.xml files
    * (We don't actually need to install all of the hadoop binaries to access the jar from Spark - they're primarily used to put the jar in its place) (I think).

Here are some other options:


We can also run with a Docker image instead of providing an executor URI:

```
spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://zk://zk-1.zk:2181,zk-2.zk:2181,zk-3.zk:2181,zk-4.zk:2181,zk-5.zk:2181/mesos \
  --conf spark.mesos.executor.docker.image=mesosphere/spark:1.1.1-2.2.0-hadoop-2.7 \
  --conf spark.mesos.executor.home=/opt/spark/dist \
  --deploy-mode client \
  hdfs://hdfs/spark-examples_2.10-1.4.0-SNAPSHOT.jar 400
```

And we can configure the task to run with the 'mesos' containerizer:
```
spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://zk://zk-1.zk:2181,zk-2.zk:2181,zk-3.zk:2181,zk-4.zk:2181,zk-5.zk:2181/mesos \
  --conf spark.mesos.executor.docker.image=mesosphere/spark:1.1.1-2.2.0-hadoop-2.7 \
  --conf spark.mesos.executor.home=/opt/spark/dist \
  --conf spark.mesos.containerizer=mesos \
  --deploy-mode client \
  hdfs://hdfs/spark-examples_2.10-1.4.0-SNAPSHOT.jar 400
```

***

## Client Mode: Running Spark jobs from a Docker container in the cluster

***

## Client Mode: Running Spark jobs from a Marathon job, in client mode

***

## Client Mode: Running Spark jobs from a Marathon job as a Docker container, in client mode

***

# Cluster Mode

***

## Cluster Mode: Running Spark jobs from the dcos command line using the `dcos spark run` command, which runs jobs in cluster mode

***

## Cluster Mode: Running Spark jobs from outside the cluster in cluster mode

***

## Cluster Mode: Running Spark jobs from inside the clsuter, in cluster mode, using hdfs

***

## Cluster Mode: Some other stuff.