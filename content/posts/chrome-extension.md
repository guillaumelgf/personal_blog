+++
date = '2025-01-13T12:18:00+04:00'
draft = false
title = 'Web scrapping with a Chrome Extension'
summary = "How to perform web scrapping on a wep app's internal resources when it is not possible to authenticate programmatically ?"
+++

<a href="https://github.com/guillaumelgf/chrome_extension">
    <img src="/images/github.svg" style="float: left; height: 30px;margin:0;">
    <span style="vertical-align:bottom;margin-left: 7px;">Source Code</span>
</a>

# Introduction

**Why would I need a Chrome extension to perform web scrapping ?**

On its most common form, web scrapping is performed by directly fetching an endpoint (public or private) to retrieve Json data.

But there are some endpoints that are not meant to be used for this... For example my bank as an internal API to retrieve my monthly expenses
and then display them in the UI, but it wants to avoid at all cost external apps to be able to fetch these endpoints.

As I am not able to programmatically authenticate and then fetch this endpoint, I decided that my best option was to **manually authenticate**
and then **intercept the responses from the HTTP calls** performed by the app and then **send this responses to a endpoint of mine** for personal use.

A great way to do this is to code a Google Chrome Extension ! Let's do this :wink:

# Creating the Extension

Chrome as a [great documentation](https://developer.chrome.com/docs/extensions/get-started/tutorial/hello-world) that will get you to run your 
first extension very quickly.

## First try - Overriding `fetch`

So the idea is to override the `fetch` or `XMLHttpRequest.open` method (depending on what the target app is using) to send the response
to an owned endpoint before returning its value to the app as usual. 

Simple.

Place this code at the beginning of your js project and you will replace the `fetch` method with yours.

``` javascript
const originalFetch = window.fetch;

window.fetch = async (...args) => {
  const response = await originalFetch(args);
  console.log("Perform some stuff after request");
  return response;
}
```

Now let's inject it in a page using our Chrome Extension. The simplest manifest would look something like :

```json
{
  "manifest_version": 3,
  "name": "Fetch Override",
  "version": "1.0",
  "description": "Overriding fetch with a custom version of mine",
  "content_scripts": [
    {
      "js": ["scripts/override_fetch.js"],
      "matches": ["http://localhost/*"],
      "run_at": "document_start" // Mandatory. We want our code to execute first
    }
  ]
}
```

I'll let you check the awesome [Google documentation](https://developer.chrome.com/docs/extensions/get-started/tutorial/scripts-on-every-tab) for the different values I have put in the Json if needed.

**Problem** : It does not work...

Why ??

Well as put in [Google terms](https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts?hl=en#isolated_world) : 

> Content scripts live in an isolated world, allowing a content script to make changes to its JavaScript environment without conflicting with the page or other extensions' content scripts.

Basically we haven't overridden the `fetch` method of the target app, instead we have overridden the one in the isolated environment
of our Chrome extension.

We need to escape this isolated environment in some way.

## Second try -  Injecting code in the DOM

Now let's create a script that will inject some code directly into the HTML page using a `<script>` object instead :

```javascript
function injectScript(src) {
  const s = document.createElement("script");
  s.src = chrome.runtime.getURL(src);
  s.onload = () => s.remove(); // Optional. This covers our tracks by removing the object from HTML code after the code has been injected
  (document.head || document.documentElement).append(s);
  console.log("Script has been injected");
}

injectScript("scripts/override_fetch.js");
```

We can now update the manifest.json as such :

```json
{
  "manifest_version": 3,
  "name": "Fetch Override",
  "version": "1.0",
  "description": "Overriding fetch with a custom version of mine",
  "content_scripts": [
    {
      "js": ["scripts/inject_script.js"],
      "matches": ["http://localhost/*"],
      "run_at": "document_start"
    }
  ],
  "web_accessible_resources": [
    {
      "resources": ["scripts/override_fetch.js"],
      "matches": ["http://localhost/*"]
    }
  ]
}
```

Giving access to the `scripts/override_fetch.js` resource, this is now **working as expected** !!

# Conclusion 

A few takeaways from this project :
- Creating a Chrome Extension
- Executing js code on pages
- Injecting code into pages