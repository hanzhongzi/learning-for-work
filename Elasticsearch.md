

# Elasticsearch

## 1.概述及其安装

1.分布式文件存储，每个字段都被索引并可被搜索

2.分布式的实时分析搜索引擎

3.高扩展，且可处理结构化和非结构化的数据

首先需要安装*Java*   **(rpm包很香~)：**

```
# wget https://download.oracle.com/otn-pub/java/jdk/14.0.1+7/664493ef4a6946b186ff29eb326336a2/jdk-14.0.1_linux-x64_bin.rpm

# yum -y install jdk-14.0.1_linux-x64_bin.rpm
```

安装Elasticsearch

```
# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.2-x86_64.rpm
```

网速会很慢，要不就翻墙下载后传到云服务器。

然后安装：

```
# yum -y installelasticsearch-7.6.2-x86_64.rpm
```

启动：

```
# systemctl restart elasticsearch.service
```

这就启动了~

检查运行状态：（默认9200，http.port:XXXX）

```
[root@VM_48_15_centos notice-center-v0.1.1-beta]# curl -X GET "localhost:9200/?pretty"
{
  "name" : "VM_48_15_centos",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "qiL-xHYwRoCSAhGonhtJIw",
  "version" : {
    "number" : "7.6.2",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "ef48eb35cf30adf4db14086e8aabd07ef6fb113f",
    "build_date" : "2020-03-26T06:34:37.794943Z",
    "build_snapshot" : false,
    "lucene_version" : "8.4.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```



配置文件默认是： `/etc/elasticsearch` 

### 官网的说明：

### Directory layout of RPM

The RPM places config files, logs, and the data directory in the appropriate locations for an RPM-based system:

| Type        | Description                                                  | Default Location                   | Setting        |
| ----------- | ------------------------------------------------------------ | ---------------------------------- | -------------- |
| **home**    | Elasticsearch home directory or `$ES_HOME`                   | `/usr/share/elasticsearch`         |                |
| **bin**     | Binary scripts including `elasticsearch` to start a node and `elasticsearch-plugin` to install plugins | `/usr/share/elasticsearch/bin`     |                |
| **conf**    | Configuration files including `elasticsearch.yml`            | `/etc/elasticsearch`               | `ES_PATH_CONF` |
| **conf**    | Environment variables including heap size, file descriptors. | `/etc/sysconfig/elasticsearch`     |                |
| **data**    | The location of the data files of each index / shard allocated on the node. Can hold multiple locations. | `/var/lib/elasticsearch`           | `path.data`    |
| **jdk**     | The bundled Java Development Kit used to run Elasticsearch. Can be overridden by setting the `JAVA_HOME` environment variable in `/etc/sysconfig/elasticsearch`. | `/usr/share/elasticsearch/jdk`     |                |
| **logs**    | Log files location.                                          | `/var/log/elasticsearch`           | `path.logs`    |
| **plugins** | Plugin files location. Each plugin will be contained in a subdirectory. | `/usr/share/elasticsearch/plugins` |                |
| **repo**    | Shared file system repository locations. Can hold multiple locations. A file system repository can be placed in to any subdirectory of any directory specified here. | Not configured                     | `path.repo`    |



----

## 2. 配置文件的说明

温馨提示：**先将原来的配置文件备好哦**

官网说明：https://www.elastic.co/guide/en/elasticsearch/reference/7.6/settings.html

Elasticsearch 有三个配置文件

- `elasticsearch.yml` 配置Elasticsearch
- `jvm.options` 配置Elasticsearch的java虚拟机
- `log4j2.properties` 配置日志

日志的书写格式遵循yaml文件，引用环境变量为 `${...}`。

例子：数据和日志目录

```yaml
path:
    data: /var/lib/elasticsearch
    logs: /var/log/elasticsearch
```

除了上面的数据和日志路径配置也可以支持多路径

```yaml
path:
  data:
    - /mnt/elasticsearch_1
    - /mnt/elasticsearch_2
    - /mnt/elasticsearch_3
```

集群名称：一个节点只可以加入一个集群，集群名在急群中是共享的，默认名称是elasticsearch，建议是取一个适当的名字。确保集群名不成福。

```yaml
cluster.name: logging-prod
```

节点名称：默认是主机名称，

```yaml
node.name: prod-data-2
```

绑定网络地址：https://www.elastic.co/guide/en/elasticsearch/reference/7.6/modules-network.html

默认是只绑定环回地址，127.0.0.1` and `[::1]。通常是需要配置主机绑定IP的。

```yaml
network.host: 127.0.0.1 #0.0.0.0 外网可访问
```

可以一个机器启动多个实例。

网络设置：为了加入集群，一个节点所有集群中的其他节点的列表，且写出可作为主节点的节点名称

```yaml
discovery.seed_hosts:
   - 192.168.1.10:9300
   - 192.168.1.11 默认9300
   - seeds.mydomain.com #如果主机名解析为多个IP地址，则该节点将尝试在所有解析的地址处发现其他节点
   - [0:0:0:0:0:ffff:c0a8:10c]:9301 #IPv6 addresses must be enclosed in square brackets.
cluster.initial_master_nodes: #主节点
   - master-node-a
 # - master-node-b
 # - master-node-c
```

集群节点： https://www.elastic.co/guide/en/elasticsearch/reference/7.6/modules-discovery-hosts-providers.html

自主选举主节点：https://www.elastic.co/guide/en/elasticsearch/reference/7.6/modules-discovery-bootstrap-cluster.html



## 3.系统注意事项

https://www.elastic.co/guide/en/elasticsearch/reference/7.6/system-config.html

1.需要关闭及其交换空间，否则节点机器不稳定。

```sh
swapoff -a #且需要永久关闭
```



2.Elasticsearch 在节点和 HTTP 客户端之间进行通信也使用了大量的套接字，所以需要大量的文件描述符：官方是65535及更高



[内存映射](https://zhuanlan.zhihu.com/p/73453720)需要大一点

```sh
sysctl -w vm.max_map_count=262144
```



## 将节点加入到集群

官方说明：https://www.elastic.co/guide/en/elasticsearch/reference/7.6/add-elasticsearch-nodes.html

**写好配置文件，启动Elasticsearch。节点自动发现并加入指定的集群。**

集群它会自动的将分发数据到可用节点。当你加入节点他是自动处理的。**注意向集群中加入新的节点的时候，一定要是一个干净的es，否则残留的数据导致加入集群失败**

主节点也会存数据，你也可以配置为专门的【[特殊节点](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/modules-node.html)】,当添加节点后，会自动的分配备份数据。



#### 节点的配置：https://www.elastic.co/guide/en/elasticsearch/reference/7.6/modules-node.html

解释：主节点必须有权访问`data/`目录（就像`data`节点一样 ），因为这是节点重新启动之间保持群集状态的位置。

```yaml
node.master: true #有无资格作为主节点
node.data: true #是否存储数据
node.max_local_storage_nodes: 3 #集群最大节点数

```

查看集群节点：`curl -X GET "localhost:9200/_cat/nodes?pretty"`

查看集群状态：`curl -X GET "localhost:9200/_cat/health?pretty"`

#### 重启集群： https://www.elastic.co/guide/en/elasticsearch/reference/7.6/restart-cluster.html

###### 待完善...





#### 集群监控：

安装kibana：`rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch`

```
# vim /etc/yum.repos.d/kibana.repo
	[kibana-7.x]
	name=Kibana repository for 7.x packages
	baseurl=https://artifacts.elastic.co/packages/7.x/yum
	gpgcheck=1
	gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
	enabled=1
	autorefresh=1
	type=rpm-md

# yum clean all && yum makecache
# yum -y install kibana
```

默然安装目录：

| Type         | Description                                                  | Default Location             | Setting     |
| ------------ | ------------------------------------------------------------ | ---------------------------- | ----------- |
| **home**     | Kibana home directory or `$KIBANA_HOME`                      | `/usr/share/kibana`          |             |
| **bin**      | Binary scripts including `kibana` to start the Kibana server and `kibana-plugin` to install plugins | `/usr/share/kibana/bin`      |             |
| **config**   | Configuration files including `kibana.yml`                   | `/etc/kibana`                |             |
| **data**     | The location of the data files written to disk by Kibana and its plugins | `/var/lib/kibana`            | `path.data` |
| **logs**     | Logs files location                                          | `/var/log/kibana`            | `path.logs` |
| **optimize** | Transpiled source code. Certain administrative actions (e.g. plugin install) result in the source code being retranspiled on the fly. | `/usr/share/kibana/optimize` |             |
| **plugins**  | Plugin files location. Each plugin will be contained in a subdirectory. | `/usr/share/kibana/plugins`  |             |

##### 配置kibana:https://www.elastic.co/guide/en/kibana/7.6/settings.html

```yaml
# vim /etc/kibana/kinbab.yml
server.port: 5601 #http访问端口
server.host: "0.0.0.0" #服务器名
elasticsearch.hosts:
["","",""] # 刚刚机器的那些个9200地址都填入t
elasticsearch.requestTimout: 9999 #请求超时时间，默认30000
```

等一会，查看浏览器`http://localhost:5601` =>心跳主机=>开启监控
