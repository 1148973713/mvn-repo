pom.xml文件导入以下配置

```xml
    <dependencies>
        <dependency>
            <groupId>com.domain.event</groupId>
            <artifactId>DSEvent</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    <dependencies>    
     
     <repositories>
        <repository>
            <id>mvn-lin</id>
            <url>https://github.com/CG-Lin/mvn-lin/master</url>
            <snapshots>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
    </repositories>
```

编写event聚合实体类继承DomainEvent

```java
@Data
@TableName(value = "order_copy")
public class CreateTickEvenDomainEvent extends DomainEvent {

    @TableId
    @TableField(value ="tick_id")
    private String tickId;

    @TableField(value ="tick_name")
    private String tickName;

    @TableField(value ="tick_price")
    private Integer tickPrice;

    @TableField(value ="tick_status")
    private Boolean tickStatus;

    @TableField(value = "tick_frozen")
    private Integer tickFrozen;
}
```

在Saga中心配置事件：

样例一：

```java
        //启动处理器
        StartHandler startHandler = new StartHandler();
        //根据需求赋予eventId
        String eventId = UUID.randomUUID().toString();
        //单个事件：eventId：事件唯一标识；createTickEvenDomainEvent：事件聚合；createTick:事件执行；rollbackTick：补偿事件
        startHandler.workEventByDisruptor(eventId, createTickEvenDomainEvent, createTickService::createTick, createTickService::rollbackTick);
```

事件执行方法：

```java
    public void createTick(CreateTickEvenDomainEvent createTickEvenDomainEvent) {
        //****自定义业务逻辑***
        System.out.println("准备扣款冻结");
        //故意设置错误，让事件进行回滚
    }
```

事件补偿方法：

```java
    public void rollbackTick(CreateTickEvenDomainEvent createTickEvenDomainEvent) {
        //****自定义补偿逻辑***
        System.out.println("回滚成功");
    }
```

样例二：

```java
public class TestDEventWork {

    private static Integer result;

    private static Integer allMoney;

    private void memberDomainEventVoid(MemberEvenDomainEvent memberEvenDomainEvent) {
        allMoney = memberEvenDomainEvent.getMemberAllMoney() - 10;
        System.out.println(memberEvenDomainEvent.getMemberName() +"扣了10元，存款剩余:" + allMoney);
    }

    private void memberCompensateDomainEventVoid(MemberEvenDomainEvent memberEvenDomainEvent) {
        allMoney = allMoney + 10;
        System.out.println(memberEvenDomainEvent.getMemberName() + "账户回滚成功，存款剩余:" + allMoney);
    }

    private void TickDomainEventVoid(TickEvenDomainEvent tickEvenDomainEvent) {
        //String tickId = createTickEvenDomainEven;
        result = tickEvenDomainEvent.getTickNum() - 1;
        String tickName = tickEvenDomainEvent.getTickName();
        System.out.println(tickName + ":正在减库存:" + result);
        //故意执行错误，便于进行回滚
        int i = 10 / 0;
    }

    private void TickCompensateDomainEventVoid(TickEvenDomainEvent tickEvenDomainEvent) {
        result = result + 1;
        String tickName = tickEvenDomainEvent.getTickName();
        System.out.println(tickName + ":库存回滚操作成功" + result);
    }

    public void sagaStatus() {
        //用户余额
        MemberEvenDomainEvent memberEvenDomainEvent = new MemberEvenDomainEvent();
        memberEvenDomainEvent.setMemberAllMoney(1000);
        memberEvenDomainEvent.setMemberName("张三");
        //订单锁库存
        TickEvenDomainEvent tickEvenDomainEvent = new TickEvenDomainEvent();
        tickEvenDomainEvent.setTickId("1111");
        tickEvenDomainEvent.setTickName("饼干");
        tickEvenDomainEvent.setTickNum(180);
        tickEvenDomainEvent.setTickStatus(false);
        //创建DisruptorHandler
        String eventId = UUID.randomUUID().toString();

        StartHandler memberEvenDomainEventStartHandler = new StartHandler();
        //多个事件
        memberEvenDomainEventStartHandler
                .workEventByDisruptor(eventId, memberEvenDomainEvent, this::memberDomainEventVoid, this::memberCompensateDomainEventVoid)
                .workEventByDisruptor(eventId, tickEvenDomainEvent, this::TickDomainEventVoid, this::TickCompensateDomainEventVoid);
    }

    public static void main(String[] args) {
        TestDEventWork testDEventWork = new TestDEventWork();
        testDEventWork.sagaStatus();
    }
}
```

效果：

```
事件全部回滚：
张三扣了10元，存款剩余:990
饼干:正在减库存:179
饼干:库存回滚操作成功180
张三账户回滚成功，存款剩余:1000
```

目前还有些不足后续补充
