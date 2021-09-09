Cross-TFM Mono.Cecil assembly resolver and reflection/metadata importer
=======================================================================

One common use of Mono.Cecil is in tools that run as part of the build
process to apply some kind of post-processing to the output assembly.
Such tools may be implemented as standalone console applications invoked
from some suitable target with `<Exec>` or as MSBuild tasks (the latter
approach is tricky because of limitations in how MSBuild handles task
assembly dependencies, but this does not concern me here).
Post-processing often involves examining or adding code to the output
assembly, and this in turn necessitates importing type, method and field
references into it. This can become complicated because the output assembly,
its dependencies that may have to be examined by the post-processor, and
the post-processor itself may all belong to different target frameworks (TFMs).
Framework types such as `System.Threading.Tasks.Task` or `System.IO.Stream`
will exist in different metadata scopes. The default implementations of
`IMetadataImporter` and `IReflectionImporter` are not aware of this, and will
end up creating invalid references making the post-processed assembly unusable.
In addition, post-processing tools may have trouble finding the files for
referenced assemblies, as the process used by MSBuild is complex and poorly
documented.

This gist provides a helper class which solves most of these problems.
It relies on the list of referenced assemblies created by MSBuild's
compilation targets to find the assembly files and to handle both mixed-TFM
dependencies (when dependencies of the assembly being post-processed belong
to different TFMs) and cross-TFM operation (when the TFM the tool is running
in differs from the TFM of the assembly being post-processed).
The importer implementations follow type forwarders in facade assemblies
which MSBuild's `ResolveAssemblyReferences` task adds to the list of references
in mixed-TFM scenarios to produce metadata references which are valid in the
post-processed assembly. The reflection importer handles cross-TFM imports
by mapping framework types to `netstandard` prior to applying the mixed-TFM
import logic. The assembly resolver looks up assembly files using the MSBuild
reference list and thus has access to the same assemblies as the compiler does.

The helper can be used in both standalone tools and MSBuild tasks.
For standalone tools, you will have to pass the list of assembly paths
and fusion names collected by MSBuild in `ReferencePath` items to your tool
through the command line or a temporary file. MSBuild tasks can take the items
directly as a task property; in this case, define the `WITH_MSBUILD_FRAMEWORK`
preprocessor constant in your tool project and use the constructor taking
`ITaskItem[]`.

Limitations
-----------

* Requires Mono.Cecil 0.10 or later (for [jbevain/cecil#470](https://github.com/jbevain/cecil/pull/470)).

* The reflection importer currently requires that a `netstandard` reference
be present, either added explicitly to compilation inputs as a `Reference` item,
or via one of the (transitively closed) dependencies being `netstandard`.

* The helper detects facade assemblies by content because MSBuild does not always
set the `Facade` metadata on `ReferencePath` items, and there appears to be no
custom attribute distinguishing facade assemblies.

References
----------

[jbevain/cecil#487](https://github.com/jbevain/cecil/issues/487),
[jbevain/cecil#505](https://github.com/jbevain/cecil/issues/505),
[jbevain/cecil#526](https://github.com/jbevain/cecil/issues/526)
