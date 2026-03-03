# Android App Accessibility Guide

## Making Android Apps Accessible to All Users

Accessibility in Android development means designing apps that can be used by **everyone**, including people with visual impairments (blind or low vision), color blindness, motor disabilities, hearing impairments, and cognitive disabilities.

---

## Table of Contents

1. [Why Accessibility Matters](#why-accessibility-matters)
2. [Accessibility for Blind / Low Vision Users](#accessibility-for-blind--low-vision-users)
3. [Accessibility for Color Blind Users](#accessibility-for-color-blind-users)
4. [Content Descriptions](#content-descriptions)
5. [TalkBack Support](#talkback-support)
6. [Focus & Navigation](#focus--navigation)
7. [Text & Font Scaling](#text--font-scaling)
8. [Touch Target Sizes](#touch-target-sizes)
9. [Contrast Ratios](#contrast-ratios)
10. [Accessibility in Jetpack Compose](#accessibility-in-jetpack-compose)
11. [Testing Accessibility](#testing-accessibility)
12. [Best Practices Checklist](#best-practices-checklist)

---

## Why Accessibility Matters

- Over **1 billion people** worldwide live with some form of disability.
- **~8% of males** and **~0.5% of females** have some form of color blindness.
- Accessibility is required by law in many countries (ADA, WCAG, EN 301 549).
- Accessible apps reach a **wider audience** and improve **UX for all users**.
- Google Play rewards accessible apps with better discoverability.

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

---

## Accessibility for Color Blind Users

Color blindness affects the ability to distinguish between certain colors. Types include:

| Type | Description | Affected Colors |
|------|-------------|-----------------|
| **Deuteranopia** | Red-Green (most common) | Red & Green look similar |
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
<LinearLayout>
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
view.announceForAccessibility("Your order has been placed successfully!")
```

### Accessibility Events

```kotlin
// Send a custom accessibility event
val event = AccessibilityEvent.obtain(AccessibilityEvent.TYPE_ANNOUNCEMENT)
event.text.add("Form submitted successfully")
view.parent.requestSendAccessibilityEvent(view, event)
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

### Managing Focus Programmatically

```kotlin
// Request focus
view.requestFocus()

// Send accessibility focus to a view after screen change
view.sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_FOCUSED)

// For dialogs/fragments, move focus to new content
binding.dialogTitle.requestFocus()
```

### Focusable Elements

```xml
<!-- Make a custom view focusable -->
<View
    android:focusable="true"
    android:clickable="true"
    android:role="button" />

<!-- Remove from focus order if not interactive -->
<View
    android:focusable="false"
    android:importantForAccessibility="no" />
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

### Programmatic Font Scaling Detection

```kotlin
val fontScale = resources.configuration.fontScale
if (fontScale > 1.5f) {
    // Adjust layout for extra large text
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

| Text Type | Minimum Contrast | Enhanced Contrast |
|-----------|-----------------|-------------------|
| Normal text (< 18pt) | **4.5:1** | 7:1 |
| Large text (≥ 18pt or bold 14pt) | **3:1** | 4.5:1 |
| UI components & graphics | **3:1** | — |

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
    onView(withId(R.id.profileImage))
        .check(matches(isImportantForAccessibility()))
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

### Touch & Interaction ✅

- [ ] All touch targets are ≥ 48dp × 48dp
- [ ] Sufficient spacing between interactive elements
- [ ] No actions require multi-finger gestures only (provide alternatives)
- [ ] Long-press alternatives are available via accessibility menu

### Screen Reader ✅

- [ ] All interactive elements have content descriptions
- [ ] Decorative views are marked as `importantForAccessibility="no"`
- [ ] Related elements are grouped logically
- [ ] Reading order is logical (top-to-bottom, left-to-right)
- [ ] Dynamic content changes are announced via `liveRegion`
- [ ] Custom views implement `AccessibilityNodeInfoCompat`
- [ ] Dialogs move focus correctly when opened

### Navigation ✅

- [ ] All features are accessible via keyboard/D-pad
- [ ] Focus order follows logical reading order
- [ ] Focus is managed correctly after navigation/screen changes
- [ ] No focus traps (except modal dialogs)

### Testing ✅

- [ ] App tested end-to-end with TalkBack enabled
- [ ] Accessibility Scanner shows no critical issues
- [ ] Automated accessibility tests are written
- [ ] Tested in grayscale / color correction mode
- [ ] Tested with large font size and display size

---

## Additional Resources

| Resource | Link |
|----------|------|
| Android Accessibility Docs | https://developer.android.com/guide/topics/ui/accessibility |
| Jetpack Compose Accessibility | https://developer.android.com/jetpack/compose/accessibility |
| WCAG 2.1 Guidelines | https://www.w3.org/TR/WCAG21/ |
| Material Design Accessibility | https://m3.material.io/foundations/accessible-design/overview |
| WebAIM Contrast Checker | https://webaim.org/resources/contrastchecker/ |
| Google Accessibility Scanner | https://play.google.com/store/apps/details?id=com.google.android.apps.accessibility.auditor |
| Color Blindness Simulator | https://www.color-blindness.com/coblis-color-blindness-simulator/ |

---

*Last Updated: March 2026*  
*Covers Android SDK 21+ and Jetpack Compose 1.x+*

