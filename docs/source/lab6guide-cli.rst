**Important Note:** We will again make use of Minio object store in this
Lab. We will use it as the endpoint for our cloud snapshots.

Deploy Minio as target for Portworx Cloud Snapshots
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a storageClass for use by Minio

::

   cat <<'EOF' > /tmp/px-ha-sc.yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
       name: px-ha-sc
   provisioner: kubernetes.io/portworx-volume
   parameters:
      repl: "3"
      io_priority: "high"
      group: "minio"

.. code:: text

   oc create -f /tmp/px-ha-sc.yaml

Deploy Minio onto the OCP cluster

.. code:: text

   echo "Installing Helm"
   curl -L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash -s -- --version v3.8.2
   helm repo add stable https://charts.helm.sh/stable
   echo "Helm installed!"
   echo "Installing mc"
   ssh node01 "sudo wget -qO /usr/bin/mc https://dl.minio.io/client/mc/release/linux-amd64/mc"
   ssh node01 "sudo chmod +x /usr/bin/mc"
   echo "mc installed!"

   echo "Setting Up Minio" 
   helm install px-minio stable/minio --namespace minio --create-namespace --set accessKey=ZZYYXXWWVVUUTT --setsecretKey=0ldSup3rS3cr3t --set persistence.storageClass=px-ha-sc --set resources.requests.memory=1Gi > /dev/null 2>&1
   helm install px-minio-2 stable/minio --namespace minio --create-namespace --set accessKey=AABBCCDDEEFF --setsecretKey=N3wSup3rS3cret --set persistence.storageClass=px-ha-sc --set resources.requests.memory=1Gi > /dev/null 2>&1
   until [[ `oc -n minio get pods | grep px-minio | grep Running | grep 1/1 | wc -l` -eq 2 ]]; do echo "Waiting for px-minioand px-minio-2 to be ready...."; sleep 1 ;done
   echo "Setup Complete ..."

   MINIO_ENDPOINT=http://$(oc -n minio get svc px-minio -o jsonpath='{.spec.clusterIP}:9000')
   echo "Configure Minio's endpoints"
   ssh node01 "mc config host add px $MINIO_ENDPOINT ZZYYXXWWVVUUTT 0ldSup3rS3cr3t --api S3v4"
   echo "Configuration Complete"

Once the set-up is complete, you can run the following command to make
sure the minio servers are up:

.. code:: text

   ssh node01 mc admin info px

Create a new Portworx credential called ``my-cloud-credentials`` with
the below parameters:

::

      provider = s3
      s3 region = us-east-1
      access key = ZZYYXXWWVVUUTT
      secret key = 0ldSup3rS3cr3t

Run the below command to obtain the object store endpoint:

.. code:: text

   MINIO_ENDPOINT=http://$(oc -n minio get svc px-minio -o jsonpath='{.spec.clusterIP}:9000');echo $MINIO_ENDPOINT

.. raw:: html

   <details>

.. raw:: html

   <summary style="color:green">

Show Solution

.. raw:: html

   </summary>

.. raw:: html

   <hr style="background-color:green">

Get the minio endpoint from the ‘px-minio-1’ service and use it to
create portworx credential: MINIO_ENDPOINT=http://$(oc -n minio get svc
px-minio -o jsonpath=‘{.spec.clusterIP}:9000’) ssh -o
strictHostKeyChecking=no node01 sudo pxctl credentials create –provider
s3 –s3-access-key ZZYYXXWWVVUUTT –s3-secret-key 0ldSup3rS3cr3t
–s3-endpoint $MINIO_ENDPOINT –s3-region us-east-1 my-cloud-credentials

.. raw:: html

   <hr style="background-color:green">

.. raw:: html

   </details>

Provision MySQL Database
~~~~~~~~~~~~~~~~~~~~~~~~

We will not create a MySQL database to use with Cloud Snapshots

.. code:: text

   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
       name: px-mysql-sc
   provisioner: kubernetes.io/portworx-volume
   parameters:
      repl: "3"
      io_profile: "db"
      io_priority: "high"
   ---
   kind: PersistentVolumeClaim
   apiVersion: v1
   metadata:
      name: px-mysql-pvc
      annotations:
        volume.beta.kubernetes.io/storage-class: px-mysql-sc
   spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mysql
   spec:
     selector:
       matchLabels:
         app: mysql
     replicas: 1
     template:
       metadata:
         labels:
           app: mysql
       spec:
         schedulerName: stork
         containers:
         - name: mysql
           image: mysql:5.6
           imagePullPolicy: "Always"
           env:
           - name: MYSQL_ALLOW_EMPTY_PASSWORD
             value: "1"
           ports:
           - containerPort: 3306
           volumeMounts:
           - mountPath: /var/lib/mysql
             name: mysql-data
         volumes:
         - name: mysql-data
           persistentVolumeClaim:
             claimName: px-mysql-pvc
   EOF

.. code:: text

   oc create -f /tmp/create-objects.yaml
   oc wait pod --for=condition=Ready -l app=mysql --timeout=-1s

.. code:: text

   POD=`oc get pods -l app=mysql | grep Running | grep 1/1 | awk '{print $1}'`
   oc exec -it $POD -- mysql -u root -e "Create database demodb"

Take Cloud Snapshot
~~~~~~~~~~~~~~~~~~~

We have deployed a mysql pod that uses PortWorx volume. Take a cloud
snapshot of this PVC called ``mysql-snapshot``. The snapshot should be
successfully backed up to the object store.

.. code:: text

   cat <<'EOF' > /tmp/cloud-snap.yaml
   apiVersion: volumesnapshot.external-storage.k8s.io/v1
   kind: VolumeSnapshot
   metadata:
     name: mysql-snapshot
     namespace: default
     annotations:
       portworx/snapshot-type: cloud
   spec:
     persistentVolumeClaimName: px-mysql-pvc
   EOF

.. raw:: html

   <details>

.. raw:: html

   <summary style="color:green">

Show Solution

.. raw:: html

   </summary>

.. raw:: html

   <hr style="background-color:green">

We have created a solution file under ‘/tmp/cloud-snap.yaml’. Create it
by running: oc apply -f /tmp/cloud-snap.yaml

.. raw:: html

   <hr style="background-color:green">

.. raw:: html

   </details>

If the cloud credentials and volume snapshot were set up correctly, you
can check the status by running the below command:

.. code:: text

   oc describe volumesnapshot.volumesnapshot mysql-snapshot

To check for the backed up objects in the object store:

.. code:: text

   ssh node01 mc ls px/

Clone PVC
~~~~~~~~~

Create a clone PVC called ``px-mysql-clone-pvc`` by restoring data from
the snapshot ``mysql-snapshot``.

.. code:: text

   cat <<'EOF' > /tmp/restore.yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: px-mysql-clone-pvc
     annotations:
       snapshot.alpha.kubernetes.io/snapshot: mysql-snapshot
   spec:
     accessModes:
        - ReadWriteOnce
     storageClassName: stork-snapshot-sc
     resources:
       requests:
         storage: 1Gi
   EOF

.. raw:: html

   <details>

.. raw:: html

   <summary style="color:green">

Show Solution

.. raw:: html

   </summary>

.. raw:: html

   <hr style="background-color:green">

We have created a solution file under ‘/tmp/restore.yaml’. Create it by
running: oc apply -f /tmp/restore.yaml Make sure the volume becomes
bound oc get pvc

.. raw:: html

   <hr style="background-color:green">

.. raw:: html

   </details>
