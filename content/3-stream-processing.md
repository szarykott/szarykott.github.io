+++
title = "Brief description of stream multiplexing in Rust"
date = 2021-09-04
+++

This post describes an idea for a stream multiplexing architecture. Issue is discussed in the context of Rust.
<!-- more -->

## The task

Application's task is to concurrently pull messages from multiple streams and multiplex messages to output sinks.

Clients are created dynamically by calling an endpoint and providing a predicate messages must fulfill. In case no client is connected stream messages need to be pulled to avoid queuing.

## Overview of solution

Chosen implementation language - Rust - puts a series of constraints on a solution, mainly related to difficulties in dealing with mutable shared state. I chose to use channels for communication between components. This way sharing memory happens due to message passing.

<br/>

![desired flow](./../../images/3/dataflow.png)

<br/>

**Point 1** - `Router` pulls messages from a stream. The stream can be a RabbitMq messages stream, WebSocket connection or other that adheres to a `Stream` abstraction.

**Point 2** - `Router` maintains a collection of sinks and multiplexes an incoming stream of messages into respective sinks. A sink can be as simple as console output or as complex as WebSocket connection.

**Point 3** - `Supervisor` when called by external clients informs all routers about new output.

**Point 4** - `Supervisor` creates all outputs.

An important thing to note is that `Router` *pulls* messages from an stream and *pushes* them to sinks. It also has to be able to dynamically accept new sinks. I did not find any other way to implement it other than event loop. It listens for new messages from an input stream and supervisor and pushes messages to sinks.

```rust
pub struct Router;

impl Router {
    pub async fn run<I, O, S>(input_stream: I, new_outputs: O)
    where
        I: Stream<Item = (usize, usize)> + Unpin,
        O: Stream<Item = (usize, S)> + Unpin,
        S: Sink<usize, Error = String> + Unpin,
    {
        let mut input_stream = input_stream.fuse();
        let mut new_outputs = new_outputs.fuse();

        let mut state: HashMap<usize, S> = HashMap::new();

        loop {
            select! {
                next_item = input_stream.next() => match next_item {
                    Some(next_item) => Self::broadcast(&mut state, next_item).await,
                    None => break
                },
                next_output = new_outputs.next() => match next_output {
                    Some(next_output) => Self::new_recipent(&mut state, next_output),
                    None => break
                }
            }
        }
    }
    // other functions ommited for brevity
}
```

This simple example defines `I` as input stream which items are `(usize, usize)` tuples, where former is message key and latter is its value. `O` is a stream of supervisor orders, `(usize, S)` tuples, where former is the same messages key as in input stream and `S` is a sink for messages. A state is maintained in a HashMap that aggregates all keys and its respecive sinks. 

This particular implementation lacks cleaning and thus leaks memory. A simple cleaning routine might be to inspect state of all sinks every X messages and clean closed sinks (this will require adding more trait bounds to `S`).

## A completed example

An input and orders stream might look like that:

```rust
pub struct DummyStream(pub usize);

impl Stream for DummyStream {
    type Item = (usize, usize);

    fn poll_next(mut self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        self.0 += 1;
        Poll::Ready(Some((if self.0 % 2 == 0 { 1 } else { 0 }, self.0)))
    }
}

pub struct Orders(pub usize);

impl Stream for Orders {
    type Item = (usize, Output<Box<dyn Fn(usize)>>);

    fn poll_next(mut self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        self.0 += 1;
        match self.0 {
            1 => Poll::Ready(Some((
                1,
                Output(Box::new(|i| println!("Output is odd: {}", i))),
            ))),
            2 => Poll::Ready(Some((
                2,
                Output(Box::new(|i| println!("Output is even: {}", i))),
            ))),
            _ => Poll::Pending,
        }
    }
}
```

Input is just a stream of subsequent integer numbers. If number is even, key is 1 and 0 otherwise. `Orders` is a stream of only two messages that define different `Output`'s for even and odd numbers.

`Output` is defined as:
```rust
pub struct Output<T>(pub T);

impl<T> Sink<usize> for Output<T>
where
    T: Fn(usize),
{
    type Error = String;

    fn start_send(self: Pin<&mut Self>, item: usize) -> Result<(), Self::Error> {
        (self.0)(item);
        Ok(())
    }

    // rest of Sink functions omitted
}

```

It simply accepts a closure that dictates what to do with next stream item.

Putting it all together, a sample program might look a bit like this:

```rust
#[tokio::main]
async fn main() {
    let stream = input_stream::DummyStream(0);
    let orders = input_stream::Orders(0);

    router::Router::run(stream, orders).await;
}
```

It outputs something like this:
![desired flow](./../../images/3/output.png)

Streams are thus multiplexed to different sinks!

Important thing about this implementation is complete lack of locks. There can be as many input streams as needed, each having its own router. Since most channel libraries allow cloning senders, a single output sink can be cloned and given away to multiple routers. Output sink can be as simple as in this example, but it can also be more complex, such as component that sends received message through WebSocket connection, but at the same time it has to be able to receive Ping messages and responds with Pongs. It slightly complicates the implementation, as Output Sink has to run an event loop of its own, but idea stays the same.  

Downside of this implementation is maintaining state. A lot has been omitted, such as aforementioned cleaning, fault tolerance, retrying, etc.

I am wondering what would be the best way to achieve the initial goal. Are there any better implementations or architectures in Rust?