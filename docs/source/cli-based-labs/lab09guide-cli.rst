========================================
Lab 09 - PVC Auto Resize using AutoPilot
========================================

In this step, we will create a Portworx volume (PVC) for postgres.

Create StorageClass and PersistentVolumeClaim
---------------------------------------------------

Take a look at the StorageClass definition for Portworx

.. code-block:: shell

  cat <<EOF > /tmp/px-repl3-sc.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-repl3-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "3"
    io_profile: "db"
    priority_io: "high"
  allowVolumeExpansion: true
  EOF

The parameters are declarative policies for your storage volume. Notice the “allowVolumeExpansion: true” this needs to be added for kubernetes volume expansion. See `here <https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-pvcs/dynamic-provisioning/>`__ for a full list of supported parameters.

Create the storage class using:

.. code-block:: shell

  oc create -f /tmp/px-repl3-sc.yaml

Take a look at the Persistent Volume Claim

.. code-block:: shell

  cat <<EOF > /tmp/px-postgres-pvc.yaml
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: px-postgres-pvc
    labels:
      app: postgres
  spec:
    storageClassName: px-repl3-sc
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  EOF

This defines the maximum volume size. Portworx will thin provision the volume.

Create the PersistentVolumeClaim using:

.. code-block:: shell

  oc create -f /tmp/px-postgres-pvc.yaml

Configuring monitoring rules
----------------------------

First we will create rules to monitor Postgres !

.. code-block:: shell

  cat <<EOF > /tmp/pwx-monitoring.yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    namespace: portworx
    name: portworx-prometheus-sm
    labels:
      name: portworx-prometheus-sm
  spec:
    selector:
      matchLabels:
        name: portworx
    namespaceSelector:
      any: true
    endpoints:
      - port: px-api
        targetPort: 9001
      - port: px-kvdb
        targetPort: 9019
  ---
  apiVersion: monitoring.coreos.com/v1
  kind: PrometheusRule
  metadata:
    labels:
      prometheus: portworx
      role: prometheus-portworx-rulefiles
    name: prometheus-portworx-rules-portworx.rules.yaml
    namespace: portworx
  spec:
    groups:
    - name: portworx.rules
      rules:
      - alert: PortworxVolumeUsageCritical
        annotations:
          description: Portworx volume {{$labels.volumeid}} on {{$labels.host}} is over 80% used for
            more than 10 minutes.
          summary: Portworx volume capacity is at {{$value}}% used.
        expr: 100 * (px_volume_usage_bytes / px_volume_capacity_bytes) > 80
        for: 5m
        labels:
          issue: Portworx volume {{$labels.volumeid}} usage on {{$labels.host}} is high.
          severity: critical
      - alert: PortworxVolumeUsage
        annotations:
          description: Portworx volume {{$labels.volumeid}} on {{$labels.host}} is over 70% used for
            more than 10 minutes.
          summary: Portworx volume {{$labels.volumeid}} on {{$labels.host}} is at {{$value}}% used.
        expr: 100 * (px_volume_usage_bytes / px_volume_capacity_bytes) > 70
        for: 5m
        labels:
          issue: Portworx volume {{$labels.volumeid}} usage on {{$labels.host}} is critical.
          severity: warning
      - alert: PortworxVolumeWillFill
        annotations:
          description: Disk volume {{$labels.volumeid}} on {{$labels.host}} is over 70% full and has
            been predicted to fill within 2 weeks for more than 10 minutes.
          summary: Portworx volume {{$labels.volumeid}} on {{$labels.host}} is over 70% full and is
            predicted to fill within 2 weeks.
        expr: (px_volume_usage_bytes / px_volume_capacity_bytes) > 0.7 and predict_linear(px_cluster_disk_available_bytes[1h],
          14 * 86400) < 0
        for: 10m
        labels:
          issue: Disk volume {{$labels.volumeid}} on {{$labels.host}} is predicted to fill within
            2 weeks.
          severity: warning
      - alert: PortworxStorageUsageCritical
        annotations:
          description: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is over 80% used
            for more than 10 minutes.
          summary: Portworx storage capacity is at {{$value}}% used.
        expr: 100 * (1 - px_cluster_disk_utilized_bytes / px_cluster_disk_total_bytes)
          < 20
        for: 5m
        labels:
          issue: Portworx storage {{$labels.volumeid}} usage on {{$labels.host}} is high.
          severity: critical
      - alert: PortworxStorageUsage
        annotations:
          description: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is over 70% used
            for more than 10 minutes.
          summary: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is at {{$value}}% used.
        expr: 100 * (1 - (px_cluster_disk_utilized_bytes / px_cluster_disk_total_bytes))
          < 30
        for: 5m
        labels:
          issue: Portworx storage {{$labels.volumeid}} usage on {{$labels.host}} is critical.
          severity: warning
      - alert: PortworxStorageWillFill
        annotations:
          description: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is over 70% full
            and has been predicted to fill within 2 weeks for more than 10 minutes.
          summary: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is over 70% full and
            is predicted to fill within 2 weeks.
        expr: (100 * (1 - (px_cluster_disk_utilized_bytes / px_cluster_disk_total_bytes)))
          < 30 and predict_linear(px_cluster_disk_available_bytes[1h], 14 * 86400) <
          0
        for: 10m
        labels:
          issue: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is predicted to fill within
            2 weeks.
          severity: warning
      - alert: PortworxStorageNodeDown
        annotations:
          description: Portworx Storage Node has been offline for more than 5 minutes.
          summary: Portworx Storage Node is Offline.
        expr: max(px_cluster_status_nodes_storage_down) > 0
        for: 5m
        labels:
          issue: Portworx Storage Node is Offline.
          severity: critical
      - alert: PortworxQuorumUnhealthy
        annotations:
          description: Portworx cluster Quorum Unhealthy for more than 5 minutes.
          summary: Portworx Quorum Unhealthy.
        expr: max(px_cluster_status_cluster_quorum) > 1
        for: 5m
        labels:
          issue: Portworx Quorum Unhealthy.
          severity: critical
      - alert: PortworxMemberDown
        annotations:
          description: Portworx cluster member(s) has(have) been down for more than
            5 minutes.
          summary: Portworx cluster member(s) is(are) down.
        expr: (max(px_cluster_status_cluster_size) - count(px_cluster_status_cluster_size))
          > 0
        for: 5m
        labels:
          issue: Portworx cluster member(s) is(are) down.
          severity: critical
  ---
  apiVersion: monitoring.coreos.com/v1
  kind: Prometheus
  metadata:
    name: prometheus
    namespace: portworx
  spec:
    replicas: 2
    logLevel: debug
    serviceAccountName: prometheus
    alerting:
      alertmanagers:
        - namespace: portworx
          name: alertmanager-portworx
          port: web
    serviceMonitorSelector:
      matchLabels:
        name: portworx-prometheus-sm
      namespaceSelector:
        matchNames:
          - portworx
      resources:
        requests:
          memory: 400Mi
    ruleSelector:
      matchLabels:
        role: prometheus-portworx-rulefiles
        prometheus: portworx
      namespaceSelector:
        matchNames:
          - portworx

.. code-block:: shell

  #oc apply -f /tmp/portworx-pxc-operator.yaml
  oc apply -f /tmp/pwx-monitoring.yaml

In this step, we will deploy the postgres application using the ``PersistentVolumeClaim`` created before.

Create secret for postgres
--------------------------

Below we are creating a Secret to store the postgres password.

.. code-block:: shell

  echo -n mysql123 > password.txt
  oc create secret generic postgres-pass --from-file=password.txt

Below we will create a Postgres `Deployment <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>`__ that uses a Portworx PVC.

Deploy Postgres
~~~~~~~~~~~~~~~

Now that we have the volumes created, let’s deploy Postgres !

.. code-block:: shell

  cat <<EOF > /tmp/postgres-app.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: postgres
  spec:
    selector:
      matchLabels:
        app: postgres
    strategy:
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 1
      type: RollingUpdate
    replicas: 1
    template:
      metadata:
        labels:
          app: postgres
      spec:
        schedulerName: stork
        containers:
        - name: postgres
          image: postgres:9.5
          imagePullPolicy: "IfNotPresent"
          ports:
          - containerPort: 5432
          env:
          - name: POSTGRES_USER
            value: pgbench
          - name: PGUSER
            value: pgbench
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef:
                name: postgres-pass
                key: password.txt
          - name: PGBENCH_PASSWORD
            value: superpostgres
          - name: PGDATA
            value: /var/lib/postgresql/data/pgdata
          volumeMounts:
          - mountPath: /var/lib/postgresql/data
            name: postgredb
        volumes:
        - name: postgredb
          persistentVolumeClaim:
            claimName: px-postgres-pvc
  EOF

Observe the ``volumeMounts`` and ``volumes`` sections where we mount the PVC.

Now use oc to deploy postgres.

.. code-block:: shell

  oc create -f /tmp/postgres-app.yaml

Verify postgres pod is ready
----------------------------

Below commands wait till the postgres pods are in ready state.

.. code-block:: shell

  watch oc get pods -l app=postgres -o wide

When the pod is in Running state then then hit ``ctrl-c`` to exit.

In this step, we will use pxctl to inspect the volume

Inspect the Portworx volume
---------------------------

Portworx ships with a `pxctl <https://docs.portworx.com/reference/cli/basics/>`__ command line that can be used to manage Portworx.

Below we will use pxctl to inspect the underlying volume for our PVC.

.. code-block:: shell

  VOL=`oc get pvc | grep px-postgres-pvc | awk '{print $3}'`
  PX_POD=$(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}')
  oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl volume inspect ${VOL}

Make the following observations in the inspect output \* ``State`` indicates the volume is attached and shows the node on which it is attached. This is the node where the Kubernetes pod is running. \* ``HA`` shows the number of configured replicas for this volume \* ``Labels`` show the name of the PVC for this volume \* ``Replica sets on nodes`` shows the px nodes on which volume is replicated \* ``Size`` of the volume is 1GB. We’ll check this later to see our volume property expanded.

Now that we have PostgreSQL up, let’s proceed to setting up our AutoPilot rule!

In this step, we will configure the AutoPilot rule for Postgres

Configure Autopilot Rule
------------------------

Learn more about `working with AutoPilot Rules <https://2.11.docs.portworx.com/portworx-install-with-kubernetes/autopilot/how-to-use/working-with-rules/#understanding-an-autopilotrule>`__ in the Portworx documentation.

Keep in mind, an AutoPilot Rule has 4 main parts.

-  ``Selector`` Matches labels on the objects that the rule should monitor.
-  ``Namespace Selector`` Matches labels on the Kubernetes namespaces the rule should monitor. This is optional, and the default is all namespaces.
-  ``Conditions`` The metrics for the objects to monitor.
-  ``Actions`` to perform once the metric conditions are met.

Below we target the Postgres PVC using an AutPilot Rule.

View the AutoPilot Rule
-----------------------

.. code-block:: shell

  cat <<EOF > /tmp/pvc-resize-rule.yaml
  apiVersion: autopilot.libopenstorage.org/v1alpha1
  kind: AutopilotRule
  metadata:
    name: auto-volume-resize
  spec:
    selector:
      matchLabels:
        app: postgres
    conditions:
      # volume usage should be less than 20%
      expressions:
      - key: "100 * (px_volume_usage_bytes / px_volume_capacity_bytes)"
        operator: Gt
        values:
          - "20"
      # volume capacity should not exceed 400GiB
      - key: "px_volume_capacity_bytes / 1000000000"
        operator: Lt
        values:
         - "20"
    actions:
    - name: openstorage.io.action.volume/resize
      params:
        # resize volume by scalepercentage of current size
        scalepercentage: "200"
  EOF

Note that we are defining the ``condition`` and the ``action`` in which our Rule is activated. In our Rule we are defining when our volume is using ``20%`` of its total available capacity, then we grow the volume using the ``openstorage.io.action.volume/resize`` action by 200 percent. Normally, you would likely use a larger threshold for volume usage.

Create the AutoPilot Rule
-------------------------

If you receive an error of ``no matches for kind "AutopilotRule"`` wait 1 minute and try again. AutoPilot installs in the background and if you clicked through this demo too fast it may not be ready just yet.

.. code-block:: shell

  oc apply -f /tmp/pvc-resize-rule.yaml

Verify that AutoPilot initialized the Postgres PVC
--------------------------------------------------

.. code-block:: shell

  watch oc get events --field-selector involvedObject.kind=AutopilotRule,involvedObject.name=auto-volume-resize --all-namespaces

Check to see that AutoPilot has recognized the PVC and initialized it.
When the events show ``transition from Initializing => Normal`` for the Postgres PVC, AutoPilot is ready. Hit ``ctrl-c`` to exit.

In this step, we will run a benchmark that uses more than 20% of our volume and show how AutoPilot dynamically increases the volume size without downtime or user intervention.

Open a shell inside the postgres container
------------------------------------------

Below commands exec into the postgres pod:

.. code-block:: shell

  POD=`oc get pods -l app=postgres | grep Running | grep 1/1 | awk '{print $1}'`
  oc exec -it $POD -- bash

Next we can launch the psql utility and create a database

.. code-block:: shell

  psql
  create database pxdemo;
  \l
  \q

Use pgbench to run a baseline transaction benchmark which will try to grow the volume to a size that is greater than the 20% that we defined in our AutoPilot Rule. This should trigger AutoPilot to resize the volume.

.. code-block:: shell

  pgbench -i -s 50 pxdemo

.. note:: Note that once the test completes, **AutoPilot will make sure the usage remains above 20% for about 30 seconds before triggering the rule.** Type ``exit`` to exit from the pod shell before proceeding.

Check to see if the rule was triggered
--------------------------------------

We can retrieve events by using the ``oc get events`` and filtering for ``AutoPilotRule`` events that match our use case. Note, that AutoPilot delays the rule from being triggered immediately to ensure that the conditions stablize, so make sure to **hang tight and see the rule get triggered if you dont see it right away, it may take a minute or two**.

.. code-block:: shell

  watch oc get events --field-selector involvedObject.kind=AutopilotRule,involvedObject.name=auto-volume-resize --all-namespaces

When you see ``Triggered => ActiveActionsPending`` the action has been activated. When you see ``ActiveActionsInProgress => ActiveActionsTake`` this means the resize has taken place and your volume should be resized by **200%**. Hit ``ctrl-c`` to clear the screen.

Inspect the volume and verify that it now has grown by 200% capacity (3GB).

.. code-block:: shell

  oc get pvc px-postgres-pvc

As you can see the volume is now expanded and our PostgresDB database didn’t require restarting.

.. code-block:: shell

  oc get pods

That’s it, you’re done!
