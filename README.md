# Need a SQL Server native way to execute multiple jobs in parallel?

SQL Server Agent runs jobs synchronously and sometimes you need asynchronous, parallel execution. I am going to show you two ways to  _creatively_ use Service Broker to implement multi-threaded processing in SQL Server with T-SQL!

## Creatively?

Service Broker is designed with the expectation that senders and receivers have two-way conversations and **receivers** end those conversations. However, one of the most useful implementations of Service Broker is as a message queue (think MSMQ) whereby a sender enqueues messages and receivers dequeue and execute, which is the Fire-and-Forget pattern. The most signifcant problem with the Fire-and-Forget pattern, that is of concern when implementing a message queue, is that it leaks conversation endpoints. (Don't worry, they only last 68 years...)

The other issues with Fire-and-Forget, as covered in the Appendix links below, are not of concern because:
* this implementation does not have a remote reciever whose configuration may change.
* we do not need the receiver to notifiy us of errors, because we're responsible people who log our job's progress... right? (Of course, right.)
* any errors encountered dequeuing messages are in sys.transmission_queue, and once resolved you're good-to-go.

## [Never give up! Never surrender!](https://youtu.be/SJ2hJezvd2I?t=23)

First, a few pointers:
* Messages reflect ACID activities. (This thought will help you discern parallelizable activities and identify any dependencies your stored procedures need to account for)
* Messages are delivered exactly once.
* Messages are processed in the order received (The queue is FIFO).
* Messages ultimately invoke a Service Broker internal activation stored procedure. Using a [Claim-Check pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check) for the messages, only the information needed to execute a task is enqueued. Implementing a [Facade pattern](https://en.wikipedia.org/wiki/Facade_pattern) in the activation stored procedure guards against poison messages will providing virutally unlimited processing flexibility.

## Common configuration bits...
```sql
-- enable Service Broker
alter database WayCoolSql set enable_broker with rollback immediate;

-- create a message type (while not strictly required, I'm enforcing XML)
create message type TaskingEvent
    authorization dbo
    validation = well_formed_xml;

-- create a messaging contract
create contract TaskingContract
    ( TaskingEvent sent by initiator );

-- create a Service Broker queue
create queue TaskingQueue;

--- create a Service Broker service
create service TaskingService
    authorization [dbo]
    on queue TaskingQueue ( TaskingContract );

-- assign an internal activation stored procedure and set the desired concurrency level
-- in an initially disabled state
alter queue TaskingQueue with activation
    (    status = off
    ,    procedure_name = dbo.ActivateTask
    ,    max_queue_readers = 4
    ,    execute as owner
    );

-- create a certificate for the activation stored procedure
create certificate TaskExecutorCertificate
	encryption by password = 'Ta5k#3x3cut3!'
	with subject = 'Certificate for Service Broker Activation Stored Procedure';

```

### Option #1: LimitedLifetime-Fire-and-Forget
This pattern is self-cleaning and addresses conversation endpoint leaks by:
* Setting the [BEGIN DIALOG CONVERSATION](https://learn.microsoft.com/en-us/sql/t-sql/statements/begin-dialog-conversation-transact-sql?view=sql-server-ver16)'s LIFETIME parameter to a value, in seconds, that allows your enqueuing process adequate time to complete. This is the 'Fire' part of the pattern as a dialog will be established with the reciever upon executing a subsequent SEND ON CONVERSATION statement. When the dialog's lifetime expires its sender or "initiator" conversation endpoint will be reclaimed thereby closing its side of the dialog. This is the 'Forget' part of the pattern as the sender is no longer able to communicate with the reciever.
* Adding a check in the activation stored procedure to see if the queue is empty or not. If empty then executing END CONVERSATION it will cause the receiver's conversation endpoint to be reclaimed in 30 minutes.

#### Sender/Initiator
Create a dialog with a 60 second lifetime and send a single message.
```sql
declare  @handle    uniqueidentifier
    ,    @msg       nvarchar(255);

set @msg = N'<xml>test message</xml>';

begin transaction;

begin dialog conversation @handle
	from service TaskingService
	to service N'TaskingService'
	on contract TaskingContract
	with
        lifetime    = 60        -- seconds
    ,   encryption  = off;

send on conversation @handle
	message type TaskingEvent
	(@msg);

commit transaction;
```
#### Receiver
Create an activation stored procedure that receives messages, takes action and ends the conversation when the queue is empty.
```sql
create procedure dbo.ActivateTask
as
begin
    set nocount on;
    
    declare  @handle     uniqueidentifier
        ,    @msg        nvarchar(max)
        ,    @msgName    sysname;
    
    begin transaction;
    
    receive	top(1)
        @handle = conversation_handle
    ,	@msg	= message_body
    ,	@msgName= message_type_name
    from TaskingQueue
    
    if @msgName = N'TaskingEvent'
    begin
        -- do work
    end;
    
    commit transaction;
    
    -- end the conversation when the queue is empty
    if not exists( select top 1 1 from dbo.TaskingQueue )
    begin
        end conversation @handle;
    end;

end;
```
### Option #2: ExtendedLifetime-Fire-and-Forget
This pattern establishes a persistent, reusable conversation that must be explictly terminated by a secondary process to reclaim its conversation endpoints.

#### First
Create a table to reference our conversation and its end points.
```sql
create table dbo.TaskingSession
    (    conversationId    uniqueidentifier    null
    ,    senderHandle      uniqueidentifier    not null
    ,    receiverHandle    uniqueidentifier    null
    );
```
#### Sender/Initiator
Create a dialog with a default lifetime and send a single message, adding code to reuse the existing conversation during subsequent runs
```sql
declare @handle           uniqueidentifier
    ,    @msg             nvarchar(255)
    ,    @isNewSession    bit = 0;

set @msg = N'<xml>newer message</xml>';

begin transaction;

-- check for existing conversation session
select @handle = ( select senderHandle from dbo.TaskingSession );

-- create a new conversation, if necessary
if @handle is null
begin
	set @isNewSession = 1;

	begin dialog conversation @handle
		from service TaskingService
		to service N'TaskingService'
		on contract TaskingContract
		with encryption	= off;

    -- record the sender's conversation handle
    insert dbo.TaskingSession ( senderHandle ) values ( @handle );
end;

send on conversation @handle
	message type TaskingEvent
	(@msg);

-- record the conversation's id and the receiver's conversation handle
if @isNewSession = 1
begin
	update	ts
		set	conversationId = se.conversation_id
		,	receiverHandle = te.conversation_handle
	from	dbo.TaskingSession				as ts
	inner	join sys.conversation_endpoints	as se	-- sender endpoint
		on	conversation_handle = @handle
		and	se.is_initiator = 1
	inner	join sys.conversation_endpoints as te	-- target endpoint
		on	se.conversation_id = te.conversation_id
		and te.is_initiator = 0;
end;

commit transaction;
```
#### Receiver
Create an activation stored procedure that receives messages and takes action.
```sql
create procedure dbo.ActivateTask
as
begin
    set nocount on;
    
    declare  @handle     uniqueidentifier
        ,    @msg        nvarchar(max)
        ,    @msgName    sysname;
    
    begin transaction;
    
    receive	top(1)
        @handle = conversation_handle
    ,	@msg	= message_body
    ,	@msgName= message_type_name
    from TaskingQueue
    
    if @msgName = N'TaskingEvent'
    begin
        -- do work
    end;
    
    commit transaction;

end;
```
#### Cleanup?
Should the need arise to shut-down, clean-up or otherwise dispose of the ExtenedLifetime's objects then simply:
```sql
-- brute force clean-up all Service Broker conversation endpoints
declare @handle uniqueidentifier;

while exists( select 1 from sys.conversation_endpoints )
begin
	set @handle = ( select top 1 conversation_handle from sys.conversation_endpoints );
	end conversation @handle with cleanup;
end;

-- brute force clean-up session tracking
truncate table dbo.TaskingSession;
```
### Conclusion
And there you have it! Two ways to implement a message queue with Service Broker without leaking resources.

Way Cool, huh?

### Appendix
(of the naysayers ;P)

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

