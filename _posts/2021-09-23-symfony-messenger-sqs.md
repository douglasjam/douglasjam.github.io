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

Similar to the Event Dispatcher, Symfony has a component called Messenger. The main difference between both is that the last one allows us to dispatch and handle events asynchronously via the most diverse transporters like AMQP, Redis, Doctrine and others. It's a plug-and-play component, easy to install and configure but what makes the big differential is its extendability.

I had a scenario where I needed to push messages into SQS, symfony already provides integration to it but offered solution did not match all my needs and I had to implement a custom transporter. My needs were:

- #### [JMS Serializer](https://github.com/schmittjoh/serializer) as default serializer

    Our codebase use for many objects JMS serializer to push and receive DTOs, JMS does not implement Symfony serializer interface and can not handle many cases without annotations. Having to use 2 different serializer would increase the project complexity

- #### SQS environment from [localstack](https://github.com/localstack/localstack) were not easily configurable
  
    For local enviromment, we use localstack to simulate the AWS services, localstack has a different configuration than the dns option provided by the Symfony SQS Transporter, the way would be to inject our manually configured SQS client

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

The `get()` method basically the opposite of send. We get SQS messages and have to deserialize them back into an envelope. The catch is to find out what type is the envelope message. The good part of the envelope structure is that the stamps are an array where the key is its class name. With that we can easily find out `JmsType` stamp and decode the envelope message as well as other stamps to deserialize them properly.

One important thing to remember is that we need to store the SQS `ReceiptHandle` in our envelope, so we can ack the message at the end, for that a new stamp `SqsMetadataStamp` was created.

```php
 public function get(): iterable
    {
        $result = $this->sqsClient->receiveMessage([
            'QueueUrl' => $this->queueUrl
        ]);

        $messages = $result->get('Messages');

        if ($messages === null) {
            return [];
        }

        $envelopes = [];
        foreach ($messages as $message) {
            $body = $message['Body'];
            $bodyDecoded = json_decode($body, true);

            $stamps = [];
            foreach ($bodyDecoded['stamps'] ?? [] as $stampClass => $stampsCollection) {
                $deserializedStamps = $this->jmsSerializer->deserialize(
                    json_encode($stampsCollection),
                    "array<$stampClass>",
                    'json'
                );

                $stamps = array_merge($stamps, $deserializedStamps);
            }

            $jmsTypes = array_filter($stamps, fn($stamp) => get_class($stamp) === JmsTypeStamp::class);
            if (count($jmsTypes) !== 1) {
                throw new \Exception('Message does not have JmsTypeStamp');
            }

            /** @var JmsTypeStamp $jmsTypeStamp */
            $jmsTypeStamp = current($jmsTypes);

            $jobDto = $this->jmsSerializer->deserialize(
                json_encode($bodyDecoded['message']),
                $jmsTypeStamp->getType(),
                'json'
            );

            $envelope = (new Envelope($jobDto, $stamps))
                ->with(new SqsMetadataStamp($message['MessageId'], $message['ReceiptHandle']));

            $this->log(self::STARTED_MESSAGE, $envelope);

            $envelopes[] = $envelope;
        }

        return $envelopes;
    }
```

## ack()

After dispatching and processing the event in message handler the ack method of the transporter is called. Its job is to mark the message as completed, in our case for SQS we have to delete the message, a very straight forward step.

```php
public function ack(Envelope $envelope): void
    {
        /** @var SqsMetadataStamp $sqsMetadataStamp */
        $sqsMetadataStamp = $envelope->last(SqsMetadataStamp::class);

        $this->sqsClient->deleteMessage([
            'QueueUrl' => $this->queueUrl,
            'ReceiptHandle' => $sqsMetadataStamp->getReceiptHandle()
        ]);

        $this->log(self::SUCCEED_MESSAGE, $envelope);
    }
```

## reject()

The reject method is called when we could not handle the message after its retries. By default, Symfony tries to handle every message 3 times before executing the reject method. The reject method is usually meant to put the message in some other state, like pushing it into another queue, or updating some database status. For SQS queues we mostly rely on `DeadLetterQueues`, a spare queue in which failed messages end up. Since the SQS mechanism already have the retry mechanism in place and `reject` behaviour I decided to let this method only for logging purposes. Another important poins is that I configured the messenger to do not retry messages, otherwise it would multiply Symfony retries with SQS configured retries.

```php
public function reject(Envelope $envelope): void
    {
        $this->log(self::FAILED_MESSAGE, $envelope);
    }
```
