# js-page-object

The [Page Object pattern](http://martinfowler.com/bliki/PageObject.html) is used in functional/acceptance testing suites which leverage a real browser, which is driven automatically through a tool such as Selenium WebDriver.

The pattern offers an abstraction between the tests themselves, and the underlying structure of the pages being tested. That is to say, an automated functional test should not know exactly how to communicate with a specific page in the system under test.

The result of applying such a pattern is that your automated functional tests are able to refer to the page via a proxy, the page object, and can be reused by any test which refers to the same page.

This means that the specific logic for how to access abnd manipulate elements on the pages are encapsulated solely in the page object, and will only ever need to be updated in a single spot should they change.

## Thanks, Alister!

[Alister Scott](https://github.com/alisterscott) has a great blog post series titled 'WebDriverJS & Mocha', which includes a post on using page objects:

- [Getting Started with WebDriverJS & Mocha](http://watirmelon.com/2015/10/28/getting-started-with-webdriverjs-mocha/)
- [WebDriverJS & Mocha Part 2: Hooks](http://watirmelon.com/2015/10/30/webdriverjs-mocha-part-2-hooks/)
- [WebDriverJS & Mocha Part 3: Page Objects](http://watirmelon.com/2015/10/30/webdriverjs-mocha-part-3-page-objects/)

I'd strongly encourage you to read his posts, and explore his [webdriver-js-demo](https://github.com/alisterscott/webdriver-js-demo) repository.

## Deprecation Notice

While I still love the idea behind the Page Object pattern, I'm becoming more skeptical of giving ownership of important layers of code to third-party libraries.

The obvious advantage to adopting libraries like what `js-page-object` was initially intended to be is to avoid duplication by taking advantage of domain-specific languages.

Unfortunately, the oft-ignored costs of coupling the destiny of your codebase to that of a third-party library is that you lose both control, but more importantly, visibility of the code you now depend on.

Since I've started making an effort to understand and carefully consider the obscurity that comes with each added layer of abstraction, I don't plan to ever get around to implementing the idea I've previously expressed below.

[George Ornbo wrote a very well-considered blog post](http://shapeshed.com/all-magic-comes-with-a-price/) on the topic of abstraction and obscurity in software development, which helps illustrate my point.

## Wrong

Previously, the only way to use Selenium WebDriver to open Google, search for something and ensure we are on the right page would be as follows:

``` javascript
var WebDriver = require('selenium-webdriver');
var Assert = require('assert');

var driver = new WebDriver.Builder().withCapabilities({'browserName': 'firefox'}).build();

driver.get('http://www.google.com');
driver.findElement(WebDriver.By.name('q')).sendKeys('webdriver');
driver.findElement(WebDriver.By.name('btnG')).click();

driver.getTitle().then(function(title) {
  Assert.equal('webdriver - Google Search', title);
});
```

The problem here is that we have mixed the concern of our test, which is to search for 'webdriver' and assert that the title is as expected, with the low-level mechanics of how we need to interact with the page, including the exact URL we need to visit and the exact name attribute of each element we need to touch.

Following this approach, these 'magic strings' will begin to spread throughout out test code, and cause a huge headache if and when anything needs to change. The combination of these conflicting responsibilities also leads to verbose, confusing tests which can confuse many people who need to read and modify them in the future.

## Right

Now, imagine a world in which we can [separate these concerns](http://en.wikipedia.org/wiki/Separation_of_concerns) and have a piece of the test code in which we define the structure and functions available in Google's home page which is then consumed by our automated functional test by simply passing through the search term and the expected title.

``` javascript
var PageObject = require('page-object');

var SearchPage = PageObject.extend({
  PageObject.url('http://www.google.com')

  var searchBox = PageObject.textbox('q');
  var searchButton = PageObject.button('btnG');
});

var ResultsPage = PageObject.extend({
  var title = PageObject.title();
});
```

Now that we're defined our page objects, we can leverage them in our test.

``` javascript
var WebDriver = require('selenium-webdriver');
var Assert = require('assert');

var driver = new WebDriver.Builder().withCapabilities({'browserName': 'firefox'}).build();

var searchPage = new SearchPage(driver).visit();
searchPage.searchBox = 'webdriver';
searchPage.searchButton().click();

var resultsPage = new ResultsPage(driver);

resultsPage.title().then(function(title) {
  Assert.equal('webdriver - Google Search', title);
});
```

And there you have it, an automated functional test which navigates to Google, searches for 'webdriver' and asserts that the title is as expected, all without having to know exactly how the two pages work.

## Under The Hood

Our `SearchPage` from the above example included three lines which leveraged the `PageObject` DSL to allow us to easily define the page elements.

Firstly, the `PageObject.url(PAGE_URL);` line allows us to define the URL which the Selenium WebDriver object will need to navigate to when a test calls the `visit()` method.

The next two lines in `SearchPage` specify the two elements, including what they are and how to access them, which triggers the `PageObject` DSL to define a few convenience methods for us to use later.

In the case of `var searchBox = PageObject.textbox('q');`, we can then call `searchPage.searchBox = 'webdriver';` and have `PageObject` call the underlying `sendKeys()` method to achieve our intended result.

The final line, `var searchButton = PageObject.button('btnG');`, gives us access to a `searchPage.searchButton()` method, which returns us the element from Selenium WebDriver which we can then call the `click()` method to submit the form.

## Custom Functions

As nice as the above example is, why can't we take it further? Our `SearchPage` is now much cleaner, but our example test still requires us to perform two tasks, typing in the search term and clicking the search button, in order to perform what is arguably a single goal.

What if we could combine the two in a custom function inside `SearchPage`?

``` javascript
var PageObject = require('page-object');

var SearchPage = PageObject.extend({
  PageObject.url('http://www.google.com')

  var searchBox = PageObject.textbox('q');
  var searchButton = PageObject.button('btnG');

  var searchFor = function(query) {
    this.searchBox = query;
    this.searchButton().click();
  };
}
});
```

Now that we've wrapped up the two calls in a nice little `searchFor()` function, we can call it like this:

``` javascript
var WebDriver = require('selenium-webdriver');
var Assert = require('assert');

var driver = new WebDriver.Builder().withCapabilities({'browserName': 'firefox'}).build();

var searchPage = new SearchPage(driver).visit();
searchPage.searchFor('webdriver');

resultsPage.title().then(function(title) {
  Assert.equal('webdriver - Google Search', title);
});
```

Not only does this custom function save us a line of code in the test itself, but it also helps to more clearly articulate the purpose and function of the page directly through the code.

## License

### The MIT License (MIT)

Copyright (c) 2013 James Ottaway

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
