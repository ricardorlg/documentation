---
id: page_objects
title: Interacting With Web Pages
sidebar_position: 5
---

# Interacting with Web Pages

Serenity integrates smoothly with Selenium WebDriver and manages details such as setting up the driver and maintaining the driver instance. It also provides a number of enhancements to standard Selenium. 

## Page Objects

If you are working with WebDriver web tests, you will be familiar with the concept of Page Objects. Page Objects are a way of isolating the implementation details of a web page inside a class, exposing only business-focused methods related to that page. They are an excellent way of making your web tests more maintainable.

In Serenity, page objects are just ordinary classes that extend the `PageObject` class. Serenity automatically injects a WebDriver instance into the page object that you can access via the `getDriver()` method, although you rarely need to use the WebDriver directly. The Serenity `PageObject` class provides a number of convenient methods that make accessing and manipulating web elements much easier than with conventional WebDriver test scripts.

Here is a simple example:

```java
...
import net.serenitybdd.core.pages.WebElementFacade;
import net.thucydides.core.pages.PageObject;
...

@DefaultUrl("http://localhost:9000/somepage")
public class FindAJobPage extends PageObject {

    WebElementFacade keywords;
    WebElementFacade searchButton;

    public void look_for_jobs_with_keywords(String values) {
        typeInto(keywords, values);
        searchButton.click();
    }

    public List<String> getJobTabs() {
        return findAll("//div[@id='tabs']//a").stream()
            .map(WebElementFacade::getText)
            .collect(Collectors.toList());
    }
}
```

The `typeInto` method is a shorthand that simply clears a field and enters the specified text.
If you prefer a more fluent-API style, you can also do something like this:

```java
@DefaultUrl("http://localhost:9000/somepage")
public class FindAJobPage extends PageObject {
	WebElementFacade keywordsField;
	WebElementFacade searchButton;

	public FindAJobPage(WebDriver driver) {
	    super(driver);
	}

	public void look_for_jobs_with_keywords(String values) {
	    enter(values).into(keywordsField);
	    searchButton.click();
	}

	public List<String> getJobTabs() {
	     return findAll("//div[@id='tabs']//a").stream()
            .map(WebElementFacade::getText)
            .collect(Collectors.toList());
	}
}
```

You can use an even more fluent style of expressing the implementation steps by using methods like `find`, `findBy`  and `then`.

For example, you can use webdriver `By` finders with element name, id, css selector or xpath selector as follows:

```java
find(By.name("demo")).then(By.name("specialField")).getValue();

find(By.cssSelector(".foo")).getValue();

find(By.xpath("//th")).getValue();
```

The `findBy` method lets you pass the css or xpath selector directly to WebDriver. For example,

```java
findBy("#demo").then("#specialField").getValue(); //css selectors

findBy("//div[@id='dataTable']").getValue(); //xpath selector
```

## Using pages in a step library

When you need to use a page object in one of your steps, you just need to declare a variable of type PageObject in your step library,  e.g.

```java
FindAJobPage page;
```

If you want to make sure you are on the right page, you can use the `currentPageAt()` method. This will check the page class for any `@At` annotations present in the PageObject class and, if present, check that the current URL corresponds to the URL pattern specified in the annotation. For example, when you invoke it using `currentPageAt()+, the following Page Object will check that the current URL is precisely `http://www.apache.org`.

```java
@At("http://www.apache.org")
public class ApacheHomePage extends PageObject {
    ...
}
```

The `@At` annotation also supports wildcards and regular expressions. The following page object will match any Apache sub-domain:

```java
@At("http://.*.apache.org")
public class AnyApachePage extends PageObject {
    ...
}
```

More generally, however, you are more interested in what comes after the host name. You can use the special `#HOST` token to match any server name. So the following Page Object will match both `http://localhost:8080/app/action/login.form` and `http://staging.acme.com/app/action/login.form`. It will also ignore parameters, so `http://staging.acme.com/app/action/login.form?username=toto&password=oz` will work fine too.

```java
@At(urls={"#HOST/app/action/login.form"})
public class LoginPage extends PageObject {
   ...
}
```

## Opening the page

A page object is usually designed to work with a particular web page. When the `open()` method is invoked, the browser will be opened to the default URL for the page.

The `@DefaultUrl` annotation indicates the URL that this test should use when run in isolation (e.g. from within your IDE).
Generally, however, the host part of the default URL will be overridden by the `webdriver.base.url` property, as this allows you to set the base URL across the board for all of your tests,
and so makes it easier to run your tests on different environments by simply changing this property value.
For example, in the test class above, setting the `webdriver.base.url` to 'https://staging.mycompany.com' would result in the page being opened at the URL of 'https://staging.mycompany.com/somepage'.

You can also define named URLs that can be used to open the web page, optionally with parameters. For example, in the following code, we define a URL called 'open.issue', that accepts a single parameter:

```java
@DefaultUrl("http://jira.mycompany.org")
@NamedUrls(
  {
    @NamedUrl(name = "open.issue", url = "http://jira.mycompany.org/issues/{1}")
  }
)
public class JiraIssuePage extends PageObject {
    ...
}
```

You could then open this page to the http://jira.mycompany.org/issues/ISSUE-1 URL as shown here:

```java
page.open("open.issue", withParameters("ISSUE-1"));
```

You could also dispense entirely with the base URL in the named URL definition, and rely on the default values:

```java
@DefaultUrl("http://jira.mycompany.org")
@NamedUrls(
  {
    @NamedUrl(name = "open.issue", url = "/issues/{1}")
  }
)
public class JiraIssuePage extends PageObject {
    ...
}
```

And naturally you can define more than one definition:

```java
@NamedUrls(
  {
          @NamedUrl(name = "open.issue", url = "/issues/{1}"),
          @NamedUrl(name = "close.issue", url = "/issues/close/{1}")
  }
)
```

You should never try to implement the `open()` method yourself. In fact, it is final. If you need your page to do something upon loading, such as waiting for a dynamic element to appear, you can use the @WhenPageOpens annotation.
Methods in the PageObject with this annotation will be invoked (in an unspecified order) after the URL has been opened. In this example, the `open()` method will not return until
the `dataSection` web element is visible:

```java
@DefaultUrl("http://localhost:8080/client/list")
    public class ClientList extends PageObject {

     @FindBy(id="data-section");
     WebElementFacade dataSection;
     ...

     @WhenPageOpens
     public void waitUntilTitleAppears() {
         element(dataSection).waitUntilVisible();
     }
}
```

## Working with web elements

### Checking whether elements are visible

The `WebElementFacade` class contains convenient fluent API for dealing with web elements, providing some commonly-used extra features that are not provided out-of-the-box by the WebDriver API.
`WebElementFacades` are largely interchangeable with WebElements: you just declare a variable of type `WebElementFacade` instead of type `WebElement`. For example, you can check that an element is visible as shown here:

```java
public class FindAJobPage extends PageObject {

    WebElementFacade searchButton;

    public boolean searchButtonIsVisible() {
        return searchButton.isVisible();
    }
    ...
}
```

If the button is not present on the screen, the test will wait for a short period in case it appears due to some Ajax magic. If you don't want the test to do this, you can use the faster version:

```java
public boolean searchButtonIsVisibleNow() {
    return searchButton.isCurrentlyVisible();
}
```

You can turn this into an assert by using the `shouldBeVisible()` method instead:

```java
public void checkThatSearchButtonIsVisible() {
    searchButton.shouldBeVisible();
}
```

This method will throw an assertion error if the search button is not visible to the end user.

### Checking whether elements are enabled

You can also check whether an element is enabled or not:

```java
searchButton.isEnabled()
searchButton.shouldBeEnabled()
```

There are also equivalent negative methods:

```java
searchButton.shouldNotBeVisible();
searchButton.shouldNotBeCurrentlyVisible();
searchButton.shouldNotBeEnabled()
```

You can also check for elements that are present on the page but not visible, e.g:

```java
searchButton.isPresent();
searchButton.isNotPresent();
searchButton.shouldBePresent();
searchButton.shouldNotBePresent();
```

### Manipulating select lists

There are also helper methods available for drop-down lists. Suppose you have the following dropdown on your page:

```xml
<select id="color">
    <option value="red">Red</option>
    <option value="blue">Blue</option>
    <option value="green">Green</option>
</select>
```

You could write a page object to manipulate this dropdown as shown here:

```java
public class FindAJobPage extends PageObject {

	@FindBy(id="color")
	WebElementFacade colorDropdown;

	public selectDropdownValues() {
	    colorDropdown.selectByVisibleText("Blue");
	    assertThat(colorDropdown.getSelectedVisibleTextValue(), is("Blue"));

	    colorDropdown.selectByValue("blue");
	    assertThat(colorDropdown.getSelectedValue(), is("blue"));

	    colorDropdown.selectByIndex(2);
	    assertThat(colorDropdown.getSelectedValue(), is("green"));

	}
	...
}
```

### Determining focus

You can determine whether a given field has the focus as follows:

```java
firstName.hasFocus()
```

You can also wait for elements to appear, disappear, or become enabled or disabled:

```java
button.waitUntilEnabled()
button.waitUntilDisabled()
```

or

```java
field.waitUntilVisible()
button.waitUntilNotVisible()
```

### Using direct XPath and CSS selectors

Another way to access a web element is to use an XPath or CSS expression. You can use the `$()` method with an XPath expression to do this more simply. For example, imagine your web application needs to click on a list item containing a given post code. One way would be as shown here:

```java
WebElement selectedSuburb = getDriver().findElement(By.xpath("//li/a[contains(.,'" ` postcode ` "')]"));
selectedSuburb.click();
```

However, a simpler option would be to do this:

```java
$("//li/a[contains(.,'" ` postcode ` "')]").click();
```

## Working with Asynchronous Pages

Asynchronous pages are those whose fields or data is not all displayed when the page is loaded. Sometimes, you need to wait for certain elements to appear, or to disappear, before being able to proceed with your tests. Serenity provides some handy methods in the PageObject base class to help with these scenarios. They are primarily designed to be used as part of your business methods in your page objects, though in the examples we will show them used as external calls on a PageObject instance for clarity.

### Checking whether an element is visible

In WebDriver terms, there is a distinction between when an element is present on the screen (i.e. in the HTML source code), and when it is rendered (i.e. visible to the user). You may also need to check whether an element is visible on the screen. You can do this in two ways. Your first option is to use the isElementVisible method, which returns a boolean value based on whether the element is rendered (visible to the user) or not:

```java
isElementVisible(By.xpath("//h2[.='A visible title']"))
```

Your second option is to actively assert that the element should be visible:

```java
shouldBeVisible(By.xpath("//h2[.='An invisible title']"));
```

If the element does not appear immediately, you can wait for it to appear:

```java
waitForRenderedElements(By.xpath("//h2[.='A title that is not immediately visible']"));
```

An alternative to the above syntax is to use the more fluid `waitFor` method which takes a css or xpath selector as argument:

```java
waitFor("#popup"); //css selector

waitFor("//h2[.='A title that is not immediately visible']"); //xpath selector
```

If you just want to check if the element is present though not necessarily visible, you can use `waitForRenderedElementsToBePresent` :

```java
waitForRenderedElementsToBePresent(By.xpath("//h2[.='A title that is not immediately visible']"));
```
or its more expressive flavour, `waitForPresenceOf` which takes a css or xpath selector as argument.

```java
waitForPresenceOf("#popup"); //css

waitForPresenceOf("//h2[.='A title that is not immediately visible']"); //xpath
```

You can also wait for an element to disappear by using `waitForRenderedElementsToDisappear` or `waitForAbsenceOf` :

```java
waitForRenderedElementsToDisappear(By.xpath("//h2[.='A title that will soon disappear']"));

waitForAbsenceOf("#popup");

waitForAbsenceOf("//h2[.='A title that will soon disappear']");
```

For simplicity, you can also use the `waitForTextToAppear` and `waitForTextToDisappear` methods:

```java
waitForTextToDisappear("A visible bit of text");
```

If several possible texts may appear, you can use `waitForAnyTextToAppear` or `waitForAllTextToAppear+:

```java
waitForAnyTextToAppear("this might appear","or this", "or even this");
```

If you need to wait for one of several possible elements to appear, you can also use the `waitForAnyRenderedElementOf` method:

```java
waitForAnyRenderedElementOf(By.id("color"), By.id("taste"), By.id("sound"));
```

## Working with timeouts

Modern AJAX-based web applications add a great deal of complexity to web testing. The basic problem is, when you access a web element on a page, it may not be available yet. So you need to wait a bit. Indeed, many tests contain hard-coded pauses scattered through the code to cater for this sort of thing.

But hard-coded waits are evil. They slow down your test suite, and cause them to fail randomly if they are not long enough. Rather, you need to wait for a particular state or event. Selenium provides great support for this, and Serenity builds on this support to make it easier to use.

### Implicit Waits

The first way you can manage how WebDriver handles tardy fields is to use the  `webdriver.timeouts.implicitlywait` property. This determines how long, in milliseconds, WebDriver will wait if an element it tries to access is not present on the page. To quote the WebDriver documentation:

“An implicit wait is to tell WebDriver to poll the DOM for a certain amount of time when trying to find an element or elements if they are not immediately available.”

The default value in Serenity for this property is currently 2 seconds. This is different from standard WebDriver, where the default is zero.

Let’s look at an example. Suppose we have a PageObject with a field defined like this:

```java
@FindBy(id="slow-loader")
public WebElementFacade slowLoadingField;
```

This field takes a little while to load, so won’t be ready immediately on the page.

Now suppose we set the `webdriver.timeouts.implicitlywait` value to 5000, and that our test uses the slowLoadingField:

```java
boolean loadingFinished = slowLoadingField.isDisplayed()
```

When we access this field, two things can happen. If the field takes less than 5 seconds to load, all will be good. But if it takes more than 5 seconds, a NoSuchElementException (or something similar) will be thrown.

That this timeout also applies for lists. Suppose we have defined a field like this, which takes some time to dynamically load:

```java
@FindBy(css="#elements option")
public List<WebElementFacade> elementItems;
```

Now suppose we count the values of the element like this:

```java
int itemCount = elementItems.size()
```

The number of items returned will depend on the implicit wait value. If we set the `webdriver.timeouts.implicitlywait` value to a very small value, WebDriver may only load some of the values. But if we give the list enough time to load completely, we will get the full list.

The implicit wait value is set globally for each WebDriver instance, but you can override the value yourself. The simplest way to do this from within a Serenity PageObject is to use the setImplicitTimeout() method:

```java
setImplicitTimeout(5, SECONDS)
```

But remember this is a global configuration, so will also affect other page objects. So once you are done, you should always reset the implicit timeout to its previous value. Serenity gives you a handy method to do this:

```java
resetImplicitTimeout()
```

See http://docs.seleniumhq.org/docs/04_webdriver_advanced.jsp#implicit-waits[Selenium Documentation] for more details on how the WebDriver implicit waits work.

### Using custom locator factories

Internally, Selenium uses the concept of Locator Factories
Normally, Serenity uses `SmartElementLocatorFactory`, an extension of the WebDriver `AjaxElementLocatorFactory`, when instantiating page objects. Among other things, this helps ensure that web elements are available and usable before they are used, allows field-by-field timeouts, and avoids long unnecessary waits on web elements after a step has failed.

The `SmartElementLocatorFactory` uses the default implicit wait, or the `timeoutInSeconds` attribute of the `@FindBy` annotation if this value has been specified (see below), or the default implicit wait value specified by the `webdriver.timeouts.implicitlywait` property.

In rare cases, you may need to customise this behaviour. To do this, you can use the `serenity.locator.factory` property to use one of the following alternative locator factories:

  * `AjaxElementLocatorFactory`: A WebDriver locator factory more suitable for Ajax applications. According to the WebDriver docs, this locator factory will return _an element locator that will wait for the specified number of seconds for an element to appear, rather than failing instantly if it's not present. This works by polling the UI on a regular basis. The element returned will be present on the DOM, but may not actually be visible._

  * `DefaultElementLocatorFactory`: the default WebDriver locator factory

If you use the `AjaxElementLocatorFactory`, you can use the `webdriver.timeouts.implicitlywait` parameter is to specify the number of seconds to wait. If no value is specified, the default wait will be 5 seconds.

### Explicit Timeouts
You can also programatically wait until an element is in a particular state. This is more flexible and useful when you need to wait for extra time in a specific situation. For example, we could wait until a field becomes visible:

```java
slowLoadingField.waitUntilVisible()
```

You can also wait for more arbitrary conditions, e.g.

```java
waitFor(ExpectedConditions.alertIsPresent())
```

The default time that Serenity will wait is determined by the `webdriver.wait.for.timeout` property. The default value for this property is 5 seconds.

Sometimes you want to give WebDriver some more time for a specific operation. From within a PageObject, you can override or extend the explicit timeout by using the withTimeoutOf() method. For example, you could wait for the #elements list to load for up to 5 seconds like this:

```java
withTimeoutOf(5, SECONDS).waitForPresenceOf(By.cssSelector("#elements option"))
```

You can also specify the timeout for a field. For example, if you wanted to wait for up to 5 seconds for a button to become clickable before clicking on it, you could do the following:

```java
someButton.withTimeoutOf(5, SECONDS).waitUntilClickable().click()
```

You can also use this approach to retrieve elements:

```java
elements = withTimeoutOf(5, SECONDS).findAll("#elements option")
```

Finally, if a specific element a PageObject needs to have a bit more time to load, you can use the timeoutInSeconds attribute in the Serenity @FindBy annotation, e.g.

```java
import net.serenitybdd.core.annotations.findby.FindBy;
...
@FindBy(name = "country", timeoutInSeconds="10")
public WebElementFacade country;
```

You can also wait for an element to be in a particular state, and then perform an action on the element. Here we wait for an element to be clickable before clicking on the element:

```java
addToCartButton.withTimeoutOf(5, SECONDS).waitUntilClickable().click()
```

Or, you can wait directly on a web element:

```java
@FindBy(id="share1-fb-like")
WebElementFacade facebookIcon;
  ...
public WebElementState facebookIcon() {
    return withTimeoutOf(5, TimeUnit.SECONDS).waitFor(facebookIcon);
}
```

Or even:

```java
List<WebElementFacade> currencies = withTimeoutOf(5, TimeUnit.SECONDS)
                              .waitFor(currencyTab)
                              .thenFindAll(".currency-code");
```


## Executing Javascript

There are times when you may find it useful to execute a little Javascript directly within the browser to get the job done. You can use the `evaluateJavascript()` method of the `PageObject` class to do this. For example, you might need to evaluate an expression and use the result in your tests. The following command will evaluate the document title and return it to the calling Java code:

```java
String result = (String) evaluateJavascript("return document.title");
```

Alternatively, you may just want to execute a Javascript command locally in the browser. In the following code, for example, we set the focus to the 'firstname' input field:

```java
	evaluateJavascript("document.getElementById('firstname').focus()");
```

And, if you are familiar with JQuery, you can also invoke JQuery expressions:

```java
	evaluateJavascript("$('#firstname').focus()");
```

This is often a useful strategy if you need to trigger events such as mouse-overs that are not currently supported by the WebDriver API.


## Uploading files

Uploading files is easy. Files to be uploaded can be either placed in a hard-coded location (bad) or stored on the classpath (better). Here is a simple example:

```java
public class NewCompanyPage extends PageObject {
    ...
    @FindBy(id="object_logo")
    WebElementFacade logoField;

    public NewCompanyPage(WebDriver driver) {
        super(driver);
    }

    public void loadLogoFrom(String filename) {
        upload(filename).to(logoField);
    }
}
```

## Using Fluent Matcher expressions

When writing acceptance tests, you often find yourself expressing expectations about individual domain objects or collections of domain objects. For example, if you are testing a multi-criteria search feature, you will want to know that the application finds the records you expected. You might be able to do this in a very precise manner (for example, knowing exactly what field values you expect), or you might want to make your tests more flexible by expressing the ranges of values that would be acceptable. Serenity provides a few features that make it easier to write acceptance tests for this sort of case.

In the rest of this section, we will study some examples based on tests for the Maven Central search site. This site lets you search the Maven repository for Maven artifacts, and view the details of a particular artifact.

We will use some imaginary regression tests for this site to illustrate how the Serenity matchers can be used to write more expressive tests. The first scenario we will consider is simply searching for an artifact by name, and making sure that only artifacts matching this name appear in the results list. We might express this acceptance criteria informally in the following way:

 * Give that the developer is on the search page,
 * And the developer searches for artifacts called 'Serenity'
 * Then the developer should see at least 16 Serenity artifacts, each with a unique artifact Id

In JUnit, a Serenity test for this scenario might look like the one:

```java
...
import static net.thucydides.core.matchers.BeanMatchers.the_count;
import static net.thucydides.core.matchers.BeanMatchers.each;
import static net.thucydides.core.matchers.BeanMatchers.the;
import static org.hamcrest.Matchers.greaterThanOrEqualTo;
import static org.hamcrest.Matchers.is;
import static org.hamcrest.Matchers.startsWith;

@RunWith(SerenityRunner.class)
public class WhenSearchingForArtifacts {

    @Managed
    WebDriver driver;

    @Steps
    public DeveloperSteps developer;

    @Test
    public void should_find_the_right_number_of_artifacts() {
        developer.opens_the_search_page();
        developer.searches_for("Serenity");
        developer.should_see_artifacts_where(the("GroupId", startsWith("net.thucydides")),
                                             each("ArtifactId").isDifferent(),
                                             the_count(is(greaterThanOrEqualTo(16))));

    }
}
```

Let's see how the test in this class is implemented. The `should_find_the_right_number_of_artifacts()` test could be expressed as follows:

 . When we open the search page

 . And we search for artifacts containing the word 'Serenity'

 . Then we should see a list of artifacts where each Group ID starts with "net.Serenity", each Artifact ID is unique, and that there are at least 16 such entries displayed.

The implementation of these steps is illustrated here:

```java
...
import static net.thucydides.core.matchers.BeanMatcherAsserts.shouldMatch;

public class DeveloperSteps {

    @Step
    public void opens_the_search_page() {
        onSearchPage().open();
    }

    @Step
    public void searches_for(String search_terms) {
        onSearchPage().enter_search_terms(search_terms);
        onSearchPage().starts_search();
    }

    @Step
    public void should_see_artifacts_where(BeanMatcher... matchers) {
        shouldMatch(onSearchResultsPage().getSearchResults(), matchers);
    }

    private SearchPage onSearchPage() {
        return getPages().get(SearchPage.class);
    }

    private SearchResultsPage onSearchResultsPage() {
        return getPages().get(SearchResultsPage.class);
    }
}
```

The first two steps are implemented by relatively simple methods. However the third step is more interesting. Let's look at it more closely:

```java
    @Step
    public void should_see_artifacts_where(BeanMatcher... matchers) {
        shouldMatch(onSearchResultsPage().getSearchResults(), matchers);
    }
```

Here, we are passing an arbitrary number of expressions into the method. These expressions actually 'matchers', instances of the BeanMatcher class. Not that you usually have to worry about that level of detail - you create these matcher expressions using a set of static methods provided in the BeanMatchers class. So you typically would pass fairly readable expressions like `the("GroupId", startsWith("net.Serenity"))` or `each("ArtifactId").isDifferent()+.

The `shouldMatch()` method from the BeanMatcherAsserts class takes either a single Java object, or a collection of Java objects, and checks that at least some of the objects match the constraints specified by the matchers. In the context of web testing, these objects are typically POJOs provided by the Page Object to represent the domain object or objects displayed on a screen.

There are a number of different matcher expressions to choose from. The most commonly used matcher just checks the value of a field in an object. For example, suppose you are using the domain object shown here:

```java
     public class Person {
        private final String firstName;
        private final String lastName;

        Person(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }

        public String getFirstName() {...}

        public String getLastName() {...}
    }
```

You could write a test to ensure that a list of Persons contained at least one person named "Bill" by using the "the" static method, as shown here:

```java
    List<Person> persons = Arrays.asList(new Person("Bill", "Oddie"), new Person("Tim", "Brooke-Taylor"));

    shouldMatch(persons, the("firstName", is("Bill"))
```

The second parameter in the the() method is a Hamcrest matcher, which gives you a great deal of flexibility with your expressions. For example, you could also write the following:

```java
    List<Person> persons = Arrays.asList(new Person("Bill", "Oddie"), new Person("Tim", "Brooke-Taylor"));

    shouldMatch(persons, the("firstName", is(not("Tim"))));
    shouldMatch(persons, the("firstName", startsWith("B")));
```

You can also pass in multiple conditions:

```java
    List<Person> persons = Arrays.asList(new Person("Bill", "Oddie"), new Person("Tim", "Brooke-Taylor"));

    shouldMatch(persons, the("firstName", is("Bill"), the("lastName", is("Oddie"));
```

Serenity also provides the DateMatchers class, which lets you apply Hamcrest matches to standard java Dates and `JodaTime` DateTimes. The following code samples illustrate how these might be used:

```java
    DateTime january1st2010 = new DateTime(2010,01,01,12,0).toDate();
    DateTime may31st2010 = new DateTime(2010,05,31,12,0).toDate();

    the("purchaseDate", isBefore(january1st2010))
    the("purchaseDate", isAfter(january1st2010))
    the("purchaseDate", isSameAs(january1st2010))
    the("purchaseDate", isBetween(january1st2010, may31st2010))
```

You sometimes also need to check constraints that apply to all of the elements under consideration. The simplest of these is to check that all of the field values for a particular field are unique. You can do this using the `each()` method:

```java
    shouldMatch(persons, each("lastName").isDifferent())
```

You can also check that the number of matching elements corresponds to what you are expecting. For example, to check that there is only one person named Bill, you could do this:

```java
     shouldMatch(persons, the("firstName", is("Bill"), the_count(is(1)));
```

You can also check the minimum and maximum values using the max() and min() methods. For example, if the Person class had a `getAge()` method, we could ensure that every person is over 21 and under 65 by doing the following:

```java
     shouldMatch(persons, min("age", greaterThanOrEqualTo(21)),
                          max("age", lessThanOrEqualTo(65)));
```

These methods work with normal Java objects, but also with Maps. So the following code will also work:

```java
    Map<String, String> person = new HashMap<String, String>();
    person.put("firstName", "Bill");
    person.put("lastName", "Oddie");

    List<Map<String,String>> persons = Arrays.asList(person);
    shouldMatch(persons, the("firstName", is("Bill"))
```

The other nice thing about this approach is that the matchers play nicely with the Serenity reports. So when you use the BeanMatcher class as a parameter in your test steps, the conditions expressed in the step will be displayed in the test report.

There are two common usage patterns when building Page Objects and steps that use this sort of matcher. The first is to write a Page Object method that returns the list of domain objects (for example, Persons) displayed on the table. For example, the getSearchResults() method used in the should_see_artifacts_where() step could be implemented as follows:

```java
    public List<Artifact> getSearchResults() {
        List<WebElement> rows = resultTable.findElements(By.xpath(".//tr[td]"));
        List<Artifact> artifacts = new ArrayList<Artifact>();
        for (WebElement row : rows) {
            List<WebElement> cells = row.findElements(By.tagName("td"));
            artifacts.add(new Artifact(cells.get(0).getText(),
                                       cells.get(1).getText(),
                                       cells.get(2).getText()));

        }
        return artifacts;
    }
```

The second is to access the HTML table contents directly, without explicitly modelling the data contained in the table. This approach is faster and more effective if you don't expect to reuse the domain object in other pages. We will see how to do this next.

### Working with HTML Tables

Since HTML tables are still widely used to represent sets of data on web applications, Serenity comes the HtmlTable class, which provides a number of useful methods that make it easier to write Page Objects that contain tables. For example, the rowsFrom method returns the contents of an HTML table as a list of Maps, where each map contains the cell values for a row indexed by the corresponding heading, as shown here:

```java
...
import static net.thucydides.core.pages.components.HtmlTable.rowsFrom;

public class SearchResultsPage extends PageObject {

    WebElement resultTable;

    public SearchResultsPage(WebDriver driver) {
        super(driver);
    }

    public List<Map<String, String>> getSearchResults() {
        return rowsFrom(resultTable);
    }

}
```

This saves a lot of typing - our `getSearchResults()` method now looks like this:

```java
    public List<Map<String, String>> getSearchResults() {
        return rowsFrom(resultTable);
    }
```

And since the Serenity matchers work with both Java objects and Maps, the matcher expressions will be very similar. The only difference is that the Maps returned are indexed by the text values contained in the table headings, rather than by java-friendly property names.

You can also read tables without headers (i.e., `<th>` elements) by specifying your own headings using the `withColumns` method. For example:

```java
    List<Map<Object, String>> tableRows =
                    HtmlTable.withColumns("First Name","Last Name", "Favorite Colour")
                             .readRowsFrom(page.table_with_no_headings);
```


You can also use the HtmlTable class to select particular rows within a table to work with. For example, another test scenario for the Maven Search page involves clicking on an artifact and displaying the details for that artifact. The test for this might look something like this:

```java
    @Test
    public void clicking_on_artifact_should_display_details_page() {
        developer.opens_the_search_page();
        developer.searches_for("Serenity");
        developer.open_artifact_where(the("ArtifactId", is("Serenity")),
                                      the("GroupId", is("net.Serenity")));

        developer.should_see_artifact_details_where(the("artifactId", is("Serenity")),
                                                    the("groupId", is("net.Serenity")));
    }
```

Now the open_artifact_where() method needs to click on a particular row in the table. This step looks like this:

```java
    @Step
    public void open_artifact_where(BeanMatcher... matchers) {
        onSearchResultsPage().clickOnFirstRowMatching(matchers);
    }
```

So we are effectively delegating to the Page Object, who does the real work. The corresponding Page Object method looks like this:

```java
import static net.thucydides.core.pages.components.HtmlTable.filterRows;
...
    public void clickOnFirstRowMatching(BeanMatcher... matchers) {
        List<WebElement> matchingRows = filterRows(resultTable, matchers);
        WebElement targetRow = matchingRows.get(0);
        WebElement detailsLink = targetRow.findElement(By.xpath(".//a[contains(@href,'artifactdetails')]"));
        detailsLink.click();
    }
```

The interesting part here is the first line of the method, where we use the filterRows() method. This method will return a list of WebElements that match the matchers you have passed in. This method makes it fairly easy to select the rows you are interested in for special treatment.

## Switching to another page

A method, switchToPage() is provided in PageObject class to make it convenient to return a new PageObject after navigation from within a method of a PageObject class. For example,

```java
@DefaultUrl("http://mail.acme.com/login.html")
public class EmailLoginPage extends PageObject {

    ...
    public void forgotPassword() {
        ...
        forgotPassword.click();
        ForgotPasswordPage forgotPasswordPage = this.switchToPage(ForgotPasswordPage.class);
        forgotPasswordPage.open();
        ...
    }
    ...
}
```

## WebElement collection loading strategies

Selenium lets you use the `@FindBy` and `@FindAll` annotations to load collections of web elements, as illustrated here:

```java
@FindBy(css='#colors a')
List<WebElement> options
```

If you are working with an asynchronous application, these lists may take time to load, so Selenium may give you an empty list because the elements have not loaded yet.

Serenity lets you find-tune this behaviour in two ways. The first is to use the wait DSL to load the elements directly, e.g.:

```java
withTimeoutOf(5, SECONDS).waitForPresenceOf(By.cssSelector("#colors a"))
```

Alternatively, you can use the `serenity.webdriver.collection_loading_strategy` property to define how Serenity loads collections of web elements when using the `@FindBy` and `@FindAll` annotations. There are three options:
 * Optimistic
 * Pessimistic (default)
 * Paranoid

Optimistic will only wait until the field is defined. This is the native Selenium behaviour.

Pessimistic will wait until at least the first element is displayed. This is currently the default.

Paranoid will wait until all of the elements are displayed. This can be slow for long lists.

## Working with fixture methods

When a UI step fails in a Serenity test, the WebDriver instance is disabled for the rest of the test. This avoids unnecessary waits as the test steps through the subsequent steps (which it needs to do to document the test steps). The exception to this rule is in the case of fixture methods, such as methods annotated with the `@After` annotation in JUnit or Cucumber.

In these methods, the WebDriver instance can be used normally. In addition to the known JUnit and Cucumber annotations, any annotation starting with the word `After` will be considered a fixture method.

For example, suppose you need to delete the customer accounts via the UI at the end of each test. You already have an `AdminSteps` step library with a `deleteAllCustomerAccounts()` method that performs this task. You could ensure that all of the customer accounts are deleted like this:

```java
@Steps
AdminSteps asAdministrator;

...

@After
public void deleteUserAccounts() {
    asAdministrator.deleteAllCustomerAccounts();
}
```

This makes it easy to perform teardown or cleanup operations that use the user interface, potentially reusing steps or tasks that are used elsewhere in the tests.

You can also override this behaviour at any point by calling the `reenableDrivers()` method, as shown here:

```java
Serenity.webdriver().reenableDrivers();
```

## Extending Serenity WebDriver Integration

Serenity offers a simple way to extend the default WebDriver capabilities and customise the driver creation and teardown activities. Simply implement the `BeforeAWebdriverScenario` and/or the `AfterAWebDriverScenario` interfaces (both are in the `net.serenitybdd.core.webdriver.enhancers` package). Serenity will execute `BeforeAWebdriverScenario` classes just before a driver instance is created, allowing you to add customised options to the driver capabilities. Any `AfterAWebDriverScenario` are executed at the end of a test, just before the driver is closed.

### The BeforeAWebdriverScenario interface

The `BeforeAWebdriverScenario` is used to enhance the `Capabilities` object that will be passed to the WebDriver instance when a new driver is created. The method call passes in the requested driver and the `TestOutcome` object, which contains information about the name and tags used for this test. It also passes in the `EnvironmentVariables`, which gives you access to the current environment configuration. An example of a simple `BeforeAWebdriverScenario` is shown below:

```java
public class MyCapabilityEnhancer implements BeforeAWebdriverScenario {

    @Override
    public DesiredCapabilities apply(EnvironmentVariables environmentVariables,
                                     SupportedWebDriver driver,
                                     TestOutcome testOutcome,
                                     MutableCapabilities capabilities) {
        capabilities.setCapability("name", testOutcome.getStoryTitle() + " - " + testOutcome.getTitle());
        return capabilities;
    }
}
```

### The AfterAWebdriverScenario interface

The `AfterAWebdriverScenario` is called at the end of a test, just before the driver is closed, and once the result of the test is known. The test result (and other details) can be obtained from the `TestOutcome` parameter. This allows any last manipulations or checks to be performed on the driver, before the end of the test. The following example checks the result of the test that has just finished, and adds a cookie with a value depending on the test outcome:

```java
public class MyTestResultUpdater implements AfterAWebdriverScenario {
    void apply(EnvironmentVariables environmentVariables,
               TestOutcome testOutcome,
               WebDriver driver) {
       if ((driver == null) || (!RemoteDriver.isARemoteDriver(driver))) {
           return;
       }

       Cookie cookie = new Cookie("testPassed",
                                   testOutcome.isFailure() || testOutcome.isError() || testOutcome.isCompromised() ? "false" : "true");
       driver.manage().addCookie(cookie);
    }
}
```

### Driver Enhancers

If you need to perform some action on the driver instance before each test, you can implement the `CustomDriverEnhancer` class. Note that this will be called immediately after the driver is created, and before any page is opened, so the number of actions is relatively limited.

### Configuring the extension packages

The last thing you need to do is to tell Serenity what package it needs to look for your extension classes. Add the package, or a parent package to your Serenity configuration using the `serenity.extension.packages`.

`serenity.extension.packages=com.acme.myserenityextensions`

You can find an example of how these classes are implemented in a real-world use case in the [serenity-browserstack](https://github.com/serenity-bdd/serenity-core/tree/master/serenity-browserstack/src/main/java/net/serenitybdd/browserstack) module on Github.


### Custom WebDriver implementations

You can add your own custom WebDriver provider by implementing the DriverSource interface. First, you need to set up the following system properties (e.g. in your `serenity.properties` file):

```
webdriver.driver = provided
webdriver.provided.type = mydriver
webdriver.provided.mydriver = com.acme.MyPhantomJSDriver
thucydides.driver.capabilities = mydriver
```

Your custom driver must implement the DriverSource interface, as shown here:

```java
public class MyPhantomJSDriver implements DriverSource {

    @Override
    public WebDriver newDriver() {
        try {
            DesiredCapabilities capabilities = DesiredCapabilities.phantomjs();
            // Add
            return new PhantomJSDriver(ResolvingPhantomJSDriverService.createDefaultService(), capabilities);
        }
        catch (IOException e) {
            throw new Error(e);
        }
    }

	@Override
    public boolean takesScreenshots() {
        return true;
    }
}
```

Note that if you use custom drivers you will be entirely responsible for configuring and instantiating the browser instance, and the driver-related Serenity configuration options will not apply. We generally recommend using custom drivers only for very exceptional circumstances, and using `BeforeAWebdriverScenario` classes for most custom configuration requirements.
