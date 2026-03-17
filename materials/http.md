# Angular and HTTP

Obviously, nearly all non-trivial projects will use asynchronous data received from a HTTP server. You may be familiar with the standard `fetch` API which is provided by browsers. That's perfectly fine and it can be used with Angular with no friction. If that's a choice you want to make for your project, great!

However, in the world of Angular the norm is to use an Angular service called `HttpClient`. The reason for this is simple: it simplifies testing Angular. Like, a lot. For real. Why? It allows you to easily mock services and return fake responses.

## To get some data

First of all, `HttpClient` is not available by default, you need to configure it in the `app.config.ts` file.

_app.config.ts_

```
import {
  ApplicationConfig,
  provideBrowserGlobalErrorListeners,
  provideZoneChangeDetection,
} from '@angular/core';
import { LogService } from './logservice';
import { provideHttpClient } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideZoneChangeDetection({ eventCoalescing: true }),
    [{ provide: LogService, useClass: LogService }],
    provideHttpClient(),
  ],
};

```

And as you may have figured out, you **inject** it to the component where you want to use it:

```
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({
  providedIn: 'root'
})
export class DudeService {
  private readonly http = inject(HttpClient);
```

The `HttpClient` service offers the usual methods:

- `get`, retrieving data

- `post`, submit entirely new data

- `put`, replace existing data

- `patch`, partial modification of existing data

- `delete`, deletes a specific resource

- `head`, same as `get`` but without any response body, useful for checking if a resource exists

All these methods return an Observable object. This allows for easy cancellation, retry etc.

Since this course focuses on Angular, we're not going to waste any time on setting up our own API. Instead we'll be using [JSONPlaceholder](https://jsonplaceholder.typicode.com/), an open fake API that requires no registration.

Let's define a simple service and add a method for fetching a fake user from the API.

_fakeapiservice.ts_

```
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';

const baseUrl = 'https://jsonplaceholder.typicode.com';

export interface User {
  id: number;
  name: string;
  username: string;
  email: string;
}

@Injectable({
  providedIn: 'root',
})
export class FakeAPIService {
  private readonly http = inject(HttpClient);

  getUsers() {
    return this.http.get<Array<User>>(`${baseUrl}/users`);
  }
}

```

We inject the `HttpClient` and define two methods for fetching either all users or a specific user by id. We then create the `getUsers()` method that returns an Observable of an array of `User` objects, here we let TypeScript infer the return type automatically (`http.get` return an Observable), that's why we don't need to import `Observable` from `rxjs` explicitly.

This only returns the response body, you can also return the full HTTP response if needed.

```
  getUsersResponse() {
    return this.http.get<User[]>(`${baseUrl}/users`, { observe: 'response' });
  }
```

Here the return type is `Observable<HttpResponse<User[]>>`, which contains the full HTTP response including headers, status code etc. The response body can be accessed via the `body` property, status code via the `status` property, headers via the `headers` property and so on.

This is how you can use the previous full example by subscribing to the Observable using `toSignal`:

_app.ts_

```
import { Component, inject, signal } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { FakeAPIService } from './fakeapiservice';

@Component({
  selector: 'mp-root',
  standalone: true,
  template: `
    <div>
      <h1>Users List</h1>

      @if (users(); as userList) {
      <ul>
        @for (user of userList; track user.id) {
        <li>
          <strong>{{ user.name }}</strong> - {{ user.email }}
        </li>
        }
      </ul>
      } @else {
      <p>Loading users...</p>
      }
    </div>
  `,
})
export class App {
  protected readonly users = toSignal(inject(FakeAPIService).getUsers());
}

```

Sending data is similarly easy, simply call the `post` or `put`.

_fakeapiservice.ts_

```
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';

const baseUrl = 'https://jsonplaceholder.typicode.com';

export interface User {
  id: number;
  name: string;
  username: string;
  email: string;
}

@Injectable({
  providedIn: 'root',
})
export class FakeAPIService {
  private readonly http = inject(HttpClient);

  getUsers() {
    return this.http.get<Array<User>>(`${baseUrl}/users`);
  }

  postUser() {
    const payload: User = {
      id: 11,
      name: 'Rock Johnston',
      username: 'rockthemustache67',
      email: 'stacheman55@maansiirtofirma.fi',
    };
    return this.http.post<User>(`${baseUrl}/users`, payload);
  }
}

```

Here we also return the response so we can display it in the template.

_app.ts_

```
import { Component, inject, signal } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { FakeAPIService, User } from './fakeapiservice';

@Component({
  selector: 'mp-root',
  standalone: true,
  template: `
    <div>
      <h1>Users List</h1>

      <button (click)="createUser()">Create fake user</button>

      @if (users(); as userList) {
      <ul>
        @for (user of userList; track user.id) {
        <li>
          <strong>{{ user.name }}</strong> - {{ user.email }}
        </li>
        }
      </ul>
      } @else {
      <p>Loading users...</p>
      } @if (createdUser(); as cu) {
      <section style="margin-top:1rem; padding:0.5rem; border:1px solid #ddd;">
        <h2>Created user (response body)</h2>
        <p><strong>ID:</strong> {{ cu.id }}</p>
        <p><strong>Name:</strong> {{ cu.name }}</p>
        <p><strong>Username:</strong> {{ cu.username }}</p>
        <p><strong>Email:</strong> {{ cu.email }}</p>
      </section>
      }
    </div>
  `,
})
export class App {
  private readonly api = inject(FakeAPIService);

  protected readonly users = toSignal<User[] | undefined>(this.api.getUsers());
  protected readonly createdUser = signal<User | null>(null);

  createUser() {
    this.api.postUser().subscribe((body: User) => {
      console.log('Created user (body):', body);
      this.createdUser.set(body);
    });
  }
}
```

Note that we're also injecting `FakeAPIService` separately here (unlike the previous example) as we use it for multiple methods.

We also didn't need to do any _serialization_ to json, Angular handles all that for you.

To _retry_ requests you can simply do a call like:

```
api.postUser().pipe(retry(5))
```

This will automatically retry a failed request 5 times.

## Options

You can also include _options_ and _headers_ in your calls of course.

`params`

```
  getPostsForUser() {
    return this.http.get<Post[]>(`${baseUrl}/posts`, { params: { userId: '1' } });
  }
```

Equivalent to: `https://jsonplaceholder.typicode.com/posts?userId=1`

`headers`

```
const headers = { Authorization: `Bearer ${token}` };

return this.http.get<Post[]>(`${baseUrl}/posts`, { headers: this.headers });
```

You would use this with ie. JSON Web Tokens for authentication.

## Interceptors

Interceptors simply allow you to intercept requests and responses. A common use case it to add authentication token to requests so that you don't have to do it manually for every request.

```
import { HttpEvent, HttpHandlerFn, HttpInterceptorFn, HttpRequest } from '@angular/common/http';
import { Observable } from 'rxjs';

export const fictionalAPIInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
): Observable<HttpEvent<unknown>> => {
  if (req.url.includes('fictionalapi.fi')) {
    const clone = req.clone({
      setHeaders: { Authorization: `Bearer ${TOKEN}` },
    });
    return next(clone);
  }

  return next(req);
};

```

This would intercept every request going to **fictionalapi.fi** and add the `Authorization` header with our Bearer token. If the request wasn't for **fictionalapi.fi** then it would be passed as is.

You would add the interceptor to your configuration in **app.config.ts**.

```
providers: [
provideHttpClient(withInterceptors([fictionalAPIInterceptor])),
]
```

You can also intercept responses:

```
export const errorHandlerInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
): Observable<HttpEvent<unknown>> => {
  const router = inject(Router);
  const errorHandler = inject(ErrorHandler);

  return next(req).pipe(
    tap({
      error: (errorResponse: HttpErrorResponse) => {
        if (errorResponse.status === HttpStatusCode.Unauthorized) {
          router.navigateByUrl('/login');
        } else {
          errorHandler.handle(errorResponse);
        }
      },
    })
  );
};
```

This would redirect the user to the login page if the response code received from the server was **401 Unauthorized**. Don't worry if you don't understand the code fully, we'll look into Angular's `router` in the next chapter.

The `tap` operator provided by RxJS allows us to "tap into" the response stream and handle errors globally. We can perform side effects based on the data received without modifying the actual response stream.

## Context

It's also possible to give some context to an interceptor. We can create a type safe token `HttpContextToken`, something like this:

```
export const IGNORE = new HttpContextToken<boolean>(() => false);
```

We can then check the context and proceed accordingly:

```
if (req.context.get(IGNORE)) {
  return next(req)
} else {
  // handle it somehow
}
```

You can also set a context to all HTTP methods using the `context` option.

```
const context = new HttpContext().set(IGNORE, true);
return http.get(`${baseUrl}/api/users`, { context })
```

This is a type-safe way to use a defined token. It created a map of the values automatically.
