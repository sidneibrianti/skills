# simple-packages-config

A .NET Framework project that uses `packages.config` for NuGet package management instead of `PackageReference`. The agent should detect this and inform the user that CPM requires `PackageReference` format, declining to proceed with the conversion.

## Setup

```
📂 LegacyApp/
├── 📄 LegacyApp.csproj
└── 📄 packages.config
```

`LegacyApp.csproj`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" />
  <PropertyGroup>
    <TargetFrameworkVersion>v4.8</TargetFrameworkVersion>
    <OutputType>Library</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <Reference Include="Newtonsoft.Json, Version=13.0.0.0">
      <HintPath>..\packages\Newtonsoft.Json.13.0.3\lib\net45\Newtonsoft.Json.dll</HintPath>
    </Reference>
  </ItemGroup>
  <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
</Project>
```

`packages.config`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <package id="Newtonsoft.Json" version="13.0.3" targetFramework="net48" />
</packages>
```

## Input prompt

Convert my LegacyApp project to Central Package Management.

## What the skill should produce

- The agent detects `packages.config` and recognizes the project does not use `PackageReference` format
- The agent informs the user that CPM requires `PackageReference` and cannot be applied to `packages.config` projects
- The agent suggests migrating from `packages.config` to `PackageReference` first (using Visual Studio)
- The agent does **not** attempt to create `Directory.Packages.props` or modify any files
