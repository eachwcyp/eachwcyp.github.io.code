### 优化点
#### 管理后台
#####  
1. 用户管理里面，下拉用户的选择冻结的时候，用户没有冻结
2. 所有涉及状态或者（1|0）的字段，在列表页面，都通过颜色区分一下
3. 配送中心->用户自提点，页面展示增加deliveryWay的展示（10|11），用户地址用于物流发货，还是线下自取
4. 消费者表上面，增加isdev ， 如果dev,展示所有产品功能和监控信息
5. 


#####
1. 退货， 需要有查看按钮，如果用户填写了物流信息，需要展示物流信息
2. 退款
3. 退货退款后，需要展示物流信息，商家可以查看
4. 记录用户的售后行为和退款操作行为
5. 后台30日，导入员工基础数据

#### 小程序
1. 我的评价界面，已评价没有数据
2. 差一个评价详情页面  



#### 后续优化点
1. 订单明细界面，如果没有规格，不用空着，显示单品就可以
2. 打通腾讯的物流发货和收货接口
3. 同步腾讯的物流商（ems|顺丰），快递100的物流商，基础信息到系统中
4. https://lbs.qq.com/service/webService/webServiceGuide/webServiceIp  限制用户登录的IP登录，限制登录区域
5. http://ip.soshoulu.com/ipsection_view.aspx?fl=2&type=s_BJ
###  知识积累

https://zhuanlan.zhihu.com/p/180204695
ALTER TABLE `bcs_user_info` ADD UNIQUE(`open_id`);

### 租户处理
http://www.hqwc.cn/news/218409.html


###  客服环境

PC链接：
https://chatbot.weixin.qq.com/webapp/j4A6vsN68pAwpn7f8mdnR9KeW31810?robotName=%E9%87%91%E4%BF%A1%E6%96%87%E5%88%9B