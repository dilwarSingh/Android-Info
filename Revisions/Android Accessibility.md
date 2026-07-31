# Android Accessibility — Last-Minute Revision

## Why Accessibility Matters

- **1.3B+ people** worldwide have disabilities (WHO, 2023); **~8% of males** and **~0.5% of females** have color blindness.
- Legal mandates: **ADA**, **WCAG**, **EN 301 549**, **European Accessibility Act (EAA)** (effective June 2025).
- Accessible apps reach a **wider audience** and improve **UX for all users**.

---

## Accessibility for Blind / Low Vision Users

- Blind/low-vision users rely on **TalkBack** (screen reader) and braille displays.
- Every UI element needs a **meaningful label**; content must follow a **logical reading order**.
- Never convey information through visuals alone.

**Common mistakes:**
- Missing `contentDescription` on images/buttons
- Decorative icons not excluded from accessibility tree
- Custom views without accessibility implementation
- `EditText` without associated label (`labelFor`)

---

## Accessibility for Color Blind Users

| Type | Confused Colors |
|------|----------------|
| **Deuteranopia** (most common) | Red & Green |
| **Protanopia** | Red appears dark/black |
| **Tritanopia** | Blue & Yellow |
| **Achromatopsia** | Full grayscale only |

- Never use **color alone** to convey state — pair with icons, text labels, or patterns.
- Prefer **Blue & Orange**, **Blue & Red**, **Black & Yellow**; avoid **Red & Green**.
- Ensure sufficient contrast even in grayscale.

```xml
<!-- BAD: color-only error indicator -->
<TextView android:textColor="@color/red" android:text="Invalid input" />

<!-- GOOD: color + icon + text -->
<ImageView android:src="@drawable/ic_error" android:contentDescription="Error" />
<TextView android:textColor="@color/red" android:text="Invalid input: Please enter a valid email" />
```

---

## Content Descriptions

- **`contentDescription`** is the primary attribute for screen reader support.
- Decorative views: set `android:contentDescription="@null"` + `android:importantForAccessibility="no"`.
- Don't include the view type in the description — TalkBack appends it automatically.
- Use `@string` resources for localization support; update descriptions dynamically when content changes.

```kotlin
imageView.contentDescription = "User profile picture for John Doe"
imageView.importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_NO
button.contentDescription = "Play ${song.title} by ${song.artist}"
```

---

## Labels & Input Accessibility

- Use **`android:labelFor`** on `TextView` to associate it with an `EditText` (API 17+).
- `android:hint` is read by TalkBack when no `labelFor` exists, but disappears on typing — prefer visible label + `labelFor`.
- **`accessibilityTraversalBefore` / `accessibilityTraversalAfter`** — control TalkBack reading order explicitly.

```xml
<TextView android:id="@+id/emailLabel" android:text="Email Address"
    android:labelFor="@id/emailInput" />
<EditText android:id="@+id/emailInput" android:hint="Enter your email"
    android:inputType="textEmailAddress" />
```

```kotlin
emailLabel.labelFor = emailInput.id  // programmatic association (API 17+)
```

---

## TalkBack Support

- Enable via `Settings → Accessibility → TalkBack` or ADB:
```bash
adb shell settings put secure enabled_accessibility_services com.google.android.marvin.talkback/com.google.android.marvin.talkback.TalkBackService
```

**Grouping elements** — set `contentDescription` on the container, mark children `importantForAccessibility="no"`.

**Live Regions** — auto-announce dynamic content changes:
```kotlin
textView.accessibilityLiveRegion = View.ACCESSIBILITY_LIVE_REGION_POLITE    // waits for speech to finish
textView.accessibilityLiveRegion = View.ACCESSIBILITY_LIVE_REGION_ASSERTIVE // interrupts immediately
view.announceForAccessibility("Your order has been placed successfully!")
```
> `AccessibilityEvent.obtain()` deprecated in **API 34** — use `AccessibilityEvent()` constructor instead.

---

## Switch Access & Motor Accessibility

- **Switch Access** lets users navigate with external switches, keyboard, or head movements.
- All interactive elements must be reachable in a **logical linear order**.
- Provide **visible focus indicators** on every focusable element.
- Avoid time-dependent interactions; always provide sufficient time or user-controlled dismissal.

```kotlin
// BAD: auto-dismissing
Snackbar.make(view, "Item deleted", Snackbar.LENGTH_SHORT).show()

// GOOD: user-controlled
Snackbar.make(view, "Item deleted", Snackbar.LENGTH_INDEFINITE)
    .setAction("Undo") { undoDelete() }.show()
```

---

## Focus & Navigation

- Use `android:nextFocusDown/Up/Left/Right` for explicit D-pad/keyboard traversal order.
- Set role programmatically via `AccessibilityDelegateCompat` — **`android:role` does not exist in XML**.
- Non-interactive views: `android:focusable="false"` + `android:importantForAccessibility="no"`.
- Custom views must be `focusable="true" clickable="true"` before a role can be meaningfully assigned.

```kotlin
ViewCompat.setAccessibilityDelegate(customView, object : AccessibilityDelegateCompat() {
    override fun onInitializeAccessibilityNodeInfo(host: View, info: AccessibilityNodeInfoCompat) {
        super.onInitializeAccessibilityNodeInfo(host, info)
        info.className = Button::class.java.name
    }
})
```

```kotlin
// TYPE_VIEW_FOCUSED is *input* focus and does NOT move the TalkBack cursor —
// perform ACTION_ACCESSIBILITY_FOCUS instead to move screen-reader focus after a screen change.
ViewCompat.performAccessibilityAction(
    view,
    AccessibilityNodeInfoCompat.ACTION_ACCESSIBILITY_FOCUS,
    null
)
```

---

## Text & Font Scaling

- Always use **`sp`** for text sizes — `dp` does not scale with user font settings.
- Allow text to wrap (`maxLines` + `ellipsize`) instead of clipping.

**Android 14+ (API 34) non-linear font scaling** — scales up to **200%**; large text scales less aggressively.
```kotlin
// GOOD: respects non-linear scaling
val textSizePx = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, 16f, resources.displayMetrics)

// BAD: ignores non-linear scaling on API 34+
val badTextSizePx = 16f * resources.configuration.fontScale
```
> On API 34+, `Configuration.fontScale` does not directly map to pixel scaling for all sizes — always use `TypedValue.applyDimension()`.

---

## Touch Target Sizes

- Minimum: **48dp × 48dp** for all interactive elements (Google recommendation).
- Expand hit area of small views with **`TouchDelegate`**:

```kotlin
val parent = button.parent as View
parent.post {
    val rect = Rect()
    button.getHitRect(rect)
    rect.top -= 20.dpToPx()
    rect.bottom += 20.dpToPx()
    rect.left -= 20.dpToPx()
    rect.right += 20.dpToPx()
    parent.touchDelegate = TouchDelegate(rect, button)
}
```

---

## Contrast Ratios

| Text Type | WCAG AA (min) | WCAG AAA |
|-----------|--------------|----------|
| Normal text (< 18pt) | **4.5:1** | 7:1 |
| Large text (≥ 18pt or bold 14pt) | **3:1** | 4.5:1 |
| UI components & graphics | **3:1** | — |

- Tools: **Android Studio Lint**, **Accessibility Scanner app**, **Colour Contrast Analyser**, WebAIM Contrast Checker.

---

## Custom Views Accessibility

- Standard widgets expose accessibility info automatically — custom views must do this manually via **`AccessibilityDelegateCompat`**.

```kotlin
ViewCompat.setAccessibilityDelegate(customView, object : AccessibilityDelegateCompat() {
    override fun onInitializeAccessibilityNodeInfo(host: View, info: AccessibilityNodeInfoCompat) {
        super.onInitializeAccessibilityNodeInfo(host, info)
        info.className = "android.widget.SeekBar"
        // Use the constructor; obtain() is deprecated
        info.rangeInfo = AccessibilityNodeInfoCompat.RangeInfoCompat(
            AccessibilityNodeInfoCompat.RangeInfoCompat.RANGE_TYPE_FLOAT,
            0f, 100f, currentValue
        )
        info.stateDescription = "Volume at ${currentValue.toInt()}%"
        info.isHeading = true
    }
})
```

- **`ExploreByTouchHelper`** — required when a single Canvas-drawn view contains multiple virtual interactive elements (e.g., a custom calendar). Implement `getVirtualViewAt()`, `getVisibleVirtualViews()`, `onPopulateNodeForVirtualView()`, `onPerformActionForVirtualView()`.

- **`ViewCompat.setScreenReaderFocusable(view, true)`** (API 28+) — makes a container focusable for screen readers without affecting keyboard focus.

---

## Accessibility Actions

- Custom actions allow switch/TalkBack users to trigger gestures programmatically (e.g., swipe-to-dismiss).

```kotlin
// Add custom action
val archiveAction = AccessibilityNodeInfoCompat.AccessibilityActionCompat(
    R.id.accessibility_action_archive, "Archive item")
info.addAction(archiveAction)

// Simple helper (no delegate boilerplate)
ViewCompat.addAccessibilityAction(itemView, "Dismiss") { _, _ ->
    dismissItem(position); true
}
```

**Compose:**
```kotlin
Modifier.semantics {
    customActions = listOf(
        CustomAccessibilityAction("Delete") { onDelete(); true },
        CustomAccessibilityAction("Archive") { onArchive(); true }
    )
}
```

---

## Pane Titles & Screen Transitions

- **`accessibilityPaneTitle`** — announces to TalkBack when a major section/screen becomes visible.

```kotlin
ViewCompat.setAccessibilityPaneTitle(fragmentContainer, "Search Results")  // programmatic
```
```xml
<FrameLayout android:accessibilityPaneTitle="Shopping Cart"> ... </FrameLayout>
```
```kotlin
// Compose
Modifier.semantics { paneTitle = "Order Summary" }
```

- Set pane title in `onViewCreated()` for Fragments; use `announceForAccessibility()` for full-screen transitions.

---

## Haptic Feedback

```kotlin
view.performHapticFeedback(HapticFeedbackConstants.CONFIRM)    // API 30+
view.performHapticFeedback(HapticFeedbackConstants.REJECT)     // API 30+
view.performHapticFeedback(HapticFeedbackConstants.LONG_PRESS)
view.performHapticFeedback(HapticFeedbackConstants.CONTEXT_CLICK) // API 23+
```

**Compose:**
```kotlin
val haptic = LocalHapticFeedback.current
haptic.performHapticFeedback(HapticFeedbackType.LongPress)
```

- Use for: action confirmation, error states, toggle changes, drag-and-drop.
- Don't overuse — excessive vibration loses meaning.

---

## Accessibility in Jetpack Compose

All accessibility in Compose flows through the **Semantics** system.

```kotlin
// Decorative image: contentDescription = null
Image(painter = ..., contentDescription = null)

// Merge children into single TalkBack node
Box(modifier = Modifier.semantics(mergeDescendants = true) {}) { ... }

// Role, state, custom action
Modifier.semantics {
    role = Role.Button
    contentDescription = "Play music"
    stateDescription = if (isPlaying) "Playing" else "Paused"
    onClick(label = "Play") { onPlayClick(); true }
}
```

- **`liveRegion = LiveRegionMode.Polite`** — announce dynamic text changes.
- **`heading()`** — mark a `Text` composable as a section heading.
- **`progressBarRangeInfo = ProgressBarRangeInfo(current, range)`** — expose progress to screen readers.
- Focus: use **`FocusRequester`** + `Modifier.focusRequester()` + `LaunchedEffect` to move focus programmatically.

---

## Testing Accessibility

| Method | What it catches |
|--------|----------------|
| **Accessibility Scanner** (Play Store) | Missing descriptions, small targets, low contrast, missing labels |
| **TalkBack manual testing** | Real end-to-end navigation experience |
| **Espresso** (`withContentDescription`, `isImportantForAccessibility`) | Automated label/role checks |
| **Compose** (`onNodeWithContentDescription`, `SemanticsMatcher`) | Semantic property assertions |
| **`./gradlew lint`** | `ContentDescription`, `ClickableViewAccessibility`, `KeyboardInaccessibleWidget`, `LabelFor` |
| **Color correction simulation** | `Settings → Accessibility → Color and motion → Color correction` (types: Deuteranomaly, Protanomaly, Tritanomaly, Grayscale) |

---

## Best Practices Checklist

**Visual**
- All images: meaningful `contentDescription` or `null` + `importantForAccessibility="no"` if decorative
- Text contrast ≥ **4.5:1** (normal) / **3:1** (large); UI components ≥ **3:1**
- No information conveyed by color alone; color-blind friendly palette

**Text & Scaling**
- All text in **`sp`**; layouts adapt to **200% font size**
- API 34+: use `TypedValue.applyDimension(COMPLEX_UNIT_SP, ...)` — never `fontScale * dp`

**Touch & Interaction**
- All touch targets ≥ **48dp × 48dp**
- Swipe-to-dismiss has an accessible action alternative
- No time-dependent interactions without user control

**Screen Reader**
- Interactive elements have content descriptions; decorative views excluded
- Dynamic content uses `liveRegion`; custom views implement `AccessibilityNodeInfoCompat`
- Canvas-drawn virtual views use `ExploreByTouchHelper`
- Input fields have `labelFor`; major sections have `accessibilityPaneTitle`

**Navigation**
- All features reachable via keyboard/D-pad and Switch Access
- Focus order is logical; focus managed correctly after navigation
- No focus traps outside of modal dialogs; focus indicators clearly visible

**Testing**
- App tested end-to-end with TalkBack enabled
- Tested with large font size (200%) and display size
- Tested with Switch Access / keyboard-only navigation
- Ran Accessibility Scanner and `./gradlew lint` for accessibility issues
- Tested color-blind simulation modes
- Automated Espresso/Compose accessibility assertions in place
