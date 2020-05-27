# RocketＭＱ 分布式事务

## 什么是事务消息

可以将其视为两阶段提交消息实现，以确保分布式系统中的**最终一致性**。事务性消息确保本地事务的执行和消息的发送可以自动执行。

## 使用约束

1. 事务的消息没有调度和批处理支持。
2. 为了避免单个消息多次检查，导致一半队列消息积累，我们限制单个消息默认检查15次，用户可以通过修改 broker 配置中 `transactionCheckMax` 参数来修改这个限制，如果一个消息一直被检查超过 `transactionCheckMax` 次，broker 会默认丢弃该消息，并同时打印一个错误日志。用户可以通过重写 `AbstractTransactionCheckListener` 来自定义超时处理方式。
3. 在 broker 配置中指定的参数 `transactionTimeout` 确定的一段时间后，检查事务性消息。用户也可以在发送消息是，通过设置设置用户属性 `CHECK_IMMUNITY_TIME_IN_SECONDS` 来改变这个限制。当发送事务性消息时，该参数优先于 `transactionMsgTimeout` 参数。
4. 事务性消息可能被多次检查或使用。
5. 提交的消息重新放置到用户的目标 Topic 可能失败。目前，它取决于日志记录。高可用性是由RocketMQ本身的高可用性机制确保的。如果您希望确保事务消息没有丢失，并且事务完整性得到了保证，那么建议使用同步双写机制。
6. 事务性消息的生产者id不能与其他类型消息的生产者id共享。与其他类型的消息不同，事务性消息允许向后查询。MQ服务器根据其生产者id查询客户端。

## 应用程序

1. 事务状态

事务消息有三种状态

- `TransactionStatus.CommitTransaction`：提交事务，这意味着允许消费者消费此消息。
- `TransactionStatus.RollbackTransaction`：回滚事务，这意味着消息将被删除，不允许使用。
- `TransactionStatus.Unknown`：中间状态，这意味着MQ需要检查回来以确定状态。

2. 发送事务消息
   1. 创建事务生产者
      使用 `TransactionMQProducer` 类创建生产者客户端，并指定一个独立的 `producerGroup`，你可以设置一个自定义线程池处理检查请求。等到执行完本地事务之后，你需要根据执行结果回复 MQ，响应状态在上一节中也已经进行了描述

~~~java
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.common.message.MessageExt;
import java.util.List;

public class TransactionProducer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        TransactionListener transactionListener = new TransactionListenerImpl();
        TransactionMQProducer producer = new TransactionMQProducer("please_rename_unique_group_name");
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("client-transaction-msg-check-thread");
                return thread;
            }
        });

        producer.setExecutorService(executorService);
        producer.setTransactionListener(transactionListener);
        producer.start();

        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 10; i++) {
            try {
                Message msg =
                    new Message("TopicTest1234", tags[i % tags.length], "KEY" + i,
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.sendMessageInTransaction(msg, null);
                System.out.printf("%s%n", sendResult);

                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }

        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }
        producer.shutdown();
    }
}
~~~

2. 实现 `TransactionListener` 接口

   `executeLocalTransaction` 方法用于在发送半开消息成功时执行本地事务。它返回前一节中提到的三个事务状态之一。

   `checkLocalTransaction` 方法用于检查本地事务状态并响应MQ检查请求。它还返回前一节中提到的三个事务状态之一。

~~~java
   import ...
   
   public class TransactionListenerImpl implements TransactionListener {
       private AtomicInteger transactionIndex = new AtomicInteger(0);
   
       private ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();
   
       @Override
       public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
           int value = transactionIndex.getAndIncrement();
           int status = value % 3;
           localTrans.put(msg.getTransactionId(), status);
           return LocalTransactionState.UNKNOW;
       }
   
       @Override
       public LocalTransactionState checkLocalTransaction(MessageExt msg) {
           Integer status = localTrans.get(msg.getTransactionId());
           if (null != status) {
               switch (status) {
                   case 0:
                       return LocalTransactionState.UNKNOW;
                   case 1:
                       return LocalTransactionState.COMMIT_MESSAGE;
                   case 2:
                       return LocalTransactionState.ROLLBACK_MESSAGE;
               }
           }
           return LocalTransactionState.COMMIT_MESSAGE;
       }
   }
~~~