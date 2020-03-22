<p>info_collector_app中有两个成员：info_collector和available_detector。</p>
<p>其中info_collector用于获取所有的perf counter信息。available_detector用于获取服务的可用性</p>
<h2><strong><strong>一、info_collector</strong></strong></h2>
<p>info_collector::on_app_stat: 通过get_app_stat周期性获取每个节点app的counter信息以及所有app的汇总信息--&gt;rows</p>
<p>1.get_apps_and_nodes 获取所有表信息--&gt;apps/节点信息--&gt;nodes</p>
<p>2.call_remote_command 获取所有节点对应的perf counter信息--&gt;results</p>
<p>3.for all nodes:</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;3.1 通过result解析其所有的perf counters --&gt; info</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;3.2 对于每个info中的counter，解析其counter name, app id, partition id</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;3.3 利用counter更新row（row: 每个app一个row）</p>
<p>4.根据获取的所有rows更新本地perf counters</p>
<p>&nbsp;</p>
<p>最后，info_collector根据pegasus_counter_reporter来报告所有的perf counters信息</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;pegasus_counter_reporter::on_reporter_timer --&gt; pegasus_counter_reporter::update: 周期性的向falcon或者log日志中填写perf counters信息(根据配置决定是否向falcon或者log日志中填写)</p>
<p>&nbsp;</p>
<h2><strong><strong>二、available_detector</strong></strong></h2>
<p>按照配置中的detect interval seconds创建定时任务，用于定时向pegasus中写入和读取随机key，通过返回状态统计数据(detect times/failure times for minute/hour/day/)。</p>
<p>并创建一个定时任务，每分钟执行一次。计算上次report经过的时间，分别判断是否执行hour report/minute report/day report, 报告的内容是：detect times, sucess times</p>
<p>&nbsp;</p>
<p>另：info_collector还包含有另外两个定时任务：on_storage_size_stat和on_capacity_unit_stat</p>
<p>on_storage_size_stat用于获取每个表的实际占用空间--&gt;std::map&lt;int32_t, std::vector&lt;int64_t&gt;&gt; st_value_by_app。 key-app_id, value-[app_partition_count, stat_partition_count, storage_size_in_mb]</p>
<p>on_capacity_unit_stat用于获取read capacity unit(读总量)/write capacity unit（写总量） --&gt; std::map&lt;int32_t, std::pair&lt;int64_t, int64_t&gt;&gt; cu_value_by_app。 key-app_id, value-(read_cu, write_cu)</p>
<p>具体探测任务是由每个pegasus_server_impl中的capacity_unit_calculator类型成员_cu_calculator来完成的。内部有两个perf counter，分别统计read cu和write cu</p>
<p>&nbsp;</p>
