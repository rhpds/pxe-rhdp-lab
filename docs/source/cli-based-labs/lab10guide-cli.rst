=================================
Lab 10 - PVC Manual Resize Volume
=================================

Lab setup
---------

First we will create a few PVCs and a StatefulSet with 2 replicas for
you to explore.

.. code-block:: shell

  cat <<EOF > /tmp/create-pvc.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1beta1
  metadata:
      name: px-default-sc
  provisioner: kubernetes.io/portworx-volume
  parameters:
     repl: "2"
     io_priority: "high"
  ---
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
  EOF

.. code-block:: shell

  cat <<EOF > /tmp/create-nginx-sts.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1beta1
  metadata:
      name: px-sc
  provisioner: kubernetes.io/portworx-volume
  parameters:
     repl: "2"
     io_priority: "high"
  allowVolumeExpansion: true
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
        annotations:
          volume.beta.kubernetes.io/storage-class: px-sc
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 5Gi
  EOF

Create the PVCs and statefulSet

.. code-block:: shell

  oc create -f /tmp/create-pvc.yaml
  oc create -f /tmp/create-nginx-sts.yaml

Wait for nginx to be ready
--------------------------

echo “Waiting for Nginx to be ready….”

.. code-block:: shell

  oc wait pod web-0 –for=condition=Ready –timeout=-1s 
  oc exec web-0 – dd if=/dev/zero of=/usr/share/nginx/html/file2.txt count=4101024 bs=1024
  echo “Nginx initialized successfully….”

Before proceeding, please make sure all the pods are up:

.. code-block:: shell 

  oc get pods -n default -l app=nginx

Challenge questions
-------------------

Inspect the PersistentVolumeClaims on this cluster (default namespace)

Q1: How many PVC’s have been created?

1. 2
2. 5
3. 3
4. 1

.. dropdown:: Show Solution
   
   Run the below command: 

   .. code-block:: shell
      
    oc get pvc
   
   Answer: 5

Q2: What is the Size of the PVC called ‘pvc1’?

1. 1Gi
2. 3Gi
3. 5Gi
4. 2Gi

.. dropdown:: Show Solution
   
   Run the below command: 
   
   .. code-block:: shell 
   
    oc describe pvc pvc1
   
   Answer: 2Gi

Q3: What is the Access Mode used for the PVC called ‘pvc3’?

1. RWX
2. RWO
3. ROX

.. dropdown:: Show Solution
   
   Run the below command: 

   .. code-block:: shell

    oc describe pvc pvc3

   Answer: RWX

Resize the pvc1
---------------

Try to update the size of ``pvc1`` to 8Gi.

.. code-block:: shell

  oc edit pvc pvc1

Are you able to do it? Inspect the storage class used by this PVC.

.. dropdown:: Show Solution

   The storage class ‘px-default-sc’ does not have ‘allowVolumeExpansion’
   enabled. As a result you cannot resize this PVC! 

   .. code-block:: 
      
    oc describe sc px-default-sc

Inspect www-web-0 PVC
---------------------

The volume mounted on the pod ``web-0`` seems to be running out of
space. Inspect it!

.. code-block:: shell

  oc exec web-0 -- df -hP /usr/share/nginx/html

Resize www-web-0 PVC
--------------------

Manually resize this PVC ``www-web-0`` to 8GiB.

.. dropdown:: Show Solution
   
   Edit the PVC and change the size to 8Gi: 

   .. code-block:: shell
      
    oc edit pvc www-web-0

Inspect www-web-0 PVC again
---------------------------

Check out the utilization of the volume after the resize.

It takes approximately 30s to complete resizing.

.. code-block:: shell

  oc describe pvc www-web-0

Once ExpandVolume succeds, run the below command:

.. code-block:: shell

  oc exec web-0 -- df -hP /usr/share/nginx/html

In this lab we successfully resized a PVC manually. This can be done
automatically using Autopilot. We will discuss this in the upcoming
lectures.
