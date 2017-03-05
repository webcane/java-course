Transactions & AOP
-----------------

By now your transaction management looks pretty cumbersome, should be something like this:

```
connection.setAutoCommit(false);
try {
  //some SQL related logic
  connection.commit();
} catch(Throwable t) {
  connection.rollback();
} finally {
  closeConnection(connection);
}
```

This is called Programmatic Transaction Management. Fortunately there are ways to make it look shorter and nicer. 

# Step 1 - Introducing Service Facade

While DAO stores the logic to retrieve and write data to the storage, one business transaction can span multiple
storage related operations. E.g. if our Dog had an Owner and we were to write information both to `OWNER` and `DOG`
tables we would've used `DogDao` and `OwnerDao` for that. But that would mean that our transactions cannot run on DAO 
level - we need to begin them somewhere higher to be able to rollback both SQL operations as one transaction.
 
We could do that in Controllers but they already are responsible to handle HTTP-related logic - we don't want to 
overload them with too much of unrelated code. So we need to introduce a new concept - Service Facade 
(Service for short). Facade is a GoF design pattern - it's an object that doesn't have heavy logic of its own but 
instead it orchestrates calls to other classes and methods. By doing that it joins together a lot of complicated logic 
forming a single workflow that's invoked altogether by the client code. In enterprise apps it usually invokes multiple 
DAOs, sends Emails, initiates its own requests to other systems. But in our tiny app it will just invoke DAO methods. 

- Create a class `DogService`, inject DAO into that class
- Add all the CRUD methods that you have in DAO and instead of having DAO in the Controller - inject this new service 
there. Now you have additional layer in the app. All the tests must keep passing.
- Think: now you need to move transaction-related logic to the services. That means that both Service and DAO would 
need to reference the same Connection in their methods. So should we begin a transaction in Service and then pass that
connection to the DAO methods? Wouldn't such API be ugly? Now we need to pass Connection to all the methods in DAO.
Could you come up with a better way to share the connection?
- Read & research: what's a `ThreadLocal`? Create your own ThreadLocal variables for fun before continuing. Look 
through examples on the Internet.
- Create class `JdbcConnectionHolder` and add methods to get thread-local connections. Inject it to DAO and use it
instead of directly working with DataSource.
- Now you're ready to move the transaction-related logic into the service. Inject your `JdbcConnectionHolder` to your
service and migrate all the transaction-related logic there as well. See how you can work with the same connection per
thread and not pass it from the service to the DAO? This is a common way of creating a global state per thread and
share it between many layers in the app.

# Step 2 - Proxy

Proxy is a general term for something that takes control and just passes it to other object. In programming there is
a GoF's Proxy Design Pattern that does just that. If I were to name single most important pattern to know - it would be
Proxy. 

- Explore Proxy Design Pattern. Implement some examples for fun before continuing.
- Create a Proxy for your `DogService`, call it `TransactionalDogService`. Make sure that this is what's used
in the Controller from now on.
- Migrate all the transaction-related logic to your proxy keeping your old DogService small and clean.

See how simple the service became now? This helps a lot since services often have a lot of logic - cluttering it with
try-catch-finally won't help with readability.

# Step 3 - DynamicProxy