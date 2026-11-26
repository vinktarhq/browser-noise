# browser-noise

Error messages that browsers produce on their own, which rarely say anything about your code. One
JSON file, four lists, MIT.

The list is what [Vinktar](https://vinktar.com) uses to label likely noise in its SDKs and on its
server, published on its own because every error tracker's users need the same list and nobody
had written it down.

## What is in `patterns.json`

**`denyMessageSubstrings`**: messages that carry no information about your code. A match
anywhere in the message is enough.

| Message | Where it comes from |
|---|---|
| `ResizeObserver loop limit exceeded`, `ResizeObserver loop completed with undelivered notifications` | The browser telling you a layout settled late. Benign by the spec's own wording. |
| `Script error.` | A cross-origin script threw and the browser hid everything but these two words. There is no stack and no file. |
| `Non-Error promise rejection captured`, `Non-Error exception captured` | Something rejected with a plain object. The SDK wrapper named the fact; the object itself is in the payload if you want it. |

**`nonAppFrameSchemes`**: a stack whose every frame starts with one of these did not run your
code. Browser extensions inject scripts into every page and their crashes land in your tracker.

**`browserNoisePatterns`**: regular expressions written as `/…/` strings. Each matches one
known source of noise.

| Pattern | Source |
|---|---|
| `^Script error\.?$`, `^ResizeObserver loop` | The two above, anchored, for trackers that match whole messages |
| `^Cannot redefine property: googletag$` | Two ad scripts, or an ad blocker and a tag manager, racing on `window.googletag` |
| `^Can't find variable: gmo$` | A Safari extension's leftover global |
| `^undefined is not an object \(evaluating 'a\.[A-Z]'\)$` | Minified extension code reading a property of nothing |
| `^vv\(\)\.getRestrictions is not a function$` | A parental-control extension |
| `_AutofillCallbackHandler` | The iOS autofill bridge throwing inside a web view |
| `^Object Not Found Matching Id:\d+` | Microsoft Outlook's SafeLink crawler executing the page |
| `^Java exception was raised during method invocation$` | An Android web view's bridge |
| `Cannot redefine property: (?:ethereum\|solana)` | Two crypto-wallet extensions both claiming `window.ethereum` |
| `^Non-Error promise rejection captured with value: Object Not Found Matching Id` | The Outlook crawler again, through a promise |
| `^Blocked a frame with origin` | A cross-origin iframe access the browser refused, as designed |
| `^Loading (?:CSS )?chunk \d+ failed` | A deploy replaced the chunk a stale tab was about to fetch. Reload the page; nothing in the code is wrong. |

**`crawlerUserAgents`**: substrings of user-agent strings from crawlers and headless runners that
execute JavaScript. Their errors are real but nobody will fix them.

## Using it

**Sentry SDKs** take strings and regular expressions in `ignoreErrors`, and a user-agent check
goes in `beforeSend`:

```js
import noise from './patterns.json';

const toRegExp = (s) => new RegExp(s.slice(1, -1));

Sentry.init({
  dsn: '…',
  ignoreErrors: [
    ...noise.denyMessageSubstrings,
    ...noise.browserNoisePatterns.map(toRegExp),
  ],
  beforeSend(event) {
    const frames = event.exception?.values?.[0]?.stacktrace?.frames ?? [];
    if (frames.length && frames.every((f) => noise.nonAppFrameSchemes.some((s) => f.filename?.startsWith(s)))) return null;
    return event;
  },
});
```

The crawler list is for the server: drop the event when the request's `User-Agent` contains one
of the substrings.

**Vinktar** labels an error that matches any of the four lists as likely noise. It is kept and
listed below the real errors, so a wrong match can still be found. Each set can be switched to
auto-ignore per project, which drops it before it is stored or counted. The SDKs send these errors
by default and drop them locally only with `ignoreKnownNoise`.

## Adding a pattern

Open an issue with the exact message, a stack trace if there is one, and where it comes from.
A pattern is accepted when the error cannot be caused by application code. A pattern that could
also hide a real bug is refused, however common the noise.

## Licence

MIT.
