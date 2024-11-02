Developing a Chrome extension in 15 minutes
===========================================

![captionless image](https://miro.medium.com/v2/resize:fit:870/format:webp/1*Xx8Jy6htTBsI_D6l0WF2pQ.png)

While trying to clear my browser local storage again and again, I thought why not automate my browser to do it ? After all I am developer, and for every minute spent manually, I should spend hours trying to automate it !!

Just kidding, this will only take a moment to do.

**Chrome extensions** allow us to run scripts to do a plethora of things such as changing css, playing media files, setting browser themes and so much more.

My goal is to clear out the browser local storage via this extension. Say I have a my-key already present in storage as below,

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*CVn4kd-Kq5aGnXK9V5xOpQ.png)

and I need to delete this my-key using the extension.

Setup
-----

The no of files needed to develop an extension is surprisingly less. Let’s start with creating an empty folder called `my-extension`

After that, the first thing we would do is add a `manifest.json` file.

```json
{
 "name": "Clear Local Storage",
 "description": "Deletes local storage items",
 "version": "1.0",
 "manifest_version": 3
}
```

The manifest file controls the meta data of the project. All extensions need a manifest file.

Script
------

Next, add a `popup.js` script file which will delete the local storage items using existing browser side JavaScript APIs

```js
function clearTokens() {
 window.localStorage.removeItem('my-key');
}
```

Nothing special here, you can try out this same function in browser by bringing up the console (F12).

UI
--

Now let’s add the UI. To do that create a `popup.html` file.

```html
<!DOCTYPE html>
<html>
<head>
<link rel="stylesheet" href="button.css">
</head>
<body>
<button id="clear">Clear</button>
<script src="popup.js"></script>
</body>
</html>
```

We have added a button here along with a script tag which mentions the script we just wrote. Essentially on this button click, we need to execute the script. We have also mentioned a css file in case we need some styling.

**Modifying** `popup.js`

```js
let clearButton = document.getElementById("clear");
clearButton.addEventListener("click", async () => {
  let [tab] = await chrome.tabs.query({ active: true, currentWindow:      true });
  chrome.scripting.executeScript({
  target: { tabId: tab.id },
  function: clearTokens,
 });
});

function clearTokens() {
 window.localStorage.removeItem('my-key');
}
```

First we add a click even listener to the button, then, using the chrome API we find out the active tab of the user and then execute the script for this active tab only as we do not want to execute this extension on every tab.

Permissions
-----------

Most of the APIs need to be registered under the `"permissions"` field in the manifest for the extension to use them. We have used the `tabs` and `scripting` APIs.

```json
{
 "name": "Clear Local Storage",
 "description": "Deletes local storage items",
 "version": "1.0",
 "manifest_version": 3,
 "permissions": [  "activeTab",
  "scripting"
 ]
}
```

**Actions**
-----------

To define what happens when we click on the extension, we use `actions` field in manifest file. In our case we want to show the `popup.html` file on load.

```json
{
 "name": "Clear Local Storage",
 "description": "Deletes local storage items",
 "version": "1.0",
 "manifest_version": 3,
 "permissions": [  "activeTab",
  "scripting"
 ], "action": {
  "default_popup": "popup.html"
  }
}
```

Import
------

Now it’s time we test out extension !

Open the extensions tab i.e `chrome://extensions/` . Click on “**Load unpacked**”

![captionless image](https://miro.medium.com/v2/resize:fit:1296/format:webp/1*togkAF8IR3OBqPjAn36u-A.png)

Point to the folder which is `my-extension` in our case containing the 3 files we just wrote.

![captionless image](https://miro.medium.com/v2/resize:fit:366/format:webp/1*I9ZLQveul53R2bq7o-o6Lg.png)

The extension should now be loaded !

![captionless image](https://miro.medium.com/v2/resize:fit:1294/format:webp/1*odbwWXl3c6OWbfN397FYUQ.png)

Go to `google.com`, where we previously added `my-key` under local storage and click on extension button from the toolbar on top.

![captionless image](https://miro.medium.com/v2/resize:fit:342/format:webp/1*uVniAwH512TspZdc6eYCJw.png)

and voila, our local storage is clear !

![captionless image](https://miro.medium.com/v2/resize:fit:1084/format:webp/1*Nr_tAYqQ8G_eR1OBz9mAlQ.png)

That’s it. You’re now a magician. Thanks for reading !
