---
title: "Symfony Messenger with custom SQS Transporter + JMS"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Symfony
  - AWS
  - SQS
---

Similar to the Event Dispatcher, Symfony has a component called Messenger. The main difference between both is that the last one allows us to dispatch and handle events asynchronously via the most diverse transporters like AMQP, Redis, Doctrine and others. It's already a plug-and-play component, easy to install and configure but what makes the big differential is its extendability.

I had a scenario where I needed to push messages into SQS, but default SQS Transporter offered by the bundle did not match all my needs. The reasons were:

- #### All my codebase use [JMS Serializer](https://github.com/schmittjoh/serializer)

    Our codebase use for many objects JMS, JMS does not implement symfony serializer interface and can not handle many cases without annotations. Having to use 2 different serializer would increase complexity

- #### SQS environment from [localstack](https://github.com/localstack/localstack) were not easily configurable
  
    For local enviromment we use localstack which has a different configuration than the dns option provided by the Symfony SQS Transporter, being able to use our already configured client would be the way to go

- ####  We wanted simple human-readable messages in the queues

    The default transporter pushes many stamps that are not needed to us, we just would like a couple that are meaningful to us

- #### We wanted to log whenever a message was created, handled, succeeded or failed

    A crucial part of our job is to see historically how a message behaved and drove its way until completion or failure, being able to log in specific parts in specific ways were a must 

In order to create a custom transporter we need to create a class that implement `TransportInterface`. This interface has 4 required methods:

```php
// used when dispatch a new message
public function send(Envelope $envelope): Envelope
// gets the messages from whatever storage you chose
public function get(): iterable;
// marks the message as succeed
public function ack(Envelope $envelope): void
// in case of failure, this method handles it
public function reject(Envelope $envelope): void
```

## send()

The first step we do when having the messenger is to dispatch messages, there is no magic here, but since I use JMS I did an extra mile by doing custom annotation for serialization special cases

```php
use Symfony\Component\Messenger\MessageBusInterface;
...
$job = new MyCustomJob();
$this->messageBus->dispatch($job);
``````

After that and the transporter configured, the `send()` method will be invoked, in here what I did is to create some new `stamps` that are JMS serializable:

- JmsTypeStamp: to identify which class I was serializing in the envelope message, so I could decode afterwards
- CreatedAt: to track when the message was created

```php
public function send(Envelope $envelope): Envelope
    {
        $message = $envelope->getMessage();
        $messageClass = get_class($message);

        $serializableEnvelope = (new Envelope($envelope->getMessage()))
            ->with(
                new JmsTypeStamp($messageClass),
                new CreatedAtStamp()
            );

        $json = $this->jmsSerializer->serialize($serializableEnvelope, 'json');

        $result = $this->sqsClient->sendMessage([
            'QueueUrl' => $this->queueUrl,
            'MessageBody' => $json
        ]);

        $this->log(self::CREATED_MESSAGE, $serializableEnvelope);

        return $serializableEnvelope;
    }
```

After this method being called, the following message would end-up in SQS:

```json
{
    "stamps": {
        "MyProject\\Stamp\\JmsTypeStamp": [
            { "type": "MyProject\\MyMessageObject" }
        ],
        "MyProject\\Stamp\\CreatedAtStamp": [
            { "createdAt": "2020-12-14 16:02:17" }
        ]
    },
    "message": {
        "name": "John Doe",
        "date": "2021-08-03",
        "address": {
            "street": "MyStreet",
            "number": 12
        }
    }
}
```

## get()

First thing I had to do is to get messages from the queue via the `get()` method. One of the reasons where I struggled using the default SQS transporter was that for tests we use localstack to emulate the AWS environment. Localstack has some peculiarities when configuring it like dns url, which didn't match the transporter criteria. By using a custom transporter I could inject whatever I wanted in my new Transporter. 


