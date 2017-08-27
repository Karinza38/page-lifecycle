# A Lifecycle for the Web
## Motivation
For detailed motivation see [this doc](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#).

With large numbers of web apps (and tabs) running, critical resources such as memory, CPU, battery, network etc easily get oversubscribed, leading to a bad end user experience.
Application lifecycle is a key way that modern OS' manage resources. On Android, iOS and also more recent Windows versions, apps can be started and stopped at will by the platform. This lets the platform streamline and re-allocate resources where they best benefit the user.

On the web, we’ve tackled this with interventions on behalf of end users and built one-off features eg. reactive tab-discarding in extreme memory pressure - which can break websites. 
While this is okay in the short term, in the long term it is important to incorporate first class support in the web platform, create the right incentive structure for web developers, and allow the system to proactively reallocate resources and avoid getting into extreme resource situations.

For a platform to support application lifecycle, it needs to both provide developers with signals about transitions between the lifecycle states, AND provide lifecycle-compatible APIs that allow key capabilities to work even when the app is backgrounded or stopped.

The web ecosystem lacks a clear lifecycle. This proposal attempts to define what the lifecycle of a web page is and add necessary extensions to enable supporting two important system interventions necessary for resource re-allocation:
* Tab discarding for memory saving
* CPU stopping for battery saving

Whereas mobile platforms have rich service-bound APIs that allow apps to deliver their experience when backgrounded, most of the web platform's capabilities are tab-coupled. Audio for instance only works when the tab is alive, so when a tab is killed in the background that plays audio, there is no way to keep that tab playing sound. A [list of background use-cases is here](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.5kyzj3e4880y). In an ideal world, web apps would be able to deliver the experience they want to their users, without having to rely on their page always being resident and running on the machine.

### Lifecycle States
![Lifecycle States](https://github.com/spanicker/web-lifecycle/blob/master/LifecycleStates.png)

For details on the app lifecycle states and definitions see [this doc](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.edtdhepwctwy).
This proposal formalizes states for STOPPED and DISCARDED.

Lifecycle State | Visibility | Developer Expectation | System Interventions
--------------- | ---------- | --------------------- | --------------------
STOPPED | Not Visible | Hand off for background work and stop execution. | CPU suspension for battery saving: stop CPU after N minutes based on resource constraints
DISCARDED | Not Visible | System has discarded background tab to reclaim memory. If user revisits tab, this will reload the tab. | Tab discarding for memory saving: fully unloaded, no memory consumption.

### End-of-life scenarios
There are 3 high level scenarios for “end-of-life”.
#### 1. System Interventions
The system stops CPU usage and moves the app to STOPPED state, or the system discards the app (reclaims memory) and moves the app to DISCARDED state.  Handling this is in-scope for this proposal.\
For detailed Scenarios and Requirements, see the [list here](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.rsruvllnv993).
#### 2. User Exit
The user may close the tab (foreground or background) or navigate away OR on mobile, swipe the app away from task switcher. The user may background the app by minimizing the window OR on mobile by going to the homescreen and task switcher.\
*NOTE:* Handling user exit scenarios is out-of-scope for this proposal. We assume no changes there from today, although we try to be be consistent with existing handlers when reusing them.\
For categories of work that happen in end-of-life see the [list of End-of-life use-cases here](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.qftifuoc2315).
#### 3. Unexpected Termination
Apps can get killed in scenarios where it is not possible to deliver a callback, such as OOM crashes, OS kills the process under memory pressure, crashes or hangs due to browser bugs, device runs out of battery etc. Therefore it is possible for apps to transition from any state to TERMINATED without any callback being fired.\
*NOTE:* Improvements to handling unexpected termination is out-of-scope for this proposal.

## Proposal
![Lifecycle Callbacks](https://github.com/spanicker/web-lifecycle/blob/master/LifecycleCallbacks.png)

We propose the following changes:
* A `stopReason` attribute will be added to events for `pagehide` and `unload`; it will return `StopReason` enum to indicate why the event fired. 
* A `previousState` attribute will be added to event for `pageshow`; it will return `PreviousState` enum to indicate the preceding lifecycle state such as DISCARDED or STOPPED.
* `pagehide` is fired to signal BACKGROUNDED -> STOPPED. `StopReason` here is `stopped`.
* `pageshow` is fired to signal STOPPED -> ACTIVE. This will be used to undo what was done in `pagehide` above. `PreviousState` here is `stopped`.
* before moving app to DISCARDED the `beforeunload` handler will run and if it returns string (i.e. needs to show modal dialog) then the tab discard will be omitted.
* `unload` is fired to signal STOPPED -> DISCARDED. `StopReason` here is `discarded`. This will enable the app to persist transient view state (eg. user’s position in a dynamic list, progress in a game) prior to tab discarding so it can be restored if user revisits the tab; or do necessary final teardown such as releasing lock in Google Docs & Gmail.
* `pageshow` is fired to signal DISCARDED -> ACTIVE. This will be used to restore state persisted in `unload` above, when the user revisits a discarded tab. `PreviousState` here is `discarded`.\

*NOTE:* We have chosen to reuse existing callbacks (pagehide, pageshow, unload) vs. adding new callbacks. While this will cause some compat issues, it has the advantage of not adding further complexity to the platform, easier for browsers to implement (faster time to ship) and consequently better story for adoption and long term interop. For details [see here](https://docs.google.com/document/d/1UuS6ff4Fd4igZgL50LDS8MeROVrOfkN13RbiP2nTT9I/edit#heading=h.9tbw6aj3tl04).

### API sketch
```
enum StopReason { "discarded", "stopped", "userexit", “navigation”... };
```
Handle BACKGROUNDED -> STOPPED
```
function handlePageHide(e) {
   // feature detect
   if (e.stopReason) { ...
   // Handle transition to STOPPED
   if (e.stopReason == “stopped”) {
     // handle state transition BACKGROUNDED -> STOPPED
}
window.addEventListener("pagehide", handlePageHide);
```
Handle STOPPED -> DISCARDED
```
function handleUnload(e) {
   // feature detect
   if (e.stopReason) { ...
   if (e.stopReason == “discarded”) {
     // handle state transition STOPPED -> DISCARDED
}
window.addEventListener("unload", handleUnload);
```
Handle STOPPED -> ACTIVE or DISCARDED -> ACTIVE
```
enum PreviousState { "stopped", “discarded” };

function handlePageShow(e) {
   // feature detect
   if (e.previousState) { ...
   // Handle transition
   if (e.previousState == “stopped”) {
     // handle state transition STOPPED -> ACTIVE
   } else if (e.previousState == “discarded”) {
     // handle state transition DISCARDED -> ACTIVE
   }
}
window.addEventListener("pageshow", handlePageShow);
```
### Callbacks in State Transition Scenarios
* A. System stops (CPU suspension) background tab; user revisits\
[BACKGROUNDED] -------------> `onpagehide` (`StopReason: “stopped”`) [STOPPED]\
--(user revisit)----> `onpageshow` (`PreviousState: “stopped”`) [ACTIVE]

* B. System discards stopped tab; user revisits\
(previously called `onpagehide` (`StopReason: “stopped”`) ----> [STOPPED]\
----(tab discard)----> `onunload` (`StopReason: “discarded”`) [DISCARDED]\
--(user revisit)----> [LOADING] -> `onpageshow` (`PreviousState: “discarded”`) [ACTIVE]

* C. System discards background tab; user revisits\
[BACKGROUNDED] ---(tab discard)------>\
`onpagehide` (`StopReason: “stopped”`) [STOPPED], `onunload` (`StopReason: “discarded”`) [DISCARDED]\
--(user revisit)----> [LOADING] -> `onpageshow` (`PreviousState: “discarded”`) [ACTIVE]