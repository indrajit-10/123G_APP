# Crash analysis — `NSInvalidArgumentException` in Firebase A/B Testing (iOS 2.4.4 build 6)

| | |
|---|---|
| Bundle | `com.123Greetings` |
| Version | 2.4.4 (6) |
| Crashlytics issue | `181010225a8f7695edf124ff32b9c406` |
| Session | `d6b139e753424dceac6be5c51541c7e1_DNE_0_v2` |
| Event time | Tue 29 Sep 2026 08:16:52 IST |
| Exception | `NSInvalidArgumentException` |
| Crashing thread | libdispatch serial queue worker (**not** the main thread) |

## Verdict

This is **not a bug in 123Greetings app code**, and **not a React Native / JavaScript
bug**. It is a nil dictionary key inserted inside the Google/Firebase SDKs while
processing an **A/B Testing (ABT) experiment payload fetched from Firebase's
servers**.

The practical consequence: the fastest fix is **server-side** (stop or roll back the
recently-started A/B Test / Remote Config personalization in the Firebase console),
because it needs no app release. The durable fix is a **Firebase iOS SDK upgrade**.

## What the stack actually says

### The throwing frame

```
Fatal Exception: NSInvalidArgumentException
2  CoreFoundation     -[__NSDictionaryM setObject:forKeyedSubscript:]
3  libdispatch.dylib  _dispatch_call_block_and_release
5  libdispatch.dylib  _dispatch_lane_serial_drain
7  libdispatch.dylib  _dispatch_root_queue_drain_deferred_wlh
8  libdispatch.dylib  _dispatch_workloop_worker_thread
```

Three facts fall straight out of these five frames:

1. **`setObject:forKeyedSubscript:` on `__NSDictionaryM`** is the compiler's
   lowering of Objective-C subscript assignment — `dict[key] = value` — on a
   *mutable* dictionary. It raises `NSInvalidArgumentException` in exactly one
   situation: **the key is `nil`** (`*** setObjectForKey: key cannot be nil`).
   A `nil` *value* is not an error here — per the documented contract it removes
   the entry. So the failure is a nil/missing **key**, not a nil value.

2. **`_dispatch_call_block_and_release` directly above it, with no enclosing
   frame**, means the block was enqueued with `dispatch_async` and the code that
   submitted it is long gone. The submitting call site is therefore *absent from
   this stack* — which is why no 123Greetings frame appears. This also means the
   insertion **cannot be wrapped in `@try/@catch` by the app**: by the time it
   throws, no app frame is on the stack to catch it.

3. **`_dispatch_lane_serial_drain`** identifies the block as running on a
   **serial** queue — the classic "NSMutableDictionary guarded by a private
   serial queue" concurrency pattern.

### The frame that names the culprit

The crashing stack alone has no app or SDK symbols. The attribution comes from a
*different, still-live* thread in the same report:

```
APMExperimentWorkerQueue
 4  libdispatch.dylib  _dispatch_sync_f_slow
 5  123Greetings       -[GULMutableDictionary dictionary]  (GULMutableDictionary.m:98)
 6  123Greetings       -[APMEExperiment copyWithZone:]
 7  123Greetings       -[APMESnapshot initWithSDKName:experiments:]
 8  123Greetings       -[APMESnapshot initWithProtobuf:]
 9  123Greetings       -[APMETaskManager experimentSnapshotsFromExperimentResponse:]
10  123Greetings       -[APMETaskManager handleFetchingExperimentsResponse:data:error:]
11  123Greetings       __35-[APMETaskManager fetchExperiments]_block_invoke
```

Read bottom-up, this is the Firebase Analytics **experiments** pipeline
(`APME*` = Analytics Measurement Experiments, the ABT integration):

- `fetchExperiments` → network response arrives
- `handleFetchingExperimentsResponse:data:error:` → parse it
- `initWithProtobuf:` → decode the experiment protobuf
- `initWithSDKName:experiments:` → build a snapshot
- `-[APMEExperiment copyWithZone:]` → deep-copy each experiment
- `-[GULMutableDictionary dictionary]` → read the backing store…
- `_dispatch_sync_f_slow` / `__DISPATCH_WAIT_FOR_QUEUE__` → **blocked, waiting for
  a serial queue to become available**

`GULMutableDictionary` (GoogleUtilities) is precisely the
"`NSMutableDictionary` + private serial queue" wrapper described above: its reads
`dispatch_sync` onto that queue and its writes `dispatch_async` a
`self->_objects[key] = object` subscript assignment onto it. So at the moment of
the crash we have, on the same serial queue:

- a **queued async write** doing `dict[key] = value` → throwing on a nil key
  (the crashed thread), and
- a **sync read** from `-[GULMutableDictionary copyWithZone:]`'s `dictionary`
  accessor, parked waiting on that queue (`APMExperimentWorkerQueue`).

Those two halves fit together exactly. The nil key is an experiment attribute
that the decoded protobuf did not supply — an experiment ID, variant ID, or SDK
name that is missing from the server payload and so decodes to `nil`, then gets
used as a dictionary key without a nil check.

One caveat, stated plainly: because the throwing frame is an orphaned async
block, the link between it and the `APMExperimentWorkerQueue` thread is inference
from co-occurrence on the same serial-queue pattern, not a single unbroken call
chain. It is the only reading consistent with all the frames present, but it is a
reading.

### Timeline corroboration

From the breadcrumb log:

| Time (IST) | Event |
|---|---|
| 08:16:42 | `screen_view` — `UIViewController` |
| 08:16:42 | `screen_view` — `UIViewController` |
| 08:16:44 | `screen_view` — `RNSScreen` |
| 08:16:47 | `screen_view` — `RNSScreen` |
| 08:16:50 | `screen_view` — `RNSScreen` |
| **08:16:52** | **crash** |

The whole session is ~10 seconds long and the crash lands 2 seconds after the
last screen view. `fetchExperiments` runs shortly after Firebase initialises, so
this is a **cold-start crash**, not the result of any particular user action. The
`UIViewController` → `RNSScreen` progression is just ordinary
`react-native-screens` navigation; nothing in it is implicated.

That startup timing matters for impact: users hitting this will see the app die
seconds after launch, repeatedly, with no way to work around it — so treat it as
high severity even if the affected-user count looks modest.

### What is ruled out

- **JavaScript / React Native product code** — `com.facebook.react.runtime.JavaScript`
  is idle in `CFRunLoopRun`, Hermes' `hades` GC thread is parked in
  `pthread_cond_wait`. No JS on any running stack.
- **The main thread** — parked in `mach_msg2_trap` inside `CFRunLoopRun`, i.e.
  idle and healthy.
- **Our own dictionary mutations** — no 123Greetings frame appears in the
  crashing stack at all.
- **A concurrency race on an unguarded dictionary** — the serial drain shows the
  dictionary *was* correctly queue-guarded. The bug is data validation, not
  thread safety. (A raw data race would more typically surface as
  `EXC_BAD_ACCESS`, or as `NSGenericException: mutated while being enumerated`.)

## Remediation

### 1. Immediate mitigation — Firebase console, no app release (do this first)

The triggering payload is served from Firebase, so the console is a kill switch
that reaches users already running 2.4.4.

- Firebase console → **A/B Testing**. Sort experiments by start date and find any
  **started, changed, or rolled out on or just before 29 Sep 2026** — this is a
  new crash signature, so a recent activation is the prime suspect.
- **Stop** (or roll back) that experiment and confirm the crash rate falls in
  Crashlytics over the following hours.
- Firebase console → **Remote Config → Personalization**. Personalizations ride
  the same ABT pipeline; if one was added recently, stop it too.

### 2. Inspect the experiment metadata for blank fields

While the experiment is stopped, look for the missing field that decodes to nil:

- Any **variant with an empty or whitespace-only name**.
- **Remote Config parameter keys** targeted by the experiment that are empty, or
  that contain leading/trailing whitespace or non-ASCII characters.
- An experiment in a **half-configured draft state** that was still published.

Recreating the experiment with every field explicitly populated is usually enough
to avoid the payload shape that triggers this.

### 3. Durable fix — upgrade the Firebase iOS SDK

Nil-key hardening in `GULMutableDictionary` and stricter ABT payload validation
have both been tightened over successive releases, so an upgrade is the real fix.

```sh
cd ios
pod outdated                 # check Firebase / GoogleUtilities / FirebaseABTesting
pod repo update
pod update Firebase FirebaseAnalytics FirebaseABTesting FirebaseCrashlytics GoogleUtilities
pod install
```

Pin the result in the `Podfile` so the fix cannot regress, and verify with
`grep -A2 'Firebase' ios/Podfile.lock`. Do **not** hand-edit `Podfile.lock`.

Check the [Firebase iOS SDK release notes](https://firebase.google.com/support/release-notes/ios)
for the current version rather than trusting a version number written down here —
and confirm the upgrade against the React Native Firebase version in
`package.json`, since `@react-native-firebase/*` constrains the native pods it
accepts. Upgrade the JS packages in step with the pods if they conflict.

### 4. If A/B Testing is not actually used, remove it

If the product isn't running experiments, the cheapest permanent fix is deleting
the code path. Drop `FirebaseABTesting` (and `@react-native-firebase/remote-config`
if Remote Config is likewise unused) from the `Podfile` / `package.json`, and
leave Personalization off in the console. `FirebaseAnalytics` and
`FirebaseCrashlytics` are unaffected and should stay.

### 5. What will *not* work

Worth stating so nobody spends a cycle on it:

- **`@try/@catch` around our own code** — the throw happens in an orphaned
  `dispatch_async` block inside the SDK; there is no app frame to catch it.
- **`NSSetUncaughtExceptionHandler`** — runs *after* the process is already
  committed to terminating. It changes reporting, not survival.
- **Any JavaScript-side error boundary** — the exception is native and never
  crosses the bridge.

### 6. Verification

- Watch the Crashlytics issue's **version-and-time breakdown**, not just the
  total: confirm new events stop arriving after the console change, and that they
  stay stopped once the SDK upgrade ships.
- Add a Crashlytics custom key at Remote Config / experiment fetch time (e.g.
  `abt_experiment_ids`) so any recurrence names the offending experiment directly
  instead of requiring this inference again.
- Keep the stopped experiment stopped until an SDK build with the fix is live in
  production, then re-introduce it and watch for a day.

## Assumptions

Recorded because they are not verifiable from the crash report alone:

- No iOS or React Native source is committed to this repository (it holds only
  `README.md` and `docs/`), so none of the above is verified against the app's
  actual `Podfile`, `Podfile.lock`, or Firebase initialisation code. The Podspec
  and package names in §3–§4 should be checked against the real project.
- The claim that a recent experiment activation triggered this rests on the crash
  being newly reported; the report itself carries no experiment identifiers.
