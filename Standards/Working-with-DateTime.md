[[_TOC_]]

# General

There are a few best practices one need to follow in order to avoid painful debugging of DateTime specific problems. Before we dive into them, let's have a quick jump course on how DateTime values work in .NET and on frontend/backend/database levels.

## Environment specific presets

Demetrix services majority of time are hosted in AWS (except some CLI cases, where app is running on local machine in the lab).
In case of AWS, both server (hosted via ECS) and database (hosted via RDS) have UTC as a machine locale time zone. That means unless you are running on local machine (dev env, frontend client, cli app in lab) there's no difference between DateTime in local and DateTime in UTC. It is not recommended to depend on this feature, - this is last resort security net that will keep you sane though. 

**Key takeaways:**
* It's **ok** to use DateTime.Now if you pass this date to save it in database - conversion to UTC will be handled automatically by Npgsql given that destination column is timestamptz / timetz
* It's **ok** to use DateTime.Now if you just want to compare value against column in database - for example check if record was deleted today. 
* It's **not ok** to use DateTime.Now if you pass this date to frontend via api - this will vary depending on local dev environment vs aws environment due to different time zones.

In general, if you ask time specific questions against database, that are based just on notion of **now** - you are ok with DateTime in both Local and UTC time zone. However if you ask questions specific to pacific time zone - make sure your date contains correct PST value, convert it to UTC, and then ask database.

# NET core (Backend/CLI)

### DateTimeKind

The most important part to keep in mind when working with DateTime type is Kind property of it. [https://docs.microsoft.com/en-us/dotnet/api/system.datetime.kind?view=net-6.0](https://docs.microsoft.com/en-us/dotnet/api/system.datetime.kind?view=net-6.0.)

DateTimeKind is an enum with three possible options:

* UTC - given DateTime instance contains value in UTC
* Local - given DateTime instance contains value in **local** time zone
* Unspecified - given DateTime instance doesn't state in which time zone it is

When we are referring to local kind, we mean time in given machine's time zone. Thankfully, as of now, it's not possible to override time zone on dotnet process level.

It's also important to understand, that comparison of DateTime instances does take into account Kind property - you can not compare date in UTC vs date in Local/Unspecified.

### TimeZoneInfo

Time zone registry is specific to OS. This leads to some problems when you are developing a cross platform app that needs to fetch a particular time zone (e.q. `Pacific Standard Time`). Problem is described here - <https://devblogs.microsoft.com/dotnet/cross-platform-time-zones-with-net-core/>

Imagine following code:

```plaintext
let tzi = TimeZoneInfo.FindSystemTimeZoneById "Central Standard Time"
printfn "Timezone: tzi.DisplayName"
```

As of .NET 5, when you will try to execute this code on Windows platform, it will work correctly. On Linux/MacOS/Android etc it will fail with

```plaintext
Exception has occurred: CLR/System.TimeZoneNotFoundException
An unhandled exception of type 'System.TimeZoneNotFoundException' occurred in System.Private.CoreLib.dll: 'The time zone ID 'Central Standard Time' was not found on the local computer.'
```

This issue has been addressed in .NET 6 - there is a cldr project containing mappings of all time zone conversions between majority of platforms including Windows/Linux/MacOS/Android etc. - <https://github.com/unicode-org/cldr>

More details explanation of this problem can be found here - <https://devblogs.microsoft.com/dotnet/date-time-and-time-zone-enhancements-in-net-6/#time-zone-conversion-apis>

TL;DR: if you use .NET 6, in case time zone info wasn't found by requested name, .NET will try to find mapping of that name to a platform it's running on.

For everything else, there's ~~Mastercard~~ TimeZoneConverter library - <https://github.com/mattjohnsonpint/TimeZoneConverter>. Currently, Demetrix Services use it to allow cross-platform time zone lookups.

```plaintext
TZConvert.GetTimeZoneInfo "Central Standard Time"
```

### DateTime conversions between time zones

Sometimes we need to convert DateTime values between different timezones. This can be accomplished via various static methods of TimeZoneInfo class:

* [ConvertTime](https://docs.microsoft.com/en-us/dotnet/api/system.timezoneinfo.converttime?view=net-6.0)
* [ConvertTimeBySystemTimeZoneId](https://docs.microsoft.com/en-us/dotnet/api/system.timezoneinfo.converttimebysystemtimezoneid?view=net-6.0)
* [ConvertTimeFromUtc](https://docs.microsoft.com/en-us/dotnet/api/system.timezoneinfo.converttimefromutc?view=net-6.0)
* [ConvertTimeToUtc](https://docs.microsoft.com/en-us/dotnet/api/system.timezoneinfo.converttimetoutc?view=net-6.0)

There are tons of overloads for these methods, but in general there are following rules to follow:

* Conversion happens from source to destination time zone. **Source** time zone is **current** time zone of given DateTime instance. **Destination** time zone is **one you want to convert to**.
* `ConvertTimeFromUtc` uses UTC as source time zone and requires only destination time zone. Logically, it expects that your DateTime instance is already in UTC (DateTimeKind is set to UTC). If not, - it will throw an exception. **If no destination time zone provided - default machine time zone is used and resulted DateTime instance will have DateTimeKind.Local. If destination time zone is different from default machine time zone - resulted DateTime instance will have DateTimeKind.Unspecified.**
* `ConvertTimeToUtc` uses UTC as destination time zone. **If input DateTime instance has DateTimeKind.Local - you can not provide source time zone, - it will result in exception**. **If input DateTime instance has DateTimeKind.Unspecified, - source time zone must be provided - otherwise conversion will result in exception.**
* `ConvertTime` will require you to provide source and destination time zones. Same rules as above apply to this case, - DateTimeKind of input DateTime instance must be UTC with source being UTC | DateTimeKind must be Local with source being current machine time zone. Depending on destination time zone you'll get DateTime instance with DateTimeKind UTC | Local | Unspecified.

Hopefully, most of the time ~~zone~~ you don't need to use any other time zone then UTC or PST (due to Demetrix headquarters being in Pacific Standard Time.

### DateTime string format

There is a lot of control when it comes to DateTime representation as string - <https://docs.microsoft.com/en-us/dotnet/standard/base-types/custom-date-and-time-format-strings>

TL;DR: When DateTime is parsed `DateTime.Parse(string, format)` / serialized `DateTime.ToString(format)`, you have to provide format that is expected. If not provided, in case of parsing dotnet will do it's best to infer it. Inferring format will depend on your system locale and string itself (year, month, day, hour, minute, second, millisecond bits). When string contains 'Z' in the end - it means given date is in UTC format. In case format is not provided for `ToString` - full date time value will be printed with respect to your system locale - default format in case of European locales is `dd/MM/yyyy hh:mm:ss`

### DateTime parsing

There are multiple ways to get date time into your code. In most cases:

* value was deserialized from database. In case of Demetrix services Npgsql takes case of deserializing value from postgres to DateTime
* value was deserialized from json passed within server-client communication. In case of frontend-backend communication it's done via Thoth library.
* value was parsed from string. For example, this happens if you are parsing some files.

In case of database and server/client communication via JSON/XML, - **all deserializers are usually smart enough to pass datetime data in UTC**. However, in case of **manual parsing** from string you should be **careful**, - it entirely depends on string format. By default, if string was in UTC (contained 'Z' in the end), given that string is inferred to be in correct format, - resulted DateTime value with have DateTimeKind.UTC. If **string wasn't in UTC**, - you'll end up with **DateTime value with DateTimeKind.Unspecified**. If you will pass it for some further processing, it is strongly recommended to convert this DateTime instance to UTC with manually specified source time zone. In case of Pacific Standard Time, code to do that looks like this:

```plaintext
open System
open TimeZoneConverter
    
// Windows and Unix are using different timezone ids, so we need this library to have cross platform time zone info
let pacificTimeZone = TZConvert.GetTimeZoneInfo("Pacific Standard Time")

/// Considers this time to be in PST and converts it to UTC.
/// This function must be used in all places where DateTime is parsed from string that
/// DO NOT specify timezone and is not in UTC (ends with 'Z')  
let defaultToPacificTimeZone (dateTime : DateTime) =
    match dateTime.Kind with
    | DateTimeKind.Utc | DateTimeKind.Local -> 
        dateTime
    | DateTimeKind.Unspecified | _ ->
        // only set the timezone if it was not specified
        // this gives correct offset taking daylight saving into account
        // Courtesy of Alfonso and https://stackoverflow.com/a/246529
        TimeZoneInfo.ConvertTimeToUtc(dateTime, pacificTimeZone)
```

### DateTime helper functions in Demetrix.Common

There are a bunch of really helpful DateTime functions that can be found in Demetrix.Common project (in Extensions file)

```
[<RequireQualifiedAccess>]
module DateTime =
    open System
    
    #if !FABLE_COMPILER
    open TimeZoneConverter
    
    // Windows and Unix are using different timezone ids, so we need this library to have cross platform time zone info
    let pacificTimeZone = TZConvert.GetTimeZoneInfo("Pacific Standard Time")

    /// Considers this time to be in PST and converts it to UTC.
    /// This function must be used in all places where DateTime is parsed from string that
    /// DO NOT specify timezone and is not in UTC (ends with 'Z')  
    let defaultToPacificTimeZone (dateTime : DateTime) =
        match dateTime.Kind with
        | DateTimeKind.Utc | DateTimeKind.Local -> 
            dateTime
        | DateTimeKind.Unspecified | _ ->
            // only set the timezone if it was not specified
            // this gives correct offset taking daylight saving into account
            // Courtesy of Alfonso and https://stackoverflow.com/a/246529
            TimeZoneInfo.ConvertTimeToUtc(dateTime, pacificTimeZone)
            
    /// Sometimes we need to assume given DateTime instance is in PST timezone, no matter what Kind it is.
    /// Postgres and server timezones are always UTC, so it doesn't matter if specified Kind is Local or UTC.
    /// Client-Server communication also works only in UTC.
    /// Dotnet DateTime doesn't store instance timezone anywhere (it's either Local or UTC).
    /// So we mark instance as Unspecified timezone and convert it to UTC from perspective of PST timezone.
    /// This function should be used to ask questions specific to PST timezone, e.q.:
    /// What harvest stages will be executed on Thursday PST
    let forceAsInPacificTimeZone (dateTime : DateTime) =
        let dateAsUnspecified = DateTime.SpecifyKind(dateTime, DateTimeKind.Unspecified)
        defaultToPacificTimeZone dateAsUnspecified
    
    /// Converts given DateTime instance to PST time. As this is used only within server side, incoming date should be
    /// in UTC (Local timezone in case of server in cloud is still UTC). After all logic specific to PST is done,
    /// convert time back to UTC via **forceAsInPacificTimeZone** if date needs to be saved or passed further.
    /// DateTime instance does not store information about timezone, - so make sure to work in UTC unless PST specific validations/assertions are required.
    let convertToPacificTime (d: DateTime) =
        TimeZoneInfo.ConvertTime(d, pacificTimeZone)

    #endif
```

* defaultToPacificTimeZone should be used if you want to make sure that DateTime parsed from string will be either in machine local time zone or in UTC. If DateTime doesn't have DateTimeKind specified, it will consider this date in PST and convert it to UTC.
* forceAsInPacificTimeZone **should be used carefully!** One of the use cases: frontend sent us a **date (without time) in UTC** - this date is when lab operations in PST are unavailabe due to a holiday. In order to properly validate that there are no operations performed on this day - we need to mark it's kind as unspecified and convert it to UTC with source time zone specified as PST. Then given DateTime instance can be used to lookup operations in lims.
* convertToPacificTime **should be used carefully!** One of the use cases: given DateTime instance is in UTC and stores information when particular operation is performed in lab. We have some rule, that no operations of this kind can be performed after 17:00. In order to verify that date satisfies this rule, we need to convert date to PST and check it's Time property against TimeSpan("17:00"). **Important:** resulted DateTime instance will have DateTimeKind.Unspecified. Once all required checks are done, if you need to pass this date further to database/frontend/other backend services, - make sure to convert it back using `defaultToPacificTimeZone`.

### DateOnly / TimeOnly in .NET 6
There are some improvements in .NET 6 related to date/time handling. Key takeaways - if you are working with only date or only time - you will be able to use the appropriate types (instead of DateTime/TimeSpan). 
https://devblogs.microsoft.com/dotnet/date-time-and-time-zone-enhancements-in-net-6/#the-dateonly-type

# Fable (Frontend)
By default, when you are working with dates on frontend, all the rules mentioned above are respected by Fable. 
Additionally, if DateTime value originates from date picker selection by user - dates always have DateTimeKind.Local.

Backend <-> Frontend communication via json always works with dates in UTC. This means, - if you need to compare dates that come from backend to dates in date picker (for example, mark some dates in calendar as busy, - make sure to convert dates from backend to local via `TimeZoneInfo.ConvertTimeFromUtc`. Otherwise, you'll end up with weird bugs where comparison of dates will fail due to them having different DateTimeKind property value.

# Postgres (Database)

Postgres is handling dates differently from .NET - it has timestamp/timestamptz/date/time/timetz/interval types. https://www.npgsql.org/doc/types/datetime.html

Handling date and time values usually isn't hard, but you must pay careful attention to differences in how the .NET types and PostgreSQL represent dates. It's worth reading the PostgreSQL date/time type documentation to familiarize yourself with PostgreSQL's types.

There are two types in Postgres that are used for storing timestamps:
- timestamp (timestamp without timezone)
- timestamptz (timestamp with timezone)

**Timestamptz - Warning**

Condradictory to name, timestamptz field is not storing a timezone information. Instead: 
When you are writing a timezone specific value (DateTime with Kind = Local), - the assumption is made that datetime information in timezone of the server, - and datetime is being converted to UTC upon write.
When you are reading a value from a timestamptz column, - it's being read into a timezone specific value (DateTime with Kind = Local), - the assumption is made that datetime information in timezone of the server.

**Timestamp**

In case of both reads/writes, - date information is saved as is, without any notion of timezone - date/time values are considered to be in undetermined timezone (DateTime with Kind = Unspecified).

Typically, problem occurs when you are saving datetime in a certain timezone. Please check given Npgsql article to understand how values are serialized/deserialized between .NET and Postgres via Npgsql driver.
https://www.npgsql.org/doc/types/datetime.html#detailed-behavior-sending-values-to-the-database

**Best practices**

We typically store datetime values in timestamptz columns, - it means conversion to UTC is being performed by Npgsql internally before saving value / conversion to local (server timezone) happens upon read. 

When using an Npgsql library, upon inserting a DateTime value into timestamptz column, you must specify DbCommandParameter of NpgsqlDbType = TimestampTz, in order for Npgsql library to correctly handle conversion of DateTime.
Depending on DateTime.Kind property value:
Local - conversion to UTC happens internally by Npgsql before sending value to db.
Unspecified|Utc - no conversion happens, value sent as-is. 

When working with Postgres timestamptz column using FSharp.Data.Npgsql type provider, NpgsqlDbType is specified by type provider internally. 
You want to make sure that any DateTime you are passing to it while using DbCommand/DataTable api is either Local or UTC value. Most possible use cases, when you get DateTime with Unspecified Kind are parsing DateTime value from string, that does not contain timezone information. In this case make sure you are setting it to correct value timezone wise, using method below:

```fsharp
namespace Demetrix.Infrastructure.Extensions

open System

module DateTime =
    open TimeZoneConverter
    
    // Windows and Unix are using different timezone ids, so we need this library to have cross platform time zone info :facepalm:
    let defaultTimeZone = TZConvert.GetTimeZoneInfo("Pacific Standard Time")

    let withDefaultTimeZone (dateTime : DateTime) =
        match dateTime.Kind with
        | DateTimeKind.Utc | DateTimeKind.Local -> 
            dateTime
        | DateTimeKind.Unspecified | _ ->
            // only set the timezone if it was not specified
            // this gives correct offset taking daylight saving into account
            // Courtesy of Alfonso and https://stackoverflow.com/a/246529
            TimeZoneInfo.ConvertTimeToUtc(dateTime, defaultTimeZone)

```
