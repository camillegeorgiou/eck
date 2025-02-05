# Elastic Stack UAM - Full Combined (ECK)

## **Intro**

This folder contains instructions on the full set of activities to complete the User Activity Monitoring Customer Architecture Play in an On-Premise environment. The approach is broken into Kibana Auditing and Elasticsearch Auditing and combines UAM and enhanced Audit Logging.

This guide assumes the following:

* A Platimun or Enterprise License
* Transport and HTTP security setup
* A monitoring cluster (8.x) with trust established against the main cluster to enable CCR. Visit this [link](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-remote-clusters.html) for more information on how to leverage remote cluster in ECK
* Relevant permissions to access system indices, create API and manage configuration
* Sudo access to the Kubernetes cluster running the ECK deployment.
* Filebeat instances to ship auditing data from Kibana and Elastic to the monitoring cluster
  * [Run Filebeat on Kubernetes](https://www.elastic.co/guide/en/beats/filebeat/current/running-on-kubernetes.html)
  * [Run Beats on ECK](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-beat.html)

***ECK Setup***
Ensure that stack-monitoring is setup and audit logging is enabled. Below is an example ECK configuration that enables stack-monitoring and audit logging.

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
spec:
  monitoring:
    metrics:
      elasticsearchRefs:
      - name: monitoring
        namespace: observability
    logs:
      elasticsearchRefs:
      - name: monitoring
        namespace: observability
  nodeSets:
  - name: default
    config:
      # https://www.elastic.co/guide/en/elasticsearch/reference/current/enable-audit-logging.html
      xpack.security.audit.enabled: true
---
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
spec:
  monitoring:
    metrics:
      elasticsearchRefs:
      - name: monitoring
        namespace: observability
    logs:
      elasticsearchRefs:
      - name: monitoring
        namespace: observability
  config:
    # https://www.elastic.co/guide/en/kibana/current/xpack-security-audit-logging.html
    xpack.security.audit.enabled: true
```

This, however, might have large volumes of unnecessary data shipped to the monitoring cluster. We can leverage audit log filters to remove what is not needed and ensure only the right logs for UAM are shipped.

Key highlights are the filtering in the elasticsearch config and the kibana config

```
# elasticsearch
spec:
  version: 8.15.4
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
      xpack.security.audit.enabled: true
      xpack.security.audit.logfile.events.include: "authentication_success"
      xpack.security.audit.logfile.events.emit_request_body: true
      xpack.security.audit.logfile.events.ignore_filters.system.users: ["*_system", "found-internal-*",  "_xpack_security", "_xpack", "elastic/fleet-server","_async_search", "found-internal-admin-proxy"]
      xpack.security.audit.logfile.events.ignore_filters.realm.realms : [ "_es_api_key" ]
      xpack.security.audit.logfile.events.ignore_filters.internal_system.indices : ["*ml-inference-native-*", "*monitoring-es-*"]


# kibana
spec:
  monitoring:
    metrics:
      elasticsearchRefs:
      - name: es-mon
        namespace: observability
    logs:
      elasticsearchRefs:
      - name: es-mon
        namespace: observability
  version: 8.15.4
  count: 1
  elasticsearchRef:
    name: es01
  config:
    # https://www.elastic.co/guide/en/kibana/current/monitoring-metricbeat.html
    monitoring.kibana.collection.enabled: false
    xpack.security.audit.enabled: true
    xpack.security.audit.ignore_filters:
    - categories: [web]
    - actions: [saved_object_open_point_in_time, saved_object_close_point_in_time, saved_object_find, space_find]
```

***Main Cluster***

1. Ensure security settings have been enabled in the ECK config 

*The xpack.http.ssl* settings are watcher specific and are required to run the reindexing.

Example config:

```
xpack.security.enrollment.enabled: true
xpack.security.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: /path-to/config/certs/elastic-certificates.p12
xpack.security.http.ssl.truststore.path: /path-to/config/certs/truststore.p12
xpack.security.http.ssl.verification_mode: certificate

xpack.http.ssl.keystore.path: /path-to/config/certs/elastic-certificates.p12
xpack.http.ssl.truststore.path: /path-to/config/certs/truststore.p12
xpack.http.ssl.verification_mode: certificate

xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.path: /path-to/config/certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /path-to/config/certs/truststore.p12
xpack.security.transport.ssl.verification_mode: certificate

```

2. Enable auditing on every Kibana and Elastic node serving the main cluster(s) as explained above, ensuring that the configurations in the ECK config are as described above:


elasticsearch:

```
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include: "authentication_success"
xpack.security.audit.logfile.events.emit_request_body: true
xpack.security.audit.logfile.events.ignore_filters.system.users: ["*_system", "found-internal-*",  "_xpack_security", "_xpack", "elastic/fleet-server","_async_search", "found-internal-admin-proxy"]
xpack.security.audit.logfile.events.ignore_filters.realm.realms : [ "_es_api_key" ]
xpack.security.audit.logfile.events.ignore_filters.internal_system.indices : ["*ml-inference-native-*", "*monitoring-es-*"]
```

kibana:

```
xpack.security.audit.enabled: true
xpack.security.audit.ignore_filters:
- categories: [web]
- actions: [saved_object_open_point_in_time, saved_object_close_point_in_time, saved_object_find, space_find]
```
> [!NOTE]  
> (Optional) In the Dev Tools console of the cluster, run the following command to insert additional dynamic audit settings. (Note: These can be updated without restarting the deployment, allowing for faster iteration of tweaking and customizing audit settings. In your environment, you may have additional users you wish to ignore.)
> ```
>    PUT _cluster/settings
>    {
>      "persistent": {
>        "xpack.security.audit.logfile.events": {
>          "emit_request_body": true,
>          "ignore_filters.ess-internal-users.users": [
>            "_system",
>            "elastic/fleet-server",
>            "found-internal-admin-proxy",
>            "found-internal-system",
>            "kibana-metricbeat"
>          ],
>          "include": [
>            "authentication_success"
>          ]
>        }
>      }
>    }
> ```

3. In Dev Tools, set up the components and pipeline to create the kibana_objects_01 index which contains the object id to name mapping

    a) In Dev Tools Console (or via the API), create an ingest pipeline using the config from [ingest-pipeline.txt.](./main-cluster-side/ingest-pipeline.txt)
    This ingest pipeline extracts the saved object id from the "_id" field of documents in the kibana_analytics index and removes fields that are not required for analysis.

    b) Create a component template using the [component-template.txt](./main-cluster-side/component-template.txt) file. This template contains the mapping for the new kibana objects index and specifies use of the ingest pipeline.

    c) Create an index template that uses the component template:

    ```
    PUT _index_template/kibana_objects-new
    {
      "index_patterns": ["kibana_objects*"],
      "composed_of": ["kibana-objects"]
    }
    ```
        
    d) Reindex the existing kibana_analytics data into the new kibana index with the formalised mappings and ingest pipeline:
      ```
      POST _reindex
      {
        "source" : {
            "index" : ".kibana_analytics"
        },
        "dest" : {
            "index" : "kibana_objects-01",
            "pipeline" : "kibana-objectid"
        }
      }
      ```
4. Create an API key and add an advanced watch using the [watcher.txt](./main-cluster-side/watcher.txt) file. This watcher checks for new documents in the kibana_analytics system index and reindexes into the new kibana index if the condition is met.

> [!IMPORTANT]  
> You'll need to first create an API Key for authorization of the request via Stack Management-> Security API Keys or in Dev Tools Console:
  
  ```
POST /_security/api_key
{
  "name": "system_index_access_key",
  "role_descriptors": {
    "system_index_access": {
      "cluster": [],
      "index": [
        {
          "names": [".kibana_analytics*", "kibana_objects-01"],
          "privileges": ["all"],
          "allow_restricted_indices": true
        }
      ]
    }
  }
}
```

- Edit the Watcher before creating it and add in the Elasticsearch (.es.) host name from the Kubernetes service and the encoded API Key. Note: When adding in the API key the line should start: "Authorization": "ApiKey xxxxxx" followed by a space and then the actual key value

Example:
```
PUT _watcher/watch/kibana-reindex
{
    "trigger": {
        "schedule": {
            "interval": "5m"
        }
    },
    "input": {
        "search": {
            "request": {
                "search_type": "query_then_fetch",
                "indices": [
                    ".kibana_analytics"
                ],
                "rest_total_hits_as_int": true,
                "body": {
                    "query": {
                        "range": {
                            "@updated_at": {
                                "gte": "now-5m"
                            }
                        }
                    }
                }
            }
        }
    },
    "condition": {
        "always": {}
    },
    "actions": {
        "reindex": {
            "webhook": {
                "scheme": "https",
                "host": "es-prod-es-http",
                "port": 9200,
                "method": "post",
                "path": "/_reindex",
                "params": {},
                "headers": {
                    "Authorization": "ApiKey <encoded>"
                },
                "body": "{\"source\": {\"index\": \".kibana_analytics\"}, \"dest\": {\"index\": \"kibana_objects-01\"}, \"conflicts\": \"abort\"}"
            }
        }
    }
}
```

5. Still in the Dev Tools console, add slowlog settings for any indices of interest. Note: consider adding these settings to index templates so future indices automatically inherit the settings upon creation.

    ```
    PUT kibana_sample_data_ecommerce/_settings
    {
        "index.search.slowlog.threshold.query.info": "0ms",
        "index.search.slowlog.threshold.fetch.info": "0ms"
    }
    ```


***Monitoring Deployment***
- The files needed for this section are contained in [mon-cluster-side folder](./mon-cluster-side/).

The Monitoring Cluster should be set up to trust the Main Cluster. This needs to be done at the config level. For more information, see: ​​[Add remote clusters using TLS certificate authentication | Elasticsearch Guide [8.11] | Elastic
](https://www.elastic.co/guide/en/elasticsearch/reference/current/remote-clusters-cert.html)

As per the Main Cluster, security settings should also include xpack.http.ssl* values to accommodate the watcher webhook action.

Configure the Main Cluster as a remote cluster:

*Modify the “seeds” value as required. Note - the port should be the transport port. You can also do this via the UI and setup CCR with the main-cluster as leader index. But in most cases, remote cluster setup should have handled this

```
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "main-cluster": {
          "skip_unavailable": false,
          "mode": "sniff",
          "proxy_address": null,
          "proxy_socket_connections": null,
          "server_name": null,
          "seeds": [
            "127.0.0.1:9300"
          ],
          "node_connections": 3
        }
      }
    }
  }
}
```

Configure the follower index for the main-cluster(s). Ensure that the follower indices begin with `kibana_objects-`. In this example we have `kibana_objects-prod`. Update the remote cluster name to match the remote cluster from information from `GET _remote/info`
```
PUT /kibana_objects-prod/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster" : "es-prod",
  "leader_index" : "kibana_objects-01",
  "settings": {
    "index.number_of_replicas": 1
  },
    "max_read_request_operation_count": 5120,
    "max_outstanding_read_requests": 12,
    "max_read_request_size": "32mb",
    "max_write_request_operation_count": 5120,
    "max_write_request_size": "9223372036854775807b",
    "max_outstanding_write_requests": 9,
    "max_write_buffer_count": 2147483647,
    "max_write_buffer_size": "512mb",
    "max_retry_delay": "500ms",
    "read_poll_timeout": "1m"
}
```

6. (Optional) Create an ILM policy to ensure that the datastream lifecycle is managed
```
PUT _ilm/policy/elastic-logs
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "30d",
            "max_primary_shard_size": "50gb"
          }
        }
      },
      "warm": {
        "min_age": "0d",
        "actions": {
          "set_priority": {
            "priority": 50
          }
        }
      }
    }
  }
}
```

7. Create a component template for the UAM fields
```
PUT _component_template/elastic-logs-8-uam_mapping
{
  "template": {
    "mappings": {
      "properties": {
        "elasticsearch": {
          "properties": {
            "uam": {
              "properties": {
                "saved_object": {
                  "properties": {
                    "id": {
                      "ignore_above": 1024,
                      "type": "keyword"
                    }
                  }
                },
                "search": {
                  "properties": {
                    "date_range": {
                      "properties": {
                        "duration": {
                          "type": "long"
                        },
                        "from": {
                          "ignore_above": 1024,
                          "type": "keyword"
                        },
                        "to": {
                          "ignore_above": 1024,
                          "type": "keyword"
                        }
                      }
                    },
                    "duration": {
                      "type": "long"
                    },
                    "hits": {
                      "type": "float"
                    },
                    "query": {
                      "ignore_above": 1024,
                      "type": "keyword"
                    },
                    "index": {
                      "ignore_above": 1024,
                      "type": "keyword"
                    },
                    "id": {
                      "ignore_above": 1024,
                      "type": "keyword"
                    },
                    "aggregations": {
                      "ignore_above": 1024,
                      "type": "keyword"
                    }
                  }
                },
                "application": {
                  "ignore_above": 1024,
                  "type": "keyword"
                },
                "opaque_id": {
                  "ignore_above": 1024,
                  "type": "keyword"
                },
                "origination": {
                  "ignore_above": 1024,
                  "type": "keyword"
                },
                "error": {
                  "ignore_above": 1024,
                  "type": "keyword"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

8. Locate the filebeat datastream created in the monitoring cluster after enabling the audit logging. Take note of the filebeat index template name used by the datastream. Then, update the index template by using the filebeat template [here](./mon-cluster-side/filebeat-index-template.txt) to use the component template and ILM policy defined in the previous steps. This updated template ensures that the UAM fields are populated and that the final pipeline `stack-uam-router` is used by the indices. Replace the index_template name and also the target index-patterns before applying the changes.

```
PUT _index_template/filebeat-8.15.3 # Change this value
```

```
# also update this section
  "index_patterns": [
    "filebeat-8.15.3"
  ],
```

9. Create the ingest pipeline in the monitoring cluster for each file in the [pipeline](./mon-cluster-side/audit-logging-pipeline/) folder.

10. Rollover the datastream to enable it to use the updates

```
POST filebeat-8.15.3/_rollover
```

11. Create an enrich policy using [enrich.txt](./mon-cluster-side/enrich.txt) and execute it:

    a) Execute the policy

  ```
  PUT _enrich/policy/objectid-policy/_execute
  ```

12. Create an ingest pipeline using [ingest-pipeline.txt](./mon-cluster-side/ingest-pipeline.txt). This utilises the new enrichment policy and set processors to set the title of the object.

13. Create a component template to formalise mappings of the new enriched index that will be used for visualizations.

    a) Use [component-template.txt](./mon-cluster-side/component-template.txt)

    b) Create an index template using the new component template:

```
PUT _index_template/kibana-transform
{
  "template": {
    "settings": {
      "index": {
        "default_pipeline": "enrich-ids",
        "final_pipeline": "enrich-ids"
      }
    }
  },
  "index_patterns": ["kibana-transform-*"],
  "composed_of": ["transform-obj"]
}
```

14. Create a transform. The transform filters data from the kibana logs based on the presence of the saved_object.id field. Do not start the transform yet.

    a) Create the transform using [transform.txt](./mon-cluster-side/transform.txt)

15. Create a watch that re-executes the enrich policy when new objects are added, using [watcher.txt](./mon-cluster-side/watcher.txt).

> [!IMPORTANT]  
> You'll need to first create an API Key for authorization of the request via Stack Management-> Security API Keys or in Dev Tools Console:
```
POST /_security/api_key
{
  "name": "enrich_access_key",
  "role_descriptors": {
    "system_index_access": {
      "cluster": ["manage_enrich"],
      "index": [
        {
          "names": ["kibana_objects-*"],
          "privileges": ["all"],
          "allow_restricted_indices": true
        }
      ]
    }
  }
}
```

- Edit the Watcher before creating it and add in the Elasticsearch (.es.) host name and API Key. Note: When adding in the API key the line should start with: "Authorization": "ApiKey xxxxxx" followed by a space and then the actual key value

Example:
```
PUT _watcher/watch/policy-execute
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "search_type": "query_then_fetch",
        "indices": [
          "kibana_objects-*"
        ],
        "rest_total_hits_as_int": true,
        "body": {
          "query": {
            "range": {
              "@updated_at": {
                "gte": "now-5m"
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "always": {}
  },
  "actions": {
    "reindex": {
      "webhook": {
        "scheme": "https",
        "host": "es-mon-es-http",
        "port": 9200,
        "method": "PUT",
        "path": "_enrich/policy/objectid-policy/_execute",
        "params": {},
        "headers": {
          "Authorization": "ApiKey <encoded>"
        }
      }
    }
  },
  "metadata": {
    "xpack": {
      "type": "json"
    }
  }
}
```

***Starting up***

16. Verify that data is being ingested:
     
    a) Check the follower index to be sure that documents are populated

    b) Start the transform to create the kibana-transform-02 index

```
POST _transform/kibana-transform-02/_start
```

17. Import the following assets via Stack Management -> Saved Objects:
- [post8.14-dashboard.ndjson](../assets/post8.14-dashboard.ndjson)

***Slowlog and Query capturing***
18. In the monitoring cluster, ensure slowlogs are turned on for the individual indices to be monitored. Example
```
    PUT kibana_sample_data_ecommerce/_settings
    {
        "index.search.slowlog.threshold.query.info": "0ms",
        "index.search.slowlog.threshold.fetch.info": "0ms"
    }
```

19. Import the following assets via Stack Management -> Saved Objects:
- [slowlog.ndjson](../assets/slowlog.ndjson)