---
title:  "Counter App with GTK4 and .NET 8"
date:   2024-01-18
categories:
  - gtk4
  - dotnet
tags:
  - gtk4
  - .NET 8
  - C#
---
## GTK and .NET
GTK is a free and open-source cross-platform widget toolkit for creating graphical user interfaces (GUIs).

.NET is the free, open-source, cross-platform framework for building modern apps and powerful cloud services.

Focus of this tutorial is to write a counter app with GTK 4 and C# targeting .NET 8.

## Project Setup
Let's begin by installing all necessary tools. First, follow the instructions on the [GTK website](https://www.gtk.org/docs/installations/) in order to install GTK 4. Then install .NET by following instructions for your platform from [download](https://dotnet.microsoft.com/en-us/download) page. We are targeting GTK4, .NET 8 and Gir.Core.Gtk-4.0 0.4.0.

Now lets create a new empty folder named `gtk4-dotnet8-counter-app` and execute following to create an empty solution:
```shell
dotnet new sln
```

Next we will add a console application and add that to our solution
```shell
dotnet new console -o Counter.App
dotnet sln add Counter.App/Counter.App.csproj
```
New lets add C# bindings for Gtk4 to our `Counter.App` project
```shell
cd Counter.App
dotnet add package GirCore.Gtk-4.0 --version 0.4.0
```
Now we can run our application by executing:
```shell
dotnet run
```
At this moment it would print `Hello, world!`.

## Application
Lets start by creating GTK Application and add an `OnActivate` event handler to create the application window.

```csharp
var application = Gtk.Application.New("org.GirCore.GTK4Counter", Gio.ApplicationFlags.FlagsNone);
application.OnActivate += (sender, args) =>
{
    var window = Gtk.ApplicationWindow.New((Gtk.Application)sender);
    window.Title = "GTK Counter App";
    window.SetDefaultSize(300, 300);
    window.Show();
};
return application.RunWithSynchronizationContext();
```

This will dispaly an empty window.
<figure>
  <a href="/assets/images/2024-01-18/01-blank-window.png"><img src="/assets/images/2024-01-18/01-blank-window.png"></a>
  <figcaption>Empty GTK Window</figcaption>
</figure>

## Label
Time to add some content. Lets start by adding a `Label` and displaying `Hello World!` text.

Add following to `OnActivate` event handler.
```csharp
    var labelCounter = Gtk.Label.New("Hello World!");
    labelCounter.SetMarginTop(12);
    labelCounter.SetMarginBottom(12);
    labelCounter.SetMarginStart(12);
    labelCounter.SetMarginEnd(12);
```

And add `labelCounter` as child of window.
```csharp
    var window = Gtk.ApplicationWindow.New((Gtk.Application)sender);
    window.Title = "GTK Counter App";
    window.SetDefaultSize(300, 300);
    window.Child = labelCounter;
    window.Show();
```

Running the application now would display `Hello World!` text in the window.
<figure>
  <a href="/assets/images/2024-01-18/02-hello-world.png"><img src="/assets/images/2024-01-18/02-hello-world.png"></a>
  <figcaption>Hello World</figcaption>
</figure>

## Increment Button
Lets create a button
```csharp
    var buttonIncrease = Gtk.Button.New();
    buttonIncrease.Label = "Increase";
    buttonIncrease.SetMarginTop(12);
    buttonIncrease.SetMarginBottom(12);
    buttonIncrease.SetMarginStart(12);
    buttonIncrease.SetMarginEnd(12);
```
Now that we have multiple widgets, we would add a `Gtk.Box` to hold all the child elements and add that `Box` as child to window instead.
```csharp
    var gtkBox = Gtk.Box.New(Gtk.Orientation.Vertical, 0);
    gtkBox.Append(labelCounter);
    gtkBox.Append(buttonIncrease);

    ...
    window.Child = gtkBox;
    ...
```
<figure>
  <a href="imag/assets/images/2024-01-18es/03-increase-button.png"><img src="/assets/images/2024-01-18/03-increase-button.png"></a>
  <figcaption>Increase Button</figcaption>
</figure>

## Add Click Handler
Lets add a counter and button click handler. We will update the counter and update the `labelCounter` with the value. 

Variable declare will look like following and set label value.
```csharp
    var counter = 0;
    
    var labelCounter = Gtk.Label.New(counter.ToString());
    ...
```

Lets add a click handler for the `buttonIncrease`.
```csharp
    buttonIncrease.OnClicked += (_, _) =>
    {
        counter++;
        labelCounter.SetLabel(counter.ToString());
    };
```
<figure>
  <a href="/assets/images/2024-01-18/04-incremented-value.png"><img src="/assets/images/2024-01-18/04-incremented-value.png"></a>
  <figcaption>Incremented Value</figcaption>
</figure>

## Add Decrease Button and Handler
Lets add another button to decrease the value and update label.
```csharp
    var buttonDecrease = Gtk.Button.New();
    buttonDecrease.Label = "Decrease";
    buttonDecrease.SetMarginTop(12);
    buttonDecrease.SetMarginBottom(12);
    buttonDecrease.SetMarginStart(12);
    buttonDecrease.SetMarginEnd(12);
    buttonDecrease.OnClicked += (_, _) =>
    {
        counter--;
        labelCounter.SetLabel(counter.ToString());
    };
```
<figure>
  <a href="/assets/images/2024-01-18/05-counter-app.png"><img src="/assets/images/2024-01-18/05-counter-app.png"></a>
  <figcaption>Counter App</figcaption>
</figure>

## Source
Source code for the demo application is hosted on GitHub in [blog-code-samples](https://github.com/kashifsoofi/blog-code-samples/tree/main/gtk4-dotnet8-counter-app) repository.

## References
In no particular order
* [GTK](https://www.gtk.org/)
* [GTK Installation](https://www.gtk.org/docs/installations/)
* [.NET](https://dotnet.microsoft.com/en-us/)
* [.NET Download](https://dotnet.microsoft.com/en-us/download)
* [Gir.Core](https://github.com/gircore/gir.core)
* [GirCore.Gtk-4.0](https://www.nuget.org/packages/GirCore.Gtk-4.0/)
* [Gir.Core Gtk-4.0 Samples](https://github.com/gircore/gir.core/tree/main/src/Samples/Gtk-4.0)
* And many more