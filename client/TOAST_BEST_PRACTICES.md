# Toast Best Practices – React Native Expo App

This document describes how our toast system currently works in the app and the patterns you should follow when triggering toasts and navigating.

---

## 1. Toast Component (`Toast.tsx`)

### 1.1 Responsibilities

- Show a toast message with:
  - Enter + exit animations (Reanimated)
  - Optional action button (with a countdown circle)
  - Accessible labels and hints
- Call `onHide` **exactly once** after the hide animation finishes.

### 1.2 Props (current)

```ts
interface ToastProps {
  message: string;
  variant?: BadgeProps["variant"]; // Allow passing variant from Badge component
  onHide: () => void; // Function we call when animation finishes
  action?: { label: string; onPress: () => void };
  duration?: number;
  accessibilityLabel?: string;
  accessibilityHint?: string;
}
```

### 1.3 Double `onHide` protection

We guard `onHide` with a ref so it is never called twice:

```ts
const hasCalledOnHideRef = useRef(false);

const safeOnHide = useCallback(() => {
  if (hasCalledOnHideRef.current) {
    return;
  }
  hasCalledOnHideRef.current = true;
  onHide();
}, [onHide]);
```

### 1.4 Hide animation and cleanup

Hide is done via a dedicated function; cleanup only cancels animations and timers:

```ts
const startHideAnimation = useCallback(() => {
  translateY.value = withTiming(50, {
    duration: 300,
    easing: Easing.in(Easing.quad),
  });
  opacity.value = withTiming(0, { duration: 250 }, (isFinished) => {
    if (isFinished) {
      runOnJS(safeOnHide)();
    }
  });
}, [translateY, opacity, safeOnHide]);

useEffect(() => {
  translateY.value = withSpring(0, { damping: 15, stiffness: 120 });
  opacity.value = withTiming(1, {
    duration: 300,
    easing: Easing.out(Easing.quad),
  });

  if (action) {
    progress.value = withTiming(0, {
      duration,
      easing: Easing.linear,
    });
  }

  const timer = setTimeout(() => {
    startHideAnimation();
  }, duration);

  return () => {
    clearTimeout(timer);
    cancelAnimation(progress);
    cancelAnimation(translateY);
    cancelAnimation(opacity);
  };
}, [duration, action, progress, translateY, opacity, startHideAnimation]);
```

### 1.5 Action button behavior

Pressing the action triggers haptics, then runs the action and hides via animation:

```ts
const handleActionPress = () => {
  Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium).catch(() => {});
  action?.onPress();
  startHideAnimation();
};
```

**Guideline:** never call `onHide` directly from the action; always go through `startHideAnimation` so the exit animation runs.

```
┌─────────────────────────────────────────────────────────┐
│                     Screen Component                      │
│  ┌───────────────────────────────────────────────────┐  │
│  │      useProfileScreenEditor Hook                  │  │
│  │  ┌─────────────────────────────────────────────┐ │  │
│  │  │  State Management                           │ │  │
│  │  │  - value, setValue                          │ │  │
│  │  │  - hasChanges tracking                      │ │  │
│  │  │  - isSaving state                           │ │  │
│  │  └─────────────────────────────────────────────┘ │  │
│  │                                                   │  │
│  │  ┌─────────────────────────────────────────────┐ │  │
│  │  │  Save Logic                                 │ │  │
│  │  │  1. Call API (updateUserProfile)           │ │  │
│  │  │  2. Update Zustand store (setProfile)      │ │  │
│  │  │  3. Show Toast (toast())                   │ │  │
│  │  │  4. Delay 300ms                            │ │  │
│  │  │  5. Navigate (setIsNavigationAllowed)     │ │  │
│  │  └─────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                            │
                            │ toast() call
                            ▼
┌─────────────────────────────────────────────────────────┐
│                  useToastStore (Zustand)                 │
│  - Manages toast queue                                   │
│  - Handles coalescing (same key)                         │
│  - Enforces max visible toasts                           │
└─────────────────────────────────────────────────────────┘
                            │
                            │ state subscription
                            ▼
┌─────────────────────────────────────────────────────────┐
│                    ToastManager Component                 │
│  - Renders toasts from store                             │
│  - Positioned absolutely at bottom                        │
│  - Passes removeToast to each Toast                      │
└─────────────────────────────────────────────────────────┘
                            │
                            │ for each toast
                            ▼
┌─────────────────────────────────────────────────────────┐
│                      Toast Component                      │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Animation Lifecycle                              │  │
│  │  1. Enter (translateY: 50→0, opacity: 0→1)       │  │
│  │  2. Show (duration ms)                            │  │
│  │  3. Exit (translateY: 0→50, opacity: 1→0)        │  │
│  │  4. Call safeOnHide() ONCE                       │  │
│  └───────────────────────────────────────────────────┘  │
│                                                           │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Protection Against Double-Call                   │  │
│  │  - hasCalledOnHideRef prevents duplicate calls   │  │
│  │  - Cleanup DOES NOT call onHide                  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Common Pitfalls to Avoid

### ❌ DON'T:

1. Call `onHide()` in useEffect cleanup
2. Navigate immediately after showing toast
3. Allow save operations without loading state
4. Use exact same key for different toast types
5. Forget to handle network errors gracefully

### ✅ DO:

1. Use `safeOnHide` with ref guard
2. Delay navigation by 300-500ms after toast
3. Show loading indicators during async operations
4. Use descriptive keys for toast coalescing
5. Handle errors with appropriate user feedback
6. Test on both iOS and Android
7. Test with slow network conditions
8. Test rapid user interactions

---

## Migration Steps

1. **Backup current files**
2. **Update Toast component** with improved version
3. **Update useProfileScreenEditor** with delay and loading state
4. **Update all screens** using the hook to show loading state
5. **Test thoroughly** with all scenarios
6. **Monitor crash reports** for any edge cases
7. **Gather user feedback** on toast visibility

---

## Performance Considerations

- **Toast Queue**: Max 1 visible toast prevents UI clutter
- **Animation Performance**: Using `react-native-reanimated` for 60fps animations
- **Memory Leaks**: Properly canceling animations in cleanup
- **Re-renders**: Using `useCallback` and refs to minimize re-renders

---

## Accessibility

Current implementation already includes:

- ✅ `accessibilityRole='alert'` for screen readers
- ✅ Custom accessibility labels and hints
- ✅ Haptic feedback for actions
- ✅ Sufficient color contrast (mentioned in comments)

---

## Future Improvements

1. **Toast Queuing**: Show multiple toasts in sequence if needed
2. **Swipe to Dismiss**: Allow users to swipe toast away
3. **Persistent Toasts**: Flag for toasts that shouldn't auto-hide
4. **Rich Content**: Support for icons, images, or custom components
5. **Toast Analytics**: Track toast effectiveness and user interactions
6. **Undo Stack**: Implement proper undo/redo for destructive actions

---

## Conclusion

Historically, the main issue was **timing** – the toast was triggered, but the screen unmounted immediately before the entrance/exit animation could complete.

The key solution is to add a ~300ms delay between calling `toast()` and allowing navigation (via `setIsNavigationAllowed(true)` in `useProfileScreenEditor`).

Additionally, we protect against double-calls with `hasCalledOnHideRef` in the `Toast` component so `onHide` is only invoked once per toast.

With these patterns in place, users reliably see toasts before navigation occurs.
