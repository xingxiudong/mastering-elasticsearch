## 使用filters优化查询

ElasticSearch支持多种不同类型的查询方式，这一点大家应该都已熟知。但是在选择哪个文档应该匹配成功，哪个文档应该呈现给用户这一需求上，查询并不是唯一的选择。ElasticSearch 查询DSL允许用户使用的绝大多数查询都会有各自的标识，这些查询也以嵌套到如下的查询类型中：
*   `constant_score`
*   `filterd`
*   `custom_filters_score`

那么问题来了，为什么要这么麻烦来使用filtering？在什么场景下可以只使用queries？ 接下来就试着解决上面的问题。


### 过滤器(Filters)和缓存

首先，正如读者所想，filters来做缓存是一个很不错的选择，ElasticSearch也提供了这种特殊的缓存，filter cache来存储filters得到的结果集。此外，缓存filters不需要太多的内存(它只保留一种信息，即哪些文档与filter相匹配)，同时它可以由其它的查询复用，极大地提升了查询的性能。设想你正运行如下的查询命令：
```javascript
{
    "query" : {
        "bool" : {
            "must" : [
            {
                "term" : { "name" : "joe" }
            },
            {
                "term" : { "year" : 1981 }
            }
            ]
        }
    }
}
```
该命令会查询到满足如下条件的文档：`name`域值为`joe`同时`year`域值为`1981`。这是一个很简单的查询，但是如果用于查询足球运动员的相关信息，它可以查询到所有符合指定人名及指定出生年份的运动员。

如果用上面命令的格式构建查询，查询对象会将所有的条件绑定到一起存储到缓存中；因此如果我们查询人名相同但是出生年份不同的运动员，ElasticSearch无法重用上面查询命令中的任何信息。因此，我们来试着优化一下查询。由于一千个人可能会有一千个人名，所以人名不太适合缓存起来；但是年份比较适合(一般`year`域中不会有太多不同的值，对吧？)。因此我们引入一个不同的查询命令，将一个简单的query与一个filter结合起来。
```javascript
{
    "query" : {
        "filtered" : {
            "query" : {
                "term" : { "name" : "joe" }
            },
            "filter" : {
                "term" : { "year" : 1981 }
            }
        }
    }
}
```
我们使用了一个filtered类型的查询对象，查询对象将query元素和filter元素都包含进去了。第一次运行该查询命令后，ElasticSearch就会把filter缓存起来，如果再有查询用到了一样的filter，就会直接用到缓存。就这样，ElasticSearch不必多次加载同样的信息。
###并非所有的filters会被默认缓存起来

缓存很强大，但实际上ElasticSearch在默认情况下并不会缓存所有的filters。这是因为部分filters会用到域数据缓存(field data cache)。该缓存一般用于按域值排序和faceting操作的场景中。默认情况下，如下的filters不会被缓存：

* numeric_range
* script
* geo_bbox
* geo_distance
* geo\_distance_range
* geo_polygon
* geo_shape
* and
* or
* not

尽管上面提到的最后三种filters不会用到域缓存，它们主要用于控制其它的filters，因此它不会被缓存，但是它们控制的filters在用到的时候都已经缓存好了。

###更改ElasticSearch缓存的行为

ElasticSearch允许用户通过使用\_chache和\_cache\_key属性自行开启或关闭filters的缓存功能。回到前面的例子，假定我们将关键词过滤器的结果缓存起来，并给缓存项的key取名为`year_1981_cache`，则查询命令如下：
```javascript
{
    "query" : {
        "filtered" : {
            "query" : {
                "term" : { "name" : "joe" }
            },
            "filter" : {
                "term" : {
                    "year" : 1981,
                    "_cache_key" : "year_1981_cache"
                }
            }
        }
    }
}
```

也可以使用如下的命令关闭该关键词过滤器的缓存：
```javascript
{
    "query" : {
        "filtered" : {
            "query" : {
                "term" : { "name" : "joe" }
            },
            "filter" : {
                "term" : {
                    "year" : 1981,
                    "_cache" : false
                }
            }
        }
    }
}
```

###为什么要这么麻烦地给缓存项的key取名

上面的问题换个说法就是，我有是否有必要如此麻烦地使用\_cache\_key属性，ElasticSearch不能自己实现这个功能吗？当然它可以自己实现，而且在必要的时候控制缓存，但是有时我们需要更多的控制权。比如，有些查询复用的机会不多，我们希望定时清除这些查询的缓存。如果不指定\_cache_key，那就只能清除整个过滤器缓存(filter cache)；反之，只需要执行如下的命令即可清除特定的缓存：
```javascript
curl -XPOST 'localhost:9200/users/_cache/clear?filter_keys=year_1981_cache'
```

###什么时候应该改变ElasticSearch 过滤器缓存的行为

当然，有的时候用户应该更多去了解业务需求，而不是让ElasticSearch来预测数据分布。比如，假设你想使用geo_distance 过滤器将查询限制到有限的几个地理位置，该过滤器在请多查询请求中都使用着相同的参数值，即同一个脚本会在随着过滤器一起多次使用。在这个场景中，为过滤器开启缓存是值得的。任何时候都需要问自己这个问题“过滤器会多次重复使用吗？”添加数据到缓存是个消耗机器资源的操作，用户应避免不必要的资源浪费。

###关键词查找过滤器

缓存和标准的查询并不是全部内容。随着ElasticSearch 0.90版本的发布，我们得到了一个精巧的过滤器，它可以用来将多个从ElasticSearch中得到值作为query的参数(类似于SQL的IN操作)。

让我们看一个简单的例子。假定我们有在一个在线书店，存储了用户，即书店的顾客购买的书籍信息。books索引很简单(存储在books.json文件中)：
```javascript
{
     "mappings" : {
         "book" : {
             "properties" : {
                "id" : { "type" : "string", "store" : "yes", "index" :
             "not_analyzed" },
                "title" : { "type" : "string", "store" : "yes", "index" :
             "analyzed" }
             }
         }
     }
}
```
上面的代码中，没有什么是非同寻常的；只有书籍的id和标题。
接下来，我们来看看clients.json文件，该文件中存储着clients索引的mappings信息：
```javascript
{
     "mappings" : {
         "client" : {
             "properties" : {
                "id" : { "type" : "string", "store" : "yes", "index" :
             "not_analyzed" },
                "name" : { "type" : "string", "store" : "yes", "index" :
             "analyzed" },
                "books" : { "type" : "string", "store" : "yes", "index" :
             "not_analyzed" }
             }
         }
     }
}
```
索引定义了id信息，名字，用户购买书籍的id列表。此外，我们还需要一些样例数据：
```javascript
curl -XPUT 'localhost:9200/clients/client/1' -d '{
 "id":"1", "name":"Joe Doe", "books":["1","3"]
}'
curl -XPUT 'localhost:9200/clients/client/2' -d '{
 "id":"2", "name":"Jane Doe", "books":["3"]
}'
curl -XPUT 'localhost:9200/books/book/1' -d '{
 "id":"1", "title":"Test book one"
}'
curl -XPUT 'localhost:9200/books/book/2' -d '{
 "id":"2", "title":"Test book two"
}'
curl -XPUT 'localhost:9200/books/book/3' -d '{
 "id":"3", "title":"Test book three"
}'
```
接下来想象需求如下，我们希望展示某个用户购买的所有书籍，以id为1的user为例。当然，我们可以先执行一个请求 `curl -XGET 'localhost:9200/clients/client/1'`得到当前顾客的购买记录，然后把books域中的值取出来，执行第二个查询：
```javascript
curl -XGET 'localhost:9200/books/_search' -d '{
"query" : {
        "ids" : {
            "type" : "book",
            "values" : [ "1", "3" ]
        }
    }
}'
```
这样做太麻烦了，ElasticSearch 0.90版本新引入了 关键词查询过滤器(term lookup filter)，该过滤器只需要一个查询就可以将上面两个查询才能完成的事情搞定。使用该过滤器的查询如下：
<pre>
curl -XGET 'localhost:9200/books/_search' -d '{
    "query" : {
        "filtered" : {
            "query" : {
                "match_all" : {}
            },
            "filter" : {
                "terms" : {
                    "id" : {
                        "index" : "clients",
                        "type" : "client",
                        "id" : "1",
                        "path" : "books"
                    },
                    "_cache_key" : "terms_lookup_client_1_books"
                }
            }
        }
    }
}'
</pre>

请注意`_cache_key`参数的值，可以看到其值为`terms_lookup_client_1_books`，它里面包含了顾客id信息。请注意，如果给不同的查询设置了相同的`_cache_key`，那么结果就会出现不可预知的错误。这是因为ElasticSearch会基于指定的key来存储查询结果，然后在不同的查询中复用。
接下来看看上述查询的返回值：
```javascript
{
    ...
    "hits" : {
        "total" : 2,
        "max_score" : 1.0,
        "hits" : [ {
            "_index" : "books",
            "_type" : "book",
            "_id" : "1",
            "_score" : 1.0, "_source" : {"id":"1", "title":"Test book one"}
        }, {
            "_index" : "books",
            "_type" : "book",
            "_id" : "3",
            "_score" : 1.0, "_source" : {"id":"3", "title":"Test book three"}
        } ]
    }
}
```
这正是我们希望看到的结果，太棒了！

### term filter的工作原理

回顾我们发送到ElasticSearch的查询命令。可以看到，它只是一个简单的过滤查询，包含一个全量查询和一个terms 过滤器。只是该查询命令中，terms 过滤器使用了一种不同的技巧——不是明确指定某些term的值，而是从其它的索引中动态加载。

可以看到，我们的过滤器基于id域，这是因为只需要id域就整合其它所有的属性。接下来就需要关注id域中的新属性：index,type,id,和path。idex属性指明了加载terms的索引源(在本例中是clients索引)。type属性告诉ElasticSearch我们的目标文档类型(在本例中是client类型)。id属性指明的我们在指定索引的指文档类型中的目标文档。最后，path属性告诉ElasticSearch应该从哪个域中加载term，在本例中是clients索引的books域。
    总结一下，ElasticSearch所做的工作就是从clients索引的client文档类型中，id为1的文档里加载books域中的term。这些取得的值将用于terms filter来过滤从books索引(命令执行的目的地是books索引)中查询到的文档，过滤条件是文档id域(本例中terms filter名称为id)的值在过滤器中存在。

<!-- note structure -->
<div style="height:50px;width:90%;position:relative;">
<div style="width:13px;height:100%; background:black; position:absolute;padding:5px 0 5px 0;">
<img src="../notes/lm.png" height="100%" width="13px"/>
</div>
<div style="width:51px;height:100%;position:absolute; left:13px; text-align:center; font-size:0;">
<img src="../notes/pixel.gif" style="height:100%; width:1px; vertical-align:middle;"/>
<img src="../notes/note.png" style="vertical-align:middle;"/>
</div>
<div style="height:100%;position:absolute;left:65px;right:13px;">
<p style="font-size:13px;margin-top:10px;">
请注意_source域必须存储，否则terms lookup功能无法使用。
</p>
</div>
<div style="width:13px;height:100%;background:black;position:absolute;right:0px;padding:5px 0 5px 0;">
<img src="../notes/rm.png" height="100%" width="13px"/>
</div>
</div>  <!-- end of note structure -->

###性能优化

前面的查询已经在ElasticSearch内部通过缓存机制进行了优化。terms会加载到filter cache中，并与查询命令提供的key关联起来。此外，一旦terms(在本例中是书的id信息)被加载到了缓存中，以后用到该缓存项的查询都不会再次从索引中加载，这意味着ElasticSearch可以通过缓存机制提高查询的效率。

如果业务场景中用到了terms lookup功能，数据量也不大，推荐用户将索引(本例中是clients索引)只设置一个分片，同时将分片的副本分发到所有含有books索引的节点上。这样做是因为ElasticSearch默认会读取本地的索引数据来避免不必要的网络传输、网络延时，从而提升系统的性能。

###从内嵌对象中加载terms

如果clients索引中books属性不再是id数组，而是对象数据，那么在查询命令中，我们就需要将id属性指向到内嵌的对象。即我们需要将`"id":"books"`变成`"id":"books.book"`。

###Terms lookup filter的缓存设置

前面已经提到，为了提供terms lookup功能，ElasticSearch引入一种新的cache类型，该类型的的缓存基于快速的**LRU(Least Recently Used)**策略。

<!-- note structure -->
<div style="height:70px;width:90%;position:relative;">
<div style="width:13px;height:100%; background:black; position:absolute;padding:5px 0 5px 0;">
<img src="../notes/lm.png" height="100%" width="13px"/>
</div>
<div style="width:51px;height:100%;position:absolute; left:13px; text-align:center; font-size:0;">
<img src="../notes/pixel.gif" style="height:100%; width:1px; vertical-align:middle;"/>
<img src="../notes/note.png" style="vertical-align:middle;"/>
</div>
<div style="height:100%;position:absolute;left:65px;right:13px;">
<p style="font-size:13px;margin-top:10px;">
如果想了解更多关于LRU缓存的知识，想了解它的工作原理，请参考网页： http://en.wikipedia.org/wiki/Page\_replacement\_algorithm#Least\_recently_used

</p>
</div>
<div style="width:13px;height:100%;background:black;position:absolute;right:0px;padding:5px 0 5px 0;">
<img src="../notes/rm.png" height="100%" width="13px"/>
</div>
</div>  <!-- end of note structure -->

ElasticSearch在elasticsearch.yml文件中提供了如下的参数来配置该缓存：
<ul>
<li>`indices.cache.filter.terms.size:`默认值为10mb，指定了ElasticSearch用于terms lookup的缓存的内存的最大容量。在绝大多数场景下，默认值已经足够，但是如果你知道你的将加载大量的数据到缓存，那么就需要增加该值。 </li>
<li>`indices.cache.filter.terms.expire_after_access:`该属性指定了缓存项最后一次访问到失效的最大时间。默认，该属性关闭，即永不失效。</li>
<li>`indices.cache.filter.terms.expire_after_write:`该属性指定了缓存项第一次写入到失效的最大时间。默认，该属性关闭，即永不失效。</li>
</ul>
