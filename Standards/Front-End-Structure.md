[[_TOC_]]

### Rules:
- All components are in the components folder
- Scope expressions:
	- Global: reuasble, available for entire application (single file)
	- Local: reusable, available for a part of the application (single file)
	- Single: only used once (dedicated folder)
- Reusable components are in one file (scope = global or local)
- Single usage components:
	- Child compoments have a dedicated folder (scope = single)
	- Entry point of component has the same name as the folder
	- Entry point of component has `component` at the end of the namespace when there are compiler conflicts
- Helper functions and views are in `Global` folder
- Types are in a `Type.fs` file, or in a folder called `Types` in case the app has a lot of types
- `Router.fs` is directly in app folder

### Folder structure excerpt

- Construction
    - Global
		- NoResults.fs (view)
		- ApiAdapter.fs (helper functions)
		- ...
    - scss
		- cropbuilder.scss
		- main.scss
    - Components (folder, not in namespace)
		- Pagination.fs (end of line, `scope is global`, meaning available for the entire app)
		- CropBuilder (folder)
			- OutputFile.fs (end of line, `scope is local`, meaning available for entire CropBuilder )
			- Overview (folder)
				- Overview.fs (end of line, `scope is single usage`, child component of CropBuilder, no more child components because no folders)
			- Workflow (folder)
				- CheckIn (folder)
					- PlateLayout (folder)
						- …
						- PlateLayout.fs (end of line, scope is single usage, child component of CheckIn, has child components because there are folders)
					- PlateCreation (folder)
						- PlateCreation.fs (end of the line, scope is single usage, child component of CheckIn, no more child components)
					- CheckIn.fs (entry point into component `CheckIn`)
				- Checklist (folder)
					- …
				- Workflow.fs (entry point into component `Workflow`)
			- Editor (folder)
				- … (similar structure as Workflow)
			- CropBuilder.fs (entry point into component `CropBuilder`)
	- Router.fs
	- Types.fs
    - App.fs (entry point for construction application)

### Namespaces excerpt

File name     | Namespace
---		      | ---
NoResults.fs | Construction.Views.NoResults
ApiAdapter.fs | Construction.Helpers.ApiAdapter
Pagination.fs | Construction.Pagination
OutputFile.fs | Construction.CropBuilder.OutputFile
Overview.fs   | Construction.CropBuilder.Overview.Component
PlateLayout.fs | Construction.CropBuilder.Workflow.CheckIn.PlateLayout.Component
PlateCreation .fs | Construction.CropBuilder.Workflow.CheckIn.PlateCreation.Component
CheckIn.fs | Construction.CropBuilder.Overview.CheckIn.Component
CropBuilder.fs | Construction.CropBuilder.Component
Router.fs | Construction.Router
Types.fs | Construction.Types
App.fs | Construction.App

##### Usage when ending in component:
- module CropBuilder = Construction.CropBuilder.Component
- module CheckIn = Construction.CropBuilder.Overview.CheckIn.Component
- module PlateLayout = Construction.CropBuilder.Workflow.CheckIn.PlateLayout.Component

### Router excerpt

``` fsharp

type WorkflowRoute =
	| CheckIn of plateType : string option * cropId : int
	| CheckList of cropId : int

type CropBuilderRoute =
	| Editor of EditorRoute
	| Wowrkflow of WorkflowRoute

type Router =
	| CropBuilder of CropBuilderRoute

```


### PRO and CON of this approach:

PRO | CON | Fix
--- | --- | ---
Folder structure (almost) matches namespaces | A lot of components have module name component |
Thinking in terms of components instead of pages (since it is difficult to draw a line for all applications what is a page and what is a component) | Reusable components have to be in one file | Add structure inside the file as you see fit
Reusability of components are expressed (global, local or single usage) | `module Module = …` opening statements needed |
Scalable | Some folders will have a lot of reusable components in it |
Relationship between components are expressed | |

### Prototype and refactoring

Ongoing, so you will find apps with a different structure.