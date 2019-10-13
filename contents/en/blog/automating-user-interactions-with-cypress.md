---
name: 'automating-user-interactions-with-cypress'
title: Automating user interactions with Cypress
year: 19 Aug 2019
id: 'automating-user-interactions-with-cypress'
description: |
  My experience using Cypress to automate user interactions for testing and development
---


At Thriva we are hard at work building the world’s first preventative healthcare company to change the way in which people think about their health. We deeply care about ensuring all our customers have a seamless experience when using our service and one of the ways we do this is by writing end-to-end (E2E) tests using Cypress. Cypress allows you to automate the way in which users interact with the application in the browser, this can be extremely useful for catching bugs but also during the development process.

![](https://cdn-images-1.medium.com/max/2000/1*k4-5E9kKm4oqb6wn4FmaSg.jpeg)

## What is Cypress?

Cypress is a javascript framework for writing E2E tests for web applications, it has mocking, stubbing and assertions built in. As it was built from the ground up, it doesn’t use Selenium at all and is (usually) very performant.

Writing the E2E tests is usually trivial, however we came across a few issues which I will detail in this article that should be useful for anyone else using Cypress.

## Setup

The majority of the Thriva website is built using [Vue.js](https://vuejs.org/), as we scaffolded the project with the Vue cli we get Cypress installed out of the box. It is relatively easy to install by [following the instructions in the docs](https://docs.cypress.io/guides/getting-started/installing-cypress.html)

Below is the folder structure for Cypress:

    # Cypress file structure
    /fixtures
    /plugins
    /specs
    /support

* Fixtures — where you store the files that will be used to mock API calls, images, videos etc.

* Plugins — provide a way of modifying the internal behaviour of Cypress

* Specs — this is where you write your E2E tests

* Support — a place to write utility functions, for example, a function that handles user authentication

## Writing E2E tests

The Cypress docs are fairly comprehensive when it comes to describing the best way of writing E2E tests. Here I will show some of the more useful features that I found when writing E2E tests.

## Stubbing data

Cypress allows you to catch API requests and stub out their data, below we are listening to GET requests to the /v1/auth API endpoint and we return the user fixture. Cypress is clever and is able to find the user.json file within the fixtures folder, we can also add stubs for images, videos etc.

```javascript
cy.server()
cy.fixture('user').as('user')
cy.route('GET', '/v1/auth', '@user')

// user.json
{
  firstName: 'Clem',
  lastName: 'JavaScript',
  company: 'Thriva Health',
  bloodResults: [
    {
      type: 'HbA1c',
      result: 30.4,
      units: 'mmol/mol',
      severity: 'normal'
    }
  ]
}
```

## Editing mocks on the fly

Sometimes you want to test the application under different states, for example, let’s say that we want to test the graph which displays our blood results for a different result value and a high severity. We can edit the fixture before it is used in the test:

```javascript
cy.server()
cy.fixture('user').then((user) => {
  user.bloodResults = [
    {
      type: 'HbA1c',
      result: 60.3,
      units: 'mmol/mol',
      severity: 'high'
    }
  ]
  cy.route('GET', 'v1/auth/**', user).as('user')
})
```

## Waiting for API requests

In certain situations you want to call a real API, perhaps to test your authentication flow. In this case you would want to wait for the API to resolve before continuing with the test. At Thriva we have a page where you can personalise your blood tests to your own personal needs, we need to call our API to get all the pricing’s for all of the different types of tests we offer. We can use cy.wait() to wait for the API to finish before performing our E2E tests:

```javascript
cy.server()
cy.route({
  method: 'GET',
  url: `/v1/blood_tests`
}).as('bloodTests')
cy.wait('@blootTests')

// once this has resolved then the rest of the tests can be run
```

## Writing tests for different devices

By default Cypress runs in a desktop web browser, in reality there is a high probability that the vast majority of your users are accessing the website with their mobile device. Cypress allows you to run your tests as though you were interacting with the app on a mobile, tablet and/or desktop:

```javascript
// Good
beforeAll(() => {
  cy.viewport('iphone-6')
})

// Bad - each time you write an it assertion the browser will reset to a desktop browser.
before(() => {
  cy.viewport('iphone-6')
})
```

The viewport function can take different parameters to render the page at different screen resolutions.

## E2E tests are not unit tests

When writing E2E tests it is not necessary to write assertions for everything like you would in a unit test. Rather, it is better to write assertions for overall functionality — Cypress was designed to be written this way:

```javascript
describe('To do app', () => {
  context('Desktop', () => {
    before(() => {
      //mock out auth
      cy.server()
      cy.fixture('user').as('user')
      cy.route('GET', '/v1/auth', '@user')
      // mock out todos
      cy.fixture('todos').as('todos')
      cy.route('GET', '/v1/todos', '@todos')
    })
    
    // GOOD
    it('should be able to add and remove items to the todos', () =>      {
    // logic to add and remove tests, asserting class names present 
    // and correct to do length
      Cypress._.times(3, (i) => {
        cy.get('.todo-input').type(`test: ${i}`)
        cy.contains('Add todo').click()
      })
      cy.get('.todo').should('have.length', 3)

      Cypress._.times(3, (i) => {
        cy.get('.remove-todo').first().click()
      })
      cy.get('.todo').should('have.length', 0)
}

    // BAD
    it('should have the .added class when todo is added')

    // BAD
    it('should have X number of items added to the todo list')
  })
})
```

## Selector Playground

The selector playground is probably my favourite feature about Cypress, rather than having to write out all your CSS selectors to find the DOM elements manually this tools finds them for you. [The documentation](https://docs.cypress.io/guides/core-concepts/test-runner.html#Selector-Playground) explains very well how to use this correctly.

## Look within

There are times when it is difficult to write query selectors as there are multiple places where there could be a match, this is particularly problematic on forms if you are trying to find a particular input element. Cypress allows you to find the parent DOM element and only look at the child elements within it:
```html
<form class='some-form'>
  <div id='one'>
    <input />
  </div>
  
  <div id='two'>
    <input />
  </div>
  
  <div id='three'>
    <input />
  </div>
</form>
```

Lets say you want to go through the form and fill out each individual input:

```javascript
cy.within('#one', ($el) => { 
  cy.get('input').type('Hello')
})

cy.within('#two', ($el) => { 
  cy.get('input').type('Maybe')
})

cy.within('#three', ($el) => { 
  cy.get('input').type('Bye')
})
```

## Keep it DRY

There are certain checks that you may want to do multiple times, or actions you want to perform before each test. Cypress gives you the ability to write your own custom commands to be used throughout the testing suite. One that we use extensively is cy.auth(), this is a command that mocks out the authentication request as all of our routes are protected. You can also add other commands for any tasks you do repeatedly.

```javascript
Cypress.Commands.add('auth', () => {
  cy.server()
  cy.fixture('auth').as('auth')
  cy.route('GET', '/v1/auth', '@auth')
})

// This can be called within our tests like this:
cy.auth()
```

## Common issues faced

When building out or E2E tests there were a number of issues that we had to overcome to ensure that they work reliably. Our major pain point was in our CI environment (Circle CI) the tests would fail very often.

![](https://cdn-images-1.medium.com/max/2000/1*rzBqx4sW2xu4l8uWNJneBw.jpeg)

There can be a number of things that could be going wrong that can ultimately cause tests to fail but the first step is identifying where there are issues.

## Page performance issues

We found that some of the pages were just not performant enough which would cause Cypress to timeout as it wasn’t able to find the DOM nodes in time as the javascript hadn’t finished evaluating. One of the ways to check this is to run the tests multiple times and find the ones that fail, you can do this by running the following command:

```javascript
// Run the tests x number of times
Cypress._.times(20, (i) => {
  it(`something ${i} times`, () => {
  
  })
})
```

To take this a step further, as the tests are running in a chrome browser it is possible to throttle CPU and network speed. You can do this by clicking in Dev Tools>Performance

If you find tests are failing then it means that something on the page isn’t rendering fast enough for Cypress to find it. You can get past this by adding an increased timeout in your before hook but ideally you would fix the underlying problem:

```javascript
// Not ideal to do this as there is an underlying issue with 
// the page performance to necessitate doing this.
before(() => {
  Cypress.config('defaultCommandTimeout', 20000)
})
```

## Fixtures were too large

Initially, when we were writing our tests we were testing using real data from our staging environment, the issue with this is that if there are any issues with the API then the test will fail. A good rule of thumb is to test the critical routes ( e.g. authentication, purchases and anything critical for the business) with a real API and to stub out the rest of the API request/responses.

As we refactored our tests to use fixture data, one of the issues we faced when writing the tests was that the stubbing of the requests was failing if the JSON representation of the data was too large. Unfortunately, Cypress doesn’t warn you of this so it was only when digging through the Github issues we were able to discover this particular issue. We then had to manually go through the data and trim it down so that Cypress could be able to stub out the API calls correctly.

## Best practices and key learnings

- Mock out as much of the data as possible, ideally using factories to generate random data on the fly — we use [chance.js](https://chancejs.com/) for this purpose.

- Mock everything except the critical routes.

- If tests are failing it is more than likely an issue with your App rather than Cypress.

- Performance test the pages where the tests are failing.

- Use the [selector playground](https://docs.cypress.io/guides/core-concepts/test-runner.html#Selector-Playground) for finding DOM elements, makes writing tests much quicker.

- Don’t use the data property for finding elements, this can break once the JS/CSS is recompiled and these values change.

- Use cy.wait() to wait for API calls to finish.

- When writing frontend code where the state of the application needs to change via UI interaction, Cypress is a great way of automating it.
