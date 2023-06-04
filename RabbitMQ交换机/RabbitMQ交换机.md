在上一节中，我们创建了一个工作队列。我们假设的是工作队列背后，每个任务都恰好交付给一个消费者(工作进程)。在这一部分中，我们将做一些完全不同的事情-我们将消息传达给多个消费者。这种模式 称为 ”发布/订阅”。
# exchange
RabbitMQ消息传递模型的核心思想是：**生产者生产的消息从不会直接发送到队列。**实际上，通常生产者甚至都不知道这些消息传递传递到了哪些队列中。
相反，**生产者只能将消息发送到交换机（exchange）**，交换机工作的内容非常简单，一方面它接收来自生产者的消息，另一方面将它们推入队列。交换机必须确切知道如何处理收到的消息。是应该把这些消息放到特定队列还是说把他们放到许多队列中还是应该丢弃它们。这就是由交换机的类型来决定。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685863552299-2149d4b1-3fa1-4c31-bb44-07eb8bd8518e.png#averageHue=%23f5dcd9&clientId=ue8dfb588-834b-4&from=paste&height=200&id=u71a56945&originHeight=200&originWidth=567&originalType=binary&ratio=1&rotation=0&showTitle=false&size=30395&status=done&style=none&taskId=u53c342b4-8d55-4a67-b6bc-df8ef1b6851&title=&width=567)
**exchange的类型：**
direct、topic、headers、fanout
**无名exchange：**
在前面部分我们对 exchange 一无所知，但仍然能够将消息发送到队列。之前能实现的 原因是因为我们使用的是默认交换，我们通过空字符串(“”)进行标识。
```java
channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
```
第一个参数是交换机的名称。空字符串表示默认或无名称交换机：消息能路由发送到队列中其实是由 routingKey(bindingkey)绑定 key 指定的，如果它存在的话
# 临时队列
之前的章节我们使用的是具有特定名称的队列（创建了hello和ack_queue）。队列的名称我们来说至关重要，我们需要指定我们的消费者去消费哪个队列的消息。
每当我们连接到 Rabbit 时，我们都需要一个全新的空队列，为此我们可以创建一个具有**随机名称的队列**，或者能让服务器为我们选择一个随机队列名称那就更好了。其次一旦我们断开了消费者的连接，队列将被自动删除。
创建临时队列的方式如下:
```java
String queueName = channel.queueDeclare().getQueue();
```
# 绑定 bindings
什么是binding呢，binding其实是exchange和queue之间的桥梁，它告诉我们 exchange 和那个队列进行了绑定关系。比如说下面这张图告诉我们的就是 X 与 Q1 和 Q2 进行了绑定
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685863810324-c3af4afd-7ee6-4dd8-83be-1d05afbdf20d.png#averageHue=%23f7e6e4&clientId=ue8dfb588-834b-4&from=paste&height=141&id=uee44384e&originHeight=141&originWidth=432&originalType=binary&ratio=1&rotation=0&showTitle=false&size=27641&status=done&style=none&taskId=u733fc769-448e-4328-b084-5db7ff63876&title=&width=432)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685863817350-fb671a9c-a7f6-4969-a6c2-035384247d5c.png#averageHue=%23f5f5f5&clientId=ue8dfb588-834b-4&from=paste&height=298&id=ud2bc455a&originHeight=298&originWidth=948&originalType=binary&ratio=1&rotation=0&showTitle=false&size=96508&status=done&style=none&taskId=u2d5a42c6-2a62-4379-8aab-5d9af4a266b&title=&width=948)
# Fanout exchange
## Fanout介绍
Fanout 这种类型非常简单。正如从名称中猜到的那样，它是将接收到的所有消息广播到它知道的 所有队列中。系统中默认有些 exchange 类型
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685863887684-8caf5b6f-b663-4754-ba9f-4dd0c4f067b4.png#averageHue=%23f3f3f2&clientId=ue8dfb588-834b-4&from=paste&height=381&id=u042e4f6d&originHeight=381&originWidth=619&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26074&status=done&style=none&taskId=u4b82e4c5-d2b4-4f71-9c1c-8ae500c7424&title=&width=619)
## Fanout实战
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685864235435-441e8cfd-b69d-4b57-959f-e24bfc0c44b1.png#averageHue=%23f6ebea&clientId=ue8dfb588-834b-4&from=paste&height=180&id=u7dbd4323&originHeight=180&originWidth=626&originalType=binary&ratio=1&rotation=0&showTitle=false&size=82601&status=done&style=none&taskId=u441d3b5f-3c92-4d05-8ecc-69625302eef&title=&width=626)
Logs和临时队列的绑定关系如下图
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685864249331-1a290c21-ff40-4808-b591-5d39c24d06cf.png#averageHue=%23f6f5f5&clientId=ue8dfb588-834b-4&from=paste&height=263&id=uc2fad3de&originHeight=263&originWidth=595&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13973&status=done&style=none&taskId=u77402111-64f7-4ddd-87c3-420c6f18b24&title=&width=595)
为了说明这种模式，我们将构建一个简单的日志系统。它将由两个程序组成:第一个程序将发出日志消 息，第二个程序是消费者。其中我们会启动两个消费者，其中一个消费者接收到消息后把日志存储在磁盘，
ReceiveLogs01 将接收到的消息打印在控制台
```java
public class ReceiveLogs01 {

    private static final String EXCHANGE_NAME = "logs";

    public static void main(String[] args) throws Exception {

        Channel channel = RabbitMQUtils.getChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
        /**
         * 生成一个临时的队列 队列的名称是随机的
         * 当消费者断开和该队列的连接时 队列自动删除
         */
        String queueName = channel.queueDeclare().getQueue();
        //把该临时队列绑定我们的 exchange 其中 routingkey(也称之为 binding key)为空字符串
        channel.queueBind(queueName, EXCHANGE_NAME, "");
        System.out.println("等待接收消息,把接收到的消息打印在屏幕........... ");

        //发送回调
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println("控制台打印接收到的消息" + message);
        };
        channel.basicConsume(queueName, true, deliverCallback, consumerTag -> {});

    }

}
```
ReceiveLogs02 把消息写出到文件
```java
public class ReceiveLogs02 {

    private static final String EXCHANGE_NAME = "logs";

    public static void main(String[] args) throws Exception {

        Channel channel = RabbitMQUtils.getChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
        /**
         * 生成一个临时的队列 队列的名称是随机的
         * 当消费者断开和该队列的连接时 队列自动删除
         */
        String queueName = channel.queueDeclare().getQueue();
        //把该临时队列绑定我们的 exchange 其中 routingkey(也称之为 binding key)为空字符串
        channel.queueBind(queueName, EXCHANGE_NAME, "");
        System.out.println("等待接收消息,把接收到的消息写到文件........... ");

        //发送回调
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            File file = new File("D:\\test\\rabbitmq_info.txt");
            FileUtils.writeStringToFile(file,message,"UTF-8");
            System.out.println("数据写入文件成功");
        };
        channel.basicConsume(queueName, true, deliverCallback, consumerTag -> {});

    }

}
```
EmitLog 发送消息给两个消费者接收：
```java
public class EmitLog {

    private static final String EXCHANGE_NAME = "logs";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitMQUtils.getChannel();

        /**
         * 声明一个 exchange
         * 1.exchange 的名称
         * 2.exchange 的类型
         */
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
        Scanner sc = new Scanner(System.in);
        System.out.println("请输入信息");
        while (sc.hasNext()) {
            String message = sc.nextLine();
            channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes("UTF-8"));
            System.out.println("生产者发出消息" + message);
        }
    }

}
```
# Direct exchange
在上一节中，我们构建了一个简单的日志记录系统。我们能够向许多接收者广播日志消息。在本节我们将向其中添加一些特别的功能——让某个消费者订阅发布的部分消息。例如我们只把严重错误消息定向存储到日志文件(以节省磁盘空间)，同时仍然能够在控制台上打印所有日志消息。
我们再次来回顾一下什么是 bindings，绑定是交换机和队列之间的桥梁关系。也可以这么理解： **队列只对它绑定的交换机的消息感兴趣**。绑定用参数：routingKey 来表示也可称该参数为 binding key， 创建绑定我们用代码:channel.queueBind(queueName, EXCHANGE_NAME, "routingKey");
绑定之后的意义由其交换类型决定。
## Direct介绍
上一节中的我们的日志系统将所有消息广播给所有消费者，对此我们想做一些改变，例如我们希 望将日志消息写入磁盘的程序仅接收严重错误(errros)，而不存储哪些警告(warning)或信息(info)日志 消息避免浪费磁盘空间。Fanout 这种交换类型并不能给我们带来很大的灵活性-它只能进行无意识的 广播，在这里我们将使用 direct 这种类型来进行替换，这种类型的工作方式是，消息只去到它绑定的 routingKey 队列中去。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685864826362-ddacbabf-959c-48be-a0bb-1e4d3d543ade.png#averageHue=%23f5e4e2&clientId=ue8dfb588-834b-4&from=paste&height=226&id=ube3211db&originHeight=226&originWidth=645&originalType=binary&ratio=1&rotation=0&showTitle=false&size=62016&status=done&style=none&taskId=ua2880ee9-287a-45b2-b691-e99cc8a6c99&title=&width=645)
在上面这张图中，我们可以看到 X 绑定了两个队列，绑定类型是 direct。队列Q1 绑定键为 orange， 队列 Q2 绑定键有两个:一个绑定键为 black，另一个绑定键为 green.
在这种绑定情况下，生产者发布消息到 exchange 上，绑定键为 orange 的消息会被发布到队列 Q1。绑定键为 blackgreen 和的消息会被发布到队列 Q2，其他消息类型的消息将被丢弃。
## 多重绑定
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685864847434-a110918a-5858-427c-845d-fd46a4fde41a.png#averageHue=%23f7eae9&clientId=ue8dfb588-834b-4&from=paste&height=172&id=u67b137c9&originHeight=172&originWidth=550&originalType=binary&ratio=1&rotation=0&showTitle=false&size=32353&status=done&style=none&taskId=u3db0f43d-5b72-4bb3-9cdc-a659a647c06&title=&width=550)
当然如果 exchange 的绑定类型是direct，**但是它绑定的多个队列的 key 如果都相同**，在这种情况下虽然绑定类型是 direct **但是它表现的就和 fanout 有点类似了**，就跟广播差不多，如上图所示。
## Direct实战
关系：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685864867782-6d75766b-9dff-41c6-9e06-0307bc17ec2e.png#averageHue=%23f8efee&clientId=ue8dfb588-834b-4&from=paste&height=262&id=u4ca0ba1f&originHeight=262&originWidth=553&originalType=binary&ratio=1&rotation=0&showTitle=false&size=72333&status=done&style=none&taskId=u5de0ac7f-20b1-4b71-bc03-3da8dad5b2a&title=&width=553)
交换机：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685864877984-7c67a4d6-d75b-4f51-8037-ab4d6dd8ffa6.png#averageHue=%23f3f3f2&clientId=ue8dfb588-834b-4&from=paste&height=310&id=u50efd707&originHeight=310&originWidth=423&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13959&status=done&style=none&taskId=u7650cd9a-8c0c-4b77-9b28-f3b995e3418&title=&width=423)
c2：绑定disk，routingKey为error
c1：绑定console，routingKey为info、warning
ReceiveLogsDirect01：
```java
public class ReceiveLogsDirect01 {
    private static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitMqUtils.getChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);

        String queueName = "disk";
        //队列声明
        channel.queueDeclare(queueName, false, false, false, null);
        //队列绑定
        channel.queueBind(queueName, EXCHANGE_NAME, "error");
        System.out.println("等待接收消息...");

        //发送回调
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            message = "接收绑定键:" + delivery.getEnvelope().getRoutingKey() + ",消息:" + message;
            System.out.println("error 消息已经接收：\n" + message);
        };
        channel.basicConsume(queueName, true, deliverCallback, consumerTag -> {
        });
    }
}
```
ReceiveLogsDirect02：
```java
public class ReceiveLogsDirect02 {
    private static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitMqUtils.getChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);

        String queueName = "console";
        //队列声明
        channel.queueDeclare(queueName, false, false, false, null);
        //队列绑定
        channel.queueBind(queueName, EXCHANGE_NAME, "info");
        channel.queueBind(queueName, EXCHANGE_NAME, "warning");

        System.out.println("等待接收消息...");

        //发送回调
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            message = "接收绑定键:" + delivery.getEnvelope().getRoutingKey() + ",消息:" + message;
            System.out.println("info和warning 消息已经接收：\n" + message);
        };
        channel.basicConsume(queueName, true, deliverCallback, consumerTag -> {
        });
    }
}
```
EmitLogDirect:
```java
public class EmitLogDirect {
    private static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitMqUtils.getChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);

        //创建多个 bindingKey
        Map<String, String> bindingKeyMap = new HashMap<>();
        bindingKeyMap.put("info", "普通 info 信息");
        bindingKeyMap.put("warning", "警告 warning 信息");
        bindingKeyMap.put("error", "错误 error 信息");
        //debug 没有消费这接收这个消息 所有就丢失了
        bindingKeyMap.put("debug", "调试 debug 信息");

        for (Map.Entry<String, String> bindingKeyEntry : bindingKeyMap.entrySet()) {
            //获取 key value
            String bindingKey = bindingKeyEntry.getKey();
            String message = bindingKeyEntry.getValue();

            channel.basicPublish(EXCHANGE_NAME, bindingKey, null, message.getBytes("UTF-8"));
            System.out.println("生产者发出消息:" + message);
        }
    }
}
```
# Topic exchange
## Topic的介绍
在上一个小节中，我们改进了日志记录系统。我们没有使用只能进行随意广播的 fanout 交换机，而是使用了 direct 交换机，从而有能实现有选择性地接收日志。
尽管使用 direct 交换机改进了我们的系统，但是它仍然存在局限性——比方说我们想接收的日志类型有 info.base 和 info.advantage，某个队列只想 info.base 的消息，那这个时候direct 就办不到了。这个时候就只能使用 **topic** 类型
> Topic的要求

发送到类型是 topic 交换机的消息的 routing_key 不能随意写，必须满足一定的要求，它必须是**一个单词列表**，**以点号分隔开**。这些单词可以是任意单词
比如说："stock.usd.nyse", "nyse.vmw", "quick.orange.rabbit".这种类型的。
当然这个单词列表最多不能超过 255 个字节。
在这个规则列表中，其中有两个替换符是大家需要注意的：

- ***(星号)可以代替一个单词**
- **#(井号)可以替代零个或多个单词**
## Topic匹配案例
绑定关系如下：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685865298194-637df0c9-e797-4730-8361-d37b930edc98.png#averageHue=%23f8efee&clientId=ue8dfb588-834b-4&from=paste&height=183&id=u5f7d5f9b&originHeight=183&originWidth=684&originalType=binary&ratio=1&rotation=0&showTitle=false&size=41227&status=done&style=none&taskId=ucae566be-eebb-49d5-8227-99ddef353c3&title=&width=684)

- Q1-->绑定的是
   - 中间带 orange 带 3 个单词的字符串` (*.orange.*)`
- Q2-->绑定的是
   - 最后一个单词是 rabbit 的 3 个单词` (*.*.rabbit)`
   - 第一个单词是 lazy 的多个单词` (lazy.#)`

上图是一个队列绑定关系图，我们来看看他们之间数据接收情况是怎么样的

|  例子  |  说明  |
| --- | --- |
| quick.orange.rabbit | 被队列 Q1Q2 接收到 |
| azy.orange.elephant | 被队列 Q1Q2 接收到 |
| quick.orange.fox | 被队列 Q1 接收到 |
| lazy.brown.fox | 被队列 Q2 接收到 |
| lazy.pink.rabbit | 虽然满足两个绑定但只被队列 Q2 接收一次 |
| quick.brown.fox | 不匹配任何绑定不会被任何队列接收到会被丢弃 |
| quick.orange.male.rabbit | 是四个单词不匹配任何绑定会被丢弃 |
| lazy.orange.male.rabbit | 是四个单词但匹配 Q2 |

注意：

- 当一个队列绑定键是#，那么这个队列将接收所有数据，就有点像 fanout 了
- 如果队列绑定键当中没有#和*出现，那么该队列绑定类型就是 direct 了
## Topic实战
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685865614320-15721efe-6366-447b-b405-f8c7271869c3.png#averageHue=%23f3f2f1&clientId=ue8dfb588-834b-4&from=paste&height=318&id=u3928a2eb&originHeight=318&originWidth=383&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14051&status=done&style=none&taskId=udc0c5267-e136-49ae-9456-7910c7741ee&title=&width=383)
EmitLogTopic：
