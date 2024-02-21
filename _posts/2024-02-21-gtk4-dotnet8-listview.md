---
title:  "GTK4 ListView with .NET 8"
date:   2024-02-21
categories:
  - gtk4
  - dotnet
tags:
  - gtk4
  - .NET 8
  - C#
---
## GtkListView
GTK 4 has added new widgets for to display lists namely GtkListView, GtkGridView and GtkColumnView. [Here](https://toshiocp.github.io/Gtk4-tutorial/sec29.html) is an excellent article by [ToshioCP](https://github.com/ToshioCP) about GtkListView and an [article](https://blog.gtk.org/2020/06/07/scalable-lists-in-gtk-4/) by Matthias Clasen on why the new widgets were added to GTK 4.

In my search I could not find any guide or getting started with GtkListView in .NET. This post is an attempt to do that and also a reference to what I did :).

## Project Setup
Let's start by creating a directory and run following command to create a new solution, project and add project to solution.
```shell
dotnet new sln --name ListView
dotnet new console -o ListView.App
dotnet sln add ListView.App/ListView.App.csproj
```

Next let's add a reference to `GirCore.Gtk-4.0` nuget package.
```shell
cd ListView.App
dotnet add package GirCore.Gtk-4.0 --version 0.5.0-preview.4
```

## Application
Let's start by adding an empty `GtkApplicationWindow`. Add following code in `Program.cs`
```csharp
﻿var application = Gtk.Application.New("org.kashif-code-samples.listview.sample", Gio.ApplicationFlags.FlagsNone);
application.OnActivate += (sender, args) =>
{
    var window = Gtk.ApplicationWindow.New((Gtk.Application) sender);
    window.Title = "ListView Sample";
    window.SetDefaultSize(300, 300);
    window.Show();
};
return application.RunWithSynchronizationContext(null);
```

Running the application, will display an empty window.
<figure>
  <a href="/assets/images/2024-02-21/01-empty-window.png"><img src="/assets/images/2024-02-21/01-empty-window.png"></a>
  <figcaption>Empty Window</figcaption>
</figure>  

## No Selection ListView
Next lets enhance our sample application to add a window that has a simple `ListView` with no selection.
First lets add a new class `CodeListViewWindow` inherting from `Gtk.Window` to our project.
`ListView` will need 2 objects,
* SelctionModel - is an interface to support selection of items
* ListItemFactory - creates or recycles GtkListItem and connects it with an item of the list model.
We are going to use `NoSelection`, `SelectionModel` needs a `ListStore` and we are going to use `StringList` to display a list of strings.
For `ListItemFactory` we are going to use `SignalListItemFactory` and setup `OnSetup` and `OnBind` signal handlers to create `Label` and bind data to that `Label`.

You might get a warning on `ListView` is a namespace but being used as a type, trick is to add `using Gtk;` statement after namespace or fully qualify `ListView`.

Here is the code for the class.
```csharp
public class CodeListViewWindow : Window
{
    public CodeListViewWindow()
        : base()
    {
        Title = "No Selection ListView";
        SetDefaultSize(300, 300);

        var stringList = StringList.New(["One", "Two", "Three", "Four"]);
        var selectionModel = NoSelection.New(stringList);
        var listItemFactory = SignalListItemFactory.New();
        listItemFactory.OnSetup += SetupSignalHandler;
        listItemFactory.OnBind += BindSignalHandler;

        var listView = ListView.New(selectionModel, listItemFactory);

        var scrolledWindow = ScrolledWindow.New();
        scrolledWindow.Child = listView;
        Child = scrolledWindow;
    }

    private void SetupSignalHandler(SignalListItemFactory sender, SetupSignalArgs args)
    {
        var listItem = args.Object as ListItem;
        if (listItem is null)
        {
            return;
        }

        var label = Label.New(null);
        listItem.Child = label;
    }

    private void BindSignalHandler(SignalListItemFactory sender, BindSignalArgs args)
    {
        var listItem = args.Object as ListItem;
        if (listItem is null)
        {
            return;
        }
        
        var label = listItem.Child as Label;
        if (label is null)
        {
            return;
        }

        var item = listItem.Item as StringObject;
        label.SetText(item?.String ?? "NOT FOUND");
    }
}
```

Next step is to add a button on our application window, then we will add a click handler to open a new window that has a no selection list view.
Update the `OnActivate` handler as follows.
```csharp
application.OnActivate += (sender, args) =>
{
    var buttonShowCodeListView = CreateButton("Show Code ListView Window");
    buttonShowCodeListView.OnClicked += (_, _) => new CodeListViewWindow().Show();

    var gtkBox = Gtk.Box.New(Gtk.Orientation.Vertical, 0);
    gtkBox.Append(buttonShowCodeListView);

    var window = Gtk.ApplicationWindow.New((Gtk.Application) sender);
    window.Title = "ListView Sample";
    window.SetDefaultSize(300, 300);
    window.Child = gtkBox;
    window.Show();
};
```
And helper method to create the button.
```csharp
static Gtk.Button CreateButton(string label)
{
    var button = Gtk.Button.New();
    button.Label = label;
    button.SetMarginTop(12);
    button.SetMarginBottom(12);
    button.SetMarginStart(12);
    button.SetMarginEnd(12);
    return button;
}
```

Running the application will show our application window with a button.  
<figure>
  <a href="/assets/images/2024-02-21/02-app-window-with-button.png"><img src="/assets/images/2024-02-21/02-app-window-with-button.png"></a>
  <figcaption>Application Window</figcaption>
</figure>  

And clicking on that button will open a new window with a simple list as follows.  
<figure>
  <a href="/assets/images/2024-02-21/03-code-list-view-window.png"><img src="/assets/images/2024-02-21/03-code-list-view-window.png"></a>
  <figcaption>Code List View Window</figcaption>
</figure>  

## BuilderListItemFactory
We can also use the `BuilderListItemFactory` instead of `SignalListItemFactory`, this allows us the define the behaviour in ui file instead of writing the signal handlers in code.
I have added the following ui file copied from [article](https://toshiocp.github.io/Gtk4-tutorial/sec29.html) by TochioCP.
```xml
<interface>
  <template class="GtkListItem">
    <property name="child">
      <object class="GtkLabel">
        <binding name="label">
          <lookup name="string" type="GtkStringObject">
            <lookup name="item">GtkListItem</lookup>
          </lookup>
        </binding>
      </object>
    </property>
  </template>
</interface>
```

Next I have added another class `TemplateListViewWindow` with following content. `ListItemTemplate.ui` file is set as Embedded Resource and for `ReadResourceAsByteArray` we would need to add `using GObject;`.
```csharp
using System.Reflection;
using Gtk;
using GObject;
using GLib;

public class TemplateListViewWindow : Window
{
    public TemplateListViewWindow()
        : base()
    {
        Title = "Template ListView";
        SetDefaultSize(300, 300);

        var stringList = StringList.New(["One", "Two", "Three", "Four"]);
        var selectionModel = SingleSelection.New(stringList);
        var bytes = Assembly.GetExecutingAssembly()
            .ReadResourceAsByteArray("ListItemTemplate.ui");
        var listItemFactory = BuilderListItemFactory.NewFromBytes(null, Bytes.New(bytes));
        var listView = ListView.New(selectionModel, listItemFactory);

        var scrolledWindow = ScrolledWindow.New();
        scrolledWindow.Child = listView;
        Child = scrolledWindow;
    }
}
```

And I have added a button to show this new window on our application window in `Program.cs`.
```csharp
    var buttonShowTemplateListView = CreateButton("Show Template ListView Window");
    buttonShowTemplateListView.OnClicked += (_, _) => new TemplateListViewWindow().Show();
    ...
    gtkBox.Append(buttonShowTemplateListView);
```
Window with ListItem template looks like following  
<figure>
  <a href="/assets/images/2024-02-21/04-template-list-view.png"><img src="/assets/images/2024-02-21/04-template-list-view.png"></a>
  <figcaption>Window with ListItem Template</figcaption>
</figure>  

## Wrapup
Here is how you can use Gtk ListView in your application, hopefully it would be a good starting point for anyone looking to start using ListView in C# with code or template.

## Source
Source code for the sample application is available on GitHub in [gtk4-dotnet8-list-view](https://github.com/kashif-code-samples/gtk4-dotnet8-list-view) repository.

## References
In no particular order
* [Gtk-4.0: List Widget Overview](https://docs.gtk.org/gtk4/section-list-widget.html)
* [GTK 4 Tutorial](https://toshiocp.github.io/Gtk4-tutorial/sec29.html)
* [Gir.Core](https://github.com/gircore/gir.core)
* [GirCore.Gtk-4.0](https://www.nuget.org/packages/GirCore.Gtk-4.0/)
and many more