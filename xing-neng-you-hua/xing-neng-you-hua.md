# 性能优化

Peer节点性能优化包含以下几个方面

* protobuf反序列化优化
  * Block、Transaction等被大量反复反序列化
  * 影响DeliverService性能
* 签名计算优化

