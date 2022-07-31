# Elasticsearch (ver 7.8+)

## Performance

- Provide specific shard routing to force the request be handled by target shards, while the data should also be routed specifically while writing.


## Reliability

### Specific Role
- Specify dedicated master-eligible node, which only responsible for the master-related tasks, avoid the resources shared by other chores (e.g. indexing, searching, ...).
- The above rule can also be applied on other tasks (e.g. coordinating, ingesting, ...).

### Snapshot Management
- Repository with official plugin provided: S3, HDFS, GCS, AZURE.
- Plan the policy for target indices, and the snapshot restoring is backward compatible across 1 major version (7.x snapshot is applicable on 7.x ~ 8.x cluster).


## Maintainability

### How to assess the shard number?
- General practice for shard size from official site: 1G ~ 40G.
- More shards while we need more indexing.
- Less shards can leads to better searching performance, but the transplanting is much higher during re-balancing.

### How to handle type casting?
- Leverage dynamic field mappings, and can standardize the field name with fixed convention (e.g. `*_datetime`, `*_count`, ...) to cooperate with simple matching rules.

### How to deprecate legacy index?
- Keep using index aliases as much as possible, so we only have to update the index aliases inside elasticsearch, no application side modification is needed.
- Pack the index renaming operation within a atomic action clause, so the operation will not crash all services even if it goes wrong.
- It can also be applied on life cycle management, while the alias can be switched to a new index every day / week.

### How to manage indices the grows over time?
- Official best practice: rollover pattern on time-based indices.
- Shrink API can also be adopted for reducing indices amount.

### How to cost down the data storage?
- Hot-warm-cold architecture, while different policies are applied on data storage.
    - Hot nodes should handle the writing, indexing and searching, more RAM will be consumed.
    - Warm nodes are read-only, more resource are used on temp caches.
    - Cold nodes are archived, caches wil also be released right after respond.
- Rollup regularly.

## References
- [腾讯万亿级 Elasticsearch 技术解密](https://cloud.tencent.com/developer/article/1598364)
- [Elasticsearch Index Best Practices](https://ithelp.ithome.com.tw/articles/10246673)
