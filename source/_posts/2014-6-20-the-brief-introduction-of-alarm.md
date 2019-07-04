---
title: ceilometer中alarm浅析
date: 2014/6/20
---
## 1 对象模型

一个alarm在ceilometer中定义为对一种计量统计值的超过一定条件的告警，或者若干个告警的逻辑（与、或）组合。

alarm的对象模型如下：

* alarm_id：alarm的id（uuid）
* type：告警类型，分为threshold类型和combination类型，分别对应于基于门槛的告警和已有告警逻辑组合成的新告警
* name：告警名称。
* description：告警描述，若创建时未指定会自动生成关于该告警触发条件的描述。
* enabled：标识告警是否激活。
* state：告警状态，包括三种：ok，表示监控值正常；alarm，表示触发告警；insufficient data表示告警状态未知。
* rule:告警触发规则，对于threshold类型告警，包括触发门槛值、计量对象、计量统计周期、统计类型（平均值、最大值、最小值等）、统计周期、查询统计数据查询条件、比较运算符（>,=等）、排除异常值；对于combination类型的告警，包括一个告警id的列表，以及逻辑运算符（and和or）。
* user_id：user id，创建时可指定。
* project_id：租户id，创建时可指定。
* evaluation_periods：评估次数，创建时可指定。
* period：上述rule中的查询统计数据的周期。
* timestamp：该alarm的上一次更新时间。
* state_timestamp：上一个状态更新的时间。
* ok_actions：一个webhooks的列表，当告警处于ok状态时调用。
* alarm_actions：一个webhooks的列表，当告警触发的时候（处于alarm状态）调用。
* insufficient_data_actions：一个webhooks的列表，当告警状态未知时调用。
* repeat_actions：布尔值，用于表示告警是否重复评估，默认为false。


## 2 服务
alarm相关有两个服务，入口如下：

    ceilometer-alarm-evaluator = ceilometer.alarm.service:alarm_evaluator
    ceilometer-alarm-notifier = ceilometer.alarm.service:alarm_notifier

## 3 alarm_evaluator服务

alarm_evaluator的服务实现主要有两种（配置项）：

    ceilometer.alarm.service.SingletonAlarmService  #alarm评估组件，用来评估并设置告警状态
    ceilometer.alarm.service.PartitionedAlarmService  #告警通知组件，用来发出告警通知
    

具体使用哪种是在配置文件里面配置的，默认为**前者**。
#### 3.1 单例模式告警服务SingletonAlarmService
SingletonAlarmService会定时（周期为配置项，默认**1分钟**）的对**当前租户中所有enable字段为True的alarms**进行评估。

在setup.cfg中定义两种alarm评估器，在初始化SingletonAlarmService服务的时候会通过插件的形式加载这两种evaluator： 

    threshold = ceilometer.alarm.evaluator.threshold:ThresholdEvaluator
    combination = ceilometer.alarm.evaluator.combination:CombinationEvaluator
 
在对alarms的进行评估的时候，首先会依次获取alarm的type（目前也只有两种threshold和combination ），选取上述对应的alarm evaluator进行评估，如果alarm的类型（目前有两种：threshold,combination）不支持，则不进行评估。


#### 3.1.1 ThresholdEvaluator的评估流程

对于Treshold类型的alarm的评估流程：

```python
    def evaluate(self, alarm):
        query = self._bound_duration(alarm, alarm.rule['query'])#获取查询参数
        statistics = self._sanitize(alarm, self._statistics(alarm, query))#获取有效的统计数据
        if self._sufficient(alarm, statistics):#判断是否满足评估条件
            ……………………#比较操作
            self._transition(alarm,statistics,map(_compare, statistics))#刷新状态并发出告警通知

```

① 首先会获取一个对所评估的计量对象的统计数据计算一个时间段，用于获取该时间段的statistics。
这里分两种情况：

若无排除异常值（参见上述alarm对象模型）：

    评估区间时长（window）= 周期（period）*（周期数（evaluation_periods）+1）
若有排除异常值，则：
    评估区间时长（window）= 周期（period）*（周期数（evaluation_periods）*2）
最后：
    start = now - 评估区间时长（window）
    end = now

②用上一步得到的时间段区间作为查询条件查询alarm中定义的计量对象的时间区间，并以period分割为列表返回。
③ 对上一步查询的结果统计数据（statistics列表）进行处理，也分为两种情况：
* 若无排除异常值，则取后evaluation_periods个统计数据，即去掉第一个。
* 若有排除异常值，通过一些列数学计算（做标准差等）排除掉异常值。
最终会得到评估周期数目个元素的统计数据列表。

④ 判断alarm是否有充分的评估条件，如果②步中得到的statistics列表为空，并且alarm的状态不为insufficient data，则记录datapoints are unknown，并且刷新db中alarm的状态为insufficient data，否则认为具有评估条件
⑤ 如果③结果满足评估条件， 用得到的statistics数据列表中每一项调用一个比较操作方法，该方法是这样实现的：
   
* 首先获取alarm rule中定义的comparison_operator（gt，lt，ge，le，eq，ne的一种，默认eq），作为op；

* 再根据alarm rule中定义的statistic字段（'max', 'min', 'avg', 'sum'的一种，默认avg）来获取②中计算到的statistic中该字段的取值作为value；
* 再去alarm rule中的threshold，即门槛值作为limit；最后执行op(value, limit)操作来确定是否满足告警触发条件，例如：instance的平均值是否等于5

⑥ 确定状态，分为以下两种情况：
* 如果第④步中所有结果都不满足，或者所有结果都不满足，则记state置为alarm（所有结果都满足），置为OK（所有结果都不满足）；

* 如果alarm自身的状态和这一步得到的state不一致或者alarm的alarm.repeat_actions不为False，则刷新alarm的状态并且做notify操作；

* 如果④中的结果即不满足全为真也不满足全为假，并且如果alarm的状态为unknown（即insufficient data）或者continuous不为False，若是前者，则取第④步中最后一项的结果，为真则alarm的状态为alarm，为假则状态不变；若是后者，则alarm的状态不变。最后刷新alarm的状态，并且做notify操作。
⑦ 通知告警，evaluator会向notifier服务发出一个异步调用，将如下的信息发给notifier进行通知告警：

            'actions': actions,  #alarm的处理动作
            'alarm_id': alarm.alarm_id,  #alarm id
            'previous': previous,  #alarm之前的状态
            'current': alarm.state,  #alarm当前状态
            'reason': unicode(reason),  #alarm状态改变原因
            'reason_data': reason_data  #alarm状态改变原因的详细信息

详见地4部分的介绍。

#### 3.1.2 CombinationEvaluator的评估流程

①首先是获取alarm rule专用alarm_ids列表专用的每一项的state并与alarm_ids组成一一对应的元组序列，（代码中zip方法和itertools.imap的使用请参见[这里](http://www.lfyzjck.com/python-zip/)和[这里](http://blog.csdn.net/xiaocaiju/article/details/6968123))

②和ThresholdEvaluator的③类似，做alarm的充足条件判断，根据上一步获取的alarm状态列表，检查是否有alarm的状态为None或者insufficient data，如果有，则更新该alarm的状态为insufficient data，并且记录reason为alarms in unkwon状态。 否则，返回真。

③接着确定alarm combination的状态。首先获取alarm rule中的operator（and或or），在遍历前面得到的所有alarm状态并用operator对应的操作（and 对应all，or对应any）来计算综合状态，若计算结果为真则得到状态为alarm，否则状态为ok，最后刷新alarm combination的状态以及reason到db并执行notify操作（同上文）。

ps:ThresholdEvaluator和CombinationEvaluator最终都会刷新状态并且执行notify操作，notify操作需要使用alarm的Notifier服务，详见后文。

#### 3.2 分区化告警服务PartitionedAlarmService

与SingletonAlarmService使用了一个定时器来周期评估alarm不同的是，PartitionedAlarmService定义了三个定时器来完成定时评估。

这里需要注意的是在服务的初始化中除了像SingletonAlarmService一样加载了所有定义的evaluators，还初始化了一个分区协调模块的对象PartitionCoordinator。

* 第一个定时器的定时周期为配置评估周期的1/4,用于协调不同分区。
* 第一个定时器的定时周期为配置评估周期的1/2,用于校验该分区的主控权
* 第一个定时器的定时周期为配置评估周期，进行正则的alarm评估流程，和SingletonAlarmService中的一致，不再详述。

PS：关于PartitionCoordinator还有很多不明了的地方，后续再学习并分享。


### 4 alarm_notifier服务
同样回到ceilometer setup.cfg，可以看到alarm_notifier的服务入口：

    ceilometer-alarm-notifier = ceilometer.alarm.service:alarm_notifier
可以看出了解了ceilometer的setup.cfg就了解了ceilometer大半了：）

和其他服务类似，alarm_notifier服务主要也分为以下几步：

1.准备工作，主要设置一些日志级别之类的。

2.加载setup.cfg中定义的notifier

    ceilometer.alarm.notifier =
        log = ceilometer.alarm.notifier.log:LogAlarmNotifier
        test = ceilometer.alarm.notifier.test:TestAlarmNotifier
        http = ceilometer.alarm.notifier.rest:RestAlarmNotifier
        https = ceilometer.alarm.notifier.rest:RestAlarmNotifier

这里定义了四种notifier（实际只有两种），其中LogAlarmNotifier时间alarm通知打印到日志中，RestAlarmNotifier是通过https的协议通知alarm。

RestAlarmNotifier会通过起一个eventlet并将alarm的信息发送POST请求到action中解析出来的url。

3.建立rpc消息consummer，监听rpc消息队列。

从AlarmEvaluator发出的alarm通知通过rpc__异步__调用的形式，调用alarm_notifier服务的的关键处理方法：notify_alarm()。

接下来会分别处理alarm的三种状态(alarm,ok,insufficient data)对应的的action(若action不为空)。

由于每一种action都是一个URL，会从头URL中解析出一个action对象，如果解析失败，则在日志中打印error信息，并且返回。

否则根据解析结果中的**action.scheme**来选取对应的notifier对象（即前面四种中选取对应的一种），并且调用该notifier对应的notify方法，发出alarm通知。

alarm通知中主要包含了3.1.1小结⑦中列出的信息。

至此，整个ceilometer的alarm相关流程简要介绍完，后续再深入学习分享。