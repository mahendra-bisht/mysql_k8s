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
