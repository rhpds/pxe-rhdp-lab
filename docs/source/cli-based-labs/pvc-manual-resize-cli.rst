=================================
Lab 10 - PVC Manual Resize Volume
=================================

Lab setup
---------

First we will create a few PVCs and a StatefulSet with 2 replicas for you to explore.

.. code-block:: shell

  cat <<EOF | oc apply -f -
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-default-sc
  provisioner: pxd.portworx.com
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

  cat <<EOF | oc apply -f -
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-sc
  provisioner: pxd.portworx.com
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
    name: web-resize
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
  EOF



Resize www-web-resize-0 PVC
--------------------

Manually resize this PVC ``www-web-resize-0`` to 8GiB.

.. dropdown:: Show Solution
   
   Edit the PVC and change the size to 8Gi: 

   .. code-block:: shell
      
    oc edit pvc www-web-resize-0

Inspect www-web-0 PVC again
---------------------------

Check out the utilization of the volume after the resize.

It takes approximately 30s to complete resizing.

.. code-block:: shell

  oc describe pvc www-web-resize-0


In this lab we successfully resized a PVC manually. This can be done automatically using Autopilot. We will discuss this in the upcoming lectures.
