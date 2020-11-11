+++
title = "Navigations"
description = ""
date = "2020-10-01"
category = [
    "Attack",
]
abuse = [
    "Downloads",
    "History",
    "CSP Violations",
    "Redirects",
    "window.open",
    "iframes",
]
defenses = [
    "Fetch Metadata",
    "SameSite Cookies",
    "COOP",
    "Framing Protections",
]
menu = "main"
weight = 2
+++

Detecting if a cross-site page triggered a navigation (or didn't) can be useful to an attacker. For example, a website may trigger a navigation in a certain endpoint [depending on the status of the user]({{< ref "#case-scenarios" >}}).

To detect if any kind of navigation occurred, an attacker can:

- Use an `iframe` and count the number of times the `onload` event is triggered.
- Check the value of `history.length`, accessible through any window reference. This gives the number of entries in the history of a victim either changed by `history.pushState` or regular navigations. To get the value of `history.length` an attacker changes the location of the window reference with the target website, changes back to same-origin, and finally reads the value.

## Download Trigger

When an endpoint sets the [`Content-Disposition: attachment`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition) header, it instructs the browser to download the response as an attachment instead of navigating to it. Detecting if this behavior occurred might allow attackers to leak private information if the outcome depends on the state of the victim's account.

### Download bar

In Chromium-based browsers when a file is downloaded, a preview of the download process appears in a bar at the bottom, integrated into the browser window. By monitoring the window height attackers could detect whether the "download bar" opened.


```javascript
// Read the current height of the window
var screenHeight = window.innerHeight;
// Load the page that may or may not trigger the download
window.open('https://example.org');
// Wait for the tab to load
setTimeout(() => {
    // If the download bar appears, the height of all tabs will be smaller
    if (window.innerHeight < screenHeight) {
      console.log('Download bar detected');
    } else {
      console.log('Download bar not detected');
    }
}, 2000);
```

{{< hint important >}}
This attack is only possible in Chromium-based browsers with automatic downloads enabled. The attack can't be also repeated since the user needs to close the download bar for it to be measurable again.
{{< /hint >}}

### Download Navigation (with iframes)

Another way to test for the [`Content-Disposition: attachment`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition) header is to check if a navigation occurred. If a page load causes a download, it will not trigger a navigation and the window will stay within the same origin.

The following snippet can be used to detect whether such a navigation has occurred and therefore detect a download attempt.

```javascript
// Set the destination URL to test for the download attempt
var url = 'https://example.org/';
// Create an outer iframe to measure onload event
var iframe = document.createElement('iframe');
document.body.appendChild(iframe);
// Create an inner iframe to test for the download attempt
iframe.srcdoc = `<iframe src="${url}" ></iframe>`;
iframe.onload = () => {
      try {
          // If a navigation occurs, the iframe will be cross-origin,
          // so accessing "inner.origin" will throw an exception
          iframe.contentWindow.frames[0].origin;
          console.log('Download attempt detected');
      } catch(e) {
          console.log('No download attempt detected');
      }
}
```

{{< hint info >}}
When there is no navigation inside an iframe caused by a download attempt, the iframe will not trigger an `onload` event directly. Because of that, in the example above, an outer iframe was used instead, that listens for `onload` event which triggers when subresources finished loading, including iframes.
{{< /hint >}}

{{< hint important >}}
This attack will work regardless of any [Framing Protections]({{< ref "xfo" >}}), because the `X-Frame-Options` and `Content-Security-Policy` headers are ignored if `Content-Disposition: attachment` is specified.
{{< /hint >}}

### Download Navigation (without iframes)

A variation of the technique presented in the previous section can also be effectively tested using `window` objects.

```javascript
// Set the destination URL
var url = 'https://example.org';
// Get a window reference
var win = window.open(url);

// Wait for the window to load.
setTimeout(() => {
      try {
          // If a navigation occurs, the iframe will be cross-origin,
          // so accessing "win.origin" will throw an exception
          win.origin;
          parent.console.log('Download attempt detected');
      } catch(e) {
          parent.console.log('No download attempt detected');
      }
}, 2000);
```

## Server-Side Redirects

### Inflation

A server-side redirect can be detected from a cross-origin page if the destination URL increases in size and contains an attacker controlled input (either in the form of a query string parameter or a path). The following technique relies on the fact that it is possible to induce an error in most web-servers by generating big requests parameters/paths. Since the redirect increases the size of the URL, it can be detected by sending exactly one character less than the server maximum capacity. That way if the size increases the server will respond with an error that can be detected from a cross-origin page (eg via Error Events).

{{< hint example >}}
An example of this attack can be seen [here](https://xsleaks.github.io/xsleaks/examples/redirect/).
{{< /hint >}}
## Cross-Origin Redirects

### CSP Violations

[Content-Security-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) (CSP) is an in-depth defense mechanism against XSS and data injection attacks. When a CSP is violated, a `SecurityPolicyViolationEvent` is thrown. An attacker can set up a CSP which will trigger a `Violation` event every time a `fetch` follows an URL not set in the CSP directive. This will allow an attacker to detect if a redirect to another origin occurred [^2] [^3].

The example below will trigger a `SecurityPolicyViolationEvent` if the website set in fetch API (line 6) redirects to a website different then `target.page`.

{{< highlight html "linenos=table,linenostart=1" >}}
<!-- Set the Content-Security-Policy to only allow example.org -->
<meta http-equiv="Content-Security-Policy"
      content="connect-src https://example.org">
<script>
// Listen for a CSP violation event
document.addEventListener('securitypolicyviolation', () => {
  console.log("Detected a redirect to somewhere other than example.org");
});
// Try to fetch example.org. If it redirects to anoter cross-site website
// it will trigger a CSP violation event
fetch('https://example.org/might_redirect', {
  mode: 'no-cors',
  credentials: 'include'
});
</script>
{{< / highlight >}}

## Case Scenarios

- An online bank decides to redirect wealthy users to unmissable stock opportunities by triggering a navigation to a reserved space on the website when users are consulting the account balance. If this is only done to a specific group of users, it becomes possible for an attacker to leak the "client status" of the user.


## Defense

| Attack Alternative  | [Same-Site Cookies]({{< ref "../defenses/opt-in/same-site-cookies.md" >}})  | [Fetch Metadata]({{< ref "../defenses/opt-in/fetch-metadata.md" >}})  | [COOP]({{< ref "../defenses/opt-in/coop.md" >}})  |  [Framing Protections]({{< ref "../defenses/opt-in/xfo.md" >}}) |
|:----------------------------------:|:--------------------------:|:---------------:|:-----:|:--------------------:|
| iframe                             |         ✔️                 |      ✔️          |  ❌   |          ✔️          |
| `history.length` (iframe)          |         ✔️                 |      ✔️          |  ❌   |          ✔️          |
| `history.length` (window.open)     |         ✔️ [(if Strict)]({{< ref "../defenses/opt-in/same-site-cookies.md#lax-vs-strict" >}})    |      ✔️          |  ✔️   |          ❌          |
| Download bar                       |         ✔️                 |      ✔️          |  ✔️   |          ✔️          |
| Download Navigation (w/ timeout)   |         ✔️ [(if Strict)]({{< ref "../defenses/opt-in/same-site-cookies.md#lax-vs-strict" >}})     |      ✔️          |  ❓   |          ✔️          |
| Download Navigation (no timeout)   |         ✔️                 |      ✔️          |  ✔️   |          ✔️          |
| CSP Violations                     |         ✔️                 |      ✔️          |  ❌   |          ❌          |

## Real-World Examples

- A vulnerability reported to Twitter used this technique to leak the contents of private tweets using [XS-Search]({{< ref "../attacks/xs-search.md" >}}). This attack was possible because the page would only trigger a navigation depending on whether there were results to the user query [^1].

## References

[^1]: Protected tweets exposure through the url, [link](https://hackerone.com/reports/491473)
[^2]: Disclose domain of redirect destination taking advantage of CSP, [link](https://bugs.chromium.org/p/chromium/issues/detail?id=313737)
[^3]: Using Content-Security-Policy for Evil, [link](http://homakov.blogspot.com/2014/01/using-content-security-policy-for-evil.html)
