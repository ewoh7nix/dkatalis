## REQUIREMENTS
- Ansible > 2.11
- Debian >= 11

## HOWTO

As we run the Elasticsearch service on AWS EC2 instance, you may need to adjust some variables on 
`inventory/group_vars` directory based on your setup on AWS. 

### Run the Ansible playbook
Provision EC2 instance:
```
ANSIBLE_CONFIG=ansible.cfg ansible-playbook provision-instance.yml -u admin
```

Deploy Elasticsearch to EC2 instance:
```
ANSIBLE_CONFIG=ansible.cfg ansible-playbook deploy-es.yml -u admin
```

The elastic password is stored in Ansible inventory/group_vars/tag_xxx and encrypted using Ansible vault.
We could check the Elasticsearch service by querying to the ES endpoint
```
$ export ELASTIC_PASSWORD="dkatalis"
$ sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://<elasticsearch_ip>:9200 
{
  "name" : "node1",
  "cluster_name" : "dkatalis-cluster-es",
  "cluster_uuid" : "_q981FHLSjyeGaJhL5biUA",
  "version" : {
    "number" : "8.13.3",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "617f7b76c4ebcb5a7f1e70d409a99c437c896aea",
    "build_date" : "2024-04-29T22:05:16.051731935Z",
    "build_snapshot" : false,
    "lucene_version" : "9.10.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}


$ sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://<elasticsearch_ip>:9200/_cluster/state?pretty
{
  "cluster_name" : "dkatalis-cluster-es",
  "cluster_uuid" : "_q981FHLSjyeGaJhL5biUA",
  "version" : 58,
  "state_uuid" : "j_ZCHgLbSrSQrPcaLhlIRA",
  "master_node" : "Uz2e5j4FSaSE0_FDVvSBIg",
  "blocks" : { },
  "nodes" : {
    "Uz2e5j4FSaSE0_FDVvSBIg" : {
      "name" : "node1",
      "ephemeral_id" : "rBtB6Y-WQhSQO5DAUB-_3Q",
      "transport_address" : "127.0.0.1:9300",
      "external_id" : "node1",
      "attributes" : {
        "ml.allocated_processors_double" : "2.0",
        "ml.allocated_processors" : "2",
        "ml.machine_memory" : "2545889280",
        "transform.config_version" : "10.0.0",
        "xpack.installed" : "true",
        "ml.config_version" : "12.0.0",
        "ml.max_jvm_size" : "1275068416"
      },
      "roles" : [
        "data",
        "data_cold",
        "data_content",
        "data_frozen",
        "data_hot",
        "data_warm",
        "ingest",
        "master",
        "ml",
        "remote_cluster_client",
        "transform"
      ],
      "version" : "8.13.3",
      "min_index_version" : 7000099,
      "max_index_version" : 8503000
    }
  },
 ---------omitted-------
```




1. What did you choose to automate the provisioning and bootstrapping of the instance? Why?
   I'm using Ansible to provisioning and deploy the needed elasticsearch packages. The ES configs
   are also managed in Ansible which's allign with its function as configuration management.

2. How did you choose to secure ElasticSearch? Why?
   We can secure Elasticsearch with the following method:
   - Enable authentication and authorization, ie: password, role-base access control(rbac) or 
     ip filtering using xpack plugins.
   - Enable HTTPS
   - Enable audit logs.
   On Debian, by default Elasticsearch was secured using password and HTTPS.

3. How would you monitor this instance? What metrics would you monitor?
   For EC2 instance, there's standard metrics that we can monitor in Cloudwatch. But for 
   Elasticsearch service, there're many metrics that can be useful to monitor, like:
   - cluster(health, status, state, num of nodes, etc)
   - sharding(status, num of shard, primary/replica shard, unasigned, index count, etc)

4. Could you extend your solution to launch a secure cluster of ElasticSearch nodes? What
   would need to change to support this use case? 
   We could add some steps to the playbook:
   - Create an enrollment token to the existing node.
   - Start enrollment process using the generated enrollmen token above. It will automatically join the cluster
     and redistribute shards to the new node.
   
5. Could you extend your solution to replace a running ElasticSearch instance with little or no downtime? How?
   We could add some steps on the playbook to exclude ES node from cluster. The step could be:
   - Make sure all nodes are green, you can check the status through cluster health
   - Exclude ES node from cluster, it will move shards to another nodes and ensure the data integrity
     and availability in the cluster.
   - Monitor the rebalancing process, it may take some times(depends on the size of data). You can monitor
     using command `GET _cat/allocation?v`. Once the process is complete, you can remove the node safely.

6. Was it a priority to make your code well structured, extensible, and reusable?
   DevOps/SRE/Cloud Infra may make a code to eliminate toil or manual repetitive tasks, many of them are
   just a simple task and invest too much time for well structured code that's just for a simple tasks
   may slower the process. 
   But, in some cases, our codes may become bigger and complex in the future, so keep following 
   standard or best practices will make our codes become more structured, maintenable, extensible
   and scalable. Reusable code could help us reduce the development time and save us from rewriting the 
   similar codes and also prevent duplication.
7. What sacrifices did you make due to time?
