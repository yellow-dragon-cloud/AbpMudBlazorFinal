# MudBlazor Theme in ABP Blazor WebAssembly FINAL

## Introduction

In this part, I will show you how to customize/override pre-built Blazor pages of [Idendity Module](https://docs.abp.io/en/abp/latest/Modules/Identity) using [MudBlazor](https://www.mudblazor.com/) components. I assume that Tenant and Setting management pages are rarely used by end-users, so these modules are not covered in this sample to keep it short. You can customize every page using the principles shown here. All you need to do is:
1. Derive a new razor page/class which inherits the class you want to override
2. Add the following attributes to the derived class:
```razor
@* to the razor file *@
@attribute [ExposeServices(typeof(BASE-CLASS))]
@attribute [Dependency(ReplaceServices = true)]
```
```charp
// or to the C# file
[ExposeServices(typeof(BASE-CLASS))]
[Dependency(ReplaceServices = true)]
```

See [Blazor UI: Customization / Overriding Components](https://docs.abp.io/en/abp/latest/UI/Blazor/Customization-Overriding-Components) for more information.

## 16. Make Sure You Have Completed Previous Parts

You must complete the steps in [Part 1](https://github.com/yellow-dragon-cloud/AbpMudBlazor), [Part 2](https://github.com/yellow-dragon-cloud/AbpMudBlazor2), [Part 3](https://github.com/yellow-dragon-cloud/AbpMudBlazor3), and [Part 4](https://github.com/yellow-dragon-cloud/AbpMudBlazor4) to continue.
