### 区域碰撞

#### 方案介绍

1、主要使用bitmap按位存储和计算的特殊数据结构来计算，与之相对应还有bitset、布隆过滤器（bloomfilter）、HyperLogLog(HLL)等算法，在大数据领域有大量应用。

2、利用es集群的高性能，编写bitmap+es的插件，完成对某个区域快速精确去重，节省存储空间和计算时间。

3、使用es的multsearch支持多区域的搜索，加快多次查询速度

4、es中需要提前将车牌号 转为long类型



kibana查询语句：

```json

POST vehicletrajectoryentity/_search
{
  "aggs": {
    "umxbitmap": {
      "umxbitmap": {
        "field": "xxxx"
      }
    }
  }
}
```





注意：工程中需要引入插件包：elasticsearchplugin-1.0-SNAPSHOT.jar，包含BitmapAggregationBuilder、BitmapUtil工具包等

client处理逻辑：

```java

public class CollisionCarDetectQueryByES {

    public static void main(String[] args) throws IOException {

        /**
         * 创建es的集群服务器地址
         */
        List<HttpHost> httpHosts = new ArrayList<>();
        httpHosts.add(new HttpHost("10.116.200.21", 9200, "http"));
        httpHosts.add(new HttpHost("10.116.200.22", 9200, "http"));
        httpHosts.add(new HttpHost("10.116.200.23", 9200, "http"));
        httpHosts.add(new HttpHost("10.116.200.25", 9200, "http"));
        httpHosts.add(new HttpHost("10.116.200.26", 9200, "http"));
//        httpHosts.add(new HttpHost("10.116.200.24", 9200, "http"));

        RestHighLevelClient client =
                new RestHighLevelClient(RestClient.builder(httpHosts.toArray(new HttpHost[httpHosts.size()])));

        /**
         * 构建多个查询参数：1、时间范围的指定；2、设备id的指定（已省略）
         * 可以构建五个区域的查询条件，具体构建逻辑有业务决定
         */
        long start1 = 1609431788104L; // time start in mills
        long end1 = 1611222777174L; // time end in mills
        QueryBuilder query1 = QueryBuilders.boolQuery()
                .must(QueryBuilders.rangeQuery("shotTime").gte(start1).lte(end1));


        long start2 = 1609431788104L; // time start in mills
        long end2 = 1611222777174L; // time end in mills
        QueryBuilder query2 = QueryBuilders.boolQuery()
                .must(QueryBuilders.rangeQuery("shotTime").gte(start2).lte(end2));


        List<QueryBuilder> queryBuilderList = new ArrayList<>();
        queryBuilderList.add(query1);
        queryBuilderList.add(query2);

        /**
         * TODO 构建 MultiSearchRequest参数
         */
        MultiSearchRequest request = new MultiSearchRequest();
        for (QueryBuilder queryBuilder : queryBuilderList
        ) {
            SearchSourceBuilder ssb = new SearchSourceBuilder();
            /**
             * bitmap聚合器使用方法，直接new一个builder即可。
             * “bitmapagg” 给聚合器一个别名,解析结果会用到
             */
            BitmapAggregationBuilder bitmapAggregationBuilder = new BitmapAggregationBuilder("bitmapagg");
            /**
             * 指定es_doc中long 的车牌号，前提需要将 “湘A123456(string)” <==> long,且唯一；
             * “shotTime”字段是示例，shotTime在es存的是long类型的
             */
            bitmapAggregationBuilder.field("shotTime");
            ssb.size(0);
            /**
             * 把bitmap添加到查询中
             */
            ssb.aggregation(bitmapAggregationBuilder);
            ssb.query(queryBuilder);
            System.out.println("ES_DSL语句:" + ssb.toString());
            SearchRequest searchRequest = new SearchRequest();
            searchRequest.source(ssb);
            //各种index: vehicletrajectoryentity carspeedindex carws car
            searchRequest.indices("vehicletrajectoryentity"); //index name
            request.add(searchRequest);
        }

        long currentTime = System.currentTimeMillis();
        /**
         * 开始mult搜索
         */
        MultiSearchResponse multiSearchResponse = client.msearch(request, RequestOptions.DEFAULT);

        MultiSearchResponse.Item[] responses = multiSearchResponse.getResponses();

        Roaring64Bitmap sumBitmap = new Roaring64Bitmap();

        /**
         * 解析es的mult返回结果
         */
        for (int i = 0; i < responses.length; i++) {
            MultiSearchResponse.Item response = responses[i];
            if (null != response.getFailure()) {
                String error = response.getFailure().getClass().getSimpleName() + ": " + response.getFailure().getMessage();
                System.out.println(error);
            } else {
                SearchResponse searchResponse = response.getResponse();
                Aggregations aggRes = searchResponse.getAggregations();
                /**
                 * 得到bitmapagg聚合器的结果
                 */
                ParsedBitmap aggResTerm = aggRes.get("bitmapagg");
                long took = searchResponse.getTook().getMillis();
                System.out.println("每次es查询耗时：" + took + " ms");

                /**
                 * 将es返回byte[] 转为bitmap结果
                 */
                Roaring64Bitmap roaring64Bitmap = BitmapUtil.deserializeBitmap(aggResTerm.getBitmapByte());

                System.out.println("每个区域去重后的size:" + roaring64Bitmap.getLongCardinality() + " 条,占用内存:" + getNetFileSizeDescription(aggResTerm.getBitmapByte().length));
                if (null == sumBitmap || sumBitmap.getLongCardinality() == 0) {
                    sumBitmap = roaring64Bitmap;
                } else {
                    /**
                     *开始对bitmap反复求交集，最终的交集结果存储在sumBitmap
                     */
                    sumBitmap.and(roaring64Bitmap);
                }
            }
        }

        System.out.println("求交集之后的size:" + sumBitmap.getLongCardinality() + ",求交集总耗时：" + (System.currentTimeMillis() - currentTime) + " ms");


//        List<Long> longList=new ArrayList<>();
//        sumBitmap.forEach(new LongConsumer() {
//            @Override
//            public void accept(long value) {
//                /**
//                 * 循环取想要的数量即可，省略...
//                 */
//                longList.add(value);
//            }
//        });
//        System.out.println("获得最终结果数量："+longList.size());
//

        client.close();
    }

    /**
     * 存储单位转换
     *
     * @param size
     * @return
     */
    public static String getNetFileSizeDescription(long size) {
        StringBuffer bytes = new StringBuffer();
        DecimalFormat format = new DecimalFormat("###.0");
        if (size >= 1024 * 1024 * 1024) {
            double i = (size / (1024.0 * 1024.0 * 1024.0));
            bytes.append(format.format(i)).append("GB");
        } else if (size >= 1024 * 1024) {
            double i = (size / (1024.0 * 1024.0));
            bytes.append(format.format(i)).append("MB");
        } else if (size >= 1024) {
            double i = (size / (1024.0));
            bytes.append(format.format(i)).append("KB");
        } else if (size < 1024) {
            if (size <= 0) {
                bytes.append("0B");
            } else {
                bytes.append((int) size).append("B");
            }
        }
        return bytes.toString();
    }
}

```

bitmap计算结果：

```powershell
ES_DSL语句:{"size":0,"query":{"bool":{"must":[{"range":{"shotTime":{"from":1609431788104,"to":1611222777174,"include_lower":true,"include_upper":true,"boost":1.0}}}],"adjust_pure_negative":true,"boost":1.0}},"aggregations":{"bitmapagg":{"umxbitmap":{"field":"shotTime"}}}}
ES_DSL语句:{"size":0,"query":{"bool":{"must":[{"range":{"shotTime":{"from":1609431788104,"to":1611222777174,"include_lower":true,"include_upper":true,"boost":1.0}}}],"adjust_pure_negative":true,"boost":1.0}},"aggregations":{"bitmapagg":{"umxbitmap":{"field":"shotTime"}}}}
每次es查询耗时：636 ms
每个区域去重后的size:3458810 条,占用内存:7.2MB
每次es查询耗时：636 ms
每个区域去重后的size:3458810 条,占用内存:7.2MB
求交集之后的size:3458810,求交集总耗时：2728 ms
```





插件安装：需要在es集群每台服务器中 进行安装

```shell
#查看plugin
/usr/share/elasticsearch/bin/elasticsearch-plugin list

#删除插件
/usr/share/elasticsearch/bin/elasticsearch-plugin remove bitmap-aggregation-plugin
#安装插件
/usr/share/elasticsearch/bin/elasticsearch-plugin  install "file:///mnt/data1/elasticsearchplugin-1.0-SNAPSHOT.zip"
```





