---
title: 浅析ceilometer中agent
date: 2014/5/20
---
### 1.引言：###
Ceilometer项目创建时最初的目的是实现一个能为计费系统采集数据的框架。在G版的开发中，社区已经更新了他们的目标，新目标是希望Ceilometer成为OpenStack里数据采集（监控数据、计费数据）的唯一基础设施，采集到的数据提供给监控、计费、面板等项目使用。

这个项目，从诞生之日起，就被用于越来越多其他不同项目（openstack的组件）的计量信息采集。
ceilometer可以为监控、调试、以及图形化工具推送收集到的信息，与此同时也将信息存入ceilometer的存储后端，这种消息推送机制定义为“multi-publisher”。

最近，heat的诞生，更加明确了OpenStack需要有一个工具来监控多样的数据从而触发不同的操作。由于ceilometer已经可以用于收集浩如烟海的数据，顺利成章地扩展出一个新的功能“alarming”。

### 2.ceilometer中的服务：

ceilometer主要由四个服务组成：
- ceilometer-api，提供ceilometer对外的接口；
- ceilometer-collector，ceilometer中最核心的组件，主要作用是监听Message Bus，将收到的消息以及相应的数据写入到数据库中，它是在核心架构中唯一一个 能够对数据库进行写操作的组件（当然，后来ceilometer-api也具有对DB写操作的权限）。除此之外，它的另一个作用是对收到的其它服务发来的notification消息做本地化处理，然后再重新发送到 Messsge Bus中去，随后再被其收集。
- ceilometer-central-agent，这个组件运行在控制节点上面，负责了除Compute（nova）之外的所有信息采集，是通过周期性调用Image, Volume, Objects, Network等这些组件的REST API的方式来轮询信息的。
- ceilometer-compute-agent，这个组件运行在compute（nova）节点上，该组件用来收集计算节点上的信息，通过Stevedore管理了一组pollster插件， 分别用来获取虚拟机的CPU, Disk IO, Network IO, Instance这些信息，值得一提的是这些信息大部分是通过调用Hypervisor的API来获取的， 目前，Ceilometer仅提供了Libvirt的API。

整个ceilometer的逻辑架构如下图：
![](/images/2014-5-20-the-agents-of-ceilometer/1.png)

详细信息可以参考：[ceilometer开发者文档](http://docs.openstack.org/developer/ceilometer/architecture.html)


### 3.ceilometer中数据的采集：

ceilometer使用三种独立的方式来获取计量数据：
- Bus listener agent,用于收集Oslo notification bus上的事件，并将其转换成为ceilometer计量样本。这是ceilometer数据采集最首选的方式
- Push agents,这是唯一一种远端获取请求数据且不暴露出来的方式。这种方式不是首选，由于部署上更加复杂一点，需要为每一个需要被监控的节点（compute节点）增加一个组件。但是相比于polling agent，这种方式更加具有弹性（高可用性）
- Polling agents，这是最后一种收集数据的方式，这种方式通过一个固定的周期轮询测量其他模块的API或其他工具收集数据。
第一种方式由ceilometer-collector提供，collector监控消息队列，获取其他组件的notifications以及polling agent 和push agent推送至消息队列的中的数据。第二、三种方式需要ceilometer-agent以及ceilometer-agent-compute/ceilometer-agent-central配合完成。

由上可以看出ceilometer主动对openstack中计量信息的采集主要依靠ceilometer中的agent来完成的。

### 4.agent###

(为了能够串起来，读者可以先看第5小结，再回到第4小结)

ceilometer中agent主要的实现框架在ceilometer.agentAgentManager中，后面可以看出ceilometer的agent的服务初始化首先会到这里建立任务，实现数据采集。
在这个类中有一个很重要的方法：
def setup_polling_tasks():

这里便于分析，将代码贴出：

    def setup_polling_tasks(self):
        polling_tasks = {}
        for pipeline, pollster in itertools.product(
                self.pipeline_manager.pipelines,
                self.pollster_manager.extensions):
            if pipeline.support_meter(pollster.name):
                polling_task = polling_tasks.get(pipeline.interval, None)
                if not polling_task:
                    polling_task = self.create_polling_task()
                    polling_tasks[pipeline.interval] = polling_task
                polling_task.add(pollster, [pipeline])

        return polling_tasks
 
 这个方法中对ceilometer中每一个pollster（关于pollster以后有机会再详细介绍，这里可以为是一种统计数据的采集器，）以及每一种pipline（这里可以简单的理解为一种即将数据转换的通道，由ceilometer中的transformer组成）

这里我们注意到for循环的循环范围使用了itertools.product(),这个方法是将参数做笛卡尔乘积得到迭代序列，也就是在这里将pipline和pollster建立一一对应的关系，接下来如果pipline支持这种pollster，则获取该pipline的polling_task，若该task不存在，则创建一个，polling_task用于后面的采集数据。

这里，目前ceilometer只有两种pipline，pipline的信息是以yaml(yaml是一种轻量级编程语言，类似于json)的格式写在配置文件里的：/etc/ceilometer/pipline.yaml。

这里在poling_task里面有一个重要的参数pipeline.interval，这个参数是后面将要提到数据采集任务的采集周期，这个数值也配置在pipline.yaml中，默认是600s（10min）。

然后，创建一个空的测量任务再将pipline和poster添加，至此，一个数据采集任务已经被添加，最终返回polling_task的列表。

而在服务开始函数start()，方法中我们可以看出：

        def start(self):
            self.pipeline_manager = pipeline.setup_pipeline(
                transformer.TransformerExtensionManager(
                    'ceilometer.transformer',
                ),
            )
    
            for interval, task in self.setup_polling_tasks().iteritems():
                self.tg.add_timer(interval,
                                  self.interval_task,
                                  task=task)


这里首先是安装了pipline，，然后调用上面的setup_polling_tasks（），将其中的每一个测量任务，按照上述周期添加一个定时器，这里定时器的处理函数式interval_task，这个函数式真正执行测量任务的，会调用该任务的poll_and_publish（）函数。

再回到ceilometer.central.manager:agent_central，也就是具体的一个agent的子类的poll_and_publish，这个方法会遍历所有的前面添加的polling_task,调用每pollster对象的get_samples()方法



### 5.central-agent

首先，ceilometer的服务启动入口，我们可以从ceilometer的安装脚本setup.cfg看到：

    console_scripts =
        ceilometer-api = ceilometer.api.app:start
        ceilometer-agent-central = ceilometer.central.manager:agent_central
        ceilometer-agent-compute = ceilometer.compute.manager:agent_compute
        ceilometer-agent-notification = ceilometer.notification:agent
        ceilometer-dbsync = ceilometer.storage:dbsync
        ceilometer-expirer = ceilometer.storage:expirer
        ceilometer-collector = ceilometer.collector:collector
        ceilometer-alarm-evaluator = ceilometer.alarm.service:alarm_evaluator
        ceilometer-alarm-notifier = ceilometer.alarm.service:alarm_notifier

ps：关于setup.py和setu.cfy，可以参考这里[这里](http://yansu.org/2013/06/07/learn-python-setuptools-in-detail.html)或是[这里](http://blog.csdn.net/lynn_kong/article/details/17540207)

这里我们可以看出ceilometer-central-agent的入口在ceilometer.central.manager:agent_central

而这个类，主要使用了eventlet起了一个服务，其中关键的实现在：

    os_service.launch(AgentManager()).wait()

从这里我们可以看出，最终会执行到AgentManager这个类的初始化，而它又继承于ceilometer/agent.py中AgentManager，后面我们可以看出ceilometer的所有的agent都继承自这个公共的AgentManager（/ceilometer/agent.py）接第3小结。

最后，调用每一个具体的pollster的get_samples()，central-agent的pollster主要有：

    image.size
    storage.objects
    storage.objects.containers
    ip.floating 
    energy
    power
    storage.objects.size
    image 

其中energy和power使用来测量能耗的，需要有kwapi的支持。这些pollster都对应于ceilometer中一个类的实现，以image为例，ceilometer会首先调用keystoneclient获取鉴权信息，在调用glance的api（image.list接口）来获取image信息，最终经过将获取到的数据经过transformer转换再调用publisher(samples)将采集到的样本信息发布（由多种发布方式，通常发布到消息队列，再由ceilometer-collector收集并存到ceilometer的存储后端（db等））。

### 6.compute-agent

compute-agent的接口基本上和central-agent的一致，主要获取虚拟机的相关信息，由于compute是运行在每一个compute节点上的，所以compute-agent获取的信息是以本节点上的虚拟机为主体的，即在其poll_and_publish方法中首先调用novaclient的instance_get_all_by_host接口获取到该节点上所有虚拟机列表，最终对每一个虚拟机执行pollster对象列表中的get_samples方法。（这里与前面有所不同的是多了一层循环，首先遍历instances，再遍历pollsters）

compute-agent的pollster在ceilmeter代码中的结构比较清晰，如下图：
![](/images/2014-5-20-the-agents-of-ceilometer/2.jpg)

这里可以看出主要获取了虚拟机的cpu、disk、net，其中instance pollster主要是虚拟机自身的信息,包括flavor信息、name、type等。


这里我们需要重点关注的是获取虚拟机的cpu、disk、net信息的时候会通过ceilometer的inspector来获取虚拟机化层的该虚拟机的信息。

目前，ceilometer实现了的inspector主要有hyperv的喝libvirt的。

inspector是通过前面查到的虚拟机信息里面的instance_name来调用虚拟机化层接口，最终返回该虚拟机的：

    cpu：number（数量）；time （使用时间）
    nic：interface（网卡地址）；stats（网卡接收流量，网卡输出流量）
    disk：stats（read字节数和write字节数b）

ps：除了central-agent和compute-agent，之外还有notification-agent，以后将会做专门介绍。
### 7.总结

可以看出，ceilometer的central-agent主要是在控制节点固定周期地调用openstack其他组件的API来轮询信息；compute-agent是通过获取所在计算节点上的所有虚拟机，然后调用virt层的接口来获取每一个虚拟机的cpu、disk、nic信息，最终将这些信息发布。

本人对于openstack中ceilometer了解的也不够深入，欢迎交流讨论。








