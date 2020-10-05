# Portworx snapshots on IBM Cloud
This doc will cover the steps to create portworx(pwx) cloud snapshots and restore data to volumes from the saved snapshots
The steps also show how Portworx can be configured to store snapshots in IBM Cloud Object Storage (COS) by creating cloud credentials on pwx.

> NOTE: The below steps are only created only for reference. Production usecases might involve additional steps and configuration.

Refer portworx docs on 
- [Types of snaphshots](https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-snapshots/)
- [Taking snapshot of a volume from PVC](https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/kubernetes-storage-101/snapshots/#snapshots)
- [On-demand cloud snapshots](https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-snapshots/on-demand/snaps-cloud/) (This is the scenario tried out in this document)

## Prerequisites

- Basic knowledge on COS, IAM, openshift, portworx
- Portworx installed on openshift cluster (ROKS)
- IBM COS instance 

## Create a bucket, service ID, COS service credentials

- (Optional) Create a COS bucket (`pwx-test-snapshots`) to store snapshots. This step is optional. If creating a bucket is not preferred beforehand, pwx can create one 

- Create a service ID to manage access to this specifc COS instance and/or COS bucket. If you don't want to use a service id, you can jump to the next step. Can be created through CLI or Cloud console

  > Refer [creating service id](https://cloud.ibm.com/docs/account?topic=account-serviceids#create_serviceid)

- Create an apikey for the service ID above
  > Refer [creating apikey for service id](https://cloud.ibm.com/docs/account?topic=account-serviceidapikeys#create_service_key)

- Create and access policy for the service id to allow write access on the COS bucket
  ```
  ▶ ibmcloud iam service-policy-create pwx-snapshots-service-id --roles Writer --service-name cloud-object-storage --service-instance  <cos-crn>  --resource-type bucket --resource pwx-test-snapshots
  ```
  > Refer [assign access to resources](https://cloud.ibm.com/docs/account?topic=account-assign-access-resources#resourceaccess)

- Create HMAC credentials for COS. Make sure to associate the credentials to the service ID created earlier to restrict access of these creds to the specific bucket created for snapshots if desired.
  > Refer [create hmac credentials](https://cloud.ibm.com/docs/cloud-object-storage?topic=cloud-object-storage-uhc-hmac-credentials-main#uhc-hmac-credentials-defined)

  `access_key_id` and `secret_access_key` from the creds created above will be used to create cloud credentials on pwx 
  ```
  "cos_hmac_keys": {
      "access_key_id": "",
      "secret_access_key": ""
    }
  ```

## Create cloud credentials
The created credentials allow pwx to store snapshots in cloud object storage. Since IBM COS is S3 compatible, the same steps can be used. 
> Refer [portworx cloud credentials](https://docs.portworx.com/reference/cli/credentials/) for details
Login to one of the portworx pods to create cloud credentials
```
▶ oc get pods -n kube-system | grep 'portworx'


[root@kube-bs6tcs3d0v6dus9f0n2g-default-00000307 ~]# /opt/pwx/bin/pxctl credentials create \
>   --provider s3 \
>   --s3-access-key <access_key_id> \
>   --s3-secret-key <secret_access_key> \
>   --s3-endpoint <bucket-endpoint> \
>   --s3-region us-south \  --bucket pwx-test-snapshots \
>   pwx-test-snapshots
Credentials created successfully, UUID:3edaf2e4-4932-4c43-89b7-c63ce3eac7ac

[root@kube-bs6tcs3d0v6dus9f0n2g-default-00000307 /]# /opt/pwx/bin/pxctl credentials list

S3 Credentials
UUID						NAME		REGION			ENDPOINT							ACCESS KEY			SSL ENABLED		ENCRYPTION		BUCKET				WRITE THROUGHPUT (KBPS)
3edaf2e4-4932-4c43-89b7-c63ce3eac7ac				us-south		s3.us-south.cloud-object-storage.appdomain.cloud		<access_key_id> true			false			pwx-test-snapshots		4397
[root@kube-bs6tcs3d0v6dus9f0n2g-default-00000307 /]# 

```
Note that, we are associating a COS bucket during credential creation because a bucket is already setup for this. If you want pwx to create a new bucket, you can create creds without associating a bucket

## Portworx Cloud Snapshots

### Create test statefulset
- Create a sample statefulset to test pwx snapshots

  ```
  ▶ oc create -f test-statefulset.yaml
  ```
  The statefulset does dynamic provisioning of volumes using the storageclass specified (A pwx storageclass is used)
  ```
  ▶ oc get pvc
  NAME                        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
  datelog-test-stateful-0     Bound     pvc-d353bfb4-6798-4238-9ae2-e6b697e7966c   1Gi        RWO            pwx-custom-random-secure-sc   9d
  datelog-test-stateful-1     Bound     pvc-7fab7f97-775a-4b88-8601-95d90f6428bb   1Gi        RWO            pwx-custom-random-secure-sc   9d
  ```
  ```
  ▶ oc get pods -w  
  NAME              READY     STATUS    RESTARTS   AGE
  test-stateful-0   1/1       Running   0          15s
  test-stateful-1   1/1       Running   0          10s
  ```
  The container used in the statefulset just prints date to the log file every 10 secs


- Check the logs from the file saved
  ```
  ▶ oc exec -it test-stateful-0 -- /bin/sh
  ~ $ tail -f /usr/share/test/date.log
  Wed Jul 29 22:54:18 UTC 2020
  Wed Jul 29 22:54:28 UTC 2020
  Wed Jul 29 22:54:38 UTC 2020
  Wed Jul 29 22:54:48 UTC 2020
  Wed Jul 29 22:54:58 UTC 2020
  Wed Jul 29 22:55:08 UTC 2020
  Wed Jul 29 22:55:18 UTC 2020
  Wed Jul 29 22:55:28 UTC 2020 
  Wed Jul 29 22:55:38 UTC 2020
  Wed Jul 29 22:55:48 UTC 2020
  ```
   Note the timestamp from the log file 

### Create snapshots

- Take a snapshot of the current state of volume attached to `datelog-test-stateful-0` pvc
  ```
  ▶ oc create -f snapshot.yaml 

  ▶ kubectl get volumesnapshot                                                                
  NAME                             AGE
  log-test-stateful-0-snapshot-1   13s
  ```
- Describe the volumesnapshot data 

  ```
  ▶ kubectl get volumesnapshotdatas       
  NAME                                                       AGE
  k8s-volume-snapshot-1b73f0d5-52c2-438a-8c92-a66e7b745855   11s

  ▶ kubectl describe volumesnapshotdatas k8s-volume-snapshot-1b73f0d5-52c2-438a-8c92-a66e7b745855 
  Name:         k8s-volume-snapshot-1b73f0d5-52c2-438a-8c92-a66e7b745855
  Namespace:    
  Labels:       <none>
  Annotations:  <none>
  API Version:  volumesnapshot.external-storage.k8s.io/v1
  Kind:         VolumeSnapshotData
  Metadata:
    Creation Timestamp:  2020-07-29T22:57:13Z
    Generation:          1
    Resource Version:    7380896
    Self Link:           /apis/volumesnapshot.external-storage.k8s.io/v1/volumesnapshotdatas/k8s-volume-snapshot-1b73f0d5-52c2-438a-8c92-a66e7b745855
    UID:                 0f5c0857-5cfc-4e9d-acc5-2bfdd6d9d630
  Spec:
    Persistent Volume Ref:
      Kind:  PersistentVolume
      Name:  pvc-d353bfb4-6798-4238-9ae2-e6b697e7966c
    Portworx Volume:
      Snapshot Id:         pwx-test-snapshots/255063705045082257-169448402957702252
      Snapshot Task ID:    927fea0d-df4e-4a7a-9391-ec57070ff569
      Snapshot Type:       cloud
      Volume Provisioner:  pxd
    Volume Snapshot Ref:
      Kind:  VolumeSnapshot
      Name:  pb-pwx-test/log-test-stateful-0-snapshot-1-927fea0d-df4e-4a7a-9391-ec57070ff569
  Status:
    Conditions:
      Last Transition Time:  2020-07-29T22:57:13Z
      Message:               Snapshot created successfully and it is ready
      Reason:                
      Status:                True
      Type:                  Ready
    Creation Timestamp:      <nil>
  Events:                    <none>
  ```

  ```
  Snapshot Id:         pwx-test-snapshots/255063705045082257-169448402957702252
  ```
  The `Snapshot Id` shows the location of the snapshot in COS

-  Corrput the data to verify if we can restore the actual data back from a snapshot 
```
▶ oc exec -it test-stateful-0 -- /bin/sh
~ $ while true; do echo "corrupt data" >> /usr/share/test/date.log; sleep 5; done
^C
~ $ tail -f /usr/share/test/date.log
Wed Jul 29 23:08:18 UTC 2020
corrupt data
corrupt data
Wed Jul 29 23:08:28 UTC 2020
corrupt data
corrupt data
Wed Jul 29 23:08:38 UTC 2020
corrupt data
corrupt data
Wed Jul 29 23:08:48 UTC 2020
Wed Jul 29 23:08:58 UTC 2020
```

### Restore a snapshot

Snapshot can be restore to an new PVC or an existing PVC

#### Restore to existing pvc 
> NOTE: App that uses the pvc should have been scheduled using stork to be able to restore from cloud snapshot using stork
In the test stateful set created above, the pod spec has `stork` scheduler 
  ```
      spec:
        schedulerName: stork
  ```
  ```
  ▶ oc create -f restore-snap-to-existing-pvc.yaml 
  volumesnapshotrestore.stork.libopenstorage.org/datelog-test-stateful-0 created
  ```
  ```
  ▶ kubectl get  volumesnapshotrestore                            
  NAME                      AGE
  datelog-test-stateful-0   7s

  ▶ kubectl describe volumesnapshotrestore datelog-test-stateful-0
  Name:         datelog-test-stateful-0
  Namespace:    pb-pwx-test
  Labels:       <none>
  Annotations:  <none>
  API Version:  stork.libopenstorage.org/v1alpha1
  Kind:         VolumeSnapshotRestore
  Metadata:
    Creation Timestamp:  2020-07-29T23:14:22Z
    Finalizers:
      stork.libopenstorage.org/finalizer-cleanup
    Generation:        6
    Resource Version:  7386673
    Self Link:         /apis/stork.libopenstorage.org/v1alpha1/namespaces/pb-pwx-test/volumesnapshotrestores/datelog-test-stateful-0
    UID:               1237c3ce-8037-4723-9566-cb38359f4f29
  Spec:
    Group Snapshot:    false
    Source Name:       log-test-stateful-0-snapshot-1
    Source Namespace:  pb-pwx-test
  Status:
    Status:  Successful
    Volumes:
      Namespace:  pb-pwx-test
      Pvc:        datelog-test-stateful-0
      Reason:     Restore is successful
      Snapshot:   k8s-volume-snapshot-1b73f0d5-52c2-438a-8c92-a66e7b745855
      Status:     Successful
      Volume:     pvc-d353bfb4-6798-4238-9ae2-e6b697e7966c
  Events:
    Type    Reason      Age   From   Message
    ----    ------      ----  ----   -------
    Normal  Successful  11s   stork  Snapshot in-Place  Restore completed

  ```

  After the restore is initiated to an existing pvc, the pods that use the pvc are automatically restarted
  ```
  ▶ oc get pods 
  NAME              READY     STATUS    RESTARTS   AGE
  test-stateful-0   1/1       Running   0          89s
  test-stateful-1   1/1       Running   0          26m

  ```

- Verify if after restore, the log file is free of the corrupt data and has the data from the snapshot

```
▶ oc exec -it test-stateful-0 -- /bin/sh

~ $ tail -50 /usr/share/test/date.log
Wed Jul 29 22:54:18 UTC 2020
Wed Jul 29 22:54:28 UTC 2020
Wed Jul 29 22:54:38 UTC 2020
Wed Jul 29 22:54:48 UTC 2020
Wed Jul 29 22:54:58 UTC 2020
Wed Jul 29 22:55:08 UTC 2020
Wed Jul 29 22:55:18 UTC 2020
Wed Jul 29 22:55:28 UTC 2020
Wed Jul 29 22:55:38 UTC 2020
Wed Jul 29 22:55:48 UTC 2020
Wed Jul 29 22:55:58 UTC 2020
Wed Jul 29 22:56:08 UTC 2020
Wed Jul 29 22:56:18 UTC 2020
Wed Jul 29 22:56:28 UTC 2020
Wed Jul 29 22:56:38 UTC 2020
Wed Jul 29 22:56:48 UTC 2020
Wed Jul 29 22:56:58 UTC 2020
Wed Jul 29 23:15:06 UTC 2020
Wed Jul 29 23:15:16 UTC 2020
Wed Jul 29 23:15:26 UTC 2020
Wed Jul 29 23:15:36 UTC 2020
Wed Jul 29 23:15:46 UTC 2020
Wed Jul 29 23:15:56 UTC 2020
Wed Jul 29 23:16:06 UTC 2020
Wed Jul 29 23:16:16 UTC 2020
Wed Jul 29 23:16:26 UTC 2020
Wed Jul 29 23:16:36 UTC 2020
Wed Jul 29 23:16:46 UTC 2020
Wed Jul 29 23:16:56 UTC 2020
Wed Jul 29 23:17:06 UTC 2020
Wed Jul 29 23:17:16 UTC 2020
Wed Jul 29 23:17:26 UTC 2020
Wed Jul 29 23:17:36 UTC 2020
Wed Jul 29 23:17:46 UTC 2020
Wed Jul 29 23:17:56 UTC 2020
Wed Jul 29 23:18:06 UTC 2020
Wed Jul 29 23:18:16 UTC 2020
Wed Jul 29 23:18:26 UTC 2020
Wed Jul 29 23:18:36 UTC 2020
Wed Jul 29 23:18:46 UTC 2020
Wed Jul 29 23:18:56 UTC 2020
Wed Jul 29 23:19:06 UTC 2020
Wed Jul 29 23:19:16 UTC 2020
Wed Jul 29 23:19:26 UTC 2020
```

No corrupt data that we simulated above is found in the file after restore

```
~ $ cat /usr/share/test/date.log | grep corrupt
~ $ 
```

#### Restore using a new PVC
```
▶ oc create -f restore-snap-to-new-pvc.yaml 
persistentvolumeclaim/restore-from-snapshot-pvc created
```
```
▶ oc get pvc
NAME                        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
datelog-test-stateful-0     Bound     pvc-d353bfb4-6798-4238-9ae2-e6b697e7966c   1Gi        RWO            pwx-custom-random-secure-sc   28h
datelog-test-stateful-1     Bound     pvc-7fab7f97-775a-4b88-8601-95d90f6428bb   1Gi        RWO            pwx-custom-random-secure-sc   28h
restore-from-snapshot-pvc   Bound     pvc-1c5678dd-5a8c-4e43-bffb-ddb77b12c5f6   1Gi        RWO            stork-snapshot-sc             33s
```

Refer - https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/stork/

- Create a pod that uses the new PVC

```
▶ oc create -f pod.yaml
pod/test created
```
```
▶ oc get pods   
NAME              READY     STATUS    RESTARTS   AGE
test              1/1       Running   0          17s
test-stateful-0   1/1       Running   0          16m
test-stateful-1   1/1       Running   0          41m

```

```
▶ oc exec -it test -- /bin/sh
/ # tail -50 /usr/share/test/date.log
Wed Jul 29 22:54:18 UTC 2020
Wed Jul 29 22:54:28 UTC 2020
Wed Jul 29 22:54:38 UTC 2020
Wed Jul 29 22:54:48 UTC 2020
Wed Jul 29 22:54:58 UTC 2020
Wed Jul 29 22:55:08 UTC 2020
Wed Jul 29 22:55:18 UTC 2020
Wed Jul 29 22:55:28 UTC 2020
Wed Jul 29 22:55:38 UTC 2020
Wed Jul 29 22:55:48 UTC 2020
Wed Jul 29 22:55:58 UTC 2020
Wed Jul 29 22:56:08 UTC 2020
Wed Jul 29 22:56:18 UTC 2020
Wed Jul 29 22:56:28 UTC 2020
Wed Jul 29 22:56:38 UTC 2020
Wed Jul 29 22:56:48 UTC 2020
Wed Jul 29 22:56:58 UTC 2020
Wed Jul 29 23:31:24 UTC 2020
Wed Jul 29 23:31:34 UTC 2020
Wed Jul 29 23:31:44 UTC 2020
Wed Jul 29 23:31:54 UTC 2020
Wed Jul 29 23:32:04 UTC 2020
```
Notice the log has data retrieved from snapshot an dthe new data being written to the volume by the pod

```
Wed Jul 29 22:56:58 UTC 2020
Wed Jul 29 23:31:24 UTC 2020
```

If you look at the logs from when we took the snapshot the timeframe matches 
