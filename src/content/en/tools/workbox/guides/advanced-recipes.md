project_path: /web/tools/workbox/_project.yaml
book_path: /web/tools/workbox/_book.yaml
description: Advanced recipes to use with Workbox.

{# wf_updated_on: 2019-02-15 #}
{# wf_published_on: 2017-12-17 #}
{# wf_blink_components: N/A #}

# Advanced Recipes {: .page-title }

## Offer a page reload for users

A common UX pattern for progressive web apps is to show a banner when a service
worker has updated and waiting to install.

To do this you'll need to add some code to your page and to your service worker.

**Add to your page**

```javascript
function showRefreshUI(registration) {
  // TODO: Display a toast or refresh UI.

  // This demo creates and injects a button.

  var button = document.createElement('button');
  button.style.position = 'absolute';
  button.style.bottom = '24px';
  button.style.left = '24px';
  button.textContent = 'This site has updated. Please click to see changes.';

  button.addEventListener('click', function() {
    if (!registration.waiting) {
      // Just to ensure registration.waiting is available before
      // calling postMessage()
      return;
    }

    button.disabled = true;

    registration.waiting.postMessage('skipWaiting');
  });

  document.body.appendChild(button);
};

function onNewServiceWorker(registration, callback) {
  if (registration.waiting) {
    // SW is waiting to activate. Can occur if multiple clients open and
    // one of the clients is refreshed.
    return callback();
  }

  function listenInstalledStateChange() {
    registration.installing.addEventListener('statechange', function(event) {
      if (event.target.state === 'installed') {
        // A new service worker is available, inform the user
        callback();
      }
    });
  };

  if (registration.installing) {
    return listenInstalledStateChange();
  }

  // We are currently controlled so a new SW may be found...
  // Add a listener in case a new SW is found,
  registration.addEventListener('updatefound', listenInstalledStateChange);
}

window.addEventListener('load', function() {
  navigator.serviceWorker.register('/sw.js')
  .then(function (registration) {
      // Track updates to the Service Worker.
    if (!navigator.serviceWorker.controller) {
      // The window client isn't currently controlled so it's a new service
      // worker that will activate immediately
      return;
    }

    // When the user asks to refresh the UI, we'll need to reload the window
    var preventDevToolsReloadLoop;
    navigator.serviceWorker.addEventListener('controllerchange', function(event) {
      // Ensure refresh is only called once.
      // This works around a bug in "force update on reload".
      if (preventDevToolsReloadLoop) return;
      preventDevToolsReloadLoop = true;
      console.log('Controller loaded');
      window.location.reload();
    });

    onNewServiceWorker(registration, function() {
      showRefreshUI(registration);
    });
  });
});
```

This code handles the various possible lifecycles of the service worker
and detects when a new service worker has become installed and is waiting to
activate.

When a waiting service worker is found we set up a 'controllerchange' listener
on `navigator.serviceWorker` so we know when to reload the window. When the
user clicks on the UI to refresh the page, we post a message to the new
service worker telling it to `skipWaiting` meaning it'll start to activate.

Note: This is one possible approach to refreshing the page on a new service
worker. For a more thorough answer as well as an explanation of alternative
approaches this
[article by Redfin Engineering](https://redfin.engineering/how-to-fix-the-refresh-button-when-using-service-workers-a8e27af6df68)
discuss a range of options.

**Add to your service worker**

```javascript
self.addEventListener('message', (event) => {
  if (!event.data){
    return;
  }

  switch (event.data) {
    case 'skipWaiting':
      self.skipWaiting();
      break;
    default:
      // NOOP
      break;
  }
});
```

This will receive a the 'skipWaiting' message and call `skipWaiting()`forcing
the service worker to activate immediately.

## "Warm" the runtime cache

After configuring some routes to manage caching of assets, you may want to
add some files to the cache during the service worker installation.

To do this you'll need to install your desired assets to the runtime cache.

```javascript
self.addEventListener('install', (event) => {
  const urls = [/* ... */];
  const cacheName = workbox.core.cacheNames.runtime;
  event.waitUntil(caches.open(cacheName).then((cache) => cache.addAll(urls)));
});
```

If you use strategies configured with a custom cache name you can do the same thing; just assign
your custom value to `cacheName`.

## Provide a fallback response to a route

There are scenarios where returning a fallback response is better than failing
to return a response at all. An example is returning a placeholder image when
the original image can't be retrieved.

All of the built-in caching strategies reject in a consistent manner when there's a network failure
and/or a cache miss. This promotes the pattern of [setting a global "catch"
handler](/web/tools/workbox/reference-docs/latest/workbox.routing#.setCatchHandler) to deal with any
failures in a single handler function:

```javascript
// Use an explicit cache-first strategy and a dedicated cache for images.
workbox.routing.registerRoute(
  new RegExp('/images/'),
  new workbox.strategies.CacheFirst({
    cacheName: 'images',
    plugins: [...],
  })
);

// Use a stale-while-revalidate strategy for all other requests.
workbox.routing.setDefaultHandler(
  new workbox.strategies.StaleWhileRevalidate()
);

// This "catch" handler is triggered when any of the other routes fail to
// generate a response.
workbox.routing.setCatchHandler(({event}) => {
  // The FALLBACK_URL entries must be added to the cache ahead of time, either via runtime
  // or precaching.
  // If they are precached, then call workbox.precaching.getCacheKeyForURL(FALLBACK_URL)
  // to get the correct cache key to pass in to caches.match().
  //
  // Use event, request, and url to figure out how to respond.
  // One approach would be to use request.destination, see
  // https://medium.com/dev-channel/service-worker-caching-strategies-based-on-request-types-57411dd7652c
  switch (event.request.destination) {
    case 'document':
      return caches.match(FALLBACK_HTML_URL);
    break;

    case 'image':
      return caches.match(FALLBACK_IMAGE_URL);
    break;

    case 'font':
      return caches.match(FALLBACK_FONT_URL);
    break;

    default:
      // If we don't have a fallback, just return an error response.
      return Response.error();
  }
});
```

## Make standalone requests using a strategy {: #make-requests }

Most developers will use one of Workbox's
[strategies](/web/tools/workbox/modules/workbox-strategies) as part of a
[router](/web/tools/workbox/modules/workbox-routing) configuration. This setup makes it easy to
automatically respond to specific `fetch` events with a response obtained from the strategy.

There are situations where you may want to use a strategy in your own router setup, or instead of
a plain `fetch()` request.

To help with these sort of use cases, you can use any of the Workbox strategies in a "standalone"
fashion via the `makeRequest()` method.

```javascript
// Inside your service worker code:
const strategy = new workbox.strategies.NetworkFirst({
  networkTimeoutSeconds: 10,
});
const response = await strategy.makeRequest({
  request: 'https://example.com/path/to/file',
});
// Do something with response.
```

The `request` parameter is required, and can either be a
[`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request) object or a string
representing a URL.

The `event` parameter is an optional
[`ExtendableEvent`](https://developer.mozilla.org/en-US/docs/Web/API/ExtendableEvent). If provided,
it will be used to keep the service worker alive (via
[`event.waitUntil()`](https://developer.mozilla.org/en-US/docs/Web/API/ExtendableEvent/waitUntil))
long enough to complete any "background" cache updates and cleanup.

`makeRequest()` returns a promise for a
[`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) object.

You can use it in a more complex example as follows:

```javascript
self.addEventListener('fetch', async (event) => {
  if (event.request.url.endsWith('/complexRequest')) {
    // Configure the strategy in advance.
    const strategy = new workbox.strategies.StaleWhileRevalidate({cacheName: 'api-cache'});

    // Make two requests using the strategy.
    // Because we're passing in event, event.waitUntil() will be called automatically.
    const firstPromise = strategy.makeRequest({event, request: 'https://example.com/api1'});
    const secondPromise = strategy.makeRequest({event, request: 'https://example.com/api2'});

    const [firstResponse, secondResponse] = await Promise.all(firstPromise, secondPromise);
    const [firstBody, secondBody] = await Promise.all(firstResponse.text(), secondResponse.text());

    // Assume that we just want to concatenate the first API response with the second to create the
    // final response HTML.
    const compositeResponse = new Response(firstBody + secondBody, {
      headers: {'content-type': 'text/html'},
    });

    event.respondWith(compositeResponse);
  }
});
```

## Serve cached audio and video {: #cached-av }

There are a few wrinkles in how some browsers request media assets (e.g., the `src` of a `<video>`
or `<audio>` element) that can lead to incorrect serving behavior unless you take specific steps
when configuring Workbox.

Full details are available in [this GitHub issue
discussion](https://github.com/GoogleChrome/workbox/issues/1663#issuecomment-448755945); a summary
of the important points is:

- Workbox must be told to respect [`Range` request
  headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests) by adding in the
  [`workbox-range-requests` plugin](/web/tools/workbox/modules/workbox-range-requests) to the
  strategy used as the handler.
- The audio or video element needs to opt-in to CORS mode using the [`crossOrigin`
  attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/CORS_settings_attributes), e.g. via
  `<video src="movie.mp4" crossOrigin="anonymous"></video>`.
- If you want to serve the media from the cache, you should explicitly add it to the cache ahead of
  time. This could happen either via precaching, or via calling
  [`cache.add()`](https://developer.mozilla.org/en-US/docs/Web/API/Cache/add) directly. Using a
  runtime caching strategy to add the media file to the cache implicitly is not likely to work,
  since at runtime, only partial content is fetched from the network via a `Range` request.

Putting this all together, here's an example of one approach to serving cached media content using
Workbox:

```html
<!-- In your page: -->
<!-- You currently need to set crossOrigin even for same-origin URLs! -->
<video src="movie.mp4" crossOrigin="anonymous"></video>
```

```javascript
// In your service worker:
// It's up to you to either precache or explicitly call cache.add('movie.mp4')
// to populate the cache.
//
// This route will go against the network if there isn't a cache match,
// but it won't populate the cache at runtime.
// If there is a cache match, then it will properly serve partial responses.
workbox.routing.registerRoute(
  /.*\.mp4/,
  new workbox.strategies.CacheFirst({
    cacheName: 'your-cache-name-here',
    plugins: [
      new workbox.cacheableResponse.Plugin({statuses: [200]}),
      new workbox.rangeRequests.Plugin(),
    ],
  }),
);
```
