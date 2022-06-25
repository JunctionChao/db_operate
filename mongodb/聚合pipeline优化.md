### Aggregation Pipeline Optimization(based on mongodb Version 4.4)

聚合管道操作有一个优化阶段，可以重塑pipeline以提高执行性能效率

#### Projection Optimization  投影（映射）优化

如果只需要文档中某些字段才能获得结果，$project 管道操作将只使用那些所需的字段，从而减少通过管道的数据量
#### Pipeline Sequence Optimization  **(Pipeline 顺序优化)**

- **(`$project` or `$unset` or `$addFields` or `$set`) + `$match` Sequence Optimization**

  如果聚合管道包含多个投影和/或$match阶段，MongoDB会对每个$match阶段执行此优化，将每个$match过滤器移动到该过滤器不依赖的所有投影阶段之前

  考虑下面一个多阶段的pipeline

  ```javascript
  { $addFields: {
      maxTime: { $max: "$times" },
      minTime: { $min: "$times" }
  } },
  { $project: {
      _id: 1, name: 1, times: 1, maxTime: 1, minTime: 1,
      avgTime: { $avg: ["$maxTime", "$minTime"] }
  } },
  { $match: {
      name: "Joe Schmoe",
      maxTime: { $lt: 20 },
      minTime: { $gt: 5 },
      avgTime: { $gt: 7 }
  } }
  ```
  优化器将$match阶段分解成四个独立的过滤器。然后，优化器将每个过滤器移动到尽可能多的投影阶段之前，根据需要创建新的$match阶段。对于上面的例子，优化器产生了以下优化后的pipeline:

  ```javascript
  { $match: { name: "Joe Schmoe" } },
  { $addFields: {
      maxTime: { $max: "$times" },
      minTime: { $min: "$times" }
  } },
  { $match: { maxTime: { $lt: 20 }, minTime: { $gt: 5 } } },
  { $project: {
      _id: 1, name: 1, times: 1, maxTime: 1, minTime: 1,
      avgTime: { $avg: ["$maxTime", "$minTime"] }
  } },
  { $match: { avgTime: { $gt: 7 } } }
  ```

- **`$sort` + `$match` Sequence Optimization**

  当有一个带有$Sort的序列，然后是$Match时，$Match将会移动到$Sort之前，以最小化要排序的对象的数量
  
  考虑下面的例子：
  
  ```
  { $sort: { age : -1 } },
  { $match: { status: 'A' } }
  ```
  
  在优化阶段，优化器将序列转换为：
  
  ```
  { $match: { status: 'A' } },
  { $sort: { age : -1 } }
  ```

- **`$redact` + `$match` Sequence Optimization**

  当管道的$redact阶段紧跟$Match阶段时，聚合有时可以在$redact阶段之前添加$Match阶段的一部分。如果添加的$Match阶段位于管道的开头，则聚合可以使用索引并查询集合以限制进入管道的文档数量
  
  考虑下面的例子：
  
  ```
  { $redact: { $cond: { if: { $eq: [ "$level", 5 ] }, then: "$$PRUNE", else: "$$DESCEND" } } },
  { $match: { year: 2014, category: { $ne: "Z" } } }
  ```
  
  优化器可以在$redact阶段之前添加相同的$Match阶段：
  
  ```
  { $match: { year: 2014 } },
  { $redact: { $cond: { if: { $eq: [ "$level", 5 ] }, then: "$$PRUNE", else: "$$DESCEND" } } },
  { $match: { year: 2014, category: { $ne: "Z" } } }
  ```

- **`$project`/`$unset` + `$skip` Sequence Optimization**

  当您有一个带有$project的序列或$unset后面跟着$skip的序列时，$skip将移动到$project之前。

  ```
  { $sort: { age : -1 } },
  { $project: { status: 1, name: 1 } },
  { $skip: 5 }
  ```

  在优化阶段，优化器将序列转换为：

  ```
  { $sort: { age : -1 } },
  { $skip: 5 },
  { $project: { status: 1, name: 1 } }
  ```

#### Pipeline Coalescence Optimization （Pipeline 合并优化）

在可能的情况下，优化阶段会将一个流水线阶段合并到它的前一个阶段。一般来说，合并发生在任何序列重新排序优化之后。

- **`$sort` + `$limit` Coalescence**

  当$sort在$limit之前时，如果中间没有修改文档数量的阶段（如$unwind，$group），优化器可以将$limit合并到$sort中。如果在$sort和$limit阶段之间有改变文档数量的管道阶段，MongoDB将不会把$limit合并到$sort中。

  例如，如果管道由以下阶段组成：

  ```
  { $sort : { age : -1 } },
  { $project : { age : 1, status : 1, name : 1 } },
  { $limit: 5 }
  ```

  在优化阶段，优化器将序列合并为：

  ```
  {
      "$sort" : {
         "sortKey" : {
            "age" : -1
         },
         "limit" : NumberLong(5)
      }
  },
  { "$project" : {
           "age" : 1,
           "status" : 1,
           "name" : 1
    }
  }
  ```

- **`$limit` + `$limit` Coalescence**

  当一个$limit紧接着另一个$limit时，两个阶段可以合并成一个$limit，其中limit的数量是两个初始limit数量中较小的一个。

  例如，一个管道包含以下序列：

  ```
  { $limit: 100 },
  { $limit: 10 }
  ```

  优化后合并为：

  ```
  { $limit: 10 }
  ```

- **`$skip` + `$skip` Coalescence**

  ```
  { $skip: 5 },
  { $skip: 2 }
  ```

  优化后合并为：

  ```
  { $skip: 7 }
  ```

- **`$match` + `$match` Coalescence**

  ```
  { $match: { year: 2014 } },
  { $match: { status: "A" } }
  ```

  优化后合并为：

  ```
  { $match: { $and: [ { "year" : 2014 }, { "status" : "A" } ] } }
  ```

- **`$lookup` + `$unwind` Coalescence**

  当一个$unwind紧随另一个$lookup之后，并且$unwind对$lookup的as字段进行操作时，优化器可以将$unwind凝聚到$lookup阶段。这样可以避免创建大的中间文件。

  ```
  {
    $lookup: {
      from: "otherCollection",
      as: "resultingArray",
      localField: "x",
      foreignField: "y"
    }
  },
  { $unwind: "$resultingArray"}
  ```

  优化后：

  ```
  {
    $lookup: {
      from: "otherCollection",
      as: "resultingArray",
      localField: "x",
      foreignField: "y",
      unwinding: { preserveNullAndEmptyArrays: false }
    }
  }
  ```