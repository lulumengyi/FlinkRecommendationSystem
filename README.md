# FlinkRecommendationSystem

本系统主要实现了从0到1的流音乐搜索推荐系统服务。主要包含搜索模块，用户画像模块，推荐模块（召回，排序）。具体实现关键点如下：

- 使用ElasticSearch搭建搜索引擎，将MySQL的数据导入至ES中，并设计ES的mapping和setting文件，优化ES的检索性能，目前服务稳定运行，百万级的数据查询100条在毫秒级可以返回。
- 通过Kafka监听用户行为日志,使用Flink CEP实时建模解析如点播、完听、切歌等用户行为，使用Flink ValueState计算用户听歌时长，并将行为采集至Hbase数据库中，使用TF-IDF计算内容标签权重构建物品画像。
- 基于用户基本信息和用户行为信息构建用户画像，标签权重随时间衰减，使用redis zset存储用户的短期画像，解析用户行为日志，实时增加内容的完听次数等信息，更新热门内容。
- 构建基于热度召回，基于协同过滤的召回模型及基于画像使用K-means聚类算法对内容相似召回模型。
- 负责推荐排序算法设计和优化，使用GBDT+LR模型进行排序，离线AUC指标0.79。使用Wide & Deep模型优化。

### 1. 系统架构 

- **1.1 系统架构图**

![system_structure](/Users/lumengyi/Downloads/flink-recommandSystem-demo-master/resources/pic/system_structure.png)



## **2.模块说明**

- ### **一. 搜索模块**

从NLU模块接受到用户指令，如”请播放周杰伦的歌曲“， 在搜索模块将会以artist=‘周杰伦’在mysql数据中进行查找。

mysql数据库中以建立了以artists，song，genre，language等10个特征分类的倒排索引。

![image-20231204213027883](/Users/lumengyi/Downloads/flink-recommandSystem-demo-master/resources/pic/image-20231204213027883.png)

- ### **二. Flink模块**

  a.在日志数据模块(flink-2-hbase)中,又主要分为6个Flink任务:

  - 用户-产品浏览历史  -> 实现基于协同过滤的推荐逻辑 

    通过Flink去记录用户浏览过这个类目下的哪些产品,为后面的基于Item的协同过滤做准备
    实时的记录用户的评分到Hbase中,为后续离线处理做准备.

    数据存储在Hbase的p_history表

  - 用户-兴趣 -> 实现基于上下文的推荐逻辑

    根据用户对同一个产品的操作计算兴趣度,计算规则通过操作间隔时间(如购物 - 浏览 < 100s)则判定为一次兴趣事件
    通过Flink的ValueState实现,如果用户的操作Action=3(收藏),则清除这个产品的state,如果超过100s没有出现Action=3的事件,也会清除这个state

    数据存储在Hbase的u_interest表

  - 用户画像计算 -> 实现基于标签的推荐逻辑

    v1.0按照三个维度去计算用户画像,分别是用户的颜色兴趣,用户的产地兴趣,和用户的风格兴趣.根据日志不断的修改用户画像的数据,记录在Hbase中.

    数据存储在Hbase的user表

  - 产品画像记录  -> 实现基于标签的推荐逻辑

    用两个维度记录产品画像,一个是喜爱该产品的年龄段,另一个是性别

    数据存储在Hbase的prod表

  - 事实热度榜 -> 实现基于热度的推荐逻辑 

    通过Flink时间窗口机制,统计当前时间的实时热度,并将数据缓存在Redis中.

    通过Flink的窗口机制计算实时热度,使用ListState保存一次热度榜

    数据存储在redis中,按照时间戳存储list

  - 日志导入

    从Kafka接收的数据直接导入进Hbase事实表,保存完整的日志log,日志中包含了用户Id,用户操作的产品id,操作时间,行为(如购买,点击,推荐等).

    数据按时间窗口统计数据大屏需要的数据,返回前段展示

    数据存储在Hbase的con表

- b. web模块

  - 前台用户界面

    该页面返回给用户推荐的产品list

  - 后台监控页面

    该页面返回给管理员指标监控



- ###  **三.推荐引擎模块**

#### **1.召回阶段**

- **3.1.1 基于热度的召回逻辑**

  现阶段召回逻辑图

[![img](https://github.com/will-che/flink-recommandSystem-demo/raw/master/resources/pic/v2.0%E7%94%A8%E6%88%B7%E6%8E%A8%E8%8D%90%E6%B5%81%E7%A8%8B.png)](https://github.com/will-che/flink-recommandSystem-demo/blob/master/resources/pic/v2.0用户推荐流程.png)

 根据用户特征，重新排序热度榜，之后根据两种推荐算法计算得到的产品相关度评分，为每个热度榜中的产品推荐几个关联的产品

- **3.1.2 基于产品画像的产品相似度计算方法**

  基于产品画像的召回逻辑依赖于产品画像和热度榜两个维度,产品画像有三个特征,包含color/country/style三个角度,通过计算用户对该类目产品的评分来过滤热度榜上的产品

  [![img](https://github.com/will-che/flink-recommandSystem-demo/raw/master/resources/pic/%E5%9F%BA%E4%BA%8E%E4%BA%A7%E5%93%81%E7%94%BB%E5%83%8F%E7%9A%84%E6%8E%A8%E8%8D%90%E9%80%BB%E8%BE%91.png)](https://github.com/will-che/flink-recommandSystem-demo/blob/master/resources/pic/基于产品画像的推荐逻辑.png)

  在已经有产品画像的基础上,计算item与item之间的关联系,通过**余弦相似度**来计算两两之间的评分,最后在已有物品选中的情况下推荐关联性更高的产品.

| 相似度 | A    | B    | C    |
| ------ | ---- | ---- | ---- |
| A      | 1    | 0.7  | 0.2  |
| B      | 0.7  | 1    | 0.6  |
| C      | 0.2  | 0.6  | 1    |

- **3.1.3 基于协同过滤的产品相似度计算方法**

  根据产品用户表（Hbase） 去计算公式得到相似度评分：

  ![img](https://github.com/will-che/flink-recommandSystem-demo/raw/master/resources/pic/%E5%9F%BA%E4%BA%8E%E7%89%A9%E5%93%81%E7%9A%84%E5%8D%8F%E5%90%8C%E8%BF%87%E6%BB%A4%E5%85%AC%E5%BC%8F.svg)

#### 2.排序阶段

召回后的list输入到排序模型中，对特征输入至模型中。

解析标签获取用户与物品训练数据-- gen_sampes.py

排序模型rankmodel
思路：为GBDT逻辑回归准备样本数据
1、取第一步merge_base.data,得到用户画像数据，用户信息数据，标签数据
2、收取样本标签，用户画像信息，物品信息
3、抽取用户画像信息，对性别和年龄生成样本数据
4、抽取物品特征信息，分词获得token，score，做样本数据
5、拼接样本，生成最终的样本信息，作为模型进行训练

模型加载 --gbdt_lr.py 思路：这里我们要用到我们的数据，就需要我们自己写load_data的部分， 首先定义main方法入口，编写load_data其次调用该方法的到x，y训练和测试集，最后输出wegiht，和b偏置

四、推荐流程阶段--main.py
step1.解析请求userid、itemid.
step2.加载模型w,b.
step3.检索redis获得候选集.
step4 精排前获取用户特征数据.
step5：获取物品特征item_feature.data.
step6:排序
step7:过滤









