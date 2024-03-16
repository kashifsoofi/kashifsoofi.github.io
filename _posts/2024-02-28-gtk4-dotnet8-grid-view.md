---
title:  "GTK4 GridView with .NET 8"
date:   2024-02-28
categories:
  - gtk4
  - dotnet
tags:
  - gtk4
  - .NET 8
  - C#
---
In the previous post [GTK4 ListView with .NET 8](https://kashifsoofi.github.io/gtk4/dotnet/gtk4-dotnet8-listview) I demonstrated how to use GTK4 ListView in a .NET application. In this post I will expoloe `GridView`.

`ListView` shows 1-dimensional list of data with one column. `GridView` shows a 2-dimensional grid.

## Project Setup
Let's start by setting up a new solution and project.
```shell
dotnet new sln --name GridView
dotnet new console -o GridView.App
dotnet sln add GridView.App/GridView.App.csproj
```
And add a reference to `GirCore.Gtk-4.0` nuget package, the latest version is `0.5.0-preview.4` at the time of writing.
```shell
cd GridView.App
dotnet add package GirCore.Gtk-4.0 --version 0.5.0-preview.4
```

## StringListGridViewWindow
Next we will add `StringListGridViewWindow` inheriting from `Gtk.Window`. We will start with a `StringList` ListStore and a `NoSelection` selection model and will use `SignalListItemFactory` to create `Label` for each of the items. I have set the `WidthRequest` and `HeightRequest` property of each label to 100 to demonstrate how the items will look on the window.

```csharp
namespace GridView.App;

using Gtk;
using static Gtk.SignalListItemFactory;

public class StringListGridViewWindow : Window
{
    public StringListGridViewWindow()
        : base()
    {
        Title = "Gtk::GridView (Gio::ListStore)";
        SetDefaultSize(400, 400);

        var stringList = StringList.New(["One", "Two", "Three", "Four", "Five", "Six", "Seven", "Eight", "Nine", "Ten"]);
        var selectionModel = NoSelection.New(stringList);
        var listItemFactory = SignalListItemFactory.New();
        listItemFactory.OnSetup += SetupSignalHandler;
        listItemFactory.OnBind += BindSignalHandler;

        var gridView = GridView.New(selectionModel, listItemFactory);

        var scrolledWindow = ScrolledWindow.New();
        scrolledWindow.Child = gridView;
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
        label.WidthRequest = 100;
        label.HeightRequest = 100;
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

We will create an instance of our window in `Program.cs` and call the `Show` method to display it on screen.
```csharp
using GridView.App;

var application = Gtk.Application.New("org.kashif-code-samples.listview.sample", Gio.ApplicationFlags.FlagsNone);
application.OnActivate += (sender, args) =>
{
    var window = new StringListGridViewWindow
    {
        Application = application
    };
    window.Show();
};
return application.RunWithSynchronizationContext(null);
```

Running the application will result in following window. Here you can see that items are arranged in a grid.
<figure>
  <a href="/assets/images/2024-02-28/01-strings-in-grid-view.png"><img src="/assets/images/2024-02-28/01-strings-in-grid-view.png"></a>
  <figcaption>Strings in Grid View</figcaption>
</figure>  

## CustomObjectGridView
Let's add another window to display a list of custom objects as grid, its going to be very simple just displaying images of numbers from [svgrepo](https://www.svgrepo.com/vectors/number/) and text underneath.
First we will add a model that will hold the data we would use to display image and text in a grid.
```csharp
public class ItemData : GObject.Object
{
    public string ImagePath { get; set; }
    public string Text { get; set; }
    public string Description { get; set; }

    public ItemData(string imagePath, string text, string description)
        : base(true, Array.Empty<GObject.ConstructArgument>())
    {
        ImagePath = imagePath;
        Text = text;
        Description = description;
    }
}
```

The code will be largely similar to the `GridView` in other window with only change is we are going to use the `SingleSelection` and dispaly a message on console when user double clicks on an item or press enter in `OnGridViewOnActiveHandler` method.
Full listing will be as follows
```csharp
public class CustomObjectGridViewWindow : Window
{
    private readonly ListStore _model;

    public CustomObjectGridViewWindow()
        : base()
    {
        Title = "Gtk::GridView (Gio::ListStore)";
        SetDefaultSize(400, 400);

        _model = ListStore.New(ItemData.GetGType());
        _model.Append(new ItemData("Resources/number-1.svg", "One", "One"));
        _model.Append(new ItemData("Resources/number-2.svg", "Two", "Two"));
        _model.Append(new ItemData("Resources/number-3.svg", "Three", "Three"));
        _model.Append(new ItemData("Resources/number-4.svg", "Four", "Four"));
        _model.Append(new ItemData("Resources/number-5.svg", "Five", "Five"));
        _model.Append(new ItemData("Resources/number-6.svg", "Six", "Six"));
        _model.Append(new ItemData("Resources/number-7.svg", "Seven", "Seven"));
        _model.Append(new ItemData("Resources/number-8.svg", "Eight", "Eight"));
        _model.Append(new ItemData("Resources/number-9.svg", "Nine", "Nine"));

        var selectionModel = SingleSelection.New(_model);
        var listItemFactory = SignalListItemFactory.New();
        listItemFactory.OnSetup += SetupSignalHandler;
        listItemFactory.OnBind += BindSignalHandler;

        var gridView = GridView.New(selectionModel, listItemFactory);
        gridView.OnActivate += OnGridViewOnActiveHandler;

        var scrolledWindow = ScrolledWindow.New();
        scrolledWindow.Child = gridView;
        Child = scrolledWindow;
    }

    private void SetupSignalHandler(SignalListItemFactory sender, SetupSignalArgs args)
    {
        if (args.Object is not ListItem listItem)
        {
            return;
        }

        var box = Box.New(Orientation.Vertical, 2);
        box.SetSizeRequest(100, 100);
        listItem.Child = box;

        var image = Image.New();
        box.Append(image);
        var label = Label.New(null);
        box.Append(label);
    }

    private void BindSignalHandler(SignalListItemFactory sender, BindSignalArgs args)
    {
        if (args.Object is not ListItem listItem)
        {
            return;
        }

        if (listItem.Child is not Box box) return;
        if (box.GetFirstChild() is not Image image) return;
        if (image.GetNextSibling() is not Label label) return;
        if (listItem.Item is not ItemData itemData) return;

        image.SetFromFile(itemData.ImagePath);
        label.SetText(itemData.Text);
    }

    private void OnGridViewOnActiveHandler(GridView sender, ActivateSignalArgs args)
    {
        var itemData = _model.GetObject(args.Position) as ItemData;
        if (itemData is null) return;

        Console.WriteLine($"Selected Text: {itemData.Text}, Description: {itemData.Description}");
    }
}
```

## Application Window
We will update `Program.cs` file to add 2 buttons to display each window on a button click.

```csharp
application.OnActivate += (sender, args) =>
{
    var buttonStringListGridView = CreateButton("String List GridView Window");
    buttonStringListGridView.OnClicked += (_, _) => new StringListGridViewWindow().Show();

    var buttonCustomObjectGridView = CreateButton("Custom Object GridView Window");
    buttonCustomObjectGridView.OnClicked += (_, _) => new CustomObjectGridViewWindow().Show();

    var gtkBox = Gtk.Box.New(Gtk.Orientation.Vertical, 0);
    gtkBox.Append(buttonStringListGridView);
    gtkBox.Append(buttonCustomObjectGridView);

    var window = Gtk.ApplicationWindow.New((Gtk.Application) sender);
    window.Title = "GridView Sample";
    window.SetDefaultSize(300, 300);
    window.Child = gtkBox;
    window.Show();
};
```

After running the application, window will look like following
<figure>
  <a href="/assets/images/2024-02-28/02-grid-view-application.png"><img src="/assets/images/2024-02-28/02-grid-view-application.png"></a>
  <figcaption>Grid View Application</figcaption>
</figure>  

And GridView window with custom object will look like as follows
<figure>
  <a href="/assets/images/2024-02-28/03-custom-object-grid-view.png"><img src="/assets/images/2024-02-28/03-custom-object-grid-view.png"></a>
  <figcaption>Custom Objects in Grid View</figcaption>
</figure>  

## Source
Source code for the sample application is available on GitHub in [gtk4-dotnet8-grid-view](https://github.com/kashif-code-samples/gtk4-dotnet8-grid-view).

## References
In no particular order
* [GTK](https://www.gtk.org/)
* [GTK Installation](https://www.gtk.org/docs/installations/)
* [.NET](https://dotnet.microsoft.com/en-us/)
* [.NET Download](https://dotnet.microsoft.com/en-us/download)
* [Gir.Core](https://github.com/gircore/gir.core)
* [GirCore.Gtk-4.0](https://www.nuget.org/packages/GirCore.Gtk-4.0/)
* [Gtk GridView](https://docs.gtk.org/gtk4/class.GridView.html)
* [svgrepo](https://www.svgrepo.com/vectors/number/)
* [Tagger](https://github.com/NickvisionApps/Tagger)