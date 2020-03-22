Backup request function can optimize the long tail problem of read delay, suitable for users with low consistency requirements.

## Investigation
----------------
There are two ways to implement backup request.

### Hedged requests
A client first sends one request to the replica believed to be the most appropriate, but then falls back on sending a secondary request after the first request has been outstanding for more than the 95th-percentile(or 99th-percentline, etc) expected latency. The client cancels remaining outstanding requests once the first result is received. This approach limits the additional load to approximately 5%(1%) while substantially shortening the latency tail. This approach limits the additional load to approximately 5%(1%) while substantially shortening the latency tail.

This approach limits the benefits to only a small fraction of requests(the tail of the latency distribution).

### Tied requests
the client send the request to two different servers, each tagged with the identity of the other server (“tied”). When a request begins execution, it sends a cancellation message to its counterpart. The corre- sponding request, if still enqueued in the other server, can be aborted imme- diately or deprioritized substantially.

There is another variation in which the request is sent to one server and forwarded to replicas only if the ini- tial server does not have it in its cache and uses cross-server cancellations.

This approach limits the benefits to not only the tail of the latency, but also median latency distribution. But it result in higher network load.

