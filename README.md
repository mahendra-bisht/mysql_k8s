## Detailed explanation of the YAML files:

## **mysql-pv.yaml**

This YAML defines two Kubernetes objects:

    A) PersistentVolume (PV)
    B) PersistentVolumeClaim (PVC)

## 1. Persistent Volume (PersistentVolume)
    A PersistentVolume (PV) is a piece of storage in the cluster that is provisioned manually or dynamically. This is a physical storage unit that remains even if pods are deleted.

Let's break down the PersistentVolume section:

**Metdata** :-

metadata:
  name: mysql-pv
  namespace: database

name: mysql-pv → Names the Persistent Volume mysql-pv.
namespace: database → Ensures that this volume exists in the database namespace.

**Specification(spec)** :-

spec:
  capacity:
    storage: 5Gi

capacity: → Defines the size of the storage.
storage: 5Gi → Allocates 5 Gigabytes of storage.

accessModes:
  - ReadWriteOnce

accessModes: → Defines how the volume can be accessed by pods.
ReadWriteOnce: → This volume can be mounted as read/write by a single node. (No multiple pods on different nodes can share this storage.)

hostPath:
  path: "/mnt/data"

hostPath: → Uses a directory on the node’s filesystem for storage.
path: "/mnt/data" → The data will be stored in /mnt/data on the node.
Note: This is a non-cloud, local storage method. In cloud environments, a better alternative would be network-based storage like AWS EBS, NFS, or Ceph.

## 2. Persistent Volume Claim (PersistentVolumeClaim)
    A PersistentVolumeClaim (PVC) is a request for storage by a pod. This claim allows a pod to dynamically request a PersistentVolume (PV) that matches its requirements.

**Metdata** :-

metadata:
  name: mysql-pvc
  namespace: database

name: mysql-pvc → Names the Persistent Volume Claim as mysql-pvc.
namespace: database → Ensures this PVC exists in the database namespace.

**Specification(spec)** :-

spec:
  accessModes:
    - ReadWriteOnce

accessModes: → Defines how the storage can be accessed (same as PV).
ReadWriteOnce: → The storage can be mounted as read/write by a single node.

resources:
  requests:
    storage: 5Gi

resources: → Specifies the requested resources.
requests: → Defines the minimum required storage.
storage: 5Gi → Requests 5 Gigabytes of storage.

## How This Works in Kubernetes

  1) The PersistentVolume (PV) is created as a 5Gi local storage in /mnt/data.
  2) The PersistentVolumeClaim (PVC) is created and requests 5Gi of storage.
  3) Kubernetes checks if there is a PV that matches the PVC request.
  4) If a matching PersistentVolume (mysql-pv) is found
      The PersistentVolumeClaim (mysql-pvc) gets bound to it.
      The pod using this PVC can now store MySQL database data persistently.
  5) Even if the pod restarts, the data remains intact in /mnt/data.

Key Takeaways:
  1) PV (PersistentVolume) = Physical storage (allocated manually or dynamically).
  2) PVC (PersistentVolumeClaim) = A request from a pod to use storage.
  3) hostPath: Local storage (good for testing, but not production-ready).
  4) ReadWriteOnce: Allows only one node to use the storage at a time.
  5) 5Gi: Defines storage capacity.

===================================================================================

## **mysql-deployment.yaml**

**API Version & Kind**

apiVersion: apps/v1
kind: Deployment

apiVersion: apps/v1 → Specifies that this is a Deployment, managed by the apps/v1 API.
kind: Deployment → Defines the resource type as a Deployment.

**Metadata**

metadata:
  name: mysql
  namespace: database

name: mysql → Names the Deployment mysql.
namespace: database → Places this Deployment inside the database namespace.

**Spec(Main Configuration)**

spec:
  replicas: 1

replicas: 1 → Ensures only one MySQL pod is running. If the pod crashes, Kubernetes will automatically restart it.

selector:
  matchLabels:
    app: mysql

selector: → Specifies how to select pods managed by this deployment.
matchLabels: → Looks for pods labeled app: mysql.

**Pod Template(Defining the MySQL Pod)**

template:
  metadata:
    labels:
      app: mysql

template: → Defines the pod that will be created.
labels: app: mysql → Ensures pods get the label app=mysql (used by Services to find this pod).

**Defining the Container**

spec:
  containers:
    - name: mysql
      image: mysql:5.7

name: mysql → Names the container inside the pod.
image: mysql:5.7 → Uses the official MySQL v5.7 Docker image.

env:
- name: MYSQL_ROOT_PASSWORD
  value: "rootpassword"
- name: MYSQL_DATABASE
  value: "mydatabase"
- name: MYSQL_USER
  value: "user"
- name: MYSQL_PASSWORD
  value: "password"

env: → Environment variables to configure MySQL.
MYSQL_ROOT_PASSWORD → Sets the root password (rootpassword).
MYSQL_DATABASE → Creates a database named mydatabase.
MYSQL_USER → Creates a user user.
MYSQL_PASSWORD → Sets the user's password (password).

Please note that we need only mysql_root_password env. variable(mandatory) whereas others are optional.

**Defining Ports**

ports:
  - containerPort: 3306

containerPort: 3306 → Exposes MySQL’s default port (3306) inside the pod.

**Mounting Persistent Storage**

volumeMounts:
- name: mysql-storage
  mountPath: /var/lib/mysql

volumeMounts: → Attaches a Persistent Volume to the MySQL pod.
mountPath: /var/lib/mysql → MySQL stores data in this directory, preventing data loss on pod restarts.

volumes:
- name: mysql-storage
  persistentVolumeClaim:
    claimName: mysql-pvc

volumes: → Uses the Persistent Volume Claim (mysql-pvc) to mount storage.
This ensures MySQL data is stored persistently even if the pod is restarted.

How It Works
✅ Deploys one MySQL pod.
✅ Uses a Persistent Volume Claim to store MySQL data.
✅ Exposes port 3306 for communication.
✅ Creates a MySQL database and user from environment variables.
✅ If MySQL crashes, Kubernetes restarts it automatically.

Bottom Line : Creates a MySQL pod with persistent storage.

===================================================================================

## mysql-service.yaml

***** A Service allows other pods or external users to connect to MySQL. *****

**API Version & Kind**

apiVersion: v1
kind: Service

apiVersion: v1 → Uses the core Kubernetes API.
kind: Service → Defines this resource as a Service.

**Metadata**

metadata:
  name: mysql
  namespace: database

name: mysql → The Service will be called mysql.
namespace: database → Places the Service in the database namespace.

**Spec(Service Configuration)**

spec:
  ports:
    - port: 3306
      targetPort: 3306

ports: → Defines how traffic is forwarded.
port: 3306 → Clients connect to this Service on port 3306.
targetPort: 3306 → Routes traffic to MySQL containers' port 3306.

selector:
  app: mysql

selector: app: mysql → Connects to pods labeled app=mysql.

clusterIP: None

Why None?
  This makes it a Headless Service.
  DNS-based discovery is used instead of a fixed IP.
  Useful for databases because MySQL clients discover the pod dynamically.

How It Works
✅ Creates a stable internal connection for MySQL.
✅ Routes traffic to MySQL pods using selector: app=mysql.
✅ Allows other pods to connect using mysql.database.svc.cluster.local.

Bottom Line : Exposes MySQL inside the cluster so other apps can connect.

===================================================================================

## mysql-configmap.yaml

***** This ConfigMap contains configuration files for MySQL instances, defining different settings for the primary (master) and replicas (slaves).*****

**primary.cnf**
This configuration is applied to the primary pod (i.e., mysql-0). Let's break down the directives:

[mysqld] : Section header specifying that these settings apply to the MySQL server (daemon).

server-id=1	: Unique ID for this MySQL server (Master here). 1 is chosen as it's the primary node. Server IDs must be unique within a replication setup.

log-bin=mysql-bin : Enables binary logging to track changes made to the database. Required for replication as replicas rely on these logs to synchronize.

binlog-format=ROW : Sets binary log format to ROW-based logging. Logs actual row changes rather than just the SQL statements. Essential for replication.

max_connections=200 : Limits the maximum number of concurrent connections to 200. Helps prevent overloading the database.

bind-address=0.0.0.0 : Makes the MySQL server listen on all network interfaces. Necessary when pods need to communicate with each other across the cluster.

🔍 How Master Replication Works
Binary Logging: The master records all changes to the database in mysql-bin log files.
Replicas read these logs and apply the same changes to their databases to stay in sync.

**replica.cnf**
This configuration applies to all replica pods (i.e., mysql-1, mysql-2, etc.).

[mysqld] : Section header specifying that these settings apply to the MySQL server (daemon).

server-id=${MYSQL_SERVER_ID} : Placeholder for unique server ID. I dynamically assign this using init-mysql. Each replica must have a unique server-id.

read_only=1 : Sets MySQL to read-only mode. Prevents replicas from accepting writes unless performed by the replication process.

relay-log=mysql-relay-bin : Configures the relay log which stores changes received from the master before applying them. Crucial for replication.

log-bin=mysql-bin : Replicas also generate binary logs if they later become masters. Useful in cascading replication setups.

binlog-format=ROW : Sets binary log format to ROW-based logging. Logs actual row changes rather than just the SQL statements. Essential for replication.

bind-address=0.0.0.0 : Makes the MySQL server listen on all network interfaces. Necessary when pods need to communicate with each other across the cluster.

⚙️ How Replica Replication Works
Connect to the Master: Replicas connect to the master and request binary logs.
Receive and Store Changes: These logs are written into the relay log (mysql-relay-bin).
Apply Changes: MySQL applies the logged changes to the replica's database.
Stay in Sync: The process continues indefinitely unless stopped or interrupted.

===================================================================================

## mysql-statefulset.yaml

***** This StatefulSet is responsible for deploying a MySQL replication cluster with one master and two replicas using Kubernetes' StatefulSet resource. *****

**Why StatefulSet?**
Unlike Deployments, StatefulSets provide:

> Stable, unique network identities for each pod.
> Persistent storage using PersistentVolumeClaims (PVCs) to maintain state across pod restarts.
> Ordered and deterministic deployment with predictable pod names like mysql-0, mysql-1, etc.

**Complete Architecture Overview**
> MySQL Primary Pod: mysql-0 (Master)
> MySQL Replica Pods: mysql-1, mysql-2 (Slaves/Replicas)
> Replication: Asynchronous MySQL replication based on binary logs.

**🛠️ Step-by-Step Breakdown**

📍 1. StatefulSet Metadata
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: database

apiVersion : Version of the Kubernetes API used for StatefulSets (apps/v1).
kind : This resource is a StatefulSet.
metadata.name : Assigns the name mysql to this StatefulSet. Each pod will have names like mysql-0, mysql-1, etc.
metadata.namespace : The StatefulSet runs in the database namespace, ensuring logical separation from other workloads.

📍 2. StatefulSet Spec Configuration
spec:
  serviceName: mysql
  replicas: 3

serviceName : The name of the headless service (mysql) to manage network identities for MySQL pods.
replicas : Deploy 3 pods: one master (mysql-0) and two replicas (mysql-1, mysql-2).

🔹 Headless Service is crucial here because MySQL replicas need to discover and connect to the master by hostname.

📍 3. Pod Selector
  selector:
    matchLabels:
      app: mysql

🛠️ Explanation
The StatefulSet will manage pods with the label app: mysql.
Ensures Kubernetes controllers know which pods belong to this StatefulSet.

📍 4. Pod Template
  template:
    metadata:
      labels:
        app: mysql

🛠️ Explanation
All pods created by this StatefulSet will have the app: mysql label.
This label helps with service discovery and traffic routing.

🚀 Pod Specification

Now let's dive into the heart of the configuration: the initContainers and containers sections.

🛠️ 5. Init Container: init-mysql

Purpose:
Dynamically configure my.cnf files for master and replicas.

initContainers:
- name: init-mysql
  image: busybox
  command:
    - sh
    - "-c"
    - |
      # Extract pod index from hostname
      POD_INDEX=$(hostname | awk -F'-' '{print $NF}')
      SERVER_ID=$((POD_INDEX + 1))

      # Generate my.cnf dynamically based on server ID
      if [ "$POD_INDEX" -eq 0 ]; then
          cp /config/primary.cnf /etc/mysql/my.cnf
      else
          echo "[mysqld]" > /etc/mysql/my.cnf
          echo "server-id=$SERVER_ID" >> /etc/mysql/my.cnf
          echo "read_only=1" >> /etc/mysql/my.cnf
          echo "relay-log=mysql-relay-bin" >> /etc/mysql/my.cnf
          echo "log-bin=mysql-bin" >> /etc/mysql/my.cnf
          echo "binlog-format=ROW" >> /etc/mysql/my.cnf
          echo "bind-address=0.0.0.0" >> /etc/mysql/my.cnf
      fi

⚙️ Breaking Down the Init Container

1. Pod Index Extraction:
POD_INDEX=$(hostname | awk -F'-' '{print $NF}')

Extracts the pod's index from its hostname.
mysql-0 → 0, mysql-1 → 1, mysql-2 → 2.

2. Server ID Calculation:
SERVER_ID=$((POD_INDEX + 1))

Each pod gets a unique server-id (e.g., pod mysql-1 → server-id=2).

3. Config Generation:

Master Pod (mysql-0): Copies primary.cnf from the ConfigMap.
Replica Pods: Dynamically generate my.cnf with unique server-id, read_only=1, and relay-log.

4. Why We Use initContainers:

Ensures MySQL configuration files are correctly set before the main container starts.
Runs only once during pod initialization.

5. Volume Mounts in Init Container
volumeMounts:
  - name: config-volume
    mountPath: /config
  - name: mysql-config
    mountPath: /etc/mysql

config-volume: Access to the ConfigMap for primary.cnf.
mysql-config: Writes the dynamically generated my.cnf to /etc/mysql.

6. MySQL Container
containers:
- name: mysql
  image: percona:5.7
  env:
    - name: MYSQL_ROOT_PASSWORD
      value: "xxxx"

Explanation
Container Name: mysql — main MySQL database container.
Image: percona:5.7, a MySQL-compatible image optimized for performance.
Environment Variables: Sets the MySQL root password (consider using Secrets in production).

📦 Volume Mounts in MySQL Container

volumeMounts:
  - name: mysql-config
    mountPath: /etc/mysql/my.cnf
    subPath: my.cnf

Explanation
Mount Path: /etc/mysql/my.cnf — MySQL expects its main config file here.
SubPath: Ensures only my.cnf from the mysql-config volume is mounted, leaving other potential files untouched.

7. Volumes

volumes:
  - name: config-volume
    configMap:
      name: mysql-config
  - name: mysql-config
    emptyDir: {}

Explanation
> config-volume:
Maps the mysql-config ConfigMap, providing primary.cnf.
 
> mysql-config:
An emptyDir volume where the init-mysql container writes the dynamically generated my.cnf file.

⚙️ How It All Works
> Pod Initialization:
Pods start sequentially: mysql-0, mysql-1, mysql-2.

> Configuration Generation:
init-mysql extracts the pod index and sets server-id accordingly.
mysql-0 → Master config from primary.cnf.
mysql-1/mysql-2 → Replica configs with read_only=1.

> MySQL Startup:
Once init-mysql completes, the mysql container starts with the appropriate configuration.

> Replication Begins:
Replicas connect to the master via the headless service (mysql.database.svc.cluster.local).
Replicas read binary logs and apply changes to their databases.

🔍 Operational Insights
🧩 Troubleshooting Commands

**Check Pod Status:**
kubectl get pods -n database -w

**Check MySQL Logs:**
kubectl logs mysql-0 -n database

**Inspect Configuration:**
kubectl exec -it mysql-0 -n database -- cat /etc/mysql/my.cnf

⚠️ Common Pitfalls
> Replication Lag:
    Monitor with SHOW SLAVE STATUS\G.
    Check network latency and I/O performance.
> Password Management:
    Never hardcode passwords in yaml. Use Secrets.
> Storage Persistence:
    Ensure appropriate PersistentVolumeClaims are defined to persist data across pod restarts.
