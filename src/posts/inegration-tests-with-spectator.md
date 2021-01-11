---
title: Angular Integration Tests With Spectator
date: '2019-12-24'
tags:
  - angular
  - testing
  - spectator
---
Integration tests are a vital tool in testing front-end code. While unit tests ensure that each component or service is doing what we expect it to do in isolation, integration tests ensure that all these individual parts are working together correctly. [This great article](https://kentcdodds.com/blog/write-tests) by Kent C. Dodds argues that integration tests offer the highest return on investment and therefore should make up the majority of your tests.

[Spectator](https://github.com/ngneat/spectator) is a library which wraps the built-in Angular testing framework to provide a simple but powerful API, resulting in reduced boilerplate and cleaner and more readable tests.

In this article I am going to show you how Spectator helps to write integration tests with ease.

If you want to jump straight to the code it's available on my [GitHub](https://github.com/cjcoops/angular-integration-test-example) and on [StackBlitz](https://stackblitz.com/github/cjcoops/angular-integration-test-example).

## Approach

When writing integration tests, I prefer to write tests which more closely resemble how the end user would use the app. What this means in practice is as follows:

- Less need to test implementation details - only user interactions and outputs
- Fewer mocks (this includes not shallow rendering components)
- Longer tests with multiple assertions which more closely resemble a manual testing workflow

## Sample Application

To demonstrate this approach I test a simple application which does the following:

- On page load it fetches a list of posts from the [JSONPlaceholder](https://jsonplaceholder.typicode.com/) API and renders the response to the screen
- While the data is loading it displays a progress bar
- Once the data is loaded we can select a user from a dropdown which will filter the posts only for that user
- We can type in a search box to filter based on a search criteria with a debounce time of 300ms

For the full code check out the [StackBlitz](https://stackblitz.com/github/cjcoops/angular-integration-test-example) but here's our `PostsService`:

```typescript
export class PostsService {
  constructor(private dataService: DataService) {}

  private posts = new BehaviorSubject<Post[]>(null);
  private loading = new BehaviorSubject<boolean>(true);

  posts$ = this.posts.asObservable();
  loading$ = this.loading.asObservable();

  // call the DataService which is responsible for making HTTP requests and then update the posts and loading states
  load() {
    this.loading.next(true);

    return this.dataService.fetch().pipe(
      tap(response => {
        this.posts.next(response);
        this.loading.next(false);
      })
    );
  }

  // return an observable with the filtered posts
  getPosts(searchTerm: string, userId: number) {
    const filterFunction = (post: Post) =>
      (!searchTerm ||
        post.title.toLowerCase().includes(searchTerm.toLowerCase())) &&
      (!userId || post.userId === userId);

    return this.posts$.pipe(map(posts => posts.filter(filterFunction)));
  }
}
```
I'm not going to go into the details here (I will show you that the implementation details don't actually matter for our tests) but the `PostsService` is responsible for storing our state and loading and filtering the posts data. 

One important thing to point out though is that the app makes use of a `DataService` which makes the HTTP requests. It's a best practice to create a separate service for making HTTP requests anyway, but I find it also makes testing easier as it is simpler to mock the service than it is to mock Angular's `HTTPClient`.

Here's our `PostsComponent`:

```typescript
export class PostsComponent implements OnInit {
  searchTermControl = new FormControl('');
  userFilterControl = new FormControl(null);
  posts$: Observable<Post[]>;
  loading$: Observable<boolean>;

  constructor(private service: PostsService) {}

  ngOnInit(): void {
    // request the posts data
    this.service.load().subscribe();

    // when the searchterm or the user filter changes get the filtered posts based on the filter
    this.posts$ = combineLatest(
      this.searchTermControl.valueChanges.pipe(
        debounceTime(300),
        startWith('')
      ),
      this.userFilterControl.valueChanges.pipe(startWith(null))
    ).pipe(
      switchMap(([searchTerm, userId]) => {
        return this.service.getPosts(searchTerm, userId);
      })
    );

    // loading state observable
    this.loading$ = this.service.loading$;
  }
}
```

## Test Setup

Firstly we need to install Spectator in our project.

``` text
npm install @ngneat/spectator --save-dev
```

Whether you are using Jasmine or Jest, Spectator will work out of the box with no extra configuration, but for this example I will stick with Jasmine.

In `posts.component.spec.ts` we start by setting up our tests using Spectator's `createComponentFactory` function. Here is where we tell Spectator what component we are testing and import any modules the component requires. In our case the component relies on the `FormsModule` and the `ReactiveFormsModule` so we will add them to the `imports` array. 

Here we would also declare any child components or any services that we need to provide, similar to how you would set up an `ngModule`, but in this simple case there are none.

``` typescript
// posts.component.spec.ts
describe('PostsComponent', () => {
  const createComponent = createComponentFactory({
    component: PostsComponent,
    imports: [FormsModule, ReactiveFormsModule],
    mocks: [DataService],
    detectChanges: false
  });

  ...
});
```

When writing integration tests we need to decide which components and services will be covered in the tests and which we will mock. In this case I want my tests to cover the `PostsComponent` and `PostsService`. I don't want to test the `DataService` and therefore I add it to the mocks array. By doing this Spectator automatically mocks the `DataService`, converting each of its functions into a Jasmine spy (`jasmine.createSpy()`). 

I don't want `ngOnInit` to run straight away because I first need to tell the `DataService` how to mock the `fetch` response, and so I set `detectChanges` to `false`.

## Writing the first test

With this set up, we're ready to write our first test! It is going to test the following:

- Initially we are showing the progress bar and no posts exist
- Initially the user dropdown is set to 'All'
- That once we receive our the posts data from the API the page shows the posts and is no longer showing the progress bar

Here's the test:

```typescript
it('should load a list of posts for all users by default', fakeAsync(() => {
  // create the test component
  const spectator = createComponent();

  // get the mocked instance of the DataService
  const dataService = spectator.get(DataService);

  // mock the fetch function to wait 100ms and return 2 posts
  dataService.fetch.and.returnValue(
    timer(100).pipe(
      mapTo([
        {
          userId: 1,
          id: 1,
          title: 'First Post'
        },
        {
          userId: 2,
          id: 2,
          title: 'Another Post'
        }
      ])
    )
  );

  // run ngOnInit
  spectator.detectChanges();

  // assert that the progress bar is showing
  expect(spectator.query(MatProgressBar)).toExist();
  expect(spectator.query(byText('First Post'))).not.toExist();

  // get the user select element
  const select = spectator.query(
    byLabel('Filter by user')
  ) as HTMLSelectElement;

  // assert that it is showing 'All' by default
  expect(select).toHaveSelectedOptions(
    spectator.query(byText('All')) as HTMLOptionElement
  );

  // advance the time 100ms to simulate the HTTP request being made
  spectator.tick(100);

  // assert that the progress bar is not showing and that both our posts are showing
  expect(spectator.query(MatProgressBar)).not.toExist();
  expect(spectator.queryAll(MatListItem).length).toEqual(2);
  expect(spectator.query(byText('First Post'))).toExist();

  expect(dataService.fetch).toHaveBeenCalledTimes(1);
}));
```

OK there's a lot going on here. 

Firstly I'm executing the test in Angular's `fakeAsync` zone. This allows us to easily test asynchronous code in a synchronous way by controlling the passage of time. 

> If you want to understand more about testing asynchronous code in Angular, I thoroughly recommend checking out [this post](https://netbasal.com/testing-asynchronous-code-in-angular-using-fakeasync-fc777f86ed13) by Netanel Basel.

After creating my test component with the factory we created earlier, I mock the `fetch` method on the `DataService`. I'm telling it to return an observable which waits 100ms to simulate an HTTP request and then deliver an array of posts.

With this mock in place I call `spectator.detectChanges()` which is the equivalent of running `ngOnInit()` on the component.

At this point I want to assert that the progress bar exists, that my the posts are not showing and that the user dropdown is showing 'All', which I can do easily using Spectator's custom matcher `toHaveSelectedOptions`.

Then I advance the timer 100ms using `spectator.tick(100)` at which point the `fetch` method will have returned the list of posts.

Finally I can assert that the progress bar no longer exists, and that the two posts exist. I also assert that the `fetch` method has been called exactly once.

Notice a few things about this test:

- Firstly I am not testing any implementation details. In this example I set up my app with a single `PostsComponent` and a `PostsService`, but as my app grew I might want to refactor this to use multiple components or a different method of managing state, for example including a state management library. What's great about this approach is that __I can refactor my code and as long as my solution still called the `DataService` then my tests wouldn't need to change!__ 😄

- Secondly I make multiple assertions. By doing this I avoid sharing state between tests which could be dangerous. What's more it isn't necessary to write a separate test for each assertion as modern test runners are able to tell us exactly what part of the test failed.

- Overall my test closely resembles how a real user would use the app, or how a manual tester would test this functionality and therefore I can be confident in my code. Using Spectator's DOM selectors (e.g. `spectator.query(byLabel('Filter by user'))`) helps with this as this is how a real user would find the element, rather than by the element's `id` for example. It also implicitly goes some way towards testing accessability.

## Filtering the posts

We still need to test filtering the posts so let's finish off by taking a look at a couple more examples and seeing how easy Spectator makes it.

When filtering by user my app uses a select input. For this I can use Spectator's `selectOption` matcher to select my option.

```typescript
const select = spectator.query(
  byLabel('Filter by user')
) as HTMLSelectElement;

spectator.selectOption(
  select,
  spectator.query(byText('User 2')) as HTMLOptionElement
);
```

Note that `selectOption` also runs `detectChanges()` after to reduce boilerplate even further. :thumbsup:

When testing the search term filter I need to simulate a user typing in the input. For that I can use the `typeInElement` helper:

```typescript
const input = spectator.query(byLabel('Filter by title'));

spectator.typeInElement('first', input);

spectator.tick(300);
```

Because I have a 300ms debounce time on the search input change, I need to run `spectator.tick(300)` to advance the timer by 300ms (the test must be run in the `fakeAsync` zone for this to work as discussed earlier).

## Conclusion

Hopefully I have shown you how to write simple and readable integration tests using Spectator, which focus on user interactions and results rather than implementation details, and give you confidence that your code will work as expected for the end user.

For the full code check out [GitHub](https://github.com/cjcoops/angular-integration-test-example) or [StackBlitz](https://stackblitz.com/github/cjcoops/angular-integration-test-example).

## Resources

Kent C. Dodds - [Write Fewer Longer Tests](https://kentcdodds.com/blog/write-fewer-longer-tests)

Kent C. Dodds - [Write tests. Not too many. Mostly integration](https://kentcdodds.com/blog/write-tests)

Netanel Basel - [Testing Asynchronous Code in Angular Using FakeAsync](https://netbasal.com/testing-asynchronous-code-in-angular-using-fakeasync-fc777f86ed13)