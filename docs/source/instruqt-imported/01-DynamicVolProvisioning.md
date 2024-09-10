---
slug: step-1
id: zzxwyzisp0q4
type: challenge
title: Dynamic Volume Provisioning for stateful applications
notes:
- type: text
  contents: Let's order some BBQ and provision persistent volumes dynamically!
tabs:
- title: Terminal
  type: terminal
  hostname: cloud-client
  cmd: su - root
- title: PX-BBQ
  type: service
  hostname: cloud-client
  path: /
  port: 81
- title: GKE Console Access
  type: service
  hostname: cloud-client
  path: /index.html
  port: 80
difficulty: basic
timelimit: 43200
---
Scenario - Persistent Storage Volume Provisioning and Availability
=====
In this scenario, you'll learn about Portworx Enterprise StorageClass parameters and deploy a demo application that uses RWO (ReadWriteOnce) Persistent Volumes provisioned by Portworx Enterprise, and see how Portworx makes them highly available.

Step 1 - Deploying Portworx Storage Classes
=====
Portworx provides the ability for users to leverage a unified storage pool to dynamically provision both Block-based (ReadWriteOnce) and File-based (ReadWriteMany) volumes for applications running on your Kubernetes cluster without having to provision multiple CSI drivers/plugins, and without the need for specific backing storage devices!

Run the following command to create a new yaml file for the block-based StorageClass configuration:
```bash
cat << EOF > ~/px-repl3.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: px-repl3
parameters:
  io_profile: db_remote
  repl: "3"
provisioner: pxd.portworx.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
EOF
```
PVCs provisioned using the above StorageClass will have a replication factor of 3, which means there will be three replicas of the PVC spread across the Kubernetes worker nodes.

Now, let's use the following command to apply this yaml file and deploy the StorageClass on our Kubernetes cluster:
```bash
oc apply -f px-repl3.yaml
```

Step 2 - Deploying Portworx BBQ With a ReadWriteOnce Volume
=====
In this step, we will deploy our Portworx BBQ application that uses MongoDB and an RWO persistent volume.

### Task 1: Create the `pxbbq` namespace
```bash
oc create ns pxbbq
```

### Task 2: Deploy the MongoDB backend in the `pxbbq` namespace
```bash
cat << EOF | oc apply -f -
# Create MongoDB Database for PXBBQ
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
  labels:
    app.kubernetes.io/name: mongo
    app.kubernetes.io/component: backend
  namespace: pxbbq
spec:
  serviceName: "mongo"
  selector:
    matchLabels:
      app.kubernetes.io/name: mongo
      app.kubernetes.io/component: backend
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: mongo
        app.kubernetes.io/component: backend
    spec:
      containers:
      - name: mongo
        image: mongo:7.0.9
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: porxie
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: "porxie"
        args:
        - "--bind_ip"
        - "0.0.0.0"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-data-dir
          mountPath: /data/db
        livenessProbe:
          exec:
            command: ["mongosh", "--eval", "db.adminCommand({ping: 1})"]
          initialDelaySeconds: 30  # Give MongoDB time to start before the first check
          timeoutSeconds: 5
          periodSeconds: 10  # How often to perform the probe
          failureThreshold: 3 
      tolerations:
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 10
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 10
      terminationGracePeriodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: mongo-data-dir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: px-csi-db
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    app.kubernetes.io/name: mongo
    app.kubernetes.io/component: backend
  namespace: pxbbq
spec:
  ports:
  - port: 27017
    targetPort: 27017
  type: ClusterIP
  selector:
    app.kubernetes.io/name: mongo
    app.kubernetes.io/component: backend
EOF
```

### Task 3: Deploy the PXBBQ frontend in the `pxbbq` namespace
```bash
cat << EOF | oc apply -f -
---
apiVersion: apps/v1
kind: Deployment                 
metadata:
  name: pxbbq-web  
  namespace: pxbbq         
spec:
  replicas: 3                    
  selector:
    matchLabels:
      app: pxbbq-web
  template:                      
    metadata:
      labels:                    
        app: pxbbq-web
    spec:                        
      containers:
      - name: pxbbq-web
        image: eshanks16/pxbbq:v4.3.1
        env:
        - name: MONGO_INIT_USER
          value: "porxie" #Mongo User with permissions to create additional databases and users. Typically "porxie" or "pds"
        - name: MONGO_INIT_PASS
          value: "porxie" #Required to connect the init user to the database. If using the mongodb yaml supplied, use "porxie"
        - name: MONGO_NODES
          value: "mongo" #COMMA SEPARATED LIST OF MONGO ENDPOINTS. Example: mongo1.dns.name,mongo2.dns.name
        - name: MONGO_PORT
          value: "27017" # MongoDB Port
        - name: MONGO_USER
          value: porxie #Mongo DB User that will be created by using the Init_User
        - name: MONGO_PASS
          value: "porxie" #Mongo DB Password for User that will be created by using the Init User
          ########## CHATBOT SECTION #############
        - name: CHATBOT_ENABLED #If CHATBOT is set to False, the other variables in this section are not needed.
          value: "False" #Set to True to enable a LLAMA3 chatbot - Requires the AIDemo to be deployed first
        - name: PXBBQ_URI
          value: "http://EXTERNAL_PXBBQ_URL_GOES_HERE" #MUST Be the external svc name for the PXBBQ application (PXBBQ NodePort/LoadBalaner)
        - name: MODEL_SERVER_URI
          value: "http://ollama.genai.svc.cluster.local:11434" #MUST be the internal svc name for the ollama service (CLUSERIP)
        - name: NEO4J_URI
          value: "bolt://database.genai.svc.cluster.local:7687" #MUST be the internal svc name for the new4j service (CLUSTERIP)
        ############# CI/CD Demo Section ##############
        - name: ARCHIVE_ORDERS
          value: "False" #USED FOR CI/CD Database testing demos. Setting this to TRUE Wipes out all previous orders
        imagePullPolicy: Always
        ports:
          - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /healthz # Health check built into PXBBQ
            port: 8080
          #initialDelaySeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /healthz # Health check built into PXBBQ
            port: 8080
          initialDelaySeconds: 15 
          timeoutSeconds: 3  
          periodSeconds: 10  
          failureThreshold: 1 
      tolerations:
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 10
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 10
      terminationGracePeriodSeconds: 0
---
apiVersion: v1
kind: Service
metadata:
  name: pxbbq-svc
  namespace: pxbbq
  labels:
    app: pxbbq-web
spec:
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30000
  type: NodePort
  selector:
    app: pxbbq-web
EOF
```

We now need to expose our service using an Openshift route:

```bash
oc expose svc -n pxbbq pxbbq-svc
```

Lastly, a few of our labs will use the `pxctl` command line tool, ensure that our alias is set up by running the following:

```bash
alias pxctl="oc -n portworx exec $(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}') -c portworx -it -- /opt/pwx/bin/pxctl"
    echo 'alias pxctl="oc -n portworx exec $(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}') -c portworx -it -- /opt/pwx/bin/pxctl"' >> ~/.bashrc
```

### Task 4: Monitor the application deployment using the following command:
```bash
watch oc get all -n pxbbq
```
When all of the pods are running with a `1/1 Ready` state, press `CTRL+C` to exit.

### Task 5: Order up some BBQ!:
Click on the `PX-BBQ` tab next to the `Terminal` tab at the top of the screen. Since you just deployed the application, you may see an Nginx error - if you do, click on the refresh icon next to the words "PX-BBQ" on the tab.

Click the "Menu" icon, and select "Login". Enter the username `guest@portworx.com` and the password `guest`. Next, click on "Menu" again and select "Order". Select a main dish, two sides, and a drink - then click on the "Place Order" button. The order history page appears - click on the hyperlink to your order number, and make sure your order is correct!

### Task 6: Inspect the MongoDB volume
Switch back to the `Terminal` tab and use the following command to inspect the MongoDB volume and look at the Portworx parameters coalias pxctl="oc -n portworx exec $(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}') -c portworx -it -- /opt/pwx/bin/pxctl"
    echo 'alias pxctl="oc -n portworx exec $(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}') -c portworx -it -- /opt/pwx/bin/pxctl"' >> ~/.bashrcnfigured for the volume:
```bash
VOL=`oc get pvc -n pxbbq | grep  mongo-data-dir-mongo-0 | awk '{print $3}'`
pxctl volume inspect $VOL
```
You can see that the `HA` parameter is set to `3` - this means we have three replicas of the persistent volume, as we declared in the StorageClass that we created.

You can also notice that these three replicas are spread out across our three worker nodes as shown in the `Replica sets on nodes` section of the pxctl output, and that all three replicas are in sync by observing the `Replication Status` output which shows `Up`.

Finally, you can see in the `Volume consumers` section that our `mongo-0` pod is the consumer of the volume, which shows that MongoDB is using our volume to store its data!

### Task 7: View your order from the MongoDB CLI
To look at the MongoDB entry generated by your order, first get a bash shell on the MondoDB pod:
```bash
oc exec -it mongo-0 -n pxbbq -- bash
```

Then, let's connect to MongoDB using mongosh:
```bash
mongosh -u porxie -p porxie
```

And finally, let's query for the order you placed earlier:
```bash
use pxbbq
db.orders.find()
exit
```
You should see your order in the database! Finally, let's exit from our exec into the MongoDB pod:
```bash
exit
```

In this step, you saw how Portworx can dynamically provision a highly available ReadWriteOnce persistent volume for your application, and you got to order up some delicious BBQ from Portworx!

Step 3 - Proving Availability for Portworx Volumes
=====
Great, we've got our Portworx BBQ application up and running on highly available PVs, but what happens when the worker node hosting the active replica dies unexpectedly? How will our application react?

First, let's observe where the MongoDB pod is running (note the ***NODE*** column):
```bash
oc get pod mongo-0 -n pxbbq -o wide
```

Next, let's get the worker node running the MongoDB pod into a variable:
```bash
NODENAME=$(oc get pod mongo-0 --no-headers=true -n pxbbq -o wide | awk '{print $7}')
```

And finally, reboot the Kubernetes worker node hosting the running MongoDB pod:
```bash
oc debug node/$NODENAME
```


```bash
chroot /host
```

```bash
systemctl reboot
```

We can watch our MongoDB pod get deleted and recreated. Note the ***NODE*** column as the pod gets evicted and recreated; the node name will change and the pod will be rescheduled on a node in a different AZ:
```bash
watch oc get pod mongo-0 -n pxbbq -o wide
```
> [!IMPORTANT]
> It takes about 20-30 seconds for Kubernetes to detect the worker node has unexpectedly gone offline, and another 10 seconds for the MongoDB pod to get evicted and rescheduled on a surviving node. You will see the MongoDB pod disappear from the watch command, and shortly thereafter will see a new MongoDB pod appear.
>
> The beauty of this is that Kubernetes is aware of where surviving replicas of the MongoDB volume are thanks to STORK (Storage Orchestration Runtime for Kubernetes), and the replacement MongoDB pod is rescheduled on a node that has a surviving replica.

Press `CTRL-C` to exit the watch command once you see the MongoDB pod has successfully restarted.

Now that we have a fresh copy of our MongoDB pod running, let's check the information in our MongoDB collection again to make sure your order is still there!

Again, exec into the freshly created MongoDB pod:
```bash
oc exec -it mongo-0 -n pxbbq -- bash
```

Then connect to MongoDB using mongosh:
```bash
mongosh -u porxie -p porxie
```

And finally, query for the order you placed earlier:
```bash
use pxbbq
db.orders.find()
exit
```
Finally, let's exit from our exec into the MongoDB pod:
```bash
exit
```

To finish making sure our application is healthy, click on the `PX-BBQ` tab, and click the refresh icon next to the "PX-BBQ" tab name. Use the menu to nagivate to "Order History". You can see that your order for Portworx BBQ is still there!

> [!IMPORTANT]
> Since this is a lab environment, we haven't been able to use active health checks for Portworx BBQ since that requires NGINX Plus instead of the Open Source version, and your lab environment may not refresh the web frontend as fast as we'd like.
>
> If your app is not responsive, simply copy the following commands into the terminal to redeploy the web frontend, refresh the application in the `PX-BBQ` tab, and you should be all set to check your order via the UI!
>
> ```bash
> oc delete -f pxbbq-web.yaml -n pxbbq
> oc apply -f pxbbq-web.yaml -n pxbbq
> ```

Step 4 - Cleanup
===
Run the following script to make sure our cluster is healthy and to cleanup demo resources:
```bash
./cleanup.sh
```

