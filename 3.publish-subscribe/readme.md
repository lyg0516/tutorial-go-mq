## Publish/Subscribe
In the previous tutorial we created a work queue. The assumption behind a work queue is delivered to exactly one worker. In this part we'll do something completely different. we'll deliver a message to multiple consumers. This patterns is known as 'publish/subscribe'.

To illustrate the pattern, we're going to build a simple logging system. It will consist of two programs -- the first will emit log messages and the second will receive and print them.

In our logging system every running copy of the receiver program will get the messages. That way we'll be able to run one receiver and direct the logs to disk. and at the same time we'll be able to run another receiver and see the logs on the screen.

Essentially, published log messages are going to be broadcast to all receivers.

## Exchange
In previous part of the tutorial we sent and received messsages to and from a queue. Actually, quite often the producer doesn't even know if a message will be delivered to any queue at all.  
Instead, the producer can only send messages to an exchange. 
An exchange is a very simple thing. On one side it receives messages from producers and the other side it
pushes them to queues. The exchange must know exactly what to do with a message it receives.
Should it be appended to a particular queue? Should it be appended to many queue? Or should it get discarded. 
The rules for that are defined by the exchange type.
![img.png](img.png)  
There are a few exchange types available. `direct`, `topic`, `headers` and `fanout`. We'll focus on the last one -- 
the fanout. Let's create an exchange of type this. and call it logs
```
err := ch.ExchangeDeclare(
    "logs",         // name
    "fanout",       // type
    true,           // durable
    false,          // auto-deleted
    false,          // internal
    false,          // no-wait
    nil,            // arguments
)
```
The fanout exchange is very simple. As you can probably guess from the name.
It just broadcasts all the messages it received to all the queues it knows.
And that's exactly what we need for our logger.
> ### The default exchange
> In previous parts of the tutorial we knew about exchanges.
> but still were able to send messages to queues. 
> That was possible because we were using a default exchange.
> which is identified by the empty string("").
> ```
> err = ch.PublishWithContext(ctx,
>   "",         // exchange
>   q.Name,     // routing key
>   false,      // mandatory
>   false,      // immediate
>   amqp.Publishing {
>       ContentType: "text/plain",
>       Body: []byte(body),
>   })
> ```
> Here we use the default or nameless exchange. messages are routed to the queue with the name specified by routing_key parameter if exists.

Now, we can publish to our named exchange instead.
```
err = ch.ExchangeDeclare(
    "logs",         // name
    "fanout",       // type
    true,           // durable
    false,          // auto-deleted
    false,          // internal
    false,          // no-wait
    nil,            // arguments
)
failOnError(err, "Failed to declare an exchange")

ctx, cancel := context.WithTimeout(context.Backgroud(), 5 * time.Second)
defer cancel()

body := bodyFrom(os.Args)
err = ch.PublishWithContext( ctx,
    "logs",     // exchange
    "",         // routing key
    false,      // mandatory
    fakse,      // immediate
    amqp.Publishing{
        contentType: "text/plain",
        Body: []byte(body),
    })
```
## Temporary queues
As you may remember previously we were using queues that specific names (remember hello and task_queue?). 
Being able to name a queue was crucial for us -- we needed to point the workers to the same queue.
Giving a queue a name is important when you want to share the queue between producers and consumers.

But that's not the case of our logger. We want to hear about all log messages. 
not just a subset of them.
We're also interested only in currently flowing messages not in the old ones. To solve that we need two things.

Firstly, whenever we connect to Rabbit we need a fresh, empty queue. 
To do this we could create a queue with a random name, or even better - let the server choose a random queue name for us.

Secondly, once we disconnect the consumer the queue should be automatically delete.
In the amqp client, when we supply queue name as an empty string. 
we create a non-durable queue with a generated name:
```
q, err := ch.QueueDeclare (
    "",     // name
    false,  // durable
    false,  // delete when unused
    true,   // exclusive 다른 클라이언트나 채널이 접근 불가, 연결이 끊기면 큐 삭제
    false,  // no-wait
    nil,    // arguments
)
```
When the method returns, the queue instance contains a random queue name generated by RabbitMQ. For example it may look like amq.get-JzTY20BRgKO-HjmUJ0wLg.
When the connection that declared it closes, the queue will be deleted because it is declared as exclusive.

## Bindings
![img_1.png](img_1.png)

We've already created a fanout exchange and a queue. 
Now we need to tell the exchange to send messages to our queue.
That relationship between exchange and queue is called a binding.
```
err = ch.QueueBind(
    q.Name,         // queue name
    "",             // routing key
    "logs",         // exchange
    false,          
    nil
)

```
From now on the logs exchange will append messages to our queue.

## Putting it all together
![img_2.png](img_2.png)
The producer program. which emits log messages, doesn't look much different from previous tutorial.
The most important change is that we now want to publish messages to our `logs` exchange instead of nameless one. 
We need to supply a `routingKey` when sending, but its value is ignored for fanout exchanges. Here goes the code for emit_log.go script.
```
package main

import (
    "context"
    "log"
    "os"
    "strings"
    "time"
    
    amqp "github.com/rabbitmq/amqp091-go"
)

funr failOnError(err error, msg string) {
    if err != nil {
        log.Panicf("%s: %s', msg, err)
    }
}

func main() P
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    failOnError(err, "Failed to connect to RabbitMQ")
    defer conn.Close()
    
    ch, err := conn.Cnhanel()
    failOnError(err, "Failed to open a channel")
    defer ch.Close()
    
    err = ch.ExchangeDeclare(
        "logs",     // name
        "fanout",   // type
        true,       // durable
        false,      // auto-deleted
        false,      // internal
        false,      // no-wait
        nil,        // arguments
    )
    failOnError(err, "Failed to declare an exchange")
    
    ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
    defer close()
    
    body := bodyFrom(os.Args)
    err = ch.PublishWithContext(ctx,
        "logs",     // exchange
        "",         // routing key
        false,      // mandatory
        false,      // immediate
        amqp.Publishing(
               ContentType: "text/plain",
               Body: []byte(body),
           })
    failOnError(err, "Failed to publish a message")
    
    log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
    var s string
    if (len(args) < 2) || os.Args[1] == "" {
        s = "hello"
    } else {
        s = strings.Join(args[1:], " ")
    }
    return s
}
```
As you see, after establishing the connection we declared the exchange.
This step is necessary as publishing to a non-existing exchange is forbidden.

The messages will be lost if no queue is bound to the exchange yet. but that's okay for us.
if no consumer is listening yet we can safely discard the message.

The code for `receive_logs.go`:
```
package main

import (
    "log"
    
    amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
    if err != nil {
        log.Panicf("%s: %s", msg, err)
    }
}

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    failOnError(err, "Failed to connect to RabbitMQ")
    defer conn.Close()
    
    ch, err := conn.Channel()
    failOnError(err, "Failed to open a channel")
    defer ch.Close()
    
    err = ch.ExchangeDeclare(
        "logs",     // name
        "fanout",   // type
        true,       // durable
        false,      // auto-deleted
        false,      // internal
        false,      // no-wait
        nil,        // arguments
    )
    failOnError(err, "Failed to declare an exchange")
    
    q, err := ch.QueueDeclare(
        "",         // name
        false,      // durable
        false,      // delete when unused
        true,       // exclusive
        false,      // no-wait
        nil,        // arguments
    )
    failOnError(err, "Failed to declare a queue")
    
    err = ch.QueueBind(
        q.Name,     // name
        "",         // routing key
        "logs",     // exchange
        false,      
        nil,
    )
    failOnError(err, "Failed to bind a queue")
    
    msgs, err := ch.Consume(
        q.Name      // queue
        "",         // consumer
        true,       // auto-ack
        false,      // exclusive
        false,      // no-local
        false,      // no-wait
        nil,        // args
    )
    failOnError(err, "Failed to register a consumer")
    
    var forever chan struct{}
    
    go func() {
        for d:= range msgs {
            log.Printf(" [x] %s", d.Body)
        }
    }()
    
    log.Printf(" [*] Waiting for logs. To exit precc CTRL+C")
    <-forever
}
```
If you want to save logs to file. just open a console and type:
```
go run receive_logs.go &> logs_from_rabbit.log
```



