你好, 我叫陈野, 毕业于天津工业大学软件工程。19年9月加入上海虎扑，主要从事识货app的研发。负责的板块有

识货app启动的时候关于设备信息的收集。识货会在抖音，快手，微博，腾讯广点通等第三方广告主上进行流量投放，当用户在外部平台上刷到识货App，点击下载的时候，外部平台会把这部分用户的一些基础信息回传给识货，比如idfa，imei，oaid，ip，ua 等，我们对这部分信息进行持久化存储。当用户打开识货app的时候，我们拿外部平台回传的信息和当前设备信息进行匹配，如果满足条件，再经过风控，设备质量等条件的筛选， 判断是否是投放带来的流量增长。我们会实时的统计这部分用户启动事件，老用户的复购事件,下单事件，并把这部分数据回传给三方平台，方便三方平台进行投放人群的修整和同事投放策略的改变。

识货首页，个人中心，分类页的开发。首页是对多个模块信息的聚合。 其中主要包括顶部金刚位的策略处理。会根据用户的地理位置，类目的喜好，点击行为进行位置的调换和类型的替换。还有其他平台在识货上的广告投放和站内活动入口对应策略的编写。还有商品信息流：主要依赖三方推荐， 包括阿里，头条，自研推荐对用户行为，性别，订单收集，推荐出合适的商品，通过调用商品库服务组装商品数据，展示给用户。

商详页的口碑产出。识货的商品口碑都来源第三方app。我们根据渠道组投递给我们的sku 渠道信息，去第三方平台上爬取评论，经过风控，脱敏等信息处理，持久化保存到db中，同时同步到es中，对于es 的修改主要是基于binglog 实现，这部分数据也需要投递给算法组，他们会根据评论内容提取出关键词和对应语句返回给我们，我们再组装到口碑表记录中。前台api 接口主要是查询es，通过标签聚合我们可以进行评论的筛选。同时因为每条评论来源于一个渠道，一个渠道来源于一个商家，我们可以对口碑数据进行商家维度的聚合，方便运营通过口碑的质量筛选商家，其中涉及的维度还有追评初评，发货问题，质量问题等。

百度，华为的合作。背景是通过搜索入口，展示识货商品， 帮助用户进行商品决策。我们会提供商品物料信息，sku，spu，skc 维度的口碑，评测，排行榜等信息，渠道购买信息，在第三方平台构建一个更轻量的识货。

5.返现。返现目前主要是用来拉新用户，带动用户开识货会员的积极性。通过和美团，饿了么合作。用户从识货点击出站，通过订单追踪，给这部分用户进行一定比例的返现，主要包含会员周期，返现比例，返现公众号，分享返现的开发。
