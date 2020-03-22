<p><strong>service node:</strong>&nbsp;一个服务，例如meta/collector/replica, 并不是对应一台物理机器，在run.sh中会使用-applist指定该机器要启动的node类型, 例如：</p>
<p>&nbsp;</p>
<p>$PWD/pegasus\_server config.ini -app\_list meta</p>
<p>service_node里有一个service_app, 其有几个子类，例如：replication_service_app、meta_service_app、info_collector_app, node不同的类型里包含了不同的service_app</p>
<p>&nbsp;</p>
<p><strong><strong>service engine:</strong></strong>&nbsp;一台物理机器上只有一个, 是一个单例, 用于管理service node, 其成员函数start_node用于根据service_app_spec启动一个service node</p>
<p>&nbsp;</p>
<p><strong><strong>replica_stub:</strong></strong>&nbsp;用于管理replica分片，在replication_service_app中存储一个replica_stub类型实例</p>
<p>&nbsp;</p>
<p><strong><strong>replica: </strong></strong>表示一个表分片副本</p>
