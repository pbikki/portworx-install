# Portworx maintenance during worker upgrade

### Action   
Worker node is replaced or upgraded on ROKS cluster
Eg: **OpenShift worker update from 4.3 --> 4.4**

Portworx status
```
▶ oc exec -it portworx-pw2jw   -- /bin/bash
[root@kube-bs6tcs3d0v62g-roks-default-0000041f /]# /opt/pwx/bin/pxctl status
Status: PX is operational
Cluster Summary
	Cluster ID: px-storage-cluster
	Cluster UUID: 8e24f746-7db3-4a4a-bc2e-96d5f
	Scheduler: kubernetes
	Nodes: 3 node(s) with storage (2 online), 1 node(s) without storage (1 online)
	IP		ID					SchedulerNodeName	StorageNode	Used		Capacity	Status	StorageStatus		Version		Kernel				OS
	<ip1>	a3f388da-726d-445f-b8aa-61a99547ff06	<ip1>		Yes		10 GiB		128 GiB		Online	Up			2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
	<ip2>	0d17f54d-bd24-4c75-92eb-34c57b02b18e	<ip2>		Yes		11 GiB		128 GiB		Online	Up			2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
(old)->	<ip3>	047c92bc-3f81-4106-bf44-5e0a45f2fae8	<ip3>		Yes		Unavailable	Unavailable	Offline	Down			2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
------>	<ip3>	691592d5-9cea-4240-b8ed-8b064df87379	<ip3>		No		0 B		0 B		Online	No Storage (This node)	2.5.2.0-176ddb7	3.10.0-1127.18.2.el7.x86_64	Red Hat
	Warnings: 
		 WARNING: Persistent journald logging is not enabled on this node.
Global Storage Pool
	Total Used    	:  21 GiB
	Total Capacity	:  256 GiB

```

### Effects 
- After the worker update, old worker is deleted and new worker is provisioned (IP is preserved in this case)
- Portworx pod comes up on the worker
- The VPC block storage attachment is lost because the update replaced the old worker with a new one 
- `pxctl status`, status of pwx changed. The old entry shows up as `Unavailable`. And, the new worker is added as a storage less node to the pwx cluster.  



## Solution

- Reattach block volume to the new worker that is created
The beta plugin `ibmcloud ks storage attachment` does not accept cluster name. Use cluster ID
Docs - https://cloud.ibm.com/docs/containers?topic=containers-cli-plugin-kubernetes-service-cli#cs_storage
```
# Target VPC gen2 infra
▶ ibmcloud target ibmcloud is target --gen 2

# Get clusters
▶ ibmcloud oc clusters

# Get workers in a cluster
▶ ibmcloud ks workers --cluster <cluster-name>

# List volumes VPC volumes 
▶ ibmcloud is volumes

# Attach volume to worker
ibmcloud ks storage attachment create --cluster <CLUSTER-ID> --volume <VOLUME-ID> --worker <WORKER-ID>

▶ ibmcloud ks storage attachment create --cluster <clusterid>  --volume r006-f08b16b1-0c95-4a99-a4aa-af4766 --worker kube-bs6tcs3d0-default-0000041f
Creating volume attachment...
Volume attachment was created...
OK
                  
ID:            0717-f5f59452-43a7-46cc-836f-1e98   
Name:          volume-attachment   
Status:        attaching   
Type:          data   
Volume ID:     r006-f08b16b1-0c95-4a99-a4aa-af466   
Volume Name:   pwx-dal1-vol   
Worker ID:     kube-bs6tcs3d0v6dn2g-default-0000041f   
Cluster:       <clusterid>  
```

- Restart portworx on the new node.
To do this, labelled the worker node with `px/service=restart`
```
▶ kubectl label node <ip3> px/service=restart
```
After the above, pod changes to non-ready state and comes back to ready state

PX recognizes that the replaced worker now has storage attached and is now `online`

```
▶ oc exec -it portworx-pw2jw   -- /bin/bash
[root@kube-bs6tcs3d0v2g-default-0000041f /]# /opt/pwx/bin/pxctl status
Status: PX is operational
Cluster Summary
	Cluster ID: px-storage-cluster
	Cluster UUID: 8e24f746-7db3-4a4a-bc2e-968b7
	Scheduler: kubernetes
	Nodes: 3 node(s) with storage (3 online), 1 node(s) without storage (0 online)
	IP		ID					SchedulerNodeName	StorageNode	Used		Capacity	Status	StorageStatus	Version		KernelOS
	<ip1>	a3f388da-726d-445f-b8aa-61a99547ff06	<ip1>	Yes		10 GiB		128 GiB		Online	Up		2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
	<ip2>	0d17f54d-bd24-4c75-92eb-34c57b02b18e	<ip2>		Yes		11 GiB		128 GiB		Online	Up		2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
------>	<ip3>	047c92bc-3f81-4106-bf44-5e0a45f2fae8	<ip3>		Yes		10 GiB		128 GiB		Online	Up (This node)	2.5.2.0-176ddb7	3.10.0-1127.18.2.el7.x86_64	Red Hat
(old)->	<ip3>	691592d5-9cea-4240-b8ed-8b064df87379	<ip3>		No		Unavailable	Unavailable	Offline	No Storage	2.5.2.0-176ddb7	3.10.0-1127.18.2.el7.x86_64	Red Hat
	Warnings: 
		 WARNING: Persistent journald logging is not enabled on this node.
Global Storage Pool
	Total Used    	:  31 GiB
	Total Capacity	:  384 GiB
```

If portworx is operational and the old entry still shows up, it will eventually be gone. It doesn't affect the portworx status. 

## References

- Refer to - https://github.com/mumutyal/px-utils for more comprehensive custom scripts around px maintenance

- https://docs.portworx.com/portworx-install-with-kubernetes/operate-and-maintain-on-kubernetes/troubleshooting/troubleshoot-and-get-support/