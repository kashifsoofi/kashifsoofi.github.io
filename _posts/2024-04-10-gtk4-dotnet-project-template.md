---
title:  "GTK4 `dotnet new` Project Template"
date:   2024-04-10
categories:
  - gtk4
  - dotnet
tags:
  - gtk4
  - .NET 8
  - C#
---
I have been writing some samples using GTK4 using `GirCore` so its time to reduce the manual project setup steps and turn it into a `dotnet new` project template.  

# Project Template
Time to follow the excellent [article](https://devblogs.microsoft.com/dotnet/how-to-create-your-own-templates-for-dotnet-new/) by Syed Ibrahim.

I have taken out the project setup section from my GTK 4 tutorials and turned it into the template. I have created a console application, added `GirCore.Gtk-4.0` nuget package to the project and updated the `Program.cs` to create a main Gtk window with `Hello World` label.  

Next tutorial for GTK4 hopefully I will use this template to bootstrap the project. And I will keep it updated with new releases of `GirCore`.  

## Usage
1. Clone repository locally  
2. Build nuget package  
`dotnet pack -o .`  
3. Install template for `dotnet new` using following command  
`dotnet new install Gtk4.Templates.1.0.0.nupkg`  
4. Create new project using template  
`dotnet new gtk4-app --param:name [ProjectName]`  
OR  
`dotnet new gtk4-app -p:n [ProjectName]`  
5. Uninstall template using following command  
`dotnet new uninstall Gtk4.Templates`  

## Source
Source code for the template is available on GitHub in [gtk4-dotnet-template](https://github.com/kashif-code-samples/gtk4-dotnet-template) repository.  

## References & Resources
* [dotnet new](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-new)  
* [dotnet-template-samples](https://github.com/dotnet/dotnet-template-samples)  
* [How to create your own templates for dotnet new](https://devblogs.microsoft.com/dotnet/how-to-create-your-own-templates-for-dotnet-new/)  
* [templating](https://github.com/dotnet/templating)  
* [Gir.Core](https://github.com/gircore/gir.core)
And many more