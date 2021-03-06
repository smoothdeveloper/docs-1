# Managing Resource Dependencies

WebSharper automates the management of resource dependencies. For the purposes
of WebSharper, a __resource__ is any HTML code that can be rendered to the
`<head>` section of a page. Most commonly, this will be a `<link>` tag pointing
to a CSS file, or a `<script>` tag pointing to a JavaScript reference. Pages
written with WebSharper infer their minimal necessary resource set and the
correct resource ordering.

Most resources are declared as subclasses of the `Core.Resources.BaseResource`
class (see "Declaring Resources and Dependencies" below for the different
ways to declare a resource). Such a resource is declared with an associated
path. The path is resolved by WebSharper as follows:

<a name="embedded"></a>
1. If the assembly contains an embedded resource file whose name
corresponds to the resource's path, and the following assembly-wide
attribute is present:

    ```fsharp
    [<assembly:System.Web.UI.WebResource("myfile.js", "mime/type")>]
    do ()
    ```

    then WebSharper

    * extracts this file into the relevant subfolder of your application
      (`/Scripts/WebSharper/<path>` for JavaScript files,
      `/Content/WebSharper/<path>` for other resources)

    * adds the correct tag to `<head>` (`<script>` for JavaScript files,
      `<link>` for CSS files) with the corresponding path.
      
2. If there is no such embedded resource file, then WebSharper simply
adds the correct tag to `<head>` without altering the path. This means
that for such resources, you *should* use absolute URLs, either inside
the application (for example `/some/path.js`) or to an external website
(for example `//my.cdn.net/file1.js`).

A resource of type `BaseResource` can also be declared with a base path
and a set of subpaths. This is useful for a library consisting of several
files that need to be loaded eg. from a CDN. In this case, step 1 above
is skipped, and a tag is added for each subpath by combining it with the
base path.

## Dependency resolution

A resource is only included by WebSharper if it is required by a
client-side element. This means that each page of your website only
contains the minimum set of resources that its contents need.

An assembly, a type, a module, a static member or a module `let`
declaration can be marked as requiring a resource (see "Declaring
Resources and Dependencies" below for the different ways to declare
a dependency). Any page that calls this item's client-side code
will have the given resource included.

A resource B can also be required by another resource A. In this case,
any code that requires A will also include B. WebSharper ensures that
B is located before A in the `<head>`.

## Declaring Resources and Dependencies

### In WebSharper Interface Generator

To declare a resource in WIG, you can use one of the following
functions:

* `Resource` declares a `BaseResource` with a single path.

```fsharp
let R1 = Resource "ResourceClassName" "path.js"
```

* `Resources` declares a `BaseResource` with a base path and multiple
subpaths.

```fsharp
let R2 =
    Resources "ResourceClassName2"
        "//my.cdn.net" ["file1.js"; "file2.js"; "file1.css"]
```

In either case, your resource must be included in the `Assembly`
declaration. A common idiom is to create a sub-namespace called
`Resources` and to include all resources in it:

```fsharp
let Assembly =
    Assembly [
        Namespace "My.Library" [
            // Library classes...
        ]
        Namespace "My.Library.Resources" [
            R1
            R2
        ]
    ]
```

To declare that a class or an assembly depends on a given resource,
you can use one of the following functions:

* `Requires` declares a dependency on resources declared in this
assembly.

```fsharp
let C =
    Class "My.Library.C"
    |+> (* members... *)
    |> Requires [R1; R2]
    
let Assembly =
    Assembly [
        // ...
    ]
    |> Requires [R3]
```

* `RequiresExternal` declares a dependency on resources declared
outside of this assembly.

```fsharp
let C2 =
    Class "My.Library.C2"
    |+> (* members... *)
    |> RequiresExternal [typeof<Other.Library.Resources.R4>]
```

### In Client-side WebSharper code

To declare a resource in a WebSharper library or application,
you can simply declare a class inheriting from `BaseResource`.
Use the constructor with a single argument for single paths,
and multiple arguments for a base path and a set of subpaths.

```fsharp
module Resources =

    open WebSharper.Core.Resources

    type R1() =
        inherit BaseResource("path.js")
        
    type R2() =
        inherit BaseResource("//my.cdn.net",
            "file1.js", "file2.js", "file3.css")
```

You can also implement more complex resources (for example,
resources that require a bit of inline JavaScript) by directly
implementing the `IResource` interface. You can emit arbitrary
HTML in the `Render` method using the provided `HtmlTextWriter`.

```fsharp
type R3() =
    interface R.IResource with
        member this.Render ctx writer = ...
```

A resource dependency can be declared on a type, a member or an
assembly by annotating it with `RequireAttribute`. It is parameterized
by the type of the resource to require:

```fsharp
[<Require(typeof<R1>)>]
type MyWidget() = ...

[<Require(typeof<R2>)>]
let F x = ...

[<assembly:Require(typeof<R3>)>]
do()
```

### In Server-side WebSharper code

Sometimes you might need to depend on a resource without having any client-side
code; typically, a CSS file. In this case, you can add the web control
`WebSharper.Web.Require` anywhere in your page. This control does not directly
output any HTML at the location where you put it, but it incurs a dependency on
the resource that you pass to it.

Here is an example UI.Next page with a dependency on `R1` from above:

```fsharp
let MyPage() =
    Content.Page(
        div [
            h1 [text "This page includes the script at `path.js`."]
            Doc.WebControl(new Web.Require<R1>())
        ]
    )
```

## Resource Implementation

The resource dependency graphs are constructed for every WebSharper-processed
assembly and are serialized to binary. They are stored within the
assembly itself.  At runtime all the graphs of all the referenced assemblies
are deserialized and merged into a single graph.

## Hashes on resource links

WebSharper computes a hash of all generated files, and files embedded and referred to from a `WebResource` attribute [(see above)](#embedded). These hashes are added as query parameters on script links generated by the Sitelets runtime to avoid browsers caching an outdated version.
 
<a name="override"></a>
## Overriding Resource URLs

External resources implemented using `BaseResource` can have their URL
overridden on a per-application basis. For example, you can force your
application to use a different version of JQuery than the one used by default
by WebSharper.

This configuration is done in the application configuration file. As a
standard, for self-hosted projects, that file is called `App.config`, and for
other types of web projects, it's `Web.config`. WebSharper recognizes both, and
uses `Web.config` if both are present.

To override a given resource URL, simply add an appSetting whose key is the
fully qualified name of the resource, and whose value is the URL you want to
use. For example, to tell WebSharper to use a local copy of JQuery located at
the root of your application, you can add the following to your application
configuration file:

```xml
<configuration>
  <appSettings>
    <add key="WebSharper.JQuery.Resources.JQuery" value="/jquery.js" />
    <!-- ... -->
```

Note that the fully qualified name is in IL format. This means that nested
types and types located inside F# modules are separated by `+` instead of `.`.

If you are a library author and have a resource declared by directly
implementing the `IResource` interface, you can make it configurable by using
the function `ctx.GetSetting`, which retrieves the setting with a given key
from the application configuration file.

<a name="cdn"></a>
## Using CDN

You can automatically point to CDN for the WebSharper core libraries. Add the following setting to your application configuration file (Web.config or App.config depending on the project type):

```xml
<appSettings>
  <add key="WebSharper.StdlibUseCdn" value="true" />
</appSettings>
```

The links generated by WebSharper, instead of pointing to `/Scripts/WebSharper/...` for scripts and `/Content/WebSharper/...` for CSS, will point to `//cdn.websharper.com/{assembly}/{version}/{filename}`. You can configure this URL by setting the `WebSharper.StdlibCdnFormat` configuration setting. And finally, you can configure the CDN URL for the resources of a specific assembly (from the standard WebSharper library or not) by setting the `WebSharper.CdnFormat.{assemblyname}` configuration setting.
