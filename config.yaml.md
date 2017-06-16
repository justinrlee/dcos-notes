This details the configurations in the minimum config.yaml provided in the basic walkthrough.

```yaml
---
### Identifies the URL used by pkgpanda from which to pull packages.  This must be reachable from each of the nodes.
bootstrap_url: http://172.16.125.20

### The name used by the cluster.  This can be anything.
cluster_name: 'dcos-oss'

### This indicates that we will use internal (static) zookeeper, hosted on each master node, for configuration management.  Exhibitor is the manager for zookeeper
exhibitor_storage_backend: static

### This indicates that each of the agents will use the IPs in the master_list (provided below) to reach the masters.
master_discovery: static

### This disables OAuth 2.0.  Not necessary, but useful for a first stack (OAuth 2.0 by default uses Google/Github/Microsoft requires internet access)
oauth_enabled: 'false'

### This is a list of your masters (used by agents to talk to the masters, and for masters to speak with each other).  You must have an odd number of masters, and all masters must be up and installed for initial cluster operation.
master_list:
 - 172.16.125.21
 - 172.16.125.22
 - 172.16.125.23

### This is the list of DNS resolvers used by the cluster (this is integrated the configuratoin for DC/OS's internal DNS server, Spartan)
resolvers:
- 8.8.4.4
```
