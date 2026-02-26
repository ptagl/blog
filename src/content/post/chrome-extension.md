---
title: "How to develop a simple Chrome extension"
description: "How I built a tiny Chrome extension to add a missing button to Intervals.icu and pipe activity data straight to an AI assistant."
publishDate: 2026-02-26
tags: ["chrome-extension", "javascript"]
---

Among all the online services I use, [Intervals.icu](https://intervals.icu) is one of my favorites when it comes to **cycling**.
It's a free service where I track my training, analyze power and HR zones, and plan the workouts for the week.
Since **AI assistants** became actually useful, I started feeding them my workout data to get a second opinion on training load, recovery, and race readiness.

Unfortunately, there was no **one-click** way to **export** an activity. My workflow was quite slow and inefficient:
open the activity, copy the summary at the top, switch tab, copy the power zone breakdown, switch again, grab the HR zones, then paste everything into a chat window and hope the AI would make sense of the mess.

What I really wanted was a **single button** that would collect all that data as structured JSON and put it in my clipboard.

That button didn't exist, and given that the project is not open source, all I could do was to **build** it, or, to better say, "inject" it into the web page through a Chrome extension.

## The skeleton

The web is full of **starter templates** for Chrome extensions.
You don't need to write the boilerplate yourself: grab any "hello world" extension from GitHub or the [official docs](https://developer.chrome.com/docs/extensions/get-started/tutorial/hello-world), and you'll have a working folder structure in minutes.

The only file that actually matters for this use case is `content.js`, the **script** that runs inside the target page.

## Three steps to inject anything into websites

The whole logic fits in three steps, and they apply to basically any site you want to extend:

**1. Make sure you're on the right page.**
The `manifest.json` already restricts your script to run only on matching URLs (via the `matches` field in `content_scripts`).
But sometimes you need a finer check inside the script itself, for example, Intervals.icu activity URLs follow a pattern like `/activity/XXXXXXXX`, so a quick `window.location.pathname` check does the job.

**2. Find a DOM anchor.**
You need somewhere to **attach** your new element. Open DevTools, inspect the page, and find a stable **container** near where you want the button to appear. Try to find an element that is easy to **select** and won't change often.

**3. Add your logic.**
In my case: call the Intervals.icu **API** to fetch the full activity JSON, then copy it to the **clipboard**.

Here are the most important parts of the implementation:

1) Identify the current activity and fetch its JSON

```js
async function getActivityData() {
  // Activity id is the last segment of the current path
  // e.g. /activity/12345 -> "12345"
  const activityId = window.location.pathname.split("/").pop();

  // Intervals.icu endpoint for a single activity
  const apiUrl = `https://intervals.icu/api/activity/${activityId}`;

  const response = await fetch(apiUrl /* (credentials/cookies are handled by the browser session) */);
  if (!response.ok) throw new Error(`API request failed: ${response.status}`);

  return response.json();
}
```

2) Find a DOM anchor where we can place our button

```js
function getButtonContainer() {
  // Pick a stable container in the activity header area.
  // This is the "anchor" where we attach our UI.
  //
  // NOTE: selectors are site-specific and may break if the UI changes.
  const header = document.querySelector(".gutter8");
  if (!header) return null;

  // Walk down the DOM to the right section (site-specific).
  const rightSection = header.lastElementChild;
  const lastSection = rightSection?.lastElementChild;

  return lastSection ?? null;
}
```

3) Inject the button and wire up the click logic

```js
function addExportButton() {
  // Avoid duplicating the button if we re-run this function.
  if (document.getElementById(EXPORT_BUTTON_ID)) return;

  const container = getButtonContainer();
  if (!container) return;

  // Minimal wrapper to fit the existing layout (site-specific styling).
  const buttonGroup = document.createElement("div");
  buttonGroup.style = "float: right; padding-right: 12px;";

  // Reuse the site's button classes to blend in.
  const exportButton = document.createElement("a");
  exportButton.id = EXPORT_BUTTON_ID;
  exportButton.className = "btn btn-default";
  exportButton.href = "#";
  exportButton.textContent = "JSON to clipboard";

  exportButton.addEventListener("click", async (event) => {
    event.preventDefault();

    // Fetch JSON and copy to clipboard
    const activityJSON = await getActivityData();
    await navigator.clipboard.writeText(JSON.stringify(activityJSON));

    // Tiny UX feedback: swap label for 2 seconds
    const originalText = exportButton.textContent;
    exportButton.textContent = "Copied!";
    setTimeout(() => (exportButton.textContent = originalText), 2000);
  });

  buttonGroup.appendChild(exportButton);
  container.appendChild(buttonGroup);
}
```

4) Wait for the DOM to be ready and inject the button

```js
// Use a mutation observer to detect when the page is ready for the injection
const observer = new MutationObserver(() => {
  if (getButtonContainer()) addExportButton();
});

observer.observe(document.body, { childList: true, subtree: true });

// Run once as well (in case the DOM is already ready).
addExportButton();
```

The full source is available on [GitHub](https://github.com/ptagl/intervals.icu-json-to-clipboard).

## Testing without publishing

You don't need a developer account to try it out. Just:

- Collect all the extension **files** (in particular `manifest.json` and `content.js`) into a folder accessible to the browser

- Open chrome://extensions or the **extension page** of your chromium-based browser

- Enable **Developer mode**

Click `Load unpacked` and point it at your folder. It's done, the extension is **live** in your browser.

## Publishing on the Chrome Web Store

When you're ready to **share** it, the process is straightforward but has a few steps:

- Create a developer account at the Chrome Developer Dashboard

- Pay the one-time $5 **registration fee**

- **Upload** your ZIP, fill in the store listing (description, screenshots, icons, and other information)

## Submit for review

The **review** can take anywhere from a few hours to a few days. Google's official [publish guide](https://developer.chrome.com/docs/extensions/develop/migrate/publish-mv3) walks you through every field.
It's tedious but not complicated.

The whole thing took me a couple of evenings. The extension is tiny, a single `content.js` and a manifest file, but it **sped up** my workflow hugely.
If you use Intervals.icu and an AI assistant, give it a try.
And if you want to **inject** something into a different site, the pattern is exactly the same.
