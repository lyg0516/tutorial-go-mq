## Round-robin dispatching
One of the advantages of using a Task Queue is the ability to easily parallelise work. If we are building up a backing of work.
we can just add more workers and that way, scale easily.

## Message acknowlegment
Doing a task can take a few seconds. you may wonder what happens if a consumer start a long task and it terminates before it completes. With out current code, once RabbitMQ delivers a message to the consumer.  
It immediately marks it for deletion. In this case,if you terminate a worker, the message it was just processing is lost.  
The messages that were dispatched to this particular worker were not yet handled are also lost.  

But we don't want to lose any tasks. If a worker dies, we'd like the task to be delivered to another worker.

In order to make sure a message is never lost. RabbitMQ supports message acknowledgments.  
An acknowledgement is sent back by the consumer to tell RabbiMQ that particular message has been received, processed and that RabbitMQ is free to delete it.

If a consumer dies (its channel is closed, connection is closed, or TCP connection is lost) without sending an ack.  
RabbitMQ will understand that a message wasn't processed fully and will re-queue it.  
If there are other consumers online at the same time. It will then quickly redeliver it to another consumer.  
That way you can be sure that no message is lost. Even if the workers occasionally die.

A timeout (30 minutes of default) is enforced on consumer delivery acknowledgement. This is helps detect buggy (stuck) consumers that never acknowledge deliveries.  
You can increase this timeout.

In this tutorial we will use manual message acknowledgements by passing a false for the 'auto-ack' argument and them send a proper acknowledgment from the worker with d.Ack(false) (this acknowledges a single delivery). once we've done with a task.
```
msgs, err := ch.Consume(
    q.Name, // queue
    "", // consumer
    false, // auto-ack
    false, // exclusive
    false, // no-local
    false, // no-wait
    nil, // args
)
failOnError(err, "Failed to register a consumer")

var forever chan struct{}

go func() {
    for d := range msgs {
        log.Printf("Received a message: %s", d.Body)
        dotCount := bytes.Count(d.Body, []byte("."))
        t := time.Duration(dotCount)
        time.Sleep( t * time.Second)
        log.Printf("Done")
        d.Ack(false)
    }
}()

log.Printf(" [*] waiting for messages. To exit press CTRL+C")
<-forever
```
Using this code, you can ensure that even if you terminate a worker using CTRL+C while it was processing a message, nothing is lost.  
Soon after the worker terminates, all unacknowledged messages are redelivered.

Acknowledgement must be sent on the same channel that received the delivery.  
Attempts to acknowledge using a different channel will result in a channel-level protocol exception. 

> ### Forgotten acknowledgment
> 
> It's a common mistake to miss the ack. It's an easy error. but the consequences are serious.
> Messages will be redelivered when your client quits (which may look like random redelivery).
> but RabbitMQ will eat more and more memory as it won't be able to release any unacked messgaes.
> 
>  In order to debug this kind of mistake you can use rabbitmqctl to print the messages_unacknowledged filed.
> 
> ```sudo rabbitctl list_queues name messages_ready message_unacknowledged```

## Message durability
We have learned how to make sure that even if the consumer dies, the task isn't lost.
But our tasks will still be lost if RabbitMQ server stops.

When RabbitMQ quits or crashes it will forget the queues and messages unless you tell it not to.  
Two thins are required to make sure that messages aren't lost: we need to mark both the queue and messages as durable.  

First, we need to make sure that the queue will survive a RabbitMQ node restart. In order to do so, we need to declare it as durable
```
q, err := ch.QueueDeclare(
    "hello",    // name
    true,       // durable
    false,      // delete when unused
    false,      // exclusive
    false,      // no-wait
    nil,        // arguments
)
```
Although this command is correct by itself. It won't work in our present setup.  
That's because we've already defined a queue called hello which is not durable. RabbitMQ dosen't allow you to redefine an existing queue with different parameters and will return an error to any program that tries to do that.  

This `durable`option change needs to be applied to both the producer and consumer code.

At this point we're sure that the `hello` queue won't be lost even if RabbitMQ restarts. Now we need to mark our messages as persistent.
- by using `amqp.Persistent`option `amqp.Publishing` takes.
```
err := ch.PublishWithContext(ctx,
    "",         // exchange
    q.Name,     // routing key
    false,      // mandatory
    false,      
    amqp.Publishing {
        DeliveryMode: amqp.Persistent,
        ContentType: "text/plain",
        Body: []byte(body)
    })
```
> ### Note on message persistence
> Marking messages as persistent doesn't fully guarantee that a message won't be lost. Although it tells RebbitMQ to save the message to disk. There is still a short time window when RabbitMQ has accepted a message and hasn't saved it yet. Also, RabbitMQ doesn't do fsync(2) for every message -- it may be just saved to cache and not really written to the disk. The persistence guarantees aren't strong. But it's more than enough for our simple task queue. If you need a stronger guarantee then you can use publisher confirms.

## Fair dispatch
You might have noticed that the dispatching still doesn't work exactly as we want. For example in a situation with two workers.  
When it all odd messages are heavy and even messages are light. 
One worker will be constantly busy and the other one will do hardly any work.  
Well, RabbitMQ doesn't know anything about that and will still dispatch messages evenly.

This happens because RabbitMQ just dispatches a message when the message enters the queue. It doesn't look at the number of acknowledged messages for a consumer. It just blindly dispatches even n-th message to the n-th consumer.

In order to defeat that we can set the prefetch count with the value of 1. This tells RabbitMQ not to give more than one message to a worker at time.
Or, in other words, don't dispatch a new message to a worker until it has processed and acknowledged the previous one. Instead, it wil dispatch it to the next worker that is not still busy.
```
err = ch.Qos(
    1, // prefetch count
    0, // prefetch size
    false, // global
)
failOnError(err, "Failed to set Qos") 
```
## Putting it all together
Final code of our `new_task.go` class
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

func failOnError(err error, msg string) {
    if err != nil {
        log.Panicf("%s: %s", msg, err)
    }
}

funcm main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    failOnError(err, "Failed to connect to RabbitMQ")
    defer conn.Close()
    
    ch, err := conn.Channel()
    failOnError(err, "Failed to open a channel")
    defer ch.Close()
    
    q, err := ch.QueueDeclare(
        "task_queue",   // name
        true,           // durable
        false,          // delete when unused
        false,          // exclusive
        false,          // no-wait
        nil,            // arguments
    )
    failOnError(err, "Failed to declare a queue")
    
    ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
    defer cancel()
    
    body := bodyFrom(os.Args)
    err = ch.PublishWithContext(ctx,        
        "",         // exchange
        q.Name,     // routing key
        false,      // mandatory
        false,  
        amqp.Publishing{
            DeliveryMode: amqp.Persistent,
            ContentType: "text/plain",
            Body:   []byte(body),
        })
    failOnError(err, "Failed to publish a message")
    log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string) {
    var s string
    if (len(args) < 2) || os.Args[1] == "" {
        s = "hello"
    } else {
        s = strings.Join(args[1:], " ")
    }
    return s
} 
```
And our `worker.go`:
```
package main

import (
    "bytes"
    "log"
    "time"
    
    amqp "github.com/rabbitmq/amqp0910go"
)

func failOnError(err error, msg string) {
    if err != nil {
        log.Panicf("%s:%s", msg, err)
    }
}

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    failOnError(err, "Failed to connect to RabbitMQ")
    defer conn.Close()
    
    ch, err := conn.Channel()
    failOnError(err, "Failed to open a channel")
    defer ch.Close()
    
    q, err := ch.QueueDeclare(
        "task_queue",   // name
        true,           // durable
        false,          // exclusive
        false,          // no-wait
        nil,            // arguments
    )
    failOnError(err, "Failed to declare a queue")
    
    err = ch.Qos(
        1,  // prefetch count
        0,  // prefetch size
        false,  //global)
    )
    
    failOnErr(err, "Failed to set Qos")
    
    msgs,err := ch.Consume(
        q.Name,         // queue
        "",             // consumer
        false,          // auto-ack
        false,          // exclusive
        false,          // no-local
        false,          // no-wait
        nil,            // args
    )
    failOnError(err, "Failed to register a consumer")
    
    var forever chan structP{
    
    go func() {
        for d := range msgs {
            log.Printf("Received a message: %s", d.Body)
            dotCount := bytes.Count(d.Body, []byte("."))
            t := time.DurationO(dotCount)
            time.Sleep(t * time.Second)
            log.Printf("Done")
            d.Ack(false)
        }
    }()
    
    log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
    <- forever
}
```