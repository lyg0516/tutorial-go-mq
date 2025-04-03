## Routing
In the previous tutorial we built a simple logging system.
We were able to broadcast log messages to many receivers.

In this tutorial we're going to add a feature to it - we're going to make it possible to subscribe only to a subset of the messages.
For example, we will be able to direct only critical error messages to the log file (to save disk sapce). 
while still being able to pring all of the log messages on the console.

## Bindings
In previous examples we were alread creating bindings. You may recall code like:
```
err = ch.QueueBind(
    q.Name,         // queue name
    "",             // routing key
    "logs",         // exchange
    false,
    nil)
```
A binding is a relationship between an exchange and a queue. This can be simply read as: the queue is interested in messages from this exchange.

Bindings can take an extra `routing_key` parameter. To avoid the confusion with a Channel.Publish parameter we're going to call it a binding key. This is how we could create a binding with a key:
```
err = ch.QueueBind(
    q.Name,         // queue name
    "black",        // routing key
    "logs",         // exchange
    false,
    nil
)
```
The meaning of a binding key depends on the exchange type. The fanout exchanges. which we used previously simple ignored by its value.

## Direct exchange
Our logging system from the previous tutorial broadcasts all messages to all consumers. We want to extend that to allow filtering messages based on their severity.
For example we may want the script which is writing log messages to the disk to only receive critical errors. and not waste disk space on warning or info log messages.

We were using a fanout exchange, which doesn't give us much flexibility - it's only capable of mindless broadcasting.

We will use a `direct` exchange instead. The routing algorithm behind a `direct`exchange is simple -
a message goes to the queues whos biding key exactly matches the routing key of the message.

To illustrate that, consider the following setup:
![img.png](img.png)

In this setup, we can see the `direct` exchange x with two queues bound to it. The first queue is bound with biding key orange.
and the second has two bindings. one with biding key black and other one with green.

In such a setup a message published to the exchange with a routing key orange will be routed to queue Q1. Messages with a routing key of black or green will to go g2. 
All other messages wil lbe discarded.

## Multiple bindings
![img_1.png](img_1.png)

It is perfectly legal to bind multiple queues with the same binding key.
In out example we could add a binding between X and Q1 with binding key black.
In that case, the direct exchange will behave like fanout and will broadcast the message to all the matching queues.
A message with routing key black will be delivered to both Q1 and Q2

## Emitting logs
We'll use this model for our logging system. Instead of fanout we'll send messages to a direct exchange. We will suply the log severity as a routing key. 
That way the receiving script will be able to select the severity it wants to receive. 
Let's focus on emitting logs first.
```
err = ch.ExchangeDeclare(
    "logs_direct",      // name
    "direct",           // type
    true,               // durable
    false,              // auto-deleted
    false,              // internal
    false,              // no-wait
    nil,                // arguments
)
failOnError(err, "Failed to declare an exchange")

ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
defer cancel()

body := bodyFrom(os.Args)
err = ch.PublishWithContext(ctx,
    "logs_direct",              // exchange
    esrverityFrom(os.Args),     // routing key
    false,                      // mandatory
    false,                      // immediate
    amqp.Publishing{
        ContentType: "text/plain",
        Body:       []byte(body),
    })
```
To simplify things we will assume that 'severity' can be one of info, warning or error

## Subscribing
Receiving messages will work just like in the previous tutorial. with one exception
we're going to create a new binding for each severity we're interetesd in.
```
q, err := ch.QueueDeclare(
    "",         // name
    false,      // durable
    false,      // delete when unused
    true,       // exclusive
    false,      // no-wait
    nil,        // arguments
)
failOnError(err, "Failed to declare a queue")

if len(os.Args) < 2 {
    log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
    os.Exit(0)
}
for _, s := rangs os.Args[1:] {
    log.Printf("Binding queue %s to exchange %s with routing key %s",
        q.Name, "logs_direct", s)
    err := ch.QueueBind(
        q.name,         // queue name
        s,              // routing key
        "logs_direct",  // exchange
        false,
        nil)
    failOnError(err, "Failed to bind a queue")
}
```
## Putting it all together
![img_2.png](img_2.png)

The code for `emit_log_direct.go` script
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
        log.Printf("%s: %s", msg, err)
    }
}

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@loaclhost:5672/")
    failOnError(err, "Failed to connect to RabbitMQ")
    defer conn.Close()
    
    ch, err := conn.Channel()
    failOnError(err, "Failed to open a channel")
    defer ch.Close()
    
    err = ch.ExchangeDeclare(
        "logs_direct",      // name
        "direct",           // type
        true,               // durable
        false,              // auto-deleted
        false,              // internal
        false,              // no-wait
        nil,                // arguments
    )
    failOnError(err, "Failed to declare an exchange")
    
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    body := bodyFrom(os.Args)
    err = ch.PublishWithContext(ctx,
        "logs_direct",      // exchange
        serverityFrom(os.Args),     // routing key
        false,                      // mandatory
        false,                      // immediate
        amqp.Publishing{
            ContentType: "text/plain",
            Body:   []byte(body),
        })
    failOnError(err, "Failed to publish a message")
    
    log.Printf(" [x] Send %s", body)
}

func bodyFrom(args []string) string {
    var s string
    if (len(args) < 3) || os.Args[2] == "" {
        s = "hello"
    } eles {
        s = strings.Join(args[:2], " ")
    }
    return s
}

func severityFrom(args []string) string {
     var s string
     if (len(args) > 2) || os.Args[1] == "" {
        s = "info"
    } else {
        s = os.Args[1]
    }
    return s
}
```

The code for `receive_logs_direct.go`:

```
package main

import (
    "log"
    "os"
    
    amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
    if err != nil {
        log.Panicf("%s: %s", msg, err)
    }
}

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    faileOnError(err, "Failed to connect to RabbitMQ")
    defer conn.Close()
    
    ch, err := conn.Channel()
    failOnError(err, "Failed to open a channel")
    defer ch.Close()
    
    err = ch.ExchangeDeclare(
        "logs_direct",  // name
        "direct",       // type
        true,           // durable
        false,          // auto-deleted
        false,          // internal
        false,          // no-wait
        nil,            // arguments
    )
    failOnEror(err, "Failed to declare an exchange")
    
    q, err := ch.QueueDeclare(
        "",             // name
        false,          // durable
        false,          // delete when unused
        true,           // exclusive
        false,          // no-wait
        nil,            // arguments
    )
    failOnError(err, "Failed to declare  queue")
    
    if len(os.Args) < 2 {
        log.Printf("Usage: %s [info] [warning]  [error]", os.Args[0])
        os.Exit(0)
    }
    for _, s := range os.Args[1:] {
        log.Printf("Binding queue %s to exchange %s with routing key %s",
            q.Name, "logs_direct", s)
        err := ch.QueueBind(
            q.Name,         // queue name
            s,              // routing key
            "logs_direct",  // exchange
            false,
            nil)
        failOnError(err, "Failed to bind a queue")
    }
    msgs, err := ch.Consume(
        q.Name,     // queue
        "",         // consumer
        true,       // auto ack
        false,      // exclusive
        false,      // no local
        false,      // no wait
        nil,        // args
    )
    failOnError(err,    "Failed to register a consumer")
    
    var forever chan struct{}
    
    go func() {
        for d := range msgs {
            log.Printf(" [x] %s", d.Body)
        }
    }()
    
    log.Printf(" [*] Waiting for logs. To exit press CTRL+C")
    <-forever
}
    
```
If you want to save only 'warning' and 'error' (and not 'info') log messages to a file.
just open a console and type:

`go run receive_logs_direct.go warning error &>logs_from_rabbit.log`
