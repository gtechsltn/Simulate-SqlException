# Simulate some SqlExceptions for Testing

## 1. Ref and Out keyword
```
public void ProcessValues(int input, ref int refVal, out int outVal)
{
    refVal += input;
    outVal = refVal * 2;
}

int input = 5;
int refVal = 10;
int outVal;

ProcessValues(input, ref refVal, out outVal);

Console.WriteLine($"refVal: {refVal}, outVal: {outVal}");

```

### Ref and Out keyword: ERROR
```
// ❌ Missing 'ref' or 'out' keyword
ProcessValues(input, refVal, outVal); // ERROR

// ❌ Wrong order
ProcessValues(ref refVal, input, out outVal); // ERROR
```

## 2. Create Fake SqlException with Error Number -2 (Timeout)
```

using System;
using System.Data.SqlClient;

class Program
{
    static void Main()
    {
        var connectionString = "Server=(local);Database=UserDb;Trusted_Connection=True;";
        using (var connection = new SqlConnection(connectionString))
        {
            var command = new SqlCommand("WAITFOR DELAY '00:01:00';", connection);
            command.CommandTimeout = 5; // Set timeout to 5 seconds
            try
            {
                connection.Open();
                command.ExecuteNonQuery();
            }
            catch (SqlException ex)
            {
                Console.WriteLine($"Exception Number: {ex.Number}");
                Console.WriteLine($"Message: {ex.Message}");
            }
        }
    }
}


using System;
using System.Data.SqlClient;
using System.Reflection;

public static class SqlExceptionCreator
{
    public static SqlException CreateSqlTimeoutException()
    {
        // Create SqlError instance
        var errorConstructor = typeof(SqlError).GetConstructor(
            BindingFlags.Instance | BindingFlags.NonPublic,
            null,
            new[] { typeof(int), typeof(byte), typeof(byte), typeof(string),
                    typeof(string), typeof(string), typeof(int), typeof(uint) },
            null);

        var sqlError = errorConstructor.Invoke(new object[] {
            -2, (byte)0, (byte)0, "server", "Timeout expired", "proc", 0, (uint)0
        });

        // Create SqlErrorCollection and add error
        var errorCollection = Activator.CreateInstance(
            typeof(SqlErrorCollection),
            true);

        typeof(SqlErrorCollection)
            .GetMethod("Add", BindingFlags.Instance | BindingFlags.NonPublic)
            .Invoke(errorCollection, new[] { sqlError });

        // Create SqlException
        var exceptionConstructor = typeof(SqlException).GetConstructor(
            BindingFlags.Instance | BindingFlags.NonPublic,
            null,
            new[] { typeof(string), typeof(SqlErrorCollection), typeof(Exception), typeof(Guid) },
            null);

        return (SqlException)exceptionConstructor.Invoke(new object[] {
            "A timeout occurred.", errorCollection, null, Guid.NewGuid()
        });
    }
}
```

### Usage
```
try
{
    throw SqlExceptionCreator.CreateSqlTimeoutException();
}
catch (SqlException ex)
{
    Console.WriteLine($"Caught SqlException - Number: {ex.Number}, Message: {ex.Message}");
}
```

### Alternative: Abstract your DB access logic
```
mockRepo.Setup(r => r.GetUser(It.IsAny<int>()))
        .Throws(SqlExceptionCreator.CreateSqlTimeoutException());
```

### ✅ Scenario
We’ll create:

An interface IUserRepository

A service UserService that uses the repository

A test UserServiceTests that simulates a SQL timeout using Moq and our custom SqlException

```
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Moq

/TestProject
├── IUserRepository.cs
├── UserService.cs
├── SqlExceptionCreator.cs
└── UserServiceTests.cs

public interface IUserRepository
{
    string GetUserById(int id);
}
public class UserService
{
    private readonly IUserRepository _repo;

    public UserService(IUserRepository repo)
    {
        _repo = repo;
    }

    public string GetUser(int id)
    {
        try
        {
            return _repo.GetUserById(id);
        }
        catch (SqlException ex) when (ex.Number == -2)
        {
            // Retry logic or specific handling
            return "Timeout occurred!";
        }
    }
}

using System;
using System.Data.SqlClient;
using System.Reflection;

public static class SqlExceptionCreator
{
    public static SqlException CreateSqlTimeoutException()
    {
        var errorConstructor = typeof(SqlError).GetConstructor(
            BindingFlags.Instance | BindingFlags.NonPublic,
            null,
            new[] { typeof(int), typeof(byte), typeof(byte), typeof(string),
                    typeof(string), typeof(string), typeof(int), typeof(uint) },
            null);

        var sqlError = errorConstructor.Invoke(new object[] {
            -2, (byte)0, (byte)0, "server", "Timeout expired", "proc", 0, (uint)0
        });

        var errorCollection = Activator.CreateInstance(
            typeof(SqlErrorCollection),
            true);

        typeof(SqlErrorCollection)
            .GetMethod("Add", BindingFlags.Instance | BindingFlags.NonPublic)
            .Invoke(errorCollection, new[] { sqlError });

        var exceptionConstructor = typeof(SqlException).GetConstructor(
            BindingFlags.Instance | BindingFlags.NonPublic,
            null,
            new[] { typeof(string), typeof(SqlErrorCollection), typeof(Exception), typeof(Guid) },
            null);

        return (SqlException)exceptionConstructor.Invoke(new object[] {
            "A timeout occurred.", errorCollection, null, Guid.NewGuid()
        });
    }
}

using Moq;
using Xunit;

public class UserServiceTests
{
    [Fact]
    public void GetUser_ShouldHandleSqlTimeoutException()
    {
        // Arrange
        var mockRepo = new Mock<IUserRepository>();
        var timeoutEx = SqlExceptionCreator.CreateSqlTimeoutException();

        mockRepo.Setup(r => r.GetUserById(It.IsAny<int>()))
                .Throws(timeoutEx);

        var service = new UserService(mockRepo.Object);

        // Act
        var result = service.GetUser(1);

        // Assert
        Assert.Equal("Timeout occurred!", result);
    }
}

dotnet test
```

## 3. Simulate a Transaction Timeout
```
BEGIN TRANSACTION;
UPDATE Users SET Name = 'Blocked' WHERE Id = 1;
-- DO NOT COMMIT

SET LOCK_TIMEOUT 5000;  -- Wait 5 seconds max for a lock
UPDATE Users SET Name = 'Trying to update' WHERE Id = 1;
```

```
public static SqlException CreateSqlException(int errorNumber, string message)
{
    var errorConstructor = typeof(SqlError).GetConstructor(
        BindingFlags.Instance | BindingFlags.NonPublic,
        null,
        new[] { typeof(int), typeof(byte), typeof(byte), typeof(string),
                typeof(string), typeof(string), typeof(int), typeof(uint) },
        null);

    var sqlError = errorConstructor.Invoke(new object[] {
        errorNumber, (byte)0, (byte)0, "server", message, "proc", 0, (uint)0
    });

    var errorCollection = Activator.CreateInstance(typeof(SqlErrorCollection), true);
    typeof(SqlErrorCollection).GetMethod("Add", BindingFlags.Instance | BindingFlags.NonPublic)
        .Invoke(errorCollection, new[] { sqlError });

    var exceptionConstructor = typeof(SqlException).GetConstructor(
        BindingFlags.Instance | BindingFlags.NonPublic,
        null,
        new[] { typeof(string), typeof(SqlErrorCollection), typeof(Exception), typeof(Guid) },
        null);

    return (SqlException)exceptionConstructor.Invoke(new object[] {
        message, errorCollection, null, Guid.NewGuid()
    });
}
```

### Usage 1:
```
var timeoutEx = SqlExceptionCreator.CreateSqlException(1222, "Lock request timeout exceeded");
throw timeoutEx;
```

### Usage 2:
```
mockRepo.Setup(r => r.SaveUser(It.IsAny<User>()))
        .Throws(SqlExceptionCreator.CreateSqlException(1222, "Lock request timeout exceeded"));
```

Common Timeout-Related SqlException | Numbers
--|--
Error Number | Meaning
-2 | Command timeout
1222 | Lock request timeout
3960 | Snapshot isolation conflict
1205 | Deadlock victim

## 4. Simulate a Deadlock

```
try
{
    // run both transactions in separate tasks to simulate deadlock
}
catch (SqlException ex) when (ex.Number == 1205)
{
    Console.WriteLine("Deadlock detected: " + ex.Message);
}

public static SqlException CreateSqlDeadlockException()
{
    return SqlExceptionCreator.CreateSqlException(1205, "Transaction (Process ID 70) was deadlocked on resources...");
}

mockRepo.Setup(r => r.Save(It.IsAny<User>()))
        .Throws(SqlExceptionCreator.CreateSqlException(1205, "Transaction was deadlocked"));
```

## 5. Simulate a TransactionScope Timeout
```
using System;
using System.Transactions;
using System.Threading;
using System.Data.SqlClient;

class Program
{
    static void Main()
    {
        try
        {
            var options = new TransactionOptions
            {
                Timeout = TimeSpan.FromSeconds(3) // force timeout quickly
            };

            using (var scope = new TransactionScope(TransactionScopeOption.Required, options))
            {
                // Simulate long-running operation
                Console.WriteLine("Starting transaction...");
                Thread.Sleep(5000); // longer than timeout

                // Any database logic here
                using (var conn = new SqlConnection("YourConnectionStringHere"))
                {
                    conn.Open();
                    // Simple query (will be rolled back)
                    new SqlCommand("SELECT 1", conn).ExecuteNonQuery();
                }

                scope.Complete();
            }
        }
        catch (TransactionAbortedException ex)
        {
            Console.WriteLine("TransactionScope timeout occurred:");
            Console.WriteLine(ex.Message);
        }
    }
}
```

### or

```
mockService.Setup(s => s.Save())
           .Throws(new TransactionAbortedException("Simulated transaction timeout."));
```

### or

```
try
{
    throw new TransactionAbortedException("Simulated transaction timeout.");
}
catch (TransactionAbortedException ex)
{
    Console.WriteLine("Handled simulated timeout: " + ex.Message);
}
```
