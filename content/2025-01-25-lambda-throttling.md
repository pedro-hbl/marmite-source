# Understanding and Mitigating AWS Lambda Throttling in High-Concurrency Workloads

## Introduction

When dealing with high-concurrency workloads, scaling AWS Lambda effectively while avoiding throttling can become a challenge. This post explores a real-world scenario where an application(just like a worker), written in Kotlin, processed over 1,000,000 records in a blob located in S3 using a custom asynchronous iteration method. Each record triggered an asynchronous Lambda invocation that interacted with DynamoDB. However, the setup led to `429 Too Many Requests` errors occurring consistently during peak loads exceeding 10,000 TPS, indicating throttling issues with AWS Lambda. 
The article will:

1. **Outline the problem** faced while processing high-concurrency workloads.
2. **Explain AWS Lambda throttling mechanisms**, based on the [AWS Compute Blog article by James Beswick](https://aws.amazon.com/blogs/compute/understanding-aws-lambdas-invoke-throttle-limits/).
3. **Discuss solutions** to mitigate throttling.

4. **TBD** Maybe in the future I'll Provide a real-world proof of concept **(POC)** to evaluate each mitigation technique.

---

## Use Case

To better illustrate the challenges and solutions, consider the following use case:

- **Dataset:** The workload involves processing a large file with 1 million records stored in an S3 bucket.
- **Data Characteristics:** Each record contains 8 columns of strings, primarily UUIDs (36 bytes each). This results in approximately 288 bytes per record.
- **Worker Configuration:** The application is deployed on a SINGLE node with the following specifications:
  - **vCPUs:** 4
  - **RAM:** 8 GB

### Resource Calculations

1. **Memory Requirements:**
   - Each record occupies 288 bytes.
   - For 100 concurrent coroutines:
     - \( 288 * 100 = 28,800 bytes approx 28.8KB \)
   - Adding a 20 KB overhead per coroutine for runtime management:
     - \( 100 * 20KB = 2,000KB approx 2MB \)
   - Total memory consumption:
     - \( 28.8KB + 2,000KB = 2.028MB \)

2. **CPU Considerations:**
   - Each vCPU can handle approximately 100-150 threads (or coroutines) effectively, depending on the workload.
   - For this use case, 4 vCPUs are sufficient to manage 100 concurrent coroutines with minimal contention.

This setup ensures that the system remains stable while processing a high volume of records efficiently.

## The Challenge

### Problem Context

A workload involving processing a large file of over 1,000,000 records can utilize concurrency in Kotlin to invoke AWS Lambda for each record. The Lambda function in this case performed a `putItem` operation on DynamoDB.

Here’s an example of the Kotlin code for `mapAsync`:

```kotlin
suspend fun <T, R> Iterable<T>.mapAsync(
    transformation: suspend (T) -> R
): List<R> = coroutineScope {
    this@mapAsync
        .map { async { transformation(it) } }
        .awaitAll()
}

suspend fun <T, R> Iterable<T>.mapAsync(
    concurrency: Int,
    transformation: suspend (T) -> R
): List<R> = coroutineScope {
    val semaphore = Semaphore(concurrency)
    this@mapAsync
        .map { async { semaphore.withPermit { transformation(it) } } }
        .awaitAll()
}
```

This method processes records significantly faster than a standard `for` loop, but it can flood the system with Lambda invocations, triggering throttling. The `429 Too Many Requests` errors can be attributed to:

1. **Concurrency Limits**: AWS imposes a limit on the number of concurrent executions per account.
2. **TPS (Transactions Per Second) Limits**: High TPS can overwhelm the Invoke Data Plane.
3. **Burst Limits**: Limits the rate at which concurrency can scale, governed by the token bucket algorithm.

### Observed Errors

- **429 Too Many Requests**: Errors indicate that the Lambda invocations exceeded allowed concurrency or burst limits.
- DynamoDB “Provisioned Throughput Exceeded” errors occurred during spikes in DynamoDB writes.

---

## AWS Lambda Throttling Mechanisms

AWS enforces three key throttle limits to protect its infrastructure and ensure fair resource distribution:

### 1. **Concurrency Limits**

Concurrency limits determine the number of in-flight Lambda executions allowed at a time. For example, with a concurrency limit of 1,000, up to 1,000 Lambda functions can execute simultaneously across all Lambdas in the account and region.

### 2. **TPS Limits**

TPS is derived from concurrency and function duration. For instance:

- Function duration: 100 ms (equivalent to 100ms =100 × 10<sup>-3</sup> = 0.1s)
- Concurrency: 1,000

TPS = Concurrency / Function Duration = 10,000 TPS

If the function duration drops below 100 ms, TPS is capped at 10x the concurrency.

### 3. **Burst Limits**

The burst limit ensures gradual scaling of concurrency, avoiding large spikes in cold starts. AWS uses the token bucket algorithm to enforce this:

- Each invocation consumes a token.
- Tokens refill at a fixed rate (e.g., 500 tokens per minute).
- The bucket has a maximum capacity (e.g., 1,000 tokens).

For more details, refer to the [AWS Lambda Burst Limits](https://docs.aws.amazon.com/lambda/latest/dg/invocation-scaling.html).

---

## Mitigation Strategies

That being said, several approaches can be employed to mitigate the throttling scenarios observed in this case. These techniques aim to address the specific constraints and challenges imposed by the problem:

### 1. **Limit Concurrency Using Semaphore**


Concurrency in Kotlin can be limited using the `mapAsync` function with a specified concurrency level:

```kotlin
val results = records.mapAsync(concurrency = 100) { record ->
    invokeLambda(record)
}
```

This implementation leverages coroutines in Kotlin to handle asynchronous operations efficiently. We don't want to deep dive here in how coroutines work, but think of it as a tool that allow lightweight threads to run without blocking, making it possible to manage multiple tasks concurrently without overwhelming system resources.

In the use case described, where the workload involves processing millions of records within 100 concurrent coroutines, the concurrency level of 100 was chosen as a reasonable limit. This decision balances the capacity of the node, configured with 4 vCPUs and 8 GB of RAM, against the resource requirements of each coroutine. For example, each coroutine processes records with a memory overhead of approximately 28.8 KB per record, plus 20 KB for runtime management. This setup ensures stability while maximizing throughput within the system’s constraints.

By introducing a Semaphore, the number of concurrent tasks can be restricted to this specified level. This prevents overloading the Lambda concurrency limits and reduces the risk of 429 Too Many Requests errors, ensuring that the system remains stable and performs reliably.

#### Estimated Time to Process

Using the following parameters:
- <code>T</code>: Execution time for a single Lambda invocation.
- <code>n</code>: Number of concurrent Lambda invocations.
- <code>Total Records</code>: Total number of records to process.

The total processing time can be calculated as:

```html
Total Time = (Total Records / n) * T
```

#### Example with <code>T = 100 ms</code>

Given:
- <code>Total Records = 1,000,000</code>
- <code>n = 100</code>
- <code>T = 100 ms</code>

Substituting into the formula:

```html
Total Time = (1,000,000 / 100) * 100 ms
```

Simplifying:

```html
Total Time = 10,000 * 100 ms = 1,000,000 ms
```

Converting to seconds and minutes:

```html
Total Time = 1,000,000 ms = 1,000 seconds = 16.67 minutes
```

#### Key Advantages:
- **Simple Implementation:** Adding a `Semaphore` to the `mapAsync` function involves minimal changes to the codebase.
- **Effective Throttling Control:** The implementation ensures that the number of concurrent Lambda invocations does not exceed the predefined limit, maintaining system stability.

#### Trade-offs:
- **Increased Processing Time:** While throttling prevents errors, it may result in longer overall processing times due to the limitation on simultaneous executions.
- **No Guarantee:** While this approach prevents the majority of `429 Too Many Requests` errors, it does not guarantee that such errors will not occur again. This is because, even when the number of concurrent Lambdas in execution is controlled, the system might still exceed burst limits, which are governed by the token bucket algorithm.
- **Difficult to Manage in Distributed Systems:** This approach is more practical in scenarios with a single node running the application. In distributed systems with multiple nodes running the same application (e.g., 10 instances), it becomes challenging to coordinate a distributed TPS control mechanism. Each node would need to communicate and share state to ensure the total TPS remains within AWS limits, which significantly increases complexity.


### 2. **Retry with Exponential Backoff**

Retries with exponential backoff handle throttled requests effectively by spreading out retry attempts over time. This reduces the chance of overwhelming the system further when transient issues or throttling limits occur. The exponential backoff algorithm increases the delay between retries after each failed attempt, making it particularly useful in high-concurrency systems and also in services/calls that might fail at times.

#### How It Works:
The implementation retries an AWS Lambda invocation up to a specified number of attempts, introducing exponentially increasing delays between retries. For example:

```kotlin
suspend fun invokeWithRetry(record: Record, retries: Int = 3) {
    var attempts = 0
    while (attempts < retries) {
        try {
            invokeLambda(record)
            break
        } catch (e: Exception) {
            if (++attempts == retries) throw e
            delay((2.0.pow(attempts) * 100).toLong())
        }
    }
}
```

#### Estimated Time to Process

Assume:
- Each retry introduces a delay that doubles after every attempt.
- <code>D</code>: Cumulative delay for retries.
- <code>r</code>: Number of retry attempts per record.

Cumulative delay is given by:

```html
D = Σ (2^i * T_retry) for i = 1 to r
```

Where:
- <code>T_retry</code> = Base retry delay (e.g., 100 ms).

Example with <code>T_retry = 100 ms</code> and <code>r = 3</code>:

```html
D = (2^1 * 100 ms) + (2^2 * 100 ms) + (2^3 * 100 ms)
D = 200 ms + 400 ms + 800 ms = 1,400 ms
```

If 10% of records require retries, the retry time is:

```html
Retry Time = (Total Records * 10%) * D / n
Retry Time = (1,000,000 * 0.1) * 1,400 ms / 100
Retry Time = 1,400,000 ms = 1,400 seconds = 23.33 minutes
```

Adding this to the initial processing time:

```html
Total Time = Initial Time + Retry Time
Total Time = 16.67 minutes + 23.33 minutes = 40 minutes
```

---

#### Pros:
- **Handles transient errors gracefully:** Retries ensure that temporary issues, such as short-lived throttling or network disruptions, do not result in failed processing.
- **Distributed systems friendly:** Can be independently implemented in each node, avoiding the need for centralized control mechanisms.
- **Reduces system load during failures:** The increasing delay between retries prevents the system from being overwhelmed.

#### Cons:
- **Adds latency:** The exponential backoff mechanism inherently increases the time taken to complete processing, can take even BIGGER times when considering worst case scenarios(potentially 10x more the total time discussed).
- **Increases code complexity and testability:** Requires additional logic to manage retries and delays and testing those scenarios when only part of the requests fail.


### 3. **Use SQS for Decoupling**

Amazon Simple Queue Service (SQS) can act as a buffer between producers (e.g., the application processing records) and consumers (e.g., AWS Lambda), enabling controlled, asynchronous processing of requests. This approach decouples the producer and consumer, ensuring the workload is processed at a rate the system can handle.

#### How It Works:
1. The application writes each record to an SQS queue instead of invoking AWS Lambda directly.
2. AWS Lambda is configured to process messages from the queue at a controlled rate, dictated by the batch size and concurrency settings.
3. This ensures that the rate of Lambda invocations remains within the account's concurrency and TPS limits.

#### Use Case:
This approach is ideal for high-throughput systems with unpredictable traffic patterns or distributed applications with multiple producer nodes. For example, processing 1 million records stored in S3 can involve multiple workers writing to the same SQS queue, while the queue ensures smooth, consistent processing by AWS Lambda.

#### Additional Pattern: AWS Serverless Land Example

This approach aligns with a pattern presented on [AWS Serverless Land](https://serverlessland.com/patterns/sqs-lambda-ddb-sam-java): **Create a Lambda function that batch writes to DynamoDB from SQS**. This pattern deploys an SQS queue, a Lambda Function, and a DynamoDB table, allowing batch writes from SQS messages to DynamoDB. It demonstrates how to leverage a batch processing mechanism to handle high-throughput scenarios effectively.

The provided SAM template uses Java 11, SQS, Lambda, and DynamoDB to create a cost-effective, serverless architecture:

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: AWS::Serverless-2016-10-31
Description: sqs-lambda-dynamodb

Globals:
  Function:
    Runtime: java11
    MemorySize: 512
    Timeout: 25

Resources:
  OrderConsumer:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: OrderConsumer
      Handler: com.example.OrderConsumer::handleRequest
      CodeUri: target/sourceCode.zip
      Environment:
        Variables:
          QUEUE_URL: !Sub 'https://sqs.${AWS::Region}.amazonaws.com/${AWS::AccountId}/OrdersQueue'
          REGION: !Sub '${AWS::Region}'
          TABLE_NAME: !Ref OrdersTable
      Policies:
        - AWSLambdaSQSQueueExecutionRole
        - AmazonDynamoDBFullAccess

  OrdersQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: OrdersQueue

  OrdersTable:
    Type: 'AWS::DynamoDB::Table'
    Properties:
      TableName: OrdersTable
      AttributeDefinitions:
        - AttributeName: orderId
          AttributeType: S
      KeySchema:
        - AttributeName: orderId
          KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: 5
        WriteCapacityUnits: 5
```

#### Estimated Time to Process

Assume:
- <code>T_batch</code>: Execution time for processing a batch.
- <code>k</code>: Overhead due to batching.
- <code>b</code>: Number of messages per batch.
- <code>n</code>: Lambda concurrency.

The total processing time is:

```html
Total Time = (Total Records / (b * n)) * (T + k)
```

Example with:
- <code>T = 100 ms</code>
- <code>k = 20 ms</code>
- <code>b = 10</code>
- <code>n = 100</code>
- <code>Total Records = 1,000,000</code>

Substitute into the formula:

```html
Total Time = (1,000,000 / (10 * 100)) * (100 ms + 20 ms)
Total Time = (1,000,000 / 1,000) * 120 ms
Total Time = 1,000 * 120 ms = 120,000 ms
```

Convert to seconds and minutes:

```html
Total Time = 120,000 ms = 120 seconds = 2 minutes
```

#### The Importance of FIFO Queues

To maintain consistency in DynamoDB, it is essential to configure the SQS queue as FIFO (First-In, First-Out). This ensures that messages are processed in the exact order they are received, which is critical in systems where the order of operations affects the final state of the database. For example:

1. **Out-of-Order Processing Issues:** If two updates to the same DynamoDB record are processed out of order (e.g., `Update1` followed by `Update2`), but `Update2` depends on `Update1`, the database could end up in an inconsistent state. FIFO queues prevent this by enforcing strict order. For our case, there was not duplicated entries on the file so FIFO was not in considerated despite being absolutely important for this usecase.

2. **Idempotency Challenges:** Even when Lambda functions are designed to be idempotent, out-of-order processing can lead to unexpected behavior if operations rely on sequential execution. For instance, appending logs or incrementing counters requires a guarantee of order.

3. **Trade-offs with FIFO:** While FIFO queues provide consistency, they come with some limitations:
   - **Lower Throughput:** FIFO queues have a maximum throughput of 300 transactions per second with batching (or 3,000 if using high-throughput mode).
   - **Increased Latency:** Enforcing order may introduce slight delays in message processing.

Despite these trade-offs, using a FIFO queue is often the most reliable way to ensure data consistency in scenarios involving DynamoDB or similar stateful systems. When consistency is critical, the benefits of FIFO queues far outweigh the downsides.

---

#### Pros:
- **Decouples producers and consumers:** The producer can continue adding messages to the queue regardless of the Lambda processing speed.
- **Prevents throttling:** SQS regulates the rate at which messages are delivered to Lambda, avoiding sudden spikes that could exceed AWS limits.
- **Distributed systems friendly:** Works seamlessly in multi-node systems, as all nodes write to the same queue without requiring coordination.

#### Cons:
- **Adds architectural complexity:** Introducing SQS requires additional components and configuration.
- **Adds code complexity:** Introduce code complexity to the insertion lambda, so its responsible for managing sqs batch write operations, reading on SQS source and also being able to operate by asynchronous invocation for legacy systems.
- **Introduces latency:** Messages may wait in the queue before being processed, depending on the Lambda polling rate and queue depth. For example, a queue depth of 10,000 messages and a polling rate of 1,000 messages per second would result in a processing delay.

---

### Architectural Insights

For small workloads, async invocation can provide faster results, as it avoids the latency of queuing and batch processing. However, as the number of requests increases, direct invocation becomes inefficient and computationally expensive due to the high TPS demand and risk of breaching AWS limits. In contrast, decoupled architectures using SQS and batch processing scale more efficiently, ensuring stability and cost-effectiveness under heavy loads.


## Conclusion

AWS Lambda throttling issues, particularly for high-concurrency workloads, can be effectively managed using a combination of strategies such as concurrency control, retry mechanisms, and decoupling with SQS. Each of these approaches has its strengths and trade-offs:

- **Limit Concurrency Using Semaphore**: A straightforward solution for single-node setups, providing reliable throttling control at the cost of slightly increased processing time. However, it requires additional considerations for distributed systems.

- **Retry with Exponential Backoff**: A robust technique for handling transient failures, distributing load over time and avoiding unnecessary retries. Yet, it can add significant latency in worst-case scenarios and increase implementation complexity.

- **Use SQS for Decoupling**: The most scalable and efficient approach when <code>T_batch = T + k</code>, with <code>k</code> being sufficiently small. While it introduces latency and complexity, its benefits make it the go-to solution for large-scale systems.

### Next Steps: Implementing a POC

While this post has focused on explaining the challenges, strategies, and theoretical calculations for mitigation, an actual Proof of Concept (POC) is essential to validate these solutions in practice. A future post might  explore how to design and execute a POC to measure the performance, reliability, and cost implications of these approaches in a real-world scenario.

For more details on Lambda throttling, refer to the [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/invocation-scaling.html) and the [AWS Compute Blog](https://aws.amazon.com/blogs/compute/understanding-aws-lambdas-invoke-throttle-limits/).

[1]: https://aws.amazon.com/blogs/compute/understanding-aws-lambdas-invoke-throttle-limits
