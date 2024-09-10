=========================================
Lab 06 - Cloud Snapshots
=========================================

.. important:: We will make use of Minio object store in this Lab. We will use it as the endpoint for our cloud snapshots.

Deploy Minio as target for Portworx Cloud Snapshots
---------------------------------------------------

Create a storageClass for use by Minio

.. code-block:: shell

  cat <<EOF > /tmp/px-ha-sc.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-ha-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "3"
    io_priority: "high"
    group: "minio"
  EOF

.. code-block:: shell

  oc create -f /tmp/px-ha-sc.yaml

Deploy Minio onto the OCP cluster

.. code-block:: shell

  echo "Installing Helm"
  curl -L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash -s -- --version v3.8.2
  helm repo add stable https://charts.helm.sh/stable
  echo "Helm installed!"

.. code-block:: shell

  echo "Installing mc"
  export WORKER_NODE=$(oc get node -l node-role.kubernetes.io/worker= | grep -Eiv "infra|NAME" | awk '{print $1}' | head -1)
  oc debug node/$WORKER_NODE
  chroot /host

.. code-block:: shell

  curl --output /tmp/mc https://dl.minio.io/client/mc/release/linux-amd64/mc
  chmod +x /tmp/mc
  echo "mc installed!"

.. code-block:: shell
  
  exit

.. code-block:: shell

  echo "Setting Up Minio" 
  oc create namespace minio
  oc -n minio adm policy add-scc-to-user anyuid -z px-minio
  helm install px-minio stable/minio --namespace minio --set accessKey=ZZYYXXWWVVUUTT --set secretKey=0ldSup3rS3cr3t --set persistence.storageClass=px-ha-sc --set resources.requests.memory=1Gi > /dev/null 2>&1
  oc -n minio adm policy add-scc-to-user anyuid -z px-minio
  until [[ `oc -n minio get pods | grep px-minio | grep Running | grep 1/1 | wc -l` -eq 1 ]]; do echo "Waiting for px-minio to be ready...."; sleep 1 ;done
  echo "Setup Complete ..."




Run the below command to obtain the object store endpoint:

.. code-block:: shell

  MINIO_ENDPOINT=http://$(oc -n minio get svc px-minio -o jsonpath='{.spec.clusterIP}:9000'); echo $MINIO_ENDPOINT
  pxctl credentials create --provider s3 --s3-access-key ZZYYXXWWVVUUTT --s3-secret-key 0ldSup3rS3cr3t --s3-endpoint $MINIO_ENDPOINT --s3-region us-east-1 my-cloud-credentials


Provision MySQL Database
------------------------

We will now create a MySQL database to use with Cloud Snapshots

.. code-block:: shell

  cat <<EOF | oc create -f -
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-mysql-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "3"
    io_profile: "db"
    io_priority: "high"
  ---
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: px-mysql-pvc
  spec:
    storageClassName: px-mysql-sc
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


.. code-block:: shell

  POD=$(oc get pods -l app=mysql | grep Running | grep 1/1 | awk '{print $1}')
  oc exec -it $POD -- mysql -u root -e "Create database demodb"

Take Cloud Snapshot
-------------------

We have deployed a mysql pod that uses PortWorx volume. Take a cloud snapshot of this PVC called ``mysql-snapshot``. The snapshot should be successfully backed up to the object store.

.. code-block:: shell

  cat <<EOF | oc apply -f -
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


If the cloud credentials and volume snapshot were set up correctly, you can check the status by running the below command:

.. code-block:: shell

  oc describe stork-volumesnapshot mysql-snapshot


Clone PVC
---------

Create a clone PVC called ``px-mysql-clone-pvc`` by restoring data from the snapshot ``mysql-snapshot``.

.. code-block:: shell

  cat <<EOF | oc apply -f -
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

