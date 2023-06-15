---
layout: post
title:  "Priority queue with public clouds, Dapr and Go"
date:   2023-06-14 13:28:29 -0700
categories: [posts]
tags: [AWS, GCP, Go, Golang, Queue, Priority, SQS, Pub/Sub, Architecture, Dapr]
author: Konstantin Vasilev
---

### **PART 1: THE CHALLENGE**

Message queues. A wonderful approach to connecting two or more services leveraging scaling and resiliency.

One of the applications serving client requests ("frontend") can offload heavy tasks to some more powerful machines for
further long processing. Those "Big Servers" also have capacity limitations, and considering the time-consuming nature of such
tasks, they all can not fit into the fleet of _"Big Servers"_ at once.

That's when we need some kind of call center for applications when someone would quickly respond with
_"Your request is very important to us. We will call you back with the results in a while. Keep in touch."_.
And in the background would feed tasks one by one across the _"Big Servers"_ pool.

![common queue diagram](/imgs_/priority_pubsub/queue1.png)

Here my story begins.

It's about one monolith application migration to a cloud environment when the customer
asked to "add some horizontal scalability".

Don't want to bore you with details. Just picture a simple API service accepting client requests to execute a very long task.
And those spoiled users of this service do not want to wait the whole period for the task to be processed, but just to click a button
and run off for another round of coffee with colleagues.
Well, this is understandable. Who wants to wait while the robot works up for about an hour? Yup, me neither.

Long story short, the monolith has been decoupled into two services and linked with a queue (tada!).
We all love microservices, right?

The customer then added one more ask: _"Would be great if we could deliver this to multiple clouds at once"_.

Here's where [Dapr](https://dapr.io) has joined the project and took its place as an abstraction layer between the
application and a cloud queue. This article is not about [Dapr](https://dapr.io), so I can only add here it's a great tool
helping to switch between different cloud components (such as AWS SQS or GCP Pub/Sub) with no code changes in the
application itself. Some more brief architectural descriptions of the Dapr you will find further down.

![common queue w_dapr_diagram](/imgs_/priority_pubsub/queue2.png)

The Dapr has very interesting
[Declarative subscription](https://docs.dapr.io/developing-applications/building-blocks/pubsub/subscription-methods/#declarative-subscriptions)
mode to work with Pub/Sub queues when you don't need to poll anything from the application side at all. You only need to
listen on an HTTP/gRPC endpoint and Dapr will deliver every message to that endpoint and even wait
while your application process it and then drop the message from the queue or return it for a retry if processing has failed.

This mode allowed us to minimize code changes in the application, especially considering its legacy nature (it's Perl, aha).

Since we always claim "we're agile!", the customer added another ask: _"Would be great to give some users capability to run their tasks with higher priority"_.

Easy-peasy! We can just enable Dapr to listen to multiple queues and deliver messages according to the queue priority.
What a nice tool....that does not support such a wonderful feature. :(

And none of the existing queueing systems in famous public clouds like AWS/GCP/Azure do not support message prioritization.
Some support exists in the [RabbitMQ](https://www.rabbitmq.com/priority.html) but this was not an option for the project needs
because not all cloud environments have Rabbit MQ out of the box and managed service was one of the requirements.

Well, at the end of the day, we got the following requirements list:
1. API and _"Big Servers"_ should be decoupled and scaled separately
2. Queueing solution should be "cloud agnostic" with support of easy "driver" (i.e. queue type) change
3. _"Big Servers"_ should consume tasks from multiple queues considering priority (tasks from "high" priority are VIP citizens and should be served before others)

### **PART 2: DIY OR DIE**

_Do It Yourself!_. It is always a great decision to reinvent the wheel once again.

Since Dapr was already integrated into all code pieces touching cooperation of the application and cloud resources,
there was no way to ditch it for good and rewrite the logic everywhere.

What if [Dapr Subscriber](https://docs.dapr.io/developing-applications/building-blocks/pubsub/howto-publish-subscribe/#subscribe-to-topics)
the component could be replaced with a custom one? The custom-made service could consume messages posted by
[Dapr Publisher](https://docs.dapr.io/developing-applications/building-blocks/pubsub/howto-publish-subscribe/#set-up-the-pubsub-component)
on the other end and then forward it to the application the same way as Dapr does, keeping the
[message format](https://docs.dapr.io/developing-applications/building-blocks/pubsub/pubsub-cloudevents/) Dapr uses?

And inside this custom application, a ~~sophisticated~~ very simple prioritization algorithm can be implemented to push
through messages arriving from a higher priority queue over the "normal" ones.

The queueing app structure could be the following:
1. Poll messages across all queues in the "stack" starting from the highest priority queue and moving toward the lowest as
   no messages left on the current "level"
2. Once a message is received, wrap it into Dapr message format and forward it to the application via HTTP, wait until the application
   process it and return a result
3. Delete the message if the response from the application is "success", otherwise - return the message to the queue for a re-try on another
   _"Big Server"_ node


![priority_queuing_diagram](/imgs_/priority_pubsub/priority_queue1.png)

Looks pretty simple, right? :)

The only thing to keep in mind is cloud agnosticism when implementing the subscriber,
i.e. define an abstraction layer where any specific implementation could be injected to support a particular cloud service.
<br>Most of the modern development platforms are supported by public cloud providers, so we can pick any SDK and implement
support for any of the queue services, i.e. create multiple "drivers" just like in Dapr and then switch between them in
the configuration.


### **PART 3: DESIGN FIRST**

Now it's time to code something :)

Where to start? The design of course!

The subscriber service should perform the following operations:
1. Poll several message queues for messages considering the priority of each queue
2. Forward each message to the application for processing and wait for the result
3. Drop the message from the queue in case of success and return it to the queue for retry if else

The first guy coming to the room is the `Queue` interface!
It is responsible for managing messages (receiving/deleting/returning).

```go
type Queue interface {
	QueueId()                  string
	ReceiveMessage()           (Message, error)
	DeleteMessage(m Message)   error
	ReturnMessage(m Message)   error
}
```

And `Message` in this case is another interface that will wrap every queue-specific message for easy processing.

For `Message` it's important to know its unique `Id`, which queue it belongs to (`QueueId`), and the actual `Data` to be
forwarded to the application for further processing. Each specific implementation will be different from queue to queue
since they all use different approaches to deliver messages.


```go
type Message interface {
	Id()       string
	QueueId()  string
	Data()     []byte
}
```

These two interfaces live in [`queue`](https://github.com/burmuley/priority-pubsub/tree/main/queue)
package and all implementations should also be law-abiding citizens of this package.

How a message received from the queue should be delivered to your application?
It's a task for `Processor`, one more handy interface!

```go
type Processor interface {
    Run(ctx context.Context, msg queue.Message, trans transform.TransformationFunc)
}
```

It's on the `Processor` to identify the data inside the message and decide how to parse it (or even not to) and where to forward it.
Btw, this guy is also settled in a separate
[`process`](https://github.com/burmuley/priority-pubsub/tree/main/process) package along with all other implementations.

Take a closer look at the weird parameter `trans` of type `transform.TransformationFunc`.
This is where we can apply a data transformation function which can help to adjust data from the queue before posting it
to the application endpoint. This helper is used further in the article to adapt to the Dapr data format.

Well, all the vital components are here!
Now it's time to glue all this stuff using a single _"director"_ that will orchestrate all the jobs for us.

Meet the `Poller` function living in the [`poll`](https://github.com/burmuley/priority-pubsub/tree/main/poll) package!
```go
type Poller func(ctx context.Context, wg *sync.WaitGroup, queues []queue.Queue, proc process.Processor, trans transform.TransformationFunc)
```

This function is designed to be running in multiple instances as [`goroutines`](https://go.dev/tour/concurrency/1) so
you can scale polling to required concurrency (i.e. throughput) by simply adjusting the number of concurrent `goroutines`.
Keep in mind, for proper scalability each `Poller` should consume only one message at once.

With this approach, the subscriber service can effectively control how many tasks the _”Big Server"_ can
handle simultaneously to not overwhelm pricey resources.

### **PART 4: IMPLEMENT OR QUIT**

What's next? Yup, you're right again - the subscriber service can not work with just interfaces.

This article will cover some parts of the implementation. Complete code of the `Priority Pub/Sub` you can find in my repository:
[`https://github.com/burmuley/priority-pubsub`](https://github.com/burmuley/priority-pubsub).

Here we only play with Dapr running on top of the AWS cloud.
The [`AWS SQS Queue`](https://github.com/Burmuley/priority-pubsub/blob/main/queue/aws_sqs.go) implementation is not
very important for this article. It's a simple set of AWS API calls with error handling. You can check the source code
in the repository at
[https://github.com/Burmuley/priority-pubsub/blob/main/queue/aws_sqs.go](https://github.com/Burmuley/priority-pubsub/blob/main/queue/aws_sqs.go)

The more important thing is how Dapr uses AWS services for messaging.

#### **Dapr Pub/Sub messaging architecture**

First of all, for message publishing Dapr targets an SNS topic (named the same way as the SQS queue). And it assumes that
the SQS queue is subscribed to receive messages from the SNS topic.

That way, when you publish a message via [Dapr API](https://docs.dapr.io/reference/api/pubsub_api/):
1. Dapr wraps the message data into [Cloud Events](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md) envelope (by default)
2. Dapr Publisher pushes the wrapped message into the SNS topic, and this wraps the message into another [SNS envelope](https://docs.aws.amazon.com/sns/latest/dg/sns-message-and-json-formats.html#http-notification-json)
3. AWS then forwards this message to the subscriber - SQS queue
4. The Dapr Subscriber then polls the message from SQS, pulls original message data from the SNS envelope, and passes the original data to the application HTTP endpoint

**Note**: I could not get why Dapr developers decided to involve SNS instead of simply pushing messages to SQS, maybe
it's the design constraints. If you know for sure the reason - enlighten me in the comments to this article. Thanks! :)

In other words, after the `Priority PubSub` service got the message published by Dapr from SQS, it needs to peel off the SNS
skin and pull out the original message before sending it to the application.

#### **HTTP message processor**

**Note**: Full code you can find in this file: [`process/http.go`](https://github.com/burmuley/priority-pubsub/blob/main/process/http.go)

The `Http` processor is represented with the following structure:

```go
type Http struct {
	config   HttpConfig
}
```

And `HttpConfig` structure contains everything we need for successful communication with the application:

```go
type HttpRawConfig struct {
	SubscriberUrl   string
	Method          string
	Timeout         int
	FatalCodes      []int
    ContentType     string `koanf:"content_type"`
}
```

Check out the `Processor` interface implementation for `Http`:

```go
func (r *Http) Run(ctx context.Context, msg queue.Message, trans transform.TransformationFunc) error {
	resChan := make(chan error)
    data := msg.Data()

   {
	   var err error
	   if trans != nil {
		   data, err = trans(data)
		   if err != nil {
			   return fmt.Errorf("%w: %w", ErrFatal, err)
		   }
	   }
   }

	go func() {
		client := http.Client{
			Timeout: time.Duration(r.config.Timeout) * time.Second,
		}

		req, err := http.NewRequestWithContext(ctx, r.config.Method, r.config.SubscriberUrl, bytes.NewBuffer(data))
		if err != nil {
			resChan <- fmt.Errorf("%w: %q", ErrFatal, err.Error())
			close(resChan)
			return
		}

		resp, err := client.Do(req)
		if err != nil {
			resChan <- fmt.Errorf("%w: %q", ErrFail, err.Error())
			close(resChan)
			return
		}
		
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			resChan <- fmt.Errorf("%w: task execution has failed", ErrFail)
			close(resChan)
			return
		}

		resChan <- nil
		close(resChan)
	}()

	for {
		select {
		case res := <-resChan:
			return res
		case <-ctx.Done():
			return ctx.Err()
		default:
			time.Sleep(5 * time.Second)
		}
	}
}
```

At the top of the function, the `resChan` channel is defined and then all the HTTP communications happen in another `goroutine`.
This is done to be able to monitor for external signals from the `context` that is passed to the `Run` function.
With the `context` it is possible to cancel the HTTP request and the `Run` function in one shot, for example in the event
of the service stop.

Same time, this `resChan` is used to receive an `error` after HTTP communication is done, which indicates the status
of the task (whether it was finished successfully or failed and we need to retry the task once again).

You might have noticed a weird field `FatalCodes` in the `HttpConfig` structure. This is the list of the HTTP response codes the application can use when the message should not be returned to the queue for another retry.
For example, this approach can be used to limit the number of retries for a particular message.

Another interesting point here is the `trans` parameter
(of type [`transform.TransformationFunc`](https://github.com/Burmuley/priority-pubsub/blob/main/transform/transform.go))
which is the main player here when we need to process Dapr messages wrapped into several envelopes.
As I mentioned before, we need to pull off the original message from SNS "wrapper".

And the `TransformationFunc` implementation [`transform.DaprAws`](https://github.com/Burmuley/priority-pubsub/blob/main/transform/dapr_aws.go)
does the perfect job here easy and simple way.

```go
func DaprAws(b []byte) ([]byte, error) {
	var snsEnvelope struct {
		Message string `json:"Message"`
	}

	err := json.Unmarshal(b, &snsEnvelope)
	if err != nil {
		return nil, fmt.Errorf("data transformation error: %w", err)
	}

	return []byte(snsEnvelope.Message), nil
}
```

The beauty of the `TransformationFunc` solution is that we can switch it on or off right in the Priority PubSub
service configuration file.

As you can see, the `Processor` interface gives you complete control and flexibility on how you want to process the message.
You can even send the message to a printer and then await a call on PBX on a special number defined in the message for the response.

Or translate it into Morse code and pass it to the Moon and wait for aliens to arrive (but I guess for this to work
there are no meaningful message processing timeouts in any of the Pub/Sub queues;) ).

#### **Simple polling algorithm**

Now it's time to talk about the `Poller` function, the glue layer for `Queue` and `Processor`.

Just to recap the function signature:
```go
type Poller func(ctx context.Context, wg *sync.WaitGroup, queues []queue.Queue, proc process.Processor, trans transform.TransformationFunc)
```

Again, for the sake of brevity, I'll not post here the full function code.
In the repository, you can find the full implementation of
[`Simple Poller`](https://github.com/burmuley/priority-pubsub/blob/main/simple.go#L45).

This `Poller` continuously checks for messages across all the `queues` in the list
(it's assumed they are sorted from high to low priority) and when it's received one simply passes it to the `proc` function
locking for a wait on the result. Once the result is here - the error analysis is performed and then `Poller` decides whether
the `Message` should be deleted from the `Queue` or sent back for another retry.

The function to poll messages considering `Queue` priority is incredibly simple:
```go
func receiveMessage(queues []queue.Queue) (queue.Message, error) {
	for _, q := range queues {
		message, err := q.ReceiveMessage()

		if err != nil {
			if errors.Is(err, queue.ErrNoMessages) {
				continue
			}
			return nil, err
		}

		return message, nil
	}

	return nil, queue.ErrNoMessages
}
```

It starts from the first `Queue` from the `queues` list and there is a message present - return it immediately, if not -
checking the next `Queue` in the list and so on.

Please note, that `ErrNoMessages` returned by the `receiveMessage` function is a special state indicating that there
were no messages in all `queues` defined in the list. The’ Poller’ uses this “flag” to jump to another iteration.

### **Summary**

That's all, folks! :)

You have just observed how the priority queue solution has been built in a few simple steps.

On the output, we got a scalable service that can be used to implement the Priority Queue pattern. The current implementation is
pretty weak in features and only covers the project needs. If I see any interest in this service I will continue its development
to the best of my ability and availability.
