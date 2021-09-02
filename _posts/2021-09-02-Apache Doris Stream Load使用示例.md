---
layout: post
title: "Apache Doris Stream load使用方法及示例"
date: 2021-09-02
description: "Apache Doris Stream load使用方法及示例"
tag: Apache Doris
---

## Stream Load介绍

Stream load 是一个同步的导入方式，用户通过发送 HTTP 协议发送请求将本地文件或数据流导入到 Doris 中。Stream load 同步执行导入并返回导入结果。用户可直接通过请求的返回体判断本次导入是否成功。

Stream load 主要适用于导入本地文件，或通过程序导入数据流中的数据

具体的原理可以参照官网，这里只介绍怎么通过Java程序通过Stream load导入文件及数据流中的数据。

## Stream Load参数说明

Stream load 由于使用的是 HTTP 协议，所以所有导入任务有关的参数均设置在 **Header** 中，以下主要介绍部分参数意义

- label
  导入任务的标识。每个导入任务，都有一个在单 database 内部唯一的 label。label 是用户在导入命令中自定义的名称。通过这个 label，用户可以查看对应导入任务的执行情况。
  label 的另一个作用，是防止用户重复导入相同的数据。**强烈推荐用户同一批次数据使用相同的 label。这样同一批次数据的重复请求只会被接受一次，保证了 At-Most-Once**
  当 label 对应的导入作业状态为 CANCELLED 时，该 label 可以再次被使用。
- column_separator
  用于指定导入文件中的列分隔符，默认为\t。如果是不可见字符，则需要加\x作为前缀，使用十六进制来表示分隔符。
  如hive文件的分隔符\x01，需要指定为-H "column_separator:\x01"。
  可以使用多个字符的组合作为列分隔符。
- line_delimiter
  用于指定导入文件中的换行符，默认为\n。
  可以使用做多个字符的组合作为换行符。
- max_filter_ratio
  导入任务的最大容忍率，默认为0容忍，取值范围是0~1。当导入的错误率超过该值，则导入失败。
  如果用户希望忽略错误的行，可以通过设置这个参数大于 0，来保证导入可以成功。
  计算公式为：
  `(dpp.abnorm.ALL / (dpp.abnorm.ALL + dpp.norm.ALL ) ) > max_filter_ratio`
  `dpp.abnorm.ALL` 表示数据质量不合格的行数。如类型不匹配，列数不匹配，长度不匹配等等。
  `dpp.norm.ALL` 指的是导入过程中正确数据的条数。可以通过 `SHOW LOAD` 命令查询导入任务的正确数据量。
  原始文件的行数 = `dpp.abnorm.ALL + dpp.norm.ALL`
- where
  导入任务指定的过滤条件。Stream load 支持对原始数据指定 where 语句进行过滤。被过滤的数据将不会被导入，也不会参与 filter ratio 的计算，但会被计入`num_rows_unselected`。
- partition
  待导入表的 Partition 信息，如果待导入数据不属于指定的 Partition 则不会被导入。这些数据将计入 `dpp.abnorm.ALL`
- columns
  待导入数据的函数变换配置，目前 Stream load 支持的函数变换方法包含列的顺序变化以及表达式变换，其中表达式变换的方法与查询语句的一致。
  列顺序变换例子：原始数据有两列，目前表也有两列（c1,c2）但是原始文件的第一列对应的是目标表的c2列, 而原始文件的第二列对应的是目标表的c1列，则写法如下： columns: c2,c1 表达式变换例子：原始文件有两列，目标表也有两列（c1,c2）但是原始文件的两列均需要经过函数变换才能对应目标表的两列，则写法如下： columns: tmp_c1, tmp_c2, c1 = year(tmp_c1), c2 = month(tmp_c2) 其中 tmp_*是一个占位符，代表的是原始文件中的两个原始列。
- exec_mem_limit
  导入内存限制。默认为 2GB，单位为字节。
- strict_mode
  Stream load 导入可以开启 strict mode 模式。开启方式为在 HEADER 中声明 `strict_mode=true` 。默认的 strict mode 为关闭。
  strict mode 模式的意思是：对于导入过程中的列类型转换进行严格过滤。严格过滤的策略如下：

1. 对于列类型转换来说，如果 strict mode 为true，则错误的数据将被 filter。这里的错误数据是指：原始数据并不为空值，在参与列类型转换后结果为空值的这一类数据。
2. 对于导入的某列由函数变换生成时，strict mode 对其不产生影响。
3. 对于导入的某列类型包含范围限制的，如果原始数据能正常通过类型转换，但无法通过范围限制的，strict mode 对其也不产生影响。例如：如果类型是 decimal(1,0), 原始数据为 10，则属于可以通过类型转换但不在列声明的范围内。这种数据 strict 对其不产生影响。

- merge_type 数据的合并类型，一共支持三种类型APPEND、DELETE、MERGE 其中，APPEND是默认值，表示这批数据全部需要追加到现有数据中，DELETE 表示删除与这批数据key相同的所有行，MERGE 语义 需要与delete 条件联合使用，表示满足delete 条件的数据按照DELETE 语义处理其余的按照APPEND 语义处理

## Stream load 实例

Doris表结构：

```sql
CREATE TABLE `doris_test_sink` (
  `id` int NULL COMMENT "",
  `number` int NULL COMMENT "",
  `price` DECIMAL(12,2) NULL COMMENT "",
  `skuname` varchar(40) NULL COMMENT "",
  `skudesc` varchar(200) NULL COMMENT ""
) ENGINE=OLAP
DUPLICATE KEY(`id`)
COMMENT "OLAP"
DISTRIBUTED BY HASH(`id`) BUCKETS 1
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "V2"
);
```

实例文本数据：

```text
10001,12,13.3, test1,this is atest
10002,100,15.3,test2,this is atest
10003,102,16.3,test3,this is atest
10004,120,17.3,test4,this is atest
10005,23,10.3, test5,this is atest
10006,24,112.3,test6,this is atest
10007,26,13.3, test7,this is atest
10008,29,145.3,test8,this is atest
10009,30,16.3, test9,this is atest
100010,32,18.3,test10,this is atest
100011,52,18.3,test11,this is atest
100012,62,10.3,test12,this is atest
```

JSON格式实例数据：

```json
{    
    "id":556393582,
    "number":"123344",
    "price":"23.5",
    "skuname":"test",
    "skudesc":"zhangfeng_test,test"
}
```



Java示例代码：

```java
import org.apache.commons.codec.binary.Base64;
import org.apache.http.HttpHeaders;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPut;
import org.apache.http.entity.FileEntity;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.DefaultRedirectStrategy;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import java.io.File;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.UUID;


/**
 * This example mainly demonstrates how to use stream load to import data
 * Including file type (CSV) and data in JSON format
 *
 */
public class DorisStreamLoader {
    // FE IP Address
    private final static String HOST = "10.220.146.10";
    // FE port
    private final static int PORT = 8030;
    // db name
    private final static String DATABASE = "test_2";
    // table name
    private final static String TABLE = "doris_test_sink";
    //user name
    private final static String USER = "root";
    //user password
    private final static String PASSWD = "";
    //The path of the local file to be imported
    private final static String LOAD_FILE_NAME = "c:/es/1.csv";

    //http path of stream load task submission
    private final static String loadUrl = String.format("http://%s:%s/api/%s/%s/_stream_load",
            HOST, PORT, DATABASE, TABLE);

    //Build http client builder
    private final static HttpClientBuilder httpClientBuilder = HttpClients
            .custom()
            .setRedirectStrategy(new DefaultRedirectStrategy() {
                @Override
                protected boolean isRedirectable(String method) {
                    // If the connection target is FE, you need to deal with 307 redirect。
                    return true;
                }
            });

    /**
     * File import
     * @param file
     * @throws Exception
     */
    public void load(File file) throws Exception {
        try (CloseableHttpClient client = httpClientBuilder.build()) {
            HttpPut put = new HttpPut(loadUrl);
            put.removeHeaders(HttpHeaders.CONTENT_LENGTH);
            put.removeHeaders(HttpHeaders.TRANSFER_ENCODING);
            put.setHeader(HttpHeaders.EXPECT, "100-continue");
            put.setHeader(HttpHeaders.AUTHORIZATION, basicAuthHeader(USER, PASSWD));

            // You can set stream load related properties in the Header, here we set label and column_separator.
            put.setHeader("label", UUID.randomUUID().toString());
            put.setHeader("column_separator", ",");

            // Set up the import file. Here you can also use StringEntity to transfer arbitrary data.
            FileEntity entity = new FileEntity(file);
            put.setEntity(entity);

            try (CloseableHttpResponse response = client.execute(put)) {
                String loadResult = "";
                if (response.getEntity() != null) {
                    loadResult = EntityUtils.toString(response.getEntity());
                }

                final int statusCode = response.getStatusLine().getStatusCode();
                if (statusCode != 200) {
                    throw new IOException(String.format("Stream load failed. status: %s load result: %s", statusCode, loadResult));
                }
                System.out.println("Get load result: " + loadResult);
            }
        }
    }

    /**
     * JSON import
     * @param jsonData
     * @throws Exception
     */
    public void loadJson(String jsonData) throws Exception {
        try (CloseableHttpClient client = httpClientBuilder.build()) {
            HttpPut put = new HttpPut(loadUrl);
            put.removeHeaders(HttpHeaders.CONTENT_LENGTH);
            put.removeHeaders(HttpHeaders.TRANSFER_ENCODING);
            put.setHeader(HttpHeaders.EXPECT, "100-continue");
            put.setHeader(HttpHeaders.AUTHORIZATION, basicAuthHeader(USER, PASSWD));

            // You can set stream load related properties in the Header, here we set label and column_separator.
            put.setHeader("label", UUID.randomUUID().toString());
            put.setHeader("column_separator", ",");
            put.setHeader("format", "json");

            // Set up the import file. Here you can also use StringEntity to transfer arbitrary data.
            StringEntity entity = new StringEntity(jsonData);
            put.setEntity(entity);

            try (CloseableHttpResponse response = client.execute(put)) {
                String loadResult = "";
                if (response.getEntity() != null) {
                    loadResult = EntityUtils.toString(response.getEntity());
                }

                final int statusCode = response.getStatusLine().getStatusCode();
                if (statusCode != 200) {
                    throw new IOException(String.format("Stream load failed. status: %s load result: %s", statusCode, loadResult));
                }
                System.out.println("Get load result: " + loadResult);
            }
        }
    }

    /**
     * Construct authentication information, the authentication method used by doris here is Basic Auth
     * @param username
     * @param password
     * @return
     */
    private String basicAuthHeader(String username, String password) {
        final String tobeEncode = username + ":" + password;
        byte[] encoded = Base64.encodeBase64(tobeEncode.getBytes(StandardCharsets.UTF_8));
        return "Basic " + new String(encoded);
    }


    public static void main(String[] args) throws Exception {
        DorisStreamLoader loader = new DorisStreamLoader();
        //file load
        //File file = new File(LOAD_FILE_NAME);
        //loader.load(file);
        //json load
        String jsonData = "{\"id\":556393582,\"number\":\"123344\",\"price\":\"23.5\",\"skuname\":\"test\",\"skudesc\":\"zhangfeng_test,test\"}";
        loader.loadJson(jsonData);

    }
}
```



具体可以参照我的Github：

[github.com/hf200012/incubator-doris/tree/demo-streamload/samples/doris-demo/stream-load-demo](https://link.zhihu.com/?target=https%3A//github.com/hf200012/incubator-doris/tree/demo-streamload/samples/doris-demo/stream-load-demo)