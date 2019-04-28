---

目前对于主要是针对几个重点接口进行分析,其他接口都目前问题不大没有写到文档中


---

## 容易出问题的几个接口

### 未拿货市场列表(api/shop/willlist)

首先是两个复杂的sql,首先是对OrderWsyShop
之后是对OrderWsyGoods继续查询

之后对每一个档口进行访问,在循环内获取档口对应的market.

建议.对于获取市场信息的接口进行合并加载.另外对于两个复杂sql进行调优.

### 拿货付款接口(api/goods/pay)
这个接口的核心是对于每一个传入的商品参数进行付款.

获取对应的订单商品,开始对订单商品进行循环.每一个订单商品

检查商品状态,无效者打入日志,但是继续循环.

计算当前的转账金额

当折扣为正的下,**添加财务中心记录 addOrderAndCreateLog**

调用函数开始转账

**在这里会设立两个锁,档口的金额锁,和平台的金额锁,分别增减对应用户的账户金额 PlatformFundsUser之后会添加日志** -- 这个之前应该把锁去掉了.

在商品过多的情况下回出现加载很慢的情况.

这里我建议吧.能否考虑合并订单商品信息获取与,日志打印,账户金额的合并计算,只对同一个用户的金额进行一次update.因为在实际的一次操作中,一般是同一个商户和网商园的账户,两个账户进行更新,合并统计是否更为合理?


### 退货签收列表 (api/return-goods/get-return-good-list)
获取对应的退货的商品(OrderWsyReturnGoods)列表

循环内,**查询对应的订单商品(OrderWsyGoods) 退货订单(OrderWsyReturn)** 最后在对于,**管理对应的退款操作日志关联数据(OrderWsyReturnOptionLog)**,前面发现主要的问题在于日志表缺少索引,在测试环境下已经把反馈时间缩小到原来的1/10.

## 其他接口

### 档口信息与当前档口的未拿货列表 api/goods/willlist

获取对应档口的信息(Shop),在获取档口下未发货商品的信息(OrderWsyGoods)


### 档口退货商品列表 (/api/return-goods/get-return-good-list)

先获取历史未签收退货订单 OrderWsyReturnOptionLog, 中获取各个当前店铺下的退货商品信息并返回

### 确认退货接口 api/return-goods/sure-return-goods
根据退货的类型进行检查.
获取对应的需要退货的商品列表.对于商品列表进行循环.计算对应的订单金额.添加退货日志(OrderWsyReturnOptionLog).根据是否有折扣,添加财务中心日志(ProductOrderToday).之后调用转账.这个与拿货付款接口的逻辑调用函数相同.
