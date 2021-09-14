# Setup Feast Online Feature Store in a K8s Cluster
![Feast_kube](https://github.com/tedhtchang/populate_feast_online_store/blob/main/images/Feast_on_Kube.png)

This notebook example is developed to work with the [Feast KFserving Transformer](https://github.com/kubeflow/kfserving/tree/master/docs/samples/v1beta1/transformer/feast) to populate a Feast Online store and run Feast Online Serving REST API server in a Kubernetes Cluster although Feast Online Feature Store can be run anywhere.

This demo is tested with Feast v0.12.1 relase.

Note: This notebook can be run inside a Kubercluster or locally.

## Requirement:
- Feast Online Feature Server - Provides REST API to the Online Store. Check [here](https://github.com/feast-dev/feast/pull/1780) for detail.
- Online Store - Redis
    - Available [options](https://docs.feast.dev/reference/online-stores)
- Kubernetes Cluster - We choose to deploy our Feast Online Feature Store in a K8s Cluster. You may setup it up anywhere as long as the Store is accessible by network address.

## Deploy:
1. Clone this repo:
    ```bash
    git clone https://github.com/tedhtchang/populate_feast_online_store.git
    cd populate_feast_online_store
    ```
1. Deploy Redis server a service and [test connectivity](https://github.com/tedhtchang/populate_feast_online_store#test-connection-to-redis-service-in-the-k8s-cluster):
    ```bash
    kubectl apply -f config/redis.yaml
    ```

1. Deploy Feature Server as a service that is initialized with the pre-configured feature [repository](https://github.com/tedhtchang/driver_rank_repo) :

    ```bash
    kubectl apply -f config/feature_server.yaml
    ```
    Starts a Feature Server that is initialized with the pre-configured feature  used in this demo.

# Populate Feast online store

Running this notebook [example](Quickstart.ipynb) populates the Feast online store using the provided [data](https://github.com/tedhtchang/driver_rank_repo/blob/master/data/driver_stats.parquet).

Load the Quickstart example into a notebook:

```
jupyter notebook Quickstart.ipynb
```
Install a classic notebook if you don't have one:
```bash
pip install notebook
```

# Additional Configuration
## S3 storage
If you wish to configure Feast SDK to use S3 or S3 compatible object store for storing driver_stats.parquet or registry.db. The following env variables need to exported:
```
# Read by boto3 library
export AWS_ACCESS_KEY_ID=<Your S3 access key id>
export AWS_SECRET_ACCESS_KEY=<Your S3 secret key>

# If you are using your own S3 compatible object store you need this env too
export FEAST_S3_ENDPOINT_URL=<your custom S3 store endpoint>
```
Example feature definition and repo config that use a custom S3 endpoint (typically none aws S3 compatible object store):
```
from datetime import timedelta
from feast import FileSource, Entity, Feature, FeatureView, ValueType
driver = Entity(name="driver_id", join_key="driver_id", value_type=ValueType.INT64,)

driver_stats_source = FileSource(
    path="s3://driver_rank_repo/driver_stats.parquet"
    s3_endpoint_override="<your custom S3 store endpoint>"
)

driver_stats_fv = FeatureView(
    name="driver_hourly_stats",
    entities=["driver_id"],
    ttl=timedelta(weeks=52),
    features=[
        Feature(name="conv_rate", dtype=ValueType.FLOAT),
        Feature(name="acc_rate", dtype=ValueType.FLOAT),
        Feature(name="avg_daily_trips", dtype=ValueType.INT64),
    ],
    batch_source=driver_stats_source,
    tags={"team": "driver_performance"},
)
```
Example of feature_repo.yaml
```
project: driver_ranking
registry: s3://driver-rank-repo/registry.db
provider: local
online_store:
  type: redis
  connection_string: "redis-service.default.svc.cluster.local:6379
```

## Redis server service address
If you have deployed using the redis-server using the [redis.yaml](config/redis.yaml), the IP can be found by using this command:
```
kubectl get svc redis-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
## Online Feature server IP address
If you have deployed using the redis-server using the [Feature_server.yaml](config/feature_server.yaml), the IP can be found by using this command:
```
kubectl get svc feature-server-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
## Test connection to Redis Service in the K8s cluster
```bash
kubectl run -i myredis --image=redis --rm=true --restart=Never -- redis-cli -h redis-service ping
# PONG
```
## Deploy a CronJob in the Cluster
This CrobJob resource maybe deployed to periodically materialize offline features into the online store. May be configured to work with [S3 storage](https://github.com/tedhtchang/populate_feast_online_store#s3-storage)
