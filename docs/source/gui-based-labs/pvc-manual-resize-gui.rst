=================================
Lab 10 - PVC Manual Resize Volume
=================================

Lab setup
---------

First we will create a few PVCs and a StatefulSet with 2 replicas for you to explore.

.. code-block:: yaml
  :name: create-default-sc.yaml

  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-default-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "2"
    io_priority: "high"

.. code-block:: yaml
  :name: create-pvc.yaml
  
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pvc1
  spec:
    storageClassName: px-default-sc
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 2Gi
  ---
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pvc2
  spec:
    storageClassName: px-default-sc
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 3Gi
  ---
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pvc3
  spec:
    storageClassName: px-default-sc
    accessModes:
    - ReadWriteMany
    resources:
      requests:
        storage: 1Gi

.. code-block:: yaml
  :name: create-px-sc.yaml
  
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "2"
    io_priority: "high"
  allowVolumeExpansion: true

.. code-block:: yaml
  :name: create-nginx-sts.yaml

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
            storage: 5Gi

Create the PVCs and statefulSet

Copy the above code blocks and paste it into the Import YAML.   

Wait for nginx to be ready
--------------------------

Waiting for Nginx to be ready….

.. code-block:: shell

  Workload -> Pods -> web-0 -> Status: Ready

Open terminal to web-0 

.. code-block:: shell
  
  Workload -> Pods -> web-0 -> Terminal

.. code-block:: shell

  dd if=/dev/zero of=/usr/share/nginx/html/file2.txt count=4101024 bs=1024

Before proceeding, please make sure all the pods are up:

.. code-block:: shell 

    Workload -> Pods -> Filter by Label: app=nginx

Challenge questions
-------------------

Inspect the PersistentVolumeClaims on this cluster (default namespace)

Q1: How many PVC's have been created?

1. 2
2. 5
3. 3
4. 1

.. dropdown:: Show Solution
   
   Run the below command: 

   .. code-block:: shell
      
    Storage -> PersistentVolumeClaims
   
   Answer: 5

Q2: What is the Size of the PVC called ``pvc1``?

1. 1Gi
2. 3Gi
3. 5Gi
4. 2Gi

.. dropdown:: Show Solution
   
   Run the below command: 
   
   .. code-block:: shell 
   
    Storage -> PersistentVolumeClaims -> pvc1 -> Capacity
   
   Answer: 2Gi

Q3: What is the Access Mode used for the PVC called ``pvc3``?

1. RWX
2. RWO
3. ROX

.. dropdown:: Show Solution
   
   Run the below command: 

   .. code-block:: shell

    Storage -> PersistentVolumeClaims -> pvc3 -> Access modes

   Answer: RWX

Resize the pvc1
---------------

Try to update the size of ``pvc1`` to 8Gi.

.. code-block:: shell

  Storage -> PersistentVolumeClaims -> pvc1 -> YAML -> edit spec.requests.resources.capacity

Are you able to do it? Inspect the storage class used by this PVC.

.. dropdown:: Show Solution

  The storage class ``px-default-sc`` does not have ``allowVolumeExpansion`` enabled. As a result you cannot resize this PVC! 

   .. code-block:: 
      
    Storage -> Storageclasses -> px-default-sc -> YAML 

Inspect www-web-0 PVC
---------------------

The volume mounted on the pod ``web-0`` seems to be running out of space. Inspect it!

.. code-block:: shell
  
  Workload -> Pods -> web-0 -> Terminal

.. code-block:: shell

  df -hP /usr/share/nginx/html

Resize www-web-0 PVC
--------------------

Manually resize this PVC ``www-web-0`` to 8GiB.

.. dropdown:: Show Solution
   
   Edit the PVC and change the size to 8Gi: 

   .. code-block:: shell
      
    Storage -> PersistentVolumeClaims -> www-web-0 -> YAML -> edit spec.requests.resources.capacity

Inspect www-web-0 PVC again
---------------------------

Check out the utilization of the volume after the resize.

It takes approximately 30s to complete resizing.

.. code-block:: shell

  Storage -> PersistentVolumeClaims -> www-web-0

Once ExpandVolume succeds, run the below command:

.. code-block:: shell
  
  Workload -> Pods -> web-0 -> Terminal

.. code-block:: shell

  df -hP /usr/share/nginx/html

In this lab we successfully resized a PVC manually. This can be done automatically using Autopilot. We will discuss this in the upcoming lectures.
