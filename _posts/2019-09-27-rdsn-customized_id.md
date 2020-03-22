<h2><strong>customized_id_mgr:</strong></h2>
<ul>
<li>成员变量：</li>
</ul>
<p>&nbsp;&nbsp;&nbsp;&nbsp;std::unordered_map&lt;std::string, int&gt; _names; id和name的映射</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;std::vector&lt;std::string&gt; _names2; 存储name，id为下标</p>
<ul>
<li>成员函数：</li>
</ul>
<p>&nbsp;&nbsp;&nbsp;&nbsp;int register_id(const char *name): 为name注册一个id。如果id存在，则直接返回，否则，产生一个递增id。并向_names和_names2中保存映射</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;const char *get_name(int id) const: 根据id获取name</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;get_id: 根据name获取id</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;int max_value() const: 获取name-id对数量</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<h2><strong><strong>customized_id&lt;T&gt;:</strong></strong></h2>
<ul>
<li>成员变量：</li>
</ul>
<p>&nbsp;&nbsp;&nbsp;&nbsp;int _internal_code: 由customized_id_mgr根据name生成的id</p>
<p>&nbsp;</p>
<ul>
<li>成员函数：</li>
</ul>
<p>&nbsp;&nbsp;&nbsp;&nbsp;const char *to_string() const: 获取_iternal_code所对应的name</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;static bool is_exist(const char *name): 判断name是否已经存在</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;static const char *to_string(int code): 获取code对应的name</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;static customized_id from_string(const char *name, customized_id invalid_value): 根据name获取对应的id</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;static int assign(const char *xxx): 调用customized_id_mgr::register_id为xxx注册一个id</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;operator int() const: 强转操作，将customized_id转换成int类型，这里返回_iternal_code</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;operator T() const: 强转操作，将customized_id转换成T类型, 这里根据_iternal_code生成一个T类型对象返回</p>
<p>&nbsp;</p>
