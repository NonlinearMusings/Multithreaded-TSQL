This article is about _creatively_ using Service Broker to implement multi-threaded processing in SQL Server with T-SQL, and the links below provide excellent guideance and advice that we will be ignoring...

Remus Rusanu 
* [Fire and Forget: Good for the military, but not for Service Broker conversations](https://rusanu.com/2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/)
* [How to prevent conversation endpoint leaks](https://rusanu.com/2014/03/31/how-to-prevent-conversation-endpoint-leaks/)

David Wentzel
* [Service Broker Demystified Series](https://davewentzel.com/content/service-broker-demystified-series/)
  * [Service Broker Demystified - Fire and Forget Anti-Pattern](https://davewentzel.com/content/service-broker-demystified-fire-and-forget-anti-pattern/)
  * [Service Broker Demystified - CLOSED conversations](https://davewentzel.com/content/service-broker-demystified-closed-conversations/)
  * [Service Broker Demystified - Can I model monologs? Yes you can!](https://davewentzel.com/content/service-broker-demystified-can-i-model-monologs-yes-you-can/)

Klaus Aschenbrenner
* [Service Broker Part 2: Why Service Broker](https://www.sqlservercentral.com/articles/service-broker-part-2-why-service-broker#:~:text=Reusing%20conversations%20has%20a%20significant%20positive%20performance%20impact.,receivers%20side%20by%20a%20factor%20of%20about%2010)

Why? 

Because SQL Server Agent Jobs run synchronously and sometimes we need to run multiple tasks asynchronously, in parallel, natively in SQL Server, without any external dependencies. Enter Service Broker!

Almost... 

Service Broker is almost exactly what we need except that it was designed for two-way communication (across servers even) and all we need is a simple, same server, one-way message queue using a Fire-and-Forget pattern that both Remus and David are adamantly against - and rightly so!

What's the problem?

Simply stated, Service Broker is designed with the expectation that receivers end conversations and since the Fire-and-Forget pattern does not follow that design resource leaks ensue in the form of endpoints that will not be deallocated until today + 2147483647 seconds, a mere 68 years, after their creation! Plus, as jobs are not Service Broker aware the idea of making them so is likely a deal breaker.

[Never give up! Never surrender!](https://youtu.be/SJ2hJezvd2I?t=23)

Cards on the table:
* Messages are atomic
* Messages are processed in the order received (FIFO)
* Messages ultimately invoke a stored procedure ([Claim-Check pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check) recommended)

### Option #1: Windowed-Fire-and-Forget
This pattern works well for recurring batches. Such as maintenance jobs, data refreshes, and ETL activities, to name a few. The "trick" is to determine how long the sender needs to sta

### Option #2: Keep-Firing!
