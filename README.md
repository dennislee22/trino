# Cloudera Trino

<img width="200" height="251" alt="image" src="https://github.com/user-attachments/assets/397a22f6-a2c4-45cb-8e85-f5309df04a10" />

Trino is a fast, distributed SQL query engine designed for querying massive datasets across heterogeneous sources using standard SQL. It supports large-scale analytics and data engineering by enabling both interactive queries and batch workloads directly, eliminating the need for costly data movement or storage format transformations. This blog focuses on a Kubernetes–based deployment example of Trino, complete with screenshots and structural insights.

## Prerequisites

- Authorization for Trino needs to be configured through Apache Ranger and you can create or update Ranger policies for specific resources and assign permissions to Trino users, groups, or roles. When a user submits a query to Trino, the system verifies the defined policies to ensure that the user has the necessary permissions to run queries.
- Ranger authorization for all the Trino supported connectors is offered through the `cm_trino` resource-based service.

<img width="1000" height="738" alt="image" src="https://github.com/user-attachments/assets/16f22115-0ef7-4819-88f7-10c4ad3abd6b" />

- In this example, user `dennis` is authorized for creating, reading, and writing data. 
<img width="1000" height="642" alt="image" src="https://github.com/user-attachments/assets/bb8237e3-53bc-4eb9-980b-bab8463cd90f" />

## Deployment Procedure
1. You can create a federation connector that can be used by a Trino virtual warehouse to query and access data from different data sources. Federation Connectors are configured and managed within Cloudera Data Warehouse (CDW) for `Trino` virtual warehouse. Users can define connectors to external data sources such as PostgreSQL, MySQL, Hive, and Iceberg, by specifying connection details like JDBC URL, credentials, and connector type, and validating connectivity with a built-in test. 

<img width="700" height="588" alt="image" src="https://github.com/user-attachments/assets/e7bf2b49-e91d-4b3c-95e4-0fbaffd0af6b" />

<img width="700" height="753" alt="image" src="https://github.com/user-attachments/assets/4d42eaea-884f-42c1-94bc-dae77b8d05af" />

2. Once created, the federation connectors are centrally listed and can be reused by Trino virtual warehouses to seamlessly query and federate data across multiple heterogeneous systems, enabling unified analytics without moving or duplicating data.
<img width="700" height="473" alt="image" src="https://github.com/user-attachments/assets/9ed0141f-afd6-4ec2-af0e-37e124f235dd" />

3. Create a `Trino` virtual warehouse with the pre-defined resource profile. You can specify which LDAP group (in this case, `cdw` group) is permitted to access this virtual warehouse.
<img width="1000" height="757" alt="image" src="https://github.com/user-attachments/assets/1b7d4203-7c92-4edc-b6c4-cc62db433447" />

<img width="1000" height="745" alt="image" src="https://github.com/user-attachments/assets/4d58f705-cde7-4112-aa75-3545f8a88d5c" />

Under the hood, the following Kubernetes resources are provisioned for this Trino namespace.

```
# kubectl -n trino-dlee-trino1 get pods
NAME                            READY   STATUS      RESTARTS   AGE
hue-huedb-create-job-nhxcl      0/1     Completed   0          3m43s
huebackend-0                    1/1     Running     0          3m23s
huefrontend-79bdd4bcbf-5p9vb    1/1     Running     0          3m23s
trino-coordinator-0             1/1     Running     0          3m23s
trino-worker-6b9d67b7fc-gnnfn   1/1     Running     0          3m23s
```

A PersistentVolumeClaim is created and bound to the Trino worker to provide local storage for caching and intermediate data during query execution.

```
# kubectl -n trino-dlee-trino1 get pvc
NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
trino-worker-6b9d67b7fc-gnnfn-cache-volume   Bound    pvc-b381327f-0aa7-4a14-98c4-df708737d0a3   1          RWO            local-path     <unset>                 6m1s
```

```
# kubectl get pv pvc-b381327f-0aa7-4a14-98c4-df708737d0a3 -o yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    local.path.provisioner/selected-node: ecs-w-01.dlee155.cldr.example
    pv.kubernetes.io/provisioned-by: rancher.io/local-path
  creationTimestamp: "2026-01-08T04:02:57Z"
  finalizers:
  - kubernetes.io/pv-protection
  name: pvc-b381327f-0aa7-4a14-98c4-df708737d0a3
  resourceVersion: "36716109"
  uid: 505889a8-db44-4479-b757-466f6ab2e8b2
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: "1"
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: trino-worker-6b9d67b7fc-gnnfn-cache-volume
    namespace: trino-dlee-trino1
    resourceVersion: "36715988"
    uid: b381327f-0aa7-4a14-98c4-df708737d0a3
  hostPath:
    path: /ecs/local/pvc-b381327f-0aa7-4a14-98c4-df708737d0a3_trino-dlee-trino1_trino-worker-6b9d67b7fc-gnnfn-cache-volume
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - ecs-w-01.dlee155.cldr.example
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-path
  volumeMode: Filesystem
status:
  lastPhaseTransitionTime: "2026-01-08T04:02:57Z"
  phase: Bound
```

4. Tick the 2 federation connector boxes (`mysqlcde` & `postgrescdp`) that were created earlier and associate them with this Trino virtual warehouse. Click `Apply Changes`.

<img width="1000" height="749" alt="image" src="https://github.com/user-attachments/assets/ee21fa5a-b25e-4a0e-95c2-11c44c83fab4" />

5. Open the Hue SQL editor on this Trino warehouse. Note that both `mysqlcde` and `postgrescdp` databases have successfully been populated. Test the SQL query on `mysqlcde` database using Hue in Trino virtual warehouse.
<img width="1000" height="738" alt="image" src="https://github.com/user-attachments/assets/586d8d66-5c60-4de6-b978-a130677aa533" />

6. Test the SQL query on `mysqlcde` and `postgrescdp` databases using Hue in Trino virtual warehouse.
<img width="1000" height="746" alt="image" src="https://github.com/user-attachments/assets/b7272997-f064-4a08-ad24-64548a921a34" />

