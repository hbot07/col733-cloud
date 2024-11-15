# Lab3: High Throughput Chain Replication using CRAQ

In this assignment you will implement [`CRAQ: A Distibuted Object-Storage System`](https://www.usenix.org/legacy/event/usenix09/tech/full_papers/terrace/terrace.pdf)
(Strong consistency variant - refer Section 2 of the paper)
from scratch and improve the read throughput as compared to the basic 
implementation of `Chain Replication` that we have provided. The read-write 
history should be linearizable.

## Specifications
- You have to implement both server and client side of the system.
- The system should be able to handle `SET` and `GET` requests.
- The system stores key-value pairs. Both key and value are strings.
- The system should be able to handle multiple clients (upto 32) concurrently.
- You need not worry about the fault tolerance of the system.
- The system consists of a single chain of 4 servers.

## Background

You are provided with a library `core/` which will help you in setting up the
server framework, handling connections and transfer of messages. You can use
this library to implement the server side of the system.
The library provides you with the following functionalities:
- `core/message.py` : To define the message format for communication between servers.
    The same format is used by the client to communicate with the server.
    All the messages should be of type `JsonMessage` and can contain any number of fields.
- `core/server.py` : Each server process will be an instance of the `Server` class.
    It listens for incoming connections and handles the messages in a separate
    thread. You can extend this class to implement your server.
    You should implement the `_process_req` method to handle the incoming requests.
- `core/network.py` : To establish connections between servers.
    The class `TcpClient` is used to establish a connection with a server.
    It maintains a pool of connections to the server and reuses them.
    The `send` method sends a message to a server and returns the response message from the server. \
    The class `ConnectionStub` maintains all the connections to the servers in the cluster.
    It takes a set of servers it can connect to and establishes connections to all of them.
    An instance of this class is made available to all the servers in the cluster.
    You are supposed to use this class to send messages to other servers in the cluster.
    It provides an interface to send messages to any server by name in the cluster without 
    worrying about the connection details. It provides the following methods:
    - `send` : To send a message to a server by name and get the response.
    - `initiate_connections` : To establish connections to all the servers in the cluster. 
        This method should be called during the cluster formation before sending any messages.
- `core/cluster.py` : The class `ClusterManager` is used to manage the cluster of servers.
    It starts the servers and establishes connections between them according to a specified topology.
    You can extend this class to implement your cluster manager 
    (See the provided chain replication implementation for reference).
    You may overwrite the `create_server` method to create your server instance.
    This class expects the following parameters:
    - topology: The cluster topology information of the form `dict[ServerInfo, set[ServerInfo]]`
    representing a set of servers each server can send messages to. This information is used to create `ConnectionStub`
    which maintains the connections between the servers as explained above.
    - master_name: Let's assume that the tail is the master for the sake of this assignment. This currently does not have much significance.
    - sock_pool_size: The number of sockets to be maintained in the pool for each server. This directly affects the number of concurrent connections that can be maintained by the server.
- `core/logger.py` : To log the messages and events in the system.
    You can use the `server_logger` and `client_logger` objects to log the messages.
    The client logs will be used to check the linearizability of the system.
    You can change the log level to `DEBUG` to see the logs on the console and debug your code.

## System Design of Chain Replication

### CrClient

The `CrClient` object can be used to send `SET` and `GET` requests to the cluster. The client picks the appropriate server to send the request based on the request type. If the request is a `SET` request, it sends the request to the `head` server of the chain. If the request is a `GET` request, it sends the request to the `tail` server of the chain.

The message format for the requests is of type `JsonMessage` and as follows:
- `SET` request: `{"type": "SET", "key": <key>, "value": <value>}`
- `GET` request: `{"type": "GET", "key": <key>}`

### CrCluster
We will manage a cluster of servers (in this assignment, we will have 4 servers) namely a, b, c and d. 
The interconnection among the servers will be mananged by the ClusterManager. 

![Image](docs/cr-chain.png)

- Topology: For each server we will store the set of servers that, this server can 
send messages to.
    ```python3
        topology={self.a: {self.b}, # a can send message to b
            self.b: {self.c},       # b can send message to c
            self.c: {self.d},       # c can send message to d
            self.d: set()}          # d does not send message to any server
    ```
- Each server will also store its previous and next server in the chain.
- The first server of the chain is called `head` and the last one is called `tail`.
- The `connect` method of `CrCluster` returns a `CrClient` object which can be used to send requests to the cluster.

### CrServer

This extends the `Server` class from the `core` library. It stores the key-value pairs in its local dictionary. It additionally stores the previous and next server in the chain. `_process_req` is called whenever a message is received by the server. The server processes the request and sends the response back to the client. 

#### Handling SET Request
Whenever the `head` server receives a `SET` request, it updates its local
dictionary and sends this request to its adjacent server. The `tail` upon receving
this request sends the acknowledgement as `"status" : "OK"` message to the
penultimate node of the chain. This way, the acknowlegment is sent back.

When this acknowledgement reaches back the `head` node, we say that the `SET`
request is completed and the head node sends the acknowledgement to the client.

#### Handling GET Request
In Chain replication, the `GET` request is always sent to the `tail` node. Hence
when it recieves such a request, it sends the response from its local dictionary.

## Checking Linearizability

**PRE-REQUISITE**: You need to have `go` version >= 1.21 installed on your system (baadal VM). You can find the installation instructions [here](https://golang.org/doc/install).

To test that the read-write history of the system is linearizable, we have used
[Porcupine](https://github.com/anishathalye/porcupine). You dont need to
understand its inner working. We have provided you the interface to interact with it.

We use the logs from the client to check if the history is linearizable or not. 
The `lcheck` library is used for testing. 
This assumes that the initial value of any key is `"0"` if no explicit SET call is made.

Run this command to test linearizability in lcheck directory:
```bash!
go run main.go <client-log-file-path>
``` 

The testcases can consist of multiple worker threads (clients) sending requests concurrently.
A request can be a `SET` or a `GET` request and the client logs the request and the response.
The following is the expected format of the client logs:
```
12:15:50 INFO worker_0 Setting key = 0
12:15:50 INFO worker_1 Getting key
12:15:50 INFO worker_1 Get key = 0
12:15:50 INFO worker_0 Set key = 0
```
The linearizability checker looks for 4 types of events in the logs:
- `Setting <key> = <value>` : A `SET` request is made by the client.
- `Set <key> = <value>` : The `SET` request is completed.
- `Getting <key>` : A `GET` request is made by the client.
- `Get <key> = <value>` : The `GET` request is completed.

NOTE: Here, worker and client are used interchangeably.

It also fetches the worker id from the log to track the order of events.
The worker name must follow the format `worker_<id>` and be unique.
A worker should only send requests one after the other, not concurrently.

Kindly ensure, that you do not modify any of the logging formats. 
This is important for the linearizability checker to work correctly and grade your submission.
A sample log file is provided in the `logs/` directory for reference.

## CRAQ
Taking the implementation of Chain Replication as reference, you have to 
implement CRAQ. Kindly use the `core/` library to setup the server framework,
handle connections, and transfer messages.

In CRAQ, the `GET` requests could be served by any server (not just the tail, as
opposed to the case of Chain Replication - Refer Section2.3 of the paper). Hence, the read throughput should be
comparatively high for CRAQ (refer Figure 4 of the paper).

You need to complete the the CRAQ implementation in the `craq` directory.
Do not modify the names of the files or classes provided in any of the directories.
We will be using the same names in our testcases.

## Measuring Throughput

The throughput of the system can be measured by the number of requests processed by the system in a given time. You can trigger requests from multiple clients concurrently to measure the throughput of the system. Each client should send requests sequentially for a specified duration. The throughput can be calculated as the total number of requests processed by the system in that duration.

In order to measure the throughput, you should limit the bandwidth of the network. You can use the `tc` command to limit the bandwidth of the network. We have provided the steps to limit the bandwidth in the Makefile. You can use the following command to limit the bandwidth to 100kbps:
```bash
make limit_ports
```
Please ensure that you limit the bandwidth for throughput measurement. Make sure the values `START_PORT` and `BANDWIDTH` in the Makefile are set correctly.

NOTE: The `tc` command requires `sudo` permissions. You can run the `make limit_ports` command with `sudo` permissions. This command works only on the linux machines. You are encouraged to use the provided `baadalvm` for this assignment.

## Submission

You need to submit a zip file of the `craq` directory. The zip file should be named as `<entry_number>_<name>.zip`.
Your submission should contain the following files:
- `craq/__init__.py`
- `craq/craq_server.py`
- `craq/craq_cluster.py`
- `craq/<any-other-file.py>`

## Evaluation
The read-write history generated by your implementation should be linearizable.
And the read-throughput of CRAQ should be higher than basic chain replication.

- 0 marks: Non-linearizable history.
- +20 Marks: Correct linearizable history of CRAQ with improved read-throughput.
    Note that your submission should achieve a minimum throughput improvement of 2X as compared to the basic chain replication for clean reads. Otherwise, you will not be eligible for these 20 marks.
- +20 Marks: Relative grading based on throughput under various tests.

# Bayou

In this lab, we implement Bayou. We do several simplifications over Bayou. 

**No Lamport clocks:** The weakly-connected distributed Bayou servers are just
processes on a single system. Therefore, there is no question of clock drift
between the servers. We need not use Lamport clocks in our setup. However,
system clocks *can go back* in time due to NTP syncs. This can be a problem 
since Bayou's design assumes monotonic clocks. We therefore use monotonic clocks
using Python's `time.monotonic` instead of system clocks.

**Separate committed/tentative states and logs:** Bayou maintains a database
where each row has two bits to signify whether the row is tentative/committed.
This is done to de-duplicate states for reduced memory consumption. We are not
going to store very large states in this lab. Therefore, for simplicity, every
server simply maintain two different states: one committed and one tentative.
Similarly, for simplicity, every server maintains two different logs: one
committed and one tentative.

**No conflict resolution:** Bayou does automatic conflict resolution. In each
write, applications provide a conflict detection and a conflict resolution
function. In the paper's calendar example, users specify multiple meeting start
times. A conflict is detected if the room is booked by someone else at an
overlapping slot. Conflict is resolved by booking at an alternate time. Writes
in this lab never have a conflict and therefore conflicts never need to be
resolved.

**Invertible writes:** Bayou maintains a separate *undo log* so that tentative
logs can be rolled back. For example, let us say `x=0` in the tentative state
before we do a tentative write of `x=4`. Bayou will remember `x=4` in the
tentative portion of the log.  Tentative logs can be exchanged between servers.
In undo log, it needs to remember `x=0` to be able to rollback `x=4`. Undo logs
are never exchanged; each server locally maintains its own undo logs. In our lab
setup, we will not maintain undo logs.  We assume that the writes are
invertible. For example, if writes are of the form `x+=1`, we can undo it by
subtracting `1`.

**No garbage collection, FT, persistence:** We never garbage collect logs so we
need not maintain the `O` timestamp. We also don't crash servers so we don't
want to think about persistence. We keep all states and logs in memory.

All these simplifications leaves us with implementing the crux of Bayou. We
still need to maintain `C` and `F` timestamps to do anti-entropy between
servers. Our writes are non-commutative so that simple CRDTs can not work, i.e,
rolling back is necessary. Our tests will send special messages to create and
heal network partitions.

To fulfill the above simplifications, our state is going to simply be a string.
Clients send operations to append a character to the state. Appends are
invertible: remove last character from the string. Appends can always be
applied, i.e, they never conflict with other appends. Appends are not
commutative.

Other states and operations can also fulfill above simplifications. For example,
the state can be a natural number and clients can send arithmetic operations
like `+` and `*` (assuming client never tries to multiply with zero).


## System Design
### Core Components
We have same system core components as in lab3. Namely:-
1. Cluster:- Knows the connection topology of the servers and is responsible for starting and stopping the servers.
2. Server:- It is responsible for handling requests from clients. Owns a connection_stub in order to communicate with the other servers if required.
3. ConnectionStub:- Responsible for falicitating sending/receiving messages between servers.
4. Client:- Represents the interface for the end user of the system.

Simulating network partition:- Bayou is meant to be used in an unreliable network connections. So in order to simulate the partition we have added
additional methods in ConnectionStub (`sever_connection`, `unsever_connection`).  `sever_connection("a")` will not actually close the tcp connection to `a`, it will
just blacklist `a`, and any attempt to send messages to `a` using the `connection_stub` will result in failure.

### BayouSpecific Components
1. BayouServer:- In addition to process requests from the clients, the server also needs to perform `anti_entropy` with other servers once in a while.
2. BayouStorage:- Storage encapsulates `committed_logs`, `tentative_logs`, `committed_state`, `tentative_state`, `c` and `f`, as discussed above and in the paper.
The interfaces provided by the storage is as follows, read their docstrings for more details:-
  1. `commit(list[LogEntry])`
  2. `tentative(SortedList[LogEntry])`
  3. `apply(list of committed LogEntry, sorted list of tentative LogEntry)`
  4. `anti_entropy(c, f)`
In this lab we are basically asking you to implement these interfaces.
3. BayouAppServer:- Bayou can be used to implement any kind of app in contrast to CRAQ that is exclusively a key-value store, e.g in Bayou paper they implement meeting booking app and biblography app.
However there will be some components in all these apps, we have pulled the common components in `BayouServer` and moved application specific stuff in `BayouAppServer`. In lab4 the app is basically
a string appender.
4. BayouAppClient:- Similarly, the client will also be application specific. Since our app is string appender the client provides two interface `append_char()` and `get_string()`
5. AppLogEntry and AppState:- Similarly, log's data structure and state's data structure is also app specific. 


## Setup

```bash
pip install -e '.'

make test_storage        # Test Storage's implementation
make test_client         # End-to-End testing from client to server
```


## Deliverables
1. Methods that are to be implemented are marked as #TODO. Just implementing those methods is enough to pass this lab.
Feel free to create new helper methods to be used in those methods.
You can find the required TODOs using `grep -rni --include="*.py" "TODO" ./bayou`
2. Restrict all your changes to `app.py`, `storage.py`, and `server.py`.
3. Zip your submission using `bash create_submission <entry_num>`.
4. Run the evaluation script provided to be sure that the evaluation script is compatible with your submission:-
   1. cd `evaluation`
   2. Run `python evaluate.py <path_to_submission_zip>`
   3. Check scores in `scores/<entry_num>.csv`.
   4. Note that the tests are not exhaustive, more hidden tests will be added during evaluation.
5. Sumbit on moodle.
   
Recommended way to complete the assignment:-
1. Make all the tests in `test_storage` pass.
2. Make all the tests in `test_client` pass.
3. Note that the tests provided by us are not exhaustive and sucessfully passing them doesn't guarantee correctness, they merely act as
sanity checks. Keep in mind that your system must provide eventual consistency in any scenario it finds itself in.
So writing good quality tests is very crucial.

