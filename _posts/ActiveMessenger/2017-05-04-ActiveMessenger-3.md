---
layout: post
title: ActiveMQ传送机制以及ACK机制详解
categories: ActiveMessenger
description: 消息中间件
keywords: java, 消息中间件，ActiveMQ
---


# ActiveMQ消息传送机制 #


一条消息的生命周期如下-一条消息从producer端发出之后，一旦被broker正确保存，那么它将会被consumer消费，然后ACK，broker端才会删除；不过当消息过期或者存储设备溢出时，也会终结它：
![]({{ "/images/posts/ActiveMessenger/ActiveMessenger-3-1.jpg" | absolute_url }})

----------

![]({{ "/images/posts/ActiveMessenger/ActiveMessenger-3-2.jpg" | absolute_url }})

这张图片中简单的描述了:1)producer端如何发送消息 2) consumer端如何消费消息 3) broker端如何调度。如果用文字来描述图示中的概念，恐怕一言难尽。图示中，提及到prefetchAck，以及消息同步、异步发送的基本逻辑；这对你了解下文中的ACK机制将有很大的帮助。


# optimizeACK #

>  "可优化的ACK"，这是ActiveMQ对于consumer在消息消费时，对消息ACK的优化选项，也是consumer端最重要的优化参数之一，你可以通过如下方式开启:

## optimizeACK与prefetchSize的设置 ##

1) 在brokerUrl中增加如下查询字符串： 

    String brokerUrl = "tcp://localhost:61616?" +   
       "jms.optimizeAcknowledge=true" +   
       "&jms.optimizeAcknowledgeTimeOut=30000" +   
       "&jms.redeliveryPolicy.maximumRedeliveries=6";  
    ActiveMQConnectionFactory factory = new ActiveMQConnectionFactory(brokerUrl); 

2) 在destinationUri中，增加如下查询字符串：

    String queueName = "test-queue?customer.prefetchSize=100";  
    Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);  
    Destination queue = session.createQueue(queueName); 

我们需要在brokerUrl指定optimizeACK选项，在destinationUri中指定prefetchSize(预获取)选项，其中brokerUrl参数选项是全局的，即当前factory下所有的connection/session/consumer都会默认使用这些值；而destinationUri中的选项，只会在使用此destination的consumer实例中有效；如果同时指定，brokerUrl中的参数选项值将会被覆盖。

optimizeAck表示是否开启“优化ACK”，只有在为true的情况下，prefetchSize(下文中将会简写成prefetch)以及optimizeAcknowledgeTimeout参数才会有意义。此处需要注意"optimizeAcknowledgeTimeout"选项只能在brokerUrl中配置。

prefetch值建议在destinationUri中指定，因为在brokerUrl中指定比较繁琐；在brokerUrl中，queuePrefetchSize和topicPrefetchSize都需要单独设定："&jms.prefetchPolicy.queuePrefetch=12&jms.prefetchPolicy.topicPrefetch=12"等来逐个指定。

## prefetchSize设置的数值大小的意义 ##

如果prefetchACK为true，那么prefetch必须大于0；当prefetchACK为false时，你可以指定prefetch为0以及任意大小的正数。不过，当prefetch=0是，表示consumer将使用PULL(拉取)的方式从broker端获取消息，broker端将不会主动push消息给client端，直到client端发送PullCommand时；当prefetch>0时，就开启了broker push模式，此后只要当client端消费且ACK了一定的消息之后，会立即push给client端多条消息。

### 对于receive方法 ###

    当consumer端使用receive()方法同步获取消息时，prefetch可以为0和任意正值；当prefetch=0时，那么receive()方法将会首先发送一个PULL指令并阻塞，直到broker端返回消息为止，这也意味着消息只能逐个获取(类似于Request<->Response)，这也是Activemq中PULL消息模式；当prefetch > 0时，broker端将会批量push给client 一定数量的消息(<= prefetch),client端会把这些消息(unconsumedMessage)放入到本地的队列中，只要此队列有消息，那么receive方法将会立即返回，当一定量的消息ACK之后，broker端会继续批量push消息给client端

### 对于MessageListener接口的onMEssage() ###

    当consumer端使用MessageListener异步获取消息时，这就需要开发设定的prefetch值必须 >=1,即至少为1；在异步消费消息模式中，设定prefetch=0,是相悖的，也将获得一个Exception。


# optimizeACK和prefeth达成一个高效的消息消费模型 #

批量获取消息，并“延迟”确认(ACK)。prefetch表达了“批量获取”消息的语义，broker端主动的批量push多条消息给client端，总比client多次发送PULL指令然后broker返回一条消息的方式要优秀很多，它不仅减少了client端在获取消息时阻塞的次数和阻塞的时间，还能够大大的减少网络开支。optimizeACK表达了“延迟确认”的语义(ACK时机)，client端在消费消息后暂且不发送ACK，而是把它缓存下来(pendingACK)，等到这些消息的条数达到一定阀值时，只需要通过一个ACK指令把它们全部确认；这比对每条消息都逐个确认，在性能上要提高很多。由此可见，prefetch优化了消息传送的性能，optimizeACK优化了消息确认的性能。

## optimizeACK + prefetch如何设置提高性能 ##

> 1) 如果consumer端消费速度很慢(对消息的处理是耗时的)，过大的prefetchSize，并不能有效的提升性能，反而不利于consumer端的负载均衡(只针对queue)；按照良好的设计准则，当consumer消费速度很慢时，我们通常会部署多个consumer客户端，并使用较小的prefetch，同时关闭optimizeACK，可以让消息在多个consumer间“负载均衡”(即均匀的发送给每个consumer)；如果较大的prefetchSize，将会导致broker一次性push给client大量的消息，但是这些消息需要很久才能ACK(消息积压)，而且在client故障时，还会导致这些消息的重发。
>   
> 
> 2) 如果consumer端消费速度很快，但是producer端生成消息的速率较慢，比如生产者10秒钟生成10条消息，但是consumer一秒就能消费完毕，而且我们还部署了多个consumer！！这种场景下，建议开启optimizeACK，但是需要设置的prefetchSize不能过大；这样可以保证每个consumer都能有"活干"，否则将会出现一个consumer非常忙碌，但是其他consumer几乎收不到消息。
>  
> 
> 3) 如果消息很重要，特别是不愿意接收到”redelivery“的消息，那么我们需要将optimizeACK=false，prefetchSize=1

注意：既然optimizeACK是”延迟“确认，那么就引入一种潜在的风险：在消息被消费之后还没有来得及确认时，client端发生故障，那么这些消息就有可能会被重新发送给其他consumer，那么这种风险就需要client端能够容忍“重复”消息。

## optimizeACK延时确认的前提 ##

> 即使当optimizeACK为true，也只会当session的ACK模式为AUTO_ACKNOWLEDGE时才会生效，即在其他类型的ACK模式时consumer端仍然不会“延迟确认”
> 
> 当consumer.optimizeACK有效时，如果客户端已经消费但尚未确认的消息(deliveredMessage)达到prefetch * 0.65，consumer端将会自动进行ACK；同时如果离上一次ACK的时间间隔，已经超过"optimizeAcknowledgeTimout"毫秒，也会导致自动进行ACK。
> 
> 此外简单的补充一下，批量确认消息时，只需要在ACK指令中指明“firstMessageId”和“lastMessageId”即可，即消息区间，那么broker端就知道此consumer(根据consumerId识别)需要确认哪些消息。


# ACK模式与类型介绍 #

## 四种ACK模式 ##

    AUTO_ACKNOWLEDGE = 1自动确认
    CLIENT_ACKNOWLEDGE = 2客户端手动确认   
    DUPS_OK_ACKNOWLEDGE = 3自动批量确认
    SESSION_TRANSACTED = 0事务提交并确认
    此外AcitveMQ补充了一个自定义的ACK模式:
    INDIVIDUAL_ACKNOWLEDGE = 4单条消息确认

### AUTO_ACKNOWLEDGE ###
自动确认,这就意味着消息的确认时机将有consumer择机确认."择机确认"似乎充满了不确定性,这也意味着,开发者必须明确知道"择机确认"的具体时机,否则将有可能导致消息的丢失,或者消息的重复接收.那么在ActiveMQ中,AUTO_ACKNOWLEDGE是如何运作的呢?
自动确认,这就意味着消息的确认时机将有consumer择机确认."择机确认"似乎充满了不确定性,这也意味着,开发者必须明确知道"择机确认"的具体时机,否则将有可能导致消息的丢失,或者消息的重复接收.那么在ActiveMQ中,AUTO_ACKNOWLEDGE是如何运作的呢?

    1) 对于consumer而言，optimizeAcknowledge属性只会在AUTO_ACK模式下有效。

 

    2) 其中DUPS_ACKNOWLEGE也是一种潜在的AUTO_ACK,只是确认消息的条数和时间上有所不同。

 

    3) 在“同步”(receive)方法返回message之前,会检测optimizeACK选项是否开启，如果没有
       开启，此单条消息将立即确认，所以在这种情况下，message返回之后，如果开发者在处理
       message过程中出现异常，会导致此消息也不会redelivery,即"潜在的消息丢失"；如果
       开启了optimizeACK，则会在unAck数量达到prefetch * 0.65时确认，当然我们可以指
       定prefetchSize = 1来实现逐条消息确认。

 

    4) 在"异步"(messageListener)方式中,将会首先调用listener.onMessage(message),此
       后再ACK,如果onMessage方法异常,将导致client端补充发送一个ACK_TYPE为
       REDELIVERED_ACK_TYPE确认指令；如果onMessage方法正常,消息将会正常确认
       (STANDARD_ACK_TYPE)。此外需要注意，消息的重发次数是有限制的，每条消息中都会包
       含“redeliveryCounter”计数器，用来表示此消息已经被重发的次数，如果重发次数达到
       阀值，将会导致发送一个ACK_TYPE为POSION_ACK_TYPE确认指令,这就导致broker端认为
       此消息无法消费,此消息将会被删除或者迁移到"dead letter"通道中。

    

    因此当我们使用messageListener方式消费消息时，通常建议在onMessage方法中使用try-
    catch,这样可以在处理消息出错时记录一些信息，而不是让consumer不断去重发消息；如果你
    没有使用try-catch,就有可能会因为异常而导致消息重复接收的问题,需要注意你的onMessage
    方法中逻辑是否能够兼容对重复消息的判断。

![]({{ "/images/posts/ActiveMessenger/ActiveMessenger-3-3.jpg" | absolute_url }})


### CLIENT_ACKNOWLEDGE ###

客户端手动确认，这就意味着AcitveMQ将不会“自作主张”的为你ACK任何消息，开发者需要自己择机确认。在此模式下，开发者需要需要关注几个方法：1) message.acknowledge()，2) ActiveMQMessageConsumer.acknowledege()，3) ActiveMQSession.acknowledge()；其1)和3)是等效的，将当前session中所有consumer中尚未ACK的消息都一起确认，2)只会对当前consumer中那些尚未确认的消息进行确认。开发者可以在合适的时机必须调用一次上述方法。为了避免混乱，对于这种ACK模式下，建议一个session下只有一个consumer。

 

我们通常会在基于Group(消息分组)情况下会使用CLIENT_ACKNOWLEDGE，我们将在一个group的消息序列接受完毕之后确认消息(组)；不过当你认为消息很重要，只有当消息被正确处理之后才能确认时，也可以使用此模式  。



如果开发者忘记调用acknowledge方法，将会导致当consumer重启后，会接受到重复消息，因为对于broker而言，那些尚未真正ACK的消息被视为“未消费”。

开发者可以在当前消息处理成功之后，立即调用message.acknowledge()方法来"逐个"确认消息，这样可以尽可能的减少因网络故障而导致消息重发的个数；当然也可以处理多条消息之后，间歇性的调用acknowledge方法来一次确认多条消息，减少ack的次数来提升consumer的效率，不过这仍然是一个利弊权衡的问题。



除了message.acknowledge()方法之外，ActiveMQMessageConumser.acknowledge()和ActiveMQSession.acknowledge()也可以确认消息，只不过前者只会确认当前consumer中的消息。其中sesson.acknowledge()和message.acknowledge()是等效的。



无论是“同步”/“异步”，ActiveMQ都不会发送STANDARD_ACK_TYPE，直到message.acknowledge()调用。如果在client端未确认的消息个数达到prefetchSize * 0.5时，会补充发送一个ACK_TYPE为DELIVERED_ACK_TYPE的确认指令，这会触发broker端可以继续push消息到client端。(参看PrefetchSubscription.acknwoledge方法)



在broker端，针对每个Consumer，都会保存一个因为"DELIVERED_ACK_TYPE"而“拖延”的消息个数，这个参数为prefetchExtension，事实上这个值不会大于prefetchSize * 0.5,因为Consumer端会严格控制DELIVERED_ACK_TYPE指令发送的时机(参见ActiveMQMessageConsumer.ackLater方法)，broker端通过“prefetchExtension”与prefetchSize互相配合，来决定即将push给client端的消息个数，count = prefetchExtension + prefetchSize - dispatched.size()，其中dispatched表示已经发送给client端但是还没有“STANDARD_ACK_TYPE”的消息总量；由此可见，在CLIENT_ACK模式下，足够快速的调用acknowledge()方法是决定consumer端消费消息的速率；如果client端因为某种原因导致acknowledge方法未被执行，将导致大量消息不能被确认，broker端将不会push消息，事实上client端将处于“假死”状态，而无法继续消费消息。我们要求client端在消费1.5*prefetchSize个消息之前，必须acknowledge()一次；通常我们总是每消费一个消息调用一次，这是一种良好的设计。



此外需要额外的补充一下：所有ACK指令都是依次发送给broker端，在CLIET_ACK模式下，消息在交付给listener之前，都会首先创建一个DELIVERED_ACK_TYPE的ACK指令，直到client端未确认的消息达到"prefetchSize * 0.5"时才会发送此ACK指令，如果在此之前，开发者调用了acknowledge()方法，会导致消息直接被确认(STANDARD_ACK_TYPE)。broker端通常会认为“DELIVERED_ACK_TYPE”确认指令是一种“slow consumer”信号，如果consumer不能及时的对消息进行acknowledge而导致broker端阻塞，那么此consumer将会被标记为“slow”，此后queue中的消息将会转发给其他Consumer。

### DUPS_OK_ACKNOWLEDGE ###

消息可重复"确认，意思是此模式下，可能会出现重复消息，并不是一条消息需要发送多次ACK才行。它是一种潜在的"AUTO_ACK"确认机制，为批量确认而生，而且具有“延迟”确认的特点。对于开发者而言，这种模式下的代码结构和AUTO_ACKNOWLEDGE一样，不需要像CLIENT_ACKNOWLEDGE那样调用acknowledge()方法来确认消息。

 

    1) 在ActiveMQ中，如果在Destination是Queue通道，我们真的可以认为DUPS_OK_ACK就是“AUTO_ACK + optimizeACK + (prefetch > 0)”这种情况，在确认时机上几乎完全一致；此外在此模式下，如果prefetchSize =1 或者没有开启optimizeACK，也会导致消息逐条确认，从而失去批量确认的特性。

 

    2) 如果Destination为Topic，DUPS_OK_ACKNOWLEDGE才会产生JMS规范中诠释的意义，即无论optimizeACK是否开启，都会在消费的消息个数>=prefetch * 0.5时，批量确认(STANDARD_ACK_TYPE),在此过程中，不会发送DELIVERED_ACK_TYPE的确认指令,这是1)和AUTO_ACK的最大的区别。

 

    这也意味着，当consumer故障重启后，那些尚未ACK的消息会重新发送过来。

### SESSION_TRANSACTED ###

当session使用事务时，就是使用此模式。在事务开启之后，和session.commit()之前，所有消费的消息，要么全部正常确认，要么全部redelivery。这种严谨性，通常在基于GROUP(消息分组)或者其他场景下特别适合。在SESSION_TRANSACTED模式下，optimizeACK并不能发挥任何效果,因为在此模式下，optimizeACK会被强制设定为false，不过prefetch仍然可以决定DELIVERED_ACK_TYPE的发送时机。

 

因为Session非线程安全，那么当前session下所有的consumer都会共享同一个transactionContext；同时建议，一个事务类型的Session中只有一个Consumer，以避免rollback()或者commit()方法被多个consumer调用而造成的消息混乱。



当consumer接受到消息之后，首先检测TransactionContext是否已经开启，如果没有，就会开启并生成新的transactionId，并把信息发送给broker；此后将检测事务中已经消费的消息个数是否 >= prefetch * 0.5,如果大于则补充发送一个“DELIVERED_ACK_TYPE”的确认指令；这时就开始调用onMessage()方法，如果是同步(receive),那么即返回message。上述过程，和其他确认模式没有任何特殊的地方。



当开发者决定事务可以提交时，必须调用session.commit()方法，commit方法将会导致当前session的事务中所有消息立即被确认；事务的确认过程中，首先把本地的deliveredMessage队列中尚未确认的消息全部确认(STANDARD_ACK_TYPE)；此后向broker发送transaction提交指令并等待broker反馈，如果broker端事务操作成功，那么将会把本地deliveredMessage队列清空，新的事务开始；如果broker端事务操作失败(此时broker已经rollback)，那么对于session而言，将执行inner-rollback，这个rollback所做的事情，就是将当前事务中的消息清空并要求broker重发(REDELIVERED_ACK_TYPE),同时commit方法将抛出异常。



当session.commit方法异常时，对于开发者而言通常是调用session.rollback()回滚事务(事实上开发者不调用也没有问题)，当然你可以在事务开始之后的任何时机调用rollback(),rollback意味着当前事务的结束，事务中所有的消息都将被重发。需要注意，无论是inner-rollback还是调用session.rollback()而导致消息重发，都会导致message.redeliveryCounter计数器增加，最终都会受限于brokerUrl中配置的"jms.redeliveryPolicy.maximumRedeliveries",如果rollback的次数过多，而达到重发次数的上限时，消息将会被DLQ(dead letter)。

### INDIVIDUAL_ACKNOWLEDGE ###

单条消息确认，这种确认模式，我们很少使用，它的确认时机和CLIENT_ACKNOWLEDGE几乎一样，当消息消费成功之后，需要调用message.acknowledege来确认此消息(单条)，而CLIENT_ACKNOWLEDGE模式先message.acknowledge()方法将导致整个session中所有消息被确认(批量确认)。

## ACK_TYPE ##

Client端指定了ACK模式,但是在Client与broker在交换ACK指令的时候,还需要告知ACK_TYPE,ACK_TYPE表示此确认指令的类型，不同的ACK_TYPE将传递着消息的状态，broker可以根据不同的ACK_TYPE对消息进行不同的操作。


    DELIVERED_ACK_TYPE = 0消息"已接收"，但尚未处理结束
    STANDARD_ACK_TYPE = 2"标准"类型,通常表示为消息"处理成功"，broker端可以删除消息了
    POSION_ACK_TYPE = 1消息"错误",通常表示"抛弃"此消息，比如消息重发多次后，都无法正确处理时，消息将会被删除或者DLQ(死信队列)
    REDELIVERED_ACK_TYPE = 3消息需"重发"，比如consumer处理消息时抛出了异常，broker稍后会重新发送此消息
    INDIVIDUAL_ACK_TYPE = 4表示只确认"单条消息",无论在任何ACK_MODE下
    UNMATCHED_ACK_TYPE = 5在Topic中，如果一条消息在转发给“订阅者”时，发现此消息不符合Selector过滤条件，那么此消息将 不会转发给订阅者，消息将会被存储引擎删除(相当于在Broker上确认了消息)。


![]({{ "/images/posts/ActiveMessenger/ActiveMessenger-3-4.jpg" | absolute_url }})

## 消息确认的时机 ##

Consumer消费消息的风格有2种: 同步/异步..使用consumer.receive()就是同步，使用messageListener就是异步；在同一个consumer中，我们不能同时使用这2种风格，比如在使用listener的情况下，当调用receive()方法将会获得一个Exception。两种风格下，消息确认时机有所不同。

### 对于receive方法 ###

同步调用时，在消息从receive方法返回之前，就已经调用了ACK；因此如果Client端没有处理成功，此消息将丢失(可能重发，与ACK模式有关)。


    //receive伪代码---过程  
    Message message = sessionMessageQueue.dequeue();  
    if(message != null){  
    ack(message);  
    }  
    return message 

### 对于onMessage方法 ###

基于异步调用时，消息的确认是在onMessage方法返回之后，如果onMessage方法异常，会导致消息不能被ACK，会触发重发。

    //基于listener  
    Session session = connection.getSession(consumerId);  
    sessionQueueBuffer.enqueue(message);  
    Runnable runnable = new Ruannale(){  
    run(){  
    Consumer consumer = session.getConsumer(consumerId);  
    Message md = sessionQueueBuffer.dequeue();  
    try{  
    consumer.messageListener.onMessage(md);  
    ack(md);//  
    }catch(Exception e){  
    redelivery();//sometime，not all the time;  
    }  
    }  
    //session中将采取线程池的方式，分发异步消息  
    //因此同一个session中多个consumer可以并行消费  
    threadPool.execute(runnable);  














