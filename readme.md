# How to Create an Active-active (CRDB) in Redis Enterprise for Openshift

## Pre-requisites 

* Two working Redis Enteprise Clusters (REC) in different Openshift environments
  * Important Note: the two RECs should have *different* `names`/`fqdn`. If this is not met, the CRDB creation will result in bad state.
* REC _admin role_ credentials to both RECs
* Appropriate resources available in each REC to create DBs of the requested size.
* Openshift Permissions: _todo_

## Topology
<img src="images/topology.png" />

## High Level Workflow
The following is the high level workflow which you will follow:
1. Document the required parameters.
2. Formulate the CRDB creation JSON payload using the parameters *from both RECs* in a single JSON document.
3. POST the JSON payload *to one* of the REC's API endpoints. (Yes, just one; it will coordinate with the other(s).)


## Required Parameters
The following parameters will be required to form the JSON payload to create the CRDB. 
| Parameter | Parameter Name in REST API | Name in `crdb-cli`| Description |
| --------- | ---  | --- | --- |
| Cluster FQDN | `name` | `fqdn` | This is the name of the REC as determined from ` GET https://<rec_api>/v1/cluster | jq .name`|
| API URL | `url` | `url` | This is the route the API endpoint as specified in `apiIngressURL`. Should be prefixed with `https://` |
| Cluster Admin Username/password | `credentials` | `username`,`password` | Cluster Admin role username/password |
| Replication Endpoint | `replication_endpoint` | `replication_endpoint` | This will be <`your_db_name`><`dbIngressSuffix`>:443 where `dbIngressSuffix` is specified in your REC spec |
| Replication TLS SNI | `replication_tls_sni` | `replication_tls_sni` | _todo_ |



Here is an example when creating a CRDB with database name <i>test-db</i>:
| Parameter Name in REST API | Example value | 
| -------------------------- | ------------- |
| `name` | `https://api-raas-site-a.apps.bbokdoct.redisdemo.com` |
| `url` | rec-a.raas-site-a.svc.cluster.local |
| `credentials` | username: `b@rl.com`, password: `something` | 
| `replication_endpoint` | <i>test-db</i><span style="color:orange">-raas-site-a.apps.bbokdoct.redisdemo.com</span>:443  |
| `replication_tls_sni` | <i>test-db</i><span style="color:orange">-raas-site-a.apps.bbokdoct.redisdemo.com</span> |


## Let's go!

Ensure you've documented the required parameters as in the above section. You'll need these!

1. Apply the `activeActive` spec to both Redis Enterprise clusters appropriately. Details about the REC API are <a href="https://github.com/RedisLabs/redis-enterprise-k8s-docs/blob/master/redis_enterprise_cluster_api.md" target="_blank">here</a>.

_todo_ add validatation of the dbIngressSuffix; is it a configmap?

REC at site A:
```
  activeActive: # edit values according to your cluster
    apiIngressUrl:  api-raas-site-a.apps.bbokdoct.redisdemo.com 
    dbIngressSuffix: -raas-site-a.apps.bbokdoct.redisdemo.com 
    method: openShiftRoute
```
REC at site B:
```
  activeActive: # edit values according to your cluster
    apiIngressUrl: api-raas-site-b.apps.bbokdoct.redisdemo.com 
    dbIngressSuffix: -raas-site-b.apps.bbokdoct.redisdemo.com
    method: openShiftRoute
```    
This will result in the API route `api-`<`clustername`> being added in both sites as below:

Redis Enterprise Cluster in Site A
```
$ oc get all
NAME                                             READY   STATUS    RESTARTS   AGE
pod/rec-a-0                                      2/2     Running   0          13m
pod/rec-a-services-rigger-7864cf7987-bsl4m       1/1     Running   0          13m
pod/redis-enterprise-operator-5d4dcf5dfc-7gklr   1/1     Running   0          152m

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/rec-a      ClusterIP   None             <none>        9443/TCP,8001/TCP,8070/TCP   13m
service/rec-a-ui   ClusterIP   172.30.225.190   <none>        8443/TCP                     13m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rec-a-services-rigger       1/1     1            1           13m
deployment.apps/redis-enterprise-operator   1/1     1            1           152m

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/rec-a-services-rigger-7864cf7987       1         1         1       13m
replicaset.apps/redis-enterprise-operator-5d4dcf5dfc   1         1         1       152m

NAME                     READY   AGE
statefulset.apps/rec-a   1/1     13m

NAME                                 HOST/PORT                                          PATH   SERVICES   PORT   TERMINATION   WILDCARD
route.route.openshift.io/api-rec-a   api-raas-site-a.apps.bbokdoct.redisdemo.com               rec-a      api    passthrough   None
route.route.openshift.io/rec-ui-a    rec-ui-a-raas-site-a.apps.bbokdoct.redisdemo.com          rec-a-ui   ui     passthrough   None
```


Redis Enterprise Cluster in Site B:
```
$ oc get all
NAME                                             READY   STATUS    RESTARTS   AGE
pod/rec-b-0                                      2/2     Running   0          150m
pod/rec-b-services-rigger-5d5d8ffb44-vp5xr       1/1     Running   0          150m
pod/redis-enterprise-operator-5d4dcf5dfc-w2v4g   1/1     Running   0          150m

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/rec-b      ClusterIP   None             <none>        9443/TCP,8001/TCP,8070/TCP   150m
service/rec-b-ui   ClusterIP   172.30.223.236   <none>        8443/TCP                     150m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rec-b-services-rigger       1/1     1            1           150m
deployment.apps/redis-enterprise-operator   1/1     1            1           150m

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/rec-b-services-rigger-5d5d8ffb44       1         1         1       150m
replicaset.apps/redis-enterprise-operator-5d4dcf5dfc   1         1         1       150m

NAME                     READY   AGE
statefulset.apps/rec-b   1/1     150m

NAME                                 HOST/PORT                                          PATH   SERVICES   PORT   TERMINATION   WILDCARD
route.route.openshift.io/api-rec-b   api-raas-site-b.apps.bbokdoct.redisdemo.com               rec-b      api    passthrough   None
route.route.openshift.io/rec-b-ui    rec-b-ui-raas-site-b.apps.bbokdoct.redisdemo.com          rec-b-ui   ui     passthrough   None
```



2. Create the JSON payload for CRDB creation request as in this <a href="./crdb.json" target="_blank">example</a> using the required parameters. Save the file as `crdb.json` in your current working directory.
```
{
    "default_db_config": {
      "name": "<db_name>",
      "replication": false,
      "memory_size": 10240000,
      "aof_policy": "appendfsync-every-sec",
      "shards_count": 1
    },
    "instances": [
      {
        "cluster": {
          "url": "<site_a_api_endpoint>",
          "credentials": {
            "username": "<site_a_username>",
            "password": "<site_a_password>"
          },
          "name": "<site_a_rec_name/fqdn>",
          "replication_endpoint": "<site_a_replication_endpoint>443",
          "replication_tls_sni": "<site_a_replication_endpoint>"
        }
      },
      {
        "cluster": {
          "url": "<site_b_api_endpoint>",
          "credentials": {
            "username": "<site_b_username>",
            "password": "<site_b_password>"
          },
          "name": "<site_b_rec_name/fqdn>",
          "replication_endpoint": "<site_b_replication_endpoint>:443",
          "replication_tls_sni": "<site_b_replication_endpoint>"
        }
      }
    ],
    "name": "<db_name>",
    "encryption": true,
    "compression": 0
  }
```

3. Apply the following to the API endpoint at *just one* cluster:
`curl -k -u b@rl.com:<snip> https://api-raas-site-a.apps.bbokdoct.redisdemo.com/v1/crdbs -X POST -H 'Content-Type: application/json' -d@crdb.json`
You should see a reply from the API as in the following which indicates the payload was well formed and the request is being actioned:
```
{
  "id": "eb091f0a-839e-48a6-989f-8b5e0034f132",
  "status": "queued"
}
```   
If successful, you will see the DB created in the Redis Enterprise UI for a healthy and syncing CRDB:
<img src="images/active-db.png" />

You will also see two new services per cluster: `test-db` and `test-db-headless`, as well as a new route which is used for the replication: `test-db-raas-site-a.apps.bbokdoct.redisdemo.com`
```
$ oc get svc,routes -n raas-site-a
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/rec-a              ClusterIP   None             <none>        9443/TCP,8001/TCP,8070/TCP   38m
service/rec-a-ui           ClusterIP   172.30.14.235    <none>        8443/TCP                     38m
service/test-db            ClusterIP   172.30.177.110   <none>        17946/TCP                    31m
service/test-db-headless   ClusterIP   None             <none>        17946/TCP                    31m

NAME                                 HOST/PORT                                          PATH   SERVICES   PORT    TERMINATION   WILDCARD
route.route.openshift.io/api-rec-a   api-raas-site-a.apps.bbokdoct.redisdemo.com               rec-a      api     passthrough   None
route.route.openshift.io/rec-ui-a    rec-ui-a-raas-site-a.apps.bbokdoct.redisdemo.com          rec-a-ui   ui      passthrough   None
route.route.openshift.io/test-db     test-db-raas-site-a.apps.bbokdoct.redisdemo.com           test-db    17946   passthrough   None
```
```
$ oc get svc,routes -n raas-site-b
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/rec-b              ClusterIP   None             <none>        9443/TCP,8001/TCP,8070/TCP   3h48m
service/rec-b-ui           ClusterIP   172.30.223.236   <none>        8443/TCP                     3h48m
service/test-db            ClusterIP   172.30.204.212   <none>        17946/TCP                    31m
service/test-db-headless   ClusterIP   None             <none>        17946/TCP                    31m

NAME                                 HOST/PORT                                          PATH   SERVICES   PORT    TERMINATION   WILDCARD
route.route.openshift.io/api-rec-b   api-raas-site-b.apps.bbokdoct.redisdemo.com               rec-b      api     passthrough   None
route.route.openshift.io/rec-b-ui    rec-b-ui-raas-site-b.apps.bbokdoct.redisdemo.com          rec-b-ui   ui      passthrough   None
route.route.openshift.io/test-db     test-db-raas-site-b.apps.bbokdoct.redisdemo.com           test-db    17946   passthrough   None
```

## Troubleshooting steps
_todo_
