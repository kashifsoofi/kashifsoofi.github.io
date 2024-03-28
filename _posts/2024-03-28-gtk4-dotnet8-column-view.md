---
title:  "GTK4 ColumnView with .NET 8"
date:   2024-03-28
categories:
  - gtk4
  - dotnet
tags:
  - gtk4
  - .NET 8
  - C#
---
In this post I will make use of [ColumnView](https://docs.gtk.org/gtk4/class.ColumnView.html) to display list of data. `ColumnView` is used to display 1-dimensional list with one or more columns. Foucus of this post will be to dispaly multiple columns.

## Project Setup
Let's start by setting up a new solution and project.
```shell
dotnet new sln --name ColumnView
dotnet new console -o ColumnView.App
dotnet sln add ColumnView.App/ColumnView.App.csproj
```
And add a reference to `GirCore.Gtk-4.0` nuget package, the latest version is `0.5.0-preview.4` at the time of writing.
```shell
cd ColumnView.App
dotnet add package GirCore.Gtk-4.0 --version 0.5.0-preview.4
```

## ColumnViewWindow
I will add a new class named `ColumnViewWindow` inheriting from `Gtk.Window`. This window will contain a `ColumnView` widget to display tabular data.

### Model
First let's add a class that we will use to display our data and a static method to generate the sample data. We will inherit this model from `GObject.Object`.
```csharp
public class UserData : GObject.Object
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Username { get; set; }
    public string Email { get; set; }

    public UserData(int id, string firstName, string lastName, string username, string email)
        : base(true, Array.Empty<GObject.ConstructArgument>())
    {
        Id = id;
        FirstName = firstName;
        LastName = lastName;
        Username = username;
        Email = email;
    }

    public static UserData[] SampleUserData()
    {
        return
        [
            new UserData(1, "Leanne", "Graham", "Bret", "Sincere@april.biz"),
            new UserData(2, "Ervin", "Howell", "Antonette", "Shanna@melissa.tv"),
            new UserData(3, "Patricia", "Lebsack", "Karianne", "Julianne.OConner@kory.org"),
            new UserData(4, "Chelsey", "Dietrich", "Kamren", "Lucio_Hettinger@annie.ca"),
            new UserData(5, "Clementine", "Bauch", "Samantha", "Nathan@yesenia.net"),
        ];
    }
}
```

### ColumnViewWindow
Next let's update the `ColumnViewWindow` to inherit from `Gtk.Window`. And then initialise `ListStore` and widgets to display the data.

First we will add a `Gio.ListStore` and add our sample data to the list store. Then we will add a single seletion model using the model.
```csharp
var model = Gio.ListStore.New(UserData.GetGType());
foreach (var userData in UserData.SampleUserData())
{
    model.Append(userData);
}

var selectionModel = SingleSelection.New(model);
selectionModel.Autoselect = false;
selectionModel.CanUnselect = true;
```
Next we will add `ColumnView` and add it to a `ScrolledWindow` and add the scrolled window widget to `ColumnViewWindow`.
```csharp
var columnView = ColumnView.New(selectionModel);
columnView.AddCssClass("data-table");

var scrolledWindow = ScrolledWindow.New();
scrolledWindow.Child = columnView;

Child = scrolledWindow;
```
### Text Columns
Next we will add text based columns displaying data using `Label` widget.
```csharp
// Id Column
var listItemFactory = SignalListItemFactory.New();
listItemFactory.OnSetup += (_, args) => OnSetupLabel(args, Align.End);
listItemFactory.OnBind += (_, args) => OnBindText(args, (ud) => ud.Id.ToString());

var idColumn = ColumnViewColumn.New(nameof(UserData.Id), listItemFactory);
columnView.AppendColumn(idColumn);

// Name Column
listItemFactory = SignalListItemFactory.New();
listItemFactory.OnSetup += (_, args) => OnSetupLabel(args, Align.Start);
listItemFactory.OnBind += (_, args) => OnBindText(args, (ud) => ud.Name);

var nameColumn = ColumnViewColumn.New(nameof(UserData.Name), listItemFactory);
columnView.AppendColumn(nameColumn);

// Sales Column
listItemFactory = SignalListItemFactory.New();
listItemFactory.OnSetup += (_, args) => OnSetupLabel(args, Align.End);
listItemFactory.OnBind += (_, args) => OnBindText(args, (ud) => $"{ud.Sales:C}");

var salesColumn = ColumnViewColumn.New(nameof(UserData.Sales), listItemFactory);
columnView.AppendColumn(salesColumn);
```

I have added helper methods to `OnSetupLabel` and `OnBindText` to reuse code to create `Label` and set text when binding data.
```csharp
private void OnSetupLabel(SetupSignalArgs args, Align align)
{
    if (args.Object is not ListItem listItem)
    {
        return;
    }

    var label = Label.New(null);
    label.Halign = align;
    listItem.Child = label;
}

private void OnBindText(BindSignalArgs args, Func<UserData, string> getText)
{
    if (args.Object is not ListItem listItem)
    {
        return;
    }

    if (listItem.Child is not Label label) return;
    if (listItem.Item is not UserData userData) return;

    label.SetText(getText(userData));
}
``` 
### ProgressBar Column
This is just to display how to use the `ProgressBar` widget and also to demonstrate that any widget can be used in the `ColumnView`.

Code to add the column
```csharp
// Percentage Column
listItemFactory = SignalListItemFactory.New();
listItemFactory.OnSetup += OnSetupProgress;
listItemFactory.OnBind += OnBindProgress;

var percentageColumn = ColumnViewColumn.New(nameof(UserData.ProgressPercentage), listItemFactory);
columnView.AppendColumn(percentageColumn);
```
And methods to create widget and set data
```csharp
private void OnSetupProgress(SignalListItemFactory sender, SetupSignalArgs args)
{
    if (args.Object is not ListItem listItem)
    {
        return;
    }

    var progressBar = ProgressBar.New();
    progressBar.SetShowText(true);
    listItem.Child = progressBar;
}

private void OnBindProgress(SignalListItemFactory sender, BindSignalArgs args)
{
    if (args.Object is not ListItem listItem)
    {
        return;
    }

    if (listItem.Child is not ProgressBar progressBar) return;
    if (listItem.Item is not UserData userData) return;

    progressBar.SetFraction(userData.ProgressPercentage * 0.01);
}
```

## Program.cs
Lets update `Program.cs` to create and show our `ColumnViewWindow`.
```csharp
var application = Gtk.Application.New("org.kashif-code-samples.columnview.sample", Gio.ApplicationFlags.FlagsNone);
application.OnActivate += (sender, args) =>
{
    var window = new ColumnViewWindow
    {
        Application = application
    };
    window.Show();
};
return application.RunWithSynchronizationContext(null);
```

Running the program will show us following window.
<figure>
  <a href="/assets/images/2024-03-28/01-column-view-window.png"><img src="/assets/images/2024-03-28/01-column-view-window.png"></a>
  <figcaption>Column View Window</figcaption>
</figure>  

## Source
Source code for the sample application is available on GitHub in [gtk4-dotnet8-column-view](https://github.com/kashif-code-samples/gtk4-dotnet8-column-view).

## References
In no particular order
* [GTK](https://www.gtk.org/)
* [GTK Installation](https://www.gtk.org/docs/installations/)
* [.NET](https://dotnet.microsoft.com/en-us/)
* [.NET Download](https://dotnet.microsoft.com/en-us/download)
* [Gir.Core](https://github.com/gircore/gir.core)
* [GirCore.Gtk-4.0](https://www.nuget.org/packages/GirCore.Gtk-4.0/)
* [ColumnView](https://docs.gtk.org/gtk4/class.ColumnView.html)
* [JSON Placeholder](https://jsonplaceholder.typicode.com/users)
And many more