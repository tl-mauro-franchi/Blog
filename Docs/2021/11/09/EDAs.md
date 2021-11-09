# Event-Driven Architecture: Common issues and how to solve them

In the last decade we have witnessed an increase demand in real-time data processing systems which has forced companies to adopt a different architectural approach from the old fashioned, but still necessary, *Request-Driven* Architecture (typically implemented using [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) HTTP or [RPC](https://en.wikipedia.org/wiki/Remote_procedure_call) calls). 
Traditionally, these Request-Driven Architecture (RDA) systems have tight coupled components that directly call each other in order to retrieve or manipulate data, or execute commands which are not often compatible with the new modern requirements (such as real-time processing of stream of data and an improved user experience) of distributed microservices systems. The answer to these new consumer needs is given by _Event-driven_ Architectures (EDAs), where consumers can subscribe to an _event_ (or _command_) in order to receive updates or instructions in an asynchronous way, without needing to poll the server (now producer of the event) for them.  

Are EDAs the silver bullet that can guarantee a real-time communication between different components of the same system (or even between different systems) without any problem? Spoiler alert, of course not.

As per any other architectural approach, even the event-driven one can bring complexity and issues to the system that need to be addressed at design time in order to avoid future expensive refactoring.

In this article I will go through common issues I stumbled upon throughout my career when I designed distributed systems based on EDAs. For each issue I will provide a possible architectural solution that could address it and the benefits of choosing that approach. 

## Benefits of an EDA

Before listing the possible issues that you could face while using an EDA approach to design your system, I will first summarise the benefits an EDA brings.

#### Fast development and deployment of new service thanks to loose coupling

As opposed to the RDA approach, developing and deploying new services for an existing EDA is easier. 
As part of an EDA system, microservices are configured to consume events and perform single operations in total autonomy from other microservices. This independence between components of the same system is also known as _loose coupling_. The only things that a new service has to be aware of are the name and structure of the events that they need to consume, and where to get them, compared to RDAs, which require services to know the respective API endpoints in order to interact with each other (not to mention also making sure that the network configuration allows them to call one another). Loose coupling enables the EDA systems to _fire-and-forget_ their events, while for RDAs, APIs need to cope with the other APIs availability and response times. 
In an EDA there is only one centralised source for the events called _Message Broker_. The most popular message brokers are [RabbitMQ](https://www.rabbitmq.com/), [Apache Kafka](https://kafka.apache.org/), [Amazon SQS](https://aws.amazon.com/sqs/) and [Amazon SNS](https://aws.amazon.com/sns/?whats-new-cards.sort-by=item.additionalFields.postDateTime&whats-new-cards.sort-order=desc). Message brokers are responsible for acquiring, storing, and delivering events in the form of _messages_ to their consumers.

#### Real-time processing

Assuming that the producer of an event and all of its consumers are up and running at the same time, EDAs enable businesses to process streams of data and perform complex operations in _real-time_. Thereby improving the customer experience by maximizing the responsiveness of the applications involved in the system.

#### Fault tolerance

Loosely coupled components enable a system to be more tolerant to faults. Adding a new feature to a single component, or the system in general, has a lower risk overall and virtually no impact compared to adding it to a monolith or tightly coupled components. Similarly, if one of the components goes down, we can still guarantee the eventual delivery and processing of events. The faulty service can simply self-heal without blocking the rest of the system. Produced events will then be consumed and processed once the service is restored. This can be summarised as increased resiliency.

#### Low latency

As we mentioned above, EDAs reduce the need for point-to-point integration generally used for data sharing. EDA can also reduce the latency of accessing data from external resources by keeping a local read-only copy of that data based on the most recently processed events that will keep the local storage up to date. By using this read-only copy, services that are processing future events will be able to access the data without depending on external resources, thus reducing overall latency of those service.

#### High independent scalability

The consuming of the same type of event by multiple different consumers who subscribed to it has no impact on the performance of the producer, who only needs to make sure to asynchronously push the event to the broker, which will then guarantee delivery to the subscribers. 
On the other end, the number of subscriptions the consumer has can affect its performance. To solve this issue you could simply either increase the number of instances of the service ([horizontal scaling](https://en.wikipedia.org/wiki/Scalability#Horizontal_(scale_out)_and_vertical_scaling_(scale_up))), or split the service into multiple independent ones.
This approach of continues processing a stream of data (instead of batching it at regular intervals like it could happen in RDAs), also allows companies to predict the processing time of the events and thus the amount of infrastructure resources needed to operate. This can improve overall cost efficiency by avoiding over-provisioning resources for the system when not necessary. We could then say that EDAs systems use infrastructure resources (network and computational ones) in an efficient way by design.

## Common issues and problems with EDAs

Hopefully with the previous section I was able to show you why EDAs for large distributed systems are important and what benefits they can bring. Unfortunately _all that glitters is not gold_ and even EDAs can hide some issues that are better to address at design time. Let's see together what they are and how we could solve them, or at least contain their impact on the system.

#### Message handler failure

In an EDA system, a message handler is a component that subscribes to a given published event in order to consume and process it. A message handler is just a normal application that can depend on other resources like a database (for example to retrieve and/or update data) or external resources like REST APIs (components in a EDA system can still depend on external APIs). Like many other applications, it can fail while processing the event data (the message) pulled from the message broker (generally a queue).

_What happens to the message when its handler fails? 
How can the system recover from that and what are the best options we can implement to solve this issue?_

 In order to respond to these questions, first we need to identify whether the failure is a definitive one or not. A definitive failure is when the message can't be processed at all (for example, the message refers to an entity that doesn't exist in the domain, the message is malformed, etc). A non-definitive failure is when the error can be due to a temporary network issue like latency (or timeout) while connecting to a database or an external API. 
 Once we have identified the issue, we have to decide how to address it. Normally the handler should always have an internal retry or back-off policy for most network issues. Once the number of application retries are exhausted we can then try to send the message back to the queue for a predefined number of times before its retention period expires.

> _The retention period for an event defines the time a message is stored on the message broker before its discarded_

In both cases, when it is clear that nothing can be done (for example the database or the external API are down and after X attempts the message could not be processed), the message should be pushed to a _dead letter queue_.

> _In message queuing the dead letter queue is a service implementation to store messages that meet one or more of the following criteria: Message that is sent to a queue that does not exist. Queue length limit exceeded. Message length limit exceeded. Message is rejected by another queue exchange_ ([Wikipedia](https://en.wikipedia.org/wiki/Dead_letter_queue))

Once the message is sent to the dead queue, it cannot be consumed unless it gets manually resent to the original queue. It is important to use dead queue letters to avoid losing unprocessed messages due to human errors, like a bug in the code or in the network configuration, and to allow investigation of malformed messages as well.

#### Message ordering and duplicate messages

Let's consider the following example: you are building a large distributed account transactions tracker system for a bank and you need to guarantee that the transactions are processed in the same exact order they are executed. What we need in this case is either to design our own solution to achieve that or to base our solution on out-of the box first-in-first-out (FIFO) queues provided by the chosen message broker.
It could happen that the standard queues provided by some message brokers are not designed to guarantee the a message is always processed after the previous one. For example, [states](https://aws.amazon.com/sqs/faqs/) that:
> _Amazon SQS standard queues (the new name for existing queues) remain unchanged, and you can still create standard queues. These queues continue to provide the highest scalability and throughput; however, you will not get ordering guarantees and duplicates might occur._

On the other hand, most of the brokers also provide a guide to migrate the non-ordered queues to a type of queue that guarantees the messages are processed in the desired way (see [this](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-moving.html) about AWS for FIFO queues). 

So, whenever we need to design system that requires that every single event is processed in the same order as they occur, and possibly only once (in the example, for instance, you may not wish to process a transaction related to a payment more than once), it is better to choose the appropriate queue (or even message broker) that guarantees that.

Regarding duplicate messages, in case the _exactly-once processing_ feature for queues is not provided by the selected broker, another solution I implement in the past (before AWS started providing these kind of queues in [2016](https://aws.amazon.com/about-aws/whats-new/2016/11/amazon-sqs-introduces-fifo-queues-with-exactly-once-processing-and-lower-prices-for-standard-queues)) was to store the hash of the key information contained in the message payload. Sometimes the same event could be published multiple times with different IDs so storing just the ID would have been inadequate. For example, considering our account transaction tracker system, a possible set of key information from the transaction event could be the coordinates of the transaction, the amount, and the execution time. 
This would make our consumers of the event known as [Idempotent receivers](https://www.enterpriseintegrationpatterns.com/patterns/messaging/IdempotentReceiver.html).

#### Big messages

When designing an EDA system, one of the aspects to take in account is related to how big the messages can be. For instance, if we imagine that we have to design a food ordering system, we need to analyse whether we want to limit the number of basket items we want the customer to be allowed to put in their order or not, and how to represent them inside of an _"Order Created"_ event that communicates to the rest of the platform that an order has been placed.

We have now 4 different options before us in order to solve this issue.
1. **Select a message broker that allows us to send and receive messages big enough** to cope even with very hungry customers: 
   - AWS SQS/SNS message size limit is set to [256kb](https://aws.amazon.com/about-aws/whats-new/2013/06/18/amazon-sqs-announces-256KB-large-payloads);
   - RabbitMQ allows us to send a message payload of up to [512 MB](https://github.com/rabbitmq/rabbitmq-common/blob/master/include/rabbit.hrl#L250-L253). Obviously this is not recommended but it is still good that we have virtually no limits with this broker;
   - Kafka limits the size of its messages to [1MB](https://kafka.apache.org/documentation/#brokerconfigs_message.max.bytes).
2. **Design the message body to contain as little information as possible to reduce its size**. For example, the order details could only contain the IDs of the items and sub items (like extras for each item on a menu) that the customer has selected, rather than including every single piece of information (name, cost, description, etc) that could be retrieved by invoking another API or from a local projection (database) of the basket domain. The latter would make the solution more complex and the producer of our Order Created message would be tightly coupled with other components and/or resources. It could still be a valid solution though. It is up to you to evaluate the correct trade-off between reducing the message size and retrieving the order data from other resources. 
3. **Offload the actual payload of the message to a different data storage** and link it in the body of the message along with enough metadata fields that could be used by the consumers of the event to process it without accessing the actual payload (as not every consumer of the event may need access to the list of items that are part of the order). For example, a typical set of metadata fields for an order could be:
    ```json
    {
     "Id": 12345,
     "CustomerData": { .. },
     "RestaurantData": { .. },
     "OrderTime": "01/01/2000",
     "Total": 50.0
     "OrderPayload":"<link to external storage>"
    }
    ```
    Sometimes it could be possible that the SDK (or [extensions of it](https://docs.aws.amazon.com/sns/latest/dg/large-message-payloads.html)) of the selected message broker has implemented a logic that, when enabled and the message we want to publish exceeds the size limits, it automatically stores the payload to an external configurable storage (for AWS we could generally pick up [S3](https://aws.amazon.com/s3/?nc2=h_ql_prod_fs_s3)).

4. **Split the message into multiple processable chunks** (for example one for each order item) and let the consumers of them process each of them until the original message is recomposed. This solution can be implemented using a [splitter](https://www.enterpriseintegrationpatterns.com/Sequencer.html) and it could be tricky to implement given that the consumer of the partitioned message has to:
   - keep track of the chucks it received and processed;
   - store them somewhere in order to make sure that every instance of the service is reassemble the original message once the last sub-message is handled;
   - maybe process the recomposed message in case an even has to be published to state that the processing is completed.
 
    There are also some other pitfalls to this approach that have made me avoid it in the past, especially if you need to execute something when every single individual message is received. For example: 
_What happens when one or multiple chucks are lost? 
How can we retrieve them? What if the last piece of the message takes too long to arrive and the message cannot be processed anymore?_
 
All the above solutions are generally good solutions to consider in order to address a large message problem but, like the previous exposed problems, they have to be taken in account while designing the system otherwise the costs involved in migrating to the new solution could be huge. Let's consider for example options 2 and 3. They both consist in changing the message structure. If we have already consumers of the message we need to create a new version (i.e. _Order Created V2_) and keep publishing both V1 and V2 at the same time until all the consumers have finally migrated away from the original and faulty version. 
In my experience this can take a very long time, especially in big enterprises where teams have different schedules and deadlines. Also sometimes it is almost impossible to get rid of the legacy messages given that some of the consumers may never be updated. This results in an increase in infrastructure costs (as we must publish multiple version of the same message when we could actually publish just one), maintenance, and risk that the publisher will fail to publish all the versions of the message.

#### Transactional operations: how to coordinate different actors involved?

In the previous example, when an order is created we need to store it to the database before communicating to the rest of the platform what the order is. In the event of a failure of communication with the message broker, our system has 2 options:
- fail the process by rolling back the operation on the database (we would like to avoid it);
- guarantee the atomicity of the operation by increasing the reliability of our system.

The _Outbox Pattern_ (based on the [Guaranteed Delivery](https://www.enterpriseintegrationpatterns.com/patterns/messaging/GuaranteedMessaging.html) pattern) can help us to achieve the second point.
In the real (but still virtual) world, every time we send an email it normally goes to an _outbox_ before it is marked as _sent_ when the email is processed. The same approach can be applied to our EDA.
When saving the order information to our database, as part of the same transaction we simply have to store to an outbox table, the information we want to communicate to the rest of the platform. This data will them be polled by the _Outbox processor_ which will publish our _Order Created_ message. Every time the processor publishes a message, it will mark the related ourbox record as done in order not to reprocessed it next time.
This pattern will guarantee to us that the message will always be published at least once as multiple instances of the outbox processor could handle the same unprocessed record at the same time. Of course we could always implement a mechanism to reduce the risk of this event occurring.

## Conclusion

In this article I have presented why Event-Driven Architectures are important while designing distributed system, and what benefits they can bring. I also went through the list of possible issues and pitfalls that an EDA can bring to the system if not addressed immediately at design time. For each of these problems I gave a set of possible solutions based on literature and on my own experience with such systems. 
No solution is bullet proof, but we can always try to reduce the damage by preventing common issues that others before us faced and solved.
