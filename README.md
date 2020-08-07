# portworx-install
This document provides instructions to install portworx on VPC Gen2 managed OpenShift Cluster on IBM Cloud (Similar steps can be followed for VPC IKS cluster).

The steps below are created following the ibm cloud docs below. Refer to these for more info and detailed steps.
- [Portworx SDS with RH Openshift on IBM Cloud](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx) 
- [Portworx SDS with IKS on IBM Cloud](https://cloud.ibm.com/docs/containers?topic=containers-portworx)

 > NOTE: The below steps are only created for reference. The portworx setup for production usecases might involve additional steps and configuration.

## Prerequisites
- A [Managed openshift cluster (ROKS) on IBM Cloud](https://cloud.ibm.com/docs/openshift?topic=openshift-clusters) to install portworx (atleast 3 workers). Refer 
    > [About Portworx](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#about-portworx) for the preferred worker node flavor and type. 

    > NOTE: ROKS cluster that has both public and private endpoints is used in this setup. Some steps might vary if private-only cluster is used. Refer [Managed OpenShift on IBM Cloud VPC cluster service architecture](https://cloud.ibm.com/docs/openshift?topic=openshift-service-arch#service-architecture_vpc) for more details on the architecture of VPC Clusters

    As of now, on VPC as baremetal servers are not supported, SDS worker node options are not available. For Non-SDS worker nodes on Virtual machines, `b2.16x64` or better is preferred for portworx. 
- [ibmcloud cli](https://cloud.ibm.com/docs/cli?topic=cli-install-ibmcloud-cli#shell_install)


## Create the volumes
- Refer to Portworx min reqs - https://docs.portworx.com/start-here-installation/  
- Provision block storage volumes for VPC. If the cluster is multi-zone and if portworx storage layer should be spanned across all zones, create block volumes in all zones. 
    > Refer [Create a stand-alone block storage volume](https://cloud.ibm.com/docs/vpc?topic=vpc-creating-block-storage#create-standalone-vol)
- Choose a desired volume profile. Choose IOPS tier and enter size. Eg: 128 GB, 10 IOPS/GB. A custom IOPS profile can also be used. 
    > Refer [Custom IOPS profile for VPC Block Volumes](https://cloud.ibm.com/docs/vpc?topic=vpc-block-storage-profiles#custom)
    
> NOTE: A volume can only be attached to one worker node. Create mutiple volumes for all/some worker nodes.


## Attach the volumes to workers
> Refer: [VPC: Adding raw Block Storage to VPC worker nodes](https://cloud.ibm.com/docs/openshift?topic=openshift-utilities#vpc_api_attach)

- Login into IBM Cloud account from the cli
    ```
    ▶ ibmcloud login
    ```
    Use `ibmcloud login --sso` to login with federated ID. 
    Other ways - [ibmcloud login](https://cloud.ibm.com/docs/cli?topic=cli-ibmcloud_cli#ibmcloud_login)

- Get cluster ID
    ```
    ▶ ibmcloud ks cluster get --cluster <cluster-name>
    ```
- Get worker node IDs
    ```
    ▶ ibmcloud oc worker ls --cluster <cluster-name>
    ```
- Get volume IDs
    ```
    ▶ ibmcloud is volumes | grep <volume-name>
    ```
- Get IAM token
    ```
    ▶ ibmcloud iam oauth-tokens
    ```
- Make an curl `POST` request to attach the created volume to a worker node
Repeat this step for all volumes. Make sure to attach the volume and worker form same zone or datacenter
 
    ```
    ▶ curl -X POST -H "Authorization: Bearer <token>" "https://us-south.containers.cloud.ibm.com/v2/storage/vpc/createAttachment?cluster=<cluster-id>&worker=<worker-id>&volumeID=<volume-id>" 
    ```
    An output like below will be observed
    ```
    {"id":"0727-b5965eb8-f7ac-4845-9ac9-5413a33aebc2","volume":{"name":"pwx-dal2-vol","id":"r006-394d474d-d5ec-47c2-8955-494c58ca880c"},"device":{"id":""},"name":"volume-attachment","status":"attaching","type":"data"}
    ```


## Setup external etcd database
> NOTE: This setup is using an external etcd db. A portworx KVDB can be used as well. But, if an automatically setup KVDB from portworx install is used, the metadata is stored inside the cluster. Refer [Setting up a key-value store for Portworx metadata](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#portworx_database)
- Provision IBM Cloud Databases for etcd from the IBM Cloud catalog
- Refer [Setting up a Databases for etcd service instance](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#databases-for-etcd) for the configuration 
- Create etcd secret that stores username, pwd, and cert info
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
    name: pwx-etcd-certs
    namespace: kube-system
    type: Opaque
    data:
     ca.pem: 
     username: 
     password:  
    ```
    ```
    ▶ oc create -f pwx-etcd-secret.yaml
    ```

## Encryption with KeyProtect
<details>
<summary> (Click to Expand) Setup portworx to use IBM Key Protect for encryption </summary>


Install can be configured to enable encryption of volumes with IBM Key Protect (KP) during provisioning or it can be enabled later.
In either case, a KeyProtect service instance and a root key should be created. 

Follow the steps 1 through 11 [Setup KeyProtect Instance for volume encryption with Portworx](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#setup_encryption) and proceed to [Portworx install](#install-portworx) to enable encryption using IBM KP during provisioning

### Enable encryption on existing portworx setup
If encryption with IBM KP has not been enabled during provisioning, it can be setup with an existing Portworx setup on OpenShift cluster by editing the `portworx` daemonset. Step 12 from [Setup KeyProtect Instance for volume encryption with Portworx](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#setup_encryption)

```
▶ oc edit daemonset portworx -n kube-system
```
Replace the `secret_type` argument value to be `ibm-kp` from the `containers` spec as below:

```
containers:
- args:
  - -A
  - -f
  - -k
  - etcd:<REDACTED>
  - -c
  - px-storage-cluster
  - -secret_type
  - ibm-kp
  - -userpwd
  - $(ETCD_USERNAME):$(ETCD_PASSWORD)
  - -ca
  - /etc/pwx/etcdcerts/ca.pem
  - -r
  - "17001"
  - -x
  - kubernetes
```
Portworx pods will be restarted automatically after the daemonset is edited

View `config.json` from one of the portworx pods to verify if the change is made. The `"secret_type": "ibm-kp"` should be added in the secret section 

```
▶ oc exec -it portworx-r4bdw  -n kube-system bash
[root@kube-bs6tcs3d0v6dus9f0n2g-<cluster>-default-00000307 /]# cd etc/pwx
[root@kube-bs6tcs3d0v6dus9f0n2g-<cluster>-default-00000307 pwx]# cat config.json 
{
    "alertingurl": "",
    "cafile": "/etc/pwx/etcdcerts/ca.pem",
    "clusterid": "px-storage-cluster",
    "dataiface": "",
    "kvdb": [
        "etcd:<REDACTED>"
    ],
    "mgtiface": "",
    "password": "<REDACTED>",
    "scheduler": "kubernetes",
    "secret": {
        "cluster_secret_key": "",
        "secret_type": "ibm-kp"
    },
    "storage": {
        "cache": [],
        "devices": [
            "/dev/vdd"
        ],
        "journal_dev": "",
        "kvdb_dev": "",
        "max_storage_nodes_per_zone": 0,
        "rt_opts": {},
        "system_metadata_dev": ""
    },
    "username": "<REDACTED>",
    "version": "1.0"
}
```
</details>

After portworx is setup for encryption using KP, check the section [Enable per volume encryption](#enable-per-volume-encryption)

## Install portworx
### Provisioning
> Refer [Installing Portworx in your cluster](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#install_portworx)
- Provision portworx service from IBM Cloud Catalog
- Choose the desired plan (DR Plan provides some additional capabilities and also allows backup to IBM COS - See details [here](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#px-dr)
- Fill in all the details. For guidance, [Install Portworx](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#install_portworx)
  For quick dev setup, you can proceed without encryption and leave the volume encryption key to default value 
    > For volume encryption, refer - [Setting up volume encryption with IBM Key Protect](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#encrypt_volumes).

    To choose **Portworx secret store type**, see below info from ibm cloud docs
    
    >   Kubernetes Secret: Choose this option if you want to store your own custom key to encrypt your volumes in a Kubernetes Secret in your cluster. The secret must not be present before you install Portworx. You can create the secret after you install Portworx. For more information, see the Portworx documentation.
    
    > IBM Key Protect: Choose this option if you want to use root keys in IBM Key Protect to encrypt your volumes. Make sure that you follow the instructions to create your IBM Key Protect service instance, and to store the credentials for how to access your service instance in a Kubernetes secret in the portworx project before you install Portworx.
    
- Enter [iam user apikey](https://cloud.ibm.com/docs/account?topic=account-userapikey#create_user_key); select the resource group, list of clusters that the user has access to are shown. Choose the cluster to install portworx on



### Verify Portworx Install
- Connect to the cluster 
    ```
    ▶ ibmcloud oc cluster config -c <cluster-name> --admin
    ```
> Refer [Connecting to the cluster from the CLI](https://cloud.ibm.com/docs/openshift?topic=openshift-access_cluster#access_oc_cli)

- Verify install
    ```
    ▶ helm ls -A
    NAME    	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART          	APP VERSION
    portworx	default  	1       	2020-07-17 14:38:16.543872499 +0000 UTC	deployed	portworx-1.0.18	2.5.1
    ```

    ```
    ▶ oc get pods -n kube-system | grep 'portworx\|stork'

    portworx-api-d94zv                         1/1       Running     0          56m
    portworx-api-nbg88                         1/1       Running     0          56m
    portworx-api-s7g2t                         1/1       Running     0          56m
    portworx-pvc-controller-64d8578f8b-7fkv5   1/1       Running     0          56m
    portworx-pvc-controller-64d8578f8b-jxcr6   1/1       Running     0          56m
    portworx-pvc-controller-64d8578f8b-q8cbb   1/1       Running     0          56m
    portworx-r6j54                             1/1       Running     0          56m
    portworx-tj6vt                             1/1       Running     0          56m
    portworx-vdvjd                             1/1       Running     0          56m
    stork-6dd4b6cd97-27jdr                     1/1       Running     1          56m
    stork-6dd4b6cd97-56gsp                     1/1       Running     0          56m
    stork-6dd4b6cd97-nzdlv                     1/1       Running     0          56m
    stork-scheduler-7b444f46dc-htwkw           1/1       Running     0          56m
    stork-scheduler-7b444f46dc-sqhfs           1/1       Running     0          56m
    stork-scheduler-7b444f46dc-x6c5m           1/1       Running     0          56m    
    ```
    > NOTE: It may take some time for the pods to reach to a `Running` state if the other portworx related pods are not Ready yet. If there are errors in other pods, you might have to check logs or use the below command to check status


- Check portworx storage cluster status.
  Exec into the portworx pods using the command below
  ```
  ▶ kubectl exec portworx-r6j54 -it -n kube-system -- /opt/pwx/bin/pxctl status
	
	
	Status: PX is operational
	License: PX-Enterprise IBM Cloud DR (expires in 1356 days)
	Node ID: <id>
		IP: <ip1>
		Local Storage Pool: 1 pool
		POOL	IO_PRIORITY	RAID_LEVEL	USABLE	USED	STATUS	ZONE		REGION
		0	LOW		raid0		128 GiB	8.4 GiB	Online	us-south-1	us-south
		Local Storage Devices: 1 device
		Device	Path		Media Type		Size		Last-Scan
		0:1	/dev/vdd	STORAGE_MEDIUM_MAGNETIC	128 GiB		17 Jul 20 14:40 UTC
		total			-			128 GiB
		Cache Devices:
		No cache devices
	Cluster Summary
		Cluster ID: px-storage-cluster
		Cluster UUID: <pwx-cluster-id>
		Scheduler: kubernetes
		Nodes: 3 node(s) with storage (3 online)
		IP	ID	SchedulerNodeName StorageNode Used Capacity  Status StorageStatus	Version		Kernel		OS
		<ip1>	<node>	<ip1>		Yes		8.4 GiB	128 GiB		Online	Up		2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
		<ip2>	<node>	<ip2>		Yes		8.4 GiB	128 GiB		Online	Up		2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
		<ip3>	<node>	<ip3>		Yes		8.4 GiB	128 GiB		Online	Up (This node)	2.5.2.0-176ddb7	3.10.0-1127.13.1.el7.x86_64	Red Hat
		Warnings: 
			 WARNING: Persistent journald logging is not enabled on this node.
	Global Storage Pool
		Total Used    	:  25 GiB
		Total Capacity	:  384 GiB
  ```
	
- Check portworx I/O classification that was assigned to the disks that are part of the Portworx cluster
    ```
    ▶ oc exec -it portworx-r6j54 -n kube-system -- /opt/pwx/bin/pxctl cluster provision-status
    NODE	NODE STATUS	POOL POOL STATUS	IO_PRIORITY	SIZE	AVAILABLE	USED	PROVISIONED	ZONE		REGIORACK
    <node>	Up		0  ( pool )	Online		LOW		128 GiB	 118 GiB		10 GiB	0 GiB		us-south-1	us-south	default
    <node>	Up		0  ( pool )	Online		LOW		128 GiB	 118 GiB		10 GiB	0 GiB		us-south-3	us-south	default
    <node>	Up		0  ( pool )	Online		LOW		128 GiB	 118 GiB		10 GiB	0 GiB		us-south-2	us-south	default
    ```
- List storageclasses setup by default from portworx install
    ```
    ▶ oc get storageclasses | grep portworx
    portworx-db-sc                          kubernetes.io/portworx-volume   30m
    portworx-db2-sc                         kubernetes.io/portworx-volume   30m
    portworx-null-sc                        kubernetes.io/portworx-volume   30m
    portworx-shared-sc                      kubernetes.io/portworx-volume   30m
    ```
> Once portworx is installed and the storage cluster is formed successfully, available storage classes can be used or custom SCs can be created. Peristence can be obtained by use of PVCs similar to how its done on k8s or openshift in general

## Validate using a sample pod

> Refer [Creating Portworx Volumes](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#add_portworx_storage)

Use dynamic provisioning to automatically create Portworx volume by creating a PVC and, optionally a StorageClass.
```
▶ mkdir -p ./portworx
```
> NOTE: One of the available storage classes can also be used in the PVC without the need of creating a custom storage class
```yaml
▶ cat <<EOF > ./portworx/custom-sc.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: pwx-custom-random-sc
provisioner: kubernetes.io/portworx-volume
parameters:
  repl: "1"
  io_profile: "random"
EOF
````

```yaml
▶ cat <<EOF > ./portworx/pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pwx-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: pwx-custom-random-sc
EOF
````


```yaml
▶ cat <<EOF > ./portworx/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
  labels:
    name: test
spec:
  containers:
  - name: test
    image: alpine:3.8
    command: ["/bin/sh"]
    args: ["-c", "mkdir -p /usr/share/test; while true; do date >> /usr/share/test/date.log; sleep 10; done"]
    volumeMounts:
      - name: datelog
        mountPath: /usr/share/test
  volumes:
    - name: datelog
      persistentVolumeClaim:
        claimName: pwx-test-pvc

EOF
```
The test pod above, writes current time and date to `/usr/share/test/date.log` and the directory is mounted as a persistent volume

```sh
▶ oc create -f ./portworx/custom-sc.yaml
▶ oc create -f ./portworx/pvc.yaml
▶ oc create -f ./portworx/pod.yaml
```

Shell into the pod and check the contents being written to the file
```
▶ oc exec -it test sh  

/ # tail -f /usr/share/test/date.log
Mon Jul 20 05:01:19 UTC 2020
Mon Jul 20 05:01:29 UTC 2020
Mon Jul 20 05:01:39 UTC 2020
Mon Jul 20 05:01:49 UTC 2020
Mon Jul 20 05:01:59 UTC 2020
Mon Jul 20 05:02:09 UTC 2020
```

To check persistence of the log file, delete the pod and create a new pod that uses the same pvc 

```
▶ oc delete -f ./portworx/pod.yaml
```

```yaml
▶ cat <<EOF > ./portworx/test-persistence-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-persistence
  labels:
    name: test
spec:
  containers:
  - name: test-persistence
    image: alpine:3.8
    command: ["/bin/sh"]
    args: ["-c", "cat /data/date.log"]
    volumeMounts:
      - name: datelog
        mountPath: /data
  volumes:
    - name: datelog
      persistentVolumeClaim:
        claimName: pwx-test-pvc
EOF
```
This pod uses the same pvc and prints the contents(generated from previous pod) of the log file saved in the volume. 

```
▶ oc create -f ./portworx/test-persistence-pod.yaml
```
```
▶ oc logs test-persistence
```
The pod logs will display the contents of file with all the log entries saved until previous pod has been deleted. The `date.log` file is persisted in the volume and hence the contents can be retrieved from this new pod by pointing it to the pvc the earlier pod is backed by.

Delete the resources created for validation

```
▶ oc delete -f ./portworx/test-persistence-pod.yaml
▶ oc delete -f ./portworx/custom-sc.yaml
▶ oc delete -f ./portworx/pvc.yaml
```

## Enable per volume encryption


There are two ways to enable per volume encryption  
<details>
<summary>With Storage Class </summary>

- Create secure storage class with `secure` parameter set to `true`

  ```yaml
  ▶ cat <<EOF > ./portworx/custom-secure-sc.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: pwx-custom-random-secure-sc
  provisioner: kubernetes.io/portworx-volume
  parameters:
    repl: "1"
    io_profile: "random"
    secure: "true"
  EOF
  ```

  ```
  ▶ oc create -f ./portworx/custom-secure-sc.yaml 
  ```

- Create a PVC to use the secure storage class. The volumes created by PVCs that use the secure storage class will be encrypted

  ```yaml
  ▶ cat <<EOF > ./portworx/pvc-secured-through-sc.yaml
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pwx-test-pvc-secured-through-sc
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
    storageClassName: pwx-custom-random-secure-sc
  EOF
  ````

  ```
  ▶ oc create -f ./portworx/pvc-secured-through-sc.yaml 
  ```
</details>

<details>
<summary>With PVC </summary>

### Using a PVC and specifying the annonation to create a secure volume   
    
If enabling encryption with the storageclass is not preferred, you can also enable it during PVC creation using
```
annotations:
 px/secure: "true"
```
- Create PVC with secure annotation
  ```yaml
  ▶ cat <<EOF > ./portworx/pvc-secure.yaml
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pwx-test-pvc-secure
    annotations:
     px/secure: "true"
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
    storageClassName: pwx-custom-random-sc
  EOF
  ```
  > In the above, a storageclass that has no secure flag set is used

  ```
  ▶ oc create -f ./portworx/pvc-secure.yaml 
  ```
</details>

### Verify
<details>
<summary>List the volumes </summary>

### List the volumes to check for encryption

- List the PVCs. Under the `Volume` column, it shows the Volume names the PVCs are bound to
  ```
  ▶ oc get pvc
  
  NAME                              STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
  pwx-test-pvc-secure               Bound     pvc-1da0c264-bf2d-4488-8257-9d0e0c1819f9   1Gi        RWO            pwx-custom-random-sc          4m1s
  pwx-test-pvc-secured-through-sc   Bound     pvc-3a28d2b5-a904-42b6-981b-0023fcdc3941   1Gi        RWO            pwx-custom-random-secure-sc   16m
  ```
- From one of the protworx pods, verify Portworx volume list

    ```
    ▶ oc exec -it portworx-r4bdw  -n kube-system -- /opt/pwx/bin/pxctl volume list
    ID			        NAME						                SIZE	HA	SHARED	ENCRYPTED	IO_PRIORITY	STATUS		    SNAP-ENABLED	
    526462371008291079	pvc-1da0c264-bf2d-4488-8257-9d0e0c1819f9	1 GiB	1	no	    yes		    LOW		    up - detached	no
    973253658930844681	pvc-3a28d2b5-a904-42b6-981b-0023fcdc3941	1 GiB	1	no	    yes		    LOW		    up - detached	no

    ```
    > Under the `Encrypted` column, you should see `yes` for PVs created through secure SCs or PVCs
</details>

<details>
<summary>Delete </summary>
Delete the resources created to try out creation of encrypted volumes

```
▶ oc delete -f ./portworx/custom-secure-sc.yaml
▶ oc delete -f ./portworx/pvc-secured-through-sc.yaml
▶ oc delete -f ./portworx/pvc-secure.yaml
```
</details>


## Cleanup

- Portworx cleanup
    ```
    oc project kube-system
    ```
    ```
    curl -fsL https://install.portworx.com/px-wipe | bash 
    ```
    ``` 
    helm delete portworx
    ```
    > Refer: [Removing Portworx from your cluster](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#remove_portworx)

- To stop billing, remove the Portworx service instance created, from IBM Cloud account.

- If not required, delete the IBM Cloud Databases for etcd service instance from IBM Cloud account

- Detach volumes
    - [Detach block volumes from worker nodes using console](https://cloud.ibm.com/docs/vpc?topic=vpc-managing-block-storage#detach)
    - [Detach block volumes from worker nodes using CLI](https://cloud.ibm.com/docs/vpc?topic=vpc-managing-block-storage-cli#detach-vol-attachment-cli)

- Delete volumes created
    - [Delete the block volumes using console](https://cloud.ibm.com/docs/vpc?topic=vpc-managing-block-storage#delete)
    - [Delete the block volumes using CLI](https://cloud.ibm.com/docs/vpc?topic=vpc-managing-block-storage-cli#delete-vol-cli)
