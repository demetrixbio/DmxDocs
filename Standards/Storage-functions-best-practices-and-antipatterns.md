[[_TOC_]]

## General storage guidelines

* By default, write *asynchronous* Task based functions.  Use *synchronous* only when it makes more sense to do so.
* We make use of both FSharp.Data.Npgsql type provider and Dapper within solution. By default, - choose type provider approach, unless dynamic generation of sql is a must and there is no good workaround (e.q. fully dynamic generation of filter predicate based on user selection).
* Unless efficient and high throughput bulk processing / aggregation from multiple tables into a view is a must, - prefer simpler, single query storage functions. If you need to update a complex entity hierarchy, coordinate it from the domain layer within same transaction.
* If a function may be called in the scope of some transaction, use the System.Transactions.TransactionScope. Npgsql will implicitly force all commands within transaction scope to use same transaction. (see examples down below)
* Do not use the `Either<'a>` and `TaskEither<'a>` types inside storage functions. In case you need to get result from the database and it might not exist, return *option* of the given type.  You can easily map `'a option` to `Either<'a>`/`TaskEither<'a> using helper functions defined in `Plough.ControlFlow`
* Explicitly return named columns rather than * in order to make code more resilient to changes
* Map type provider generated records into explicit records used elsewhere in the code (Data or Shared layer)
* Always map results into a proper collection type - list, array, map etc.
* Use `SingleRow=true` for single line queries in order to get an Option result.
* Watch for connection / transaction leaks. Connections, commands and transaction scopes must be instantiated with `use` instead of `let`. Otherwise connection lifetime is unpredictable (till the object is garbage collected which can be a while) and can lead to connection pool starvation.
* In some cases we are implementing so called "query api storages" (for more details - check ...). It requires defining the storage module that is used to transpile query api DSL into sql. Such storage modules make use of Dapper for dynamic querying, and their name ends with `QueryStorage` suffix (as opposed to simple `Storage`). Good simple example of such query api storage is `Demetrix.[App].Domain.[Domain].Storage.ChemicalQueryStorage`.
* Storage module functions are typically prefixed with the type of SQL operation that will be performed: `get | insert | update | delete`. Again, - this is rather a recommendation for 90% of scenarious. There are some cases of exotic names, as long as they are more descriptive (e.q. `TagStorage.nonExpiredTagExistsById`, 'ExperimentVesselStorage.nonExpiredExperimentVesselsDoNotExist', 'StageStorage.checkMaxStatusOfChildren').

## Run-time errors involving NULL when joining tables

An example scenario:  The query you are running returns a Boolean but it turns out that it can sometimes return a null.

Normally the type provider would return a Boolean option (to reflect the fact that sometimes the query doesn't return true or false). Unfortunately postgres is isn't completely thorough with its analysis of its own types, so it basically lies to the type provider and says you are good, you will get a true/false and then at runtime it returns a null (this usually happens when you join tables).

If you can work out which query is doing this, the fix is to use a function inside postgres that converts null to some of the value (assuming that's okay), or you could modify the query to not return the role at all if you didn't want it.

The function is `coalesce(columnName,defaultValue)`.  For example, `coalesce(favorited,false)` would default to false when null is the answer. In case you just want to force nullability on return column type (e.q. nullability of columns due to LEFT JOIN), simply wrapping the column into `coalesce(column_name)` will make the trick.

Example:

```fsharp
let getUsersByIds userIds =
    task {
        use! connection = Db.openConnectionAsync()
        use getUsers = Db.CreateCommand<"
            SELECT
                u.id, u.created, u.valid_from, u.valid_to, u.alias_name, u.barcode, u.is_robot,
                COALESCE(s.client_id) AS client_id, COALESCE(s.client_name) AS client_name,
                u.first_name, u.last_name, COALESCE(p.email) AS email, COALESCE(p.sub_id) as sub_id,
                COALESCE(b.id) AS b_id, COALESCE(b.issued) AS b_issued, COALESCE(b.vendor) AS b_vendor,
                COALESCE(b.label) AS b_label, COALESCE(b.barcode) AS b_barcode
            FROM admin.user u
            LEFT JOIN admin.system s ON u.is_robot = TRUE AND s.id = u.id
            LEFT JOIN admin.person p ON u.is_robot = FALSE AND u.id = p.id
            LEFT JOIN material.barcode b ON b.id = u.barcode AND b.assigned = 'user'
            WHERE u.id=ANY(@user_ids)">(connection)
        let! results = getUsers.TaskAsyncExecute(user_ids = userIds)
        return results
               |> Seq.map (fun s ->
                   let user =
                      { Id = s.id
                        FirstName = s.first_name |> Option.defaultValue ""
                        LastName = s.last_name |> Option.defaultValue ""
                        AliasName = s.alias_name
                        IsRobot = s.is_robot
                        Barcode = s.barcode |> Option.map (fun _ -> s.b_barcode.Value)
                        Details = if s.is_robot then
                                    { ClientId = s.client_id
                                      ClientName = s.client_name } |> System
                                  else
                                    { Email = s.email.Value
                                      SubId = s.sub_id.Value } |> Person }
                   user.Id, user)
               |> Map.ofSeq
    }
```

## Working with Postgres types
There is often a need in defining internal storage types, that are not exposed to api clients, and typically are used to store some intermediate information during certain data processing pipeline. Such types are defined within a separate `Types` module, defined per schema (e.q. `Demetrix.[App].Domain.[Domain].Storage.Types`)

Whenever you create a new db object (enum, table, custom type etc), always set type owner to "read_write" user.

### Enums
Whenever you have a column that can be of predictable finite set of values, - use postgres enum functionality. Pro tip - postgres has notion of enum value priority. Good way to think about it - each enum case has some byte value assigned to it, from smallest to highest. Order is defined by initial enum creation. This feature is very usefull in various data aggregation scenarious, when you can take MIN/MAX of multiple records AND/OR compare enum value against some expected case (e.q. action.status > 'running').

Example enum creation in db migration script:
```sql
CREATE TYPE harvest.action_status AS ENUM ('pending', 'running', 'stopped', 'finished', 'skipped');
ALTER TYPE harvest.action_status OWNER TO read_write;
```

There are cases when you need to extend existing enum type with new value. While postgres supports easy syntax for such operation (`ALTER TYPE enum_type ADD VALUE 'new_value';`), sadly it can not be used within a transaction block. Db migrations are always applied transactionally, so we need to use the following workaround to extend enums:
```sql
-- set type of all columns that make use of given enum to text
ALTER TABLE harvest.action ALTER COLUMN status TYPE TEXT USING status::TEXT;
ALTER TABLE harvest.action ALTER COLUMN status SET DEFAULT 'pending';
-- drop old type
DROP TYPE harvest.action_status;
-- create a new enum from scratch
CREATE TYPE harvest.action_status AS ENUM ('pending', 'finished', 'skipped', 'running', 'stopped');
-- set type of all columns that make use of given enum to newly created type
ALTER TABLE harvest.action ALTER COLUMN status SET DEFAULT 'pending'::harvest.action_status;
ALTER TABLE harvest.action ALTER COLUMN status TYPE harvest.action_status USING status::harvest.action_status;
```

Npgsql types, inferred by type provider are **(almost) never** directly exposed to upper layers (the only case when violation of this rule can be justified is really efficient bulk processing, where you want to void extra mapping). Mapping inferred postgres enum type provider to an application level enum of discriminated union happens within Types module in Storage layer of domain project:

```fsharp
type ExperimentActionStatus = Demetrix.Data.Storage.Db.harvest.Types.action_status

// Action stages. Pending | Running | Stopped | Finished | Skipped
let enumToActionStatus = function
    | ExperimentActionStatus.pending -> ActionStatus.Pending
    | ExperimentActionStatus.running -> ActionStatus.Running
    | ExperimentActionStatus.stopped -> ActionStatus.Stopped
    | ExperimentActionStatus.finished -> ActionStatus.Finished
    | ExperimentActionStatus.skipped -> ActionStatus.Skipped    
    | x -> failwithf "Impossible action status %A" x

let actionStatusToEnum = function
    | ActionStatus.Pending -> ExperimentActionStatus.pending
    | ActionStatus.Running -> ExperimentActionStatus.running
    | ActionStatus.Stopped -> ExperimentActionStatus.stopped
    | ActionStatus.Finished -> ExperimentActionStatus.finished
    | ActionStatus.Skipped -> ExperimentActionStatus.skipped   

let stringToActionStatus = function
    | "pending" -> ActionStatus.Pending
    | "running" -> ActionStatus.Running
    | "stopped" -> ActionStatus.Stopped
    | "finished" -> ActionStatus.Finished
    | "skipped" -> ActionStatus.Skipped  
    | x -> failwithf "Impossible action stage %A" x
```

### Json

Intro to json in postgres: https://www.postgresql.org/docs/9.5/functions-json.html

There are cases when we make use of schemaless within the app. Typically this approach is used for assigning ontology tags to some objects using tag_assignment table. 

* Assigning a tag to some object means creation of a new entry in tag_assignment table, which consists of three columns: id_obj, id_domain, tags. 
* `id_domain` is an FK to ontology.domain - each object type (e.q. vessel, action, experiment_vessel) got it's own domain. 
* Combination of `id_domain` and `id_obj` is unique
* `tags` column is of jsonb type and stores a json object consisting of key value pairs, where key is tag id, and value - any optional json value, assigned to given tag for given object.

On application level, schemaless data is modeled via next types, defined in `Demetrix.Common.ValueObject`:
```fsharp
[<Struct>]
type AssignedTagValue = AssignedTagValue of json : string
with member this.Value with get() = let (AssignedTagValue json) = this in json

module AssignedTagValue =
    let get (AssignedTagValue json) = json
```

Deserialization of jsonb column makes use of helper function defined in `Demetrix.Lims.Common.Db` module:
```fsharp
let deserializeTags (tags : string option) : Map<TagIdentifier, AssignedTagValue option> =
    let tagLookup = tags |> Option.defaultValue "{}" |> JObject.Parse :> IDictionary<string, JToken>
    tagLookup
    |> Seq.map (fun pair ->
        let value = if pair.Value.Type = JTokenType.Null then None else pair.Value.ToString() |> AssignedTagValue |> Some
        pair.Key |> int |> TagIdentifier, value)
    |> Map.ofSeq
```

Serialization on the other hand is a bit trickier. Good example can be found in `Demetrix.Lims.Domain.Ontology.Storage.TagStorage`:
```fsharp
/// Bulk update of the tag assignments of same domain for the given objects.
/// If the given list of tags is empty, attempt to delete the record rather than updating/adding it,
/// to keep things tidy.
let updateTagAssignments (seed : TagAssignmentsSeed) =
    task {
        use scope = Db.createTransactionScope()
        let objectIds = seed.TagAssignments |> Seq.map (fun s -> s.ObjectId) |> Array.ofSeq
        do! deleteTagAssignments seed.DomainId objectIds

        let! tagAssignments = task {
            use table = new Db.onto.Tables.tag_assignment()
            let result =
                seed.TagAssignments
                |> Seq.map (fun tagAssignment ->
                    if not tagAssignment.Tags.IsEmpty then
                        let tagsParam =
                            tagAssignment.Tags
                            |> Seq.map (fun s ->
                                let tagKey = s.Key.Value |> string
                                let tagValue =
                                    match s.Value |> Option.map AssignedTagValue.get with
                                    | None -> Encode.nil
                                    | Some value -> JsonValue.Parse value
                                tagKey, tagValue)
                            |> List.ofSeq
                            |> Encode.object
                            |> Encode.toString 0
                        table.AddRow(seed.DomainId.Value, tagAssignment.ObjectId, tagsParam)

                    tagAssignment.ObjectId, { DomainId = seed.DomainId
                                              ObjectId = tagAssignment.ObjectId
                                              Tags = tagAssignment.Tags })
                |> Map.ofSeq
            use! connection = Db.openConnectionAsync()
            table.Update(connection, batchSize=Db.defaultBatchSize) |> ignore
            return result
        }
        scope.Complete()
        return tagAssignments
    }
```

## Useful tips

### Multiple result sets

Sometimes there's a need in storage function making multiple queries. In most cases it signals that logic is too tackled and can be refactored to make use of Domain layer. However if that's not a case (e.q. returning list of items by query + total count), instead of defining two DbCommands and querying db twice, - we can do everything in one roundtrip:

```fsharp
let getVesselsContainingMaterial materialType index limit =
    task {
        use scope = Db.createTransactionScope()
        let materialTypeParam = Storage.materialTypeToEnum materialType
        
        use! connection = Db.openConnectionAsync()
        use countCmd = Db.CreateCommand<"
            SELECT COUNT(1)
            FROM material.vessel v
            INNER JOIN material.material_place_time mpt ON v.id = mpt.id_vessel
            INNER JOIN material.material_type mt ON mpt.id_material_type = mt.id
            WHERE mpt.end_time > now() AND mt.type = @material_type;
            
            SELECT v.id, v.ext_id, v.sublocation, v.parent, v.vessel_type, v.vessel_role, v.updated, v.created,
                   v.project, v.id_user, v.logical_name, v.id_barcode AS barcode_id, COALESCE(b.barcode, NULL) AS barcode_value,
                   v.is_virtual, COALESCE(ta.tags,'{}') AS tags
            FROM material.vessel v
            LEFT JOIN material.barcode b ON v.id_barcode = b.id
            LEFT JOIN onto.domain d ON d.name = @domain
            LEFT JOIN onto.tag_assignment ta ON v.id = ta.id_obj AND ta.id_domain = d.id
            INNER JOIN material.material_place_time mpt ON v.id = mpt.id_vessel
            INNER JOIN material.material_type mt ON mpt.id_material_type = mt.id
            WHERE mpt.end_time > now() AND mt.type = @material_type
            ORDER BY id OFFSET @offset LIMIT @limit;">(connection)
        let! result = countCmd.TaskAsyncExecute(material_type = materialTypeParam,
                                                domain = Domain.vessel.Value,
                                                offset = int64 (index * limit),
                                                limit = int64 limit)
        scope.Complete()

        return { Items = result.ResultSet2 |> Seq.map (fun s ->
                   { Id = s.id
                     ExtId = s.ext_id |> Option.defaultValue ""
                     Location = { ParentId = s.parent |> Option.defaultWith (fun () -> failwith "Vessel must always have a parent")
                                  Sublocation = s.sublocation }
                     Type = TagIdentifier s.vessel_type
                     Role = TagIdentifier s.vessel_role
                     Project = TagIdentifier s.project
                     LogicalName = s.logical_name |> Option.defaultValue ""
                     Barcode = s.barcode_id |> Option.map (fun barcodeId -> { VesselBarcode.Id = barcodeId; Value = s.barcode_value.Value })
                     Tags = s.tags |> Db.deserializeTags
                     IsVirtual = s.is_virtual
                     UserId = s.id_user
                     Created = s.created
                     Updated = s.updated }) |> Seq.toList
                 PageSize = limit
                 PageIndex = index
                 TotalItemCount = result.ResultSet1 |> Seq.tryHead |> Option.flatten }
    }
```

### Nullable projections from LEFT JOIN

Sadly, the only information about column nullability that type provider gets from postgres is whether column in given table is nullable or not. However in case of `LEFT JOIN` the final projection will end up with nullable fields, even if they originate from non-nullable columns. We can hint type provider that given field is nullable via `COALESCE(foo.bar, NULL)` trick. In example below, we query vessels with their tags and barcodes. In case tags do not exist, we return empty array; in case barcode doesn't exist, we want to return nullable barcode value:

```
let getVesselsByIds vesselIds =
    task {
        use! connection = Db.openConnectionAsync()
        use cmd = Db.CreateCommand<"
            SELECT DISTINCT
                v.id, v.ext_id, v.sublocation, v.parent, v.vessel_type, v.vessel_role, v.updated, v.created,
                v.project, v.id_user, v.logical_name, v.id_barcode AS barcode_id, COALESCE(b.barcode, NULL) AS barcode_value, v.is_virtual,
                COALESCE(ta.tags,'{}') AS tags
            FROM material.vessel v
            LEFT JOIN material.barcode b ON v.id_barcode = b.id
            LEFT JOIN onto.domain d ON d.name = @domain
            LEFT JOIN onto.tag_assignment ta ON v.id = ta.id_obj AND ta.id_domain = d.id
            WHERE v.id = ANY(@v_ids)">(connection)
        let! xs = cmd.TaskAsyncExecute(v_ids = vesselIds, domain = Domain.vessel.Value)
        return xs
               |> Seq.map (fun s ->
                   s.id, { Id = s.id
                           ExtId = s.ext_id |> Option.defaultValue ""
                           Location = { ParentId = s.parent |> Option.defaultWith (fun () -> failwith "Vessel must always have a parent")
                                        Sublocation = s.sublocation }
                           Type = TagIdentifier s.vessel_type
                           Role = TagIdentifier s.vessel_role
                           Project = TagIdentifier s.project
                           LogicalName = s.logical_name |> Option.defaultValue ""
                           Barcode = s.barcode_id |> Option.map (fun barcodeId -> { VesselBarcode.Id = barcodeId; Value = s.barcode_value.Value })
                           Tags = s.tags |> Db.deserializeTags
                           IsVirtual = s.is_virtual
                           UserId = s.id_user
                           Created = s.created
                           Updated = s.updated })
               |> Map.ofSeq
    }
```

### Lateral join 

Postgresql is much more rich then common sql standard. One of usefull features is so called `LATERAL JOIN`. One can use lateral join in case when you want to use outer query variables inside join body rather then within `ON` condition. It can drastically change performance (in both ways), so make sure to run `EXPLAIN` while experimenting with it in performance critical scenarios. In example below we are getting experiment stage, it's started/finished dates and status, that are calculated from underlying actions. Instead of `INNER JOIN` on actions with further `GROUP BY stage.id` in outer query, - more readable and faster version is to go for `LATERAL JOIN`:

```sql
SELECT
    s.id, s.created, s.updated, s.id_experiment, s.stage_type, s.id_output_vessel,
    (vta.tags ->> @output_vessel_number_tag_id::text)::int AS output_vessel_number,
    s.previous_stage, s.id_workflow_import,
    a.planned_execution_date, a.started, a.finished, a.status
FROM harvest.stage s
INNER JOIN material.vessel v ON s.id_output_vessel = v.id
INNER JOIN onto.tag_assignment vta ON v.id = vta.id_obj AND vta.id_domain = @vessel_domain_id
CROSS JOIN LATERAL
    (
        SELECT
            MIN(id_stage) AS id_stage,
            MIN(planned_execution_date) AS planned_execution_date,
            MIN(started) AS started,
            MAX(finished) AS finished,
            MAX(status) as status
        FROM harvest.action WHERE id_stage = s.id AND umbrella_action IS NULL AND valid_to > now()
    ) a 
WHERE s.id = @stage_id
```

## Examples of common operations within storage later

### Simple retrieval call

```fsharp
let getVesselsByBarcodes (barcodes : string []) =
    task {
        use! connection = Db.openConnectionAsync()
        use cmd = Db.CreateCommand<"
            SELECT
                v.id, v.ext_id, v.sublocation, v.parent,
                v.vessel_type, v.vessel_role, v.updated, v.created, v.project,
                v.id_user, v.logical_name, v.id_barcode AS barcode_id, b.barcode AS barcode_value,
                v.is_virtual, COALESCE(ta.tags,'{}') AS tags
            FROM material.vessel v
            INNER JOIN material.barcode b ON v.id_barcode = b.id
            LEFT JOIN onto.domain d ON d.name = @domain
            LEFT JOIN onto.tag_assignment ta ON v.id = ta.id_obj AND ta.id_domain = d.id
            WHERE b.barcode = ANY(@barcodes)">(connection)  
        let! result = cmd.TaskAsyncExecute(barcodes=barcodes, domain = Domain.vessel.Value)
        return result
               |> Seq.map (fun s ->
                    { Id = s.id
                      ExtId = s.ext_id |> Option.defaultValue ""
                      Location = { ParentId = s.parent |> Option.defaultWith (fun () -> failwith "Vessel must always have a parent")
                                   Sublocation = s.sublocation }
                      Type = TagIdentifier s.vessel_type
                      Role = TagIdentifier s.vessel_role
                      Project = TagIdentifier s.project
                      LogicalName = s.logical_name |> Option.defaultValue ""
                      Barcode = s.barcode_id |> Option.map (fun barcodeId -> { VesselBarcode.Id = barcodeId; Value = s.barcode_value })
                      Tags = s.tags |> Db.deserializeTags
                      IsVirtual = s.is_virtual
                      UserId = s.id_user
                      Updated = s.updated
                      Created = s.created })
               |> Seq.toList
    }
```

In case query will return at most one result, use `Db.CreateCommand` method with `SingleRow=true` parameter, e.q:
```fsharp
use cmd = Db.CreateCommand<"
    SELECT
        v.id, v.ext_id, v.sublocation, v.parent,
        v.vessel_type, v.vessel_role, v.updated, v.created, v.project,
        v.id_user, v.logical_name, v.barcode AS barcode_id, b.barcode AS barcode_value,
        v.is_virtual, COALESCE(ta.tags,'{}') AS tags
    FROM material.vessel v
    INNER JOIN material.barcode b ON v.barcode = b.id
    LEFT JOIN onto.domain d ON d.name = @domain
    LEFT JOIN onto.tag_assignment ta ON v.id = ta.id_obj AND ta.id_domain = d.id
    WHERE b.barcode = @barcode", SingleRow=true>(transaction.Connection, transaction)  
```

**Tip:** In general, in case you have a bulk version of function, you can create a single row overload like so:

```fsharp
let getVesselByBarcode barcode =
    getVesselsByBarcodes [| barcode |] |> Task.map List.tryHead
```

### Common table expressions

In case we need to perform multiple queries, while result of later query explicitly depends on result of previous one (pipeline style), common table expressions are very handy. In example below we are preselecting all experiment stages with non-virtual vessels, which got non-finished underlying actions, and then finish those actions in second query. Example contains only one CTE, however in practice you can chain any number of queries, and they are not necessarily have to depend on each other:

```sql
WITH stages_to_finish AS (
    SELECT s.id, MAX(a.planned_execution_date) AS execution_time
    FROM harvest.stage s
     INNER JOIN material.vessel v ON v.is_virtual = false AND v.logical_name != '54_rack' AND s.id_output_vessel = v.id
     INNER JOIN harvest.action a ON a.id_stage = s.id AND a.valid_to > now()
    WHERE a.finished IS NULL
    GROUP BY s.id
)
UPDATE harvest.action a SET
    status = 'finished'::harvest.action_status,
    started = s.execution_time,
    finished = s.execution_time,
    updated = now()
FROM stages_to_finish s
WHERE s.id = a.id_stage AND a.finished IS NULL AND a.valid_to > now();
```

### Recursive queries

One very interesting use case of common table expressions is querying hierarchical data, like ontology tags, experiment stages/actions, vessels (via parent-child relationship) etc. Using `WITH RECURSIVE` CTE we can make self referencing query that will recursively join on itself up till the point when that join won't yield no results on some iteration (recursion exit condition). On example below, we will select all vessels of certain type from LIMS. Note that vessel type is an ontology tag, - that means that tag inheritance takes place - if we want to select all plates, - we need to query for all vessels whose type is indirect child (descendant) of 'plate' ontology tag (e.q. vessels with type '96 well plate', '384 well plate' are still plates, as defined by onto.relationship binding between ontology tags). In order to select all plates, we need to recursively generate vessel_type lookup definition (key = tag, value = descendant tag), than group it by key (key = tag, value = array of descendant tags), and only then query lims for vessels:

```sql
WITH RECURSIVE flat_type_lookup AS
(
    SELECT t.id, t.id as base_id, t.name AS type, t.name as base_type
    FROM onto.tag t
    INNER JOIN onto.namespace ns ON t.id_namespace = ns.id AND ns.name = @vessel_type_ns
    UNION
    SELECT t.id, current_level_rel.base_id, t.name AS type, current_level_rel.base_type
    FROM flat_type_lookup current_level_rel
    INNER JOIN onto.relationship rel ON current_level_rel.id = rel.child
    INNER JOIN onto.tag t ON t.id = rel.parent
),
type_lookup AS
(
    SELECT base_id, base_type, array_agg(type) AS type FROM flat_type_lookup GROUP BY base_id, base_type
)
SELECT v.id, v.ext_id, v.sublocation, v.parent, v.vessel_type, v.vessel_role, v.updated, v.created,                   
       v.project, v.id_user, v.logical_name, v.barcode AS barcode_id, b.barcode AS barcode_value,
       v.is_virtual, COALESCE(ta.tags,'{}') AS tags
FROM material.vessel v
INNER JOIN type_lookup ON type_lookup.base_id = v.vessel_type
INNER JOIN material.barcode b ON b.id = v.barcode
LEFT JOIN onto.domain d ON d.name = @domain
LEFT JOIN onto.tag_assignment ta ON v.id = ta.id_obj AND ta.id_domain = d.id
WHERE @vessel_type = ANY(type_lookup.type)
```

## How do I force Postgres to use certain index criteria?

In case of some tricky queries, Postgres might get confused and do crazy things like full table scan instead of index scan, that kills performance of the query in case of big table. Below are Darren's adventures on optimizing such query, pasted from Slack conversation.

darren  11:18 PM
Dropping some notes here from looking at the query performance.  We did some table analysis which moved things around but probably didn't change things dramatically.   This piece of query seems to be quite problematic (it's triggering a full table scan of stage)

![image](/uploads/416fba92baa93c26984642d928b4d512/image.png)

Causes db to try this (full scan to find all the non deleted rows as a first step)

![image](/uploads/5196bacb48d768263bf44766385ffb6a/image.png)

not sure why it doesn't wait to get the list of ids , then pull stages and finally filter down the non deleted ones, but it seems to think scanning the stage table is a good plan

darren  11:27 PM
we tried various example joins between stage queries (it joins stage 4 times) to try and get it to misbehave and it was mostly pretty thoughtful till it hits certain steps.  Trying to work out if this is reasonable or not, but I can't think why it would want to do a full table scan of stage

max  11:43 PM
you can 'force' parent index like so: https://stackoverflow.com/a/30859337

```
ON collection_action.parent = c.id AND
                    (CASE WHEN s.stage_type = ANY(collection_vessel_stage_types.ids) THEN collection_action.deleted IS NULL AND collection_action.parent != s.id ELSE false END)
```

darren  12:36 AM
nice ... so put a seemingly awful choice in front of it to divert it into the arms of an indexed column.   What puzzles me a bit is that it goes for a full scan as a backup - I feel like there is something I'm missing still

darren  1:14 AM
I did some vaccum analyze runs on a few of the bigger tables including scan and it doesn't seem to change its calculations much.  It's curious

darren  2:45 AM
It's fascinating - the optimizer takes a dim view of things naturally - if it thought it were better it would have proposed it itself probably.  Going to test if it really is worse but on paper (in silico?) it still wants to scan that collection_action table

![image](/uploads/f7c22dc13a51484f81427b87702d0615/image.png)

darren  3:00 AM
part of the problem is that it seems to be working from the other end - it is building the tree up from the botton and still wants to take the collection_action table and strip it down first and join against the parent.  It's a little odd - that join clause is acting on collection_action and c and s all in the one statement which I'm sure makes the optimizer a little more surly than usual

darren  3:05 AM
And in what can be described only as an 'oh wow :disappointed: ' moment,  simply reordering the join criteria from most stringent to least stringent in the ON clause reduces query time from 60 seconds to 7 seconds

![image](/uploads/497d656acda6ef7d2133b7562d083fcb/image.png)

put the parent first - actually looks like I might have duplicated that criteria.
Here is the full query I have been testing

THis is hilarious - if I comment out the redundant second criteria in the query,  it reverts to 55+ seconds instead of 5
![image](/uploads/e861fd81c6e90d62b4c0c13d229a2e81/image.png)

And commenting out the first parent = c.id and leaving the second is also > 1 minutes to run

![image](/uploads/2613b438926bea729d6ff62cc8985409/image.png)

This is a fun game - rechecking the two copies of parent = c.id and it's 9 ish seconds again!

![image](/uploads/7da166ea776fcdca9f1ab0240495c189/image.png)

Do the two copies have to be spaced like that or can I put them side by side?
they can be adjacent like this

![image](/uploads/9753d71d99efca4a00c377f5eff22e7a/image.png)

So, what have we learned? - the postgres optimizer is a little arbitrary and capricious but can be tricked by saying a join is importand AND a join is important.   That one line fix is probably worth including even with the other optimizations we are shipping in the next release

I am also starting to wonder if we have some job that is polling that service automatically - there is always a copy of that query running actively every time I've looked.  I will look when it's quieter but is there any chance that we have a front end tool that just reloads regularly or a hangfire task that's frequent?

## ADO.NET / Npgsql 

### Data table API

It is worth mentioning that type provider supports so called DataTable api. It can be used to insert/update data in certain db table. 

**Note 1:** Table must have a primary key in order to be supported, otherwise update will result in runtime exception. 
**Note 2:** In order to execute insert/update statements within one db roundtrip, provide `batchSize` parameter upon calling `DataTable.Update` method. There is `Db.defaultBatchSize` constant defined that will work in 99% of scenarios. Batch size is basically constraint around maximum amount of parameters that can be passed to postgres driver for one command.

In example below we will add new records to chemicals and material type tables:

```fsharp
let insertChemical userId chemical =
    task {
        use scope = Db.createTransactionScope()
        use! connection = Db.openConnectionAsync()
        use chemicalTable = new Db.material.Tables.chemical()
        let chemicalRow = chemicalTable.NewRow(inchi_key=chemical.InChI,
                                               iupac_name=chemical.IUPAC,
                                               formula=chemical.Formula,
                                               description=chemical.Description,
                                               state=(Storage.matterStateToEnum chemical.State),
                                               short_name=chemical.ShortName,
                                               synonym_group=(Guid.NewGuid().ToString("N")))

        chemicalTable.Rows.Add(chemicalRow)
        chemicalTable.Update(connection) |> ignore

        use materialTypeTable = new Db.material.Tables.material_type()
        let materialTypeRow = materialTypeTable.NewRow(``type``=Db.material.Types.``type``.chemical,
                                                       id_chemical=(Some chemicalRow.id),
                                                       id_user=userId)
        materialTypeTable.Rows.Add(materialTypeRow)
        materialTypeTable.Update(connection) |> ignore
        scope.Complete()
        return { Id = chemicalRow.id
                 InChI = chemicalRow.inchi_key
                 IUPAC = chemicalRow.iupac_name
                 Formula = chemicalRow.formula
                 Description = chemicalRow.description
                 State = chemicalRow.state |> Storage.enumToMatterState
                 ShortName = chemicalRow.short_name
                 SynonymGroup = chemicalRow.synonym_group }
    }
```

We can also use DataTable api to not only insert new records to table, but also update existing ones. In order to update existing records, DataTable object must be instantiated from `SELECT` projection. In example below we will update locked status of existing gsl document:

```fsharp
let private getGslSourceDocDataTable (connection : Npgsql.NpgsqlConnection) ids =
    task {
        use cmd = Db.CreateCommand<"
            SELECT id, name, content, created, updated, parents, locked, id_user, valid_to, compiler_flags, 
            user_access_rights_write_allowed, hashed_content
            FROM construction.gsl_source_doc WHERE id = ANY(@ids)", ResultType=ResultType.DataTable>(connection)
        return! cmd.TaskAsyncExecute(ids=ids)
    }

let updateLockedStatusByGslDocumentId (UserIdentifier userId) (GslIdentifier id) locked =
    task {
        use scope = Db.createTransactionScope()
        
        do! task {
            use! connection = Db.openConnectionAsync()
            use! table = getGslSourceDocDataTable connection [| id |]
            let row = table.Rows.[0]
            row.locked <- locked
            table.Update(connection) |> ignore
        }
        
        let! result = GslDocumentQueryStorage.getGSLDocument (UserIdentifier userId) (GslIdentifier id)
        scope.Complete()
        return result
    }
```

**Note:** While this is a perfectly valid pattern, it will issue 3 rountrips for a simple update (1. select into DataTable; 2. update DataTable; 3. getGSLDocument function will retrieve updated record back). Making things worse, this function under none circumstances **can not be called** in service layer in for loop / map etc, - **this will lead to 3N queries, where N is number of items to update**. Much better way for bulk updates will be described below.

### Bulk processing - Binary import

In cases where performance is crucial, we can use binary import instead of classical inserts or datatable api:

```fsharp
let [<Literal>] importCmd = "
    COPY material.material_place_time
        (created, id_material_type, id_vessel, start_time, end_time, material_expiration_date, quantity, id_unit_of_measure, id_delivering_material_transition, quantity_type)
    FROM STDIN (FORMAT BINARY)"
    
let insertMaterialPlaceTimes (materialPlaceTimeSeeds : MaterialPlaceTime list) =
    task {
        use scope = Db.createTransactionScope()
        use! connection = Db.openConnectionAsync()
        use writer = connection.BeginBinaryImport(importCmd)
        for seed in materialPlaceTimeSeeds do
            writer.StartRow()
            writer.Write(seed.Created, NpgsqlDbType.TimestampTz)
            writer.Write(seed.MaterialTypeId, NpgsqlDbType.Integer)
            writer.Write(seed.VesselId, NpgsqlDbType.Integer)
            writer.Write(seed.StartTime, NpgsqlDbType.TimestampTz)
            writer.Write(seed.EndTime, NpgsqlDbType.TimestampTz)
            writer.Write(seed.MaterialExpirationDate, NpgsqlDbType.TimestampTz)
            writer.Write(seed.Quantity, NpgsqlDbType.Numeric)
            writer.Write(seed.UnitOfMeasure, NpgsqlDbType.Integer)
            match seed.DeliveringTransition with
            | Some transition -> writer.Write(transition, NpgsqlDbType.Integer)
            | None -> writer.WriteNull()
            writer.Write(Storage.quantityTypeToString seed.QuantityType, NpgsqlDbType.Text)
        writer.Complete() |> ignore
        scope.Complete()
    }
```

### Bulk processing - CTE + UNNEST(@param) version

**The most preferable approach for bulk processing** is making use of common table expression for changeset, issuing update / insert, and retrieve updated version afterwards in single roundtrip. In case of single parameter the pattern is fairly straightforward, however if we have multiple parameters per each record, - we run into problem of passing tuples to postgres. The workaround is to pass arrays per each column, with further `UNNEST` function. Upon calling `UNNEST` against arrays of same length, - we will get beatiful records in CTE. In example below we define function that will update logical name of vessels, and will provide an overload for single update:

```fsharp
let renameVessels (changeset : Map<VesselId, string>) now =
    task {
        let ids = changeset |> Seq.map (fun s -> s.Key) |> Seq.toArray
        let logicalNames = changeset |> Seq.map (fun s -> s.Value) |> Seq.toArray
        
        use! connection = Db.openConnectionAsync()
        use cmd = Db.CreateCommand<"
            WITH changeset AS (
                SELECT UNNEST(@ids::int[]) AS id, UNNEST(@names::text[]) AS logical_name
            ),
            updated AS (
                UPDATE material.vessel v SET
                    logical_name = c.logical_name,
                    updated = @now
                FROM changeset c
                WHERE v.id = c.id
                RETURNING v.id
            )
            SELECT v.id, v.ext_id, v.sublocation, v.parent, v.vessel_type, v.vessel_role, v.updated, v.created,
                   v.project, v.id_user, v.logical_name, v.id_barcode AS barcode_id, COALESCE(b.barcode, NULL) AS barcode_value,
                   v.is_virtual, COALESCE(ta.tags,'{}') AS tags
            FROM material.vessel v
            INNER JOIN updated u ON v.id = u.id
            LEFT JOIN material.barcode b ON v.id_barcode = b.id
            LEFT JOIN onto.domain d ON d.name = @domain
            LEFT JOIN onto.tag_assignment ta ON v.id = ta.id_obj AND ta.id_domain = d.id">(connection)

        let! result = cmd.TaskAsyncExecute(ids=ids, names=logicalNames, now=now, domain=Domain.vessel.Value)
        return result
               |> Seq.map (fun s ->
                    s.id, { Id = s.id
                            ExtId = s.ext_id |> Option.defaultValue ""
                            Location = { ParentId = s.parent |> Option.defaultWith (fun () -> failwith "Vessel must always have a parent")
                                         Sublocation = s.sublocation }
                            Type = TagIdentifier s.vessel_type
                            Role = TagIdentifier s.vessel_role
                            Project = TagIdentifier s.project
                            LogicalName = s.logical_name |> Option.defaultValue ""
                            Barcode = s.barcode_id |> Option.map (fun barcodeId -> { VesselBarcode.Id = barcodeId; Value = s.barcode_value.Value })
                            Tags = s.tags |> Db.deserializeTags
                            IsVirtual = s.is_virtual
                            UserId = s.id_user
                            Updated = s.updated
                            Created = s.created })
               |> Map.ofSeq
    }
```

### Bulk processing - CTE + UNNEST(@param) version + NULLABLE values

There are some useful patterns for handling nullable values when using the bulk processing with CTE and the UNNEST approach. By default, the type provider doesn’t recognize nullable database columns. Without the explicit use of CASE statements, the compiler will produce errors. The unnest function is applied as above, but careful inspection of variables via CASE statements are used to assign null values in the SELECT portion of the CTE.  
In the example below, five of the nine columns can be null.  Note the use of JSON to serialize arrays of floats to a string for the nullable measurement_values column and the subsequent array aggregation of the de-serialized string.

Note: if this was not a bulk upload and the SQL statement didn’t include the ON CONFLICT or RETURNING clause, data could be inserted via the DataTable api which does handle nullable values.

```fsharp
    
do! task {
    let time_point = timePoints |> Seq.map (fun s -> s.MeasurementTimePoint.TimePoint) |> Seq.toArray
    let closest_time_point = timePoints |> Seq.map (fun s -> s.MeasurementTimePoint.ClosestTimePoint) |> Seq.toArray
    let average_value = timePoints |> Seq.map(fun s -> s.MeasurementTimePoint.MeasurementValue)|> Seq.toArray
    
    // The following fields are nullable, but to get around the type provider's limitations (Npgsql v 3.2.6),
    // we're build array of default values (e.g. -1.0) for any null values.  We subsequently interrogate these values 
    // in the SELECT statement below using CASE clauses, thus "tricking" the type provider into inserting nulls
    // when needed.
    let measurement_value_sd =
        timePoints
        |> Seq.map (fun s ->
            s.MeasurementStDev
            |> Option.defaultValue -1.)
        |> Seq.toArray
        
    let measurement_cv =
        timePoints
        |> Seq.map (fun s ->
            s.MeasurementCV
            |> Option.defaultValue -1.)
        |> Seq.toArray
        
    let measurement_count =
        timePoints
        |> Seq.map (fun s ->
            s.MeasurementCount
            |> Option.defaultValue -1)
        |> Seq.toArray
     
    let units =
        timePoints
        |> Seq.map (fun s ->
            s.MeasurementTimePoint.Unit)
        |> Seq.toArray
    // Some history here: Measurement values is an array of floats. We tried building a two-dimensional array of floats
    // but when we subsequently tried to un-nest the multidimensional array in the SELECT, this produced one base element per row
    // and we lost the desired 1 dimensionality of the measurement values array.
    // Instead, the array is serialized into a json string,  making it easy to later check if the array is null
    // in the CASE clause. Some SQL gymnastics are used to de-serialize the json and then aggregate the result
    // into an array of floating point values.
    let json_measurement_values =
        timePoints
        |> Seq.map (fun s ->
            s.MeasurementValues
            |> Option.map Json.SerializeToString |> Option.toObj)
        |> Seq.toArray
        
    use cmd2 = Db.CreateCommand<"""

    WITH changeset AS (
       SELECT
            UNNEST(@time_point::float[]) AS time_point,
            UNNEST(@closest_time_point::float[]) AS closest_timepoint,
            UNNEST(@measurement_value_sd::float[]) AS measurement_value_sd,
            UNNEST(@measurement_cv::float[]) AS measurement_cv,
            UNNEST(@measurement_value::float[]) AS measurement_value,
            UNNEST(@measurement_count::int[]) AS measurement_count,               
            UNNEST(@measurement_values::text[]) AS measurement_values,
            UNNEST(@units::text[]) AS units,         
            @id::int AS id
    )
    INSERT INTO fermentation.tank_timepoint_value (time_point,
        closest_time_point,
        measurement_value_sd,
        measurement_cv,
        measurement_value,
        measurement_count,
        measurement_values,
        id_experiment_measurement_tank,
        units)
    SELECT
        c.time_point,
        c.closest_timepoint,
        CASE WHEN c.measurement_value_sd = -1::float THEN NULL ELSE c.measurement_value_sd END,
        CASE WHEN c.measurement_cv = -1::float THEN NULL ELSE c.measurement_cv END,
        c.measurement_value,
        CASE WHEN c.measurement_count = -1 THEN NULL ELSE c.measurement_count END,
        CASE WHEN c.measurement_values is NULL THEN NULL ELSE array_agg(m::text::float) END,
        c.id,
        c.units
    FROM changeset c, jsonb_array_elements(c.measurement_values::jsonb) m
    group by time_point, closest_timepoint, measurement_value_sd, measurement_cv, measurement_value, measurement_count, measurement_values,  id,units
    ON CONFLICT (time_point, closest_time_point, measurement_value, units, id_experiment_measurement_tank)
    DO UPDATE SET id_experiment_measurement_tank=EXCLUDED.id
    RETURNING ID""">connection
        
    let! _ = cmd2.TaskAsyncExecute(time_point=time_point,
                                   closest_time_point= closest_time_point,
                                   measurement_value= average_value,
                                   measurement_value_sd = measurement_value_sd,
                                   measurement_cv = measurement_cv,
                                   measurement_count = measurement_count,
                                   measurement_values= json_measurement_values,
                                   units=units,
                                   id=result.Value)
    ()
}

```

### Bulk processing - dynamic sql - SELECT FROM VALUES

Another way to approach bulk processing is dynamic generation of query. The pros of this approach is full flexibility (sql query can be generated and is not a compile time constant), however we sacrifice compile time safety (direct usage of Npgsql istead of type provider). In example below we are bulk inserting measurements and points. In order to do that, we first generate changeset records, than insert measurements (returning inserted id, and associated x, y, create columns), and then insert points per each measurement. **Note** that we must use `Seq.chunkBySize` in order to not run into max db parameters limit from postgres.

```fsharp
// Bulk insert of measurements and measurement points.
// No actions linked here to measurements.
let insertMeasurementsBulk (measurements : (CreateMeasurementWithMetadata * (seq<PointValues>)) list) =
    task {
        use scope = Db.createTransactionScope()
        // Max npgsql params = 65535 / 9 = 7281
        let itemsInSingleCommand = 7281
        for batch in measurements |> Seq.chunkBySize itemsInSingleCommand do
            use! connection = Db.openConnectionAsync()
            use cmd = new Npgsql.NpgsqlCommand()
            cmd.Connection <- connection
            cmd.CommandTimeout <- 300
            
            let cmdText =
                batch
                |> Seq.mapi (fun index measurementTuples ->
                    let measurement, points = measurementTuples.Deconstruct()
                    
                    let idVesselParam = sprintf "id_vessel_%i" index                   
                    let initiatedParam = sprintf "initiated_%i" index
                    let equipmentParam = sprintf "equipment_%i" index
                    let executedByParam = sprintf "executed_by_%i" index
                    let pointStructureParam = sprintf "point_structure_%i" index
                    let mTypeParam = sprintf "m_type_%i" index
                    let metadataParam = sprintf "m_meta_%i" index
                    let xParam = sprintf "x_%i" index // several time values
                    let yParam = sprintf "y_%i" index // several 'value arrays'
                  
                    // TODO: executedBy - as 'nullable' parameter
                    let executedBy = if measurement.ExecutedById.IsSome then measurement.ExecutedById.Value else 1
                    
                    let xs = points |> Seq.map (fun p -> p.X) |> Array.ofSeq
                    let ys = points |> Seq.map (fun p -> JsonConvert.SerializeObject(p.Y)) |> Array.ofSeq                  
                    
                    cmd.Parameters.Add(idVesselParam, NpgsqlDbType.Integer, Value = measurement.VesselId) |> ignore                   
                    cmd.Parameters.Add(initiatedParam, NpgsqlDbType.TimestampTz, Value = measurement.Initiated) |> ignore
                    cmd.Parameters.Add(equipmentParam, NpgsqlDbType.Integer, Value = measurement.Equipment) |> ignore
                    cmd.Parameters.Add(executedByParam, NpgsqlDbType.Integer, Value = executedBy) |> ignore
                    cmd.Parameters.Add(pointStructureParam, NpgsqlDbType.Integer, Value = measurement.PointStructureId) |> ignore
                    cmd.Parameters.Add(mTypeParam, NpgsqlDbType.Integer, Value = measurement.MeasurementType) |> ignore
                    cmd.Parameters.Add(metadataParam, NpgsqlDbType.Jsonb, Value = JsonConvert.SerializeObject(measurement.Metadata)) |> ignore
                    cmd.Parameters.Add(xParam, NpgsqlDbType.Array ||| NpgsqlDbType.Double, Value = xs) |> ignore
                    cmd.Parameters.Add(yParam, NpgsqlDbType.Array ||| NpgsqlDbType.Jsonb, Value = ys) |> ignore    
                    
                    sprintf "(@%s, @%s, @%s, @%s, @%s, @%s, @%s::jsonb, @%s::double precision[], @%s::jsonb[], now())" 
                        idVesselParam initiatedParam equipmentParam executedByParam pointStructureParam mTypeParam metadataParam xParam yParam) 
                |> String.concat ", "  
                |> sprintf "WITH data AS (
                                SELECT *, row_number() OVER () as rnum FROM (VALUES  %s)
                                AS t (id_vessel, initiated, equipment, executed_by, point_structure, type, metadata, x, y, created)
                            ),
                            m_ids AS (
                                INSERT INTO measure.measurement (id_vessel, initiated, equipment, executed_by, point_structure, type, metadata)
                                SELECT id_vessel, initiated, equipment, executed_by, point_structure, type, metadata
                                FROM data
                                RETURNING id
                            ),
                            im AS (
                                SELECT *, row_number() OVER () as rnum
                                FROM m_ids
                                AS  t (id)
                            ),
                            point_tmp AS (
                                SELECT im.id,
                                unnest(data.x) as x,
                                jsonb_array_elements(unnest(data.y))::double precision as y,
                                data.created
                                FROM im
                                INNER JOIN data on im.rnum = data.rnum
                                )
                            INSERT INTO measure.point (id_measurement, x, y, created)
                            SELECT
                                pt.id, pt.x, array_agg(pt.y), pt.created
                            FROM point_tmp pt
                            group by pt.id, pt.x, pt.created"
            cmd.CommandText <- cmdText           
            
            let! _ = cmd.ExecuteNonQueryAsync()
            ()
        scope.Complete()
    }
```

## Transaction scope best practices and common errors

For transactional operations within applications, we are using [System.Transactions.TransactionScope](https://docs.microsoft.com/en-us/dotnet/framework/data/transactions/implementing-an-implicit-transaction-using-transaction-scope). 
The TransactionScope class provides a simple way to mark a block of code as participating in a transaction, without requiring you to interact with the transaction itself. A transaction scope can select and manage the ambient transaction automatically. Due to its ease of use and efficiency, it is recommended that you use the TransactionScope class when developing a transaction application.

In addition, you do not need to enlist resources explicitly with the transaction. Any System.Transactions resource manager (such as Npgsql) can detect the existence of an ambient transaction created by the scope and automatically enlist.

However, if not used properly, it can lead to runtime exceptions. 
Let's list common antipatterns based on example domain:
```fsharp
type Book =
    { Id : int
      Title : string
      AuthorId : int }
    
type Author =
    { Id : int
      FullName : string }

type BookDetails =
    { Id : int
      Title : string
      Author : Author }

let getBookById (bookId : int) : Task<Book option> =
    task {
        use! conn = Db.openConnectionAsync()
        use cmd = Db.CreateCommand<"SELECT id, title, id_author FROM public.book WHERE id = @id", SingleRow=true>(conn)
        let! result = cmd.TaskAsyncExecute(id=bookId)
        return result |> Option.map (fun s -> { Id = s.id; Title = s.title; AuthorId = s.id_author })
    }

let getAuthorById (authorId : int) : Task<Author option> =
    task {
        use! conn = Db.openConnectionAsync()
        use cmd = Db.CreateCommand<"SELECT id, full_name FROM public.author WHERE id = @id", SingleRow=true>(conn)
        let! result = cmd.TaskAsyncExecute(id=authorId)
        return result |> Option.map (fun s -> { Id = s.id; FullName = s.full_name })
    }

```

### 1. Multiple connections open at the same time
Having multiple open connections within same execution scope will lead to ambient transaction lifted to so called distributed transaction (aka Microsoft Distributed Transaction Control). That will result in runtime exception. Sometimes such antipatterns are hard to spot, when storage function is calling other nested storage functions, - in general, it is advised to keep storage functions as simple as possible - create connection, create command, execute command, map and return results. Below you can find variation of this antipattern:

```fsharp
let getBookDetailsById (bookId : int) : Task<BookDetails option> =
    task {
        use! conn = Db.openConnectionAsync()
        use cmd = Db.CreateCommand<"SELECT id, title, id_author FROM public.book WHERE id = @id", SingleRow=true>(conn)
        match! cmd.TaskAsyncExecute(id=bookId) with
        | Some book ->
            // getAuthorById will create new connection, while current one from above command was not closed nor disposed, leading to error
            let! author = getAuthorById book.AuthorId
            return Some { Id = book.Id; Title = book.Title; Author = author.Value }
        | None ->
            return None
    }
```

### 2. Subtle disposal of command before mapping the results of execution
Always make sure that connection and command is instantiated with `use` key word and exist only as long as the are really necessary - there's nothing worse then connection that is left open forever, - that leads to connection pool starvation. The longer you require open connection, - the more likely given piece of code will become a bottleneck of application - each request will open it's own connection while execution it, leading to only `n` concurrent requests to application possible instead of thousands, where `n` is number of connections allowed by database server. In case of multiple calls to database within single storage function, - make sure each call is wrapped with it's own scope like:

```fsharp
let storage_function param = 
    task {
        let! result1 = task { (*open connection, create command, execute and map result*) }
        let! result2 = task { (*open connection, create command, execute and map result*) }
        return map result1 result2
    }
```

*Important: always wrap inner query calls with task {}, - otherwise you risk connection to be disposed before result is mapped*. Below example will lead to runtime exception, as `cmd.TasAsyncExecute` of `let! book =` assignment was not awaited within connection/command lifetime scope

```
let getBookDetailsById (bookId : int) : Task<BookDetails option> =
    task {
        let! book =
            use conn = Db.openConnection()
            use cmd = Db.CreateCommand<"SELECT id, title, id_author FROM public.book WHERE id = @id", SingleRow=true>(conn)
            cmd.TaskAsyncExecute(id=bookId)

        match book with
        | Some s ->
            let! author = getAuthorById s.AuthorId
            return { Id = s.Id; Title = s.Title; Author = author.Value }
        | None ->
            return None
    }
```

Correct way:
```fsharp
let getBookDetailsById (bookId : int) : Task<BookDetails option> =
    task {
        let! book = task {
            use! conn = Db.openConnectionAsync()
            use cmd = Db.CreateCommand<"SELECT id, title, id_author FROM public.book WHERE id = @id", SingleRow=true>(conn)
            return! cmd.TaskAsyncExecute(id=bookId)
        }

        match book with
        | Some s ->
            let! author = getAuthorById s.AuthorId
            return { Id = s.Id; Title = s.Title; Author = author.Value }
        | None ->
            return None
    }

```

### 3. Completed non disposed transaction scope
Transaction scope api is pretty straight forward, - if you need some part of workflow be transactional, - open transaction scope using `use scope = Db.createTransactionScope()`, and, once finished, call `scope.Complete()`. All logic involving database (both Dapper and type provider) will be using single transaction, which will get committed once scope is completed. In case scope will get disposed without completion, - transaction will be rollbacked. Transaction scopes can be nested.
*However, if transaction scope was completed and not yet disposed*, all queries after completion and before disposal will fail in runtime. Below is example of such antipattern:

```fsharp
let getBookDetailsById (bookId : int) : Task<BookDetails option> =
    task {
        use scope = Db.createTransactionScope()
        let! book = getBookById bookId
        scope.Complete()
        match book with
        | None -> return None
        | Some s ->
            // underlying command execution will fail
            let! author = getAuthorById s.AuthorId
            return Some { Id = s.Id; Title = s.Title; Author = author.Value }
    }
```
As you can see, scope got completed, but it won't be disposed up till the point we return result from the `getBookDetails` function. That means, `getAuthorById` call will fail with scope completed exception. There are two possible fixes:

* Dispose scope manually before second call
```fsharp
let getBookDetailsById (bookId : int) : Task<BookDetails option> =
    task {
        use scope = Db.createTransactionScope()
        let! book = getBookById bookId
        scope.Complete()
        scope.Dispose()
        match book with
        | None -> return None
        | Some s ->
            // underlying command execution will fail
            let! author = getAuthorById s.AuthorId
            return Some { Id = s.Id; Title = s.Title; Author = author.Value }
    }
```

* Wrap book retrieval into sub lifetime scope
```fsharp
let getBookDetailsById (bookId : int) : Task<BookDetails option> =
    task {
        let! book = task {
            use scope = Db.createTransactionScope()
            let! result = getBookById bookId
            scope.Complete()
            return result
        }
        match book with
        | None -> return None
        | Some s ->
            let! author = getAuthorById s.AuthorId
            return Some { Id = s.Id; Title = s.Title; Author = author.Value }
    }
```
