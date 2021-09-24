---
title: "Symfony Messenger with custom SQS Transporter + JMS"
excerpt_separator: "<!--more-->"
categories:
  - Software Development
tags:
  - Symfony
  - AWS
  - SQS
---

Similar to the Event Dispatcher, Symfony has a component called Messenger. The main difference between both is that the last one allows us to dispatch and handle events asynchronously via the most diverse transporters like AMQP, Redis, Doctrine and others. It's a plug-and-play component, easy to install and configure but what makes it great is its extendable design.

I had a scenario where I needed to push messages into SQS, Symfony Messenger already provides an integration to it but the offered solution did not match all my needs, and I ended up implementing a custom transporter. My needs were:

- #### Use [JMS Serializer](https://github.com/schmittjoh/serializer) as serializer

    Our codebase uses the JMS serializer lib to convert objects to and from JSON via our endpoints, JMS does not implement the Symfony serializer interface and can not simply handle any type of object. Having to mix different serializer would increase the project complexity and was a no-go for us

- #### SQS environment from [localstack](https://github.com/localstack/localstack) was not easily configurable
  
    For our local development environment we use `localstack` to simulate the AWS services, localstack has a different configuration than the dns option provided by the Symfony SQS Transporter that just didn't fit. The way to go would be to inject our manually configured SQS client

- ####  We wanted simple human-readable messages in the queues

    The default transporter pushes many stamps that are not needed to us, we just would like a couple that are meaningful to us like createdDate and what DTO object was in the message

- #### We wanted to log whenever a message was created, handled, succeeded or failed

    A crucial part of our job is to see historically how a message behaved and drove its way until completion or failure, being able to log in specific parts in specific ways were a must. Symfony messenger already logs all its activities but not in the way we wanted to

## How to create a custom transporter

The first thing you need to do is to create a class that implements `TransportInterface`. This interface has 4 required methods:

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

All starts by dispatching messages that end-up in our queue, there is no magic here but since we use JMS an extra effort was needed. We needed to create a custom JMS serializable stamps to attach to our envelope, they were:

- JmsTypeStamp: to identify which class I was serializing in the envelope message, so I could decode afterwards
- CreatedAt: to track when the message was created

After serializing the envelope we just had to push it to SQS.

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

## get()

The `get()` method basically the opposite of send. We get SQS messages and have to deserialize them back into an envelope. The catch is to find out what type is the envelope message. The good part of the envelope structure is that the stamps are an array where the key is its class name. With that we can easily find out `JmsType` stamp and decode the envelope message as well as other stamps to deserialize them properly.

One important thing to remember is that we need to add the SQS `ReceiptHandle` into our envelope, so we can ack the message at the end, for that a new stamp `SqsMetadataStamp` was created.

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

The reject method is called when we could not handle the message after its retries. By default, Symfony tries to handle every message 3 times before executing the reject method. The reject method is usually meant to put the message in some other state, like pushing it into another queue, or updating some database status. For SQS queues we mostly rely on `DeadLetterQueues`. Since the SQS mechanism already have the retry and DLQ mechanism in place we decided to let this method only for logging purposes. Another important point is that I configured the messenger to do not retry messages, otherwise it would retry every message 3 times for every SQS retry, totalizing 9 times instead of 3.

```php
public function reject(Envelope $envelope): void
    {
        $this->log(self::FAILED_MESSAGE, $envelope);
    }
```

## Enabling the Transporter

Before enabling and using your custom transporter you need to teach Symfony how to construct it. To do this, create a factory class implementing `TransportFactoryInterface` that returns your transporter. The method `supports` is used by Symfony to identify which transporter will be used based on the bundle configuration in `messenger.yaml`. Pay attention to use an exclusive pattern for it in order to do not collide with others like `sqs://`. You can also use the dns string to inject parameters and extract them from it, like `my-transport://queue-name/account` but I rather do it via the options parameter.

```php
class SqsTransportFactory implements TransportFactoryInterface
{
    use LoggerAwareTrait;

    public function __construct(private SqsClient $sqsClient, private JmsSerializer $jmsSerializer)
    {
    }

    public function createTransport(string $dsn, array $options, SerializerInterface $serializer): TransportInterface
    {
        return new SqsTransport($this->sqsClient, $this->jmsSerializer, $this->logger, $options['queue_name']);
    }

    public function supports(string $dsn, array $options): bool
    {
        return str_starts_with($dsn, 'transporter-sqs://');
    }
}
```

Tag the transporter factory in your container configuration file `services.yaml`:

```yaml
    MyNamespace\MyCustomTransporter:
        tags: [messenger.transport_factory]
```

The last step is to activate your transporter in the bundle configuration, one important thing to look is that I configured `retry_strategy.max_retries` to 0 in order to prevent symfony to retry and to rely on SQS retry mechanism.
The routing part is optional when you have only one transporter activated but if you have more you may want to tell in which to use explicitly in case you do not want to route it to 2 or more transporters.

```yaml
framework:
    messenger:
        transports:
            jobs-sqs:
                dsn: 'transporter-sqs://'
                retry_strategy:
                    max_retries: 0
                options:
                    queue_name: '%env(SQS_QUEUE_NAME)%'

        routing:
            'MyNamespace\MyCustomJob': transporter-sqs
```

# Running

First thing that needs to happen is to dispatch messages that will end up in the queue, usually you want to do this via your controller or other job, you simply need to create a message and dispatch it via the `MessageBus` as below:

```php
use Symfony\Component\Messenger\MessageBusInterface;
...
$job = new MyCustomJob();
$this->messageBus->dispatch($job);
``````
A message like below should end-up in the SQS queue:

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

To start consuming messages you should another proccess, ideally via CLI that executes the command below. This command has many parameters and options like limit how long it should execute, how many messages to pull, max memory and others. Make sure you read symfony documentation to get to know all of them.

```./bin/console messenger:consume```

What did you think about our approach to adjust the messenger to our needs, would you do something different?