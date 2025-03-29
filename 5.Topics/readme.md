## Topics
In the previous tutorial we improved our logging system. 
Instead of using a fanout exchange only capable of dummy broadcasting.
we used a direct one. and gained a possibility of selectively receiving the logs.

Although using `direct`exchange improved our system. 
it still has limitations - it can't do routing based on multiple criteria.

In our logging system we might want to subscribe to not only logs based on severity.
bu also based on the source which emitted the log. 
You might know this concept from the `syslog` unix tool. 
which routes logs based on boty severity (info/warn/crit...) and facility(auth/cron/kern,...).

That would give us a lot of flexibility - we may want to listen to just critical eorrs coming from 'cron' but also all logs from 'kern'.

To implment that in our logging system we need to learn about a more complex `topic` exchange.

## Topic exchange
Messages sent to a `topic` exchange can't have an arbitrary `routing-key` - it must be a list of words.
delimited by dots. The words can be anything. but usually they specify some features connected to the message.
A few valid routing key examples: `stock.usd.nyse`, `nyse.vmw`, `quick.orange.rabbit`.
There can be as many words in the routing key as you like, up to the limit of 255 bytes.

The binding key must also be in the same form. The logic behind the `topic` exchange is similar to a `direct`one - 
a message sent with a particular routing key will be delivered to all the queues that are bound with a matching binding key. 
However there are two important special cases for biding keys:
- \*(star) can substitute for exactly one word.
- \#(hash) can substitute for zero or more words.


It's easiest to explain this in an example:

![img.png](img.png)

In this example, we're going to send messages which all describe animals.
The messages will be sent with a routing key that consist of three words (two dots). 
The first word in the routing key will describe speed.
second a colour and third a species: \<speed\>.\<colour\>.\<species\>

We created three bindings: Q1 is bound with binding key *.orange.* and Q2 with *.*.rabbit and
lazy.#.

These bindings can be summarised as:
- Q1 is interested in all the orange animals.
- Q2 wants to hear everything about rabbits, and everything about lazy animals.

A message with a routing key set to quick.orange.rabbit will be delivered to both queues.
Message `lazy.orange.elephant` also will go to both of them. On the other hand `quick.orange.fox` will only go to the first queue,
and `lazy.bronw.fox` only to the second. 
`lazy.pink.rabbit` will be delivered to the second queue only once, even though it matches two bindings.
`quick.brown.fox` doesn't match any binding so it will be discarded

What happens if we break our contract and send a message with one or four words. like `quick.orange.new.rabbit`? Well, these message won't match any bindings and will be lost.

On the other hand `lazy.orange.new.rabbit`, even though it has four words, it will match the last binding and will be delivered to the second queue.

> ### Topic exchange
> Topic exchange is powerful and can behave like other exchanges.
> 
> When a queue is bound with #(hash) binding key - it will receive all the messages, regardless of the routing key - like in fanout exchange
> 
> When special characters * (star) and # (hash) aren't used in bindings. the topic exchange will behave just like a `direct` one

## Putting it all together
We're going to use a `topic` exchange in our logging system. We'll start off with a working assumption that the routing keys of logs will have two words: \<facility>:\<severity>

The code is almost the same in the previous tutorial.

emit_log_topic_go:
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

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        body := bodyFrom(os.Args)
        err = ch.PublishWithContext(ctx,
                "logs_topic",          // exchange
                severityFrom(os.Args), // routing key
                false, // mandatory
                false, // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 3) || os.Args[2] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[2:], " ")
        }
        return s
}

func severityFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "anonymous.info"
        } else {
                s = os.Args[1]
        }
        return s
}
```

The code for `receive_logs_topic.go`
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
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when unused
                true,  // exclusive
                false, // no-wait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        if len(os.Args) < 2 {
                log.Printf("Usage: %s [binding_key]...", os.Args[0])
                os.Exit(0)
        }
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_topic", s)
                err = ch.QueueBind(
                        q.Name,       // queue name
                        s,            // routing key
                        "logs_topic", // exchange
                        false,
                        nil)
                failOnError(err, "Failed to bind a queue")
        }

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto ack
                false,  // exclusive
                false,  // no local
                false,  // no wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

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