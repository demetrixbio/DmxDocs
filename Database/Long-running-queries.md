[[_TOC_]]

**Use case**: there are some materialized views in the warehouse that take a long time (~20 minutes) to refresh.  The default database timeout is set to 600 seconds or 10 minutes in the Db.fs file (Demetrix.Lims.Common or Demetrix.Warehouse.Common).  These scheduled jobs (via Hangfire) were failing with the following type of error.
```
Npgsql.NpgsqlException (0x80004005): Exception while reading from stream
 ---> System.TimeoutException: Timeout during reading attempt
```

One solution is to pass a `commandTimeout` parameter to the Db.CreateCommand call, for example:

```
 use cmd = Db.CreateCommand<"REFRESH MATERIALIZED VIEW common.assay_wells_with_measurements">(connection,commandTimeout=2400)
```

This approach works well if you don't need a transaction scope.  Be aware that transaction scopes have their own transaction timeouts and you will need to manage that local transaction timeout. Here's some test code that Darren wrote demonstrating how local control of timeouts works for non-transaction blocks:

```c#
open Plough.ControlFlow
open FSharp.Data.Npgsql
open System
module Db =
    (*
        Default batch size for table updates. Should work just fine in 99.999999 of scenarios.
        Note that there is parameters count validation in Npgsql library: 'A statement cannot have more than 65535 parameters'
        How to calculate: max allowed batch size for table = 65535 / number_of_columns_in_table
        Default batch size should be safe for tables that contain less then 32 columns 
    *)
    let [<Literal>] defaultBatchSize = 2000
    
    ///The time to wait (in seconds) while trying to execute a command before terminating the attempt and generating an error
    let [<Literal>] defaultCommandTimeout = 600
    
    /// compile time connection string set via lims specific txt file with default value if file not found
    let [<Literal>] connectionString = "Host=127.0.0.1;Username=postgres;Password=postgres;Database=lims;Pooling=true" // TextFile<"compile_time_connection_string.txt">.Text
    
    let [<Literal>] methodTypes = MethodTypes.Task ||| MethodTypes.Sync
type Db = NpgsqlConnection<ConnectionString=Db.connectionString,
                           CollectionType=CollectionType.ResizeArray, MethodTypes=Db.methodTypes,
                           Prepare=true, XCtor=true, CommandTimeout=Db.defaultCommandTimeout>
type Db<'a>() =
    static member inline openConnectionAsync() = 
        task {
            let conn = new Npgsql.NpgsqlConnection(Db.connectionString)
            do! conn.OpenAsync(Async.DefaultCancellationToken)
            return conn
        }
    
    static member inline openConnection() =
        let conn = new Npgsql.NpgsqlConnection(Db.connectionString)
        conn.Open()
        conn
    
/// Simple 5 second count WORKS
let test1() =
    task {
        use! connection = Db.openConnectionAsync()
        let start = DateTime.Now
        printfn "Running command"
        use testCommand = Db.CreateCommand<"select  * from pg_sleep(5)">(connection)
        let! result = testCommand.TaskAsyncExecute()
        printfn "returned from command"
        printfn $"Elapsed time: {DateTime.Now - start}"
        return 1;
    }
(*
lower timeout to 2, sleep for 5 seconds, then timeout - WORKS as expected
Unhandled exception. System.AggregateException: One or more errors occurred. (Exception while reading from stream)
 ---> Npgsql.NpgsqlException (0x80004005): Exception while reading from stream
 ---> System.TimeoutException: Timeout during reading attempt
 *)
let test2() =
    task {
        use! connection = Db.openConnectionAsync()
        let start = DateTime.Now
        printfn "Running command"
        use testCommand = Db.CreateCommand<"select  * from pg_sleep(5)">(connection,commandTimeout=2)
        let! result = testCommand.TaskAsyncExecute()
        printfn "returned from command"
        printfn $"Elapsed time: {DateTime.Now - start}"
        return 1;
    }
(*
Should exceed timeout of 600 seconds
and it does..
$ dotnet run
Running command
Unhandled exception. System.AggregateException: One or more errors occurred. (Exception while reading from stream)
 ---> Npgsql.NpgsqlException (0x80004005): Exception while reading from stream
 ---> System.TimeoutException: Timeout during reading attempt
*)
let test3() =
    task {
        use! connection = Db.openConnectionAsync()
        let start = DateTime.Now
        printfn "Running command"
        use testCommand = Db.CreateCommand<"select  * from pg_sleep(700)">(connection)
        let! result = testCommand.TaskAsyncExecute()
        printfn "returned from command"
        printfn $"Elapsed time: {DateTime.Now - start}"
        return 1;
    }
(*
700 second sleep with 1000 seconds should succeed
IT DOES
$ time dotnet run
Running command
returned from command
Elapsed time: 00:11:40.2627704
success! result: 1
Outer Elapsed time: 00:11:40.6636191
real    11m44.153s
user    0m0.000s
sys     0m0.077s
*)
let test4() =
    task {
        use! connection = Db.openConnectionAsync()
        let start = DateTime.Now
        printfn "Running command"
        use testCommand = Db.CreateCommand<"select  * from pg_sleep(700)">(connection,commandTimeout=1000)
        let! result = testCommand.TaskAsyncExecute()
        printfn "returned from command"
        printfn $"Elapsed time: {DateTime.Now - start}"
        return 1;
    }
// try 31 minute job with 35 min timeout
(*
$ time dotnet run
Running command
returned from command
Elapsed time: 00:31:00.2051993
success! result: 1
Outer Elapsed time: 00:31:00.6665752
real    31m5.685s
user    0m0.015s
sys     0m0.061s
*)
let test5() =
    task {
        use! connection = Db.openConnectionAsync()
        let start = DateTime.Now
        printfn "Running command"
        use testCommand = Db.CreateCommand<"select  * from pg_sleep(1860)">(connection,commandTimeout=2100)
        let! result = testCommand.TaskAsyncExecute()
        printfn "returned from command"
        printfn $"Elapsed time: {DateTime.Now - start}"
        return 1;
    }
let test = test5
let testTemplate() =
    task {
        use! connection = Db.openConnectionAsync()
        let start = DateTime.Now
        printfn "Running command"
        use testCommand = Db.CreateCommand<"select  * from pg_sleep(5)">(connection,commandTimeout=2)
        let! result = testCommand.TaskAsyncExecute()
        printfn "returned from command"
        printfn $"Elapsed time: {DateTime.Now - start}"
        return 1;
    }
[<EntryPoint>]
let main argv =
    let start = DateTime.Now
    try
        try
            let result = test() |> Plough.ControlFlow.Task.asAsync |> Async.RunSynchronously
            printfn $"success! result: {result}"
        with x ->
            printfn $"ExceptionElapsed time: {DateTime.Now - start}"
            printfn $"Exception: {x}"
    finally
        printfn $"Outer Elapsed time: {DateTime.Now - start}"
    0
```