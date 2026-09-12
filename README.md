# Oasis Ledger — unsigned iOS build via Capacitor + GitHub Actions

This wraps your app in a native iOS shell (Capacitor) and builds it on a
**free, cloud-hosted Mac** (GitHub Actions), so you never need your own Mac.

## 1. Get this building on GitHub (5 minutes)
1. Create a new **public** repository on github.com (public = free unlimited macOS build minutes).
2. Upload every file in this folder, keeping the folder structure (the `.github/workflows/build-ios.yml` file must stay at that exact path).
3. Go to the repo's **Actions** tab → you'll see "Build unsigned iOS IPA" → click **Run workflow**.
4. Wait ~5–10 minutes. When it finishes, open the run and download the **OasisLedger-unsigned-ipa** artifact — that's your `.ipa`.

That artifact is a real unsigned IPA, built on genuine Apple toolchain, with zero Mac or Xcode on your end.

## 2. The part no tooling can skip: getting it onto your phone
iOS itself — not this build process — refuses to run any app that isn't
signed at install time, even one built by Apple's own tools. There is no
option here, sideloading included, that gets around this; it's an OS-level
check, not a missing step in this project. Two realistic free routes:

- **AltStore / SideStore** — a small companion app you run once (on a Mac,
  Windows, *or* Linux machine — any of the three works) that signs the IPA
  with your free Apple ID and installs it. Free Apple ID signatures expire
  after 7 days, so it silently re-signs itself each time your phone connects
  to the same Wi-Fi as that computer (or over the internet with AltServer
  running).
- **Sideloadly** — same idea, Windows/Mac app, drag-and-drop the `.ipa`
  in with your Apple ID.

Both are one-time-per-week, not per-use — you don't need the computer open
while using the app, just for that periodic re-sign.

## 3. Before your first real build
Open `capacitor.config.json` and change `"appId"` from
`com.yourname.oasisledger` to something unique to you (reverse-domain style,
e.g. `com.jsmith.oasisledger`) — Apple's signing tools key off this.

## What was changed to make it feel like an iPhone app
- **No pinch/double-tap zoom** — the viewport now locks scale to 1, so it behaves like a fixed app layout instead of a webpage.
- **Bottom sheets instead of floating pop-ups** — on a phone, every modal now slides up from the bottom edge with rounded top corners and a small drag handle, matching how iOS presents forms and options, instead of appearing as a centred box with margins on all sides.
- **No accidental text-selection bubbles** — buttons, tabs, and nav items no longer show the "Copy / Select" callout when pressed and held.
- **Press feedback** — buttons and tabs shrink slightly when tapped, closer to how real iOS controls respond, instead of only changing colour.
- **No page-wide rubber-banding** — only the actual scrollable areas bounce at their edges now; the app shell itself stays put.
- **Real status bar styling** — when running as the compiled app (not the browser/PWA version), the status bar now matches the app's background instead of using whatever default iOS picks, which can otherwise make the clock/battery icons hard to read.
- **Light haptic tap** — buttons, tabs, nav items, and toggles give a small haptic buzz on tap, the same as native iOS controls. This only activates inside the compiled app; it does nothing in a browser.

The status bar and haptics need two extra Capacitor packages, `@capacitor/status-bar`
and `@capacitor/haptics` — already added to `package.json`, so the existing
GitHub Actions build picks them up automatically on the next run. No extra
steps needed on your end.

## Layout fixes so nothing gets visually cut off
- **Accounts list** — name, type, balance, and date used to fight for space in one table row on a phone. Each account now lays out as a stacked card (name, then type, then balance, then date) with the delete button tucked in the corner, so no field is ever squeezed.
- **Itemising a cash withdrawal** — same fix for the description/category/amount row: description now gets its own full line on narrow screens.
- **Long category, account, and transaction names** now wrap onto a second line instead of being truncated with "…". (Short, dense summary lists — the home-screen recent-transactions feed, KPI numbers, the calendar heatmap — still truncate/fit on one line by design; those are meant to stay scannable, and nothing there is a name you typed in.)

## Quick-add from a Shortcut, Back Tap, or Triple Back Tap (live budget updates, no manual save)
The app already auto-saves everything in the background the instant data changes — there's no "save" step in the UI to forget. What's new is a way to add a transaction *without opening the app*, and have it show up correctly the next time you look at the app — same as if you'd typed it in.

**One thing to know up front:** iOS doesn't let any third-party app listen for the Back Tap / Triple Back Tap gesture directly — that gesture only exists inside Settings → Accessibility → Touch → Back Tap, and its only job is to launch a Shortcut (or a system action). There's no API around that. So the real setup is: Back Tap triggers a Shortcut, and that Shortcut triggers Oasis Ledger via a link. The good news is once the Shortcut is built, it genuinely feels instant and one-tap.

The build now registers a custom link the app responds to:
```
oasisledger://add?amount=250&desc=Chai&cat=Chai%20%26%20coffee&type=expense&note=KFC
```
- `amount` — required, the number only (no currency symbol)
- `desc` — optional, what to call it (defaults to "Quick add")
- `cat` — optional, a category **name or id** (e.g. `Groceries` or `grocery` both work) — defaults to Uncategorized, which you can recategorize later in the app
- `type` — optional, `income` or `expense` — forces the direction; if you leave it out, it's inferred from whichever category you picked
- `note` — optional, fills in the new personal-note field on the transaction (see below) so you can skip typing it in the app afterwards

Opening that link pushes a real transaction into your data, saves it, and re-renders — so if the app is already open, the budget updates on screen immediately; if it's closed, it's already counted the next time you open it.

### Building the Shortcut (covers your income/expense → amount → description → category flow)
In the Shortcuts app, new Shortcut:
1. **Choose from Menu** — options "Expense" and "Income".
2. Under **Expense**: another **Choose from Menu** listing whichever categories you use most (e.g. Groceries, Dining out, Chai & coffee, Fuel, Ride-hailing…) — set the "type" variable to `expense` and "cat" to the matching name in each branch.
3. Under **Income**: same idea with Salary, Freelance & side income, Gifts & refunds, etc., and `type` set to `income`.
4. **Ask for Input** (Number) → "Amount".
5. **Ask for Input** (Text) → "Description" (what it was) — optionally a second **Ask for Input** (Text) for "Note" if you want the KFC-style personal note too.
6. **Text** action to build the URL: `oasisledger://add?amount=[Amount]&desc=[Description]&cat=[cat]&type=[type]&note=[Note]` — run "URL Encode" on Description and Note first (Shortcuts has a built-in action for this) in case you type spaces or punctuation.
7. **Open URLs** with that text.

Name this Shortcut something like "Add money". Then:
8. Go to **Settings → Accessibility → Touch → Back Tap**, and set **Double Tap** and/or **Triple Tap** to run it.

If you want *different* behavior on double vs. triple (say, double = quick expense, triple = quick income), just make two separate Shortcuts and assign one to each tap count — that's exactly what Back Tap supports natively, no extra plumbing needed on the app side.

## Personal notes on each transaction
Every transaction in the full Transactions list now has a small dashed-border text box under its description — type anything in it (e.g. category says "Dining out", note says "KFC") and it saves automatically as you type, completely separate from the original description. Leave it blank and nothing changes from before.

## The spending breakdown bar under each budget
Under each category's progress bar in the Budget view, if you've spent on more than one distinct thing in that category this month, a second thin bar now appears showing what your spending within that category is actually made of — split by whatever's in the note field (falling back to the transaction's description if you haven't added a note). Using your example: if "Food" categories add up to 6,000 and you've got 3,000 tagged "KFC" and 3,000 tagged "Ice Cream", that bar shows two even segments with a small legend underneath naming each and its share. Categories where everything shares one description just show the normal single bar, same as before.

Common category names to use in `cat=`: Groceries, Dining out, Food delivery, Chai & coffee, Fuel, Ride-hailing, Clothing, Online shopping, Doctor & hospital, Pharmacy, Subscriptions, Entertainment, Travel, Gifts & events, Zakat, Sadaqah & charity, Cash withdrawal, Salary, Freelance & side income. (Full list is the `C` array near the top of `www/index.html`.)

## What's in this folder
- `www/` — your app (same file you started with, already iOS-tuned)
- `capacitor.config.json` — tells Capacitor which web files to wrap and what to name/ID the app
- `package.json` — the Capacitor packages the build needs
- `.github/workflows/build-ios.yml` — the free cloud-Mac build pipeline (now also registers the `oasisledger://` link scheme)
