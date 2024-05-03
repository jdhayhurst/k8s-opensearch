# k8s-opensearch
Opensearch deployment with the k8s operator

## Requirements
1. Kubernetes cluster with the [opensearch operator installed](https://github.com/opensearch-project/opensearch-k8s-operator/blob/main/docs/userguide/main.md).
2. _If using the GCS snapshot repository_ create kubernetes secret with the GCP credentials 
   `kubectl -n <NAMESPACE> create secret generic gcp-credentials --from-file=gcs.client.default.credentials_file=<GCS_CREDENTIALS_JSON>`


### For a simple 3 node cluster:
- `kubectl apply -f simple/os-cluster.yaml` - change namespace as needed with the `namespace` property in the yaml. 
- Access the cluster: `curl -k -u admin:admin -X GET 'https://opensearch:9200'`

### For a blue/green deployment:
1. Deploy "blue" version: `kubectl apply -f blue-green/os-cluster-blue.yaml`
2. Deploy another service to act like a proxy: `kubectl apply -f blue-green/svc.yaml` - this serves the blue cluster and is the service that other microservices will connect to i.e. access the cluster with `curl -k -u admin:admin -X GET 'https://opensearch:9200'`
3. Deploy "green" version: `kubectl apply -f blue-green/os-cluster-green.yaml`
4. When you want to switch to the "green" version, change the `selector` in the proxy service to `opster.io/opensearch-cluster: opensearch-green` - `https://opensearch:9200` will now point to green instead of blue.

### Data release
#### One idea for releasing data is:
a. assume you're starting by serving data from blue and want to release new data
b. load data directly into the green cluster
c. switch the service to point to green

#### An other approach is to use the [snapshot and restore](https://opensearch.org/docs/latest/tuning-your-cluster/availability-and-recovery/snapshots/snapshot-restore/) feature e.g.:
a. assume you're starting by serving data from blue and want to release new data
b. load the data (on an opensearch cluster somewhere) and take a snapshot (you need to register the snapshot repo) with snapshot api call
c. use the restor api to restore the data on "green" cluster
d. switch the service to point to green

### Example snapshot and restore API calls:
1. Register a snapshot repo on GCS
```
curl -X PUT 'https://localhost:9200/_snapshot/gcs_repository' \
-H 'Content-Type: application/json' \
-k -u admin:admin \
-d '{
  "type": "gcs",
  "settings": {
    "bucket": "open-targets-pre-data-releases",
    "base_path": "snapshots/platform",
    "client": "default"
  }
}'
```

2. Take a snapshot
```
curl --location --request PUT 'http://localhost:9200/_snapshot/gcs_repository/release_24-06' \
--header 'Content-Type: application/json' 
-k -u admin:admin \
--data '{
  "ignore_unavailable": true,
  "include_global_state": false,
  "partial": false
}'
```

3. Restore from a snapshot (it's important to exclude the ".*" indices)
```
curl -k -u admin:admin -X POST 'https://localhost:9200/_snapshot/gcs_repository/release_24-06/_restore' \
-H 'Content-Type: application/json' \
-d '{"indices": "-.*"}'
```