---
title:  "Entity Framework Core Owned Entity"
date:   2024-02-15
categories:
  - dotnet
  - ef-core
tags:
  - .NET 8
  - entity framework
  - dapper
---
## Owned Entity Types
As defined on [Owned Entity Types](https://learn.microsoft.com/en-us/ef/core/modeling/owned-entities)
> EF Core allows you to model entity types that can only ever appear on navigation properties of other entity types. These are called owned entity types. The entity containing an owned entity type is its owner.

If we talk in terms Domain Driven Design, the entity containing owned entity equates to the aggregate root and owned entity would become an aggregate.

Also owned entity is a good candidate to model values objects in Domain Driven Design. Value objects don't have a surrogate identity instead all fields make up the object identity.

> An aggregate will have one of its component objects be the aggregate root. Any references from outside the aggregate should only go to the aggregate root. The root can thus ensure the integrity of the aggregate as a whole.

Based on the rule above EF Core has following limitations
* A `DbSet<T>` of owned entity cannot be created.
* You cannot call `Entity<T>()` with an owned type on `ModelBuilder`.

## Domain
For the purpose of this post, we will create a `Customer` service, that will be responsible for CRUD operations on `Customer`. I will only implement the `Create` and `Read` operation for simplicity.

Our `Customer` object (root aggregate) will look like following
```json
{
    "id": 1,
    "firstName": "John",
    "lastName": "Doe",
    "BillingAddress": {
        "line1": "10 Downing Street",
        "line2": null,
        "line3": null,
        "line4": null,
        "city": "London",
        "postCode": "SW1A 2AA",
        "country": "United Kingdom"
    },
    "ShippingAddress": {
        "line1": "10 Downing Street",
        "line2": null,
        "line3": null,
        "line4": null,
        "city": "London",
        "postCode": "SW1A 2AA",
        "country": "United Kingdom"
    }
}
```

As you can see we have address hanging off our root object `Customer` and we only want to query those address starting from `Customer` so the address makes a good candidate as a value object or in terms of implementation an `EF Core Owned Entity`. Moreover we have 2 addresses so we can add 2 instances of the same owned entity to our root aggregate.

## Project Setup
Let's start by creating a new empty folder and create a new solution using following command.
```shell
dotnet new sln --name Customers
```

Let's add a webapi project and add it to solution.
```shell
dotnet new webapi --use-controllers -o Customers.Api
dotnet sln add Customers.Api/Customers.Api.csproj
```

I will use SQLite for simplicity, let's add a reference to `Microsoft.EntityFrameworkCore.Sqlite` using following command in project dir.
```shell
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
```

## Database Setup
I will use `FluentMigrator` to initialise database on startup. Let's start by adding SQLite runner to the project.
```shell
dotnet add package FluentMigrator.Runner.SQLite
```

### Migration File
Next let's add a migration file to create `Customer` table. I have prefixed migration file with date time using pattern `yyyyMMdd_HHmm_` and followed by `_Table_` to denote this migration is creating a table and then finally followed by the table name.
I have used the same date time value as `Migration` attribute value. The contents of the file are as follows.
```csharp
using FluentMigrator;

namespace Customers.Api;

[Migration(202402132026)]
public class _20240213_2026_Table_Customer_Create : Migration
{
    public override void Up()
    {
        Create.Table("Customer")
            .WithColumn("Id").AsInt32().PrimaryKey().Identity()
            .WithColumn("FirstName").AsString(50)
            .WithColumn("LastName").AsString(50)
            .WithColumn("BillingAddressLine1").AsString(50)
            .WithColumn("BillingAddressLine2").AsString(50).Nullable()
            .WithColumn("BillingAddressLine3").AsString(50).Nullable()
            .WithColumn("BillingAddressLine4").AsString(50).Nullable()
            .WithColumn("BillingAddressCity").AsString(50)
            .WithColumn("BillingAddressPostCode").AsString(50)
            .WithColumn("BillingAddressCountry").AsString(50)
            .WithColumn("ShippingAddressLine1").AsString(50)
            .WithColumn("ShippingAddressLine2").AsString(50).Nullable()
            .WithColumn("ShippingAddressLine3").AsString(50).Nullable()
            .WithColumn("ShippingAddressLine4").AsString(50).Nullable()
            .WithColumn("ShippingAddressCity").AsString(50)
            .WithColumn("ShippingAddressPostCode").AsString(50)
            .WithColumn("ShippingAddressCountry").AsString(50);
    }

    public override void Down()
    {
        Delete.Table("Customer");
    }
}
```

I have decided to use the same table to store address. We can use a separate table to store the address values and use `ToTable` method to configure EF to store address in a separate table.

### Migration Runner
Next let's add a class to execute the migrations.
```csharp
using FluentMigrator.Runner;

namespace Customers.Api;

public static class CustomersDbMigrator
{
    public static void Run(IServiceProvider serviceProvider)
    {
        // Instantiate the runner
        var runner = serviceProvider.GetRequiredService<IMigrationRunner>();

        // Execute the migrations
        runner.MigrateUp();
    }
}
```

### Run Migrations
Add following to `appsettings.json`
```json
"ConnectionStrings": {
  "CustomersConnectionString": "Data Source=Customers.db"
}
```
And finally update `Program.cs` to register `FluentMigrator` and run migrations before running our app.
```csharp
var customersConnectionString = builder.Configuration.GetConnectionString("CustomersConnectionString");
...
builder.Services.AddFluentMigratorCore();
builder.Services.ConfigureRunner(rb => rb
        .AddSQLite()
        .WithGlobalConnectionString(customersConnectionString)
        .ScanIn(typeof(_20240213_2026_Table_Customer_Create).Assembly).For.Migrations());
...
using (var scope = app.Services.CreateScope())
{
    CustomersDbMigrator.Run(scope.ServiceProvider);
}
```
Running first time will create our table.

## EF Core Setup
### Models
Let's start by adding `Address` under `Models` folder.
```csharp
public class Address
{
    public string Line1 { get; set; }
    public string? Line2 { get; set; }
    public string? Line3 { get; set; }
    public string? Line4 { get; set; }
    public string City { get; set; }
    public string PostCode { get; set; }
    public string Country { get; set; }
}
```
And our root object `Customer`
```csharp
public class Customer
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public Address BillingAddress { get; set; }
    public Address ShippingAddress { get; set; }
}
```

### Entity Type Configuration
Next we will configure Address properties as Owned Entity by using `OwnsOne` method of `EntityTypeBuilder` in `CustomerEntityTypeConfiguration`. It will look like following
```csharp
public class CustomerEntityTypeConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(Microsoft.EntityFrameworkCore.Metadata.Builders.EntityTypeBuilder<Customer> builder)
    {
        builder.ToTable("Customer");

        builder.OwnsOne(p => p.BillingAddress, p =>
        {
            p.Property(pp => pp.Line1).HasColumnName("BillingAddressLine1");
            p.Property(pp => pp.Line2).HasColumnName("BillingAddressLine2");
            p.Property(pp => pp.Line3).HasColumnName("BillingAddressLine3");
            p.Property(pp => pp.Line4).HasColumnName("BillingAddressLine4");
            p.Property(pp => pp.City).HasColumnName("BillingAddressCity");
            p.Property(pp => pp.PostCode).HasColumnName("BillingAddressPostCode");
            p.Property(pp => pp.Country).HasColumnName("BillingAddressCountry");
        });
        builder.OwnsOne(p => p.ShippingAddress, p =>
        {
            p.Property(pp => pp.Line1).HasColumnName("ShippingAddressLine1");
            p.Property(pp => pp.Line2).HasColumnName("ShippingAddressLine2");
            p.Property(pp => pp.Line3).HasColumnName("ShippingAddressLine3");
            p.Property(pp => pp.Line4).HasColumnName("ShippingAddressLine4");
            p.Property(pp => pp.City).HasColumnName("ShippingAddressCity");
            p.Property(pp => pp.PostCode).HasColumnName("ShippingAddressPostCode");
            p.Property(pp => pp.Country).HasColumnName("ShippingAddressCountry");
        });
    }
}
```
### DbContext
Then we will add `ConsumersDbContext`
```csharp
public class CustomersDbContext : DbContext
{
    public DbSet<Customer> Customers { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        new CustomerEntityTypeConfiguration().Configure(modelBuilder.Entity<Customer>());
    }

    public CustomersDbContext(DbContextOptions<CustomersDbContext> options)
        : base(options)
    { }
}
```
### Dependency Injection
Finally lets add our db context to dependency injection in `Program.cs`.
```csharp
...
builder.Services
    .AddDbContext<CustomersDbContext>(options =>
        options.UseSqlite(customersConnectionString));
...
```

## Customers Controller
Before we add a controller to expose endpoints for `CR` operations. Let's add an interface and service to `Create` and `Get` customer model from database.
```csharp
public interface ICustomersService
{
    Task<Customer?> GetCustomerAsync(int id, CancellationToken cancellationToken = default);
    Task<int> CreateCustomerAsync(Customer customer, CancellationToken cancellationToken = default);
}

public class CustomersService : ICustomersService
{
    private readonly CustomersDbContext _customersDbContext;

    public CustomersService(CustomersDbContext customersDbContext)
    {
        _customersDbContext = customersDbContext;
    }

    public async Task<Customer?> GetCustomerAsync(int id, CancellationToken cancellationToken = default)
    {
        return await _customersDbContext.Customers
            .FirstOrDefaultAsync(x => x.Id == id, cancellationToken);
    }

    public async Task<int> CreateCustomerAsync(Customer customer, CancellationToken cancellationToken = default)
    {
        await _customersDbContext.Customers.AddAsync(customer, cancellationToken);
        await _customersDbContext.SaveChangesAsync(cancellationToken);
        return customer.Id;
    }
}
```

Now let's add a new controller that would expose the endpoints to perform the `Create` and `Read` operations.
```csharp
[ApiController]
[Route("api/[controller]")]
public class CustomersController : ControllerBase
{
    private readonly ICustomersService _customersService;

    public CustomersController(ICustomersService customersService)
    {
        _customersService = customersService;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetCustomerAsync(int id)
    {
        var customer = await _customersService.GetCustomerAsync(id);
        return Ok(customer);
    }

    [HttpPost]
    public async Task<IActionResult> CreateCustomerAsync([FromBody] CustomerRequest request)
    {
        var customer = new Customer
        {
            FirstName = request.FirstName,
            LastName = request.LastName,
            BillingAddress = new Address
            {
                Line1 = request.BillingAddress.Line1,
                Line2 = request.BillingAddress.Line2,
                Line3 = request.BillingAddress.Line3,
                Line4 = request.BillingAddress.Line4,
                City = request.BillingAddress.City,
                PostCode = request.BillingAddress.PostCode,
                Country = request.BillingAddress.Country,
            },
            ShippingAddress = new Address
            {
                Line1 = request.ShippingAddress.Line1,
                Line2 = request.ShippingAddress.Line2,
                Line3 = request.ShippingAddress.Line3,
                Line4 = request.ShippingAddress.Line4,
                City = request.ShippingAddress.City,
                PostCode = request.ShippingAddress.PostCode,
                Country = request.ShippingAddress.Country,
            },
        };
        var id = await _customersService.CreateCustomerAsync(customer);
        return Ok(new CustomerResponse(id));
    }
}

public record CustomerRequest(
    string FirstName,
    string LastName,
    CustomerRequestAddress BillingAddress,
    CustomerRequestAddress ShippingAddress);

public record CustomerRequestAddress(
    string Line1,
    string? Line2,
    string? Line3,
    string? Line4,
    string City,
    string PostCode,
    string Country);

public record CustomerResponse(int id);
```
I have added the request and response object in the same file as controller. Reason for this is that this is only going to be used in this controller. However if you are going to use the same classes in other .NET projects, it would be a good idea to extrat out a contracts project and publish as nuget package that clients can use.

## Test
Now code is done, let execute `dotnet run` from terminal and open Swagger UI in browser and test the operations.

## Dapper
Let's explore how to do the same with `Dapper` that is a popular micro-ORM. Let's start by adding package reference.
```shell
dotnet add package Dapper --version 2.1.28
```
Update our service to work with `Dapper`
```csharp
Task<Customer?> GetCustomerAsync(int id, bool useDapper = false, CancellationToken cancellationToken = default);
Task<int> CreateCustomerAsync(Customer customer, bool useDapper = false, CancellationToken cancellationToken = default);
```

### Create Customer
Creating a customer with dapper will be a bit involved, we would need to write the insert sql statement and flatten the object to pass all parameters. I have manully flattened the object and created a param object but `AutoMapper` will be very useful in this situation.
```csharp
if (useDapper)
{
    var sql = @"
        INSERT INTO Customer
            (FirstName, LastName,
            BillingAddressLine1, BillingAddressLine2, BillingAddressLine3, BillingAddressLine4, BillingAddressCity, BillingAddressPostCode, BillingAddressCountry,
            ShippingAddressLine1, ShippingAddressLine2, ShippingAddressLine3, ShippingAddressLine4, ShippingAddressCity, ShippingAddressPostCode, ShippingAddressCountry)
        VALUES
            (@FirstName, @LastName,
            @BillingAddressLine1, @BillingAddressLine2, @BillingAddressLine3, @BillingAddressLine4, @BillingAddressCity, @BillingAddressPostCode, @BillingAddressCountry,
            @ShippingAddressLine1, @ShippingAddressLine2, @ShippingAddressLine3, @ShippingAddressLine4, @ShippingAddressCity, @ShippingAddressPostCode, @ShippingAddressCountry);
        SELECT last_insert_rowid();";
    var param = new {
        customer.FirstName,
        customer.LastName,
        BillingAddressLine1 = customer.BillingAddress.Line1,
        BillingAddressLine2 = customer.BillingAddress.Line2,
        BillingAddressLine3 = customer.BillingAddress.Line3,
        BillingAddressLine4 = customer.BillingAddress.Line4,
        BillingAddressCity = customer.BillingAddress.City,
        BillingAddressPostCode = customer.BillingAddress.PostCode,
        BillingAddressCountry = customer.BillingAddress.Country,
        ShippingAddressLine1 = customer.ShippingAddress.Line1,
        ShippingAddressLine2 = customer.ShippingAddress.Line2,
        ShippingAddressLine3 = customer.ShippingAddress.Line3,
        ShippingAddressLine4 = customer.ShippingAddress.Line4,
        ShippingAddressCity = customer.ShippingAddress.City,
        ShippingAddressPostCode = customer.ShippingAddress.PostCode,
        ShippingAddressCountry = customer.ShippingAddress.Country,
    };

    using var connection = new SqliteConnection(_customersConnectionString);
    var id = await connection.ExecuteScalarAsync<int>(
        sql,
        param
    );
    return id;
}
```

### Get Customer
Update `GetCustomerAsync` method to return `Customer` using `Dapper`. We will use `splitOn` to map the result to multiple objects. We would also need to write the full column list and alias all the columns same as the nested object property names. Had we used separate table for `Address` we could have just used `*` instead.
```csharp
if (useDapper)
{
    var query = @"
        SELECT
            FirstName,
            LastName,
            BillingAddressLine1 AS Line1,
            BillingAddressLine2 AS Line2,
            BillingAddressLine3 AS Line3,
            BillingAddressLine4 AS Line4,
            BillingAddressCity AS City,
            BillingAddressPostCode AS PostCode,
            BillingAddressCountry AS Country,
            ShippingAddressLine1 AS Line1,
            ShippingAddressLine2 AS Line2,
            ShippingAddressLine3 AS Line3,
            ShippingAddressLine4 AS Line4,
            ShippingAddressCity AS City,
            ShippingAddressPostCode AS PostCode,
            ShippingAddressCountry AS Country
        FROM Customer
        WHERE Id = @id";
    using var connection = new SqliteConnection(_customersConnectionString);
    var customer = await connection.QueryAsync<Customer?, Address, Address, Customer?>(
        query,
        (customer, billingAddress, shippingAddress) => {
            customer.BillingAddress = billingAddress;
            customer.ShippingAddress = shippingAddress;
            return customer;
        },
        new { id },
        splitOn: "Line1, Line1");

    return customer.FirstOrDefault();
}
```
We will add a query parameter to `Get` endpoint to optionally use `Dapper`
```csharp
    public async Task<IActionResult> GetCustomerAsync(int id, [FromQuery] bool useDapper)
    {
        var customer = await _customersService.GetCustomerAsync(id, useDapper);
        ...
```

## Source
Source code for the demo application is host on GitHub in [ef-core-owned-entity](https://github.com/kashif-code-samples/ef-core-owned-entity) repository.

## References
In no particular order
* [Owned Entity Types](https://learn.microsoft.com/en-us/ef/core/modeling/owned-entities)
* [DDD Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
* [SQLite](https://www.sqlite.org/index.html)
* [SQLite EF Core Database Provider](https://learn.microsoft.com/en-us/ef/core/providers/sqlite/?tabs=dotnet-core-cli)
* [Fluent Migrator](https://fluentmigrator.github.io/)
* [Dapper](https://github.com/DapperLib/Dapper)
* [AutoMapper](https://docs.automapper.org/en/stable/)
and many more.