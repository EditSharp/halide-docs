# Extensions

An extension is a folder in the app's extensions folder (App Settings ▸ Extensions ▸ Open Extensions Folder) holding
an `extension.json` and the extension's DLLs.

```json
{
  "id": "com.example.scopes",
  "name": "Scopes",
  "version": "1.0.0",
  "assembly": "Scopes.dll",
  "entry": "Scopes.ScopesExtension",
  "api": "1.0",
  "description": "Waveform and vectorscope views."
}
```

The first time the app finds an extension, and whenever its DLLs or manifest change, it asks before running it.
The Extensions page in App Settings turns extensions on and off, reloads them, and shows why one stopped.

## Building one

`edit-sharp-extensions/Sample` is a complete example. Its project references the app's own assemblies without copying
them, and its build output folder is an extension folder:

```xml
<Reference Include="EditSharp GUI" HintPath="$(EditSharpBin)/EditSharp GUI.dll" Private="false" />
<Reference Include="EditSharp" HintPath="$(EditSharpBin)/EditSharp.dll" Private="false" />
<Reference Include="GodotSharp" HintPath="$(EditSharpBin)/GodotSharp.dll" Private="false" />
```

The entry type implements [`IExtension`](xref:EditSharpGUI.Api.Extensions.IExtension):

```csharp
public sealed class Sample : IExtension
{
    public void Activate(ExtensionContext context)
    {
        context.AddCommand(new Command
        {
            Id = "editsharp.sample.hello",
            Title = "Say Hello",
            Run = (ctx, args) => "Hello",
        });
        context.AddMenuItem("Edit", "editsharp.sample.hello");
        context.AddShortcut("editsharp.sample.hello", new KeyCombo(Key.H, Control: true, Alt: true));

        context.AddView(new ViewDefinition("editsharp.sample.notes", "Notes", project =>
        {
            VBoxContainer notes = new();
            notes.AddChild(new Label { Text = $"Notes for {project.Name}" });
            return notes;
        }, Beside: "inspector"));

        context.AddSettingsPage("Sample", new SampleSettings());
        context.AddImporter([".samplewav"], path => new AudioMedia { Path = path });
        context.OnProjectOpened(project => context.Log($"{project.Name} opened"));
    }

    public void Deactivate() { }
}
```

Everything added through the [`ExtensionContext`](xref:EditSharpGUI.Api.Extensions.ExtensionContext) is removed again
when the extension is turned off or reloaded. For anything else, pass an `IDisposable` to `context.Track`.

## What an extension can do

- **Commands**, with menu items in File, Edit, View or Playback and default keys. The user can rebind the keys on the
  Keyboard Shortcuts page.
- **Views** that dock, tab and float like the built-in ones, and appear in the View menu.
- **Settings pages**: an object whose `[Editable]` properties App Settings shows, like its own pages. Saving the
  values is up to the extension.
- **Importers** that make media from new file types.
- **Anything in the API**: projects, the timeline, media, playback, layouts, and the EditSharp model itself
  (edited inside a batch, so it can be undone).

## Limits

- **Views** are built from Godot's stock controls, or from scenes using only them. Godot can't attach scripts from an
  extension's own classes.
- **Failures**: an exception from an extension's command, view, importer or event handler stops the extension and
  shows the error on the Extensions page. The app carries on.
- **Reloading** unloads the extension's assemblies. Anything the extension kept hold of outside the context can keep
  them loaded until the app restarts.
