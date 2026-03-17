# Accessibility Scale for Preview (e.g., WKWebView)

This document describes how to enable OS-level accessibility zoom for Box Preview when embedded in contexts where browser zoom does not apply (e.g., Apple's WKWebView in a native app). The `accessibilityScale` option applies CSS `zoom` to the preview content so that users can view documents at the magnification required by their accessibility needs.

## Overview

- **Option:** `accessibilityScale` (number, e.g. `1.5` for 150%)
- **Behavior:** CSS `zoom` is applied to the `.bp-content` container (content and in-content controls, not the header/toolbar). For the **Document (PDF) viewer**, the document is scaled only via the PDF.js internal scale (so it stays sharp); the **toolbar** and **thumbnails rail** are scaled separately via CSS zoom on their containers (`.bp-ControlsRoot` and `.bp-thumbnails-container`) so they match the accessibility zoom without affecting PDF quality.
- **Reflow:** Content does **not** reflow or rewrap. It scales uniformly (like browser pinch-to-zoom). Horizontal scrollbars appear when zoomed content exceeds the viewport.
- **Minimum value:** Scale values below 1 are clamped to 1 (magnification only).

## Initial Load — URL Parameter

When loading Preview from the Box web app (e.g., EndUserApp), pass the scale via a query parameter:

```
https://app.box.com/file/FILE_ID?accessibilityScale=1.5
```

The web app reads `accessibilityScale` from the URL and passes it through to the Preview SDK. No additional client code is required for initial load when using the URL.

## Initial Load — Preview SDK / Box UI Elements

When embedding Preview directly (e.g., via the Preview SDK or Box UI Elements ContentPreview), pass `accessibilityScale` in the options:

**Preview SDK (JavaScript):**

```javascript
var preview = new Box.Preview();
preview.show(fileId, accessToken, {
  container: '.preview-container',
  accessibilityScale: 1.5,
  // ...other options
});
```

**Box UI Elements (React):**

```jsx
<ContentPreview
  fileId={FILE_ID}
  token={TOKEN}
  accessibilityScale={1.5}
  // ...other props
/>
```

Any prop not explicitly listed on ContentPreview is passed through to the Preview library, so `accessibilityScale` is forwarded as-is.

## Runtime Scale Changes (No Page Reload)

There is no standard web API to detect OS-level zoom preferences. The scale must be passed explicitly (e.g., from native code). For **runtime** changes (e.g., user changes accessibility settings mid-session), use one of the following.

### Option 1: JavaScript injection (content-only zoom)

Inject CSS into the preview content container so only the content zooms (toolbar stays at 100%):

```javascript
// Example: WKWebView
webView.evaluateJavaScript(
  "document.querySelector('.bp-content').style.zoom = (scale)",
);
```

Replace `scale` with your desired value (e.g. `1.5`).

### Option 2: WKWebView page zoom (entire page)

Scale the entire page, including toolbar and chrome:

```javascript
webView.pageZoom = 1.5;
```

No Box changes or DOM targeting required.

### Option 3: Reload with new URL parameter

If you prefer a full reload with a new scale:

```
https://app.box.com/file/FILE_ID?accessibilityScale=2.0
```

You can preserve the current page using the hash fragment: `?accessibilityScale=2.0#p=5` (e.g. open at page 5).

## Scroll Position on Runtime Zoom Change

When you change zoom at runtime (e.g., via JS injection), the browser keeps scroll positions in pixels, but the content size changes, so the user may need to scroll slightly to re-center. For **proportional** scroll retention, the host app can save and restore scroll ratio:

1. Before changing zoom, read scroll position and content height (e.g. from `.bp-doc`: `scrollTop`, `scrollHeight`).
2. Apply the new zoom (e.g. set `.bp-content.style.zoom`).
3. Restore scroll so the ratio (e.g. `scrollTop / scrollHeight`) is preserved on the new `scrollHeight`.

Example pattern (pseudocode for native → JS):

```javascript
// 1. Get current scroll state
var doc = document.querySelector('.bp-doc');
var top = doc.scrollTop;
var height = doc.scrollHeight;

// 2. Change zoom
document.querySelector('.bp-content').style.zoom = newScale;

// 3. Restore proportional position
var ratio = top / height;
doc.scrollTop = ratio * doc.scrollHeight;
```

If using the URL parameter reload approach, use the `#p=N` fragment to open at a specific page; fine-grained offset within the page is not preserved across reload.

## Reflow and Horizontal Scroll

- **Text reflow:** None. CSS zoom scales content uniformly. Text does not rewrap; lines stay the same.
- **PDFs / images / presentations:** Rendered at fixed layout; they scale up. No reflow.
- **Horizontal scroll:** At zoom &gt; 100%, content can exceed viewport width. The viewer’s scroll container (e.g. `.bp-doc`) shows horizontal scrollbars as needed. This is expected and matches standard accessibility zoom behavior.

## Testing the Integration

1. **Preview SDK locally**

   - Build the Preview SDK and load a page that initializes Preview.
   - In the console: `preview.show(fileId, token, { accessibilityScale: 1.5 });`
   - Confirm document, image, and text previews scale and that scrolling, zoom controls, and text selection work.

2. **EndUserApp (URL parameter)**

   - Start the app and open: `http://localhost:8080/file/FILE_ID?accessibilityScale=1.5`
   - Verify the same behavior as above.

3. **Runtime injection**

   - With a preview already open, run in the console:  
     `document.querySelector('.bp-content').style.zoom = 2`
   - Confirm content zooms and scrollbars appear as expected.

4. **Unit tests**
   - In `box-content-preview`: `yarn test` (includes `accessibilityScale` parsing and `applyAccessibilityScale` tests).

## Does PDF.js support this scaling?

**Yes.** PDF.js does not need to do anything special. CSS `zoom` is a browser layout feature: the browser scales the entire element and its subtree (including the PDF canvas). The PDF viewer continues to render at its normal size; the browser then draws that content at the zoomed size. So it works with PDFs, images, text, and all other viewers.

## Troubleshooting: “Nothing happens”

### URL parameter (`?accessibilityScale=1.5`) has no effect

The Box web app (EndUserApp) loads the Preview library from a **CDN bundle** (by version). That bundle is a **released** build of box-content-preview. Your new `accessibilityScale` code only runs if the **loaded** Preview script includes it.

- **To test the URL parameter:** Use a build of box-content-preview that contains the accessibility scale changes. For example:
  1. Build box-content-preview locally (`yarn build` or the script your team uses to produce the preview bundle).
  2. Point EndUserApp (or box-ui-elements) at that build—e.g. via `yarn link`, a local path, or a pre-release version—so the app loads this build instead of the CDN.
- Until that build is what the app loads, the CDN script will ignore `accessibilityScale`, so the URL param will appear to do nothing.

### Console `document.querySelector('.bp-content').style.zoom = 1.5` has no effect

- **Run the script in the same document as the preview.** If the preview is inside an **iframe**, run the snippet in the iframe’s context: in DevTools, switch the console’s target to the iframe (e.g. “top” vs the iframe entry), then run the line again.
- **Confirm the element exists:** Run `document.querySelector('.bp-content')`. If it returns `null`, the preview may not be in this document or the viewer may not be mounted yet (open a file and wait for the preview to load).
- **Use a string if needed:** Some environments expect a string:  
  `document.querySelector('.bp-content').style.zoom = '1.5'`
- **Browser support:** CSS `zoom` is supported in Chrome, Safari, and Edge. Firefox supports it in 72+.

### Quick check that zoom works (no Box code)

To verify that CSS zoom works in your environment at all, open a preview, then in the **same frame** as the preview run:

```javascript
var el = document.querySelector('.bp-content');
if (el) {
  el.style.zoom = '1.5';
} else {
  console.log('No .bp-content in this document – try the iframe context');
}
```

If this makes the content get larger, the browser and context are fine; the remaining issue is making sure the **loaded** Preview script is a build that includes the `accessibilityScale` option and applies it on load.

## References

- WEBAPP-48249 (JIRA): Accessibility Scale for WKWebView Preview
- Preview option is parsed in `box-content-preview/src/lib/Preview.js` (`parseOptions()`).
- Zoom is applied in `box-content-preview/src/lib/viewers/BaseViewer.js` (`applyAccessibilityScale()`, called from `setup()`).
- EndUserApp passes the option from `?accessibilityScale=` in `src/components/preview/components/PreviewContent.tsx` (`getContentPreviewProps()`).
