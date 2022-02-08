# es

## 总体
    1、文档，索引文件。是按段存储的。 搜索的时候逐个段都会被查询到
    2、索引文件定期刷到磁盘。或者手动刷到磁盘里。日志文件（WAL write ahead log）默认每次都刷到磁盘里。
        文档写入时，不会立刻写入到磁盘里。
        先写入到内存中。但是已经可以被索引到了。定期刷新到持盘里形成一个段。 
        文档删除是标记删除。形成一个删除文件。.del。
        由于定期刷盘。并且每刷一次。形成一次新段。所以段会很多。需要定期的进行段合并。
        小段合并成大段。被标记删除的段不会出现在新段里。合并后之前的小段会被删除。这时候标记删除的文档才会在磁盘中删除。

        为了保证未刷磁盘的索引数据不丢失。系统有一个操作日志文件。这个文件刷盘频率高于索引文件默认每次操作都写盘。系统重起后。从最后一次保存点之后的操作日志进行回放。完成数据的修复。

    3、每个可被索引的字段。生成一个倒排索引文件，正排索引。
    4、根据ID更新文档。会做两个步骤。一个是根据ID查找出具体文档。然后才是更新数据。
        更新数据其实又分为三步。1、旧文档标记删除。2、写入新文档。3、更新索引。

    5、文档分到那个分片里，文档根据文档ID和分片数求余。

### 文档存储
    原始文档行式压缩存储。文件读取依赖CPU和IO。
    由三部分组成。 
    fnm 文件保存文档field的原数据信息。
        文档的字段名称。类型。 
    fdx 文档的位置索引信息。
        1024个chunk归为一个block。 block记录chunk的起始文档ID。以及chunk在fdt的位置。
    fdt 原始文档物理存储
        多个chunk组成。总大小大于16KB，lucene引擎会对原始文档进行压缩存储。

    原始文档查询时候，
    1、根据二分查找fdx,定位chunk所在的block。
    2、根据block中chunk的起始文档ID找出所在chunk。
    3、通过chunk在fdt中的位置。找出完整的chunk数据（需要解压）

    默认情况下，所有字段内容都存储在_source字段里。获取任何字段信息耗费的开销是一致的。
    如果某些字段频繁获取。可以考虑单独存储。这样避免频繁读取文件。解析文件。
    mapping 里字段定义是添加 store:true 即可。

### 索引存储
    倒排索引可以解决：词 在那些文档里出现过。 注意：所有字段混合在一起。
    倒排索引文件由三部分组成
    tip   词典索引。包括前缀和后缀指针
        前缀词典采用fst格式。查找复杂度为 O(len(str))
        fst优点：
            共享前缀，节省空间，内存占用率低，压缩比高。模糊查询支持好
            内存存放前缀索引，磁盘存放后缀词典
        缺点：结构复杂，输入要求有序，更新不易。

    tim   后缀词块、文档列表指针

    doc   文档列表、词频
        文档列表
            1、差分压缩。每个记录只记录当前文档ID和前一个文档ID的差值。
            2、256个文档ID存放在一个block中。
            3、block的头信息存放block中每个文档ID的占用的bit位数。
        例子：
            现在有 
                73， 300， 302， 332， 343， 372 这几个文档ID
            查分值为：
                73， -227， 2， 30， 11， 29
            分组存放
                组1 73， -227， 2     
                    最大值227 需要8比特来标识。那么组1那三个数都需要8比特
                组2 30， 11， 29
                    最大值为30 需要5个比特存储。那么组2这三个数都需要5比特
            最后存储格式为：
                组1  8, 73， -227，2 
                组2  5，30， 11， 29

                6个4字节的数据 压缩为 1bit + 3 * 8bit + 1bit + 3 * 5bit = 7字节
        文档列表合并
            使用跳表。优先遍历数据量小的链表。去大链表中进行查询。如果存在则该文档符合条件。加快文档列表合并

    倒排索引查询过程
    1、内存中加载字典树(tip)。通过FST匹配前缀找到后缀词块位置
    2、根据词块位置，读取磁盘中的tim(后缀词) 并找出后缀相应的倒排表位置信息。
    3、根据倒排表位置去doc文件中加载倒排表。
    4、借助跳表结构。加快文档ID合并。

    doc_values 正排索引 列式存储。所有列数据组合在一起就是完整的数据。

    解决字段聚合、排序、范围查询功能。

    doc_values 每个字段默认开启。要想关闭。在mapping里 指定 doc_values:false

    dvm     元数据信息
    dvd     文档ID以及字段值得存储
    
    数值型的压缩
    1、如果所有的数值各不相同或者缺失，设置一个标记并记录这些值
    2、如果这些值小于256.使用一个简单的编码表。
    3、如果这些值大于256，检测是否存在一个最大公约数
    4、如果没有存在公约数。从最小的数值开始，统一计算偏移量进行编码。

    字符串型的压缩
    1、去重后存放到顺序表中（按字典顺序排序）。每个值有个ID。
    2、实际存储的是是这个ID。 ID占用空间大小固定。比字符串要小的多。


## roaring bitmap
    减少了正常bitmap的占用空间
    把32位整形数，16 bit高位 作为分组。组内存储  0-65535 这些尾数（2字节）。
    最多有 65536 组数据。每组数据最多 65535个数据。
    尾数存储有三种格式。
    1、 ArrayContainer
        数量小于 4096 的。用数组存储。 查找就用二分法。 最多占用空间为 4096 * 2 Byte = 8KB
    2、 BitmapContainer
        数量大于等于4096的。用bitmap存储 占用 65536 个bit 即 8KB。
    3、RunContainer
        存储连续的数量。压缩率比较高。最坏打算。都不连续。65536/2个数。每个数由 2字节本身值，2字节连续数。 共65536 * 2Byte  = 128KB 空间。 
        比如
        11，12，13，14 ，21，22，23，25
        存储为，11，3，21，2，25，0

    es 里存储格式:
    1、所有数据 / 65536 求出前组号。 相同的为一组。 那么最多就有 65536
    2、组号相同的一组内。
        数量小于 4096 的。用数组存储。 查找就用二分法。 最多占用空间为 4096 * 2 Byte = 8KB
        数量大于等于4096的。用bitmap存储 占用 65536 个bit 即 8KB。

## query or filter 
    
    filter  
        唯一值过滤结果(精确值最好用过滤)，会以bitmap的形式存储在内存中。相同条件查询时，速度会有优化。
            LRU策略删除。新加文档也会更新存在的bitmap。
        范围查询结果不会缓存。

        低版本的 filtered .高版本的是bool组合查询。
        先过滤然后再查询。 过滤有缓存。使用bitmap 来加速。

    后置过滤 post_filter
        先查询。后过滤。过滤结果不缓存。 使用跳表来加速过滤。
        只应该用于聚合查询。

    query 
        涉及到结果的评分。效率比filter查询慢。结果不缓存。

## query_then_fetch 和 dfs_query_then_fetch
    
    主要是分片内数据量多少不均匀。导致的idf差异值不同

    query_then_fetch：
        1、发送查询到每个shard
        2、找到所有匹配的文档，并使用本地的Term/Document Frequency信息进行打分
        3、对结果构建一个优先队列（排序，标页等）
        4、返回关于结果的元数据到请求节点。注意，实际文档还没有发送，只是分数
        5、来自所有shard的分数合并起来，并在请求节点上进行排序，文档被按照查询要求进行选择
        6、最终，实际文档从他们各自所在的独立的shard上检索出来
        7、结果被返回给用户

    dfs_query_then_fetch：
        1、预查询每个shard，询问Term和Document frequency
        2、发送查询到每个shard
        3、找到所有匹配的文档，并使用全局的Term/Document Frequency信息进行打分
        4、对结果构建一个优先队列（排序，标页等）
        5、返回关于结果的元数据到请求节点。注意，实际文档还没有发送，只是分数
        6、来自所有shard的分数合并起来，并在请求节点上进行排序，文档被按照查询要求进行选择
        7、最终，实际文档从他们各自所在的独立的shard上检索出来
        8、结果被返回给用户


## es 下载
    mac 版本  https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.5.1-darwin-x86_64.tar.gz

    7.5.1 是版本号。可以根据实际需要进行更改切换。

## 安装 ik 中文分析插件
    
    注意点。 ik 版本必须和 es版本相同否则无法安装成功

    https://github.com/medcl/elasticsearch-analysis-ik

    分析(analysis) 整体流程如下:
    1、过滤非法字符（无用字符）。
    2、分词
    3、标记过滤。过滤无用的或者不需要建立索引的词语。

    对文档而言，分析是在文档create, update时操作的。建立倒排索引。
    对查询而言，分析是在查询前操作的。把查询条件进行分析后获取到具体的索引KYE。这样可以进行检索。

## mapping
    
    mapping 可以把文档解析成数据对象。进而进行索引排序。
    sting 类型的数据。
        默认是添加索引的 index: analyzed。
        如果不想建立索引。需要显示的指定例如：

        name: {
            type: string,
            index: not_analyzed,
        }

### 多字段映射
    name: {
        type: "text",
        analyzer: "ik_max_word",
        search_analyzer: "ik_smart",
        fields: {
            standard: {
            type: "text",
            analyzer: "standard"
            }
        },
    },

    在字段name的基础上。给name添加了name.standard 的子字段。 name.standard字段上使用默认的分析器建立索引。只做简单切分。中文按字，英文按空格
    name 字段上使用 ik_max_word 分析器建立索引。


## 查询
    过滤查询。通过 bool/ must 、should、must_not / filter 进行组合查询。
    查询耗时比较高。所以尽量少在查询上加过多的条件。使用过滤耗时会有很大优化。

    过滤 过滤结果有缓存。
    查询器 结果不缓存。

    must        相当于 and
    should      相当于 or
    must_not    相当于 not

    1、全文搜索
        match           单字段条件
            match: {
                field_name:{
                    'query': xx,
                    'operator': 'and', / or   # 假如查询条件可以切分为多个词 默认是or 
                    'minimum_should_mathc': '75%' # 控制精度 即 关键词里有最少有几项都符合才认为是符合条件的。
                }
            }

        multi_match 多字段
            multi_match:{
                query: 'xx',
                type: best_fields
                fields: [field1, field2^2], # ^2代表着boost权重为2， field1的权重为1
            }

            type : 
                best_fields 按单个字段最优匹配结果。
                most_fields 多个字段结合来看。
                cross_fields 

        * match_phrase  短语匹配 相关度。 需要再看看。* 
            查询条件里的字必须都存在并且。存在的顺序还得和查询条件的顺序一致。中间可以指定允许的最大间隔。
           match_phrase:{
                filed:{
                    query: xx,
                    slop: 1         # 匹配间隔度。
                }
           }
           slop=1。相当于中间可以隔一个字。
           slop=3。相当于中间可以隔三个字。
           slop字
    2、确定值查询
        term 单个确定值
            'term':{filed: xx}
        terms 多个确定值
            'terms':{
                field:[xx, xx, xx ]
            }
        range 范围过滤
            range:{
                filed:{
                    'gte': x,
                    'lt': y,
                }
            }
        exists 存在某个字段
            exists:{
                'field': 'field_name'
            }
        missing 没有某个字段
            missing:{
                'field': 'field_name'
            }

    3、组合查询
        bool 类型
        query:{
            bool: {
                should:[],
                must:[],
                mininum_should_match: 2 # should 条件的精度控制。最少符合俩个才算。
                filter:{

                }
            }
        }
        dis_max 类型。 
        按单一条件。的匹配度计算分值。 不再混合field1+filed2分数之后的平均值。
        加上tie_breaker 参数后，会匹配filed1, filed2。最后分值计算为 最佳分值 + 其他分值 * tie_breaker 。 tie_breaker 值在【0，1】之间。0代表最佳分值。1代表bool组合查询类型。一般tie_breaker选择在【0.1，0.4】之间。
        query:{
            dis_max:{
                queries:[
                    {
                        match: {
                            filed1: 'xx',
                        }
                    },
                    {
                        match: {
                            filed2: 'xx',
                        }
                    },
                ],
                tie_breaker: 0.3        # 在0 到1 之间
            }
        }

    4、控制查询权重。
        通过boost控制。 默认1.如果提高权重更改boost大于1. 如果降低权重更改boost值在0-1之间。
        query:{
            bool: {
                must:{}
                should:[
                    {
                        math: {
                            filed_name: {
                                query: xx,
                                boost: 3,           # boost 可以更改分值权重。默认为1
                            }
                        }
                    },
                    {
                        math: {
                            filed_name: {
                                query: xx,
                                boost: 2,           # 越高权重越大。排名越靠前。 
                            }
                        }
                    },
                ]
            }
        }

    5、模糊匹配
        prefix 如果查询的字段没有索引就是全量索引。小数据集合可以使用
            prefix:{
                filed: 'xx'
            }

        wildcard 通配符查询 通过？匹配任意字符 * 匹配任意多个  如果查询的字段没有索引就是全量索引。小数据集合可以使用
            wildcard:{
                filed: xx?x*xx
            }

        regexp 正则匹配  如果查询的字段没有索引就是全量索引。小数据集合可以使用
            regexp: {
                field: 正则规则。
            }

        match_phrase_prefix 短语前缀
            match_phrase_prefix:{
                    filed:{
                        query: xx,
                        slop: 2,
                        "max_expansions": 50    # 最多返回结果50
                    }
            }


## 分值说明
    1、 但一查询。只有match这种的。 文档分值score 是根据。词频、反向文档率（在其他文档中出现的频率） 字段长度 来计算得来的。
    2、组合查询 有多个must, should,  文档分值score 是由匹配must should 语句分数总和再除以must should 语句的总数。
    说白了就是同一个文档。分别在must, should 查询条件中的分值相加后 再除以符合条件的总数量的均值。

## api 接口
    1、查看mapping 字段映射情况
        http://localhost:9200/index_name/_mapping

    2、检查分析器，分词情况

        http://localhost:9200/index_name/_analyze
        standard 默认分析器。英文是按空格分词。 中文是逐字切分。
        1、指定字段
            {
                "field":"name",
                "text": "中国人名" 
            }
        2、指定分析器
            {
                "analyzer": "ik_smart",
                "text": "中国人名" 
            }

    3、查询
        http://localhost:9200/index_name/_search
        POST
            body{}
    
    4、查询解释器
        http://localhost:9200/index_name/_explain
        body {}

    5、创建索引
        http://localhost:9200/index_name 
        method put
        body:{
            setting:{
                "number_of_shards" :   1,   #   分片数
                "number_of_replicas" : 0    #   副本数
            }
            mapping:{

            }
        }
        
    6、删除索引
        http://localhost:9200/index_name 
        method DELETE

    7、更新索引
        更新setting
            http://localhost:9200/index_name/_settings
            method PUT
            body: {
                "number_of_replicas" : 1
            }
        更改mapping
            http://localhost:9200/index_name
            method PUT
            body: {
                mappings:{
                    "dynamic": "strict"
                }
            }

    8、创建别名
        http://localhost:9200/index_name_alias/_alias/index_name
        method PUT

        查看单个别名
        GET http://localhost:9200/index_name/_alias/*

        查看所有别名信息
        GET http://localhost:9200/_aliases

        注意：别名不能和现有的索引名称相同。


    9、零停机。进行索引替换
        http://localhost:9200/_aliases
        method POST
        body:{
            "actions": [
                { "remove": { "index": "my_index_v1", "alias": "my_index" }},
                { "add":    { "index": "my_index_v2", "alias": "my_index" }}
            ]
        }

    10、开启索引
        http://localhost:9200/<index_name>/_open
        method POST

    11、关闭索引
        http://localhost:9200/<index_name>/_close
        method POST

    12、查看记录数
        http://localhost:9200/<index_name>/_count 
        method GET
    
    13、查看当前机器上的索引情况
        http://localhost:9200/_cat/indices?v

    14、集群健康情况
        http://localhost:9200/_cluster/health

    15、查看索引设置信息
        http://172.16.16.16:9200/<index_name>/_settings

        更改索引设置
        PUT http://172.16.16.16:9200/<index_name>/_settings
        {
            "index.blocks.read_only_allow_delete": null
        }

        当磁盘满的话。会报 read-only 错误。 再腾出磁盘空间后。再运行这个命令。
        curl -XPUT -H "Content-Type: application/json" http://10.10.10.10:9200/_all/_settings -d '{"index.blocks.read_only_allow_delete": null}'

    16、根据ID查看文档
        http://172.16.16.16:9200/<index_name>/<type_name>/<id>
        like: http://172.16.16.16:9200/collection_rick/collection/vHgUymfcwKx49_GiNE6cDQ

    17、查看统计信息
        http://localhost:9200/_stats | python -m json.tool    
        格式化显示json数据

        查看节点统计信息
        http://localhost:9200/_nodes/stats | python -m json.tool 

    18、查看等待中的任务
        http://localhost:9200/_cluster/pending_tasks
