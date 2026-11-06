# Tutorial 3: A handover map for the team that takes over

> **Status:** preview tutorial for IOCO View 0.1.0; see the README for recorded verification.
> **Edition:** Community (free) for every step except the optional change report in
> step 5, which needs Pro or the trial. **Time to follow:** about 20 minutes.

When an engagement ends, the receiving team inherits a codebase they did not write. The
most useful first answer for them is "which files serve this URL?", and the most useful
safeguard is a recorded starting point they can compare future changes against. This
tutorial builds a handover pack that provides both, using only files committed to the
client's repository.

## The task

> Your consultancy's engagement on the shop app ends on Friday. The client's in-house team
> takes over on Monday. Hand over:
>
> 1. a URL-to-file map they can read without installing anything;
> 2. the list of shared code, so they know which changes cross features;
> 3. the settings and a saved baseline, so their developers and CI map the app the same way
>    and can compare future changes with the state you handed over.

## What you need

- Everything from [Tutorial 1](01-trace-a-url-to-its-code.md#what-you-need), including
  the `ioco` command defined in a Git Bash terminal
  ([The `ioco` command](01-trace-a-url-to-its-code.md#the-ioco-command)); steps 1–5 have
  no VS Code equivalent. You also need a fresh
  copy of the sample made with the setup script from the tutorial sample download. From
  the folder you extracted `ioco-tutorial-sample.zip` into:

  ```sh
  sh ioco-tutorial-sample/setup-sample.sh client-shop
  cd client-shop
  ```

  In PowerShell, run `ioco-tutorial-sample\setup-sample.ps1 -Target client-shop` instead,
  then continue in Git Bash.
- For the optional step 5 only, the Pro trial or a signed Pro license; see
  [Tutorial 2](02-review-a-shared-service-change.md#what-you-need).

The handover steps create local Git commits. On a clean machine, configure a sample-only
identity from inside `client-shop` before continuing:

```sh
git config user.name "IOCO tutorial"
git config user.email "tutorial@example.invalid"
```

These settings apply only to this sample repository. For your own app, use your normal
Git author identity instead.

## 1. Record the settings everyone will map with

```sh
ioco config init EShopLayered.sln --project Web
```

This writes `ioco.json` next to the solution:

```json
{
  "version": 1,
  "input": "EShopLayered.sln",
  "depth": 1,
  "projects": [
    "Web"
  ],
  "baseline": ".ioco/baseline.json"
}
```

The file fixes the input, the dependency depth and the project scope, so a developer's
editor, a teammate's CLI and the CI job all produce the same map. `ioco` finds it from the
current directory or any parent up to the Git root. A run that contradicts it, for example
with a different `--depth`, is rejected rather than quietly producing a different map.

## 2. Save the baseline and commit both files

```sh
ioco baseline
ioco config check
git add ioco.json .ioco/baseline.json
git commit -m "Add IOCO settings and handover baseline"
```

```
Wrote baseline .ioco/baseline.json: 1 project(s), 18 endpoint(s), depth 1, commit 2ee7b95763cf. Commit it with ioco.json.
...ioco.json is valid: 1 of 1 project(s) in scope loaded, 18 endpoint(s) at depth 1, baseline .ioco/baseline.json matches.
```

(Your commit hash will differ.) The baseline is the full map in JSON. Its paths are
relative and it records the commit it was mapped at, so it holds no machine-specific
paths and can be committed as is. Later runs report how many commits have changed the
mapped source since the baseline, which tells the team when to refresh it.

`ioco baseline` refuses to write a baseline if a project in scope failed to load, because
an incomplete baseline would hide those URLs from every later comparison. If that happens
on a client's app, fix the load problem or narrow `--project` first.

## 3. Write the readable map

```sh
mkdir handover
ioco map > handover/ioco-map.txt
```

With no input given, `ioco` uses `ioco.json`. The file has two parts the receiving team
will use:

- **The URL tree**, with every file that serves each URL and why (see
  [Tutorial 1](01-trace-a-url-to-its-code.md#3-find-the-url-in-the-tree) for how to read
  it).
- **The scatter report**, which shows for each feature where its files live today:

  ```
    Web: Order  (4 endpoint(s); IOCO folder: Order/; 10 folder(s) today)
      Controllers/
        OrderController.cs                           -> Order/
      Infrastructure/
        SmtpEmailSender.cs                           shared by 3 features (account, Manage, Order)
      Interfaces/
        IOrderViewModelService.cs                    used by 2 features (api, Order); rule of three not met
      ...
  ```

The first line of `ioco-map.txt` contains the absolute path of the solution on your
machine. Edit it out if the path names something you would rather not share. The baseline
JSON from step 2 has no such paths; prefer it over `ioco map --json` output for anything
you hand over.

## 4. Summarise the shared code

The receiving team's riskiest changes are to code several features share. List it:

```sh
grep -E "shared by|used by 2" handover/ioco-map.txt | sort -u
```

```
      IOrderViewModelService.cs                    used by 2 features (api, Order); rule of three not met
      IUserProfileService.cs                       used by 2 features (account, Manage); rule of three not met
      ManageController.cs                          used by 2 features (account, Manage); rule of three not met
      OrderViewModelService.cs                     used by 2 features (api, Order); rule of three not met
      SmtpEmailSender.cs                           shared by 3 features (account, Manage, Order)
      UserProfileService.cs                        used by 2 features (account, Manage); rule of three not met
```

Copy the result into the handover note below. `SmtpEmailSender.cs` is the one file used by
three features; [Tutorial 2](02-review-a-shared-service-change.md) shows how to review a
change to it.

## 5. Optional (Pro): report what the engagement changed

If the client wants a record of which URLs your engagement's changes touched, run a review
over the engagement's commits and include the report:

```sh
git diff --name-status <first-engagement-commit>..HEAD | ioco review --format html --output handover/engagement-changes.html
```

The HTML is self-contained and lists, for each URL, the changed files and the evidence that
relates them, with the limits of the analysis. It is a list of candidates to review, not a
statement of what changed at runtime. Reports stay readable after a trial or subscription
ends.

## 6. Write the handover note

Add `handover/README.md` and commit the folder:

```markdown
# Shop app: IOCO handover (<date>)

- `ioco.json` and `.ioco/baseline.json` (repository root): the settings and the map as handed
  over at commit <hash>. 18 URLs, project Web, dependency depth 1.
- `ioco-map.txt`: every URL, the files that serve it, and why. Search it for a URL.
- Shared code: SmtpEmailSender.cs is used by three features (account, Manage, Order);
  <paste the rest of step 4>. Review changes to these files across all their features.
- Known analysis limits: <copy from analysis-limits.md,
  the ones that apply to this app>.

To regenerate: install the IOCO View extension for VS Code (free), open this repository,
and use the IOCO view, or run `ioco map` from the repository root.
To compare with the handover state (Pro): `ioco compare`.
To list URLs touched by a branch (Pro): `git diff --name-status main...HEAD | ioco review`.
```

```sh
git add handover
git commit -m "Add IOCO handover pack"
```

## What IOCO did and did not do here

- It produced the URL-to-file map, the scatter and shared-code lists, and a baseline whose
  paths and settings the next team can reproduce.
- It did not judge the architecture. "Shared by 3 features" is a fact about the current
  files, not a recommendation to move them.
- Static-analysis limits apply. Note in the handover any routing the app does dynamically,
  because the map will not show it.

## Try it on your own app

```sh
ioco config init path/to/YourApp.sln
ioco baseline
ioco map > ioco-map.txt
```

## Demo script (3 minutes)

1. (0:00) State the task: engagement ends Friday, the client's team starts Monday.
2. (0:20) `ioco config init`, show `ioco.json`.
3. (0:45) `ioco baseline`, `ioco config check`; point at "commit" and relative paths.
4. (1:20) `ioco map > handover/ioco-map.txt`; scroll to the Order scatter.
5. (1:50) The `grep` for shared code; one file at three features.
6. (2:20) Show the finished `handover/README.md`.
7. (2:45) Close: everything is in the client's repository; nothing was uploaded.
