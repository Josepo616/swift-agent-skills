---
description: Implement SwiftUI accessibility best practices
user_invocable: true
---

# Skill 11 — SwiftUI Accessibility

**Source:** Apple Human Interface Guidelines, WWDC Sessions  
**Applies to:** All views and interactive elements

---

## Labels for VoiceOver

```swift
// Icon-only buttons MUST have labels
Button { toggleFavorite() } label: {
    Image(systemName: "heart.fill")
}
.accessibilityLabel("Remove from favorites")
.accessibilityHint("Double tap to remove")

// Combine related elements
HStack {
    Image(systemName: "face.smiling")
    Text("Happy — 85% confident")
}
.accessibilityElement(children: .combine)

// Hide decorative elements
Image("decorative-divider")
    .accessibilityHidden(true)
```

## Dynamic Type (Scalable Text)

```swift
// GOOD: System text styles auto-scale
Text("How are you feeling?")
    .font(.title2)

// BAD: Hardcoded size does NOT scale
Text("How are you feeling?")
    .font(.system(size: 20))

// Custom fonts that scale
Text("Result")
    .font(.custom("MyFont", size: 24, relativeTo: .title))
```

## Color and Contrast

```swift
// Use semantic colors that adapt to settings
Text("Mood: Happy")
    .foregroundStyle(.primary)   // adapts to light/dark/high-contrast

// Never rely on color alone — add icon + text
Label("Error: Please enter text", systemImage: "exclamationmark.triangle")
    .foregroundStyle(.red)
```

## Minimum Touch Targets

- Apple requires **44x44 point minimum** for interactive elements
- Use `.frame(minWidth: 44, minHeight: 44)` or `.contentShape(Rectangle())` to expand tap areas

## Key Rules

- [ ] Every icon-only button gets an `accessibilityLabel`
- [ ] Use system text styles (`.title`, `.body`, `.caption`) — they auto-scale
- [ ] Never rely solely on color to convey state
- [ ] Hide decorative images with `accessibilityHidden(true)`
- [ ] Minimum 44x44pt tap targets for all interactive elements
- [ ] Test with VoiceOver on a real device

## Testing Checklist

- [ ] Enable VoiceOver and navigate the entire app by swiping
- [ ] Enable max Dynamic Type — verify layouts don't break
- [ ] Enable "Reduce Motion" — verify animations degrade gracefully
- [ ] Enable "Increase Contrast" — verify readability
- [ ] Use Xcode Accessibility Inspector for automated audits
