In general, a rate is a simple count of occurrences over time. However, there are several different techniques for measuring and limiting rates.

### Fixed Window 
In a fixed window algorithm, a window size of n seconds (typically using human-friendly values, such as 60 or 3600 seconds) is used to track the rate. Each incoming request increments the counter for the window. If the counter exceeds a threshold, the request is discarded. The windows are typically defined by the floor of the current timestamp, so 12:00:03 with a 60 second window length, would be in the 12:00:00 window.

The advantage of this algorithm is that it ensures more recent requests gets processed without being starved by old requests. However, a single burst of traffic that occurs near the boundary of a window can result in twice the rate of requests being processed, because it will allow requests for both the current and next windows within a short time. Additionally, if many consumers wait for a reset window, for example at the top of the hour, then they may stampede your API at the same time.

### Sliding window
This is a hybrid approach that combines the low processing cost of the fixed window algorithm, and the improved boundary conditions of the sliding log. Like the fixed window algorithm, we track a counter for each fixed window. Next, we account for a weighted value of the previous window’s request rate based on the current timestamp to smooth out bursts of traffic. For example, if the current window is 25% through, then we weight the previous window’s count by 75%. The relatively small number of data points needed to track per key allows us to scale and distribute across large clusters.

### Token Bucket
A token bucket maintains a rolling and accumulating budget of usage as a balance of tokens. This technique recognizes that not all inputs to a service correspond 1:1 with requests. A token bucket adds tokens at some rate. When a service request is made, the service attempts to withdraw a token (decrementing the token count) to fulfill the request. If there are no tokens in the bucket, the service has reached its limit and responds with backpressure. For example, in a GraphQL service, a single request might result in multiple operations that are composed into a result. These operations may each take one token. This way, the service can keep track of the capacity that it needs to limit the use of, rather than tie the rate-limiting technique directly to requests

### Leaky Bucket
A leaky bucket is similar to a token bucket, but the rate is limited by the amount that can drip or leak out of the bucket. This technique recognizes that the system has some degree of finite capacity to hold a request until the service can act on it; any extra simply spills over the edge and is discarded. This notion of buffer capacity (but not necessarily the use of leaky buckets) also applies to components adjacent to your service, such as load balancers and disk I/O buffers.

### Conclusion

With twice the rate of requests being processed occured near the boundary of a window, fixed window is passed.

Sliding Windows are a variant of fixed Windows. The more Windows you have, the higher the accuracy, but the more resources you consume. The fewer Windows you have, the less resources you consume, but the less accurate you get.

So We mainly consider the following two ways：token bucket and leaky bucket.

***Token Bucket vs Leaky Bucket***

- LB discards packets; TB does not. TB discards tokens.

- LB sends packets at an average rate. TB allows for large bursts to be sent faster by speeding up the output.

- TB allows saving up tokens (permissions) to send large bursts. LB does not allow saving

- With LB, when there are a large number of burst requests in a short period of time, each request has to wait in the queue for a while before it can be answered, even if the server is not under any load.

Because LB does not allow saving tokens to send large bursts. So if we have a large request, which is larger than the size of bucket, the request will not be processed.

So, I recommend choosing the token bucket algorithm.

### Reference
[how to design a scalable rate limiting algorithm](https://konghq.com/blog/how-to-design-a-scalable-rate-limiting-algorithm/)
[rate limiting strategies techniques](https://cloud.google.com/solutions/rate-limiting-strategies-techniques)
[分布式服务限流实战](https://www.infoq.cn/article/Qg2tX8fyw5Vt-f3HH673)
[leaky bucket tocken buckettraffic shaping](https://www.slideshare.net/vimal25792/leaky-bucket-tocken-buckettraffic-shaping)
