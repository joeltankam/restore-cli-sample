# Sample project

This repository is a sample related to [dotnet/project-system#8381](https://github.com/dotnet/project-system/issues/8381).

## Summary

The `project.assets.json` file is incomplete for a C# project when it transitively depends on another C# project through a C++/CLI project. This problem seems to only occur in **Visual Studio 2022 17.3** (tested with Preview 4.0 to 6.0) and not previous versions.

Lets say we have three projects: `ProjectA` (C#), `ProjectB` (C++/CLI) and `ProjectC` (C#); so that `ProjectA` references `ProjectB` which references `ProjectC`:

```txt
ProjectA (C#) --(depends on)--> ProjectB (C++/CLI) --> ProjectC (C#)
```

If the `ProjectReference` from `ProjectB` to `ProjectC` does not contain the metadata `Project` (e.g. `<Project>{e5599aee-f39e-4613-b48d-7824eca526e5}</Project>`), MSBuild (through command line) and Visual Studio restore won't behave the same way. Typically, the transitive reference to `ProjectC` is missing from `ProjectA`'s asset file when performing NuGet restore in Visual Studio:

```diff
  (...)
  "targets": {
    ".NETFramework,Version=v4.8": {
      "ProjectB/1.0.0": {
        "type": "project",
-       "dependencies": {
-         "ProjectC": "1.0.0"
-       },
        "compile": {
          "bin/placeholder/ProjectB.dll": {}
        },
        "runtime": {
          "bin/placeholder/ProjectB.dll": {}
        }
      },
-     "ProjectC/1.0.0": {
-       "type": "project",
-       "framework": ".NETFramework,Version=v4.8",
-       "compile": {
-         "bin/placeholder/ProjectC.dll": {}
-       },
-       "runtime": {
-         "bin/placeholder/ProjectC.dll": {}
-       }
-     }
    }
  }
  (...)
```

## Steps to Reproduce

1. Perform a restore on `ProjectA` using MSBuild and check the contain of `.\ProjectA\obj\project.assets.json`: `msbuild -v:m -t:Restore .\ProjectA\ProjectA.csproj`
2. Open the solution in Visual Studio 2022 17.3 Preview, force a restore if needed, and check the content of `.\ProjectA\obj\project.assets.json` again; you will notice the changes mentioned in the summary

## Expected Behavior

MSBuild CLI and Visual Studio should have the same behavior, which means the restore should not be performed when opening the solution in Visual Studio. Package Manage log should be:

> All packages are already installed and there is nothing to restore.

## Actual Behavior

The restore is performed again in Visual Studio for `ProjectA`:

> Restored (...)\ProjectA\ProjectA.csproj (in 0.9 ms).
> NuGet package restore finished.

Changing `.\ProjectA\obj\project.assets.json` as described in the summary; creating incremental compilation issues when switching from command-line to VS (e.g. some projects are unnecessarily re-restored and recompiled).
