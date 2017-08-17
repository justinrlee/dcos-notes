# FAQs

This page is a generic list of how-to's and FAQs for DC/OS.

## How do I find the latest version of the DC/OS CLI?

DC/OS CLI releases do not strictly line up with the DC/OS versions - they're released independently.  Because of this, the DC/OS CLI links found in the DC/OS UI may not always be up to date.  You can find the latest versions here:

https://github.com/dcos/dcos-cli/releases

## How come I can't use the `dcos security` command?

The `dcos security` command comes from the dcos-enterprise-cli Universe package, and can be installed by running `dcos package install dcos-enterprise-cli --cli`.  Note that the features of this CLI add-on will only work with the Enterprise edition of DC/OS.

## How do I configure additional Mesos attributes on my nodes, for use with constraints?

The official documentation for this can be found here: https://docs.mesosphere.com/1.9/installing/faq/#q.-how-to-add-mesos-attributes-to-nodes-to-use-marathon-constraints

Alternately, you can do this on a per-node basis by:

1) Editing and/or creating /var/lib/dcos/mesos-slave-common

2) Add a line with the MESOS_ATTRIBUTES environment variable (this will be read by the dcos-mesos-slave or dcos-mesos-slave-public systemd unit):

```

MESOS_ATTRIBUTES=location:bare_metal;disk_type:ssd

```

(You can specify additional attributes by separating by semicolons)

## How do I determine what version of DC/OS a specific node is on?

Look at /opt/mesosphere/etc/dcos-version.json

## What is this `jq` command I see throughout the documentation?

'jq' is an open source command-line JSON parser.  It can be found here: https://stedolan.github.io/jq/

You can quickly install it with this:

```

curl -LO https://github.com/stedolan/jq/releases/download/jq-1.5/jq-linux64
chmod 755 jq-linux64
sudo cp -p jq-linux64 /usr/bin/jq

```

## My agents are complaining about certificate issues when trying to obtain items from URIs.  How do I fix this?

If you're using self-signed or custom certificates on your artifact repositories, you can configure your agents to trust them (in DC/OS 1.9.x) by completing the following:

1) Copy your custom CA cert or certificate to /var/lib/dcos/pki/tls/cert (create this directory if it does not exist already)

2) Hash your certificate, and use the hash output as the filename for a soft link to the actual certificate.

For example:

```bash

mkdir -p /var/lib/dcos/pki/tls/certs
cp test.pem /var/lib/dcos/pki/tls/certs/
cd /var/lib/dcos/pki/tls/certs/
ln -s test.pem "$(openssl x509 -hash -noout -in "test.pem")".0

```

This should result in a directory that looks roughly like this:

```
[root@ip-10-10-0-138 certs]# ll
total 4
lrwxrwxrwx. 1 root root    8 Apr 26 00:33 87e86989.0 -> test.pem
-rw-r--r--. 1 root root 1285 Apr 26 00:31 test.pem
```