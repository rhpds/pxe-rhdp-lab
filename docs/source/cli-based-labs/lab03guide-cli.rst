=========================
Lab 03 - Deploy Cassandra
=========================

Before we deploy cassandra, we will need to create a Portworx volume
(PVC) for Cassandra. In order to create PVCs, we need a StorageClass
which defined the class of storage available to us.

Create StorageClass
-------------------------

Take a look at the StorageClass definition for Cassandra:

.. code-block:: shell

  cat <<EOF > /tmp/cassandra-sc.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: px-storageclass
  provisioner: pxd.portworx.com
  parameters:
    repl: "2"
    priority_io: "high"
    group: "cassandra_vg"
  EOF

Note that we define a replication factor of 2 to accelerate Cassandra
node recovery and we also defined a group name for Cassandra so that we
can take
`3DSnapshots <https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-snapshots/snaps-3d/>`__
which will be consistent across the whole Cassandra cluster. In
production environment which larger clusters you would also add the
“fg=true” parameter to your StorageClass to ensure that Portworx places
each Cassandra volume and their replica on separate nodes so that in
case of node failure we never failover Kafka to a node where it is
already running. To enable this feature with a 3 volume group and 2
replicas you need a minimum of 6 worker nodes.

The parameters are declarative policies for your storage volume. See
`here <https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-pvcs/dynamic-provisioning/>`__
for a full list of supported parameters.

Create the storage class using:

.. code-block:: shell

   oc create -f /tmp/cassandra-sc.yaml

Now that we have the StorageClass created, let’s deploy Cassandra!

In this step, we will deploy a 3 node Cassandra application using a
stateful set. To learn more about stateful sets follow this
`link <https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/>`__.

Create the Cassandra StatefulSet
--------------------------------------

Create a Cassandra
`StatefulSet <https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/>`__
that uses a Portworx PVC.

.. code-block:: shell

  cat <<EOF > /tmp/cassandra.yaml
  apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: cassandra
    name: cassandra
  spec:
    clusterIP: None
    ports:
      - port: 9042
    selector:
      app: cassandra
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: cassandra
  spec:
    serviceName: cassandra
    replicas: 1
    selector:
      matchLabels:
        app: cassandra
    template:
      metadata:
        labels:
          app: cassandra
      spec:
        # Use the stork scheduler to enable more efficient placement of the pods
        schedulerName: stork
        terminationGracePeriodSeconds: 1800
        containers:
        - name: cassandra
          image: gcr.io/google-samples/cassandra:v14
          imagePullPolicy: Always
          ports:
          - containerPort: 7000
            name: intra-node
          - containerPort: 7001
            name: tls-intra-node
          - containerPort: 7199
            name: jmx
          - containerPort: 9042
            name: cql
          resources:
            limits:
              cpu: "500m"
              memory: 1Gi
            requests:
             cpu: "500m"
             memory: 1Gi
          securityContext:
            capabilities:
              add:
                - IPC_LOCK
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "PID=$(pidof java) && kill $PID && while ps -p $PID > /dev/null; do sleep 1; done"]
          env:
            - name: MAX_HEAP_SIZE
              value: 512M
            - name: HEAP_NEWSIZE
              value: 100M
            - name: CASSANDRA_SEEDS
              value: "cassandra-0.cassandra.default.svc.cluster.local"
            - name: CASSANDRA_CLUSTER_NAME
              value: "K8Demo"
            - name: CASSANDRA_DC
              value: "DC1-K8Demo"
            - name: CASSANDRA_RACK
              value: "Rack1-K8Demo"
            - name: CASSANDRA_AUTO_BOOTSTRAP
              value: "false"
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          readinessProbe:
            exec:
              command:
              - /bin/bash
              - -c
              - /ready-probe.sh
            initialDelaySeconds: 15
            timeoutSeconds: 5
          # These volume mounts are persistent. They are like inline claims,
          # but not exactly because the names need to match exactly one of
          # the stateful pod volumes.
          volumeMounts:
          - name: cassandra-data
            mountPath: /cassandra_data
    # These are converted to volume claims by the controller
    # and mounted at the paths mentioned above.
    volumeClaimTemplates:
    - metadata:
        name: cassandra-data
      spec:
        storageClassName: px-storageclass
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: cqlsh
  spec:
    containers:
    - name: cqlsh
      image: mikewright/cqlsh
      command:
        - sh
        - -c
        - "exec tail -f /dev/null"
  EOF

Take a look at the yaml:

.. code-block:: shell

  cat /tmp/cassandra.yaml

Observe that the stateful set is exposed through a headless service.
Also note how PVCs will be dynamically created with each member of the
stateful set based on the ``volumeClaimTemplates`` and it’s
``storage-class`` sections. Finally, you will also see that we are
starting with a single node (replicas: 1).

Now use oc to deploy Cassandra.

.. code-block:: shell

  oc create -f /tmp/cassandra.yaml

Verify Cassandra pod is ready
-----------------------------------

Below commands wait till the Cassandra pod are in ready state. Take note
of the node it’s running on.

.. code-block:: shell

  watch oc get pods  -o wide

This takes a few minutes, when the cassandra-0 and cqlsh pods are in
STATUS ``Running`` and ``READY 1/1``, hit ``ctrl-c`` to exit.

In this step, we will use pxctl to inspect the volume

Inspect the Portworx volume
---------------------------------

Portworx ships with a
`pxctl <https://docs.portworx.com/reference/cli/basics/>`__ command line
that can be used to manage Portworx.

Below we will use ``pxctl`` to inspect the underlying volumes for our
Cassandra pod.

.. code-block:: shell

  VOLS=`oc get pvc | grep cassandra | awk '{print $3}'`
  PX_POD=$(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}')
  oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl volume inspect $VOLS

Make the following observations in the inspect output \* ``State``
indicates the volume is attached and shows the node on which it is
attached. This is the node where the Kubernetes pod is running. \*
``HA`` shows the number of configured replicas for this volume \*
``Labels`` show the name of the PVC for this volume \*
``Replica sets on nodes`` shows the px nodes on which volume is
replicated

Now that we have Cassandra up, let’s proceed to run some tests!

In this step, we will initialize a sample database in our cassandra
instance.

Create a table and insert data
------------------------------------

Start a CQL Shell session:

.. code-block:: shell

  oc exec -it cqlsh -- cqlsh cassandra-0.cassandra.default.svc.cluster.local --cqlversion=3.4.4

Create a keyspace with replication of 3 and insert some data:

.. code-block:: sql

  CREATE KEYSPACE portworx WITH REPLICATION = {'class':'SimpleStrategy','replication_factor':3};
  USE portworx;
  CREATE TABLE features (id varchar PRIMARY KEY, name varchar, value varchar);
  INSERT INTO portworx.features (id, name, value) VALUES ('px-1', 'snapshots', 'point in time recovery!');
  INSERT INTO portworx.features (id, name, value) VALUES ('px-2', 'cloudsnaps', 'backup/restore to/from any cloud!');
  INSERT INTO portworx.features (id, name, value) VALUES ('px-3', 'STORK', 'convergence, scale, and high availability!');
  INSERT INTO portworx.features (id, name, value) VALUES ('px-4', 'share-volumes', 'better than NFS, run wordpress on k8s!');
  INSERT INTO portworx.features (id, name, value) VALUES ('px-5', 'DevOps', 'your data needs to be automated too!');

Select rows from the keyspace we just created:

.. code-block:: sql

  SELECT id, name, value FROM portworx.features;

Now that we have data created let’s ``quit`` the cqlsh session.

Flush data to disk
------------------------

Before we proceed to the failover test we will flush the in-memory data
onto disk so that when the cassandra-0 starts on another node it will
have access to the data that was just written (Cassandra keeps data in
memory and only flushes it to disk after 10 minutes by default).

.. code-block:: shell

  oc exec -it cassandra-0 -- nodetool flush

In this step, we will simulate failure by cordoning the node where
Cassandra is running and then deleting the Cassandra pod. The pod will
then be resheduled by the `STorage ORchestrator for Kubernetes
(STORK) <https://github.com/libopenstorage/stork/>`__ to make sure it
lands on one of the nodes that has of replica of the data.

Simulate a node failure to force Cassandra to restart
-----------------------------------------------------------

First we will cordon the node where Cassandra is running to simulate a
node failure or network partition:

.. code-block:: shell

  NODE=`oc get pods -o wide | grep cassandra-0 | awk '{print $7}'`
  oc adm cordon ${NODE}

Then delete the Cassandra pod:

.. code-block:: shell

  POD=`oc get pods -l app=cassandra -o wide | grep -v NAME | awk '{print $1}'`
  oc delete pod ${POD}

Once the cassandra pod gets deleted, Kubernetes will start to create a
new cassandra pod on another node.

Verify replacement pod starts running
-------------------------------------------

Below commands wait till the new cassandra pod is ready.

.. code-block:: shell

  watch oc get pods -l app=cassandra -o wide

Once the pod is in ``Running`` and ``READY(1/1)`` state. Hit ctrl-c to
exit.

Before you proceed you should uncordon your node:

.. code-block:: shell

  oc adm uncordon ${NODE}

Now that we have the new cassandra pod running, let’s check if the
database we previously created is still intact.

In this step, we will check the state of our sample Cassandra database.

Verify data is still available
------------------------------------

Start a CQL Shell session:

.. code-block:: shell

  oc exec -it cqlsh -- cqlsh cassandra-0.cassandra.default.svc.cluster.local --cqlversion=3.4.4

Select rows from the keyspace we previously created:

.. code-block:: sql

  SELECT id, name, value FROM portworx.features;

Now that we have verify our data survived the node failure let’s
``quit`` the cqlsh session before continuing to the next step.

*THIS STEP IS OPTIONAL, (Click “Next” to move to snapshot and restore)*

Scale the cluster
-----------------------

In this step, we will scale our Cassandra stateful set to 3 replicas to
show how portworx Dyanamically creates new PVCs as the statefulset
scales.

Run this command to add two nodes to the Cassandra cluster:

.. code-block:: shell

  oc scale sts cassandra --replicas=3

You can watch the cassandra-1 and cassandra-2 pods get added:

.. code-block:: shell

  watch oc get pods -o wide

After all pods are ``READY 1/1`` and ``Running`` you can hit ``ctrl-c``
to exit the watch screen. Now, to verify that Cassandra is in a running
state you can run the nodetool status utility to verify the health of
our Cassandra cluster

.. code-block:: shell

  oc exec -it cassandra-0 -- nodetool status

It will take a minute or two for all three Cassandra nodes to come
online and discover each other. When it’s ready you should see the
following output in from the ``nodetool status`` command (address and
host ID will vary):

.. code-block:: shell

  root@cassandra-0:/# nodetool status
  Datacenter: DC1-K8Demo
  ======================
  Status=Up/Down
  |/ State=Normal/Leaving/Joining/Moving
  --  Address    Load       Tokens       Owns (effective)  Host ID                               Rack
  UN  10.32.0.4  153.59 KiB  32           100.0%            2fb16c55-1337-4b04-a4a4-13da82cca0cf  Rack1-K8Demo
  UN  10.38.0.3  178.86 KiB  32           100.0%            ee7f6cb5-a631-4987-8888-28d008cfb959  Rack1-K8Demo
  UN  10.40.0.5  101.46 KiB  32           100.0%            e2adf023-04f7-44a4-824b-55e75be7d74c  Rack1-K8Demo

When you see your Cassandra node is in Status=Up and State=Normal (UN)
that means the cluster is fully operational.

Pro Tip: Use jq to get useful cluster configuration summary
-----------------------------------------------------------

Get the pods and the knowledge of the Hosts on which they are scheduled.

.. code-block:: shell

  oc get pods -l app=cassandra -o json | jq '.items[] | {"name": .metadata.name,"hostname": .spec.nodeName, "hostIP": .status.hostIP, "PodIP": .status.podIP}'

In this step, we will take a snapshot of all volumes for our Cassandra
cluster, then drop our database table.

Take snapshot using oc
----------------------------

First let’s insert a new record in our features table so we can show
that the snapshot will take the latest available data:

.. code-block:: shell

  oc exec -it cqlsh -- cqlsh cassandra-0.cassandra.default.svc.cluster.local --cqlversion=3.4.4
  INSERT INTO portworx.features (id, name, value) VALUES ('px-6', '3DSnaps', 'Application/Cluster aware snapshots!');
  SELECT id, name, value FROM portworx.features;
  quit

We’re going to use STORK to take a 3DSnapshot of our Cassandra cluster.
Take a look at the px-snap.yaml file and notice that we are going to force 
a ``nodetool flush`` command on eachcluster member before we take the snapshot.
As explained before, that will force all data to be written to disk in order 
to ensure consistency of the snapshot. We also defined the volume group 
name (cassandra_vg) so Portworx will synchronously quiesce I/O on all volumes 
before triggering their snapshots.

.. code-block:: shell

  cat <<EOF > /tmp/px-snap.yaml
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: Rule
  metadata:
    name: cassandra-presnap-rule
  rules:
    - podSelector:
        app: cassandra
      actions:
      - type: command
        value: nodetool flush
  ---
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: GroupVolumeSnapshot
  metadata:
    name: cassandra-group-snapshot
  spec:
    preExecRule: cassandra-presnap-rule
    pvcSelector:
      matchLabels:
        app: cassandra
  EOF

Now let’s take a snapshot.

.. code-block:: shell

  oc create -f /tmp/px-snap.yaml

You can see the snapshots using the following command:

.. code-block:: shell

  watch oc get volumesnapshot.volumesnapshot

When you see all 3 volumesnapshots appear, take note of the names and
hit ``ctrl-c`` to exit the screen.

Drop features table
-------------------------

Now we’re going to go ahead and do something stupid because it’s
Katacoda and we’re here to learn.

.. code-block:: shell

  oc exec -it cqlsh -- cqlsh cassandra-0.cassandra.default.svc.cluster.local --cqlversion=3.4.4
  DROP TABLE IF EXISTS portworx.features;
  SELECT id, name, value FROM portworx.features;
  quit

You should have received an “Error” since the table is deleted. Ok, so
we deleted our database, what now?

Create clones from your snapshots and restore from those snapshots.

First edit ``/tmp/vols-from-snaps`` and insert the volumesnapshots names
from the above ``oc get volumesnapshots`` output.

.. code-block:: shell

  cat <<EOF > /tmp/vols-from-snaps.yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: cassandra-snap-data-cassandra-restored-0
    annotations:
      snapshot.alpha.kubernetes.io/snapshot: cassandra-group-snapshot-cassandra-data-cassandra-0-<REPLACE>
  spec:
    accessModes:
       - ReadWriteOnce
    storageClassName: stork-snapshot-sc
    resources:
      requests:
        storage: 10Gi

  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: cassandra-snap-data-cassandra-restored-1
    annotations:
      snapshot.alpha.kubernetes.io/snapshot: cassandra-group-snapshot-cassandra-data-cassandra-1-<REPLACE>
  spec:
    accessModes:
       - ReadWriteOnce
    storageClassName: stork-snapshot-sc
    resources:
      requests:
        storage: 10Gi

  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: cassandra-snap-data-cassandra-restored-2
    annotations:
      snapshot.alpha.kubernetes.io/snapshot: cassandra-group-snapshot-cassandra-data-cassandra-2-<REPLACE>
  spec:
    accessModes:
       - ReadWriteOnce
    storageClassName: stork-snapshot-sc
    resources:
      requests:
        storage: 10Gi
  EOF

.. code-block:: shell

  vim /tmp/vols-from-snaps.yaml

Then create the clones.

.. code-block:: shell

  oc create -f /tmp/vols-from-snaps.yaml

View the PVCs

.. code-block:: shell

  oc get pvc

Restore cassandra. We delete the original Cassandra deployment only
because we dont have enough nodes in this lab to host two. Then we
create the new cassandra statefulset based on our cloned snapshots.

.. code-block:: shell

  cat <<EOF > /tmp/cassandra-app-restore.yaml
  apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: cassandra-restored
    name: cassandra-restored
  spec:
    clusterIP: None
    ports:
      - port: 9042
    selector:
      app: cassandra-restored
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: cassandra-restored
  spec:
    serviceName: cassandra-restored
    replicas: 1
    selector:
      matchLabels:
        app: cassandra-restored
    template:
      metadata:
        labels:
          app: cassandra-restored
      spec:
        # Use the stork scheduler to enable more efficient placement of the pods
        schedulerName: stork
        terminationGracePeriodSeconds: 1800
        containers:
        - name: cassandra
          image: gcr.io/google-samples/cassandra:v14
          imagePullPolicy: Always
          ports:
          - containerPort: 7000
            name: intra-node
          - containerPort: 7001
            name: tls-intra-node
          - containerPort: 7199
            name: jmx
          - containerPort: 9042
            name: cql
          resources:
            limits:
              cpu: "500m"
              memory: 1Gi
            requests:
             cpu: "500m"
             memory: 1Gi
          securityContext:
            capabilities:
              add:
                - IPC_LOCK
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "PID=$(pidof java) && kill $PID && while ps -p $PID > /dev/null; do sleep 1; done"]
          env:
            - name: MAX_HEAP_SIZE
              value: 512M
            - name: HEAP_NEWSIZE
              value: 100M
            - name: CASSANDRA_SEEDS
              value: "cassandra-restored-0.cassandra-restored.default.svc.cluster.local"
            - name: CASSANDRA_CLUSTER_NAME
              value: "K8Demo"
            - name: CASSANDRA_DC
              value: "DC1-K8Demo"
            - name: CASSANDRA_RACK
              value: "Rack1-K8Demo"
            - name: CASSANDRA_AUTO_BOOTSTRAP
              value: "false"
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          readinessProbe:
            exec:
              command:
              - /bin/bash
              - -c
              - /ready-probe.sh
            initialDelaySeconds: 15
            timeoutSeconds: 5
          # These volume mounts are persistent. They are like inline claims,
          # but not exactly because the names need to match exactly one of
          # the stateful pod volumes.
          volumeMounts:
          - name: cassandra-snap-data
            mountPath: /cassandra_data
    # These are converted to volume claims by the controller
    # and mounted at the paths mentioned above.
    volumeClaimTemplates:
    - metadata:
        name: cassandra-snap-data
      spec:
        storageClassName: px-storageclass
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: cqlsh-restored
  spec:
    containers:
    - name: cqlsh
      image: mikewright/cqlsh
      command:
        - sh
        - -c
        - "exec tail -f /dev/null"
  EOF

.. code-block:: shell

  oc delete -f /tmp/cassandra.yaml
  oc create -f /tmp/cassandra-app-restore.yaml

Wait for restored cassandra database to be Running (1/1). *Note there
will be only 1 replica restored*

.. code-block:: shell

  watch oc get pods

When you see all pods Running (1/1), hit ``ctrl-c`` to exit the screen.

New let’s verify the data is restored.

Start a CQL Shell session:

.. code-block:: shell

  oc exec -it cqlsh-restored -- cqlsh cassandra-restored-0.cassandra-restored.default.svc.cluster.local --cqlversion=3.4.4

Select rows from the keyspace we previously created:

.. code-block:: sql

  SELECT id, name, value FROM portworx.features;

You have now restored from a snapshot! Go ahead and ``quit`` the cqlsh
session before finishing.

Thank you for trying the playground. To view all our scenarios, go
`here <https://rhpds-portworx.readthedocs.io/en/latest/index.html>`__

To learn more about `Portworx <https://portworx.com/>`__, below are some
useful references. - `Deploy Portworx on
Kubernetes <https://docs.portworx.com/scheduler/kubernetes/install.html>`__
- `Create Portworx
volumes <https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-pvcs/>`__
- `Use cases <https://portworx.com/use-case/kubernetes-storage/>`__
