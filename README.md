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
