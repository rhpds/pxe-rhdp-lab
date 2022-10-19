===========================
Lab 08 - Snapshot Schedules
===========================

Deploy MySQL and create a schedulePolicies
------------------------------------------

.. code-block:: shell

  cat <<EOF > /tmp/create-mysql.yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
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
    replicas: 1
    selector:
      matchLabels:
        app: mysql
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

  cat <<EOF > /tmp/create-schedpol.yaml
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: SchedulePolicy
  metadata:
    name: daily
  policy:
    daily:
      time: "10:14PM"
      retain: 3
  ---
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: SchedulePolicy
  metadata:
    name: pol1
  policy:
    interval:
      intervalMinutes: 60
      retain: 3
  ---
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: SchedulePolicy
  metadata:
    name: weekly
  policy:
    weekly:
      day: "Thursday"
      time: "10:13PM"
      retain: 5
  EOF

.. code-block:: shell

  oc apply -f /tmp/create-mysql.yaml
  oc apply -f /tmp/create-schedpol.yaml

Before proceeding, make sure all the pods are up and ready:

.. code-block:: shell

  oc get pods -n mysql-app  -l app=mysql

Challenge questions
-------------------

How many schedule policies have been created?

.. dropdown:: Show Solution
   
  Run: 

  .. code-block:: shell

    oc get schedulepolicies

  Answer: 8

What is the retenton period of the ``weekly`` policy?

1. 2
2. 5
3. 3
4. 4

.. dropdown:: Show Solution

  Run: 
  
  .. code-block:: shell
    
    oc describe schedulepolicies weekly

  Answer: 5

What is snapshot frequency set for the policy ``pol1``?

1. Everyday at 6 AM
2. Everyday at 12 AM
3. Every 60 minutes

.. dropdown:: Show Solution
   
  Run: 

  .. code-block:: shell
    
    oc describe schedulepolicies pol1

  Answer: Every 60 minutes

Create a new snapshot schedule policy
-------------------------------------

Create a daily snapshot schedule policy called ``daily-schedule`` at ``10 PM``, ``retain 5``.

.. code-block:: shell

  cat <<EOF > /tmp/sched-pol.yaml
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: SchedulePolicy
  metadata:
    name: daily-schedule
  policy:
    daily:
      time: "10:00PM"
      retain: 5
  EOF

.. dropdown:: Show Solution

  Run the below command to create the snapshot: 

  .. code-block:: shell

    oc create -f /tmp/sched-pol.yaml


Create a storageClass that uses this schedule policy
----------------------------------------------------

Create a storage class ``px-nginx-scheduled`` with the newly created schedule policy ``daily-schedule``

.. code-block:: shell

  cat <<EOF > /tmp/px-nginx-scheduled.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-nginx-scheduled
  provisioner: pxd.portworx.com
  parameters:
    repl: "2"
    io_priority: "high"
    snapshotschedule.stork.libopenstorage.org/default-schedule: |
      schedulePolicyName: daily-schedule
      annotations:
        portworx/snapshot-type: local
  EOF

.. dropdown:: Show Solution
  
  Run the below command to create the storage class: 
  
  .. code-block:: shell
  
    oc create -f /tmp/px-nginx-scheduled.yaml


Create a Nginx StatefulSet that utilizes this storageClass
----------------------------------------------------------

Create a new NGINX StatefulSet, making use of the ``px-nginx-scheduled`` storage class.

Use the YAML file ``/tmp/create-nginx-sts.yaml`` to create the deployment.

.. code-block:: shell

  cat <<EOF > /tmp/create-nginx-sts.yaml
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
        storageClassName: px-nginx-scheduled
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
  EOF

The PVC's created by the StatefulSet will be backed up automatically as per the schedule policy ``daily-schedule``.

.. dropdown:: Show Solution

  Run the below command to create the NGINX sts: 

  .. code-block:: shell
    
    oc create -f /tmp/create-nginx-sts.yaml
