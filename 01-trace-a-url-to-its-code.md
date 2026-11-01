# Tutorial 1: From a bug report URL to the code that serves it

> **Status:** preview tutorial; clean-machine end-to-end verification is pending.
> **Edition:** Community (free). **Time to follow:** about 15 minutes.

A support ticket arrives with a URL and nothing else. In a layered ASP.NET Core app the
code behind that URL is spread across controllers, views, partials, view models and
services, and the route itself may come from a convention in `Program.cs`. This tutorial
uses IOCO to list every file that serves the URL, with the reason each one is included,
and then uses that list to find the bug.

## The task

> **Ticket #1042:** "A customer pasted `https://shop.example/Order/Detail/42` into a support
> chat. The page says *Not found*. The same order opens fine when they click *Detail* on
> their order history page."

Find the files that serve `/Order/Detail/42`, work out why it shows *Not found*, and
decide where the fix belongs.

## What you need

- Windows x64 with VS Code, the
  [.NET 8 runtime](https://dotnet.microsoft.com/download/dotnet/8.0) or later (the SDK is
  not needed) and Git. These tutorials were checked with IOCO View 0.1.0, a preview whose
  supported hosts are Windows x64; see the extension's Marketplace page for the exact
  versions.
- **IOCO View** from the VS Code Marketplace,
  [`DarrenJMcLeod.ioco-vscode`](https://marketplace.visualstudio.com/items?itemName=DarrenJMcLeod.ioco-vscode).
  Install it from the Extensions view, or run:

  ```sh
  code --install-extension DarrenJMcLeod.ioco-vscode
  ```

  The extension contains the IOCO CLI, so you do not need a separate download, a NuGet
  package or a clone of the IOCO repository. Mapping is Community (free); no account or
  key is needed.
- The tutorial sample download, `ioco-tutorial-sample.zip`, from the
  [sample download](https://github.com/Tempuskg/ioco-tutorials/releases/download/tutorial-sample-v0.1.0/ioco-tutorial-sample.zip). It contains `EShopLayered`, an
  eShopOnWeb-shaped app in the classic layered layout (two projects and 18 URLs), the
  setup scripts and the file Tutorial 2 changes. It is MIT-licensed; see `LICENSE.txt`
  in the archive.

### The `ioco` command

The tutorials show each step as an `ioco` command. The extension does not put `ioco` on
your PATH, so define it once in each terminal you use. Both definitions run the CLI that
the extension installed:

```sh
# Git Bash
ioco() { dotnet "$(ls -d ~/.vscode/extensions/darrenjmcleod.ioco-vscode-*/resources/cli/ioco.dll | tail -n 1)" "$@"; }
```

```powershell
# PowerShell
$iocoDll = (Get-ChildItem "$HOME\.vscode\extensions\darrenjmcleod.ioco-vscode-*\resources\cli\ioco.dll" | Sort-Object FullName | Select-Object -Last 1).FullName
function ioco { $input | dotnet $iocoDll @args }
```

Check it with `ioco --version`, which prints the extension's version (`0.1.0`). If the
command finds no `ioco.dll`, install the extension first; if you installed it into another
extensions folder (for example VS Code Insiders), change the path to match. The command
blocks below use Git Bash syntax; Tutorial 3 uses `grep`, so run it in Git Bash.

This tutorial also works without a terminal: open the sample's folder in VS Code and use
the **IOCO** view in the Explorer instead of `ioco map`. Each step says where.

## 1. Make your own copy of the sample

Extract `ioco-tutorial-sample.zip` into a working folder; it creates an
`ioco-tutorial-sample` folder there. The setup script copies the sample into its own Git
repository next to it, so you can change it and diff your changes freely. Run it from the
working folder:

```sh
# macOS/Linux/Git Bash
sh ioco-tutorial-sample/setup-sample.sh ioco-tutorial-shop
```

```powershell
# PowerShell
ioco-tutorial-sample\setup-sample.ps1 -Target ioco-tutorial-shop
```

If PowerShell refuses to run a downloaded script, see the archive's `README.md`.

```sh
cd ioco-tutorial-shop
```

## 2. Map the solution

```sh
ioco map EShopLayered.sln
```

The first lines confirm what was analysed:

```
2 of 2 project(s) loaded, 18 endpoint(s), dependency depth 1
```

IOCO reads the source with Roslyn. It does not build, restore or run the app, and nothing
leaves your machine.

**In VS Code:** run `code .` in `ioco-tutorial-shop`. The **IOCO** view in the Explorer
maps `EShopLayered.sln` when the folder opens; its title shows the solution and the URLs
node shows 18 endpoints.

## 3. Find the URL in the tree

The URL tree is organised by path segment, so `/Order/Detail/42` is under
`Order › Detail › {id?}`:

```
    Order
      ...
      Detail
        {id?}
          * GET /Order/Detail/{id?}  [mvc] OrderController.Detail
              route source           src/Web/Controllers/OrderController.cs  OrderController.Detail matched by conventional route 'default' ({controller=Home}/{action=Index}/{id?})
              route config           src/Web/Program.cs  conventional route 'default': {controller=Home}/{action=Index}/{id?}
              view                   src/Web/Views/Shared/NotFound.cshtml  View("NotFound") by name in OrderController.Detail resolves to src/Web/Views/Shared/NotFound.cshtml
              model                  src/Web/ViewModels/OrderViewModel.cs  model passed to View(nameof(Detail), viewModel) in OrderController.Detail (+1 more)
              view                   src/Web/Views/Order/Detail.cshtml  View(nameof(Detail), viewModel) by name in OrderController.Detail resolves to src/Web/Views/Order/Detail.cshtml
              partial view           src/Web/Views/Order/_OrderItems.cshtml  partial "_OrderItems" referenced by src/Web/Views/Order/Detail.cshtml:6
              model                  src/Web/ViewModels/OrderItemViewModel.cs  @model IEnumerable<OrderItemViewModel> in src/Web/Views/Order/_OrderItems.cshtml
              service                src/Web/Interfaces/IOrderViewModelService.cs [depth 1]  injected into OrderController constructor (orderViewModelService)
              service implementation src/Web/Services/OrderViewModelService.cs [depth 1]  implementation of IOrderViewModelService registered by AddScoped<IOrderViewModelService, OrderViewModelService>() in Program.cs:16
```

**In VS Code:** expand **URLs › Order › Detail › {id?}** and then the endpoint. The same
files are grouped by role; hover over a file to see its reason.

Nine files in seven folders, each with the reason it is on the list. Two lines matter for
this ticket:

- **route source / route config:** there is no `[Route]` attribute. The URL reaches
  `OrderController.Detail` through the conventional route `{controller}/{action}/{id?}`
  in `Program.cs`, so the last segment, `42`, arrives as a route value named **`id`**.
- **view `NotFound.cshtml`:** the action has a code path that returns `View("NotFound")`.
  That is the page the customer saw.

## 4. Read the files IOCO pointed to

Open the route source, `src/Web/Controllers/OrderController.cs`:

```csharp
[HttpGet]
public async Task<IActionResult> Detail(int orderId)
{
    var viewModel = await _orderViewModelService.GetOrderAsync(orderId);
    if (viewModel is null)
    {
        return View("NotFound");
    }

    return View(nameof(Detail), viewModel);
}
```

The parameter is `orderId`, but the route value is `id`. Model binding matches by name,
so `/Order/Detail/42` leaves `orderId` at `0`, no order has that id, and the action
returns *Not found*.

Why does the order history page work? Open `src/Web/Views/Order/MyOrders.cshtml`, which
is on the map under `ANY /Order/MyOrders/{id?}`:

```cshtml
<a asp-action="Detail" asp-route-orderId="@order.OrderNumber">Detail</a>
```

`orderId` is not a route parameter, so the tag helper emits `/Order/Detail?orderId=42`,
which binds. Only hand-typed or shared `/Order/Detail/42` links fail.

> The sample has no sign-in configured and `OrderController` is `[Authorize]`, so this
> tutorial confirms the cause by reading the code rather than by running the page.

## 5. Decide where the fix belongs

Two fixes are possible, and the map helps you see what each touches:

- **Rename the parameter** to `id` in `OrderController.Detail` (and `Cancel`, which has
  the same shape). The links that pass `orderId` must then change too: `MyOrders.cshtml`,
  `Detail.cshtml` (the cancel form) and the redirect in `Pages/Order/Checkout.cshtml.cs`.
- **Put `orderId` in the route**: keep the parameter and add an attribute route,
  `[HttpGet("/Order/Detail/{orderId:int}")]`. The tag helpers then generate
  `/Order/Detail/42` themselves, but old `?orderId=42` links stop matching, because the
  segment is now required. Run `ioco map` again and the tree shows the new template:

  ```
        Detail
          {orderId:int}
            * GET /Order/Detail/{orderId:int}  [mvc] OrderController.Detail
                route source           src/Web/Controllers/OrderController.cs  attribute route [HttpGet("/Order/Detail/{orderId:int}")] on OrderController.Detail
  ```

  so you can confirm the route you meant is the route you got.

Pick one, edit your copy, and run `ioco map EShopLayered.sln` again to check the result.
In VS Code, saving the `.cs` file refreshes the IOCO view.

## What IOCO did and did not do here

- It listed the files that serve the URL and the reason for each, including the
  conventional route and the *Not found* view, which a text search for "Detail" would mix
  in with the other `Detail` pages and endpoints.
- It did not find the bug. You found it by reading the two files the map put in front of
  you.
- The map is static analysis. Custom route conventions, non-constant route strings and
  other dynamic routing are not modelled; see
  [what the analysis does and does not see](analysis-limits.md).

## Try it on your own app

```sh
ioco map path/to/YourApp.sln > map.txt
```

Search `map.txt` for the URL from your last bug report. Or, in VS Code, open the **IOCO**
view in the Explorer and expand the URL.

## Demo script (3 minutes)

1. (0:00) Show the ticket text with the URL.
2. (0:20) In VS Code, open the sample; the IOCO view maps it automatically. Expand
   `Order › Detail › {id?}`.
3. (0:50) Point at **route source** and **route config**: conventional route, `{id?}`.
4. (1:20) Click the endpoint to open `OrderController.Detail` at its line. Highlight
   `int orderId`.
5. (1:50) Open `MyOrders.cshtml` from the tree. Highlight `asp-route-orderId`.
6. (2:20) State the cause in one sentence and show the two fixes.
7. (2:45) Close on the limits: IOCO showed where to look; the reading was ours.
