---
layout: post
title:  "Building C++/WinRT apps that consume C# components"
date:   2019-06-29 00:00:00 -0700
categories: Windows WinRT C++ C# Development
---
_Disclaimer: I'm not by any means a MSBuild expert and some of the rules described here were originally written by people with deeper understanding of .NET Native/CoreCLR. However I'm not aware of any guide existing for enabling this specific scenario and I've already seen multiple people facing this same problem, so I'm documenting it here._s

## The context

The Windows Runtime is cool. It allows developers to weave together native code (C++/CX or the newer C++/WinRT), .NET managed code (C# or VB.NET) and even JavaScript/TypeScript, passing data around and sharing code in a language-agnostic way.

The clearest example are [the WinRT Windows APIs](https://docs.microsoft.com/en-us/uwp/api/): you don't need to worry at all about what language they are implemented in since WinRT always exposes APIs that "look native" and use the standard types and patterns of the language you are using. But this also applies to any components you create, for example if you write a component in C++/WinRT and consume it inside the JavaScript code of a Hosted Web App or a WebView then `hstring`s will be exposed as standard JavaScript strings, `IVector`s as arrays…

I don't know if there is any hard usage data about how people use WinRT components in the real world, but according to my own observations it seems most usage falls under two buckets:

   - The WinRT component and the consumer are written in the same language, and WinRT Components are primarily used as a tool to divide the code in modules, facilitating code reuse and allowing to define proper boundaries between components, rather than as a language interoperability tool.
   - The WinRT code is written in a "lower" level language than the consumer, and WinRT is used to allow the higher level language to invoke the lower level code. For example, a .NET app might consume UI controls written in C++, or a web app might delegate computationally intensive tasks to a C++ component.

While it makes sense that these are the most common scenarios, there is a less common situation which is also interesting: the need to consume "higher level components" from a native app. C#/.NET has a rich ecosystem of libraries, and many of those have no C++ equivalents (e.g. the Microsoft Graph SDK or the Azure Cognitive Services SDK), so being able to pull those in should make the life of the C++/WinRT developer way easier… or at least that is the theory.

## The problem

The Windows Runtime is perfectly capable of activating and using managed modules from native code, but there seems to be a gap in the tooling: the default project created by Visual Studio won't copy all the necessary files when building your solution. This issue surfaces when you reference a C# component from the C++/WinRT app and try to invoke one of its methods:

![ERROR_FILE_NOT_FOUND](/assets/winrt-error-filenotfound.png)

## The solution

### Copying the .NET Native / Core CLR files

The _ERROR_FILE_NOT_FOUND_ error gives us a hint about the underlying issue: the .NET Core Runtime/.NET Native files haven't been pulled in automatically into the build output, and the app is failing to load them. In order to fix that, we need to modify the project file to import the missing .props and .targets files. [You can learn more about the role of those files here](https://devblogs.microsoft.com/nuget/native-support/), the TL;DR version is that they define additional MSBuild steps that are executed before and after the steps defined in the .vcxproj file.

We need to open the <Project name>.vcxproj file and make some additions to ensure that those files get copied as part of the build process. First, we need to define some additional properties for the SDK version. Just before the </PropertyGroup> line, paste the following lines:

```
<!-- Start Custom .NET Native properties -->
<DotNetNativeVersion>2.2.3</DotNetNativeVersion>
<!-- The name 'DotNetNativeVersion' is critical for restoring the right .NET framework libraries -->
<UWPCoreRuntimeSdkVersion>2.2.8</UWPCoreRuntimeSdkVersion>
<!-- Unlike 'DotNetNativeVersion' this property is only used for convenience -->
<!-- End Custom .NET Native properties -->
```

(You might need to tweak the version numbers depending on what SDK version you are using)

Then, after the </PropertyGroup> line, paste the references to the .props files:

```
<!-- Start Custom .NET Native targets -->
<!-- Import all of the .NET Native / CoreCLR props at the beginning of the project -->
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\Microsoft.Net.UWPCoreRuntimeSdk\$(UWPCoreRuntimeSdkVersion)\build\Microsoft.Net.UWPCoreRuntimeSdk.props" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x86.Microsoft.Net.UWPCoreRuntimeSdk\$(UWPCoreRuntimeSdkVersion)\build\runtime.win10-x86.Microsoft.Net.UWPCoreRuntimeSdk.props" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x64.Microsoft.Net.UWPCoreRuntimeSdk\$(UWPCoreRuntimeSdkVersion)\build\runtime.win10-x64.Microsoft.Net.UWPCoreRuntimeSdk.props" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-arm.Microsoft.Net.UWPCoreRuntimeSdk\$(UWPCoreRuntimeSdkVersion)\build\runtime.win10-arm.Microsoft.Net.UWPCoreRuntimeSdk.props" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\Microsoft.Net.Native.Compiler\$(DotNetNativeVersion)\build\Microsoft.Net.Native.Compiler.props" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x86.Microsoft.Net.Native.Compiler\$(DotNetNativeVersion)\build\runtime.win10-x86.Microsoft.Net.Native.Compiler.props" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x64.Microsoft.Net.Native.Compiler\$(DotNetNativeVersion)\build\runtime.win10-x64.Microsoft.Net.Native.Compiler.props" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-arm.Microsoft.Net.Native.Compiler\$(DotNetNativeVersion)\build\runtime.win10-arm.Microsoft.Net.Native.Compiler.props" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x86.Microsoft.Net.Native.SharedLibrary\$(DotNetNativeVersion)\build\runtime.win10-x86.Microsoft.Net.Native.SharedLibrary.props" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x64.Microsoft.Net.Native.SharedLibrary\$(DotNetNativeVersion)\build\runtime.win10-x64.Microsoft.Net.Native.SharedLibrary.props" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-arm.Microsoft.Net.Native.SharedLibrary\$(DotNetNativeVersion)\build\runtime.win10-arm.Microsoft.Net.Native.SharedLibrary.props" />
<!-- End Custom .NET Native targets -->
```

And at the very end, before the </Project> line, paste the references to the .targets files:

```
<!-- Start Custom .NET Native targets -->
<!-- Add a workaround to make sure the .NET framework libraries are correctly copied to the AppX folder in Debug (CoreCLR) mode -->
<Target Name="AfterInjectNetCoreFramework" AfterTargets="InjectNetCoreFramework">
<ItemGroup>
<PackagingOutputs Include="@(_InjectNetCoreFrameworkPayload)" Condition="'%(_InjectNetCoreFrameworkPayload.NuGetPackageId)' == '$(_CoreRuntimePackageId)' and '$(UseDotNetNativeToolchain)' != 'true'">
<TargetPath>%(Filename)%(Extension)</TargetPath>
<ProjectName>$(ProjectName)</ProjectName>
<OutputGroup>CopyLocalFilesOutputGroup</OutputGroup>
</PackagingOutputs>
</ItemGroup>
</Target>
<!-- Import all of the .NET Native / CoreCLR targets at the end of the project -->
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x86.Microsoft.Net.UWPCoreRuntimeSdk\$(UWPCoreRuntimeSdkVersion)\build\runtime.win10-x86.Microsoft.Net.UWPCoreRuntimeSdk.targets" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x64.Microsoft.Net.UWPCoreRuntimeSdk\$(UWPCoreRuntimeSdkVersion)\build\runtime.win10-x64.Microsoft.Net.UWPCoreRuntimeSdk.targets" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-arm.Microsoft.Net.UWPCoreRuntimeSdk\$(UWPCoreRuntimeSdkVersion)\build\runtime.win10-arm.Microsoft.Net.UWPCoreRuntimeSdk.targets" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\Microsoft.Net.Native.Compiler\$(DotNetNativeVersion)\build\Microsoft.Net.Native.Compiler.targets" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x86.Microsoft.Net.Native.Compiler\$(DotNetNativeVersion)\build\runtime.win10-x86.Microsoft.Net.Native.Compiler.targets" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x64.Microsoft.Net.Native.Compiler\$(DotNetNativeVersion)\build\runtime.win10-x64.Microsoft.Net.Native.Compiler.targets" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-arm.Microsoft.Net.Native.Compiler\$(DotNetNativeVersion)\build\runtime.win10-arm.Microsoft.Net.Native.Compiler.targets" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x86.Microsoft.Net.Native.SharedLibrary\$(DotNetNativeVersion)\build\runtime.win10-x86.Microsoft.Net.Native.SharedLibrary.targets" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-x64.Microsoft.Net.Native.SharedLibrary\$(DotNetNativeVersion)\build\runtime.win10-x64.Microsoft.Net.Native.SharedLibrary.targets" />
<Import Condition="'$(WindowsTargetPlatformMinVersion)' &gt;= '10.0.16299.0'" Project="$(ProgramFiles)\Microsoft SDKs\UWPNuGetPackages\runtime.win10-arm.Microsoft.Net.Native.SharedLibrary\$(DotNetNativeVersion)\build\runtime.win10-arm.Microsoft.Net.Native.SharedLibrary.targets" />
<!-- End Custom .NET Native targets -->
```

Once we are done, we can reload the solution and rebuild the app. If you don't have any NuGet dependencies on your C# code this will be enough for the app to work. However if there are additional dependencies you'll need to make sure those are copied too.

### Copying the NuGet .dll files

Now our managed code is now running, but it is failing to load the .dll from our 3rd party NuGet dependency. (This might manifest as the exact same error or an error in a different location, depending on how early your library is loaded in the C# code).

Unless you specified otherwise in your project / Visual Studio configuration, you can find the missing .dll files in _%userprofile%\.nuget\packages_, under the subfolders that match your package id and version, E.g. _%userprofile%\.nuget\packages\newtonsoft.json\12.0.2\lib\netstandard2.0\Newtonsoft.Json.dll_.

We have multiple options for getting those files into the app, the simplest option would be to paste the files manually into the build output and into the publish packages, but if we are going to be building the app both locally and in a CI pipeline we probably want to integrate that into the MSBuild process.

In order to do that, what I usually do is create a _nugetPackages_ folder in the project location, copy the .dll files there and commit them to the git repo. This is not the cleanest way of ensuring the files are available everywhere, but it is definitely the simplest. An alternative approach would be to have an script that downloads the NuGet packages and extracts and copies the .dll files to a temporary folder, and run that script as part of the build process both locally and in the CI build.

Once we have the files there, we can add the necessary rules to the .vcxproj file to ensure they are copied as part of the build. At the end, just before the Project closing tag, add an ItemGroup containing one entry per .dll file:

```
<ItemGroup>
	<None Include="nugetDependencies\Newtonsoft.Json.dll">
		<Link>%(Filename)%(Extension)</Link>
		<DeploymentContent>true</DeploymentContent>
	</None>
</ItemGroup>
```

After adding this, try rebuilding and running your app. This time it should run just fine!

### One more tip

For a seamless debugging experience, right click the C++ project, go to Properties>Debugging>Debugger type and select _Managed and Native_. Once you do this the Visual Studio debugger will move seamless between breakpoints present in both solutions.

## To sum up

After following these steps you should be able to fully integrate C# code into your C++/WinRT app and build your app locally, on CI pipelines, and generate store packages successfully. You will need to remember to update the project file every time you add or update a NuGet dependency.



