### KIP-962

I authored another [KIP-962](https://cwiki.apache.org/confluence/display/KAFKA/KIP-962%3A+Relax+non-null+key+requirement+in+Kafka+Streams).
It was a rewarding experience once again.
This KIP enhanced my grasp of Kafka Streams internals.
I learned about the intricate process of how it builds it topology.

The implementation was done in:
 - https://github.com/apache/kafka/pull/14174
 - https://github.com/apache/kafka/pull/14107

Going forward, there probably needs to be an additional change with regard to when and how to filter null-key records upon repartition.
See the discussion [here](https://github.com/apache/kafka/pull/14174#discussion_r1347986930).
