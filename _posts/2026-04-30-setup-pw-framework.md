---
layout: post
title: Setting up a Playwright framework to test a web based app.
---

<i> Objective: set up a playwright test framework for a web application, in order to test a web front end, apis and back end data. This post assumes Playwright has already been installed and configured for VSC (see previous blog post for a remainder), but focuses on project structure, how to effectively use fixtures/page to set up and use resources for that specific test class, api testing, integration versus component tests and more.
</i>

### System Under Test:

System Under Test:
The system under test for this post is an application I have created which uses a sql backend, and spring boot to set up a web based front end, served by rest apis. This was a straightforward architecture but is relevant in my experience to commercial projects- its possible to use any example project that follows this structure:

| Layer       | Technology                                                        |
| ----------- | ----------------------------------------------------------------- |
| Backend     | Java + Spring Boot                                                |
| API         | REST (JSON endpoints via @RestController)                         |
| Data access | Spring Data JPA + SQLite (existing database, no migration needed) |
| Tests       | Playwright (TypeScript or Java)                                   |
| Front end   | html                                                              |


### Playwright Structure

#### Abstracting page objects and helpers

A playwright test for a web app will involve interacting and verifying a web page, or multiple web pages. Using the page object model in playwright allows each page to have its own class for locators, helpers etc. so multiple tests can re-use this page class. For example in this project, Page Object Model for the landing page. Contains all locators and helper functions relevant to the index page tests.
 
 This approach can be used for other component objects (sidebars, footers etc.), is more maintainable as we only have to update locators in one place, and the test itself is a lot more readable- it only needs to focus on the logic of the test, the implementation is abstracted to the page class.

Example- in this integration test, the `indexPage` page is imported and the test can use the `performSearch` helper.:

```typescript
test('search word not found, returns no results', async ({ indexPage }) => {

await indexPage.goto();

await indexPage.performSearch('green');

await expect(indexPage.noResultsMessage).toBeVisible();

await expect(indexPage.noResultsMessage).toHaveText('No books found.');

	});

});
```

The indexPage page class contains the `performSearch` method:

![pw index page from vsc](/images/pw_index_page.jpg)

 If the test didn't use the page class and its helper, the functional code would have to be included in the test itself- it would work but would make the test harder to read (more lines of code), harder to maintain, and the function would have to be repeated for any other test performing a search on the index page.
 
 For more on POM, see the helpful documentation https://playwright.dev/docs/pom.

### Managing test data

A key consideration in software development test strategies is how to deal with test data- if actual data is used by the tests then the integrity of the data is lost, if we always write new data then the data tables quickly become cluttered and unusable.

In this project the solution adopted is to create a copy 'test' database using the same schema as the 'production' table, but populated with test data.

So the integration tests are run against an instance of a test database, the test can create, read, update or delete data without compromising the original data. To clear down test data, we keep a `test-seed.db` table with the original test data, this is copied over `test.db` by the global setup class, the tests then run and read/write/delete test data, but the original data is reset by the global teardown class.

## Component versus Integration<br>

The test folder covers two different levels of testing- component and integration tests:

**Integration tests** — real Spring Boot + real SQLite

- Test the whole stack works together
- Need the test DB
- Slower, but proves end-to-end behaviour

**Mock/UI tests** — Playwright intercepts all API calls

- Test the frontend in isolation
- No backend needed at all
- Faster, great for edge cases

Integration tests interact with the database, and are required to test the end-to-end process i.e. a button click initiates and api to fetch data. 

For example the integration test `search word not found, returns no results` involves 
inputting a search word e.g. 'green'
clicking search to trigger the url `http://localhost:8080/api/books/search?keyword=green
The response returned is [] so the front end displays "No books found."

![SUT landingpage with network tab](/images/pw-landingpage-network-tab.jpg)

If this project involved hundreds/thousands of book titles, with many new test titles being added, it would be better to be able to mock the api response. In this scenario, the test is purely the response generated for no search results, a discrete component.

The component test `'No books found' on screen when a book not found` mocks the json response to the api route identified in Swagger, i.e. `http://localhost:8080/api/books/search?keyword=green - intercept the api path `**/api/books/search**'` and force it to always return an empty array of book results in the json body: 
`await page.route('**/api/books/search**', async route => {

``` typescript
await page.route('**/api/books/search**', async route => {
await route.fulfill({
status: 200,
contentType: 'application/json',
body: JSON.stringify([])

});
```

This triggers the 'No books found' message.

This is a simplified example, but does ensure that even if we added many mock books to our test database, the search term for this specific test will always return the not found result- we're effectively just testing the not found response, not relying on data.

Note this test uses **page.route** and not **APIRequestContext** (another playwright api tool) as we need the api response to trigger the browser reaction. APIRequestContext would return the API response directly to the test code, but since there's no browser involved, there's nothing to trigger the UI — so playwright couldn't assert that 'No books found.' appears on screen.

### Overall structure <br>

Figure 3 below summarises the structure covered above to illustrate the file structure for this project, there is not a default or requires structure but it is a logical approach to lay out test tets types, set up a dependancy with fixtures and pages etc.

![Project structure diagram](/images/playwright_project_structure_v4.svg)




## Running tests in a CI pipeline
Adopting a playwright framework allows for efficient, robust testing at different levels (i.e. component/integration level)- and as they're automated can be run each time a change is made to regression test the code base. This is particularly powerful in a CI/CD project, where many devs and QAs are pushing code to the main branch- adding Playwright tests to the pipeline means that regression testing can happen as part of integration.

When considering which tests to include in a pipeline, its useful to consider the testing pyramid, a software project would have many unit tests (which are discrete, and 'small'), less component and integration tests (which take more time and resource than unit tests) and a lower amount of end to end tests (these would take longer to run. )

- **Unit tests** — base of the pyramid, fastest, most numerous (not included on this project)
- **Component/integration tests** — middle layer (the component tests with mocked APIs sit here)
- **End-to-end tests** — the UI tests, top of the pyramid, slowest, fewest.

https://www.mountaingoatsoftware.com/uploads/blog/Testpyramid.jpg

For info this project only the component tests are run in the pipeline. The integration tests have a dependancy on the test db, and while its feasible to have a copy in the cloud, following the test pyramid this project runs the lower level tests (component tests) which have no dependancies in the CI pipeline, and the higher level tests (integration) locally:

#### Configuring the tests to run in a CI pipeline

The yml file in the `.github/workflows/` folder enables playwright tests to run in the github CI pipeline: 

specifies which branch to run the tests on:

on: push: branches: [ main, master ] pull_request: branches: [ main, master ]

It specifies the playwright installation, and which set of pw tests to run. 

``` yaml
- name: Run Playwright tests

run: npx playwright test tests/component```