Copy Primary⁣ 
然而仅仅依靠Move Primary可能无法满足Primary负载均衡，这时候就需要进行Copy Primary了。

当没有成功获取增广路径时，则说明简单通过角色切换的方式已经无法达到负载均衡了，必须通过迁移Primary来实现了。 迁移Primary算法的实现相对简单，其具体执行步骤如下：
1. 将节点按照Primary数量按从小到大排序，得到pri_queue
2. 对pri_queue上，id_min始终指向pri_queue的头结点，id_max始终指向pri_queue的尾节点，如下图所示:
 +------+------+------+------+------+------+------+------+
 |                                                       |
 V                                                       V
id_min                                                id_max
3. 对当前id_max上的所有Primary，分别找到其对应的磁盘并获取其磁盘负载，选择负载最大的磁盘及其对应的Primary，进行迁移
4. 对当前id_min/id_max指向的Primary数量分别+1/-1。重新排序，并循环执行上述步骤，直到id_min节点上的Primary数量 >= N/M，此时说明达到了平衡
Secondary负载均衡
上述讲解了Primary负载均衡，当然Secondary也同样需要负载均衡，否则的话可能会出现不同节点上Primary均衡，但是partition总数不均衡的情况。 因为在做Primary迁移时已经做过角色切换了，Secondary迁移就不用像Primary这么复杂，不用考虑角色切换的问题了。此时直接进行copy就可以。因此Secondary的负载均衡，直接采用Copy Primary一样的算法实现，这里不再赘述。 
