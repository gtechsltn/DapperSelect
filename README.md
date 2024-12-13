# Dapper Select: Dapper Get SQL Select To Dynamic Type

## Nuget Packages
```
Install-Package log4net
Install-Package Newtonsoft.Json
Install-Package Dapper.Contrib
Install-Package Humanier
Install-Package NodaTime
Install-Package CsvHelper
Install-Package Refit
```

## Connection String for .NET Framework
```
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <connectionStrings>
    <add name="DefaultConnection" connectionString="Data Source=.;Initial Catalog=mssql;Integrated Security=SSPI;Connect Timeout=30;Pooling=True;Max Pool Size=10;" />
  </connectionStrings>
</configuration>
```

## Connection String for .NET Core
```
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <connectionStrings>
    <add name="DefaultConnection" connectionString="Data Source=.;Initial Catalog=mssql;Integrated Security=SSPI;Connect Timeout=30;Pooling=True;Max Pool Size=10;Encrypt=True;TrustServerCertificate=True;MultipleActiveResultSets=true;" />
  </connectionStrings>
</configuration>
```

## Log4Net Configuration
```
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <configSections>
    <section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net" />
  </configSections>
  <system.diagnostics>
    <trace autoflush="true">
      <listeners>
        <add name="textWriterTraceListener" type="System.Diagnostics.TextWriterTraceListener" initializeData="logs\log4net.trace.txt" />
      </listeners>
    </trace>
  </system.diagnostics>
  <log4net>
    <appender name="ConsoleAppender" type="log4net.Appender.ConsoleAppender">
      <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="%date [%thread] %-5level %logger [%ndc] &lt;%property{auth}&gt; - %message%newline" />
      </layout>
    </appender>
    <appender name="DailyRollingFileAppender" type="log4net.Appender.RollingFileAppender">
      <threshold value="ALL" />
      <file value="logs\traceroll.day.log" />
      <appendToFile value="true" />
      <rollingStyle value="Composite" />
      <datePattern value="yyyyMMdd" />
      <maximumFileSize value="10MB" />
      <maxSizeRollBackups value="-1" />
      <CountDirection value="1" />
      <preserveLogFileNameExtension value="true" />
      <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="[${COMPUTERNAME}] %d{ISO8601} %6r %-5p [%t] %c{2}.%M() - %m%n" />
      </layout>
    </appender>
    <root>
      <!-- ALL, DEBUG, INFO, WARN, ERROR, FATAL, OFF -->
      <level value="INFO" />
      <appender-ref ref="ConsoleAppender" />
      <appender-ref ref="DailyRollingFileAppender" />
    </root>
  </log4net>
</configuration>
```

## Program.cs
```
class Program
{
  static readonly ILog _log = LogManager.GetLogger(MethodBase.GetCurrentMethod().DeclaringType);
  static readonly string _connString = ConfigurationManager.ConnectionStrings["DefaultConnection"].ConnectionString;

  private static void Main(string[] args)
  {
    log4net.Config.XmlConfigurator.Configure();
    ...
    Console.Write("Press any key to exit...");
    Console.ReadKey();
  }
}
```

## MS SQL Server: Read dynamic

```
using var cn = new SqlConnection(connectionString);
 
IEnumerable<dynamic> customers = cn.Query("SELECT TOP 10 * FROM CUSTOMER");
 
foreach (dynamic customer in customers)
{
    WriteLine($"{customer.FirstName} {customer.SecondName} {customer.Height} {customer.Age}");
}
```

## MS SQL Server: Read and Write
```
using System;
using System.Data.SqlClient;

public static class DataWriterExtensions
{
    public static object OrDBNull(this object someVar)
    {
        return someVar ?? DBNull.Value;
    }
}

public static class DataReaderExtensions
{
    public static T SafeGet<T>(this SqlDataReader reader, string colName)
    {
        var colIndex = reader.GetOrdinal(colName);
        return reader.IsDBNull(colIndex) ? default(T) : reader.GetFieldValue<T>(colIndex);
    }
    public static ActiveFilter ToActiveFilter(this SqlDataReader rdr)
    {
        return new ActiveFilter()
        {
            ID = SafeGet<Int64>(rdr, nameof(ActiveFilter.ID)),
            //TODO: Add more column here ...
        }
    }
}
```

## CRUD Samples
```
using System;
using System.Collections.Generic;
using System.Data;
using System.Data.SqlClient;
...
private static readonly log4net.ILog _log = log4net.LogManager.GetLogger(typeof(UserSettingDataAccess));
...

public Int64 Create(ActiveFilter activefilter)
{
    Int64 ID = default(Int64);
    try
    {
        using (var cnn = new SqlConnection(_connString))
        {
            cnn.Open();

            using (var cmd = cnn.CreateCommand())
            {
                cmd.CommandText = "activefilter_Insert";
                cmd.CommandType = CommandType.StoredProcedure;

                cmd.Parameters.Add(new SqlParameter("@AFName", activefilter.AFName.OrDBNull()));

                var objParamID = new SqlParameter("@ID", SqlDbType.BigInt);
                objParamID.Direction = ParameterDirection.Output;
                cmd.Parameters.Add(objParamID);

                var rowsAffected = cmd.ExecuteNonQuery();

                Int64? keyValue = objParamID.Value as Int64?;
                if (keyValue.HasValue)
                {
                    ID = keyValue.Value;
                    activefilter.ID = ID;
                }
            }

            cnn.Close();
        }

        return ID;
    }
    catch (Exception ex)
    {
        _log.Error(ex);
        throw;
    }
}
```

## Dapper Transaction
```
using System;
using System.Data.SqlClient;
using Dapper;

class Program
{
    static void Main(string[] args)
    {
        string connectionString = "YourConnectionStringHere"; // Update with your connection string

        using (var connection = new SqlConnection(connectionString))
        {
            connection.Open();

            using (var transaction = connection.BeginTransaction())
            {
                try
                {
                    // First Query
                    string insertQuery1 = "INSERT INTO Users (Name, Age) VALUES (@Name, @Age)";
                    connection.Execute(insertQuery1, new { Name = "John Doe", Age = 30 }, transaction);

                    // Second Query
                    string insertQuery2 = "INSERT INTO Users (Name, Age) VALUES (@Name, @Age)";
                    connection.Execute(insertQuery2, new { Name = "Jane Doe", Age = 25 }, transaction);

                    // Commit transaction
                    transaction.Commit();
                    Console.WriteLine("Transaction committed successfully.");
                }
                catch (Exception ex)
                {
                    // Rollback transaction
                    transaction.Rollback();
                    Console.WriteLine($"Transaction rolled back. Error: {ex.Message}");
                }
            }
        }
    }
}
```

## Stored Procedure
```
CREATE PROCEDURE GetUserById
    @Id INT
AS
BEGIN
    SELECT * FROM Users WHERE Id = @Id
END
```

## Dapper vs Stored Procedure
```
using System;
using System.Data;
using System.Data.SqlClient;
using Dapper;

class Program
{
    public class User
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public int Age { get; set; }
    }

    static void Main(string[] args)
    {
        string connectionString = "YourConnectionStringHere"; // Update with your connection string

        using (var connection = new SqlConnection(connectionString))
        {
            connection.Open();

            // Define the parameter for the stored procedure
            var parameters = new { Id = 1 }; // Change the ID as needed

            // Call the stored procedure
            var user = connection.QuerySingleOrDefault<User>("GetUserById", parameters, commandType: CommandType.StoredProcedure);

            if (user != null)
            {
                Console.WriteLine($"User Found: {user.Name}, Age: {user.Age}");
            }
            else
            {
                Console.WriteLine("User not found.");
            }
        }
    }
}
```

## Dapper TransactionScope MultipleActiveResultSets
```
using System;
using System.Data.SqlClient;
using System.Transactions;
using Dapper;

class Program
{
    public class User
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public int Age { get; set; }
    }

    static void Main(string[] args)
    {
        string connectionString = "Server=YourServer;Database=YourDatabase;User Id=YourUsername;Password=YourPassword;MultipleActiveResultSets=True;";

        using (var transactionScope = new TransactionScope())
        {
            using (var connection = new SqlConnection(connectionString))
            {
                connection.Open();

                // Example: Insert a new user
                var newUser = new User { Name = "Alice Smith", Age = 28 };
                string insertQuery = "INSERT INTO Users (Name, Age) VALUES (@Name, @Age)";
                connection.Execute(insertQuery, newUser);

                // Example: Retrieve users
                string selectQuery = "SELECT * FROM Users";
                var users = connection.Query<User>(selectQuery);

                Console.WriteLine("Users in Database:");
                foreach (var user in users)
                {
                    Console.WriteLine($"Id: {user.Id}, Name: {user.Name}, Age: {user.Age}");
                }

                // Complete the transaction
                transactionScope.Complete();
            }
        }
    }
}
```
## Dapper vs MSTest
```
Install-Package Dapper
Install-Package MSTest.TestFramework
Install-Package MSTest.TestAdapter
```

# Dapper Service class
```
using System.Data.SqlClient;
using Dapper;

public class UserRepository
{
    private readonly string _connectionString;

    public UserRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    public User GetUserById(int id)
    {
        using (var connection = new SqlConnection(_connectionString))
        {
            connection.Open();
            return connection.QuerySingleOrDefault<User>("SELECT * FROM Users WHERE Id = @Id", new { Id = id });
        }
    }
}

public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
}
```

## Unit Testing with MSTest
```
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class UserRepositoryTests
{
    private UserRepository _userRepository;
    private string _connectionString = "YourConnectionStringHere"; // Update with your connection string

    [TestInitialize]
    public void Setup()
    {
        _userRepository = new UserRepository(_connectionString);
    }

    [TestMethod]
    public void GetUserById_ValidId_ReturnsUser()
    {
        // Arrange
        int userId = 1; // Assuming a user with ID 1 exists in the database
        
        // Act
        var user = _userRepository.GetUserById(userId);

        // Assert
        Assert.IsNotNull(user);
        Assert.AreEqual(userId, user.Id);
    }

    [TestMethod]
    public void GetUserById_InvalidId_ReturnsNull()
    {
        // Arrange
        int userId = 999; // Assuming no user with ID 999 exists in the database
        
        // Act
        var user = _userRepository.GetUserById(userId);

        // Assert
        Assert.IsNull(user);
    }
}
```

## Integration Testing
```
Install-Package Dapper
Install-Package MSTest.TestFramework
Install-Package MSTest.TestAdapter
Install-Package Microsoft.Data.SqlClient (or System.Data.SqlClient)
```

```
using System.Data.SqlClient;
using Dapper;

public class UserRepository
{
    private readonly string _connectionString;

    public UserRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    public User GetUserById(int id)
    {
        using (var connection = new SqlConnection(_connectionString))
        {
            connection.Open();
            return connection.QuerySingleOrDefault<User>("SELECT * FROM Users WHERE Id = @Id", new { Id = id });
        }
    }

    public void AddUser(User user)
    {
        using (var connection = new SqlConnection(_connectionString))
        {
            connection.Open();
            string insertQuery = "INSERT INTO Users (Name, Age) VALUES (@Name, @Age)";
            connection.Execute(insertQuery, user);
        }
    }
}

public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
}
```

```
using Microsoft.VisualStudio.TestTools.UnitTesting;
using System;

[TestClass]
public class UserRepositoryIntegrationTests
{
    private UserRepository _userRepository;
    private string _connectionString = "YourConnectionStringHere"; // Update with your connection string

    [TestInitialize]
    public void Setup()
    {
        _userRepository = new UserRepository(_connectionString);

        // Clean up the Users table before each test
        using (var connection = new SqlConnection(_connectionString))
        {
            connection.Open();
            connection.Execute("DELETE FROM Users");
        }
    }

    [TestMethod]
    public void AddUser_UserIsAddedSuccessfully()
    {
        // Arrange
        var newUser = new User { Name = "Alice Smith", Age = 28 };

        // Act
        _userRepository.AddUser(newUser);
        var retrievedUser = _userRepository.GetUserById(newUser.Id);

        // Assert
        Assert.IsNotNull(retrievedUser);
        Assert.AreEqual(newUser.Name, retrievedUser.Name);
        Assert.AreEqual(newUser.Age, retrievedUser.Age);
    }

    [TestMethod]
    public void GetUserById_UserDoesNotExist_ReturnsNull()
    {
        // Arrange
        int nonExistentUserId = 999; // Assuming this user ID doesn't exist

        // Act
        var user = _userRepository.GetUserById(nonExistentUserId);

        // Assert
        Assert.IsNull(user);
    }
}
```
