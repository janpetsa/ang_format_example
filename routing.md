# Routing in Angular

Angular's router has a simple goal: to manage the state of the application based on the URL in the browser. This means that when the user navigates to different URLs, the router will determine which components to display and what data to load. All of this is done without reloading the entire page, which is a key feature of single-page applications (SPAs).

To do this we use a router. Historically Angular had a built-in router called `ngRoute` . It was however quite simple and not fit for larger projects, especially if you had views inside other views. Back then `ngRoute` as commonly replaced by a 3rd party module called `ui-router`. If an AI-tool ever produces code using these, ignore it.

The modern way is to use Angular's built-in `RouterModule`.

## Getting ready

First we have to refactor a bit.

Since `app.ts` is our entry point it will simply look like this:

_app.ts_

```
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'mp-root',
  imports: [RouterOutlet],
  template: `<router-outlet></router-outlet>`,
})
export class App {}

```

Let's create a simple `home.ts`:

```
import { Component, inject, signal } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { FakeAPIService, User } from './fakeapiservice';

@Component({
  selector: 'mp-home',
  template: '<h1>Home Component</h1>',
})
export class HomeComponent {}

```

And finally a simple `notfound.ts` component:

```
import { Component } from '@angular/core';

@Component({
  selector: 'app-not-found',
  template: `<h1>404 - Page Not Found</h1>
    <p>The page you are looking for does not exist.</p>`,
})
export class NotFoundComponent {}

```

## provideRouter

Angular's router is not provided by default when you create a new Angular application. Instead, you need to explicitly provide it in your application's configuration, just like the HTTP client.

First, let's create a dedicated file where we define our routes.

_app.routes.ts_

```
import { Routes } from '@angular/router';
import { HomeComponent } from './home';
import { NotFoundComponent } from './notfound';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'home', component: HomeComponent },
  { path: '**', component: NotFoundComponent },
];

```

Then add it to our configuration.

_app.config.ts_

```
import {
  ApplicationConfig,
  provideBrowserGlobalErrorListeners,
  provideZoneChangeDetection,
} from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { routes } from './app.routes';
import { provideRouter } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideHttpClient(),
    [provideRouter(routes)],
  ],
};

```

With these simple changes going to `/` or `/home` will render the `HomeComponent`. Any other path will render `NotFoundComponent`. Note that in `app.router.ts` the so called "catch-all" route `**` needs to be **last**. Any route defined after it will be caught by it (hence the name).

## Navigation

You need to able to navigate between components. We can't use the default HTML `a`-tags either, if an user clicks on one of those your whole Angular application will reload, that's what we want to avoid, we're building a SPA after all.

Angular, of course, provides a built-in solutions for handling this. We use the built-in directive `RouterLink` and point to the path where we want to go.

At this point, unless we need to show specific interaction between components that requires definitions in both components, we're going to skip unnecessary component definitions.

Let's modify our **Home** component and add a link to `/dudes`. We also need to define the new route in **app.config.ts** as previously instructed. Just remember the note about the catch-all-route.

_home.ts_

```
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'mp-home',
  imports: [RouterLink],
  template: `
    <h1>Home Component</h1>
    <a routerLink="/dudes">Go to Dude Central</a>
  `,
})
export class HomeComponent {}

```

Import, use the directive, use in the template. Simple as that. Note that the leading slash in the path (`/dudes`) is required. Otherwise the path will be relative to the current component. Let's say your home is in `/home` and you forget the trailing slash. That will then redirect to `/home/dudes` instead of just `/dudes`. This can be useful with nested components, but be aware of the difference.

You can also do navigation straight from the code.

```
import { Component, inject } from '@angular/core';
import { Router, RouterLink } from '@angular/router';

@Component({
  selector: 'mp-home',
  imports: [RouterLink],
  template: `
    <h1>Home Component</h1>
    <a routerLink="dudes">Go to Dude Central</a>
    <button (click)="preferTheLadies()">Go to Lady Central</button>
  `,
})
export class HomeComponent {
  private readonly router = inject(Router);

  protected preferTheLadies() {
    this.router.navigate(['/ladies']);
  }
}
```

You use the `Router` service and it's `navigate` method to do this. It takes the path you want to navigate as the first element and you can add route parameters based on variables for example.

```
  protected preferTheLadies(Id: number) {
    this.router.navigate(['/ladies', Id, 'profile']);
  }
```

With an input of 1 this would navigate to: `/ladies/1/profile`

Navigating from the code can be useful ie. when you need conditional routing (role-based access) or you're waiting for an async operation (fetching from the backend) and want to automatically redirect once the Promise resolves.

You can also define parameters in the URL for when you need dynamic URLs. Let's say we had a specific page for each Dude in the form of `dudes/<dude-id>`. We can define this pattern in the configuration.

```
  { path: 'dudes/:dudeId', component: Dude },
```

Dynamic links can be then defined using `routerLink`

```
<a [routerLink]="['/dudes', dude().id]">See Dude</a>
```

To access those parameters in the target component (`Dude` in this example) you need to use one of Angular's built-in services called `ActivatedRoute`. Once injected you can access the object's `snapshot` property to access the URL parameters.

```
export class Dude {
  protected readonly dude: Signal<DudeModel | undefined>;

  constructor() {
    const route = inject(ActivatedRoute);
    const id = route.snapshot.paramMap.get('dudeId');
    const dudeService = inject(DudeService);
    this.dude = toSignal(dudeService.get(Number(id)));
  }
}
```

This can now be accessed from `dudes/1/bobby`.

Here we are getting and using the `dudeId` parameters and then using `DudeService` to fetch the mock data using it (as previously seen).

We can also achieve this by not using `snapshot` and instead accessing the `paramMap` Observable.

```
import { Component, inject, Signal, computed } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { ActivatedRoute, ParamMap, Router } from '@angular/router';
import { map, switchMap } from 'rxjs';
import { DudeService } from './dudeservice';
import { DudeModel } from './dudemodel';

@Component({
  selector: 'mp-dude',
  standalone: true,
  template: `
    @if (dude(); as d) {
    <h2>{{ d.name }}</h2>
    <p>Age: {{ d.age }}</p>
    <div>
      <button (click)="navigatePrevious()" [disabled]="!hasPrevious()">Previous</button>
      <button (click)="navigateNext()" [disabled]="!hasNext()">Next</button>
    </div>
    } @else {
    <p>Loading...</p>
    }
  `,
})
export class Dude {
  private readonly router = inject(Router);
  private readonly route = inject(ActivatedRoute);
  private readonly dudeService = inject(DudeService);

  protected readonly dude: Signal<DudeModel | undefined>;
  protected readonly currentId = toSignal(
    this.route.paramMap.pipe(map((params: ParamMap) => Number(params.get('dudeId'))))
  );

  protected readonly hasPrevious = computed(() => {
    const id = this.currentId();
    return id !== undefined && id > 1;
  });

  protected readonly hasNext = computed(() => {
    const id = this.currentId();
    return id !== undefined && id < 6;
  });

  constructor() {
    this.dude = toSignal(
      this.route.paramMap.pipe(
        map((params: ParamMap) => params.get('dudeId')!),
        switchMap((id) => this.dudeService.get(Number(id)))
      )
    );
  }

  protected navigatePrevious() {
    const id = this.currentId();
    if (id && id > 1) {
      this.router.navigate(['/dudes', id - 1]);
    }
  }

  protected navigateNext() {
    const id = this.currentId();
    if (id && id < 6) {
      this.router.navigate(['/dudes', id + 1]);
    }
  }
}

```

Here we subscribe to the Observable in the constructor. This way if the parameter changes while we're already in the component (navigating from one dude to another) the data will update accordingly. We also added "Previous" and "Next" buttons to navigate between dudes. Now, when the URL changes from `dudes/x` to `dudes/y` the `paramMap` observable will emit an event and the correct race is fetched and displayed.

The code may seem like a jump in difficulty, but everything used has been covered so far.

## Redirecting

There's many scenarios where you simply want one URL to redirect to another, maybe you want your root URL to redirect to `/home` or `/dashboard` instead.

This is easy to achieve by defining a route as follows:

`{ path: '', pathMatch: 'full', redirectTo: '/home' },`

With redirects, I used pathMatch: 'full' instead of the default 'prefix'.
'prefix' matches when the URL starts with the route’s path, so an empty path ('') with 'prefix' would redirect every URL.

Angular always picks the first matching route, ie:

```
{ path: 'dudes/:id', component: DudeDetailComponent },
{ path: 'dudes/new', component: DudeCreateComponent },
```

For the URL `/dudes/new`, the router matches `dudes/:id` first, so it activates DudeDetailComponent.

In this case, switch them up.

## Route guards

Route guards allow you to restrict routes to specific roles, meaning they can't be accessed through URLs.

Guards come in few different flavors.

1. `canActivate`: disables the activation of a route if set requirements are met. Can also define a navigation to another URL in the case user can't activate the route. Common use case would be prohibiting users who are not logged in from accessing resources meant for logged in users and automatically redirecting them to a login page.

2. `canActivateChild`: allows you to disable the activation of children routes of a specific route. Common use case would be when you have a parent route that requires authentication and you want to ensure that all its child routes also require authentication.

3. `canDeactivate`: used to prevent users from leaving a route. Common use case would be when a user is filling out a form and tries to navigate away without saving. You can prompt the user with a confirmation dialog before allowing them to leave the route.

4. `canMatch`: used to prevent the router from matching a route. Common use is to have several routes with the same path with different components and use `canMatch` to determine which component to load based on specific conditions (like user roles, feature flags, etc.).

You'd add a route like this:

```
{ path: 'dudes', component: DudeCentral, canActivate: [loggedInCheck] }
```

`loggedInCheck` would be a function that contains the logic on whether to allow activation of the route or not, it should return a boolean, `UrlTree` or an Observable or Promise with `boolean|UrlTree`. `UrlTree` should contain the URL where to navigate in case the user is not allowed to activate the route.

Here's an example implementation of `loggedInCheck`:

```
import { inject } from '@angular/core';
import {
  CanActivateFn,
  Router,
  UrlTree,
  ActivatedRouteSnapshot,
  RouterStateSnapshot,
} from '@angular/router';
import { UserService } from './userservice';

export const loggedInCheck: CanActivateFn = (
  _route: ActivatedRouteSnapshot,
  _state: RouterStateSnapshot
): boolean | UrlTree => {
  const userService = inject(UserService);
  const router = inject(Router);

  return userService.isLoggedIn() || router.parseUrl('/login');
};

```

We inject `UserService` to check if the user is logged in. If they are, we return `true` allowing the route activation. If not, we return a `UrlTree` that redirects the user to `/login`.

If you many routes that all share a guard (such as checking if user is logged in), you reduce boilerplate by using an empty-path with no component as a parent.

```
{
  path: '',
  canActivate: [loggedInCheck],
  children: [
    { path: 'some/path', component: CompOne},
    { path: 'dude/aloo', component: CompTwo},
    { path: 'smoke/crack', component: CompThree}
  ]
}
```

This approach also makes your routes easier to structure as it becomes easier to see what paths require logging in and which ones don't.

## Resolvers

Resolvers allow you to fetch data before a route is activated. This ensures that the necessary data is available when the component is loaded, improving user experience by avoiding loading states within the component itself.

This makes development simpler as well, since you can assume the data is already there when the component is initialized and you don't have to handle cases where the data is `null` or `undefined`.

Here's what an example resolver would look like.

```
import { ActivatedRouteSnapshot, ResolveFn } from '@angular/router';
import { DudeModel } from './dudemodel';
import { Observable } from 'rxjs';
import { DudeService } from './dudeservice';
import { inject } from '@angular/core';

export const dudeResolver: ResolveFn<DudeModel> = (
  route: ActivatedRouteSnapshot
): Observable<DudeModel> => {
  const id = route.paramMap.get('dudeId')!;
  const dudeService = inject(DudeService);
  return dudeService.get(id);
};
```

We simply get the `dudeId` parameter and the use our service to fetch our data from the backend with our service. We return an observable of type `DudeModel`. To use this in a route you add as follows:

```
{
path: 'dudes/:dudeId',
  component: Dude,
  resolve: {
    dude: dudeResolver
}
}
```

We associate the resolver with an object key, here we are using `dude`. To use this data in our `Dude` component we would obtain the data using the defined key like this:

```
protected readonly dude = signal(inject(ActivatedRoute).snapshot.data['dude']);
```

## Router extras

Angular's router also supports some additional features that can be useful in certain scenarios:

1. Router events: You can subscribe to various router events to perform actions during navigation, such as logging, analytics, or custom loading indicators.

2. Parameter binding: You can bind route parameters directly to component properties using the `ActivatedRoute` service, making it easier to access dynamic data in your components.

3. Lazy loading: You can configure routes to load modules lazily, improving the initial load time of your application by only loading the necessary code when a route is accessed.

They are however outside the scope of this introduction to go into detail about these features.

## Summary

- Angular's router manages application state based on the URL without reloading the page.

- Use `RouterModule` to set up routing in your Angular application.

- Define routes in a dedicated file and provide them using `provideRouter`.

- Use `RouterLink` directive for navigation in templates and `Router` service for programmatic navigation.

- Define dynamic routes with parameters using `:paramName` syntax and access them using `ActivatedRoute`.

- Use redirects to navigate from one URL to another.

- Use resolvers to fetch data before activating a route, ensuring data availability in components.
