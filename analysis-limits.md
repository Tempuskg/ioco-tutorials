## What the analysis does and does not see

The engine loads code **without MSBuild**. It reads project files as XML, compiles sources
with Roslyn against the .NET runtime's reference assemblies and the referenced projects,
and never restores or builds. So it works on code that doesn't compile and needs no
package restore. ASP.NET Core types are matched by name rather than by symbol. Known limits
in 1.0:

- `Compile Remove`/`Include` items, conditional properties and `Directory.Build.props` are
  ignored. Sources come from the SDK default globs (`**/*.cs`, or `**/*.vb` in a `.vbproj`,
  and `**/*.cshtml`, excluding `bin/`, `obj/`, and folders that hold another project).
- Visual Basic projects (`.vbproj`) get the SDK's default project imports, plus their
  `<Import>` items and `OptionStrict`, `OptionExplicit`, `OptionInfer` and `OptionCompare`.
  A C# project sees a Visual Basic one it references (and the reverse) through the
  declarations of its types. A member or base type there that does not compile, such as a
  method whose signature uses a package type, is left out: the type is still found, but
  calls to that member from the other language are not followed. Only Visual Basic
  WinForms menus are mapped; Visual Basic WPF, web and console projects are not
  (`IOCO006`), and neither are F# projects (`IOCO003`).
- Routes must be constant strings. Custom `IRouteTemplateProvider` attributes, route
  conventions, `AddPageRoute`, a custom Razor Pages `RootDirectory`, and custom view
  location expanders are not modelled.
- For conventional routing, only the first matching route is reported for each action.
- MapGroup prefixes are followed through locals, fields and fluent calls in the same
  method. A group builder passed in as a parameter contributes no prefix.
- Layouts (`_Layout`, `_ViewStart`) and view components are not listed as serving files.

Menus and commands (1.3):

- Menus and commands are read from source. Menu items, toolbar buttons and context-menu
  items built at run time or bound to `ItemsSource`, status strips, keyboard shortcuts and
  third-party ribbons are not listed. Toolbar buttons, context-menu items and WPF ribbon
  controls are listed from 1.4, and so are the windows of an app that has no menu. A
  window is mapped only when it is reachable from a startup window IOCO recognizes. A window
  opened by reflection, by a type name in a string, or from a startup that IOCO does not
  recognize (an `ApplicationContext` that shows forms, a DI host that resolves the first
  window by an interface) is not listed. A toolbar button that runs another entry's handler is
  listed in that entry's `alsoReachedFrom`, which records where it is declared but adds no
  file, so a change to only that declaration is not matched to the entry. Command and verb names must be constant strings. WinUI, UWP, Avalonia and MAUI
  projects, and Visual Basic WPF projects, are not mapped; .NET Framework WinForms and WPF projects are, with
  their sources found by the same globs as SDK projects.
- WinForms, WPF and the command-line libraries are not referenced (nothing is restored), so
  their types are recognized by name: a source type of your own named `Command` or
  `MenuStrip` is treated as yours, not the library's.
- A handler's files are what it creates, calls and uses, followed through its own type's
  methods and into the forms and view models it opens. Code reached through reflection,
  messaging (an event aggregator or mediator), or a service locator by name is not
  included, and a WinForms `ItemClicked` handler that switches on the clicked item is not
  matched to each item.
- A WPF `Command` binding is followed on the window's view model (or a `DataContext`
  binding on the way down to the menu). A `DataContext` binding on the root element is taken
  to be that view model, and one with `Source`, `RelativeSource` or `ElementName` is not
  followed. A view model found by naming convention must be
  unique; otherwise the binding is reported (`IOCO022`) and not followed.
- Hand-written dispatch is followed from `Main` and top-level statements, through methods
  that are given the arguments or a slice of them. Commands parsed into an options object
  first (`var options = Parse(args); if (options.Command == "map")`) are not seen.
