---
layout: post
title: "SwiftUI State Hazards: Race Conditions Between Static and Reactive Data"
date: 2025-12-05 09:00:00 +0900
categories: ios swiftui architecture engineering
author: raykim
author_url: https://github.com/raykim2414
subtitle: "Error Case #6 · Race Conditions Between Static and Reactive Data"
---

In SwiftUI architecture, relying on a **Single Source of Truth** is the golden rule. However, real-world applications often have to bridge the gap between modern SwiftUI views and legacy code or global managers that rely on static variables.

This post explores a common but subtle architectural bug: a race condition that occurs when mixing **Reactive State** (like `@Published`) with **Static Variables**, and how to resolve it using proper event ordering and state containment.

## The Problem Pattern: Mixed State Dependency

The issue arises when a View's rendering logic depends on two different types of data sources simultaneously:

1.  **Reactive State**: A `@Published` property (e.g., `appState.isReady`) that automatically triggers a view re-render.
2.  **Static/Global State**: A static variable (e.g., `GlobalConfig.sharedValue`) that does **not** trigger re-renders.

### The Scenario

Consider a high-concurrency async flow (like authentication, configuration loading, or session setup) where the system transitions from "Loading" to "Active".

1.  The system broadcasts an event: "Setup Complete".
2.  The View receives this update via the Reactive State (`isReady = true`) and attempts to render the "Active" screen.
3.  Inside the `body`, the View checks a secondary condition relying on the Static State (`!GlobalConfig.isEmpty`).

### Code Example: The Race in Action

Here is a minimal reproducible example of how this bug manifests.

```swift
// A legacy global configuration holder
class GlobalConfig {
    static var userId: String?
}

// The SwiftUI View's State Object
class AppState: ObservableObject {
    @Published var isReady = false

    func completeSetup() {
        // ❌ DANGER: Triggering the View update BEFORE data is ready
        isReady = true  // 1. Triggers View body re-evaluation immediately
        
        // 2. Too late! The View has already tried to read GlobalConfig.userId
        // and found it nil/empty.
        GlobalConfig.userId = "123" 
    }
}

struct ContentView: View {
    @StateObject var state = AppState()
    
    var body: some View {
        if state.isReady {
            // If GlobalConfig.userId hasn't been set yet, this Text might crash or be empty
            Text("Hello, user \(GlobalConfig.userId ?? "Unknown")")
        } else {
            ProgressView()
        }
    }
}
```

**The Race Condition**:
If the `isReady` assignment happens *microseconds before* the Static Variable `GlobalConfig.userId` is assigned, the View renders based on the **new** Reactive State (`true`) but the **old** Static State (`nil`). Since the Static Variable update happens *after* the render pass and doesn't trigger a new render itself, the UI gets stuck in an invalid intermediate state (e.g., "Hello, user Unknown").

## Root Cause Analysis

The core failure is an **Event Ordering Mismatch** combined with **Non-Reactive Dependencies**.

- **Execution Order**:
    1.  `isReady = true` (Reactive trigger)
    2.  View evaluates `body` (Reactive State is true, Static is empty).
    3.  `GlobalConfig.userId` assigned (Too late).
- **Silent Failure**: Because the View doesn't observe the Static Variable, the subsequent assignment is ignored by the UI layer, leaving the user on a blank or incorrect screen.

### Why QA Missed It: The Debug vs. Release Trap

You might wonder, "Why didn't we catch this during testing?" This is a classic Heisenbug that often behaves differently in Debug builds versus Release builds.

*   **Debug Builds**: Slower execution, extra logging, and lack of compiler optimizations might introduce enough latency between instructions that `GlobalConfig.userId` happens to get set *just before* the View completes its render cycle.
*   **Release Builds**: With aggressive compiler optimizations and faster execution, the CPU instructions for `isReady = true` trigger the UI update cycle instantly. The main thread might prioritize the UI layout pass before the subsequent line of code (`GlobalConfig.userId = ...`) effectively "settles" in memory for the View to read, especially if these operations are hopping between threads or actors.

## Architectural Solutions

To build a robust system that handles fast async flows without race conditions, we apply three architectural fixes.

### 1. Eliminate Static Dependencies in Views (Single Source of Truth)

Views should never read static variables for conditional branching. Instead, they should rely 100% on the observable state provided by their ViewModel or Store.

*   **Bad**: `if viewModel.isReady && !GlobalData.isEmpty { ... }`
*   **Good**: `if viewModel.isReady { ... }`

The ViewModel should encapsulate the validation of the global data and expose a single boolean that represents the "truth" to the View.

### 2. Enforce "Write-Before-Notify"

In your data managers or services, you must guarantee that data consistency is established *before* notifying listeners.

**Incorrect:**
```swift
notifyObservers(.didUpdate) // ⚠️ Listeners read old data here
GlobalData.value = newValue
```

**Correct:**
```swift
GlobalData.value = newValue // ✅ Data is consistent
notifyObservers(.didUpdate) // Listeners read new data
```

### 3. Explicit State Synchronization

For critical state transitions, the ViewModel should explicitly synchronize its internal state with the external systems using Reactive tools like Combine or modern Concurrency. This ensures the View only updates when the data is *actually* available.

**Solution A: Using Combine**

Instead of manually toggling booleans, derive the state from the data itself.

```swift
class SafeAppState: ObservableObject {
    @Published var isReady = false
    private var cancellables = Set<AnyCancellable>()

    init() {
        // Listen to the actual data source changes
        GlobalConfigPublisher.shared.$userId
            .receive(on: RunLoop.main)
            // Only become "ready" when we have a valid ID
            .map { $0 != nil } 
            .assign(to: \.isReady, on: self)
            .store(in: &cancellables)
    }
}
```

**Solution B: Using Async/Await**

If you are using imperative logic, ensure strict ordering on the Main Actor.

```swift
class SafeAppState: ObservableObject {
    @Published var isReady = false
    
    @MainActor
    func setup() async {
        // 1. Await the actual data operation
        let userId = await fetchUserId()
        
        // 2. Update the global store
        GlobalConfig.userId = userId
        
        // 3. Update the View state LAST
        // This guarantees that when the View renders, GlobalConfig.userId is set.
        self.isReady = true
    }
}
```

## Takeaways

1.  **Decouple Views from Globals**: SwiftUI Views are functions of their state. If that state lives in a static variable outside the SwiftUI dependency graph, the View becomes unpredictable.
2.  **Atomic Updates**: Treat "updating a value" and "notifying about the update" as an atomic operation where the write always precedes the notification.
