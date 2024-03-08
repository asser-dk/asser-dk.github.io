# Blazor

[Blazor](https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor) is one of dotnets latest web app frameworks.

## Cache invalidation strategies

### Use the ModuleVersionId as a version query string

No matter if you are loading a static file, or dynamically loading a collocated js or css file in a component cache invalidation can be a headache.

A reliable strategy that *I* found is to use the Module Version Id (MVID) as a `version` query parameter.

Say I have a component `Foo.razor` living in a RCL (Component Library) project. It has a collocated `Roo.razor.js` file that I want to dynamically load.

```csharp
// Snippet of Foo.razor (or Foo.razor.cs for that matter)

// Reference to the Javascript module/file
private IJSObjectReference? module;

[Inject]
public IJSRuntime JsRuntime { get; set; } = default!;

protected override async Task OnAfterRenderAsync(bool firstRender)
{
    await base.OnAfterRenderAsync(firstRender);

    if (firstRender)
    {
        module = await JsRuntime.InvokeAsync<IJSObjectReference>("import", $"./_content/Foo.razor.js?version={GetCacheVersion()}");
    }
}

private static string GetCacheVersion()
    => Assembly.GetAssembly(typeof(Foo))!.ManifestModule.ModuleVersionId.ToString(); // I chose to use the current component as the type identifier for the assembly but you can use whatever.
```

And a lot of this can be replaced by this small utility extension you can then use in all your projects:

```csharp
public static class IJSRuntimeExtensions
{
    /// <summary>
    /// Asynchronously imports a JavaScript file with an appended version query string set to the assembly module version id of the specified component type.
    /// </summary>
    /// <typeparam name="TComponent">The type of the component used to determine the assembly for generating the version query string.</typeparam>
    /// <param name="jsRuntime">The <see cref="IJSRuntime"/> instance used to invoke the JavaScript import function.</param>
    /// <param name="path">The relative path to the JavaScript file. For example, <c>"./_content/Components/Foo.razor.js"</c>.</param>
    /// <returns>A <see cref="Task"/> that completes with an <see cref="IJSObjectReference"/> representing the imported JavaScript module.</returns>
    public static ValueTask<IJSObjectReference> ImportAsync<TComponent>(this IJSRuntime jsRuntime, string path)
    {
        var version = Assembly.GetAssembly(typeof(TComponent))!.ManifestModule.ModuleVersionId;

        return jsRuntime.InvokeAsync<IJSObjectReference>("import", $"{path}?version={version}");
    }
}
```

You would also be able to use this in a `<script>` or `<link>` tag in your layout or `.cshtml` file:

```html
<link href="css/site.css?version=@(Assembly.GetAssembly(typeof(Program))!.ManifestModule.ModuleVersionId.ToString())" rel="stylesheet" />
```

And the same approach for a `<script>` tag would also work. (I chose to use `Program.cs` as the "assembly pointer" in this case but again change it to whatever suits your needs.)

This approach works great (for me) because:

* It works for both `<script>` tags and `IJSRuntime.InvokeAsync` invocations
* It works the same no matter if your components live in a RCL (Razor (Blazor) Component Library) or directly in a blazor site/app
* You can actually take advantage of browser caches instead of voiding them using either the "current timestamp" or a new guid
* It's faster than using a hash of the static files
* If you enable [Deterministic Compilation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/code-generation#deterministic) the MVID remains the same for repeated builds of the same source code, ensuring the cache is invalidated less often

Some downsides that I see:

* Low cache granularity - Since the MVID changes with **any** modification to a project you migth cause a higher number of unnecessary cache invalidations. Having your components in their own separate RCL projects can reduce this however.
* 



#### Deterministic Compilation

As mentioned above, if you enable [Deterministic Compilation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/code-generation#deterministic) the MVID will remain the same over repeated builds as long as the "input" (for the entire project - see link) remains the same. Without this MSBuild will also include the current timestamp 

 (You should probably have deterministic compilation enabled by default in most cases anyways.)