# Android App Accessibility Guide

## Making Android Apps Accessible to All Users

Accessibility in Android development means designing apps that can be used by **everyone**, including people with visual impairments (blind or low vision), color blindness, motor disabilities, hearing impairments, and cognitive disabilities.

---

## Table of Contents

1. [Why Accessibility Matters](#why-accessibility-matters)
2. [Accessibility for Blind / Low Vision Users](#accessibility-for-blind--low-vision-users)
3. [Accessibility for Color Blind Users](#accessibility-for-color-blind-users)
4. [Content Descriptions](#content-descriptions)
5. [Labels & Input Accessibility](#labels--input-accessibility)
6. [TalkBack Support](#talkback-support)
7. [Switch Access & Motor Accessibility](#switch-access--motor-accessibility)
8. [Focus & Navigation](#focus--navigation)
9. [Text & Font Scaling](#text--font-scaling)
10. [Touch Target Sizes](#touch-target-sizes)
11. [Contrast Ratios](#contrast-ratios)
12. [Custom Views Accessibility](#custom-views-accessibility)
13. [Accessibility Actions](#accessibility-actions)
14. [Pane Titles & Screen Transitions](#pane-titles--screen-transitions)
15. [Haptic Feedback](#haptic-feedback)
16. [Accessibility in Jetpack Compose](#accessibility-in-jetpack-compose)
17. [Testing Accessibility](#testing-accessibility)
18. [Best Practices Checklist](#best-practices-checklist)

---

## Why Accessibility Matters

- Over **1.3 billion people** worldwide live with some form of disability (WHO, 2023).
- **~8% of males** and **~0.5% of females** have some form of color blindness.
- Accessibility is required by law in many countries (ADA, EAA) and measured against standards (WCAG, EN 301 549).
- The **European Accessibility Act (EAA)** takes effect June 2025, requiring digital products to be accessible.
- Accessible apps reach a **wider audience** and improve **UX for all users**.

---

## Accessibility for Blind / Low Vision Users

Blind and low-vision users rely on **screen readers** (TalkBack on Android) and **braille displays** to interact with apps.

### Key Principles

| Principle | Description |
|-----------|-------------|
| **Meaningful Labels** | Every UI element must have a descriptive label |
| **Logical Reading Order** | Content should be read in a logical sequence |
| **Non-visual Feedback** | Avoid conveying info only through visuals |
| **Screen Reader Compatibility** | App must work fully with TalkBack enabled |

### Common Mistakes to Avoid

- ❌ Using images without `contentDescription`
- ❌ Decorative icons that interrupt reading flow
- ❌ Custom views that aren't accessible
- ❌ Touch targets that are too small
- ❌ Modals that trap focus incorrectly
- ❌ Missing `labelFor` on input fields
- ❌ Using `EditText` without an associated label

---

## Accessibility for Color Blind Users

Color blindness affects the ability to distinguish between certain colors. Types include:

| Type | Description | Affected Colors |
|------|-------------|-----------------|
| **Deuteranopia** | Red-Green (red-green deficiency is the most common type) | Red & Green look similar |
| **Protanopia** | Red-Green | Red appears dark/black |
| **Tritanopia** | Blue-Yellow | Blue & Yellow confusion |
| **Achromatopsia** | Full color blindness | Only sees grayscale |

### Design Strategies for Color Blind Users

#### ✅ DO: Use Multiple Visual Cues

Never rely on **color alone** to convey information. Always combine color with:
- Icons / symbols
- Text labels
- Patterns or textures
- Shape differences

```xml
<!-- BAD: Only color used to show error -->
<TextView
    android:textColor="@color/red"
    android:text="Invalid input" />

<!-- GOOD: Color + icon + text -->
<LinearLayout
    android:orientation="horizontal">
    <ImageView
        android:src="@drawable/ic_error"
        android:contentDescription="Error" />
    <TextView
        android:textColor="@color/red"
        android:text="Invalid input: Please enter a valid email" />
</LinearLayout>
```

#### ✅ DO: Use Color-Blind Friendly Palettes

Prefer these color combinations:
- **Blue & Orange** (universally distinguishable)
- **Blue & Red**
- **Black & Yellow**
- Avoid **Red & Green** combinations

#### ✅ DO: Use Sufficient Contrast

Ensure text and UI elements have enough contrast even in grayscale.

---

## Content Descriptions

`contentDescription` is the most critical attribute for screen reader support.

### XML Layout

```xml
<!-- Image with meaningful description -->
<ImageView
    android:id="@+id/profileImage"
    android:src="@drawable/ic_profile"
    android:contentDescription="@string/profile_picture_description" />

<!-- Decorative image (should be ignored by screen reader) -->
<ImageView
    android:src="@drawable/ic_decorative_divider"
    android:contentDescription="@null"
    android:importantForAccessibility="no" />

<!-- Button with clear label -->
<ImageButton
    android:src="@drawable/ic_send"
    android:contentDescription="@string/send_message" />
```

### Programmatically in Java/Kotlin

```kotlin
// Setting content description
imageView.contentDescription = "User profile picture for John Doe"

// For decorative views
imageView.importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_NO

// Dynamic content descriptions
button.contentDescription = "Play ${song.title} by ${song.artist}"
```

### Rules for Good Content Descriptions

- ✅ Be concise but meaningful ("Submit form" not just "Button")
- ✅ Don't include the view type ("Submit" not "Submit button" — TalkBack adds that)
- ✅ Use `@string` resources (supports localization)
- ✅ Update descriptions dynamically when content changes
- ❌ Don't leave image buttons with no description
- ❌ Don't use file names as descriptions ("ic_send_24dp")

---

## Labels & Input Accessibility

Associating labels with input fields is essential for screen reader users to understand what each field expects.

### Using `labelFor` in XML

```xml
<!-- Associate a label with its input field -->
<TextView
    android:id="@+id/emailLabel"
    android:text="Email Address"
    android:labelFor="@id/emailInput" />

<EditText
    android:id="@+id/emailInput"
    android:hint="Enter your email"
    android:inputType="textEmailAddress" />
```

### Using `labelFor` Programmatically

```kotlin
// Associate a label with an input field (API 17+)
// labelFor is a View property; for back-compat use ViewCompat.setLabelFor()
emailLabel.labelFor = emailInput.id
```

### Hints as Accessible Labels

```xml
<!-- When no visible label exists, use android:hint -->
<EditText
    android:id="@+id/searchField"
    android:hint="Search products"
    android:inputType="text" />
```

> **Note:** `android:hint` is read by TalkBack as the field label when no `labelFor` association exists. However, the hint disappears when the user starts typing. Prefer a visible label with `labelFor` for best accessibility.

### `accessibilityTraversalBefore` / `After`

Control the order in which TalkBack reads elements:

```xml
<!-- Force TalkBack to read the warning before the button -->
<TextView
    android:id="@+id/warningText"
    android:text="This action cannot be undone"
    android:accessibilityTraversalBefore="@id/deleteButton" />

<Button
    android:id="@+id/deleteButton"
    android:text="Delete" />
```

---

## TalkBack Support

**TalkBack** is Android's built-in screen reader. It reads content descriptions, announces UI changes, and allows navigation via swipe gestures.

### Enabling TalkBack (for testing)

```
Settings → Accessibility → TalkBack → Enable
```

Or via ADB:
```bash
adb shell settings put secure enabled_accessibility_services com.google.android.marvin.talkback/com.google.android.marvin.talkback.TalkBackService
```
> **Note:** This package name is for the Google TalkBack app (`com.google.android.marvin.talkback`). On some OEM devices the screen reader may use a different package name (e.g., Samsung ships its own TalkBack variant). Verify the actual package name on a target device with: `adb shell settings get secure enabled_accessibility_services`

### Grouping Elements for TalkBack

```xml
<!-- Group related elements so TalkBack reads them together -->
<LinearLayout
    android:focusable="true"
    android:contentDescription="John Doe, Software Engineer, Online"
    android:importantForAccessibility="yes">

    <ImageView android:importantForAccessibility="no" ... />
    <TextView android:text="John Doe" android:importantForAccessibility="no" />
    <TextView android:text="Software Engineer" android:importantForAccessibility="no" />
    <TextView android:text="Online" android:importantForAccessibility="no" />

</LinearLayout>
```

### Live Regions (Announcing Dynamic Changes)

```xml
<!-- Announces changes automatically to screen reader -->
<TextView
    android:id="@+id/statusText"
    android:accessibilityLiveRegion="polite"
    android:text="Loading..." />
```

```kotlin
// Polite: waits for current speech to finish
textView.accessibilityLiveRegion = View.ACCESSIBILITY_LIVE_REGION_POLITE

// Assertive: interrupts current speech immediately
textView.accessibilityLiveRegion = View.ACCESSIBILITY_LIVE_REGION_ASSERTIVE

// Trigger an announcement manually
// ⚠️ DEPRECATED in Android 16 (API 36): announceForAccessibility() is deprecated.
// Prefer setAccessibilityLiveRegion() (above), setAccessibilityPaneTitle(), or Activity.setTitle().
view.announceForAccessibility("Your order has been placed successfully!")
```

> **Note:** `AccessibilityEvent.obtain()` was deprecated in API 34 (Android 14). Use the `AccessibilityEvent()` constructor instead. `announceForAccessibility()` itself was deprecated in **Android 16 (API 36)** — use `View.setAccessibilityLiveRegion()` or `ViewCompat.setAccessibilityPaneTitle()` instead.

---

## Switch Access & Motor Accessibility

**Switch Access** allows users with motor disabilities to interact with Android devices using external switches, a keyboard, or head movements instead of the touchscreen.

### Key Design Principles for Switch Access

| Principle | Description |
|-----------|-------------|
| **Linear Navigation** | All interactive elements must be reachable in a logical linear order |
| **Clear Focus Indicators** | Focus highlight must be clearly visible on all elements |
| **No Time-Dependent Actions** | Avoid interactions that require quick responses |
| **Action Alternatives** | Every gesture must have a switch-accessible alternative |

### Enabling Switch Access (for testing)

```
Settings → Accessibility → Switch Access → Enable
```

### Ensuring Compatibility

```xml
<!-- Ensure all interactive views have visible focus states -->
<Button
    android:id="@+id/actionButton"
    android:text="Submit"
    android:focusable="true"
    android:background="?attr/selectableItemBackground" />
```

```kotlin
// Provide custom focus highlight for custom views
view.setOnFocusChangeListener { v, hasFocus ->
    v.background = if (hasFocus) {
        ContextCompat.getDrawable(context, R.drawable.focused_background)
    } else {
        ContextCompat.getDrawable(context, R.drawable.default_background)
    }
}
```

### Avoiding Time-Based Interactions

```kotlin
// ❌ BAD: Auto-dismissing Snackbar without user control
Snackbar.make(view, "Item deleted", Snackbar.LENGTH_SHORT).show()

// ✅ GOOD: Snackbar with action and longer duration
Snackbar.make(view, "Item deleted", Snackbar.LENGTH_INDEFINITE)
    .setAction("Undo") { undoDelete() }
    .show()
```

---

## Focus & Navigation

### Keyboard / D-Pad Navigation

```xml
<!-- Define explicit focus traversal order -->
<EditText
    android:id="@+id/firstNameField"
    android:nextFocusDown="@id/lastNameField"
    android:nextFocusRight="@id/lastNameField" />

<EditText
    android:id="@+id/lastNameField"
    android:nextFocusDown="@id/emailField"
    android:nextFocusUp="@id/firstNameField" />
```

### Focusable Elements

```xml
<!-- Make a custom view focusable and accessible as a button -->
<View
    android:focusable="true"
    android:clickable="true" />
```

```kotlin
// Set the role/class name programmatically (android:role does NOT exist in XML)
ViewCompat.setAccessibilityDelegate(customView, object : AccessibilityDelegateCompat() {
    override fun onInitializeAccessibilityNodeInfo(
        host: View,
        info: AccessibilityNodeInfoCompat
    ) {
        super.onInitializeAccessibilityNodeInfo(host, info)
        info.className = Button::class.java.name
    }
})
```

```xml
<!-- Remove from focus order if not interactive -->
<View
    android:focusable="false"
    android:importantForAccessibility="no" />
```

### Managing Focus Programmatically

```kotlin
// Request focus
view.requestFocus()

// Move accessibility (TalkBack) focus to a view after a screen change.
// TYPE_VIEW_FOCUSED is *input* focus and does NOT move the screen-reader cursor —
// perform the ACTION_ACCESSIBILITY_FOCUS action instead:
ViewCompat.performAccessibilityAction(
    view,
    AccessibilityNodeInfoCompat.ACTION_ACCESSIBILITY_FOCUS,
    null
)

// For dialogs/fragments, move focus to new content
binding.dialogTitle.requestFocus()
```

---

## Text & Font Scaling

Users with low vision often use **large text** or **display size** settings. Your app must respect these.

### Use Scalable Text Units (SP)

```xml
<!-- ✅ GOOD: Uses SP — scales with system font size -->
<TextView
    android:textSize="16sp" />

<!-- ❌ BAD: Uses DP — doesn't scale with font settings -->
<TextView
    android:textSize="16dp" />
```

### Handle Large Text Gracefully

```xml
<!-- Allow text to wrap instead of cutting off -->
<TextView
    android:maxLines="5"
    android:ellipsize="end"
    android:layout_width="0dp"
    android:layout_weight="1" />
```

### Non-Linear Font Scaling (Android 14+)

Starting with **Android 14 (API 34)**, the system uses **non-linear font scaling** up to 200%. This means text already at large sizes won't scale as aggressively, preventing excessively large text from breaking layouts.

```kotlin
// Convert SP to pixels correctly (respects non-linear scaling on API 34+)
val textSizePx = TypedValue.applyDimension(
    TypedValue.COMPLEX_UNIT_SP,
    16f,
    resources.displayMetrics
)

// ❌ BAD: Manual calculation ignores non-linear scaling on Android 14+
val badTextSizePx = 16f * resources.configuration.fontScale

// ✅ GOOD: Use TypedValue.applyDimension() which correctly handles non-linear scaling
```

> **Important:** On Android 14+, `Configuration.fontScale` may not directly map to the actual pixel scaling for all text sizes. Always use `TypedValue.applyDimension()` with `COMPLEX_UNIT_SP` for accurate conversion.

### Programmatic Font Scaling Detection

```kotlin
val fontScale = resources.configuration.fontScale
if (fontScale > 1.5f) {
    // Adjust layout for extra large text
}

// On Android 14+, check if non-linear scaling is in effect
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
    // fontScale can now go up to 2.0 (200%)
    // Use TypedValue.applyDimension() for accurate SP-to-PX conversion
}
```

---

## Touch Target Sizes

Small touch targets are problematic for users with motor disabilities and the elderly.

### Minimum Touch Target Size

Google recommends a minimum of **48dp × 48dp** for all interactive elements.

```xml
<!-- Ensure minimum touch target -->
<ImageButton
    android:layout_width="48dp"
    android:layout_height="48dp"
    android:padding="12dp"
    android:src="@drawable/ic_small_icon_24dp" />
```

### Using Touch Delegate for Small Views

```kotlin
// Expand touch area of a small view
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

Sufficient contrast ensures text is readable for users with low vision or color blindness.

### WCAG Contrast Requirements

| Text Type | Minimum Contrast (WCAG AA) | Enhanced Contrast (WCAG AAA) |
|-----------|---------------------------|------------------------------|
| Normal text (< 18sp, or bold < 14sp) | **4.5:1** | 7:1 |
| Large text (≥ 18sp, or bold ≥ 14sp) | **3:1** | 4.5:1 |
| UI components & graphics | **3:1** | — |

> **Note:** Android uses `sp` (scale-independent pixels) for text sizes, not CSS `pt`. At default settings 18sp ≈ 18pt, but sp scales with the user's font-size preference while pt does not.

### Checking Contrast

Use these tools to verify contrast ratios:
- **[WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)**
- **[Colour Contrast Analyser](https://www.tpgi.com/color-contrast-checker/)**
- **Android Studio Lint** (warns about low contrast)
- **Accessibility Scanner app** (on-device testing)

### Example: Safe Color Combinations

```xml
<!-- ✅ High contrast: White text on dark background -->
<TextView
    android:textColor="#FFFFFF"
    android:background="#1A1A2E" />

<!-- ✅ High contrast: Dark text on light background -->
<TextView
    android:textColor="#212121"
    android:background="#FAFAFA" />

<!-- ❌ Low contrast: Light gray on white -->
<TextView
    android:textColor="#CCCCCC"
    android:background="#FFFFFF" />
```

---

## Custom Views Accessibility

When creating custom views, you must manually provide accessibility information that standard widgets handle automatically.

### Using `AccessibilityDelegateCompat`

```kotlin
ViewCompat.setAccessibilityDelegate(customView, object : AccessibilityDelegateCompat() {
    override fun onInitializeAccessibilityNodeInfo(
        host: View,
        info: AccessibilityNodeInfoCompat
    ) {
        super.onInitializeAccessibilityNodeInfo(host, info)

        // Set the class name so TalkBack announces the correct type
        info.className = "android.widget.SeekBar"

        // Set range info for slider-like views (use the constructor; obtain() is deprecated)
        info.rangeInfo = AccessibilityNodeInfoCompat.RangeInfoCompat(
            AccessibilityNodeInfoCompat.RangeInfoCompat.RANGE_TYPE_FLOAT,
            0f,      // min
            100f,    // max
            currentValue  // current
        )

        // Add state description
        info.stateDescription = "Volume at ${currentValue.toInt()}%"

        // Mark as a heading (API 28+ equivalent)
        info.isHeading = true
    }

    override fun onInitializeAccessibilityEvent(host: View, event: AccessibilityEvent) {
        super.onInitializeAccessibilityEvent(host, event)
        event.className = "android.widget.SeekBar"
    }
})
```

### Using `ExploreByTouchHelper` for Virtual Views

For custom views that draw multiple interactive elements on a single Canvas:

```kotlin
class CustomCalendarView(context: Context) : View(context) {

    private val touchHelper = object : ExploreByTouchHelper(this) {

        override fun getVirtualViewAt(x: Float, y: Float): Int {
            // Return the ID of the virtual view at this position
            return findDayAt(x, y) ?: INVALID_ID
        }

        override fun getVisibleVirtualViews(virtualViewIds: MutableList<Int>) {
            // Add all virtual view IDs (e.g., day cells 1..31)
            for (i in 1..daysInMonth) {
                virtualViewIds.add(i)
            }
        }

        override fun onPopulateNodeForVirtualView(
            virtualViewId: Int,
            node: AccessibilityNodeInfoCompat
        ) {
            // Provide accessibility info for each virtual view
            node.text = "Day $virtualViewId"
            node.contentDescription = "Day $virtualViewId, ${monthName} $year"
            node.addAction(AccessibilityNodeInfoCompat.AccessibilityActionCompat.ACTION_CLICK)
            node.setBoundsInScreen(getDayBoundsInScreen(virtualViewId))
        }

        override fun onPerformActionForVirtualView(
            virtualViewId: Int,
            action: Int,
            arguments: Bundle?
        ): Boolean {
            if (action == AccessibilityNodeInfoCompat.ACTION_CLICK) {
                onDaySelected(virtualViewId)
                return true
            }
            return false
        }
    }

    init {
        ViewCompat.setAccessibilityDelegate(this, touchHelper)
    }
}
```

### `setScreenReaderFocusable` (API 28+)

```kotlin
// Make a container focusable for screen readers, reading all children together
// Similar to setting focusable=true but only for accessibility
ViewCompat.setScreenReaderFocusable(containerView, true)
```

```xml
<!-- XML equivalent (API 28+) -->
<LinearLayout
    android:screenReaderFocusable="true"
    android:focusable="false">
    <!-- Children will be read together -->
</LinearLayout>
```

---

## Accessibility Actions

Custom accessibility actions allow users to perform complex interactions that might otherwise require gestures.

### Adding Custom Actions

```kotlin
ViewCompat.setAccessibilityDelegate(itemView, object : AccessibilityDelegateCompat() {
    override fun onInitializeAccessibilityNodeInfo(
        host: View,
        info: AccessibilityNodeInfoCompat
    ) {
        super.onInitializeAccessibilityNodeInfo(host, info)

        // Add a custom "Delete" action
        val deleteAction = AccessibilityNodeInfoCompat.AccessibilityActionCompat(
            AccessibilityNodeInfoCompat.ACTION_CLICK,
            "Delete item"
        )
        info.addAction(deleteAction)

        // Add custom action with unique ID
        val archiveAction = AccessibilityNodeInfoCompat.AccessibilityActionCompat(
            R.id.accessibility_action_archive,
            "Archive item"
        )
        info.addAction(archiveAction)
    }

    override fun performAccessibilityAction(
        host: View,
        action: Int,
        args: Bundle?
    ): Boolean {
        when (action) {
            R.id.accessibility_action_archive -> {
                archiveItem()
                return true
            }
        }
        return super.performAccessibilityAction(host, action, args)
    }
})
```

### Replacing Swipe-to-Dismiss with Accessible Action

```kotlin
// For RecyclerView items with swipe-to-dismiss,
// add an accessible alternative via custom actions
ViewCompat.addAccessibilityAction(
    itemView,
    "Dismiss"
) { _, _ ->
    dismissItem(position)
    true
}
```

### Custom Actions in Jetpack Compose

```kotlin
Box(
    modifier = Modifier
        .semantics {
            customActions = listOf(
                CustomAccessibilityAction("Delete") {
                    onDelete()
                    true
                },
                CustomAccessibilityAction("Archive") {
                    onArchive()
                    true
                }
            )
        }
) {
    Text("Email from John Doe")
}
```

---

## Pane Titles & Screen Transitions

Pane titles help TalkBack announce screen or section changes, so users know what content is now visible.

### Setting Pane Title in XML

```xml
<FrameLayout
    android:accessibilityPaneTitle="Shopping Cart">
    <!-- Cart content -->
</FrameLayout>
```

### Setting Pane Title Programmatically

```kotlin
// When content changes significantly, set/update the pane title
ViewCompat.setAccessibilityPaneTitle(fragmentContainer, "Search Results")
```

### Pane Titles in Jetpack Compose

```kotlin
Box(
    modifier = Modifier.semantics {
        paneTitle = "Order Summary"
    }
) {
    // Screen content
}
```

### Window Transitions

```kotlin
// ⚠️ announceForAccessibility() is deprecated in Android 16 (API 36).
// Use ViewCompat.setAccessibilityPaneTitle() or Activity.setTitle() instead.
// Deprecated example (still works below API 36):
// window.decorView.announceForAccessibility("Navigated to Settings")

// Preferred approach: set pane title on the root view
ViewCompat.setAccessibilityPaneTitle(window.decorView, "Settings")

// For Fragment transitions, set pane title on the root view
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    ViewCompat.setAccessibilityPaneTitle(view, "Profile Settings")
}
```

---

## Haptic Feedback

Haptic feedback provides tactile responses that benefit users who rely on non-visual feedback.

### Basic Haptic Feedback

```kotlin
// Provide haptic feedback on important interactions
view.performHapticFeedback(HapticFeedbackConstants.CONFIRM)   // API 30+
view.performHapticFeedback(HapticFeedbackConstants.REJECT)    // API 30+
view.performHapticFeedback(HapticFeedbackConstants.LONG_PRESS)
view.performHapticFeedback(HapticFeedbackConstants.CONTEXT_CLICK) // API 23+
```

### Haptic Feedback in Jetpack Compose

```kotlin
val haptic = LocalHapticFeedback.current

Button(
    onClick = {
        haptic.performHapticFeedback(HapticFeedbackType.LongPress)
        onSubmit()
    }
) {
    Text("Submit")
}
```

### When to Use Haptic Feedback

- ✅ Confirming an action (form submitted, item deleted)
- ✅ Error/warning states (invalid input, boundary reached)
- ✅ Toggle state changes (switch on/off)
- ✅ Drag-and-drop interactions (pickup, drop)
- ❌ Don't overuse — excessive vibration becomes annoying and loses meaning

---

## Accessibility in Jetpack Compose

Jetpack Compose has first-class accessibility support through **Semantics**.

### Basic Semantics

```kotlin
// Content description for an Image
Image(
    painter = painterResource(id = R.drawable.profile),
    contentDescription = "User profile picture" // null for decorative
)

// Merging descendants for TalkBack
Box(
    modifier = Modifier.semantics(mergeDescendants = true) {}
) {
    Icon(Icons.Default.Star, contentDescription = null)
    Text("Favorite")
}
```

### Custom Semantics

```kotlin
// Custom role and state
Box(
    modifier = Modifier
        .semantics {
            role = Role.Button
            contentDescription = "Play music"
            stateDescription = if (isPlaying) "Playing" else "Paused"
            onClick(label = "Play") {
                onPlayClick()
                true
            }
        }
        .clickable { onPlayClick() }
) { ... }
```

### Progress Indicators

```kotlin
// Accessible progress bar
LinearProgressIndicator(
    progress = { downloadProgress },
    modifier = Modifier.semantics {
        contentDescription = "Downloading: ${(downloadProgress * 100).toInt()}%"
        progressBarRangeInfo = ProgressBarRangeInfo(
            current = downloadProgress,
            range = 0f..1f
        )
    }
)
```

### Handling Focus in Compose

```kotlin
val focusRequester = remember { FocusRequester() }

TextField(
    value = text,
    onValueChange = { text = it },
    modifier = Modifier
        .focusRequester(focusRequester)
        .semantics { contentDescription = "Search field" }
)

LaunchedEffect(Unit) {
    focusRequester.requestFocus()
}
```

### Live Region in Compose

```kotlin
Text(
    text = statusMessage,
    modifier = Modifier.semantics {
        liveRegion = LiveRegionMode.Polite
    }
)
```

### Heading Semantics

```kotlin
Text(
    text = "Section Title",
    style = MaterialTheme.typography.headlineMedium,
    modifier = Modifier.semantics { heading() }
)
```

---

## Testing Accessibility

### 1. Android Accessibility Scanner

Install from the Play Store. Scans your screen and reports:
- Missing content descriptions
- Small touch targets
- Low contrast text
- Missing labels on inputs

### 2. TalkBack Manual Testing

Navigate your entire app using only TalkBack:
- Swipe right to move to next element
- Swipe left to go back
- Double-tap to activate
- Two-finger swipe to scroll

### 3. Automated Testing with Espresso

```kotlin
@Test
fun checkContentDescriptions() {
    onView(withId(R.id.submitButton))
        .check(matches(withContentDescription("Submit the registration form")))
}

@Test
fun checkImportantViewsAreAccessible() {
    // There is no built-in matcher for importantForAccessibility — use a custom one.
    val isImportantForA11y = object : TypeSafeMatcher<View>() {
        override fun describeTo(description: Description) {
            description.appendText("is important for accessibility")
        }
        override fun matchesSafely(view: View): Boolean =
            view.isImportantForAccessibility
    }

    onView(withId(R.id.profileImage))
        .check(matches(isImportantForA11y))
}
```

### 4. Compose Accessibility Testing

```kotlin
@Test
fun testAccessibility() {
    composeTestRule.setContent {
        MyScreen()
    }

    // Check that the button has the right content description
    composeTestRule
        .onNodeWithContentDescription("Submit form")
        .assertIsDisplayed()
        .assertHasClickAction()

    // Check heading semantics
    composeTestRule
        .onNodeWithText("Welcome")
        .assert(SemanticsMatcher.expectValue(SemanticsProperties.Heading, Unit))
}
```

### 5. Lint Checks

Android Lint includes accessibility checks. Run via:

```bash
./gradlew lint
```

Common lint rules:
- `ContentDescription` — Images missing descriptions
- `ClickableViewAccessibility` — Clickable views with no accessibility action
- `KeyboardInaccessibleWidget` — Widgets not reachable by keyboard
- `LabelFor` — Input fields missing associated labels

### 6. Simulate Color Blindness

On Android, you can simulate color blindness:
```
Settings → Accessibility → Color and motion → Color correction
```
Types available: Deuteranomaly, Protanomaly, Tritanomaly, Grayscale

---

## Best Practices Checklist

### Visual Accessibility ✅

- [ ] All images have meaningful `contentDescription` (or `null` if decorative)
- [ ] Text contrast ratio ≥ 4.5:1 for normal text, ≥ 3:1 for large text
- [ ] UI components contrast ratio ≥ 3:1
- [ ] Information is never conveyed by color alone
- [ ] App works correctly in grayscale mode
- [ ] Color-blind friendly color palette is used
- [ ] App supports Dark Mode with proper contrast

### Text & Scaling ✅

- [ ] All text sizes use `sp` units (not `dp`)
- [ ] Layouts adapt to 200% font size without content being cut off
- [ ] No text is rendered as images (use actual `TextView`)
- [ ] Text is not truncated unnecessarily
- [ ] Non-linear font scaling on Android 14+ is handled correctly (use `TypedValue.applyDimension()`)

### Touch & Interaction ✅

- [ ] All touch targets are ≥ 48dp × 48dp
- [ ] Sufficient spacing between interactive elements
- [ ] No actions require multi-finger gestures only (provide alternatives)
- [ ] Long-press alternatives are available via accessibility menu
- [ ] Swipe-to-dismiss has accessible action alternatives
- [ ] No time-dependent interactions without user control

### Screen Reader ✅

- [ ] All interactive elements have content descriptions
- [ ] Decorative views are marked as `importantForAccessibility="no"`
- [ ] Related elements are grouped logically
- [ ] Reading order is logical (top-to-bottom, left-to-right)
- [ ] Dynamic content changes are announced via `liveRegion`
- [ ] Custom views implement `AccessibilityNodeInfoCompat`
- [ ] Dialogs move focus correctly when opened
- [ ] Input fields have associated labels (`labelFor`)
- [ ] Pane titles are set for major screen sections
- [ ] Custom views with virtual sub-views use `ExploreByTouchHelper`

### Navigation ✅

- [ ] All features are accessible via keyboard/D-pad
- [ ] All features are accessible via Switch Access
- [ ] Focus order follows logical reading order
- [ ] Focus is managed correctly after navigation/screen changes
- [ ] No focus traps (except modal dialogs)
- [ ] Focus indicators are clearly visible on all interactive elements

### Testing ✅

- [ ] App tested end-to-end with TalkBack enabled
- [ ] App tested with Switch Access enabled
- [ ] Accessibility Scanner shows no critical issues
- [ ] Automated accessibility tests are written
- [ ] Tested in grayscale / color correction mode
- [ ] Tested with large font size (200%) and display size

---

## Additional Resources

| Resource | Link |
|----------|------|
| Android Accessibility Docs | https://developer.android.com/guide/topics/ui/accessibility |
| Jetpack Compose Accessibility | https://developer.android.com/jetpack/compose/accessibility |
| WCAG 2.2 Guidelines | https://www.w3.org/TR/WCAG22/ |
| WCAG 2.1 Guidelines | https://www.w3.org/TR/WCAG21/ |
| Material Design Accessibility | https://m3.material.io/foundations/accessible-design/overview |
| WebAIM Contrast Checker | https://webaim.org/resources/contrastchecker/ |
| Google Accessibility Scanner | https://play.google.com/store/apps/details?id=com.google.android.apps.accessibility.auditor |
| Color Blindness Simulator | https://www.color-blindness.com/coblis-color-blindness-simulator/ |
| ExploreByTouchHelper Reference | https://developer.android.com/reference/androidx/customview/widget/ExploreByTouchHelper |
| European Accessibility Act (EAA) | https://ec.europa.eu/social/main.jsp?catId=1202 |

---

*Last Updated: June 2026*  
*Covers Android SDK 24+ (with notes for API 28, 33, 34+) and Jetpack Compose (BOM 2026.06)*
