=========================================
Lab 05 - Kubernetes Snapshots
=========================================

.. include:: import-yaml.rst

We will create a Deployment to use with snapshots and restores.
---------------------------------------------------------------

MySQL Deployment

.. code-block:: yaml
  :name: create-mysql.yaml

  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-db-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "3"
    io_profile: "db"
    io_priority: "high"
  ---
  apiVersion: v1
  kind: Namespace
  metadata:
    name: mysql-app
  spec: {}
  status: {}
  ---
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: px-mysql-pvc
    labels:
      app: mysql
    namespace: mysql-app
  spec:
    storageClassName: px-db-sc
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
    namespace: mysql-app
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

Copy the above code block and paste it into the Import YAML.   

Before proceeding to the next step, please make sure the mysql pod is running:

.. code-block:: shell

  Workloads -> Pods

How many pods have been created for MYSQL with label ``app=mysql`` in this cluster (all namespaces)?

.. dropdown:: Show Solution
  
  .. code-block:: shell

    Workloads -> Pods, filter by label: app=mysql

  .. image:: images/mysql-app-pod.png

  Answer: 1

How many PVCs have been created for MYSQL?

1. 2
2. 1
3. 3
4. 4

.. dropdown:: Show Solution

  .. code-block:: shell
    
    Storage -> PersistentVolumeClaims
    
  Answer: 1

What is the name of the storage class used to create this PVC?

.. dropdown:: Show Solution
  
  .. image:: images/px-mysql-pvc.png

  Answer: px-db-sc

What is the ``io_profile`` used for this storage class?

.. dropdown:: Show Solution

  .. code-block:: shell

    Storage -> StorageClasses -> px-db-sc -> YAML

  .. image:: images/px-db-sc-describe.png

  Answer: db

Create a snapshot for MySQL
---------------------------

Create a snapshot called ``mysql-snap`` for the PVC ``px-mysql-pvc``.

.. code-block:: yaml
  :name: mysql-snap.yaml

  apiVersion: volumesnapshot.external-storage.k8s.io/v1
  kind: VolumeSnapshot
  metadata:
    name: mysql-snap
    namespace: mysql-app
  spec:
    persistentVolumeClaimName: px-mysql-pvc
 
Copy the above code block and paste it into the Import YAML.   

Restore the snapshot for MySQL
------------------------------

Restore the snapshot to the same PVC ``px-mysql-pvc`` in the same
Namespace as the source. Call the restore object as
``mysql-snap-restore``.

.. code-block:: yaml
  :name: restore-mysql.yaml

  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: VolumeSnapshotRestore
  metadata:
    name: mysql-snap-restore
    namespace: mysql-app
  spec:
    groupSnapshot: false
    sourceName: mysql-snap
    sourceNamespace: mysql-app
   
Copy the above code block and paste it into the Import YAML.   


We will create a Statefulset to use with snapshots and restores.
----------------------------------------------------------------

We will create a new StatefulSet for you to explore.

NGinx statefulSet

.. code-block:: yaml
  :name: create-nginx-sts.yaml
  
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
      name: px-sc
  provisioner: pxd.portworx.com
  parameters:
     repl: "2"
     io_priority: "high"
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx
    labels:
      app: nginx
  spec:
    ports:
    - port: 80
      name: web
    clusterIP: None
    selector:
      app: nginx
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: web
  spec:
    serviceName: "nginx"
    replicas: 2
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: k8s.gcr.io/nginx-slim:0.8
          ports:
          - containerPort: 80
            name: web
          volumeMounts:
          - name: www
            mountPath: /usr/share/nginx/html
    volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        storageClassName: px-sc
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi

Copy the above code block and paste it into the Import YAML.   

Before proceeding to the next step, please make sure all the resources are up:

.. code-block:: shell
   
  Workloads -> Pods

Note: Please wait until both pods are in a ``Running`` state.

Create a snapshot for Nginx
---------------------------

Create a group snapshot called ``nginx-group-snap`` for the PVC's of the nginx StatefulSet.

.. code-block:: yaml
  :name: nginx-snap.yaml

  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: GroupVolumeSnapshot
  metadata:
    name: nginx-group-snap
  spec:
    pvcSelector:
      matchLabels:
        app: nginx
    restoreNamespaces:
     - default

Copy the above code block and paste it into the Import YAML.   

Restore the snapshot for Nginx
------------------------------

Restore the snapshot taken for the pod ``web-0`` to a new PVC ``web-clone-0`` in the ``default`` namespace.

.. note:: 
   
  Use this command to find the volumesnapshot identifier for web-0: 

  .. code-block:: shell

    Home -> Search -> Resources 
    Search for VolumeSnapshots.volumesnapshot.external-storage.k8s.io/v1

  Copy the identifier that will be found in the Name after “nginx-group-snap-www-web-0-”. Now, use the below template to create a clone from the volumesnapshot for PVC of ``pod - 0`` of the nginx StatefulSet. You must modify the yaml file to add the volumesnapshot identifier for web-0. The line to be edited is highlighted. 

  .. image:: images/volumesnapshot-example.png

  .. important:: 

    Edit the below section and use the snapshot ID found from above.
    
  Create the restore object after editing. 
  
.. code-block:: yaml
  :name: restore-nginx.yaml

  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: web-clone-0
    annotations:
      snapshot.alpha.kubernetes.io/snapshot: nginx-group-snap-www-web-0-<snapshot_id>
  spec:
    accessModes:
       - ReadWriteOnce
    storageClassName: stork-snapshot-sc
    resources:
      requests:
        storage: 1Gi

Copy the above code block and paste it into the Import YAML.   
