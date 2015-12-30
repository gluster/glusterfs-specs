Feature
-------
Events APIs for Gluster

Summary
-------
Eventing framework will emit notification whenever Gluster Cluster
state changes.

Owners
------
Aravinda VK <avishwan@redhat.com>


Detailed Description
--------------------
Let us imagine we have a Gluster monitoring system which displays
list of volumes and its state, to show the realtime status, monitoring
app need to query the Gluster in regular interval to check volume
status, new volumes etc. Assume if the polling interval is 5 seconds
then monitoring app has to run gluster volume info command ~17000
times a day!

How about asking Gluster to send notification whenever something is
changed?


How To Test
-----------
Start the eventsdash.py using `python
$SRC/events/tools/eventsdash.py`. Register this dashboard URL as
webhook using `webhook-add` sub command.

    gluster-eventsapi webhook-test http://<IP>:9000/listen

Where IP is hostname or IP of the node where eventsdash.py is
running. This IP should be accessible from all Gluster nodes.

If Webhook Test is OK from all nodes, then

    gluster-eventsapi webhook-add http://<IP>:9000/listen

eventsdash.py will show the Gluster Events.

User Experience
---------------
Run following command to start/stop Events API server in all Peers,
which will collect the notifications from any Gluster daemon and emits
to configured client.

    gluster-eventsapi start|stop|restart|reload

Status of running services can be checked using,

    gluster-eventsapi status

Gluster Events can be consumed using Websocket API or by using
Webhooks.

#### Consuming events using Webhooks:

Events listener is a HTTP(S) server which listens to events emitted by
the Gluster. Create a HTTP Server to listen on POST and register that
URL using,

    gluster-eventsapi webhook-add <URL> [--bearer-token <TOKEN>]

For example, if HTTP Server running in `http://192.168.122.188:9000`
then add that URL using,

    gluster-eventsapi webhook-add http://192.168.122.188:9000

If it expects a Token then specify it using `--bearer-token` or `-t`
We can also test Webhook if all peer nodes can send message or not
using,

    gluster-eventsapi webhook-test <URL> [--bearer-token <TOKEN>]

Configurations can be viewed/updated using,

    gluster-eventsapi config-get [--name]
    gluster-eventsapi config-set <NAME> <VALUE>
    gluster-eventsapi config-reset <NAME|all>

If any one peer node was down during config-set/reset or webhook
modifications, Run sync command from good node when a peer node comes
back. Automatic update is not yet implemented.

    gluster-eventsapi sync

Eventing Client can be outside of the Cluster, it can be run even on
Windows. But only requirement is the client URL should be accessible
by all peer nodes.(Or tools like ngrok(https://ngrok.com) can be used)

#### Consuming events using Websocket API:

Websocket API will be part of Gluster REST Server, all the
REST related configuration is applicable here.

    ws://hostname/v1/events

> WebSocket is a protocol providing full-duplex communication channels
> over a single TCP connection. The WebSocket protocol was
> standardized by the IETF as RFC 6455 in 2011, and the WebSocket API
> in Web IDL is being standardized by the W3C.(Ref:
> [Wikipedia](https://en.wikipedia.org/wiki/WebSocket))

`hostname` can be any one of the node in Cluster. If that node goes
down, application can connect to any other node in Cluster and start
listening to the events.

    from websocket import create_connection
    import time

    ws = create_connection("ws://hostname/v1/events")

    while True:
        ev = ws.recv()
        print "Received Event '{0}'".format(ev)
        time.sleep(1)

    ws.close()

Register the application using `gluster-rest app-add` command.

Design of Gluster REST Server is discussed
[here](http://review.gluster.org/13214)

Applications can persist the peers list(`GET /v1/peers`) of the
Cluster. If a connected node goes down in Cluster, application can
choose another node to connect and continue listening to the events.

Design
------

#### Event Types
Gluster events can be categorized into different types, for example

1. **User driven events** - When user executes any CLI command which
   changes the state of Cluster, Volume or Bricks. For example, Volume
   Create, Peer Attach, Volume Start, Snapshot Create, Geo-rep Create,
   Geo-rep start etc.
2. **Local events** - The events which can be detected locally and related
   to local resources. For example, Geo-rep worker going to Faulty,
   Brick process going down etc.
3. **Cluster events** or Events related to multiple nodes - The events
   which are not related to single node, but Cluster level
   information. Notification may sent out from multiple nodes without
   filtering.

Most of the User driven events are also Cluster events, but handled
differently compared to other Cluster events. Notifications for user
driven events will be sent only from node where command is run.

**Note:** All planned Gluster Events are listed in the end.

#### Recording the Events
`gf_event`, new API will be introduced to send message to socket
/var/run/gluster/events.sock. This new API will be available for
C, Python and Go.

This can be called from any component of Gluster. gsyncd will use the
python lib for sending message.

Example format of message(Format may change during implementation)

    Volume.Create=gv1
    Volume.Start=gv1
    Volume.Set=gv1,changelog.changelog,on
    Georep.State.Faulty=...

Pseudo code:

    gf_event(key, format, values..) ->
        if EVENTING_ENABLED{
            connect(EVENT_SOCKET) # AF_UNIX
            format_and_send(key, value)
        }

Example usage in Volume create(`cli/src/cli-cmd-volume.c` file)

    #if (USE_EVENTS)
            // On successful Volume creation
            if (ret == 0) {
                    gf_event (EVENT_VOLUME_CREATE, "name=%s", volname);
            }
    #endif

#### Agent - glustereventsd
Agent listens to `events.sock` for any new events from Gluster
processes, gathers additional information required and broadcasts to
all peer nodes by sending HTTP POST to Gluster REST
servers(/v1/listen). REST server will send that message to all
connected applications.

#### Architecture

In each node of Gluster Cluster,

      glusterd    glusterfsd       gsyncd          cli       ...
         |            |               |             |          |
         |            |               |             |          |
     (gf_event)   (gf_event)     (gf_event)    (gf_event)  (gf_event)
         |            |               |             |          |
         v            v               v             v          v
     +------------------------------------------------------------+
     |                      events.sock                           |
     |                   (format + broadcast)                     |
     |                                                            |
     +------------------------------------------------------------+

`glustereventsd` will be run in all the nodes of Cluster and sends
events independently.(Except for Websocket use case)

Cluster view(Websockets API),

    +-------------------+              +-------------------+
    |                   |              |                   |
    |   Node 1          |              |    Node 2         |
    |   REST server<----------+------------>REST server    |
    |                   |     |        |                   |
    |   Agent           |     |        |    Agent          |
    |                   |     |        |                   |
    +-------------------+     |        +-------------------+
                              |
    +-------------------+     |        +-------------------+
    |                   |     |        |                   |
    |   Node 3          |     |        |   Node 4          |
    |   REST server<-+  |     +----------->REST server     |
    |                |  |     |        |                   |
    |   Agent -------+--------+        |   Agent           |
    |                   |              |                   |
    +-------------------+              +-------------------+

Above figure shows event received in Node 3, which is broadcast to
all the other nodes using REST call(/v1/listen)

Cluster view(Webhooks)

    +-------------------+              +-------------------+
    |                   |              |                   |
    |   Node 1          |              |    Node 2         |
    |                   |              |                   |
    |                   |              |                   |
    |   Agent           |              |    Agent          |
    |                   |              |                   |     +-----------+
    +-------------------+              +-------------------+     | Webhook   |
                              +--------------------------------->|           |
    +-------------------+     |        +-------------------+     +-----------+
    |                   |     |        |                   |
    |   Node 3          |     |        |   Node 4          |
    |                   |     |        |                   |
    |                   |     |        |                   |
    |   Agent ----------------+        |   Agent           |
    |                   |              |                   |
    +-------------------+              +-------------------+

Above figure shows event received in Node 3, which is sent directly to
the configured Webhook. Each node can directly send events to
configured webhooks.

#### List of Gluster Events
##### User driven Events

1. Volume Create/Start/Stop/Set/Reset/Delete
2. Peer Attach/Detach
3. Bricks Add/Remove/Replace
4. Volume Tier Attach/Detach
5. Rebalance Start/Stop
6. Quota Enable/Disable
7. Self-heal Enable/Disable
8. Geo-rep Create/Start/Config/Stop/Delete/Pause/Resume
9. Bitrot Enable/Disable/Config
10. Sharding Enable/Disable
11. Snapshot Create/Clone/Restore/Config/Delete/Activate/Deactivate

##### Local Events

1. Change in Geo-rep Worker Status Active/Passive/Faulty
2. Brick process Up/Down
3. Socket disconnects
4. Bitrot files
5. Faulty Geo-replication process

##### Cluster Events

1. Quota crosses the limit
2. Gluster Cluster quorum is lost
3. Split brain state
4. Self heal started/Ended
5. Async Task completion(Rebalance, Remove-brick)
6. Snapshot hard limit and soft limit crosses

Cluster events are not planned with `Glusterd 1.0` since we need
distributed store to achieve that.

#### Future

1. Selectively Enable/disable eventing based on the Event keys, for
   example `gluster-eventsapi disable Volume.*`
2. Filters for Events(Channels/tag)
3. Integration with other projects like Skyring, Storaged etc.

Status
------
In Development

Current status
--------------
Design is in progress.

Related Feature Requests and Bugs
---------------------------------
None

Benefit to GlusterFS
--------------------
With Events API, we can monitor Gluster effectively.

Scope
-----

#### Nature of proposed change
- New service(`glustereventsd`) to listen to the messages sent from Gluster

#### Implications on manageability
New service `glustereventsd`

#### Implications on presentation layer
None

#### Implications on persistence layer
None

#### Implications on 'GlusterFS' backend
None

#### Modification to GlusterFS metadata
None

#### Implications on 'glusterd'
None

Dependencies
------------
None

Comments and Discussion
-----------------------
