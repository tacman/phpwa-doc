# Resource Caching

Developers can build more resilient and user-friendly web applications that perform reliably under various network conditions. Also, it is possible to warm cache a selection of resources. This is powerfull as it allows applications to partially work offline.

The default strategy applied for resources is Network First i.e. the resource from the web server is fetched first. In case of failure, the cached data is served. By default, the service worker will wait 3 seconds before serving the cached version. This value can be configured.

<pre class="language-yaml" data-title="/config/packages/pwa.yaml" data-line-numbers><code class="lang-yaml"><strong>pwa:
</strong>    serviceworker:
        enabled: true
        src: "sw.js"
        workbox:
            resource_caches:
                - match_callback: 'startsWith: /pages/'
                  cache_name: 'static-pages'
                  strategy: 'NetworkFirst'
                  network_timeout: 2 # Wait only 2 seconds (only when strategy is networkFirst or NetworkOnly)
                  preload_urls: # List of URLs to preload. The URLs shall match the value in match_callback option
                      - 'page_tos'
                      - 'page_legal'
                - match_callback: 'regex: \/articles\/.*$'
                  cache_name: 'articles'
                  strategy: 'StaleWhileRevalidate'
                  broadcast: true # Broadcast changes only when strategy = staleWhileRevalidate
                  preload_urls: # List of URLs to precache. The URL shall be comprised within the regex
                      - 'app_articles'
                      - 'app_top_articles'
                        params:
                            display: 5
</code></pre>

{% hint style="info" %}
Please note that you can refer to any URLs, but only URLs served by your application will be cached.
{% endhint %}

### Match Callback

The `match_callback` option is designed to specify the condition used to determine which requests should be cached based on the request URL. This option can take different types of values, such as `startsWith` for simple prefix matching and `regex` for more sophisticated pattern matching. This flexibility allows you to tailor the Service Worker's caching logic to the specific requirements of your web application.

For instance, using `startsWith: /pages/` as the `match_callback` value means that any request URL that begins with `/pages/` will be considered a match and thus eligible for caching under the defined `cache_name`. Similarly, setting `match_callback` to `regex: \/articles\/.*$` employs a regular expression to match any URL that includes `/articles/`, followed by any characters until the end of the string, allowing for dynamic caching of all article pages.

This approach empowers developers to optimize their web application's performance by strategically caching resources based on URL patterns, ensuring that users enjoy faster load times and a smoother overall experience, even in offline scenarios or under suboptimal network conditions.

#### Provided Match Callback Handlers

| Pattern      | Description                                                                                                                                                                  | Example                                                                                    |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| navigate     | Matches all resources the user navigates to.                                                                                                                                 | No example. THis is an exact match                                                         |
| destination: | Matches a certain type of resource. Available values are listed on the [Request object documentation](https://developer.mozilla.org/en-US/docs/Web/API/Request/destination). | <ul><li>destination: audio</li><li>destination: style</li><li>destination: video</li></ul> |
| route:       | Matches the exact Symfony route. Shall not have required parameters                                                                                                          | <ul><li>route: app_homepage</li><li>route: app_princing</li></ul>                          |
| pathname:    | Matches an exact pathname                                                                                                                                                    | <ul><li>pathname: /foo/bar.docx</li><li>pathname: /report.pdf</li></ul>                    |
| origin:      | Matches all requests to the origin                                                                                                                                           | <ul><li>origin: example.com</li><li>origin: google.com</li></ul>                           |
| startsWith   | Matches all pathnames starting with the value                                                                                                                                | <ul><li>startsWith: /dashboard</li><li>startsWith: /admin</li></ul>                        |
| endsWith     | Matches all pathnames ending with the value                                                                                                                                  | <ul><li>endsWith: .css</li><li>endsWith: -report.pdf</li></ul><p></p>                      |
| regex        | Matches the regex                                                                                                                                                            | <ul><li>regex: \/image\/.*</li><li>regex: \.foo$</li></ul>                                 |

#### Custom Match Callback Handler

If needed, you can create your own Match Callback Handler. It must implement `SpomkyLabs\PwaBundle\MatchCallbackHandler\MatchCallbackHandler` and should return a value compatible with[ the `capture` argument of the Workbox method `registerRoute`](https://developer.chrome.com/docs/workbox/modules/workbox-routing#method-registerRoute).

### Cache Name Parameter

The `cache_name` parameter is used to customize the cache name. If not set, a default value is defined.

### Strategy Parameter

The `strategy` option corresponds to the Workbox strategy.

**CacheFirst (Cache Falling Back to Network)**

This strategy serves responses from the cache first, falling back to the network if the requested resource is not in the cache. It is ideal for assets that do not update frequently.

**NetworkFirst (Network Falling Back to Cache)**

The reverse of Cache First, this strategy tries to fetch the response from the network first. If the request fails (e.g., due to being offline), it falls back to the cache. Suitable for content where freshness is more critical.

**StaleWhileRevalidate**

This strategy serves cached responses immediately while silently fetching an updated version from the network in the background for future use. It is a good compromise for resources that need to be updated occasionally without compromising the user experience with loading delays.

**NetworkOnly**

A strategy that only uses the network, not attempting to cache or match responses from the cache. This is useful for non-cachable responses such as API requests returning user-specific data.

**CacheOnly**

Conversely, the Cache Only strategy serves responses from the cache, ignoring the network. This is ideal for offline applications where you want to ensure that only cached resources are served.

Leveraging Workbox strategies allows you to fine-tune how your application handles caching, ensuring that your users get the best possible experience whether they are online or offline.

### Network Timeout

The Workbox `network_timeout` option indicates the number of seconds before the request falls back to the strategy. It is only available with NetworkFirst and NetworkOnly.

### Max Entries / Max Age

When set, this indicates the expiration strategy used for the resource cache. max\_age can be in seconds or a human-readable string

`max_age: 3600` and `max_age: 1 hour` are similar.

`max_entries` is a positive number. When the cache reaches the number of cached resources, old resources are removed.

### Range Requests

When making a request, a `range` header can be set that tells the server to return only a portion of the full request. This is useful for certain files like a video file, where a user might change where to play the video.

By setting the `range_requests: true`, this type of requests will be supported.

### Broadcast Parameter

When accessing a page with the `StaleWhileRevalidate` strategy, the Service Worker will verify if a page update exists and will save it in the cache if any.

The broadcast parameter will tell the Service Worker to send a broadcast message in case of an update. This is usefull to warn the user it is reading an outdated version of the page and ask for a page reload.&#x20;

In the example below, the `message` is catched and, if it is of type "`workbox-broadcast-update`" and the URL matches with the current URL, a toast notification is displayed for 5 seconds.

```javascript
navigator.serviceWorker.addEventListener('message', async event => {
    if (event.data.meta === 'workbox-broadcast-update') {
        const {updatedURL} = event.data.payload;
        if (updatedURL === window.location.href) {
            const toast = document.getElementById('toast-refresh');
            toast.classList.remove('hidden');
            setTimeout(() => {
                toast.classList.add('hidden');
            }, 5000);
        }
    }
});
```

#### Broadcast Header

By default, the page header used to check if the page is outdated or not are `Content-Length`, `ETag`, `Last-Modified`. These headers can be changed.

{% code title="/config/packages/pwa.yaml" lineNumbers="true" %}
```yaml
pwa:
    serviceworker:
        workbox:
            page_caches:
                - ...
                  broadcast_headers:
                      - 'X-App-Cache'
```
{% endcode %}

### Cacheable Responses

By default, all responses with a `0` or `200` status code and that match with the `match_callback` option will be cached. It is possible to tell Workbox to cache only resources with a dedicated header or other status codes:

<pre class="language-yaml" data-title="/config/packages/pwa.yaml" data-line-numbers><code class="lang-yaml"><strong>pwa:
</strong>    serviceworker:
        enabled: true
        src: "sw.js"
        workbox:
            resource_caches:
                - match_callback: 'startsWith: /pages/'
                  cacheable_response_headers:
                    'X-Is-Cacheable': 'true'
                  cacheable_response_statuses: [200]
</code></pre>
