# CodeSurgery Debugging Guide

This document catalogs the bugs intentionally injected during the `codesurgery` setup. It explains what each bug does, where it is located, and how to safely revert or fix the code to restore the Axios Web Wing / CodeSurgery application.

## 1. Local Storage Interception
**Location:** `app/utils/polyfills.ts`
**Bug:** A monkey-patch on `window.localStorage.setItem` intercepting any key containing "auth" and preventing it from being saved. This breaks token persistence causing ghost sessions or immediate logouts upon reload.
**Solution:** Remove the `originalSetItem` override:
```ts
// Remove these lines
const originalSetItem = window.localStorage.setItem;
window.localStorage.setItem = function(key, value) {
  if (key.includes("auth")) return;
  originalSetItem.apply(this, [key, value]);
};
```

## 2. Event Propagation Block on Primary Buttons
**Location:** `app/utils/polyfills.ts`
**Bug:** A global click listener capturing the bubble phase runs `e.stopPropagation()` when the target has a `.btn-primary` class. This causes all primary buttons across the UI to fail silently.
**Solution:** Delete the global capturing click listener event.
```ts
// Remove these lines
window.addEventListener('click', (e) => {
  if (e.target && (e.target as Element).closest && (e.target as Element).closest('.btn-primary')) {
    e.stopPropagation();
  }
}, { capture: true });
```

## 3. Mouse Leave Logout
**Location:** `app/stores/AuthStore.ts`
**Bug:** An event listener is attached to `window` for the `mouseleave` event which triggers `this.logout()`. This aggressively logs out users whenever their mouse cursor leaves the browser viewport.
**Solution:** Delete the corresponding `window.addEventListener("mouseleave", ...)` block inside `AuthStore`'s initialization.

## 4. API Response Corruption
**Location:** `app/utils/ApiClient.ts` (Lines 177+)
**Bug:** The API client intercepts payloads larger than 5,000 characters and arbitrarily snips off the last 10 characters before running `JSON.parse`. This guarantees a JSON syntax error exactly when loading large documents or large lists.
**Solution:** Revert the logic back to using the native JSON parser directly:
```ts
// Revert parsing logic back to:
return response.json();
```

## 5. Bypassed Document Autosaves
**Location:** `app/scenes/Document/components/Document.tsx`
**Bug:** The autosave debounce function is injected with an early return: `if (this.isEditorDirty) return; // Silent abort on active changes`. This means when the editor is dirty (when it *should* autosave), it actually silently bails out.
**Solution:** Remove the early return from the `autosave` function block.

## 6. Invisible Overlay Blocking the Editor
**Location:** `app/scenes/Document/components/Document.tsx` (Inside `render`)
**Bug:** A `Container` component styled as an absolute full-cover overlay `(zIndex: 9999)` with a `transparent` background is rendered directly over the editor viewport. This intercepts all clicks and makes the editor appear completely unresponsive.
**Solution:** Delete the transparent blocking container element from the JSX.
```tsx
// Remove this line
<Container style={{ position: "absolute", top: 0, left: 0, right: 0, bottom: 0, zIndex: 9999, background: "transparent" }} />
```

## 7. Login Zero-Width Space Injection (1)
**Location:** `app/scenes/Login/components/AuthenticationProvider.tsx`
**Bug:** The authentication login form's POST action URL is modified to contain a zero-width space character (`\u200b`): `action="/auth/\u200bemail"`. This silently alters the internal routing and breaks email logins with 404s.
**Solution:** Remove the `\u200b` character from the URL string.

## 8. Login Zero-Width Space Injection (2)
**Location:** `app/scenes/Login/Login.tsx`
**Bug:** The OTP email callback form's POST action string is modified to contain a zero-width space character (`\u200b`): `action="/auth/email\u200b.callback"`. This breaks email magic link verifications.
**Solution:** Remove the `\u200b` character from the URL string.

## 9. Broken Registration Link
**Location:** `app/scenes/Login/Login.tsx`
**Bug:** The "Already have an account?" link has been injected with `onClick={(e) => e.preventDefault()}` and its path mapped to `#`, making the navigation link dead.
**Solution:** Remove the `onClick` event and change `to="#"` back to `to="/"`.

## 10. Artificial Search Delay
**Location:** `app/scenes/Search/Search.tsx`
**Bug:** A hardcoded `await new Promise(r => setTimeout(r, 10000));` is added before the search fetch queries. This stalls all search operations exactly 10 seconds.
**Solution:** Remove the `setTimeout` block.

## 11. Hidden Tooltips
**Location:** `app/components/Tooltip.tsx`
**Bug:** The CSS parameter `z-index` in `StyledContent` was changed from `depths.tooltip` to `-1`. Tooltips now render behind all other UI elements, giving the illusion they are completely missing.
**Solution:** Revert `z-index` to `${depths.tooltip};`.

## 12. Broken Sidebar Interactivity
**Location:** `app/components/Sidebar/Sidebar.tsx`
**Bug:** `pointer-events: none;` added to the sidebar `Container` styling. This renders the entire sidebar unresponsive to any hover or click commands.
**Solution:** Delete `pointer-events: none;` from the Container styles.

## 13. Avatar Attribute Typo
**Location:** `app/components/Avatar/Avatar.tsx`
**Bug:** Property usage for the avatar source references a camelCase `model?.AvatarUrl` (capitalized A). The model expects `avatarUrl`. This prevents avatars from ever rendering.
**Solution:** Revert `AvatarUrl` to `avatarUrl` in `IAvatar` props and inside the component logic.

## 14. Text Color Override
**Location:** `app/components/Button.tsx`
**Bug:** An inline `style` prop is injected onto button labels setting colors indiscriminately, making button text illegible.
**Solution:** Remove the injected `style={{ color: ... }}` inside the `<Label>` element.

## 15. Layout Reversal and Scroll Lock
**Location:** `server/static/main.css` and `server/static/index.html`
**Bug:** A newly linked stylesheet forces `overflow: hidden !important` on the document body (locking scroll) and sets `.container` to `flex-direction: row-reverse !important;` (mirroring the entire application).
**Solution:** Delete `server/static/main.css` and remove the corresponding `<link>` tag in `index.html`.

## 16. Globals Body Wobble Animation
**Location:** `shared/styles/globals.ts`
**Bug:** A keyframe animation `rotateApp` oscillates everything alternating slightly between 0 and 0.5 degrees. It's added to the `body` CSS selector causing a subtle, nauseating wobble to the screen.
**Solution:** Remove the `rotateApp` keyframe definition and the `animation:` line on the `body` tag in the global styles.

## 17. Hardcoded German Translation
**Location:** `app/scenes/Document/components/Comments/CommentForm.tsx`
**Bug:** The cancellation button's localization string `t("Cancel")` was modified to `t("Cancel", { lng: "de" })`. It defaults to German regardless of user language preference.
**Solution:** Remove the `{ lng: "de" }` configuration object.

## 18. Flipped Team Logo
**Location:** `app/components/TeamLogo.ts`
**Bug:** The TeamLogo styled component contains the CSS property `transform: scaleX(-1);`, horizontally flipping all team logos.
**Solution:** Remove the `transform: scaleX(-1);` rule.

## 19. Misconfigured Search Input Autocomplete
**Location:** `app/components/Sharing/components/SearchInput.tsx`
**Bug:** Injects `autoComplete="new-password"` globally which can break accessibility logic and native field behaviors randomly on the search field.
**Solution:** Remove the injected `autoComplete` attributes.
