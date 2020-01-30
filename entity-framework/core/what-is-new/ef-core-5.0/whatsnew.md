---
title: What's new in EF Core 5.0
author: ajcvickers
ms.date: 01/29/2020
uid: core/what-is-new/ef-core-5.0/whatsnew.md
---

# What's new in EF Core 5.0

EF Core 5.0 is currently in development.
This page will contain an overview of interesting changes introduced in each preview.
The first preview of EF Core 5.0 is tentatively expected in the next couple of months.

This page does not duplicate the [plan for EF Core 5.0](plan.md).
The plan describes the overall themes for EF Core 5.0, including everything we are planning to include before shipping the final release.

As we make progress on EF Core 5.0, many of the entries here will turn into links to the official documentation.

## Preview 1 (Not yet shipped)

### Simple Logging

This [feature](https://github.com/aspnet/EntityFrameworkCore/issues/1199) is the moral equivalent of `Database.Log` in EF6. That is, it provides a simple way to get logs from EF Core without the need to configure any kind of external logging framework.

EF Core replaces `Database.Log` with a `LogTo` method called on DbContextOptionsBuilder in either AddDbContext or OnConfiguring. For example:

```C#
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    => optionsBuilder.LogTo(Console.WriteLine);
```

Overloads exist to:

* Set the minimum log level
  * Example: `.LogTo(Console.WriteLine, LogLevel.Information)`
* Filter for only specific events:
  * Example: `.LogTo(Console.WriteLine, new[] {CoreEventId.ContextInitialized, RelationalEventId.CommandExecuted})`
* Filter for all events in specific categories:
  * Example: `.LogTo(Console.WriteLine, new[] {DbLoggerCategory.Database.Name}, LogLevel.Information)`
* Use a custom filter over event and level:
  * Example: `.LogTo(Console.WriteLine, (id, level) => id == RelationalEventId.CommandExecuting)`

Output format can be minimally configured (API is in flux) but the default output looks something like:

```
warn: 12/5/2019 09:57:47.574 CoreEventId.SensitiveDataLoggingEnabledWarning[10400] (Microsoft.EntityFrameworkCore.Infrastructure)
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data, this mode should only be enabled during development.
dbug: 12/5/2019 09:57:47.581 CoreEventId.ShadowPropertyCreated[10600] (Microsoft.EntityFrameworkCore.Model.Validation)
      The property 'BlogId' on entity type 'Post' was created in shadow state because there are no eligible CLR members with a matching name.
info: 12/5/2019 09:57:47.618 CoreEventId.ContextInitialized[10403] (Microsoft.EntityFrameworkCore.Infrastructure)
      Entity Framework Core 5.0.0-dev initialized 'BloggingContext' using provider 'Microsoft.EntityFrameworkCore.SqlServer' with options: SensitiveDataLoggingEnabled
dbug: 12/5/2019 09:57:47.644 CoreEventId.ValueGenerated[10808] (Microsoft.EntityFrameworkCore.ChangeTracking)
      'BloggingContext' generated temporary value '-2147482647' for the 'Id' property of new 'Blog' entity.
...
```

### Translation of Contains on byte arrays

Queries using Contains on byte[] properties are now translated to SQL. For example:

```C#
var blogs = context.Blogs.Where(e => e.Picture.Contains((byte)127)).ToList();
```

Translates to the following on SQL Server:

```
info: 12/5/2019 11:42:42.022 RelationalEventId.CommandExecuted[20101] (Microsoft.EntityFrameworkCore.Database.Command)
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT [b].[Id], [b].[Picture], [b].[Title]
      FROM [Blogs] AS [b]
      WHERE CHARINDEX(0x7F, [b].[Picture]) > 0
```

### Simple way to get generated SQL

EF Core 5.0 introduces the `ToQueryString` extension method which will return the SQL that EF Core will generate when executing a LINQ query. For example, the code:

```C#
var query = context.Set<Customer>().Where(c => c.City == city);
Console.WriteLine(query.ToQueryString())
```

results in this output when using the SQL Server database provider:

```SQL
DECLARE p0 nvarchar(4000) = N'London';

SELECT [c].[CustomerID], [c].[Address], [c].[City], [c].[CompanyName], [c].[ContactName], [c].[ContactTitle], [c].[Country], [c].[Fax], [c].[Phone], [c].[PostalCode], [c].[Region]
FROM [Customers] AS [c]
WHERE [c].[City] = @__city_0
```

Notice that declarations for parameters of the correct type are also included in the output. This allows copy/pasting to SQL Server Management Studio, or similar tools, such that the query can be executed for debugging/analysis.

### Debug views

Debug views are an easy way to look at the internals of EF Core when debugging issues. A debug view for the Model was implemented some time ago. For EF Core 5.0, we have made the model view easier to read and added a new debug view for tracked entities in the state manager.

#### Model debug view

Drill down into the Model property of the DbContext in your debugger of choice and expand the `DebugView` property.

![image](https://user-images.githubusercontent.com/1430078/70750412-39bc0a00-1ce3-11ea-9b34-5ab914658b78.png)

The `LongView` is the model view we have had for some time. The `ShortView` is new and doesn't include model annotations, which make it much easier to read. For example, here is one of our test models:

```
Model: 
  EntityType: Chassis
    Properties: 
      TeamId (int) Required PK FK AfterSave:Throw
      Name (string)
      Version (no field, byte[]) Shadow Concurrency BeforeSave:Ignore AfterSave:Ignore ValueGenerated.OnAddOrUpdate
    Navigations: 
      Team (_team, Team) ToPrincipal Team Inverse: Chassis PropertyAccessMode.Field
    Keys: 
      TeamId PK
    Foreign keys: 
      Chassis {'TeamId'} -> Team {'Id'} Unique ToDependent: Chassis ToPrincipal: Team
  EntityType: Driver
    Properties: 
      Id (int) Required PK AfterSave:Throw ValueGenerated.OnAdd
      CarNumber (Nullable<int>)
      Championships (int) Required
      Discriminator (no field, string) Shadow Required
      FastestLaps (int) Required
      Name (string)
      Podiums (int) Required
      Poles (int) Required
      Races (int) Required
      TeamId (int) Required FK Index
      Version (no field, byte[]) Shadow Concurrency BeforeSave:Ignore AfterSave:Ignore ValueGenerated.OnAddOrUpdate
      Wins (int) Required
    Navigations: 
      Team (_team, Team) ToPrincipal Team Inverse: Drivers PropertyAccessMode.Field
    Keys: 
      Id PK
    Foreign keys: 
      Driver {'TeamId'} -> Team {'Id'} ToDependent: Drivers ToPrincipal: Team
    Indexes: 
      TeamId
  EntityType: Engine
    Properties: 
      Id (int) Required PK AfterSave:Throw ValueGenerated.OnAdd
      EngineSupplierId (int) Required FK Index Concurrency
      Name (string) Concurrency
    Navigations: 
      EngineSupplier (_engineSupplier, EngineSupplier) ToPrincipal EngineSupplier Inverse: Engines PropertyAccessMode.Field
      Gearboxes (_gearboxes, ICollection<Gearbox>) Collection ToDependent Gearbox PropertyAccessMode.Field
      StorageLocation (Location) ToDependent Location PropertyAccessMode.Field
      Teams (_teams, ICollection<Team>) Collection ToDependent Team Inverse: Engine PropertyAccessMode.Field
    Keys: 
      Id PK
    Foreign keys: 
      Engine {'EngineSupplierId'} -> EngineSupplier {'Id'} ToDependent: Engines ToPrincipal: EngineSupplier
    Indexes: 
      EngineSupplierId
  EntityType: EngineSupplier
    Properties: 
      Id (int) Required PK AfterSave:Throw ValueGenerated.OnAdd
      Name (string)
    Navigations: 
      Engines (_engines, ICollection<Engine>) Collection ToDependent Engine Inverse: EngineSupplier PropertyAccessMode.Field
    Keys: 
      Id PK
  EntityType: Gearbox
    Properties: 
      Id (int) Required PK AfterSave:Throw ValueGenerated.OnAdd
      EngineId (no field, Nullable<int>) Shadow FK Index
      Name (string)
    Keys: 
      Id PK
    Foreign keys: 
      Gearbox {'EngineId'} -> Engine {'Id'} ToDependent: Gearboxes
    Indexes: 
      EngineId
  EntityType: Location
    Properties: 
      EngineId (no field, int) Shadow Required PK FK AfterSave:Throw ValueGenerated.OnAdd
      Latitude (double) Required Concurrency
      Longitude (double) Required Concurrency
    Keys: 
      EngineId PK
    Foreign keys: 
      Location {'EngineId'} -> Engine {'Id'} Unique Ownership ToDependent: StorageLocation
  EntityType: Sponsor
    Properties: 
      Id (int) Required PK AfterSave:Throw ValueGenerated.OnAdd
      ClientToken (no field, Nullable<int>) Shadow Concurrency
      Discriminator (no field, string) Shadow Required
      Name (string)
      Version (no field, byte[]) Shadow Concurrency BeforeSave:Ignore AfterSave:Ignore ValueGenerated.OnAddOrUpdate
    Keys: 
      Id PK
  EntityType: SponsorDetails
    Properties: 
      TitleSponsorId (no field, int) Shadow Required PK FK AfterSave:Throw ValueGenerated.OnAdd
      ClientToken (no field, Nullable<int>) Shadow Concurrency
      Days (int) Required
      Space (decimal) Required
      Version (no field, byte[]) Shadow Concurrency BeforeSave:Ignore AfterSave:Ignore ValueGenerated.OnAddOrUpdate
    Keys: 
      TitleSponsorId PK
    Foreign keys: 
      SponsorDetails {'TitleSponsorId'} -> TitleSponsor {'Id'} Unique Ownership ToDependent: Details
  EntityType: Team
    Properties: 
      Id (int) Required PK AfterSave:Throw
      Constructor (string)
      ConstructorsChampionships (int) Required
      DriversChampionships (int) Required
      EngineId (no field, Nullable<int>) Shadow FK Index
      FastestLaps (int) Required
      GearboxId (Nullable<int>) FK Index
      Name (string)
      Poles (int) Required
      Principal (string)
      Races (int) Required
      Tire (string)
      Version (no field, byte[]) Shadow Concurrency BeforeSave:Ignore AfterSave:Ignore ValueGenerated.OnAddOrUpdate
      Victories (int) Required
    Navigations: 
      Chassis (_chassis, Chassis) ToDependent Chassis Inverse: Team PropertyAccessMode.Field
      Drivers (_drivers, ICollection<Driver>) Collection ToDependent Driver Inverse: Team PropertyAccessMode.Field
      Engine (_engine, Engine) ToPrincipal Engine Inverse: Teams PropertyAccessMode.Field
      Gearbox (_gearbox, Gearbox) ToPrincipal Gearbox PropertyAccessMode.Field
    Keys: 
      Id PK
    Foreign keys: 
      Team {'EngineId'} -> Engine {'Id'} ToDependent: Teams ToPrincipal: Engine
      Team {'GearboxId'} -> Gearbox {'Id'} Unique ToPrincipal: Gearbox
    Indexes: 
      EngineId
      GearboxId Unique
  EntityType: TestDriver Base: Driver
  EntityType: TitleSponsor Base: Sponsor
    Navigations: 
      Details (_details, SponsorDetails) ToDependent SponsorDetails PropertyAccessMode.Field
```

#### State manager debug view

The state manager is a little mode hidden than the model. To find it, drill down into the ChangeTracker property of the DbContext in your debugger of choice and then look in the `StateManager` property and expand the `DebugView`.

![image](https://user-images.githubusercontent.com/1430078/70750441-493b5300-1ce3-11ea-9203-2e4b1a3a740e.png)

The short view of the state manager shows:

* Each entity being tracked
* Its primary key value(s)
* The entity state - one of Added, Unchanged, Modified, or Deleted.
* The values of an foreign key properties

For example:

```
Engine (Shared) {Id: 1} Unchanged FK {EngineSupplierId: 1}
Location (Shared) {EngineId: 1} Unchanged FK {EngineId: 1}
Team (Shared) {Id: 4} Modified FK {EngineId: 1} FK {GearboxId: <null>}
```

The long view shows everything in the short view together with:

* The current value of each property
* Whether or not the property is marked as modified
* The original value of the property, if different from the current value
* The entity referenced by a reference navigation using the referenced entity's primary key value
* The list if entities referenced by a collection navigation, again using primary key values

For example:

```
Engine {Id: 1} Unchanged
  Id: 1 PK
  EngineSupplierId: 1 FK
  Name: 'FO 108X'
  EngineSupplier: <null>
  Gearboxes: <null>
  StorageLocation: {EngineId: 1}
  Teams: [{Id: 4}]
Location (Shared) {EngineId: 1} Unchanged
  EngineId: 1 PK FK
  Latitude: 47.64491
  Longitude: -122.128101
Team {Id: 4} Modified
  Id: 4 PK
  Constructor: 'Ferrari'
  ConstructorsChampionships: 16
  DriversChampionships: 15
  EngineId: 1 FK Modified Originally 3
  FastestLaps: 221
  GearboxId: <null> FK
  Name: 'Scuderia Ferrari Marlboro'
  Poles: 203
  Principal: 'Stefano Domenicali'
  Races: 805
  Tire: 'Bridgestone'
  Version: '0x000000000001405A'
  Victories: 212
  Chassis: <null>
  Drivers: []
  Engine: {Id: 1}
  Gearbox: <null>
```

