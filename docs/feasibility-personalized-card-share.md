# Feasibility — Personalized Message in Card Share

**Verdict: feasible, build it.** The core design is sound and most of it is about a
day's work. Three of the six steps carry defects that would ship bugs as written,
and one step rests on infrastructure the app does not currently have.

---

## What this was checked against

The repository has no application source, so the analysis is grounded in the
compiled `AndroidManifest.xml` committed in `5f09315` — the shipped `com.g123`
build:

| | |
|---|---|
| Package | `com.g123` |
| Version | 1.175 (versionCode 1310) |
| minSdk / targetSdk | 24 / **36** |
| Stack | Expo + React Native (`expo.modules.*`, react-native-firebase, RevenueCat, RNCWebView) |

That confirms the document is targeting the right APIs — `react-native`'s `Share`
and `Linking` are the correct primitives for this app.

**Not verified:** the iOS side. No `Info.plist` is available in the repo, so every
iOS claim below is reasoned from platform behaviour, not from this app's config.

---

## Step-by-step assessment

### Step 1 — shared `buildCardShareText()` · sound

Ship as written. The empty-message guard is correct and `?.trim()` handles both
`undefined` and whitespace-only input.

Only nitpick: if the user's text already opens with the card title you get a
duplicated line. Not worth guarding against.

### Step 2 — generic Share sheet · correct, one API correction

The central advice — put the link inside `message` rather than the separate `url`
field — is right, and it is the non-obvious detail most implementations get wrong.
Keep it.

Two corrections:

- `title` is the **Android** chooser dialog title. On iOS it is largely ignored;
  `subject` is the field that reaches mail targets.
- `console.error` in the catch means a failed share is invisible to the user.
  Surface a toast.

One trade-off the document doesn't name: a URL inside the message string won't
produce a rich link preview in some iOS targets that would have rendered one from
the `url` field. That is the right trade — reliable delivery beats a preview.

### Step 3 — WhatsApp button · works, but the control flow is fragile

Two real defects:

1. **Nothing is caught.** Both `Linking.openURL()` calls are un-awaited and
   un-guarded. `openURL` rejects when no handler exists, so a failure produces an
   unhandled promise rejection and no fallback — the exact case this code exists
   to handle.
2. **The structure is backwards.** Ask-then-attempt makes the whole feature
   depend on `canOpenURL`, which is the one call the platforms restrict.

This matters more than it first appears, because `canOpenURL` is the *only* thing
Step 4's permissions gate:

- **iOS** — `LSApplicationQueriesSchemes` gates `canOpenURL` only. `openURL` is
  unaffected by it.
- **Android** — package visibility filtering gates queries and intent resolution.
  It does **not** block launching an implicit intent.

So attempt-then-fallback works on both platforms with no native changes at all:

```js
async function handleWhatsAppShare({ cardTitle, personalMessage, cardUrl }) {
  const text = encodeURIComponent(
    buildCardShareText({ cardTitle, personalMessage, cardUrl })
  );
  try {
    await Linking.openURL(`whatsapp://send?text=${text}`);
  } catch {
    try {
      await Linking.openURL(`https://wa.me/?text=${text}`);
    } catch {
      // surface a toast — both paths failed
    }
  }
}
```

Still do Step 4 — it is cheap, and it keeps `canOpenURL` honest if you ever want to
hide the WhatsApp button when WhatsApp is absent. But it stops being a blocker.

The `encodeURIComponent` reasoning in the document is correct, including the
comparison to `rawurlencode()`. Newlines become `%0A` and WhatsApp handles them;
emoji encode correctly as UTF-8.

### Step 4 — permission settings · necessary as stated, and currently missing

Confirmed directly in the shipped manifest. There **is** a `<queries>` block. It
declares:

- `<package name="com.facebook.katana">`
- `<package name="com.instagram.android">`
- intent queries for `VIEW`+`https`, `SEND`, `SENDTO`, `sms`/`smsto`, `tel`,
  billing, custom tabs, and media capture

**`com.whatsapp` is not among them, and no `whatsapp` scheme is queried.** With
`targetSdk="36"`, `canOpenURL('whatsapp://…')` therefore returns `false` on every
Android 11+ device today, whether or not WhatsApp is installed. The document's
diagnosis — the app silently falls back to the web link even when WhatsApp is
present — is exactly right.

Two things it misses:

1. **`com.whatsapp.w4b`.** WhatsApp Business is a separate package. Users who have
   only Business installed still fail the check. Declare both.
2. **The edit will not survive.** This is an Expo app and the manifest looks
   prebuild-generated (note the auto-derived `exp+123greetings` scheme).
   Hand-editing `android/app/src/main/AndroidManifest.xml` is overwritten by the
   next `expo prebuild`. On iOS, `ios.infoPlist.LSApplicationQueriesSchemes` in
   `app.json` is supported as the document says; on Android there is no equivalent
   `android.queries` field — verify against your Expo SDK version, and if it is
   absent this needs a small `withAndroidManifest` config plugin. Budget for the
   plugin, not for a one-line edit.

### Step 5 — strip formatting characters · defective as written; cut it

Three problems:

1. `clean` is computed and never used. Nothing wires it into
   `buildCardShareText()`.
2. `personalMessage.replace(...)` throws when the value is `undefined` — the core
   function uses `?.` for the same value.
3. The regex is indiscriminate. It strips `_` and `*` from ordinary text:
   `my_file`, `2*3`, `__init__` all lose characters silently.

There is also a reason not to do it at all: WhatsApp *renders* `*bold*` and
`_italic_` natively, so stripping removes a feature on the one channel this feature
is built around.

**Recommendation: skip this step.** Its own stated precondition almost certainly
applies — a plain `<TextInput>` produces plain text. If a rich-text editor is ever
introduced, strip at the editor boundary, not in the share path.

### Step 6 — what the link opens · right advice, wrong second half

"`cardUrl` should be a web link, not an app-only deep link" is correct and
important. Keep it.

"If the app is installed, that web page can then redirect into the app" is **not
true today on Android.** `com.g123.MainActivity` declares exactly two
intent-filters:

```
MAIN + LAUNCHER
VIEW + DEFAULT + BROWSABLE + scheme "exp+123greetings"
```

There is no `https` intent-filter for `123greetings.com` and no `autoVerify`. A
recipient tapping a card link gets a browser page and nothing else — there is no
handoff into the app to redirect *to*.

Building that is a separate project: an App Links intent-filter plus
`/.well-known/assetlinks.json` served from the domain (and an AASA file plus the
Associated Domains entitlement on iOS). It needs web-infrastructure access and the
release signing-certificate fingerprint. It should not block this feature — nothing
here depends on it.

---

## Gaps the document doesn't cover

**Release mechanics.** `expo.modules.updates.ENABLED` is `false` in the shipped
build, so there is no OTA channel. Every part of this — including the pure-JS
steps — ships in a store release. The upside: no reason to split the native change
into its own release, since it all travels together anyway.

**Message length.** There is no cap. Long text pushes `whatsapp://send?text=` and
`wa.me/?text=` past practical URL length limits, where it truncates or fails to
open with no error a user can interpret. Put a `maxLength` on the input (400–500
characters is generous for a card note) and show a counter.

**The "same message everywhere" promise.** The flow diagram claims one message for
every share option. The manifest queries `com.facebook.katana` and
`com.instagram.android`, which implies dedicated Facebook and Instagram buttons
already exist. Neither accepts pre-filled text — Facebook's platform policy
prohibits it and Instagram Stories takes media only. If those buttons exist, they
cannot honour the promise, and the UI should not imply that they will.

**State handling.** Unspecified: does the message box keep its text after the user
returns from WhatsApp? Is it cleared when they move to a different card? Retaining
per-card is the better default — people commonly share the same card twice.

**Analytics.** This is a growth feature with no measurement designed into it. Log
share attempts by channel and whether a personal message was present, or there is
no way to tell whether it worked.

**Testing.** `canOpenURL` cannot be exercised in the iOS simulator, which has no
WhatsApp. Physical devices are required — specifically one Android 11+ device with
WhatsApp installed, to prove the visibility fix actually landed.

**Safety.** Low risk. The message goes straight from the input into the OS share
sheet or WhatsApp and never reaches a server, so there is no stored-content,
escaping, or moderation surface. Worth confirming that stays true: if the personal
message is ever persisted or rendered server-side, it becomes a moderation problem
and an escaping problem at the same time.

---

## Effort

| Work | Estimate |
|---|---|
| Steps 1–3, message box UI, character counter, state | 1–1.5 d |
| Step 4, including Expo config plugin and both WhatsApp packages | 0.5 d |
| Analytics instrumentation | 0.25 d |
| Device QA — iOS and Android 11+, WhatsApp present and absent | 1 d |
| **Total** | **~3 days** |
| Step 6 App Links — separate track, needs web infra + signing fingerprint | 2–3 d |

Step 5 is cut, not estimated.

---

## Recommendation

Build steps 1, 2, 3, 4 and the first half of 6 as a single change. Restructure
step 3 to attempt-then-fallback so the feature does not depend on `canOpenURL`.
Drop step 5. Track App Links as separate work, and correct the "same message for
every share option" claim before it reaches the UI.
