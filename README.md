# IoRT - High Velocity Data Streaming

At Fetch Robotics, our robots are coordinated by centralized backend services. Sensory information
and navigational statistics are streaming at a lightning pace to our cloud infrastructures. The
central server maintains a global state that oversees the operation of a robotics fleet. Due to the
physical limitation of robots in in our deployment site, data streaming at a massive scale has been
posed as a challenge to us.  

## Background

We coordinate robots by maintaining a real-time source of truth on the central server. This source
of truth is represented by the real-time data coming from robots’ sensor and navigational output.
Each robot is responsible for sending laser point clouds and planned trajectory for a navigational
path. Each message payload contains thousands of float point data and is sent at the interval of 10
Hertz. For production scale,, we often manage a fleet of hundreds of robots across multiple customer
sites.

### Technology Stack

The purpose of this section is to provide context into ROS, a multi process communication
framework which implements a pub/sub and client/services models similar to Redis and gRPC.

In here, we must give a brief overview of what ROS is and how its architecture influences our
decision on adapting Golang. ROS implements a computational graph where each node is a process that
performs computation.

![node_communication](https://raw.githubusercontent.com/calvinfeng/gophercon2019/master/node_communication.png)

Inside this framework, nodes are constantly exchanging data with one another either through direct
XMLRPC calls or long-lived TCP connections. Go’s first class support of concurrency through
go-routines and channels makes interfacing with ROS’s asynchronous pub/sub model very easy.

The streams node is a ROS based process responsible for streaming high velocity robotics data to the
server. To do this it has to manage multiple connections between numerous ROS nodes and our backend
services.  

### Constraints

Mobile robotics have much of the similar constraints of mobile IoT devices. The primary constraints
we face in production with respect to high velocity data streaming can be effectively summarized by
the following:

- Prioritization of CPU usage
- Battery Management
- Low Bandwidth
- Intermittent Network Connectivity

These practical constraints are difficult engineering challenges. Compounding these set of
constraints with complex languages without strong concurrency and networking support becomes an
intractable problem for a small team at a fast pace startup. Our approach with Go shows a robust and
scalable approach to addressing these constraints.

## Go Robot

### Concurrency

Go has significantly alleviated our pain in dealing with concurrency, without sacrificing performance.
Performance is critical to manage CPU usage and battery consumption with IoT based devices. We wrote
our data streaming node using `rosgo`, which we won’t be spending that much time on.

In Python, we would expect to see the following callback pattern.

```python
import rospy
from threading import Lock
from std_msgs.msg import String

mutex = Lock()
message_batch = []

def callback(data):
    with mutex:
        message_batch.append(data)

def main():
    rospy.init_node('gopher')
    rospy.Subscriber('topic_1', String, callback)
    rospy.Subscriber('topic_2', String, callback)
    rospy.Subscriber('topic_3', String, callback)
```

Context switch is a killer in performance. The same logic can be done in Go without the need for
putting a thread into waiting state because Go runtime scheduler will take care of goroutine swapping
for us.

```golang
import (
    "rosgo"
    "time"
)

func main() {
    rosgo.InitNode("gopher")
    sub1 := rosgo.NewSubscriber("topic_1", rosgo.StringMessage)
    defer sub1.Unregister()

    sub2 := rosgo.NewSubscriber("topic_2", rosgo.StringMessage)
    defer sub2.Unregister()

    sub3 := rosgo.NewSubscriber("topic_3", rosgo.StringMessage)
    defer sub3.Unregister()

    batch := []ros.StringMessage{}
    timeout := time.After(10 * time.Second)
    for {
        select {
        case msg := <-fanIn(sub1.Register(), sub2.Register(), sub3.Register()):
            batch = append(batch, msg)
        case <-timeout:
            return
    }
}
```

More importantly, before the fan in operation, we can choose to perform parallel data manipulation
before we multiplex the data into the main goroutine. This section of the talk aims to address how
concurrency and parallelism is easily handled with Go.

### Data Compression

Traditionally we would send our data to the server by serializing it into JSON text. Compression is
a must but compression is computationally expensive, thus we prefer Go. Our findings show gzip
compression works even better on protobuf messages for the majority of our data.

On average our payload size is on the order of kilobytes before compression in JSON format. Using
gzip to perform compression, we can get it down to one-third of its original size. With gzipped
protobuf messages, we can get it further down to one-fourth of its original size. On top of that, we
can default to use gRPC bidirectional streaming call easily in Go.


```golang
conn, err := grpc.Dial(":1234", grpc.WithInsecure())
if err != nil {
    log.Fatal(err)
}

cli := pb.NewDataStreamingClient(conn)

stream, err := client.PublishToDataStreams(context.Background())
if err != nil {
    log.Fatal(err)
}

for batch := range batchCh {
    payload := pb.DataPayload{Batch: batch}
    if err := stream.Send(&payload); err != nil {
        log.Error(err)
    }
}
```

With this approach, we enjoy the strong service contract between client and server while harvesting
the benefit of data compression provided by gzip on protobuf.

### Message Prioritization

Robots are constantly moving in and out of WiFi access points in a warehouse. This introduces delays
and connectivity issues. Real world connectivity also means low bandwidth availability from one area
to the next. The challenge is when bandwidth shrinks or connection drops, what to do with the data
we batched?

Our high velocity streams node needs to simultaneously stream multiple data sources. Each type of
data source has two forms of prioritization. We consider the priority for a given data type A’s
priority with itself as well as the priority of data type A against data type B.

![data_streaming_node](https://raw.githubusercontent.com/calvinfeng/gophercon2019/master/data_streaming_node.png)

We show how we designed a “best effort” message prioritization through heuristics that consider a
data type priority at a given bandwidth. Certain data sources need historical data, while others
need the most recent values.  For example, the position of a robot cannot be compromised because
robotic orchestration relies on this centerpiece information to assign robots to appropriate tasks.
However, the location data of an inventory item can wait.

The logic of controlling what to send and when to send is intuitively conveyed through channels.
When bandwidth drops, we throttle our send rate through a tight feedback loop. Periodically we
check whether bandwidth has resumed to its normal level before we remove our throttler. The
bandwidth monitor allows data type queues to dynamically adjust data fidelity when possible.

## Talk Format

1. [<1 mins] Speaker introduction
    - Who is the speaker?
    - What kind of software or solution is the speaker building with Go?
2. [3 mins] Robot Networking - the section aims to address the background and context of the problem
    - What does robot have to do with networking?
    - What kind of data are robots supposed to send to server?
    - Why are those data needed?
3. [2 mins] Mobile device challenges - the section aims to elaborate on the challenges
    - Why is sending data hard work?
    - What are the constraints and limitations imposed on robots in a warehouse?
4. [2 mins] Traditional ROS Approach - the section aims to address what is lacking in the current solution.
    - What’s wrong with C++ & Python on robot?
    - Why do we need concurrency on robot?
5. [15 mins] Go Robot - the section aims to provide a better solution.
    - How does the following keynotes address the challenges we face at Fetch?
        - Concurrency Pattern
        - Data Compression
        - Message Prioritization
6. [2 mins] Conclusion
    - Why you should consider Go when you build your next IoT mobile devices?
    - Summarize the success we have with adapting Go at Fetch Robotics.
