## Remote procedure call (RPC)
In the second tutorial we learned how to use Work Queues to distribute time-consuming tasks among multiple workers.

But what if we need to run a function on a remote computer and wait for the result? Well, that's a different story.
This pattern is commonly known as Remote Procedure Call or RPC.

In this tutorial, we're going to use RabbitMQ to build an RPC system: a client and a scalable RPC server.
As we don't have any time-consuming tasks that are worth distributing. we're going to create a dummy RPC service that returns Fibonacci numbers.

> ### A note on RPC
> Although RPC is a pretty common pattern in computing. It's often criticised.
> The problems arise when a programmer is not aware whether a function is local or if it's a slow RPC.
> Confusions like that result in an unpredictable system and adds unnecessary complexity to debugging. 
> Instead of simplifying software, misused RPC can result in unmaintainable spaghetti code.
> 
> - Make sure it's obvious which function call is local and which is remote
> - Document your system. Make the dependencies between components clear.
> - Handle error cases. How should the client react when the RPC server is down for a long time?
> 
> When in doubt avoid RPC. If you can, you should use an asynchronous pipeline - instead of RPC - like blocking
> results are asynchronously pushed to a next computation stage.
## Callback queue 
In general doing RPC over rabbitMQ is easy. A client sends a request message and a server replies with a response message.
In order to receive a response we need to send a 'callback' queue address with the request. We can use the default queue. Lets try it:
```
q, err := ch.QueueDeclare(
    "",     // name
    false,  // durable
    false,  // delete when unused
    true,   // exlusive
    false,  // noWait
    nil,    // arguments
)

ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
defer cancel()

err = ch.PublishWithContext(ctx,
    "",             // exchange
    "rpc_queue",    // routing key
    false,          // mandatory
    false,          // immediate
    amqp.Publishing{   
        ContentType:    "text/plain",
        CorrelationId:  corrId,
        ReplyTo:    q.Name,
        Body:       []body(strconv,Itoa(n)),
    }) 
```
> ### Message properties
> The AMQP 0-9-1 protocol predefines a set of 14 properties that go with a message.
> Most of the properties are rarely used. With the exception of the following.
> - Persistent: Marks a message as persistent (with a value of true) or transient (false). 
> - content_type: Used to describe the mime-type of the encoding. For example for the often used JSON encoding it is a good practice. to set this property to: application/json
> - reply_to: Commonly used to name a callback queue
> - correlation_id: Useful to correlate RPC response with requests.

### Correlation Id
Creating a callback queue for every RPC request is inefficient. A better way us creating a single callback queue per client.

That raises a new issue. Having received a response in that queue it's not clear to which request the response belongs.
That's when the `correlation_id` property is used. 
WE're going to set it to a unique value for every request.
Later, when we receive a message in the callback queue we'll look at the property.
and based on that we'll be able to match a response with a request.
If we see an unknown `correlation_id` value,
we may safely discard the message - it doesn't belong to our requests.

You may ask, why should we ignore unknown messages in the callback queue.
rather than failing with an error? it's due to a possibility of a race condition on the server side.
Although unlikely, it is possible that the RPC sever will die just after sending us the answer.
but before sending an acknowledgment message for the request.
If that happens, the restarted RPC server will process the request aging. 
That's why on the client we must handle the duplicate response gracefully. and the RPC should ideally be idempotent.
### Summary
![img.png](img.png)

Our RPC will work like this:
- When the Client starts up, it creates an exclusive callback queue.
- For and RPC request, the Client sends a message with two properties: `reply-to`. which is set to the callback queue and `correlation_id` which is set to a unique value for every request.
- The request is sent to an `rpc_queue` queue.
- The RPC worker (aka: server) is waiting for requests on that queue. When a request appears. It does the job and sends a message with the result back to the Client. using the queue from `reply_to` field.
- The client waits for data on the callback queue. When a message appears, it checks the `correlation_id` property. If it matches the value from the request it returns the response to the application.

## Putting it all together
The Fibonacci function:
```
func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
        }
}
```
We declare our fibonacci function. It assumes only valid positive integer input. 

rpc_server.go
```
package main

import (
        "context"
        "log"
        "strconv"
        "time"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
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
                "rpc_queue", // name
                false,       // durable
                false,       // delete when unused
                false,       // exclusive
                false,       // no-wait
                nil,         // arguments
        )
        failOnError(err, "Failed to declare a queue")

        err = ch.Qos(
                1,     // prefetch count
                0,     // prefetch size
                false, // global
        )
        failOnError(err, "Failed to set QoS")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                false,  // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        var forever chan struct{}

        go func() {
                ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
                defer cancel()
                for d := range msgs {
                        n, err := strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")

                        log.Printf(" [.] fib(%d)", n)
                        response := fib(n)

                        err = ch.PublishWithContext(ctx,
                                "",        // exchange
                                d.ReplyTo, // routing key
                                false,     // mandatory
                                false,     // immediate
                                amqp.Publishing{
                                        ContentType:   "text/plain",
                                        CorrelationId: d.CorrelationId,
                                        Body:          []byte(strconv.Itoa(response)),
                                })
                        failOnError(err, "Failed to publish a message")

                        d.Ack(false)
                }
        }()

        log.Printf(" [*] Awaiting RPC requests")
        <-forever
}
```

The server code is rather straightforward
-  As usual we start by establishing the connection. channel and declaring the queue.
- We might want to run more that one server process. In order to spread the load equally over multiple servers we need to set the prefetch setting on channel.
- We use Channel.Consume to get the go channel where we receive messages from the queue. Then we enter the goroutine where we do the work the response back.

The code for our RPC client: rpc_client.go
```
package main

import (
        "context"
        "log"
        "math/rand"
        "os"
        "strconv"
        "strings"
        "time"

        amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Panicf("%s: %s", msg, err)
        }
}

func randomString(l int) string {
        bytes := make([]byte, l)
        for i := 0; i < l; i++ {
                bytes[i] = byte(randInt(65, 90))
        }
        return string(bytes)
}

func randInt(min int, max int) int {
        return min + rand.Intn(max-min)
}

func fibonacciRPC(n int) (res int, err error) {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when unused
                true,  // exclusive
                false, // noWait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        corrId := randomString(32)

        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        err = ch.PublishWithContext(ctx,
                "",          // exchange
                "rpc_queue", // routing key
                false,       // mandatory
                false,       // immediate
                amqp.Publishing{
                        ContentType:   "text/plain",
                        CorrelationId: corrId,
                        ReplyTo:       q.Name,
                        Body:          []byte(strconv.Itoa(n)),
                })
        failOnError(err, "Failed to publish a message")

        for d := range msgs {
                if corrId == d.CorrelationId {
                        res, err = strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")
                        break
                }
        }

        return
}

func main() {
        rand.Seed(time.Now().UTC().UnixNano())

        n := bodyFrom(os.Args)

        log.Printf(" [x] Requesting fib(%d)", n)
        res, err := fibonacciRPC(n)
        failOnError(err, "Failed to handle RPC request")

        log.Printf(" [.] Got %d", res)
}

func bodyFrom(args []string) int {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "30"
        } else {
                s = strings.Join(args[1:], " ")
        }
        n, err := strconv.Atoi(s)
        failOnError(err, "Failed to convert arg to integer")
        return n
}
```