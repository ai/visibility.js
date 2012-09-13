# Visibility.js – a wrapper for the Page Visibility API

Visibility.js allows you to determine whether your web page is visible to a
user, is hidden in background tab or is prerendering. It allows you to use the page
visibility state in JavaScript logic and improve browser performance by
disabling unnecessary timers and AJAX requests, or improve user interface
experience (for example, by stopping video playback or slideshow when user
switches to another browser tab).

Moreover, you can detect if the browser is just [prerendering] the page while
the user has not still opened the link, and don’t count this as a visit in your
analytics module, or do not run heavy calculations or other actions which will
disable the prerendering.

This library is a wrapper of the [Page Visibility API]. It eases usage of the
API by hiding vendor-specific property prefixes and adding some high-level
functions.

In most cases you don’t need to check whether the Page Visibility API is
actually supported in the browser as, if it does not, the library will just
assume that the page is visible all the time, and your logic will still work
correctly, albeit less effective in some cases.

Page Visibility API is natively supported by Google Chrome, Firefox and IE 10.
For others browsers you can use `lib/visibility.fallback.js` with focus/blur
hack (note that this hack have issue, that document become to be hidden,
when browser just lose focus, but still visible for user).

Sponsored by [Evil Martians].

[Page Visibility API]: http://www.w3.org/TR/2011/WD-page-visibility-20110602/
[prerendering]:        http://code.google.com/chrome/whitepapers/prerender.html
[MozVisibility]:       https://github.com/private-face/mozvisibility
[Evil Martians]:       http://evilmartians.com/

## Translations

Документация на русском: <http://habrahabr.ru/blogs/javascript/125833/>

## States

Currently the Page Visibility API supports three visibility states:

* `visible`: user has opened the page and works within it.
* `hidden`: user has switched to another tab or minimized browser window.
* `prerender`: browser is just prerendering a page which may possibly be opened
   by the user to make the apparent loading time lesser.

## Timers

The main use case for this library is to enable some of the times only when
content is visible to the user, i.e. the ones animating a countdown animation.

`Visibility.every(interval, callback)` is similar to
`setInterval(callback, interval)`, but calls `callback` every `interval` ms only
if the page is visible. For example, let’s create a countdown timer:

```js
Visibility.every(1000, function () {
    updateCountdownAnimation();
});
```

You can provide an additional interval which will be used when the page
is hidden. In next example, a check for inbox updates will be run every 1 minute
for a visible page and every 5 minutes for a hidden one:

```js
var minute = 60 * 1000;
Visibility.every(minute, 5 * minute, function () {
    checkForEmail();
});
```

Note that the callback will also be executed on every `hidden`->`visible` state
change to update old contents.

A syntactic sugar for specifying time intervals is supported when
[jQuery Chrono plugin] is included before Visibility.js. It can be used like
this:

```js
Visibility.every('minute', '5 minutes', function () {
    checkNewMails();
});
```

`Visibility.every` returns a timer identifier, much like the `setTimeout`
function. It cannot be passed to `clearInterval`, through, and you should use
`Visibility.stop(id)` to stop the timer.

```js
var slideshow = Visibility.every(5 * 1000, function () {
    nextSlide();
});

$('.stopSlideshow').click(function () {
    Visibility.stop(slideshow);
});
```

If the browser does not support the Page Visibility API, `Visibility.every` will
fall back to `setInterval`, and `callback` will be run every `interval` ms for
both the hidden and visible pages.

[jQuery Chrono plugin]: https://github.com/avk/jQuery-Chrono

## Initializers

In another common use case you need to execute some actions upon a switch to
particular visibility state.

### Waiting until the page becomes visible

`Visibility.onVisible(callback)` checks current state of the page. If it is
visible now, it will run `callback`, otherwise it will wait until state changes
to `visible`, and then run `callback`.

For example, let’s show an animated notification only when the page is visible,
so if an user opens a page in the background, the animation will delay until
the page becomes visible, i.e. until the user has switched to a tab with
the page:

```js
Visibility.onVisible(function () {
    Notification.animateNotice("Hello");
});
```

If a browser doesn’t support Page Visibility API, `Visibility.onVisible` will
run the `callback` immediately.

### Wait until the page is opened after prerendering

A web developer can hint a browser (using Prerendering API) that an user is
likely to click on some link (i.e. on a “Next” link in a multi-page article),
and the browser then may prefetch and prerender the page, so that the user will
not wait after actually going via the like.

But you may not want to count the browser prerendering a page as a visitor in
your analytics system. Moreover, the browser will disable prerendering if you
will try to do heavy computations or use audio/video tags on the page. So, you
may decide to not run parts of the code while prerendering and wait until the
user actually opens the link.

You can use `Visibility.afterPrerendering(callback)` in this cases. For example,
this code will only take real visitors (and not page prerenderings) into
account:

```js
Visibility.afterPrerendering(function () {
    Statistics.countVisitor();
});
```

If the browser doesn’t support Page Visibility API,
`Visibility.afterPrerendering` will run `callback` immediately.

## Low-level API

In some cases you may need more low-level methods. For example, you may want to
count the time user has viewed the page in foreground and time it has stayed in
background.

`Visibility.isSupported()` will return `true` if browser supports the
Page Visibility API:

```js
if( Visibility.isSupported() ) {
    Statistics.startTrackingVisibility();
}
```

`Visibility.state()` will return a string with visibility state. More states
can be added in the future, so for most cases a simpler `Visibility.hidden()`
method can be used. It will return `true` if the page is hidden by any reason.
For example, while prerendering, `Visibility.state()` will return `"prerender"`,
but `Visibility.hidden()` will return `true`.

This code will aid in collecting page visibility statistics:

```js
$(document).load(function () {

    if ( 'hidden' == Visibility.state() ) {
        Statistics.userOpenPageInBackgroundTab();
    }
    if ( 'prerender' == Visibility.state() ) {
        Statistics.pageIsPrerendering();
    }

});
```

And this example will only enable auto-playing when the page is opening as a
visible tab (not a background one):

```js
$(document).load(function () {

   if ( !Visibility.hidden() ) {
       VideoPlayer.play();
   }

});
```

Using `Visibility.change(callback)` you can listen to visibility state changing
events. The `callback` takes 2 arguments: an event object and a state name.

Let’s collect some statistics with this evented approach:

```js
Visibility.change(function (e, state) {
    Statistics.visibilityChange(state);
});
```

## Installing

### Packages

Visibility.js is shipped with 4 files:

* `visibility.core` – core module.
* `visibility.timers` – `every` and `stop` methods to set `setTimeout` depend
  on visibility state.
* `visibility` – `visibility.core` and `visibility.timers` together.
* `visibility.fallback` – fallback for browser without Page Visibility API.
  It use document `focus`/`blur` events, so document become to be hidden,
  when browser just lose focus, but still visible for user.

### Ruby on Rails

For Ruby on Rails you can use gem for Assets Pipeline.

1. Add `visibilityjs` gem to `Gemfile`:

   ```ruby
   gem "visibilityjs"
   ```

2. Install gems:

   ```sh
   bundle install
   ```

3. Include Visibility.js in your `application.js.coffee`:

   ```coffee
   #= require visibility
   ```

   If you willn’t use `every` method, you can reduce library size by including
   only core module:

   ```coffee
   #= require visibility.core
   ```

### CDN

If you don’t use any assets packaging manager use [CDNJS]. Add to your site:

```html
<script src="//cdnjs.cloudflare.com/ajax/libs/visibility.js/0.5/visibility.min.js"></script>
```

[CDNJS]: http://cdnjs.com/

### Other

If you need just a files, you can take already minified packages from
[github.com/ai/visibility.js/downloads].

[github.com/ai/visibility.js/downloads]: https://github.com/ai/visibility.js/downloads

## Contributing

1. To run tests you need node.js and npm. For example, in Ubuntu run:

   ```sh
   sudo apt-get install nodejs npm
   ```

2. Next install npm dependencies:

   ```sh
   npm install
   ```

3. Run test server:

   ```sh
   ./node_modules/.bin/cake test
   ```

4. Open tests in browser: [localhost:8000].
5. Also you can see real usage example in integration test
   `test/integration.html`.

[localhost:8000]: http://localhost:8000
