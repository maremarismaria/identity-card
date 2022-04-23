---
layout: post
title: "Front-End Testing Summary"
date: 2022-04-22 10:00:00 +0000
categories: testing frontend
---

# 1. Testing Best Practices

> *Getting software to work is only half of the job.* (See Robert C. Martin)

## Introduction

### The Golden Rule

> "[...] when looking at test code it should feel as easy as modifying an HTML document and not like solving 2X(17 × 24) [...] allows the reader to get the grab instantly without spending even a single brain-CPU cycle [...]", @goldbergyoni

### The Test Anatomy

- **Test Naming: target, scenario and expectation**
    1. What is being tested?
    2. Under what circumstances and scenario?
    3. What is the expected result?

    {% highlight javascript %}
    // Unit under test
    describe("Transfer service", () => { // the name of the unit under test
      //Scenario
      describe("When no credit", () => { // additional level of categorization like the scenario or custom categories
        //Expectation
        test("Then the response status should decline", () => {});

        //Expectation
        test("Then it should send email to admin", () => {});
      });
    });
    {% endhighlight %}

- **Test structure: the Arrange, Act & Assert (AAA) pattern**
    1. *Arrange*: all the setup code to bring the system to the scenario the test aims to simulate
    2. *Act*: execute the unit under test
    3. *Assert*: ensure that the received value satisfies the expectation

    {% highlight javascript %}
    describe("Customer classifier", () => {
      test("When customer spent more than 500$, should be classified as premium", () => {
        //Arrange
        const customerToClassify = { spent: 505, joined: new Date(), id: 1 };
        const DBStub = sinon.stub(dataAccess, "getCustomer").reply({ id: 1, classification: "regular" });

        //Act
        const receivedClassification = customerClassifier.classifyCustomer(customerToClassify);

        //Assert
        expect(receivedClassification).toMatch("premium");
      });
    });
    {% endhighlight %}

- **Do behavioral/black-box testing instead of white-box testing**

    Whenever a public behavior is checked, the private implementation is also implicitly tested and your tests will break only if there is a certain problem (e.g. wrong output).

- **Don't "foo", use realistic input data**

    Use dedicated libraries like [Faker](https://www.npmjs.com/package/faker) to generate pseudo-real data that resembles the variety and form of production data.

- **Avoid global test fixtures and seeds, add data per-test**
- **If needed, use only short & inline snapshots**

    Otherwise, there are 1000 reasons for your test to fail - it’s enough for a single line to change for the snapshot to get invalid and this is likely to happen a lot.

    {% highlight javascript %}
    it("When visiting TestJavaScript.com home page, a menu is displayed", () => {
      //Arrange

      //Act
      const receivedPage = renderer
        .create(<DisplayPage page="http://www.testjavascript.com">Test JavaScript</DisplayPage>)
        .toJSON();

      //Assert
      const menu = receivedPage.content.menu;
      expect(menu).toMatchInlineSnapshot(`
    		<ul>
    			<li>Home</li>
    			<li>About</li>
    			<li>Contact</li>
    		</ul>
    	`);
    });
    {% endhighlight %}

- **Don’t catch errors, expect them**

    {% highlight javascript %}
    it("When no product name, it throws error 400", async () => {
      await expect(addNewProduct({}))
        .to.eventually.throw(AppError)
        .with.property("code", "InvalidInput");
    });
    {% endhighlight %}

- **Other generic good testing hygiene: learn and practice TDD**

    Consider writing the tests before the code in a [red-green-refactor style](https://blog.cleancoder.com/uncle-bob/2014/12/17/TheCyclesOfTDD.html), ensure each test checks exactly one thing, when you find a bug — before fixing write a test that will detect this bug in the future, let each test fail at least once before turning green, start a module by writing a quick and simplistic code that satisfies the test - then refactor gradually and take it to a production grade level.

## Front-end layer

- **Separate UI from functionality**

    {% highlight javascript %}
    test("When users-list is flagged to show only VIP, should display only VIP members", () => {
      // Arrange
      const allUsers = [{ id: 1, name: "Yoni Goldberg", vip: false }, { id: 2, name: "John Doe", vip: true }];

      // Act
      const { getAllByTestId } = render(<UsersList users={allUsers} showOnlyVIP={true} />);

      // Assert - Extract the data from the UI first
      const allRenderedUsers = getAllByTestId("user").map(uiElement => uiElement.textContent);
      const allRealVIPUsers = allUsers.filter(user => user.vip).map(user => user.name);
      expect(allRenderedUsers).toEqual(allRealVIPUsers); //compare data with data, no UI here
    });
    {% endhighlight %}

- **Query HTML elements based on attributes that are unlikely to change**

    If the designated element doesn't have such attributes, create a dedicated test attribute like `data-testid="errors-label"`.

- **Whenever possible, test with a realistic and fully rendered component**

    This technique works for small/medium components that pack a reasonable size of child components. Fully rendering a component with too many children will make it hard to reason about test failures (root cause analysis) and might get too slow. In such cases, write only a few tests against that fat parent component and more tests against its children.

- **Have very few end-to-end tests that spans the whole system**

    When coding your mainstream tests (not E2E tests), avoid involving any resource that is beyond your responsibility and control like backend API and use stubs instead (i.e. test double).

    {% highlight javascript %}
    // test
    test("When no products exist, show the appropriate message", () => {
      // Arrange
      nock("api")
        .get(`/products`)
        .reply(404);

      // Act
      const { getByTestId } = render(<ProductsList />);

      // Assert
      expect(getByTestId("no-products-message")).toBeTruthy();
    });
    {% endhighlight %}

- **Speed-up E2E tests by reusing login credentials**

    {% highlight javascript %}
    let authenticationToken;

    // happens before ALL tests run
    before(() => {
      cy.request('POST', 'http://localhost:3000/login', {
        username: Cypress.env('username'),
        password: Cypress.env('password'),
      })
      .its('body')
      .then((responseFromLogin) => {
        authenticationToken = responseFromLogin.token;
      })
    })

    // happens before EACH test
    beforeEach(setUser => () {
      cy.visit('/home', {
        onBeforeLoad (win) {
          win.localStorage.setItem('token', JSON.stringify(authenticationToken))
        },
      })
    })
    {% endhighlight %}

- **Have one E2E smoke test that just travels across the site map**

# 2. Front-end Testing Tools

![pyramid](https://miro.medium.com/max/1200/1*S-WQ9KwM7kkmwKWy41SPYw.png)


## [Jest](https://jestjs.io/)

- Used extensively. Minimal setup and very well documented
- Mocking support and assertion library included
- Ready for visual regression testing (snapshots)
    - Keep in mind: if your component does not update often, is not complex, and is easy to see exactly what you are testing, then a snapshot test might work. Otherwise, you could fall into a bad habit of updating snapshot tests blindly.
- Alternatives:
    - [Jasmine](https://jasmine.github.io/), [Mocha](https://mochajs.org/): older and flexible than Jest, but a lot of configuration is needed. Also, you have to choose a mocking framework or an assertion library explicitly ([Sinon](https://sinonjs.org/) for mocking or [Chai](https://www.chaijs.com/) for asserting test cases, for instance).

## [Testing Library](https://testing-library.com/)

- Works well with Jest and it really challenges you to think hard about what exactly you are testing: testing an individual component is important but testing how all these components work together is arguably more important.
- There is also a [plugin](https://github.com/testing-library/cypress-testing-library) to use testing-library queries for end-to-end tests in Cypress
- Alternatives:
    - [Enzyme](https://enzymejs.github.io/enzyme/):
        - Developed by Airbnb for testing React components’ outputs
        - Different purposes: when you write your tests with react-testing-library, you’re testing your application as if you were the user interacting with the application’s interface. With Enzyme, you’re actually testing the props and state of your components, meaning you are testing the internal behavior of components to confirm that the correct view is being presented to users. If all these props and state variables have this value, then we assume that the interface presented to the user is what we expect it to be.

## [Cypress](https://www.cypress.io/)

- Easy installation, test writing and debugging
- Not based on Selenium
- Runs directly in the browser
    - Cypress Test Runner: browser instance in which you see all your tests' steps on the left-hand side. Cypress makes DOM snapshot before each test steps, so you can easily inspect them.
- No multi-browser testing (only Chrome)
- Alternatives:
    - [Selenium](https://www.selenium.dev/): requires a unit testing framework or a runner plus an assertions library to build out its capabilities. Selenium runs against different browsers and supports N languages.

## Other tools

### [Husky](https://github.com/typicode/husky)

- It could be useful for running our testing scripts in the pre-push Git hook, for instance. The push action will be performed only if all the tests are passed.

# Sources

| Author | Title | URL |
| --- | --- | --- |
@applitools	| Cypress vs Selenium WebDriver: Better, Or Just Different? | https://medium.com/@applitools/cypress-vs-selenium-webdriver-better-or-just-different-2dc76906607d
@eviedevie | Cypress: The future of end-to-end testing for web applications | https://medium.com/tech-quizlet/cypress-the-future-of-end-to-end-testing-for-web-applications-8ee108c5b255
@goldbergyoni | Comprehensive and exhaustive JavaScript & Node.js testing best practices | https://github.com/maremarismaria/javascript-testing-best-practices
@sunilsandhu | I tested a React app with Jest, Enzyme, Testing Library and Cypress. Here are the differences. | https://medium.com/javascript-in-plain-english/i-tested-a-react-app-with-jest-testing-library-and-cypress-here-are-the-differences-3192eae03850
Alex McPeak	| Selenium vs. Cypress: Is WebDriver on Its Way Out? | https://crossbrowsertesting.com/blog/test-automation/selenium-vs-cypress/
Arnab Roy Chowdhury	| Best 8 JavaScript Testing Frameworks In 2020 | https://www.lambdatest.com/blog/top-javascript-automation-testing-framework/
Kent C. Dodds | Static vs Unit vs Integration vs E2E Testing for Frontend Apps | https://kentcdodds.com/blog/unit-vs-integration-vs-e2e-tests
Lyudmil Latinov | Testing with Cypress – lessons learned in a complete framework | https://automationrhapsody.com/testing-with-cypress-lessons-learned-in-a-complete-framework/
Robert C. Martin (Uncle Bob) | "The Cycles of TDD", The Clean Code Blog | https://blog.cleancoder.com/uncle-bob/2014/12/17/TheCyclesOfTDD.html
VVAA | "Testing", The State of JavaScript 2019 | https://2019.stateofjs.com/testing/
Will Soares	| Enzyme vs. react-testing-library: A mindset shift | https://blog.logrocket.com/enzyme-vs-react-testing-library-a-mindset-shift/
@chrisgirard | Testing components with Jest and React Testing Library | https://itnext.io/testing-components-with-jest-and-react-testing-library-d36f5262cde2