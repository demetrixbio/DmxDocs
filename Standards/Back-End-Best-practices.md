[[_TOC_]]

# General guidelines

1.  Naming of query functions should follow the uniqueness of the underlying entity.
    - If something is not unique, use `getNamespaces` (list return type) and `countNamespaces` (integer return type)
    - If something is unique, use `getNamespace` (option return type) and `namespaceExists` (bool return type) - NB: `SELECT EXISTS 
 (...)` construct with `SingleRow=true` could be very useful as the db result type becomes a single boolean automatically

# Using a task inside a taskEither

To lift a task inside a taskEither, you can use `let!` (pronounced as let bang).

```fsharp
let searchBuiltStitchesByTargetId (targetId : TargetIdExp) : TaskEither<TargetDetails list> =
    taskEither {
        let! foundTargets = PartStorage.searchBuiltStitchesByTargetId targetId

        // ...
    }
```

# Returning storage results directly from service

To return the results from the storage directly, you can use `TaskEither.ofTask`

```fsharp
open Plough.ControlFlow

let getCropOverview (query : CropQuery) =
    SearchCropStorage.getCropsPagedByQuery query.Filter query.PageIndex query.PageSize
    |> TaskEither.ofTask
```

# Using an either inside a taskEither

Calling an _either_ from inside an _taskEither_:

```fsharp
let fromWellLabel (containerTopology: ContainerTopology) (well: string) : Either<WellSublocation> =
        either {
            match containerTopology with
            | ContainerTopology.OneDimensional _ -> return (well |> int) - 1
            | ContainerTopology.TwoDimensional d ->
                let rows = generateRowLabels d.Rows
                let row = rows |> Array.findIndex(fun x -> well.Substring(0, 1) = x)
                let col = well.Substring(1, well.Length - 1) |> int
                return row * d.Cols + col - 1
            | _ -> return! "Invalid container to retrieve sublocation from well label" |> Validation |> Either.fail
        }

let getVessels (id : VesselPk) : TaskEither<Vessel list> =
    taskEither {
        // ...
        let! rowSublocation = row.WellPosition.Value |> Vessel.fromWellLabel vesselTopology
        // ...
    }
```


# Running and collecting the results of a sequence of _taskEither_ functions:

```fsharp
taskEither {
    let! harvest = Api.StrainBuilding.GetHarvestStructureById harvestId
    let! designs = harvest.Designs
                   |> Seq.map (fun el () -> DesignEditor.init)
                   |> TaskEither.collect
             // ....
        }
```

Note that `collect` is different from other sequences, requiring the return type of the mapping to be `unit -> TaskEither<'a>`. This is because tasks are greedy (and will start immediately), leading to concurrent connection issues if mapped over with the actual task CEs being returned, rather than a function that returns the CE.
