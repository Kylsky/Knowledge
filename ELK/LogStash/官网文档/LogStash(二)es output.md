# LogStash(二)es output

由于工作中用es作为logstash的输出，因此在众多的logstash的output plugin中，先挑es output来看看。

es output只使用HTTP协议。从Logstash 2.0开始，HTTP是与Elasticsearch交互的首选协议。出于一些原因，官方强烈建议在节点协议上使用HTTP。HTTP只是稍微慢一点，但是更容易管理和使用。当使用HTTP协议时，可以升级Elasticsearch版本，而不必在锁步中升级Logstash。



## 写入数据到不同es索引

如果你向同一个Elasticsearch集群发送事件，但你的目标是不同的索引，你可以：

1. 使用不同的Elasticsearch输出，每个输出的索引参数值都不同
2. 使用一个Elasticsearch输出并使用动态变量替换索引参数

每个Elasticsearch output是一个连接到es集群的新客户端:

1. 这个output必须初始化客户端并连接到Elasticsearch(如果客户端更多，重启时间会更长)
2. 它有一个关联的连接池

为了尽可能减少到Elasticsearch的连接数量，最大化并发量并减少“小”批量请求的数量(这些请求很容易填满队列)，通常使用单个Elasticsearch output更有效。举例如下：

```
output {
      elasticsearch {
        index => "%{[some_field][sub_field]}-%{+YYYY.MM.dd}"
      }
}
```

可以使用mutate过滤器和添加[@metadata]字段来为每个事件设置目标索引。[@metadata]字段不会被发送到Elasticsearch。举例如下：

```
filter {
      if [log_type] in [ "test", "staging" ] {
        mutate { add_field => { "[@metadata][target_index]" => "test-%{+YYYY.MM}" } }
      } else if [log_type] == "production" {
        mutate { add_field => { "[@metadata][target_index]" => "prod-%{+YYYY.MM.dd}" } }
      } else {
        mutate { add_field => { "[@metadata][target_index]" => "unknown-%{+YYYY}" } }
      }
    }
    output {
      elasticsearch {
        index => "%{[@metadata][target_index]}"
      }
    }
```



## 配置选项

| Setting                                                      | Input type                                                   | Required |
| ------------------------------------------------------------ | ------------------------------------------------------------ | -------- |
| [`action`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-action) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`bulk_path`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-bulk_path) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`cacert`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-cacert) | a valid filesystem path                                      | No       |
| [`cloud_auth`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-cloud_auth) | [password](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#password) | No       |
| [`cloud_id`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-cloud_id) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`custom_headers`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-custom_headers) | [hash](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#hash) | No       |
| [`doc_as_upsert`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-doc_as_upsert) | [boolean](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#boolean) | No       |
| [`document_id`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-document_id) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`document_type`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-document_type) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`failure_type_logging_whitelist`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-failure_type_logging_whitelist) | [array](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#array) | No       |
| [`healthcheck_path`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-healthcheck_path) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`hosts`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-hosts) | [uri](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#uri) | No       |
| [`http_compression`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-http_compression) | [boolean](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#boolean) | No       |
| [`ilm_enabled`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-ilm_enabled) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string), one of `["true", "false", "auto"]` | No       |
| [`ilm_pattern`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-ilm_pattern) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`ilm_policy`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-ilm_policy) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`ilm_rollover_alias`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-ilm_rollover_alias) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`index`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-index) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`keystore`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-keystore) | a valid filesystem path                                      | No       |
| [`keystore_password`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-keystore_password) | [password](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#password) | No       |
| [`manage_template`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-manage_template) | [boolean](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#boolean) | No       |
| [`parameters`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-parameters) | [hash](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#hash) | No       |
| [`parent`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-parent) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`password`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-password) | [password](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#password) | No       |
| [`path`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-path) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`pipeline`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-pipeline) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`pool_max`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-pool_max) | [number](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#number) | No       |
| [`pool_max_per_route`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-pool_max_per_route) | [number](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#number) | No       |
| [`proxy`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-proxy) | [uri](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#uri) | No       |
| [`resurrect_delay`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-resurrect_delay) | [number](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#number) | No       |
| [`retry_initial_interval`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-retry_initial_interval) | [number](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#number) | No       |
| [`retry_max_interval`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-retry_max_interval) | [number](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#number) | No       |
| [`retry_on_conflict`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-retry_on_conflict) | [number](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#number) | No       |
| [`routing`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-routing) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`script`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-script) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`script_lang`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-script_lang) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`script_type`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-script_type) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string), one of `["inline", "indexed", "file"]` | No       |
| [`script_var_name`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-script_var_name) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`scripted_upsert`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-scripted_upsert) | [boolean](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#boolean) | No       |
| [`sniffing`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-sniffing) | [boolean](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#boolean) | No       |
| [`sniffing_delay`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-sniffing_delay) | [number](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#number) | No       |
| [`sniffing_path`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-sniffing_path) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`ssl`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-ssl) | [boolean](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#boolean) | No       |
| [`ssl_certificate_verification`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-ssl_certificate_verification) | [boolean](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#boolean) | No       |
| [`template`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-template) | a valid filesystem path                                      | No       |
| [`template_name`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-template_name) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`template_overwrite`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-template_overwrite) | [boolean](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#boolean) | No       |
| [`timeout`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-timeout) | [number](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#number) | No       |
| [`truststore`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-truststore) | a valid filesystem path                                      | No       |
| [`truststore_password`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-truststore_password) | [password](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#password) | No       |
| [`upsert`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-upsert) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`user`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-user) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`validate_after_inactivity`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-validate_after_inactivity) | [number](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#number) | No       |
| [`version`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-version) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string) | No       |
| [`version_type`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-version_type) | [string](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html#string), one of `["internal", "external", "external_gt", "external_gte", "force"]` | No       |

### action

协议不可知的(例如，非http，非java特定的)配置，在这里，协议不可知的方法，弹力搜索行动执行。有效的行动是:

index/delete/create/update/