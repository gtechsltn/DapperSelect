# Dapper Select: Dapper Get SQL Select To Dynamic Type

```
using var cn = new SqlConnection(connectionString);
 
IEnumerable<dynamic> customers = cn.Query("SELECT TOP 10 * FROM CUSTOMER");
 
foreach (dynamic customer in customers)
{
    WriteLine($"{customer.FirstName} {customer.SecondName} {customer.Height} {customer.Age}");
}
```
