
用户执行stream load主要有两种方式：

- 将请求直接提交给be，并由该节点作为本次stream load任务的coordinator。

- 将http请求提交给fe，fe再通过http重定向将数据导入请求转发给某一个be节点，该be节点作为本次stream load任务的coordinator，此时的fe仅仅起到一个转发作用。

本文中主要介绍第二种方式。其主要执行流程如下：

- 用户提交stream load请求到fe

- 
