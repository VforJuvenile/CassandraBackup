# This is work-in-progress

# Cassandra Kubernetes Backup and Restoration Steps

This guide is using Cassandra-aws-backup.sh. It's already included with customized [Cassandra docker image][cassandra_kube]. The script is based on [Google Cloud Storage for Cassandra Disaster recovery][gcs_recovery] script and modified to make it work to AWS.

Cassandra custom docker image contains the following addition to support the backup and restore. 
- AWS CLI (apt-get install awscli)
- rsync (apt-get install rysnc)
- Cron (apt-get install cron)
- Incremental backup set to true on cassandra.yaml

## Pre-requisite 

Environment variables needed for AWS. 
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY


## Performing manual backup

Bash remotely to one of the Cassandra pod clusters. For example, using deployment statefulset with pod name Cassandra. 

```bash
kubectl exec -it cassandra-0 bash
```

Run the following command to start the backup. This will create a snapshot and upload to S3 Bucket. 

```bash
./cassandra-aws-backup.sh -b s3://{s3_bucket} -vcC -u <cassandra_username> -p <cassandra_password>
```


 
 ## Performing restore
 
For Cassandra containerized, the restore is a little bit tricky. We need to create a cassandra pod on Kubernetes that doesn't start automatically so we can restore the database. The way the containerized works is there is always one main process (PID = 1) that needs to keep running in the foreground. In order to do that we need to add a process that doesn't complete, basically, will take PID = 1 instead of cassandra. 

One way is to modify the statefulset yaml and add the following entry.

```bash
    command: [ "/bin/bash", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
```

Example YAML. 

```bash
  - name: cassandra
    image: siganberg/cassandra_kube
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 7000
      name: intra-node 
    - containerPort: 7001
      name: tls-intra-node
    - containerPort: 7199
      name: jmx
    - containerPort: 9042
      name: cql        
    # Just spin & wait forever
    command: [ "/bin/bash", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
```

Kubernetes will create the Cassandra pod but won't start which exactly what we want for the restoration process.

From here we can do remote bash again from one of the cassandra pod. 

```bash
kubectl exec -it cassandra-0 bash
```

Execute the following commands to manually create the folders.

```bash
mkdir /var/lib/cassandra/commitlog
mkdir /var/lib/cassandra/data
mkdir /var/lib/cassandra/saved_caches
```

We need to get the backup path of the snapshot that we want to restore. You can use the command below to get the list from AWS Bucket. Alternatively, you can go the AWS console and browse. 

```bash
./cassandra-aws-backup.sh -b s3://{s3_bucket} inventory
```

The command above should give you all available snapshots based on pod hostname. 
 

For example, To start the restore execute this command.  
 
```bash
./cassandra-aws-backup.sh -v -u <cassandra_username> -p <cassandra_password> -b s3://{fullpath_of_compressed_tar}.tar restore
```

After the successful restore, modify the statefulset YAML. Remove the following entry and cassandra should restart normally again bounded to the restored data. 

```bash
    command: [ "/bin/bash", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
```

> Note: Your statefulset cassandra needs to have persistent volume (PV). The persistent volume usually bounded by fixed name using persistent volume claim (PVC). This makes cassandra storage resilient even if we kill the POD like the way we are doing on these steps by adding and removing the infinite sleep.  

## TODO

- Add instructions for setting up CRON backup.



## Author
Francis Marasigan


 [gcs_recovery]: https://cloud.google.com/solutions/google-cloud-storage-for-cassandra-disaster-recovery
 [cassandra_kube]: https://hub.docker.com/r/siganberg/cassandra_kube/
