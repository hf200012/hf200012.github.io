---
layout: post
title: "Apache Doris 删除数据恢复"
date: 2021-11-03
description: "Apache Doris 删除数据恢复"
tag: Apache Doris
---
# Apache Doris 删除数据恢复

Apache Doris为了避免误操作造成的灾难，支持对误删除的数据库/表/分区进行数据恢复，在`drop table`或者 `drop database`之后，Doris不会立刻对数据进行物理删除，而是在 Trash 中保留一段时间（默认1天），管理员可以通过RECOVER命令对误删除的数据进行恢复

## 1.数据恢复命令

```sql
## 恢复 database
RECOVER DATABASE db_name;
## 恢复 table
RECOVER TABLE [db_name.]table_name;
## 恢复 partition
RECOVER PARTITION partition_name FROM [db_name.]table_name;
```

具体用法可以使用`help recover` 来查看

>说明：
     1. 该操作仅能恢复之前一段时间内删除的元信息。默认为 1 天。（可通过fe.conf中`catalog_trash_expire_second`参数配置）

  2. 如果删除元信息后新建立了同名同类型的元信息，则之前删除的元信息不能被恢复



## 2.使用示例

 1. 恢复名为 example_db 的 database
    
      ```sql
      RECOVER DATABASE example_db;
      ```
      
2. 恢复名为 example_tbl 的 table

   ```sql
   RECOVER TABLE example_db.example_tbl;
   ```
   
3. 恢复表 example_tbl 中名为 p1 的 partition
   
   ```sql
     RECOVER PARTITION p1 FROM example_tbl;
   ```
   
   
