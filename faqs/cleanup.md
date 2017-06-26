# Framework Cleanup

When a framework (such as Spark / Elastic) is removed from DC/OS by clicking the `Destroy` button in the DC/OS Services UI, it tends to leave a lot of stuff hanging around in DC/OS.  You may notice that if you destroy a service and re-create it, it will either (a) have stuff left over from previous deployments or (b) will refuse to properly start up.

The root cause of this is this: many DC/OS services are started as a Marathon task, which is controlled by the Marathon framework.  These Marathon tasks will then register with the Apache Mesos masters as a new framework (distinct from the Marathon framework).  The new framework is then allowed to launch its own tasks directly against the Mesos API.  

**Clicking the `Destroy` button in the DC/OS UI will kill the framework task, but will not kill tasks that were started by that framework task.**

Here's a basic example:

* Some of the frameworks store data in zookeeper, which is the distributed datastore used by DC/OS.  Destroy

There 'proper' way to take down a framework consists of two steps:

1. Stopping/killing the framework (through the dcos cli) (note that this will render the framework unusable until you clean it up and re-start)

2. Cleaning up the framework (through the use of the janitor script) (note that this will remove resources for a framework ( all state for the framework)

This is much more destructive than killing a framework through the UI; if you kill a framework through the UI it kills the scheduler, but you can start a new instance of the scheduler with all the same parameters just by re-deploying the framework.  If you use this process to kill a framework, it will remove all state from the framework, such as topic configuration.  So you probably don't want to do this for Kafka unless you're ready to re-configure it from scratch.

So depending on your configuration for Kafka, you may or may not want to go down this path, but I wanted to provide you the options.  We can discuss tomorrow.

Here's how to do these steps:

Step 1:
The 'correct' way to perform step 1 (stop/kill the package) is to uninstall it with the dcos cli:
dcos package uninstall --app-id=<app-name> kafka

If, however, you've done 'destroyed' it in the UI, then it will be in a inactive but incomplete state.  This is because you will have killed the framework without killing its children tasks.  If you get to this point, you can identify the framework id by looking in the mesos UI, and then tear down all the children tasks with this:

> curl -v -X POST http://hostname:5050/teardown -d 'frameworkId=\<frameworkId\>'

For example:

```
curl -v -X POST leader.mesos:5050/teardown -d 'frameworkId=4f657f09-c73f-408f-8d9e-37228a0f1a8e-0003'
```

Step 2:
The janitor script is documented here: https://docs.mesosphere.com/1.9/deploying-services/uninstall/#framework-cleaner

The official way to do this is from a Docker container which has all of the dependencies.  However, due to your environment, it may be significantly easier to just run it as a python3 script.  You have to set up some environment stuff.

Also, the easiest way to do this is to run it from a master.

In DC/OS 1.8.x: (I tested this in 1.8.8, have not tested it in 1.8.5; I believe it should be roughly the same, but if you have issues, let me know:

source /opt/mesosphere/environment
/opt/mesosphere/bin/python3 janitor.py <options>

In DC/OS 1.9.0
dcos-shell # To start a 'dcos shell' which has all the env stuff
/opt/mesosphere/bin/python3 janitor.py <options>

The options you need should look something like this:

-r <role> -p <principal> -z <zookeeper-path>

So you'd run it something like this:

<whichever env setup you need>
/opt/mesosphere/bin/python3 janitor.py -r kafka-role -p kafka-principal -z dcos-service-kafka

And here's the janitor.py script: