# Velro Restore to Different Destination Cluster

## Use Case

This was a hybrid restore using Velero install by VMware Tanzu Misson Control (TMC) to restore workload from kubernetes cluster 1 to cluster 2 - mimicing either a total loss of cluster or workload migration.  At the time of this work, TMC did not support cross cluster restores directly from the UI.  TMC now supports this natively so the process is no longer required.  However the process remains usefull for upstream Velero cross cluster restores.

## Background

Velero creates a different file/folder structure for each cluster it backs up.  To restore a backup to a cluster different from the original cluster the backup was created on, you must mount the backup location from the source cluster to the destination cluster.  You can use the Velero CLI to add additional backup locations to a cluster.

## Pre-requisites

- Velero CLI 
- Velero Installed on both clusters (source and destination)
- Completed backup on source cluster
- Credentials for backup target 
- Configuration for backup target (Region, S3Uri, S3address).  This can be any S3 compatible endpoint (S3, Minio, Isilon, etc)

## Source Cluster (Cluster 1)

In order to add the source cluster (cluster 1) mount point to the detination cluster (cluster 2) you will need to gather some information about the source cluster backup location and configuration

1. Get the backup storage location on source cluster using kubectl
```
kubectl get backupstoragelocation -A

NAMESPACE   NAME           PHASE       LAST VALIDATED   AGE     DEFAULT
velero      velero-s3-bb   Available   1s               6h47m
```
2. Get backup storage locations details
```
k get backupstoragelocation velero-s3-bb -n velero -oyaml

<snip>
spec:
  config:
    bucket: velero-cluter1bucket
    profile: default
    publicUrl: https://s3.us-west-2.amazonaws.com
    region: us-west-2
    s3ForcePathStyle: "true"
    s3Url: https://s3.us-west-2.amazonaws.com
  credential:
    key: bsl
    name: velero-velero-iam-bb
  objectStorage:
    bucket: velero-cluster1bucket
    prefix: 01G39PC77MTCCRG98765/
  provider: aws
</snip>
```
3. From backupstoragelocation output we need the **provider, bucket, prefix, s3Url, region and s3ForcePathStyle** values
```
provider = aws (this will be aws even if you are using Minio, Isilon or some other s3 compatible storage)
bucket = velero-cluster1bucker
prefix = 01G39PC77MTCCRG98765/ (include /)
region = us-west-2
s3Url = https://s3.us-west-2.amazonaws.com  (for non S3 endpoints include IP/FQDN and Port http://192.168.10.23:9020 for example)
s3ForcePathStyle = true
```

## Destination Cluster (Cluster 2)

### Create the secret with credentials to your S3 compatible storage endpoint

Regardless of the type of S3 compatible storage you use, the credentials are supplied to Velero in the form of the aws_access_key_id and aws_secret_access_key.  You could decode this from the source cluster by finding the secret in the velero namespace that your backupstoragelocation is using.  The output from step 2 above shows the name of this secret (velero-iam-bb)

You can decode this secret data payload to get the values for aws_access_key_id and aws_secret_access_key.  I won't cover this here but the commands to properly decode the secret is in the troubleshooting section.  

You may also wonder if you need to do this step if all of your velero enabled clusters use the same credentials and therefor would have a preexisting secret on the destination cluster already.  The answer is YES, at least as far as I can tell the velero backup-location create command expects the secret in a single key:value pair.  Because of this I cannot find a way to add both the access_key and secret_access_key with there values.  So we need to create a secret that is configured in a cetain way.

1. Create a file to use for secret creation.
```
cat << EOF > cluster1-creds 
[default]
aws_access_key_id=myaccesskey
aws_secret_access_key=myaccesssecret
EOF
```
2. Create the new secret on Destination cluster (cluster 2) 
```
kubectl config use-context cluster2
kubectl create secret generic cluster1-creds-secret -n velero --from-file=/path/to/cluster1-creds
secret/cluster1-creds-secret created
```
3. Verify secret is created
```
kubectl get secrets -n velero

NAME                        TYPE                                  DATA   AGE
cluster1-creds-secret       Opaque                                1      7s
default-token-xjg9l         kubernetes.io/service-account-token   3      7h20m
snapshot-credentials        Opaque                                1      7h20m
velero-iam-bb               Opaque                                2      7h19m
velero-restic-credentials   Opaque                                1      7h20m
velero-token-4k4hq          kubernetes.io/service-account-token   3      7h20m
velero-velero-iam-bb        Opaque                                1      7h19m
```
4. Describe Secret
```
kubectl describe secret cluster1-creds-secret -n value -oyaml

apiVersion: v1
data:
  cluster1-creds: W2RlZm...............
kind: Secret
metadata:
  creationTimestamp: "2022-08-04T03:33:41Z"
  name: cluster1-creds-secret
  namespace: velero
  resourceVersion: "196027"
  uid: d9c2d213-56da-415d-83ba-a3591663ebac
type: Opaque
```
Notice the value cluster1-creds under the secret data.  We will need in the next step so take note.
5. For troubleshooting you can validate the value of the secret cluster1-creds data by base64 decodeing the value
```
echo -n W2RlZm........ | base64 -d

[default]
aws_access_key_id=myaccesskey
aws_secret_access_key=myaccesssecret
```

### Create the source backup location on destination cluster (cluster 2)

We are now ready to assemble all of the information we gathered from the source cluster and the secret we created on the destination cluster to add the new location to the destination cluster using the Velero CLI

1. Recap of infomation we gathered from souce cluster (cluster 1)
```
provider = aws (this will be aws even if you are using Minio, Isilon or some other s3 compatible storage)
bucket = velero-cluster1bucket
prefix = 01G39PC77MTCCRG98765/ (include /)
region = us-west-2
s3Url = https://s3.us-west-2.amazonaws.com  (for non S3 endpoints include IP/FQDN and Port 192.168.10.23:9002 for example)
s3ForcePathStyle = true
```
2. Recap of secret created on destination cluster (cluster 2)
```
secret name = cluster1-creds-secret
data value = cluster1-cred
```
3. Add source (cluster) 1 backup location to destination (cluster 2)
```
velero backup-location create cluster1-target --provider aws --bucket velero-cluster1bucker --prefix 01G39PC77MTCCRG98765/ --credential cluster1-creds-secret=cluster1-cred --config region=us-west-2,s3ForcePathStyle="true",s3Url=https://s3.us-west-2.amazonaws.com
```
4. Verify new backup location was created and it available.  You should see 2 targets (velero-s3-bb is cluster 2 default location and cluster1-target is the new location that represents the backup location for cluster 1)
```
velero backup-location get

NAME                  PROVIDER   BUCKET/PREFIX                               PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
cluster1-target         aws        velero-cluster1bucket/01G39PC77MTCCRG98765/   Available   2022-05-25 18:16:18 -0500 CDT   ReadWrite
velero-s3-bb            aws         velero-cluster2bucket/01G39PC77MTCCRABCDE/   Available   2022-05-25 18:16:15 -0500 CDT   ReadWrite
```
Note the cluster has to be listed as Available or there is something wrong with your configuation.  Check troubleshooting section for tips

### Restore cluster 1 backup to cluster 2

1. Set backup location to cluster 1 location
```
velero backup-location set tsm1-cluster-target
```
2. List backups available in cluster 1 target location
```
velero backup get

NAME                STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION      SELECTOR
cluster1-full      Completed    0        0          2022-05-25 18:01:16 -0500 CDT   29d       cluster1-targer          <none>
```
3. Restore cluster1-full job to cluster 2
```
velero restore create cluster1-restore --from-backup cluster1-full
```
4. Monitor Status of backup jobs
```
velero restore get
velero restore describe cluster1-restore
velero restore logs hello-restore
```

## Troubleshooting Command

1. If backup location is not listed as available check pod logs on velero pod in velero namespace and look for any clues to errors
```
kubectl get pods -n velero
NAME                      READY   STATUS    RESTARTS   AGE
restic-bs584              1/1     Running   0          7h46m
restic-glkmj              1/1     RUnning   0          7h46m
velero-7c59dd7f5d-jbwv2   1/1     Running   0          7h46m

kubectl logs velero-7c59dd7f5d-jbwv2 -n velero
```
2. Verify backupstoragelocation on destination (cluster 2)
```
kubectl get backupstoragelocation -A
kubectl get backupstoragelocation -n velero -oyaml
```
Verify s3Url, region, secret, bucket and prefix information is correct for your cluster 1 target location
3. Verify secret on destination cluster is correct for the backup target you are adding
```
kubectl get secrets -n velero
kubectl get secret cluster1-creds-secret -n velero -oyaml

apiVersion: v1
data:
  cluster1-creds: W2RlZm......
kind: Secret
metadata:
  creationTimestamp: "2022-08-04T03:33:41Z"
  name: cluster1-creds-secret
  namespace: velero
  resourceVersion: "196027"
  uid: d9c2d213-56da-415d-83ba-a3591663ebac
type: Opaque

kubectl get secret cluster1-creds-secret -n value -o jsonpath='{.data.cluster1-creds}' | base64 -d

[default]
aws_access_key_id=myaccesskey
aws_secret_access_key=myaccesssecret
```
4. Next steps is to verify cluster 2 has access to s3Url IP and Port