# Deploying Search & Vector Search

## Prerequisites
If you've followed [these steps](https://github.com/vinilage/mck-om/tree/main) you are good to go!
- [A local Kubernetes cluster](https://github.com/vinilage/mck-om/tree/main)
- [OpsManager deployed](https://github.com/vinilage/mck-om/tree/main)
- [A replica-set deployed](https://github.com/vinilage/mck-om/blob/main/replica-set/README.md)
- [SCRUM authentication enabled and database users created](https://github.com/vinilage/mck-om/blob/main/user/README.md)
- [TLS is enabled](https://github.com/vinilage/mck-om/blob/main/tls/README.md)

## Import sample data to the local database

Deploy a `MongoDB Tools Pod` to support importing the data.
Go to `./search/` and run:

```
kubectl apply -f mongodb-tools.yaml
```

Connect to the newly created `mongodb-tools-pod` to execute commands via Shell.
You can use `K9s` by clicking in the `mongodb-tools-pod` and then `s` to open Shell.

Or you can execute:

```
kubectl exec -n mongodb-operator -it mongodb-tools-pod -- sh
```

Download the sample data from the internet to the `tmp` folder inside of the pod:

```
curl -fSL https://atlas-education.s3.amazonaws.com/sample_mflix.archive -o /tmp/sample_mflix.archive
```

Copy the local certificate `tls/ca.pem` to the `tmp` folder:

```
kubectl cp <your directory>/mck-om/tls/ca.pem \
  mongodb-operator/mongodb-tools-pod:/tmp/ca.pem \
  --context k3d-mongodb-mck-cluster
```

Import sample data to the local database (replica-set) with the TLS certificate:

```
mongorestore \
  --uri="mongodb://mdb-admin:12345678@replica-set-svc.mongodb-operator.svc.cluster.local:27017/?replicaSet=replica-set&authSource=admin&tls=true&tlsCAFile=/tmp/ca.pem" \
  --archive=/tmp/sample_mflix.archive \
  --nsInclude 'sample_mflix.*'
```

### Verify if the `sample_mflix` database is imported

To connect to the replica set from outside the Kubernetes cluster, we need to forward the port of the replicaset service:
```
kubectl port-forward -n mongodb-operator svc/replica-set-svc 27017:27017
```

Connect with Compass with the following connection string (remember to use the TLS `ca.pem` certificate):
```
mongodb://mdb-admin:12345678@localhost:27017/admin?authSource=admin&directConnection=true
```

![Alt text](/img/Compass-sample-data.png)

### Deploy Search

When deploying Search with MCK, it will automatically enable Search and Vector Search in the database.  

#### We will deploy:
- 3 Search members (for high-availability)
- 1 Envoy LoadBalancer to balance the traffic between the Search pods
- TLS certificates (Server and Client) for the LoadBalancer 

For this, you need to run:  

```
kubectl apply -f search-tls.yaml
```

### Create a Search index

Connect to the `mongodb-tools-pod`:
```
kubectl exec -n mongodb-operator -it mongodb-tools-pod -- sh
```

Connect to the primary member of the MongoDB Database (to be able to `write`) and with `admin` user (using TLS):
```
mongosh \
  --username mdb-admin \
  --password 12345678 \
  --authenticationDatabase admin \
  --tls \
  --tlsCAFile /tmp/ca.pem \
  "mongodb://replica-set-0.replica-set-svc.mongodb-operator.svc.cluster.local:27017,replica-set-1.replica-set-svc.mongodb-operator.svc.cluster.local:27017,replica-set-2.replica-set-svc.mongodb-operator.svc.cluster.local:27017/?replicaSet=replica-set"
```

Prepare to create the search index inside of the `sample_mflix` database:
```
use sample_mflix
```

Create a `Vector Search` index in the `embedded_movies` collection:
```
db.embedded_movies.createSearchIndex("vector_index", "vectorSearch",
    { "fields": [ {
      "type": "vector",
      "path": "plot_embedding_voyage_3_large",
      "numDimensions": 2048,
      "similarity":
      "dotProduct",
      "quantization": "scalar"
    } ] });
```

Create a `Search` index in the `movies` collection:
```
db.movies.createSearchIndex("default", { mappings: { dynamic: true } });
```

## Verify the indexes

You can verify if the indexes were created for Search:
```
db.runCommand({"listSearchIndexes": "movies"});
````

or Vector Search:
```
db.runCommand({"listSearchIndexes": "embedded_movies"});
```

## Verify with Compass
With `Compass` you can also see the search indexes `if you are connected to the primary member`.
You can check if you are connected to the primary by opening the MongoDB Shell in Compass.
You should see `Enterprise replica-set [direct: primary] admin >`.

If you see `secondary`, forward the port directly to the primary (check in OM what is your primary).
Example would be:

```
kubectl port-forward pod/replica-set-0 27017:27017 -n mongodb-operator
```

The indexes should be visible in Compass:

![Alt text](/img/compass-search-index.png)


## Querying

You can now query the data.
See the [documentation](https://www.mongodb.com/docs/kubernetes/current/tutorial/fts-vs-quickstart/#query-the-data-using-the-index.) and try out different Search and Vector Search queries.

An example of a result for Vector Search using Compass:
![Alt text](/img/compass-search-result.png)


## Enable Auto Embeddings (optional)

### Create the API keys
Create the API Keys in Atlas or in the VoyageAI.
Add your keys to the Secret in the `auto-embeddings.yaml` file, fields `query-key` and `indexing-key`.

### Enable auto-embeddings
Run:

```
kubectl apply -f auto-embeddings.yaml
```

## Deploy Dedicated Search Nodes (OPTIONAL)

By default, `search.yaml` deploys one `mongot` pod (`replica-set-search-0`) on the same K3d node as the database.
These steps add dedicated K3d nodes for Search, run 2 `mongot` pods on them, and add an operator-managed Envoy load balancer (`replica-set-search-lb-0-*`) that routes the search queries from `mongod` to the `mongot` pods.

> Each `mongot` pod requests 2 CPU and 3Gi of memory. Make sure Docker (or Podman) has enough resources for 2 `mongot` pods in addition to Ops Manager and the replica set.

### 1. Create the search nodes
Add 2 agent nodes to the K3d cluster:
```
k3d node create search-node --cluster mongodb-mck-cluster --role agent --replicas 2
```
Verify that `k3d-search-node-0` and `k3d-search-node-1` are `Ready`:
```
kubectl get nodes
```

### 2. Label and taint the search nodes
The label lets the `mongot` pods target the search nodes. The taint keeps other pods off them:
```
kubectl label node k3d-search-node-0 k3d-search-node-1 workload=search --overwrite
kubectl taint node k3d-search-node-0 k3d-search-node-1 dedicated=search:NoSchedule
```
Verify that only the search nodes have the label and the taint:
```
kubectl get nodes -o custom-columns='NAME:.metadata.name,WORKLOAD:.metadata.labels.workload,TAINTS:.spec.taints[*].key'
```

### 3. Update `search.yaml`
Add `replicas`, `loadBalancer` and `statefulSet` to the `spec`:
```
apiVersion: mongodb.com/v1
kind: MongoDBSearch
metadata:
  name: replica-set # same name as the MongoDB replica-set resource to avoid specifying source.mongodbResourceRef
  namespace: mongodb-operator
spec:
  clusters:
    - replicas: 2
      loadBalancer:
        managed: {}
      resourceRequirements:
        limits:
          cpu: "3"
          memory: 5Gi
        requests:
          cpu: "2"
          memory: 3Gi
      statefulSet:
        spec:
          template:
            spec:
              nodeSelector:
                workload: search
              tolerations:
                - key: dedicated
                  operator: Equal
                  value: search
                  effect: NoSchedule
              affinity:
                podAntiAffinity:
                  requiredDuringSchedulingIgnoredDuringExecution:
                    - labelSelector:
                        matchLabels:
                          app: replica-set-search-svc
                      topologyKey: kubernetes.io/hostname
```
- `replicas`: number of `mongot` pods. When it's greater than `1`, `loadBalancer` is required.
- `loadBalancer.managed`: the operator deploys an Envoy load balancer between `mongod` and the `mongot` pods.
- `nodeSelector` and `tolerations`: place the `mongot` pods only on the search nodes.
- `podAntiAffinity`: places each `mongot` pod on a different search node.

> This layout is for operator version `1.8.0`. Newer operator versions move these fields under `spec.clusters[]`. If `kubectl apply` fails with `unknown field`, check the fields that your operator supports with `kubectl explain mongodbsearch.spec`.

If you enabled auto-embeddings, add the same fields to the `MongoDBSearch` resource in `auto-embeddings.yaml`. Otherwise, applying that file removes the dedicated search node configuration.

### 4. Apply the changes
```
kubectl apply -f search.yaml
```

### 5. Move the existing `mongot` pod to a search node
If `replica-set-search-0` was deployed before the search nodes, it stays `Pending`, because its storage (PVC) was created on the `k3d-mongodb-mck-cluster-server-0` node and K3d's `local-path` storage can't move between nodes.
Check the pod events:
```
kubectl describe pod replica-set-search-0 -n mongodb-operator
```
If the events show `didn't match PersistentVolume's node affinity`, delete the PVC and the pod:
```
kubectl delete pvc data-replica-set-search-0 -n mongodb-operator --wait=false
kubectl delete pod replica-set-search-0 -n mongodb-operator
```
The StatefulSet recreates the pod and its PVC on a search node. The PVC only contains the search indexes, so no data is lost. `mongot` rebuilds the indexes from the replica set, and search queries can return partial results until it finishes.

### 6. Verify
```
kubectl get mongodbsearch -n mongodb-operator
kubectl get pods -n mongodb-operator -o wide | grep search
```
- The `MongoDBSearch` resource shows `PHASE` and `LOADBALANCER` as `Running`.
- `replica-set-search-0` and `replica-set-search-1` are `Running`, each on a different `k3d-search-node-*` node.
- The load balancer pod `replica-set-search-lb-0-*` runs on `k3d-mongodb-mck-cluster-server-0`, because it doesn't tolerate the search node taint.

Wait until the indexes show `status: 'READY'` again (see [Verify the indexes](#verify-the-indexes)), then run the queries in [TEXT_SEARCH_QUERIES.md](TEXT_SEARCH_QUERIES.md) and [VECTOR_SEARCH_QUERIES.md](VECTOR_SEARCH_QUERIES.md).
