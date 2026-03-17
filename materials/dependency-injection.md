# Dependency injection

Dependency injection is a common design pattern that simplifies the process of managing dependencies in applications. In Angular, dependency injection is built into the framework and allows you to inject services and other dependencies into your components and other classes.

We could have a component that needs to use another part of our application, ie. a service, that's the dependency. What is applied here is known as "inversion of control". We let the framework create the dependencies and provide them to the component rather than having the component create its own dependencies.

This loose coupling allows for easy replacement of services by another, since the component doesn't know anything about the implementation itself. It also allows for easier testing since the dependencies can easily be replaced by mock versions.

Let's take a simple service that logs to the browser console, a very common operation during development.

_logservice.ts_

```
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root',
})
export class LogService {
  log(message: string): void {
    console.log(message);
  }
}
```

We annotate the component with the `Injectable` annotation and use the `providedIn` parameter with the value `root`. This makes the component available as an injeactable value. You could manually add it to the `providers` array in `app.config.ts`, but using the value `root` for automates the service registration and lets Angular handle the instatiation and injection. For those who are familiar with design patterns, Angular uses the `singleton`-pattern here and makes one instance of this service that's shared between all of its users.

And here's how it's used:

_dudeservice.ts_

```
import { inject, Injectable } from '@angular/core';
import { DudeModel } from './dudemodel';
import { LogService } from './logservice';

@Injectable()
export class DudeService {
  private readonly loggingService = inject(LogService);

  async list(): Promise<Array<DudeModel>> {
    this.loggingService.log('dude-service: get dudes');
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve([
          { id: 1, name: 'Bobby Sax', age: 45 },
          { id: 2, name: 'Jane Doe', age: 30 },
        ]);
      }, 1000);
    });
  }
}


```

Note that we've also marked `DudeService` as an `Injectable`. This is because we're now treating it as a proper replacable service. Since we're only using it locally in a single instance we're omitting the `providedIn` parameter. To use `LogService` we use the `inject` function and give it the service as a parameter. Note that `inject` can only be called when initializing the property or in a constructor. Trying to initialize it inside the `list()` method for example would thrown an error.

You would then use the `DudeService` in the same way.

_app.ts_

```
export class App {
  private readonly dudeService = inject(DudeService);
  protected readonly dudes = signal<DudeModel[]>([]);

  constructor() {
    effect(async () => {
      this.dudes.set(await this.dudeService.list());
    });
  }
}

```

So, how do we switch a dependency? Well, that's a bit of a chore. To define these dependencies we have to add them to our project's `app.config.ts`.

_app.config.ts_

```
import {
  ApplicationConfig,
  provideBrowserGlobalErrorListeners,
  provideZoneChangeDetection,
} from '@angular/core';
import { LogService } from './logservice';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideZoneChangeDetection({ eventCoalescing: true }),
    { provide: LogService, useClass: LogService },
  ],
};

```

This would be the manual equivalent of the `providedIn: 'root'` we added to the `LogService` component. What's happening here is that we're telling the `Injector` that there's a link between the `token` called `LogService` and the class `LogService`. `Injector` will manage the injectable elements based on these definitions. A `token` is usually a class reference (as in the example) or an instance on the `InjectionToken` class. If the token and the class are the same thing, you can just write the short form, ie. `[LogService]`. The token is what defines the instances, so the same service with a different token would be a separate instance.

This of course also means that you can change the class while keeping the same `token`. That's how you can easily switch the dependencies depending on whether your in development, production, testing etc. If we had a production version of LogService that did something else with ie. some backend API, we could simply change it:

```
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideZoneChangeDetection({ eventCoalescing: true }),
    { provide: LogService, useClass: LogAPIService },
  ],
```

Now all the instances where `LogService` is being injected use the new class.

Of course, you'd probably want this to be automated rather than switch the defitinion every single time you were to push into production. We can do something like this:

```
import {
  ApplicationConfig,
  InjectionToken,
  provideBrowserGlobalErrorListeners,
  provideZoneChangeDetection,
  isDevMode,
  inject,
} from '@angular/core';
import { LogService } from './logservice';
import { LogAPIService } from './logapiservice';

export const IS_PROD = new InjectionToken<boolean>('IsProd');

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideZoneChangeDetection({ eventCoalescing: true }),
    { provide: IS_PROD, useValue: !isDevMode() },
    {
      provide: LogService,
      useFactory: () => {
        const isProd = inject(IS_PROD);
        return isProd ? new LogAPIService() : new LogService();
      },
    },
  ],
};

```

To recap: `LogService` is the service used during development, `LogAPIService` is used when in production. We first define a new `InjectionToken` of the type bool called `IS_PROD`. We then use it with `useValue` and Angular's built-in `isDevMode()` method. `isDevMode()` simply returns true if the app is running in dev mode (ie. running using `ng serve`) and false otherwise. Now, `useClass` has been replaced with `useFactory`, which is a function that returns the correct instance based on the value of `IS_PROD`. Note that we use `inject` to get the value of `IS_PROD` inside the factory function. `useClass` doesn't support such dynamic behavior.

If we had defined `IS_PROD` as just a const value, ie. `const IS_PROD = false;` instead. In this case we could use `{ provide: LogService, useClass: IS_PROD ? LogAPIService : LogService }]`, but then we have to manually manage the `IS_PROD` value when going into production.

In this case we should also remove the `providedIn: 'root'` parameter from `LogService`. Manual definitions always have precedence over automatic ones, but others reading the code may find it confusing (which definition is the intended one?).

From a Typescript perspective, you'd probably want to make sure that both `LogService` and `LogAPIService` implement a common interface, this assures the services are interchangeable. Angular doesn't enforce this, but it's a good practice and essentially a requirement in serious production grade applications.

## Injector hierarchy

In some cases we may want to use dependencies that have been declared at a root level at another level. Maybe we have a component that wants to use it's own `LogService` instance.

_dudes.ts_

```
import { Component, inject } from '@angular/core';
import { LogService } from './logservice';
import { LogAPIService } from './logapiservice';

@Component({
  selector: 'mp-dudes',
  providers: [{ provide: LogService, useClass: LogAPIService }],
  template: `<strong>Dudes</strong>`,
})
export class Dudes {
  constructor() {
    inject(LogService).log('Dudes here');
  }
}

```

We simply use the same format as previously defined and add it to the `providers` configuration option. Now `Dudes` class will use it's own version of `LogAPIService` that's not shared by other components.

## Services provided by Angular

Angular has some built-in services. Let's look at a couple of them here.

The first one is the `title` service. It simply allows you to easily change the title of your page by adding a `<title>` element in the `<head>` and updating the value.

```
import { Component, inject } from '@angular/core';
import { Title } from '@angular/platform-browser';

@Component({
  selector: 'mp-root',
  template: `<div>Dudes</div>`,
})
export class App {
  constructor() {
    inject(Title).setTitle('Dudes');
  }
}

```

Simple.

There's also the `meta` that allows you to add or update the `<meta>` tags in the <`head`>.

```
import { Component, inject } from '@angular/core';
import { Meta, Title } from '@angular/platform-browser';

@Component({
  selector: 'mp-root',
  template: `<div>Dudes</div>`,
})
export class App {
  constructor() {
    inject(Meta).addTag({ name: 'Sufferer', content: 'Pain' });
  }
}

```

Would result in: `<meta name="Sufferer" content="Pain">`. You can read more about metadata [here](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta).

We'll see more of Angular's built-in services in other parts where they can be given a more appropriate context.

## Summary

- Dependency injection is a design pattern that simplifies the management of dependencies in applications.

- Angular has built-in support for dependency injection, allowing you to inject services and other dependencies into your components and classes.

- Services are marked with the `@Injectable` decorator and can be provided at the root level or locally within components.

- You can switch dependencies by changing the provider definitions in your application's configuration.
