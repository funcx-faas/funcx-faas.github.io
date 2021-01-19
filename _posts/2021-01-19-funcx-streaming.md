---
layout: post
author: Dennis Gannon, School of Informatics, Computing and Engineering, Indiana University 
title: Interacting with a Running FuncX Instance
---


This short report is an addendum to the report “[A Look at Parsl and FuncX: Two Excellent Parallel Scripting Tools for Clouds
and Supercomputers](https://funcx.org/2021/01/11/look-at-parsl-funcx.html).”  At the conclusion of that report it was stated
that one of the missing pieces of our analysis was a description of how distributed FuncX function instances can communicate
with other objects to handle remote interactions or to process streaming data.  Here is another way to state the problem.
FuncX gives you a way to package and launch a function on a remote resource, but you have no direct way to interact with that
executing function until it returns or otherwise terminates.  We present three different scenarios that show how to do this.
The programming patterns presented are not new, but they do illustrate new uses for FuncX.

Consider the following scenario.   You have a function that loads a deep learning vision model and you want it to run on a
remote CUDA-capable device and then use the camera on that device to capture a picture and return the analysis and the image
to you.   Even if the model has been previously cached on the remote device, loading it, moving the model to the CUDA engine
and then analyzing the image can take over 10 seconds. However, once it has been run through the CUDA pipeline, subsequent
images can be processed 100 times faster.    How can we interact with the function execution to tell it “Take another picture
now” (taking advantage of 100 fold speed up) and return the result of the inference and the image without having to terminate
the function’s execution?    

<img src="/images/blog/2021-01-19/Picture1.png" width="35%" style="border:0px solid black;">
{: style="text-align: center;"}

Figure 1. Interacting with a remote execution.
{: style="text-align: center;"}

In Figure 1 above we have a small desktop client application that communicates with our function execution.  There is a button at
the top to send a message to the waiting function to activate the camera and do the inference to recognize the image.   The first
image was a picture of a chair and it took 21.0 seconds.  The next was a picture of a desk.  Third image was of a book filled
office which was labeled “library”.   These last two images only took about 0.16 seconds.   The full transaction recorded on the 
client was

<img src="/images/blog/2021-01-19/Picture2.png" width="40%" style="border:0px solid black;">

The original function was launched from a Jupyter notebook on the same desktop that is running the client app.  When the client app
is closed it sends  a “quit”  message to the function which causes it to terminate normally and return to the jupyter notebook.
The key to making this work is the communication between the client app and the executing function.  The communication mechanism
used here is asynchronous queueing as illustrated below.

<img src="/images/blog/2021-01-19/Picture3.png" width="60%" style="border:0px solid black;">
{: style="text-align: center;"}

Figure 2.   Client and running instance communicating asynchronously through queues.
{: style="text-align: center;"}

It is important to note that this is not a ‘request-response’ scenario.  It is fully asynchronous.  Both the client and the function
instance run loops that monitor their input queue.  The client sends either a “quit” action or a “take picture” action depending on
the user input. A separate thread continuously monitors the input stream of messages from the function instance.   Messages coming
from the function instance are either informational, such as “model loaded” or “model cuda-ized” meaning that the model has been moved
to the Nvidia Cuda cores.   The client displays these in the text box.  The other message that the instance sends are the inference
classifications such as “library” followed by the time it took for the inference.   When it sees a “class” message it uses secure copy
(scp) to pull the image and display it.

We implemented this scenario with two different queue systems: RabbitMQ with Pika and AWS Simple Queue Service.   RabbitMQ is installed
as a separate service on a host visible to the network that has the client and the Jetson device.  Pika is the AMQP protocol python
implementation that allows communication with the RabbitMQ service.   Message consumers based on Pika use a continuation-passing style
in which the consumer provides a callback function that will be invoked when the queue has a message to deliver. Our FuncX function that
is run on the Jetson device takes the form below.

<img src="/images/blog/2021-01-19/Picture4.png" width="60%" style="border:0px solid black;">

When invoked by FuncX on the remote device it first establishes a connection and channel back to the RabbitMQ service.  It then goes about
loading and initializing the Resnet18 computer vision model and moving it to the Cuda Nvidia cores.  It next registers the callback
function and issues a “start_consuming” action.   At this point the function will wait for messages on the “command” queue.  When it
receives a “quit” command it stops consuming and returns to the continuation point which is the return from the function back to the FuncX
calling Jupyter notebook.  

The solution using AWS Simple Queue Service is conceptually easier to grasp.  SQS allows us to read a message from a queue if there is a
message there.   If not, the read operation waits for a prescribed interval of time and if nothing arrives in the queue, it will return.
The main body of the function is as follows,

<img src="/images/blog/2021-01-19/Picture5.png" width="40%" style="border:0px solid black;">

The function goes into a loop that start with a receive-message on its input queue.  It asks for 1 message and the maximum wait time is
5 seconds.  If a message arrives in that interval it is either “quit” or “take picture”.   If it is “quit” it send a signal back to the
client program signaling it to quit and it then return from the FuncX invocation.   

The source code for both solutions is in the [dbgannon/pars-funcx github repository](https://github.com/dbgannon/parsl-funcx) as
funcx-interactive-camera-final.ipynb (and html).
The desktop client program are also there.   You need to supply an AWS account information to run the aws example.  To run the rabbitmq
version you need to have an instance of rabbitmq running.  It doesn’t cost you anything to download and run it, but it is not a fun
installation.  

## Dealing with Streams

If we want to monitor a data stream from an instrument on the remote resource it is easy enough to write a function that will go into a
continuous loop gathering that data, doing needed analysis and sending results to some remote database.   The problem is that we may need
to stop the function, perhaps to replace it with a better version, but the FuncX client does not give us a way to do so.   There are
several solutions to sending graceful termination signals to such a function and we discuss one below.   

Another challenge is designing a protocol that allows to or more remotely executing functions to exchange messages reliably without
entering deadlock states.  

The scenarios we are interested in are

1. A function generates a stream of messages that can be sent to zero or more listeners.   For example, the sending function may be
drawing samples from an instrument.  If the sender is sending message it should not wait on a “blocking send” for a listener to show up
because the instrument may be generating interesting values that will be missed.   Consequently, it may be best to just push the messages
to some “device” that allows listeners to pick up the most recent values.   We will look at a sample where multiple senders are sending
messages to 0 or more listeners and we will use a publish-subscribe model.  Listeners select a “channel” to subscribe to and they will
receive only the messages that are sent on that channel.  You may have multiple listeners on a channel and listeners may come and go as
needed.   Senders can send messages on any channel and the send operations are non-blocking.   This example uses ZMQ service for the
messaging.  The routing device is a small 10-line program that runs on a server exposed to the network. 

<img src="/images/blog/2021-01-19/Picture6.png" width="60%" style="border:0px solid black;">
{: style="text-align: center;"}

2. In the case above we use a routing device to channel messages from senders to listeners and if there is no listener on a channel,
we just drop messages that come in on that channel.  In the second case we want to avoid dropping messages.  To do this we encapsulate
collections of function invocations together with a queue service into a single “component”.  Messages sent to the component queue are
processed in a first-in-first-out manner by one of the function instances.  In our example we consider the case of two “components”: a
front-end that receives messages from our control program and a backend that receives messages from the front-end.  The front-end
function processes the message and then save the result in an external table or it forwards the message to a back-end component that has
multiple function instances to process the request.   This design allows messages that require more cpu intensive processing to be
handled by a pool of back-end workers as shown below.

<img src="/images/blog/2021-01-19/Picture7.png" width="60%" style="border:0px solid black;">
{: style="text-align: center;"}

## Pub-Sub Routing with ZMQ in FuncX

The first case above is illustrated by a  demo in notebook funcx-zmq-final.ipynb that shows how four funcx instances an communicate
streams through a zmq pub-sub routing filter. The routing filter is a simple program that runs on a server with a "public" IP. In this
case it is a small NVIDIA jetson device and the access is via the local area network at address 10.0.0.6 and it is listening on port 5559
for the listeners to subscribe and on port 5560 for the publishers.  This is a standard ZMQ example and the core of the program is shown below.

<pre>
context = zmq.Context(1)
# Socket facing clients
frontend = context.socket(zmq.SUB)
frontend.bind("tcp://*:5559")
frontend.setsockopt(zmq.SUBSCRIBE, "")
  # Socket facing services
backend = context.socket(zmq.PUB)
backend.bind("tcp://*:5560")
zmq.device(zmq.FORWARDER, frontend, backend)
</pre>

In the demo we have two listeners. One listener subscribes to messages on channel 5 and the other on channel 3. We have two "sender"
functions that send a stream of messages to the router. One "Sally" runs on a small server and the other "Fred" is invoked from the
notebook. The only difference is that Sally sends a message every 0.2 seconds and Fred sends messages twice as often. Both alternate
messages between channels 3 and 5.  The code for Fred is below.

<img src="/images/blog/2021-01-19/Picture8.png" width="50%" style="border:0px solid black;">

In this case it sends only 22 messages of the form

Server#fredx 

For x in the range 0 to 21 alternating between channels 3 and 5.   It then sends a “Stop” message on both channels.

The stop message causes all listeners on channels 3 and 5 to terminate.  The listeners are also quite simple.

<img src="/images/blog/2021-01-19/Picture9.png" width="40%" style="border:0px solid black;">

The listener subscribes on the supplied topic channel and processes messages until the “Stop” message is received.  It is also easy
to stop a specific listener if we provide the listener with a unique id so that it can stop when it sees a Stop message with that ID
attached.  This is important for the case when a listener needs to be replaced by an upgraded instance without missing messages.  One
starts the new version and then stop the old version with its specific kill signal.   The Jupyter notebook illustrates this by showing
how the listeners receive the messages from the senders in interleaved order.  

## Reliable Messaging Using a Component with Queue Model 

To implement the solution in case 2 above we need a queue management system with the following properties

1.	It must allow FIFO queues to be easily created and destroyed.

2.	To ensure the solution remains deadlock free and termination guarantees it must be possible for a process to read from the head
of the queue and, if there is nothing there the reader is released in a bounded amount of time.  

3.	The queues should be able to hold an unbounded number of messages.

Of course, 3 is impossible, so we satisfy ourselves with queues built into substantial storage backends.  The easiest way to do this
is to use Azure storage queues or the AWS simple queue service SQS.  SQS is simple to use and it is very inexpensive.  (For the work
on this project my expenses were far less than $1.)  

For the demo we have two components:

1.	A Front-end component that receives messages and processes them in FIFO order. If the message is "forward" it passes the message to
a Back-end component. Otherwise if the message is not "Stop", it processes message and stores the result in a table. The table we use is
in Azure Storage Service because it is cheap and reliable.

2.	The Back-end component consists of one or more instances of a backend processor functions which pull messages from the input queue for
that component. We can control throughput of the back-end component by increasing or decreasing the number of functions servicing the
queue. When the back-end processing functions complete and execution they store the result in the queue.

The function we incorporate into the front end is as follows.

<img src="/images/blog/2021-01-19/Picture10.png" width="50%" style="border:0px solid black;">

In this demo every message is a simple python dictionary with a field called “action” which tell the function what action to take.    

The details of the auxiliary function are in the full code in the notebook aws-sqs-example in the [repository](https://github.com/dbgannon/parsl-funcx).
The back end function is similar

<img src="/images/blog/2021-01-19/Picture11.png" width="50%" style="border:0px solid black;">

The following simple wrapper creates an instance of the components.   The parameters are:

1.	the base name of the component (like "Front" or "Back")

2.	the name of the input queue for this component.

3.	the output name which could be another queue name or the table name.

4.	repl_factor: the number of function instances of the type of func_id to create.

5.	the end_point that will be the host service for the function instances.

6.	func_id the uuid of the function resulting from funcx registration as above.

<img src="/images/blog/2021-01-19/Picture12.png" width="60%" style="border:0px solid black;">

This launcher creates repl_factor copies of our function instance.  Running this on Kubernetes launches one pod per instance with
each listening to the input queue.   The return value is the list of funcx return values for each instance.  

### Final thoughts

The solutions above are somewhat ad hoc and the programming patterns are not new.  A possible improvement to FuncX would be to make
a dedicated message queue service available.  This could be an extension of existing Globus functionality already being used in FuncX.     




