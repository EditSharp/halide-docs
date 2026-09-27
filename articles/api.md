# The API

EditSharp's API has three layers. Each builds on the one below it.

```
Extensions ─┐             ┌─ menus, shortcuts, the GUI itself
Tests ──────┼─► Commands ◄┘        named, JSON arguments, one undo entry each
Remote ─────┘      │
                   ▼
              Services       the real C# API: projects, timeline, media, playback, layout…
                   │
                   ▼
              EditSharp      the model: Project, Timeline, Clip, History
```

## Services

`EditSharpApp.Instance` is the root. Everything about one open project goes through a
[`ProjectHandle`](xref:EditSharpGUI.Api.ProjectHandle); there's no hidden "current project" inside the services.

```csharp
EditSharpApp app = EditSharpApp.Instance;

ProjectHandle project = await app.Projects.OpenAsync(@"C:\Films\Trip.esproj");
IMedia song = project.Media.Import(["song.mp3"])[0];

project.Timeline.Place(song, at: Time.FromSeconds(5));   // one undo entry, "Place…"
project.Playback.Seek(Time.FromSeconds(5));
project.Playback.Play();

using (project.Batch("Rough cut"))                        // many edits, one undo entry
{
    project.Timeline.Split(Time.FromSeconds(12));
    project.Timeline.Delete(project.Timeline.ClipsAt(Time.FromSeconds(13)));
}

project.Layout.Float("inspector");
project.Layout.Apply("Audio");
```

| Service | What it covers |
|---|---|
| [`Projects`](xref:EditSharpGUI.Api.ProjectsService) | open, create, find; opened and closed events |
| [`ProjectHandle`](xref:EditSharpGUI.Api.ProjectHandle) | name, file, dirty, save, close, batch, the model and its history |
| [`Timeline`](xref:EditSharpGUI.Api.TimelineService) | clips, place, move, trim, split, delete, duplicate, link, channels, selection, playhead, zoom |
| [`Media`](xref:EditSharpGUI.Api.MediaService) | import, find, remove, open in the source viewer |
| [`Playback`](xref:EditSharpGUI.Api.PlaybackService) | play, pause, seek, shuttle, step, loop, position events |
| [`Layout`](xref:EditSharpGUI.Api.LayoutService) | open, close, float and dock views; apply, save, rename, delete and reset layouts |
| [`Commands`](xref:EditSharpGUI.Api.CommandsService) | every named command, run by id |
| [`Views`](xref:EditSharpGUI.Api.ViewRegistry), [`Menus`](xref:EditSharpGUI.Api.MenuService), [`Importers`](xref:EditSharpGUI.Api.ImporterRegistry), [`SettingsPages`](xref:EditSharpGUI.Api.SettingsPageRegistry) | what extensions add |

Every call must come from the main thread. Remote calls are moved there for you.

### Undo

Every service edit is one entry in the project's history, named after what it did. To group several edits into one
entry, use `project.Batch("name")`. Batches nest: an inner one joins the outer entry. Call `Cancel()` on a batch to
roll it all back instead. `project.Batch("name", () => { … })` rolls back by itself if the code throws.

The model (`project.Project`) can also be edited directly, as long as the edit happens inside a batch or another
history transaction.

## Commands

A command is a named action: `project.save`, `timeline.splitAt`, `layout.float`. Menus, shortcuts, extensions and
remote callers all run commands, so anything the menus can do, a script can do.

```csharp
app.Commands.Run("timeline.seek", project, new JsonObject { ["at"] = 2.5 });
```

Commands behind menu items and keys act on the view last clicked, the same way the key would. The API commands
(`timeline.*`, `layout.*`, `media.importPaths`, `project.create`…) take explicit JSON arguments instead, and never
open dialogs. The [command catalogue](commands.md) lists every command with its arguments.

## Versioning

`EditSharpApp.Version` is `major.minor`. The minor number goes up when something is added; the major when something
existing changes. An extension declares the version it was built against, and runs on any app with the same major
number and at least that minor number.
