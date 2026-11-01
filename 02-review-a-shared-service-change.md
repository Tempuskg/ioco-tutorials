# Tutorial 2: Which URLs does a shared-service change touch?

> **Status:** preview tutorial; clean-machine end-to-end verification is pending.
> **Edition:** Pro. `ioco review` (steps 3 and 5) needs the free 14-day Pro trial (no
> card) or a signed Pro license. Steps 1, 2 and 4 work in Community. **Time to follow:** about 20 minutes.

A pull request changes one class that several features depend on. The diff shows the
class; it does not show which pages and APIs reach it. This tutorial turns a Git diff into
a list of URLs to review and test, with the reason each URL is on the list, and then
narrows that list by reading the code.

## The task

> **PR #318 – "Fail fast on blank email recipients":** `SmtpEmailSender.SendEmailAsync`
> now throws `ArgumentException` when the recipient is empty, instead of silently
> dropping the message.

Before approving the PR, produce the list of URLs a reviewer should check, decide which
ones can actually reach the new exception, and attach the list to the PR.

## What you need

- Everything from [Tutorial 1](01-trace-a-url-to-its-code.md#what-you-need): IOCO View
  from the VS Code Marketplace, the `ioco` command defined in your terminal
  ([The `ioco` command](01-trace-a-url-to-its-code.md#the-ioco-command)), the tutorial
  sample download and your copy of the sample made with its setup script
  ([Tutorial 1, step 1](01-trace-a-url-to-its-code.md#1-make-your-own-copy-of-the-sample)).
  If you changed it in Tutorial 1, make a fresh copy.
- The Pro trial or a signed Pro license. The trial is free for 14 days, needs no payment
  card and is limited to one per person. It starts when you choose it, after a successful
  map: run `ioco license trial start`, or in VS Code run **IOCO: Start 14-day Pro Trial**
  (the sparkle icon in the IOCO view's title bar). Both use the same license on your
  computer, so a trial started in one applies to the other; `ioco license status` shows
  what is active. Paid licenses are not on sale yet in this preview. When the trial ends,
  `ioco review` stops and everything else keeps working.

## 1. See how shared the class is

```sh
cd ioco-tutorial-shop    # from the folder you extracted the sample into
ioco map EShopLayered.sln
```

The scatter report at the end of the output lists, for each feature, where its files live
today. `SmtpEmailSender.cs` appears under three features:

```
  Web: account  (1 endpoint(s); IOCO folder: Account/; 5 folder(s) today)
    ...
    Infrastructure/
      SmtpEmailSender.cs                           shared by 3 features (account, Manage, Order)
```

Three features is the IOCO rule-of-three threshold for shared code, so a change here is a
cross-feature change. In VS Code, the same line is under **Scatter › Order** (and
**account** and **Manage**) in the IOCO view.

## 2. Make the change

Copy the PR's version of the class over the sample's:

```sh
cp ../ioco-tutorial-sample/samples/SmtpEmailSender.cs src/Web/Infrastructure/SmtpEmailSender.cs
git diff --name-status
```

```
M	src/Web/Infrastructure/SmtpEmailSender.cs
```

The new method body:

```csharp
public Task SendEmailAsync(string email, string subject, string message)
{
    // The tutorial change: refuse to queue mail without a recipient instead of silently dropping it.
    ArgumentException.ThrowIfNullOrWhiteSpace(email);
    return Task.CompletedTask;
}
```

## 3. Turn the diff into URLs

```sh
git diff --name-status | ioco review EShopLayered.sln
```

On a branch you would use `git diff --name-status main...HEAD`. `ioco review` has no
VS Code command; the nearest editor equivalent is to right-click `SmtpEmailSender.cs` and
choose **IOCO: Show Candidate Affected Endpoints** (also Pro), which lists the same seven
URLs for that one file. `ioco review` also accepts
`--name-only` and `git status --porcelain` output, or a file passed with `--changes`.

```
URLs to review from changed files (7)
  GET /Manage/ChangePassword  [mvc] ManageController.ChangePassword
    M src/Web/Infrastructure/SmtpEmailSender.cs  service implementation [depth 1]
  POST /Manage/ChangePassword  [mvc] ManageController.ChangePassword
    M src/Web/Infrastructure/SmtpEmailSender.cs  service implementation [depth 1]
  GET /Manage/MyAccount  [mvc] ManageController.MyAccount
    M src/Web/Infrastructure/SmtpEmailSender.cs  service implementation [depth 1]
  POST /Manage/MyAccount  [mvc] ManageController.MyAccount
    M src/Web/Infrastructure/SmtpEmailSender.cs  service implementation [depth 1]
  GET /Manage/TwoFactor  [mvc] ManageController.TwoFactorAuthentication
    M src/Web/Infrastructure/SmtpEmailSender.cs  service implementation [depth 1]
  GET,POST /Order/Checkout  [page] Pages/Order/Checkout
    M src/Web/Infrastructure/SmtpEmailSender.cs  service implementation [depth 1]
  POST /account/send-verification-email  [mvc] ManageController.SendVerificationEmail
    M src/Web/Infrastructure/SmtpEmailSender.cs  service implementation [depth 1]

Changed files with no mapped URL (0)
```

Seven URLs in three features, each listed because `SmtpEmailSender` is the registered
implementation of `IEmailSender`, which the endpoint's controller or page model injects.
The report ends with its limitations; the first one is the rule for reading it:

```
  - Candidate URLs to review, not proof of runtime impact: each is listed because a changed
    file is related to it in the map. An endpoint whose relationships did not change can
    still behave differently, and an unlisted endpoint can still be affected.
```

## 4. Narrow the list by reading the code

The map relates an endpoint to everything its class injects. Whether a given action calls
the service is a question for the code. Search the two injecting classes:

```sh
git grep -n "_emailSender.SendEmailAsync"
```

```
src/Web/Controllers/ManageController.cs:77:        await _emailSender.SendEmailAsync(User.Identity?.Name ?? "", "Confirm your email", "...");
src/Web/Pages/Order/Checkout.cshtml.cs:37:        await _emailSender.SendEmailAsync(buyer, "Order received", $"Order {orderId} received.");
```

- `POST /account/send-verification-email` and `POST /Order/Checkout` call the sender.
  Both pass `User.Identity?.Name ?? ""` (Checkout via `buyer`), so a signed-in user with no
  name now gets an exception instead of a silently dropped email. **That is the review
  comment for this PR:** decide whether those two actions should check the address first,
  and test both with a user whose name is empty.
- The five other `Manage` URLs inject the sender through the shared constructor but do not
  call it. They stay on the list as a smoke test, at lower priority.

## 5. Attach the review to the PR

```sh
git diff --name-status | ioco review EShopLayered.sln --format markdown --output email-sender-review.md
git diff --name-status | ioco review EShopLayered.sln --format html --output email-sender-review.html
```

The Markdown can be pasted into the PR description. The HTML file is self-contained and
records the evidence for every URL, the generation time, tool and schema versions, and the
limitations. Reports you generate stay yours and keep opening after a trial or
subscription ends.

To also see whether the change altered the map itself (an added endpoint, a new
dependency), save a baseline before the change with `ioco map EShopLayered.sln --json >
baseline.json` and add `--baseline baseline.json`. [Tutorial 3](03-client-handover-map.md)
sets up a shared baseline for a team.

## What IOCO did and did not do here

- It turned a one-file diff into seven URLs across three features, with the relationship
  that put each one there.
- It did not decide which URLs can reach the new exception; a two-line search did that.
- Services injected deeper than `--depth` (default 1) are not followed. If the changed
  class sits behind another service, run with `--depth 2`.

## Try it on your own app

```sh
git diff --name-status main...HEAD | ioco review path/to/YourApp.sln
```

Pick a recent PR that touched a shared service and compare the list with what was tested.

## Demo script (3 minutes)

1. (0:00) Show the PR title and the one-file diff.
2. (0:20) Show `SmtpEmailSender.cs  shared by 3 features` in the scatter section (or the
   VS Code Scatter node).
3. (0:45) Run `git diff --name-status | ioco review EShopLayered.sln`. Scroll the seven
   URLs.
4. (1:30) Read the first limitation aloud: candidates, not proof.
5. (1:50) `git grep` for `SendEmailAsync`; two callers, both can pass `""`.
6. (2:30) Open the HTML report and show where it records evidence and versions.
7. (2:50) Close: the review comment came from reading two files the map pointed to.
