# Hello World
我们用Java编写两个程序。发送单个消息的生产者和接收消息并打印出来的消费者
在下图中，“ P” 是我们的生产者，“ C” 是我们的消费者。中间的框是一个队列 RabbitMQ 代表使用者保留的消息缓冲区
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685858842243-e5c4200f-fa86-445c-b52f-7410372c24c3.png#averageHue=%23f5dddb&clientId=u95ff03d9-f8c4-4&from=paste&height=102&id=u5d445d16&originHeight=102&originWidth=510&originalType=binary&ratio=1&rotation=0&showTitle=false&size=16249&status=done&style=none&taskId=u4a183780-62f2-4577-b0a3-b5cd9d23f4f&title=&width=510)
注意使用Java连接`RabbitMQ`时，RabbitMQ默认使用5672端口
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685858875545-5c52aa05-8faa-4c0a-9623-02ee5fb58aeb.png#averageHue=%23f3f3f3&clientId=u95ff03d9-f8c4-4&from=paste&height=165&id=ub3240771&originHeight=165&originWidth=591&originalType=binary&ratio=1&rotation=0&showTitle=false&size=44414&status=done&style=none&taskId=u8063b7ed-4f28-4135-aa94-d180bbf4ca8&title=&width=591)
依赖：
`pom.xml`
```xml
<dependency>
  <groupId>com.rabbitmq</groupId>
  <artifactId>amqp-client</artifactId>
  <version>5.16.0</version>
</dependency>
<!--操作文件流的一个依赖-->
<dependency>
  <groupId>commons-io</groupId>
  <artifactId>commons-io</artifactId>
  <version>2.11.0</version>
</dependency>
```
`Prohucer.java`生产者
```java
public class Producer {

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws Exception{
        // 创建一个连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("175.24.183.52");
        factory.setUsername("admin");
        factory.setPassword("123456");
        // channel 实现了自动close接口 自动关闭 不需要显示关闭
        // 创建连接
        Connection connection = factory.newConnection();
        // 获取信道
        Channel channel = connection.createChannel();
        /**
         * 生成一个队列
         * 1.队列名称
         * 2.队列里面的消息是否持久化，也就是是否存储
         * 3.该队列是否只供一个消费者进行消费，是否进行共享true 可以多个消费者消费
         * 4.是否自动删除 最后一个消费者断开连接以后，该队列是否自动删除true 自动删除
         * 5.其他参数
         */
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        String message = "hello world!";
        /**
         * 发送一个消息
         * 1.发送到哪个交换机
         * 2.路由的key是哪个
         * 3.其他的参数信息
         * 4.发送消息的消息体
         */
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        System.out.println("消息发送完毕！");
    }

}
```
`Consumer.java`消费者
```java
public class Consumer {

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws Exception{
        // 创建一个连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("175.24.183.52");
        factory.setUsername("admin");
        factory.setPassword("123456");
        // channel 实现了自动close接口 自动关闭 不需要显示关闭
        // 创建连接
        Connection connection = factory.newConnection();
        // 获取信道
        Channel channel = connection.createChannel();

        System.out.println("等待接收消息。。。。。");

        //推送的消息如何进行消费的接口回调
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody());
            System.out.println(message);
        };
        //取消消费的一个回调接口 如在消费的时候队列被删除掉了
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消息消费被中断");
        };
        /**
         * 消费者消费消息 - 接受消息
         * 1.消费哪个队列
         * 2.消费成功之后是否要自动应答 true 代表自动应答 false 手动应答
         * 3.消费者未成功消费的回调
         * 4.消息被取消时的回调
         */
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);
    }

}
```
# Work Queues
Work Queues——工作队列(又称任务队列)的主要思想是避免立即执行资源密集型任务，而不得不等待它完成。 相反我们安排任务在之后执行。我们把任务封装为消息并将其发送到队列。在后台运行的工作进 程将弹出任务并最终执行作业。当有多个工作线程时，这些工作线程将一起处理这些任务。
## 轮询分发消息
在这个案例中我们会启动两个工作线程，一个消息发送线程，我们来看看他们两个工作线程是如何工作的。
### 1、提取工具类
`RabbitMQUtils.java`
```java
public class RabbitUtils {

    //得到一个连接的 channel
    public static Channel getChannel() throws Exception {
        //创建一个连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("175.24.183.52");
        factory.setUsername("admin");
        factory.setPassword("123456");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        return channel;
    }

}
```
### 2、启动两个工作线程来接收消息
```java
public class Work01 {

    private static final String QUEUE_NAME = "hello";

    public static void main(String[] args) throws Exception {

        Channel channel = RabbitMQUtils.getChannel();

        //消息接受
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String receivedMessage = new String(delivery.getBody());
            System.out.println("接收到消息:" + receivedMessage);
        };
        //消息被取消
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println(consumerTag + "消费者取消消费接口回调逻辑");

        };

        System.out.println("C1 消费者启动等待消费.................. ");
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);

    }

}
```
这里要设置一下IDEA：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685860926315-10487d9e-dfa3-4196-a3ff-98fcdcf5eff7.png#averageHue=%23f2f1f0&clientId=u95ff03d9-f8c4-4&from=paste&height=677&id=ufa8bfce4&originHeight=677&originWidth=1044&originalType=binary&ratio=1&rotation=0&showTitle=false&size=74569&status=done&style=none&taskId=u8f5e94c9-0aa6-4b11-b9bc-d8fea122aac&title=&width=1044)
启动程序。
### 3、启动一个发送消息线程
结果展示：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685861213297-b49e4431-33fa-4f6f-b516-ceb472e14234.png#averageHue=%23fbfaf9&clientId=u95ff03d9-f8c4-4&from=paste&height=257&id=u17131a65&originHeight=257&originWidth=866&originalType=binary&ratio=1&rotation=0&showTitle=false&size=34049&status=done&style=none&taskId=u9bc780df-f3ae-4f89-a5c7-9c2780745f9&title=&width=866)
可以看到两个消费者都是按照顺序接收消息的。
# 消息应答
消费者完成一个任务可能需要一段时间，如果其中一个消费者处理一个长的任务并仅只完成了部分突然它挂掉了，会发生什么情况。RabbitMQ 一旦向消费者传递了一条消息，便立即将该消息标记为删除。在这种情况下，突然有个消费者挂掉了，我们将丢失正在处理的消息。以及后续发送给该消费这的消息，因为它无法接收到。
为了保证消息在发送过程中不丢失，引入消息应答机制，消息应答就是：**消费者在接收到消息并且处理该消息之后，告诉 rabbitmq 它已经处理了，rabbitmq 可以把该消息删除了。**
## 自动应答
消息发送后立即被认为已经传送成功，这种模式需要在**高吞吐量和数据传输安全性方面做权衡**,因为这种模式如果消息在接收到之前，消费者那边出现连接或者 channel 关闭，那么消息就丢失 了,当然另一方面这种模式消费者那边可以传递过载的消息，**没有对传递的消息数量进行限制**，当然这样有可能使得消费者这边由于接收太多还来不及处理的消息，导致这些消息的积压，最终使 得内存耗尽，最终这些消费者线程被操作系统杀死，**所以这种模式仅适用在消费者可以高效并以 某种速率能够处理这些消息的情况下使用。**
## 手动消息应答的方法

- Channel.basicAck(用于肯定确认)

RabbitMQ 已知道该消息并且成功的处理消息，可以将其丢弃了

- Channel.basicNack(用于否定确认)
- Channel.basicReject(用于否定确认)

与 Channel.basicNack 相比少一个参数，不处理该消息了直接拒绝，可以将其丢弃了
**Multiple的解释：**
手动应答的好处是可以批量应答并且减少网络拥堵
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685861352059-73ec2c44-f0ea-42a2-89fb-0785ac87c6d2.png#averageHue=%232d2c29&clientId=u95ff03d9-f8c4-4&from=paste&height=120&id=ud3586898&originHeight=120&originWidth=602&originalType=binary&ratio=1&rotation=0&showTitle=false&size=53840&status=done&style=none&taskId=u6b9872bd-2079-4ab9-8ec4-f43b0789a6f&title=&width=602)

- true 代表批量应答 channel 上未应答的消息

比如说 channel 上有传送 tag 的消息 5,6,7,8 当前 tag 是8 那么此时5-8 的这些还未应答的消息都会被确认收到消息应答

- false 同上面相比只会应答 tag=8 的消息 5,6,7 这三个消息依然不会被确认收到消息应答

![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685861372911-445de631-590e-4746-9faa-097a91cd5685.png#averageHue=%23f6f5f4&clientId=u95ff03d9-f8c4-4&from=paste&height=473&id=u2e316b2a&originHeight=473&originWidth=569&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14261&status=done&style=none&taskId=u3cd55bac-0271-46bf-b2b7-0be0101dbf8&title=&width=569)
## 消息自动重新入队
如果消费者由于某些原因失去连接(其通道已关闭，连接已关闭或 TCP 连接丢失)，导致消息未发送 ACK 确认，RabbitMQ 将了解到消息未完全处理，并将对其重新排队。如果此时其他消费者可以处理，它将很快将其重新分发给另一个消费者。这样，即使某个消费者偶尔死亡，也可以确保不会丢失任何消息。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685861413876-dc14557b-c330-4436-9905-376dc95b68ba.png#averageHue=%23efe1e0&clientId=u95ff03d9-f8c4-4&from=paste&height=408&id=u56889519&originHeight=408&originWidth=938&originalType=binary&ratio=1&rotation=0&showTitle=false&size=100869&status=done&style=none&taskId=ua9e64fa4-63a6-48ab-89a7-06d83582150&title=&width=938)
## 消息手动应答代码
默认消息采用的是自动应答，所以我们要想实现消息消费过程中不丢失，需要把自动应答改为手动应答
消费者在上面代码的基础上增加了一下内容
```java
channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
```
消息生产者：
```java
public class Task02 {

    private static final String TASK_QUEUE_NAME = "ack_queue";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitMQUtils.getChannel();
        //声明队列
        channel.queueDeclare(TASK_QUEUE_NAME, false, false, false, null);
        Scanner sc = new Scanner(System.in);
        System.out.println("请输入信息");
        while (sc.hasNext()) {
            String message = sc.nextLine();
            //发布消息
            channel.basicPublish("", TASK_QUEUE_NAME, null, message.getBytes("UTF-8"));
            System.out.println("生产者发出消息" + message);
        }
    }

}
```
消费者1：
```java
public class Work03 {

    private static final String TASK_QUEUE_NAME = "ack_queue";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitMQUtils.getChannel();
        System.out.println("C1 等待接收消息处理时间较 短");

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("接收到消息:" + message);
            /**
             * 1.消息标记 tag
             * 2.是否批量应答未应答消息
             */
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        };

        CancelCallback cancelCallback = (s) -> {
            System.out.println(s + "消费者取消消费接口回调逻辑");
        };

        //采用手动应答
        boolean autoAck = false;
        channel.basicConsume(TASK_QUEUE_NAME, autoAck, deliverCallback, cancelCallback);
    }

}
```
消费者2：
```java
public class Work04 {

    private static final String TASK_QUEUE_NAME = "ack_queue";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitMQUtils.getChannel();
        System.out.println("C1 等待接收消息处理时间较 短");

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody());
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("接收到消息:" + message);
            /**
             * 1.消息标记 tag
             * 2.是否批量应答未应答消息
             */
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        };

        CancelCallback cancelCallback = (s) -> {
            System.out.println(s + "消费者取消消费接口回调逻辑");
        };

        //采用手动应答
        boolean autoAck = false;
        channel.basicConsume(TASK_QUEUE_NAME, autoAck, deliverCallback, cancelCallback);
    }

}
```
启动程序，正常情况下是这样的：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685861890283-e306a830-ab66-459f-a45e-90bbd6b6f85a.png#averageHue=%23fcfcfc&clientId=u95ff03d9-f8c4-4&from=paste&height=252&id=u54afb199&originHeight=252&originWidth=448&originalType=binary&ratio=1&rotation=0&showTitle=false&size=16884&status=done&style=none&taskId=u5d08210f-3e98-4244-97fc-ad20b3f7320&title=&width=448)
下面我们再发送消息，在发送消息ddd之后，将消费者2停掉，按理说该 消费者2 来处理该消息，但是由于它处理时间较长，在还未处理完，也就是说 消费者2 还没有执行 ack 代码的时候，消费者2被停掉了， 此时会看到消息被 消费者1 接收到了，说明消息 ddd 被重新入队，然后分配给能处理消息的 消费者1 处理了
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685862130311-ba9d8f52-45f8-4ba0-9371-bab52606dee0.png#averageHue=%23fdfcfc&clientId=u95ff03d9-f8c4-4&from=paste&height=376&id=u671729d4&originHeight=376&originWidth=587&originalType=binary&ratio=1&rotation=0&showTitle=false&size=41586&status=done&style=none&taskId=u4f3c2c31-c86c-4546-9f1f-5562bce3596&title=&width=587)
# RabbitMQ持久化
当 RabbitMQ 服务停掉以后，消息生产者发送过来的消息不丢失要如何保障？默认情况下 RabbitMQ 退出或由于某种原因崩溃时，它忽视队列和消息，除非告知它不要这样做。确保消息不会丢失需要做两件事：**我们需要将队列和消息都标记为持久化。**
> 队列如何实现持久化

之前我们创建的队列都是非持久化的，rabbitmq 如果重启的化，该队列就会被删除掉，如果要队列实现持久化需要在声明队列的时候把 durable 参数设置为持久化
```java
//让队列持久化
boolean durable = true;
//声明队列
channel.queueDeclare(TASK_QUEUE_NAME, durable, false, false, null);
```
下面是在管理面板看到的队列是否持久化的区别，`task_queue`是持久化的：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685862402282-a70702a6-2eeb-412a-984e-d5113102dfec.png#averageHue=%23f1f0ef&clientId=u95ff03d9-f8c4-4&from=paste&height=161&id=uce5d03e7&originHeight=161&originWidth=843&originalType=binary&ratio=1&rotation=0&showTitle=false&size=17653&status=done&style=none&taskId=u6647582e-6e4d-43d3-9f71-4ec67c6e114&title=&width=843)
> 消息实现持久化

需要在消息生产者修改代码，添加这个属性：`MessageProperties.PERSISTENT_TEXT_PLAIN`
```java
channel.basicPublish("", TASK_QUEUE_NAME, MessageProperties.PERSISTENT_BASIC, message.getBytes("UTF-8"));
```
将消息标记为持久化并不能完全保证不会丢失消息。尽管它告诉 RabbitMQ 将消息保存到磁盘，但是这里依然存在当消息刚准备存储在磁盘的时候 但是还没有存储完，消息还在缓存的一个间隔点。此时并没 有真正写入磁盘。持久性保证并不强，但是对于我们的简单任务队列而言，这已经绰绰有余了。
# 不公平分发
在最开始的时候我们学习到 RabbitMQ 分发消息采用的轮训分发，但是在某种场景下这种策略并不是很好，比方说有两个消费者在处理任务，其中有个**消费者 1** 处理任务的速度非常快，而另外一个**消费者 2** 处理速度却很慢，这个时候我们还是采用轮训分发的化就会到这处理速度快的这个消费者很大一部分时间处于空闲状态，而处理慢的那个消费者一直在干活，这种分配方式在这种情况下其实就不太好，但是 RabbitMQ 并不知道这种情况它依然很公平的进行分发。
为了避免这种情况，**在消费者中消费之前**，我们可以设置参数：`channel.basicQos(1);`
```java
//不公平分发
int prefetchCount = 1;
channel.basicQos(prefetchCount);
//采用手动应答
boolean autoAck = false;
channel.basicConsume(TASK_QUEUE_NAME, autoAck, deliverCallback, cancelCallback);
```
意思就是如果这个任务我还没有处理完或者我还没有应答你，你先别分配给我，我目前只能处理一个 任务，然后 rabbitmq 就会把该任务分配给没有那么忙的那个空闲消费者，当然如果所有的消费者都没有完 成手上任务，队列还在不停的添加新任务，队列有可能就会遇到队列被撑满的情况，这个时候就只能添加 新的 worker 或者改变其他存储任务的策略。
# 预取值分发
带权的消息分发
本身消息的发送就是异步发送的，所以在任何时候，channel 上肯定不止只有一个消息另外来自消费 者的手动确认本质上也是异步的。因此这里就存在一个未确认的消息缓冲区，因此希望开发人员能**限制此缓冲区的大小**，**以避免缓冲区里面无限制的未确认消息问题**。这个时候就可以通过使用 basic.qos 方法设 置“预取计数”值来完成的。
该值定义通道上允许的未确认消息的最大数量。一旦数量达到配置的数量， RabbitMQ 将停止在通道上传递更多消息，除非至少有一个未处理的消息被确认，例如，假设在通道上有未确认的消息 5、6、7，8，并且通道的预取计数设置为 4，此时RabbitMQ 将不会在该通道上再传递任何消息，除非至少有一个未应答的消息被 ack。比方说 tag=6 这个消息刚刚被确认 ACK，RabbitMQ 将会感知 这个情况到并再发送一条消息。消息应答和 QoS 预取值对用户吞吐量有重大影响。
通常，增加预取将提高 向消费者传递消息的速度。**虽然自动应答传输消息速率是最佳的，但是，在这种情况下已传递但尚未处理的消息的数量也会增加，从而增加了消费者的 RAM 消耗**(随机存取存储器)应该小心使用具有无限预处理的自动确认模式或手动确认模式，消费者消费了大量的消息如果没有确认的话，会导致消费者连接节点的 内存消耗变大，所以找到合适的预取值是一个反复试验的过程，不同的负载该值取值也不同 100 到 300 范 围内的值通常可提供最佳的吞吐量，并且不会给消费者带来太大的风险。
预取值为 1 是最保守的。当然这将使吞吐量变得很低，特别是消费者连接延迟很严重的情况下，特别是在消费者连接等待时间较长的环境 中。对于大多数应用来说，稍微高一点的值将是最佳的。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685862789948-7f0bdfd3-de62-4466-94c4-14e7ab94d622.png#averageHue=%23f7f6f6&clientId=u95ff03d9-f8c4-4&from=paste&height=224&id=ue0ae5089&originHeight=224&originWidth=609&originalType=binary&ratio=1&rotation=0&showTitle=false&size=28248&status=done&style=none&taskId=u9f08ed94-bca2-44d5-91a1-073a0d0ead3&title=&width=609)
