etcd-aws-cluster
==============

This container serves to assist in the creation of an etcd (2.x) cluster from an AWS auto scaling group. It writes a file to /etc/sysconfig/etcd-peers that contains parameters for etcd:

- ETCD_INITIAL_CLUSTER_STATE
  - either `new` or `existing`   
  - used to specify whether we are creating a new cluster or joining an existing one
- ETCD_NAME
  - the name of the machine joining the etcd cluster
  - this is obtained by getting the instance if from amazon of the host (e.g. i-694fad83)
- ETCD_INITIAL_CLUSTER
  - this is a list of the machines (id and ip) expected to be in the cluster, including the new machine
  - e.g., "i-5fc4c9e1=http://10.0.0.1:2380,i-694fad83=http://10.0.0.2:2380"
- ETCD_FORCE_NEW_CLUSTER
  - This gets set if the backup bucket contains a backup, but we want to ignore the existing config and just use the relevant data.
  - e.g., "true"
- ETCD_DATA_DIR
  - Can also be passed into the container to override the default directory.
  - e.g. default, "/var/lib/etcd2"
- ETCD_RESTORED
  - This gets set if the backup was performed, etcd will ignore it.
  - e.g., "my-bucket"

This file can then be loaded as an EnvironmentFile in an etcd2 drop-in to properly configure etcd2:

```
[Service]
EnvironmentFile=/etc/sysconfig/etcd-peers
```

It also writes a file to /etc/sysconfig/etcd-tools that contains parameters for flannel and etcdctl:

- ETCDCTL_ENDPOINT
  - This configures the etcdctl client to use the proper protocol.
- FLANNELD_ETCD_ENDPOINTS
  - This provides the endpoints that flannel should be reading/writing from/to.


This file can then be loaded as an EnvironmentFile in an flanneld drop-in to properly configure flannel and etcdctl:

```
[Service]
EnvironmentFile=/etc/sysconfig/etcd-tools
```

Workflow
--------

- get the instance id and ip from amazon
- fetch the autoscaling group this machine belongs to
- obtain the ip of every member of the auto scaling group
- for each member of the autoscaling group detect if they are running etcd and if so who they see as members of the cluster

  if no machines respond OR there are existing peers but my instance id is listed as a member of the cluster  

    - assume that this is a new cluster
    - write a file using the ids/ips of the autoscaling group 
  
  else 

    - assume that we are joining an existing cluster
    - check to see if any machines are listed as being part of the cluster but are not part of the autoscaling group
      -  if so remove it from the etcd cluster  
    - add this machine to the current cluster
    - write a file using the ids/ips obtained from query etcd for members of the cluster


Usage
-----

```docker run -v /etc/sysconfig/:/etc/sysconfig/ monsantoco/etcd-aws-cluster```

Environment Variables
* PROXY_ASG - If specified forces into proxy=on and uses the vaue of PROXY_ASG as the autocaling group that contains the master servers
* ASG_BY_TAG - If specified in conjunction with PROXY_ASG uses the value of PROXY_ASG to look up the server by the Tag Name
* ETCD_CLIENT_SCHEME - defaults to http
* ETCD_PEER_SCHEME - defaults to http
* BACKUP_BUCKET - no default, skips tasks related to s3 without it.


Demo
----

We have created a CloudFomation script that shows sample usage of this container for creating a simple etcd cluster: https://gist.github.com/tj-corrigan/3baf86051471062b2fb7
