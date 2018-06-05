---
published: false
---
## How to implement JMS ReplyTo using SpringBoot

Request-Response is a [message-exchange-pattern](https://en.wikipedia.org/wiki/Messaging_pattern). In some cases, a message producer may want the consumers to reply to a message. The JMSReplyTo header indicates which destination, if any, a JMS consumer should reply to. The JMSReplyTo header is set explicitly by the JMS client; its contents will be a javax.jms.Destination object (either Topic or Queue).

In some cases, the JMS client will want the message consumers to reply to a temporary topic or queue set up by the JMS client. When a JMS message consumer receives a message that includes a JMSReplyTo destination, it can reply using that destination. A JMS consumer is not required to send a reply, but in some JMS applications, clients are programmed to do so.

For simplicity, this pattern is typically implemented in a purely synchronous fashion, as in web service calls over HTTP, which holds a connection open and waits until the response is delivered or the timeout period expires. However, request–response may also be implemented asynchronously, with a response being returned at some unknown later time.

For more information, check [here](https://en.wikipedia.org/wiki/Request%E2%80%93response). 

Now, let's jump into the code. In Spring, there are 2 ways to implement this (at least I know of).
1. Using [JMSTemplate](https://github.com/spring-projects/spring-framework/blob/master/src/docs/asciidoc/integration.adoc#jms-jmstemplate)
1. Using [Spring Integration](http://spring.io/projects/spring-integration)

For demo purpose, I used [ActiveMQ](http://activemq.apache.org/). However, you can implement this in other messaging systems like IBM MQ, Rabbit MQ, Tibco EMS, etc. In this demo, I send an ObjectMessage of type _Order_ and reply with a _Shipment_ object.

### Using JMSTemplate
1. First, we include the required dependencies. Replace the `activemq` dependency with your messaging system's jars if not using ActiveMQ

	```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-activemq</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.activemq.tooling</groupId>
            <artifactId>activemq-junit</artifactId>
            <version>${activemq.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
	```
1. Using the default **spring.activemq.** properties to configure the application with the ActiveMQ. However, you can do this inside a **@Configuration** class as well.

	```yml
    spring:
      activemq:
        broker-url: tcp://localhost:61616
        non-blocking-redelivery: true
        packages:
          trust-all: true    
    ```
1. Note in the above configuration _spring.activemq.packages.trust-all_ can be changed to _spring.activemq.packages.trusted_ with the appropriate packages.
1. Now spring will do it's magic and inject all the required Beans as usual :) However, in our code, we need to _EnableJms_

	```java
    import org.springframework.context.annotation.Configuration;
    import org.springframework.jms.annotation.EnableJms;

    @EnableJms
    @Configuration
    public class ActiveMQConfig {

        public static final String ORDER_QUEUE = "order-queue";
        public static final String ORDER_REPLY_2_QUEUE = "order-reply-2-queue";

    }
	```
1. First, we will configure the **Producer**

	```java
    @Slf4j
    @Service
    public class Producer {

        @Autowired
        JmsMessagingTemplate jmsMessagingTemplate;

        @Autowired
        JmsTemplate jmsTemplate;

        public Shipment sendWithReply(Order order) throws JMSException {
            jmsTemplate.setReceiveTimeout(1000L);
            jmsMessagingTemplate.setJmsTemplate(jmsTemplate);

            Session session = jmsMessagingTemplate.getConnectionFactory().createConnection()
                    .createSession(false, Session.AUTO_ACKNOWLEDGE);

            ObjectMessage objectMessage = session.createObjectMessage(order);

            objectMessage.setJMSCorrelationID(UUID.randomUUID().toString());
            objectMessage.setJMSReplyTo(new ActiveMQQueue(ORDER_REPLY_2_QUEUE));
            objectMessage.setJMSCorrelationID(UUID.randomUUID().toString());
            objectMessage.setJMSExpiration(1000L);
            objectMessage.setJMSDeliveryMode(DeliveryMode.NON_PERSISTENT);

            return jmsMessagingTemplate.convertSendAndReceive(new ActiveMQQueue(ORDER_QUEUE),
                    objectMessage, Shipment.class); //this operation seems to be blocking + sync
        }


    }
    ```
1. Note in the above code that, [JmsMessagingTemplate](https://docs.spring.io/spring-framework/docs/4.3.x/javadoc-api/org/springframework/jms/core/JmsMessagingTemplate.html) is used instead of _JmsTemplate_ because, we are interested in the method _convertSendAndReceive_