# Developers guide

- [Prerequisites](#prerequisites)
- [Advice](#advice)
  - [DOM vs HAR advice](#dom-vs-har-advice)
    - [DOM advice](#dom-advice)
    - [HAR advice](#har-advice)
    - [The best of two worlds](#the-best-of-two-worlds)
- [Testing HTTP/2 vs HTTP/1](#testing-http2-vs-http1)
- [Test in your browser](#test-in-your-browser)
- [Add a test case](#add-a-test-case)
- [Test your changes against a web page](#test-your-changes-against-a-web-page)
- [Add a new category](#add-a-new-category)

## Prerequisites

You need:

- [Node.js](https://nodejs.org/en/)
- [npm](https://nodejs.org/en/)
- [Firefox](https://www.mozilla.org/en-US/firefox/new/)
- [Chrome](https://www.google.com/chrome/browser/desktop/)

When you got them installed you can clone the project (or rather first fork it and clone your fork).

```bash
git clone git@github.com:sitespeedio/coach.git
```

Inside your coach folder install the dependencies and run the tests to check that everything works:

```bash
cd coach
npm install
npm test
```

If the test works you are ready start!

## Advice

The coach helps you with web performance and gives you advice about things you can do better. The advice needs to follow the following structure:

```js
return {
  id: 'uniqueid',   // a unique id
  title: 'The title of the advice',
  description: 'More information of the advice',
  advice: 'Information of what the user should do to fix the problem',
  score: 100,   // a number between 0-100, 100 means the page don't need any advice
  weight: 5,    // a number between 0-10, how important is this advice in this category? 10 means super important
  offending: [], // an array of assets that made the score less than perfect
  tags: ['performance','html']
};
```

Does it look familiar? Yep it is almost the same structure as an YSlow rule :)


### DOM vs HAR advice

The coach analyzes a page in two steps: First it executes JavaScript in the browser to do checks that are a perfect fit for JavaScript: examine the rendering path, check if images are scaled in the browser and more.

Then the coach take the HAR file generated from the page and analyze that too. The HAR is good if you want the number of responses, response size and check cache headers.

In the last step the  coach merges the advice into one advice list and creates an overall score.

Isn't that cool? We got one more thing that we [intend to implement](https://github.com/sitespeedio/coach/issues/13)): the combination of the two: A HAR advice that takes input from a DOM. This is cool because the coach will then have the power to know it all.

#### DOM advice

Each DOM advice needs to be an [IIFE](https://en.wikipedia.org/wiki/Immediately-invoked_function_expression) and return an object that holds the data.

A simple example is the cssPrint advice that looks for a print stylesheet.

```js
(function(util) {
 'use strict';

 var offending = [];

 var links = document.getElementsByTagName('link');
 for (var i = 0, len = links.length; i < len; i += 1) {
   if (links[i].media === 'print') {
     offending.push(util.getAbsoluteURL(links[i].href));
   }
 }

 var score = offending.length * 10;

 return {
   id: 'cssPrint',
   title: 'Do not load print stylesheets, use @media type print instead',
   description: 'Loading a specific stylesheet for printing slows down the page, even though it is not used',
   advice: offending.length > 0 ? 'The page has ' + offending.length + ' print stylesheets. You should include that stylesheet using @media type print instead.':'',
   score: Math.max(0, 100 - score),
   weight: 1,
   offending: offending,
   tags: ['performance', 'css']
 };

})(util);
```


#### HAR advice

We use [PageXray](https://github.com/sitespeedio/pagexray) to convert the HAR file into a Page object that are easier to work with.

Each HAR advice needs to implement a processPage function. The function will be called once for each page.

Lets look at a simple advice that checks if the total page size is too large.

```js
'use strict';

let util = require('../util');

module.exports = {
  id: 'pageSize',
  title: 'Total page size shouldn\'t be too big',
  description: 'Avoid having pages that have transfer size over the wire of more than 2 MB (desktop) and 1 MB (mobile) because that is really big and will hurt performance. ',
  weight: 3,
  tags: ['performance', 'mobile'],
  processPage: function(page, domAdvice, options) {
    // in options we have the input parameters
    // so we can do specific advice for devices like
    let sizeLimit = options.mobile ? 1000000 : 2000000;
    if (page.transferSize > sizeLimit) {
      return {
        score: 0,
        offending: [],
        advice: 'The page total transfer size is ' + util.formatBytes(page.transferSize) + ', which is more than the coach limit of ' + util.formatBytes(sizeLimit) + '. That is really big and you should check what you can do to make it smaller.'
      };
    }

    // and the domAdvice is the data we got from running the DOM advice
    // we can get things like what assets are loaded inside of HEAD
    // knowing which JavaScript files are loaded synchronous
    // if (domAdvice.coachAdvice.results.info.head.jssync.length > 0)

    return {
      score: 100,
      offending: [],
      advice: ''
    };
  }
};
```

What's extra cool is that a HAR advice can both act on input (specific advice for device or browser) and on the result from the DOM advice running in the browser (=you can let the HAR advice know which assets are loaded inside of head etc).

#### The best of two worlds

As an extra feature, the HAR advice override the DOM advice if the advice has the same id. This means you can easily combine data from the two and still output one advice.

There's no advice that use that functionality today but be sure it will soon be.

It also means we can use information from the resource timing API v2 (where we can get response size) making the DOM advice even more powerful, but when you combine the HAR & DOM advice we can get all the values from the HAR (or some if we want that).

## Testing HTTP/2 vs HTTP/1

Both DOM and HAR advice have help methods that makes it easy to give different advice depending on the protocol.

### DOM

```js
if (util.isHTTP2()) {
  // special handling for H2 connections
}
```

### HAR

```js
if (util.isHTTP2(page)) {
  // special handling for H2 connections
}
```

## Test in your browser

You can and should test your advice in your browser. It's easy. Just copy/paste your advice into the browser console. If you use the [utility methods](https://github.com/sitespeedio/coach/blob/master/lib/dom/util.js)  you need to copy/paste that too inside your console before you test your advice.

## Add a test case

When you create a new advice you also need to create unit tests. We run the tests in both Firefox & Chrome.

We create a new unique HTML page for each rule (or two if you want to test different behavior). If you only have one page, name it the same as the advice + '.html' and it will be picked up.

A simple test run for the print CSS advice looks like this:

```js
it('We should find out if we load a print CSS', function() {
  return runner.run('cssPrint.js')
    .then((result) => {
      assert.strictEqual(result.offending.length, 1);
    });
});
```

Right now all these tests run in <https://github.com/sitespeedio/coach/blob/master/test/dom/performance/indexTest.js>.

Each test case runs against a specific HTML page located in [test/http-server](test/http-server). Create a suitable HTML page with the structure you want to test. Create the test case in [test/dom](test/dom) or [test/har](test/har) and run it with `npm test`.

## Test your changes against a web page

The coach uses Browsertime as runner for browsers. When you done a change, make sure to build a new version of the combined JavaScript and then test against a url.

```bash
npm run combine
bin/index.js https://www.sitespeed.io firefox
```

## Add a new category

When you browse the code you will see that the coach knows more things than performance.

We have accessibility best practice, performance, and info today. Do the coach need to know something else? Let us know and we can create that category (it's as easy as create a new folder) and we can start add new advice there.
