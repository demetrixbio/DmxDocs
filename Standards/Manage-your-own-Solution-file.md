[[_TOC_]]

## Intro

As you may have noticed, we have deleted all the unmaintained and noisy Visual Studio solution files from the repository and kept only one: `Demetrix.sln`

The reason being is basically it was very hard to maintain those many files and also totally unnecessary.

Since solution files

* are really just a grouped view of certain number of projects that have to be built together,
* it is very unlikely that someone will need all the 71 projects (at the time of writing) at once,
* bigger solutions lead to bigger solution load times, slower IDE responses, slower compilations,

we decided to have one complete Solution file, that's `Demetrix.sln`, we maintain that as a group, and individual developers can maintain their own "view", their own solution based on this base solution.

## How to create my own solution?

It's very easy.

1. Just copy `Demetrix.sln` to a new file in the repository, give it a different name, for instance `Demetrix.Legezam.sln`
2. Open up this solution in an IDE
3. Run a "Clean solution"  (build menu -> clean solution in rider)
4. Identify the projects that you don't need and right click, "unload project"
5. Build your solution
6. A) If it built successfully, proceed to step 8.
7. B) If it failed to build, then possibly you unloaded a project that is required by a loaded project. The build log will give hints about this. Once you identified the missing projects, right click on them, reload project. Proceed to step 5.
8. Remove the projects that remained unloaded from the solution, either by right click and remove, or just hit delete on your keyboard. (don't worry, it won't delete the project from the filesystem, will just remove from the solution)
9. Do a final build
10. Complete

![remove_project_from_solution](uploads/4a5ea06ec3200365477e510c5a9e79f2/remove_project_from_solution.gif)

## Example

This is an example of how to set up a solution for developing strains: (the unloaded projects should be removed from the solution):

![image](uploads/d370b69160aa1f6713b5ea00f1244d01/image.png)
![image](uploads/f2b3fc8d65f368464099c9bb06201d64/image.png)
![image](uploads/51007fff366b936331e93df789e80c8e/image.png)

### Result

![image](uploads/d444416008080eaa7f315711ba90de05/image.png)

## FAQ

### How do i actually identify the projects that i don't need?

Unfortunately, there is no silver bullet process to follow here, but there are some suggestions:

* If you are working on Warehouse, for instance, chances are good that you won't need projects, that reside inside `src/lims` and `src/commercial` . Likewise, if you are developing commercial app, you won't need lims and warehouse. With this step, you will get rid of majority of projects already
* When you are working in a specific domain of a given application, you can get rid of unrelated domains within the same app. This requires some practice, but eventually, as you get more experience about the area you are working on, this will get more and more easy to identify. For instance, if you work on StrainDesign domain, it has nothing to do with Inventory, BioCatalog, Experiment and such domains therefore those are safe to remove
* If you are working on a specifc client app, you can safely delete the others since client projects are not referencing each other
* Migrations project is almost always removable
* Don't remove shared projects! Something that has `Common` , `Shared`, `API` in the name, it is likely that it defines fundamental things that is relevant for all other projects therefore it is not recommended to remove those

### How i won't accidentally commit my personal solution to the git repository, since i have created it there?

Don't worry, we added some entries to `.gitignore` file so that:

* all `.sln` files get ignored in the repository, your custom solution won't accidentally be committed
* `Demetrix.sln` is an exception, so everyone will still receive all updates to the base solution

### What about XX.Domain.Web.fsproj?

At the time of writing, `XX.Domain.Web.fsproj` projects reference all the backend projects of the given app, so if you remove one of those, you can't build Web.fsproj anymore.

No worries:

* Just remove the web project from your personal solution, make your changes in your personal solution and do the Web related changes in the base solution. There shouldn't be big changes in the web project
* There are efforts being made where we decouple domain specific web code from the Web project so Web project will be crossed out as a problem in personal solutions eventually

### Do i need a custom solution at all?

It depends. Technically speaking, a custom solution is equivalent of the base solution with unnecessary projects unloaded. An unloaded project doesn't consume any resources on your computer just as if it werent there at all.

This question is rather about personal comfort. Having the base solution open with many projects unloaded causes a lot more "noise" in your solution explorer and makes it harder to navigate. 

So the whole question rather depends on personal preferences and there is no anything you cannot do with any of solutions.

### Help! - My solution doesn't compile

It is very likely that you have removed a project that would otherwise be needed by one of the projects in your personal solution. Fortunately, msbuild will give hints in the build log about which project is missing. Just add that project back to your solution and everything should be fine.

### Help! - My solution doesn't load

Probably a change on master made your solution be incompatible with the base solution. Since creating a personal solution is about 5-10 minutes with some practice, probably it is easier to delete your personal solution and recreate it from scratch.

### Help! - I am totally stuck!

As always, everyone is very happy to help you both in setting up your peronal solution, and troubleshooting it. Just shout in `#computing` channel.