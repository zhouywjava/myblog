---
title: 原创:esper 事件引擎
date: 2019-01-23 09:13:05
tags: 
- framework
- 事件
- 异步
comments: true
categories: 
- Java架构
- 源码
- 事件
- 异步
---
espertech 是一个(CEP)复杂事件处理引擎，适合做实时分析，网上的资料比较少。我们看看能不能借助一些资料，做一个demo。CEP的一个重要特点就是他是一个内存计算工具和类SQL语句。内存计算可以说是一把双刃剑。好处自不必说，一个字：快！坏处也显而易见，数据有丢失的风险，而且还有容量的限制。内存计算不是高可用的。CEP的类SQL语句，可以理解为处理模型的定义与描述。在Esper中，这个句子叫做EPL，即Event Process Language。
<!--more-->

{% blockquote 转自：A.King https://www.cnblogs.com/aking1988/p/3282450.html %}
{% endblockquote %}

这个是一个入门的例子。接下来,针对例子涉及的几个重要对象做一个介绍
创建引擎实例 EPServiceProvider。可以结合spring 的IOC原理。将这个类的实例化托管给容器。具体的做法是使用 继承AbstractFactoryBean<EPServiceProvider>类 。如果有需要还可以实现  implements BeanNameAware, FactoryBean<EPServiceProvider>, InitializingBean, DisposableBean 这些接口来进行扩展。
{% codeblock 1: 创建引擎实例 EPServiceProvider lang:java %}
  EPServiceProvider provider = EPServiceProviderManager.getDefaultProvider(config);
{% endcodeblock %}
{% codeblock 2: 生产者生产事件 EPRuntime lang:java %}
    EPRuntime er = provider.getEPRuntime();
    er.sendEvent(new MyEvent(1,"aaa"));
{% endcodeblock %}
{% codeblock 3: 事件管理 EPAdministrator lang:java %}
 EPServiceProvider provider = EPServiceProviderManager.getDefaultProvider(config);
 EPAdministrator admin = provider.getEPAdministrator();
{% endcodeblock %}
{% codeblock 4: 事件状态类 EPStatement lang:java %}
  //创建EPL查询语句实例，功能：查询所有进入的myEvent事件
  EPStatement statement = admin.createEPL("select id, name from"+MyEvent.class.getName() ); 
{% endcodeblock %}
{% codeblock 5: 事件状态类添加事件监听  lang:java %}
    EPStatement statement = admin.createEPL("select id, name from myEvent"); 
    //为statement实例添加监听
    statement.addListener((newEvents,oldEvents)-> { 
        for (EventBean eb : newEvents) {
            System.out.println("id:" + eb.get("id") + " name:" + eb.get("name"));
        }
    });
{% endcodeblock %}
6：事件
事件 可以是：POJO，java.util.Map，Object Array，XML。

7：进程模型（监听事件）
1.UpdateListener：UpdaterListener是Esper提供的一个接口，用于监听某个EPL在引擎中的运行情况，即事件进入并产生结果后会通知UpdateListener。比如：
EPL:select name from User
//假设newEvents长度为一 注意：events是一个数组，用来存放一段时间内的所有事件。这边假设只有一个事件。
newEvents[0].get("name")能得到进入的User事件的name属性值
 
EPL:select count(*) from User.win:time(5 sec)
//假设newEvents长度为一
newEvents[0].get("count(*))能得到5秒内进入引擎的User事件数量有多少

// Apple事件进入Esper，只有amount大于200的才能进入win:length，并且length长度为5
EPL:select * from Apple(amount>200).win:length(5)

// Apple事件进入Esper并进入win:length(5)，但是只有amount大于200的才能触发UpdateListener
EPL:select * from Apple.win:length(5) where amount>200

// 统计进入的5个Apple事件，amount的总数是多少
select sum(amount) from Apple.win:length_batch(5)
 
// 统计进入的5个Apple事件，amount的总数是多少，并按照price分组
select price, sum(amount) from Apple.win:length_batch(5) group by price
 
// 统计进入的5个Apple事件，amount的总数和name，并按照price分组
select price, name, sum(amount) from Apple.win:length_batch(5) group by price

{% blockquote 转自：luonanqin https://blog.csdn.net/luonanqin/article/details/21300263 %}
{% endblockquote %}
这是一篇比较完整的博客，我们看看能学到些什么。博客举了一个计算苹果平均价格的例子
{% blockquote 计算3个苹果的平均值%}
    public class AppleListener implements UpdateListener
    {
    	public void update(EventBean[] newEvents, EventBean[] oldEvents)
    	{
    		if (newEvents != null)
    		{
    			Double avg = (Double) newEvents[0].get("avg(price)");
    			System.out.println("Apple's average price is " + avg);
    		}
    	}
    }
    ...
    public static void main(String[] args) throws InterruptedException {
        EPServiceProvider epService = EPServiceProviderManager.getDefaultProvider();

        EPAdministrator admin = epService.getEPAdministrator();

        //EPL：即Event Process Language 可以理解为处理模型的定义与描述。 win:length_batch(3) 就是说收到3个事件后触发事件
        String product = Apple.class.getName();
        String epl = "select avg(price) from " + product + ".win:length_batch(3)";
        
        EPStatement state = admin.createEPL(epl);
        state.addListener(new AppleListener());

        EPRuntime runtime = epService.getEPRuntime();

        Apple apple1 = new Apple();
        apple1.setId(1);
        apple1.setPrice(5);
        runtime.sendEvent(apple1);

        Apple apple2 = new Apple();
        apple2.setId(2);
        apple2.setPrice(2);
        runtime.sendEvent(apple2);

        Apple apple3 = new Apple();
        apple3.setId(3);
        apple3.setPrice(5);
        runtime.sendEvent(apple3);
    }
{% endblockquote %}
上面的例子有几个注意的地方，a.UpdateListener 当满足事件的事情发生以后 会调用监听方法。b.EPL 语法与sql很相近。数据源不是表，而是一个事件。
那篇博客还介绍了许多事件流的计算场景,还有重中之重的EPL语法。这边不重复介绍。接下来,看看如何封装到spring中。总结了下项目中采用的封装方式,与应用场景。我们站在应用的角度一步一步去剖析它。

1.当你要使用esper 引擎来做事件通知的时候。要用spring 先初始化一个对象
{% blockquote 控制反转创建消息发送对象%}
 ...
 /**
 * 消息发送对象
 */
 private EventManager eventManager;
 ...
{% endblockquote %}
我们探查下spring 如何做的控制反转。
{% blockquote 提供了set 方法，ioc创建EventManager容器的时候会同步注入EPServiceProvider对象%}
public class EventManager {
    private EPServiceProvider serviceProvider;

    public void sendEvent(Object theEvent) {
        if (theEvent == null) {
            throw new RuntimeException("发送的事件对象不能为空!");
        }
        EPRuntime runtime = serviceProvider.getEPRuntime();
        runtime.sendEvent(theEvent);
    }

    public void setServiceProvider(EPServiceProvider serviceProvider) {
        this.serviceProvider = serviceProvider;
    }
}
{% endblockquote %}
继续探查ioc创建EPServiceProvider对象
{% blockquote 提供了set 方法，ioc创建EventManager容器的时候会同步注入EPServiceProvider对象%}
public class EPServiceProviderFactoryBean extends AbstractFactoryBean<EPServiceProvider> implements BeanNameAware, FactoryBean<EPServiceProvider>, InitializingBean, DisposableBean {
{% endblockquote %}
通过ioc 的工厂方法，实现了ioc 的控制反转。后面的实现接口表示的是spring 创建实例的时候的一些扩展点。我们看下createInstance()这个方法的实现
{% blockquote %}
    @Override
    protected EPServiceProvider createInstance(){
     EPServiceProvider provider = getProvider();
     EPAdministrator administrator = provider.getEPAdministrator();
     BeanFactory beanFactory = getBeanFactory();
     //ListableBeanFactory 这个是稳定的ioc容器
     if (beanFactory instanceof ListableBeanFactory) {
        ListableBeanFactory listableBeanFactory = (ListableBeanFactory) beanFactory;
        //注意这一步是精彩一步，这步是从容器中获取所有的EPStatementResidtrar 的实现。
        //然后注册EPStatement 。因为注册 EPStatement state = admin.createEPL(epl);state.addListener(new AppleListener()); 
        //是需要EPAdministrator 对象的，因此registerStatements 方法中传入了EPAdministrator 对象的引用
        //这步是在暗示什么呢？就是如果要使用流式计算，那么只需要实现EPStatementRegistrar 这个接口即可
        // 实现的例子就是：EPStatement state = admin.createEPL(epl);state.addListener(new AppleListener()); 
        Collection<EPStatementRegistrar> registrars = BeanFactoryUtils.beansOfTypeIncludingAncestors(listableBeanFactory, EPStatementRegistrar.class, true, true).values();
        for (EPStatementRegistrar registrar : registrars) {
            registrar.registerStatements(administrator);
        }
     }
     return provider;
    }
{% endblockquote %}
点睛之笔后都可以同Apple 的例子一样了。
{% blockquote 发送消息%}
...
MessageTransferEvent transferEvent = new MessageTransferEvent();
transferEvent.setSubject("文件下载操作已经处理");
transferEvent.setContent("文件下载失败，失败原因：数据格式不合规范"); <br/>or
transferEvent.setContent("文件下载成功，路径是：getFilePath()");
transferEvent.setRecipients(vo.getUserId());
transferEvent.setReceiveType(MessageTransferEvent.RECEIVE_TYPE_SINGLE);
eventManager.sendEvent(transferEvent);
...
{% endblockquote %}

END


