# Reactive programming in Angular

Let's say we have some value that relies on another value. You could solve handling keeping the state of these values in sync doing something like this:

```
private items: Item[] = [];
private subtotal: number = 0;

addItem(item: Item) {
  this.items.push(item);
  this.recalculateSubtotal();
}

removeItem(index: number) {
  this.items.splice(index, 1);
  this.recalculateSubtotal();
}

private recalculateSubtotal() {
  this.subtotal = this.items.reduce((acc, item) => acc + item.price, 0);
}
```

In this example we are manually managing the `subtotal` state. We have to call the `recalculateSubtotal()` method every time we add or remove an item from the `items` list. This is the `imperative` way to handle such updates. The problem is, if we were to add new a new method and forgot to call `recalculateSutotal()` the state wouldn't be in sync anymore.

Another way to do this is to use a `declarative` approach where we specify **what** the value of `subtotal` is, **not** when to update it.

```
private readonly items = signal<Item[]>([]);
private readonly subtotal = computed(() => {
  return this.items().reduce((acc, item) => acc + item.price, 0);
});

addItem(item: Item) {
  this.items.update(currentItems => [...currentItems, item]);
}

removeItem(index: number) {
  this.items.update(currentItems => {
    const newItems = [...currentItems];
    newItems.splice(index, 1);
    return newItems;
  });
}
```

Here we are using the previously seen `signal` and `computed signal` which are **reactive primitives**.

However, things get more complex when we start dealing with `async` operations. The main issues are:

- The data is not available instantly, you have to wait.
- If the input changes (ie. user clicks on something new) the first request loses its relevancy and should be cancelled. There's a chance of a `race condition` where the old data arrives after the new data.
- You may need data from multiple sources and have to wait for all of them to finish before you can show the new value(s)

In this case using a simple `computed signal` becomes a very complex and error-prone to manage.

In Angular the standard way to handle more complex reactive states is `RxJS`. It provides you with another reactive primitive called `Observable`. `Observable` is a `stream` of events (synchronous or asynchronous). It also comes with it's own operations and functions for combining, transforming, filtering etc. the streams.

## RxJS

The basic cycle is to **subscribe** to an `Observable` and wait for it to emit events. The events can be anything, there can be any number of them, and they can be synchronous or asynchronous. It all comes down to the type of the Observable.

Observables can also **complete**. If an Observable signals it completion it means you will **unsubscribe** from it. After all, there's nothing more to listen for. Observables can also emit an error. In the case of an error you will also unsubscribe.

Let's take a look at a simple (and somewhat pointless) example to see the basic syntax. Note that the style shown here should be avoided for any long-lived streams as it can cause a memory leak.

_app.ts_

```
import { Component } from '@angular/core';
import { filter, from, map, Observable } from 'rxjs';

@Component({
  selector: 'mp-root',
  template: `<p>Check the console for output.</p>`,
})
export class App {
  constructor() {
    const numbers$: Observable<number> = from([2, 4, 6, 8, 10, 12]).pipe(
      map((x) => x * 3),
      filter((x) => x > 20)
    );
    numbers$.subscribe((x) => console.log(x));
  }
}

```

Here we create an Observable of numbers using the `from` function and store it in the variable `numbers`. Using the `$` suffix is a naming convention used to mark an Observable, it's however not part of any official syntax nor is it required. Note that the first part **defines** just defines the stream.

`[1, 2, 3, 4, 5]` is the **source**, the values will be emitted **one by one**. After this we _pipe_ the values received from the source through a series of operators. Note that `filter` and `map` are imported from `rxjs`, they're not the same as the built-in Javascript ones but have the same functionality. As a quick reminder, the operations are:

- `map`, transforms each value, ie. takes `2`, returns `2 * 3 = 6`.

- `filter`, removes values that fail the test (`x > 20`), ie. `6 < 20` = fails the test.

After this we **subscribe** to the stream with `numbers$.subscribe((x) => console.log(x));`. It essentially means that for every value that comes from the pipeline, pass it to `console.log`. It may seem odd that we both define the pipeline and then subscribe separately, but it's a fundamental case of **separation of concerns**. None of the operations run before there's a subscriber (why would you do any work if nobody's listening?). An Observable with no subscibers is a **"cold" observable**.

You can also define a callback handler for errors by giving the `subscribe` method an **observer object**.

```
import { Component } from '@angular/core';
import { filter, from, map, Observable } from 'rxjs';

@Component({
  selector: 'mp-root',
  template: `<p>Check the console for output.</p>`,
})
export class App {
  constructor() {
    const numbers$: Observable<number> = from([2, 4, 6, 8, 10, 12]).pipe(
      map((x) => {
        if (x === 10) {
          throw new Error(`Error: The number ${x} is not allowed!`);
        }
        return x * 3;
      }),
      filter((x) => x > 20)
    );
    numbers$.subscribe({
      next: (x) => console.log('Value:', x),
      error: (err) => console.error('An error was caught:', err.message),
      complete: () => console.log('Stream completed successfully.'),
    });
  }
}
```

Here we'll throw an error if the value from the stream is `10`. The observable will catch it and log the error message. We also defined `complete` that runs only if the the stream manages to finish without an error.

## RxJS + Signals

How would we then use RxJS with Angular's signals? Let's say we have `Dude` class that stores an ID of a dude in a signal. We want RxJS to display the details of a dude whenever the ID changes. However, RxJS requires an Observable, not a signal. Thankfully, Angular provides `toObservable()` function since it's a such a common scenario. `toObservable()` transforms `Signal<T>` into `Observable<T>`and handles any changes to the signal automatically.

Here's an example:

_dude.ts_

```
import { Component, input } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';
import { Observable } from 'rxjs';

@Component({
  selector: 'mp-dude',
})
export class Dude {
  readonly dudeId = input.required<number>();
  private readonly dudeId$: Observable<number> = toObservable(this.dudeId);
}

```

Now let's say we want to fetch the details of a dude whenever the observable emits (and possibly cancel any fetches that are already running). We can use the `switchMap` operator. We won't delve into `switchMap` right now, but it's function is to cancel any previous fetches that have yet to return.

_dude.ts_

```
import { Component, inject, input } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';
import { Observable, of, switchMap } from 'rxjs';
import { DudeService } from './dudeservice';
import { DudeModel } from './dudemodel';

@Component({
  selector: 'mp-dude',
})
export class Dude {
  private dudeService = inject(DudeService);
  readonly dudeId = input.required<number>();
  private readonly dude$: Observable<DudeModel> = toObservable(this.dudeId).pipe(
    switchMap((raceId) => this.dudeService.get(raceId))
  );
}

```

But how do we actually display the data? We obviously need to subscribe to the observable and store the data somewhere for the template to display it. We also need to unsubscribe when the component is destroyed. What we want to do is convert the observable back to a signal using the `toSignal()` function.

Let's use a slightly more complicated example. You can ignore the `timer`-parts, they're just fluff.

**dude.ts**

```
import { Component, inject, input, Signal, effect, signal } from '@angular/core';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { Observable, switchMap, map, timer } from 'rxjs';
import { DudeService } from './dudeservice';
import { DudeModel } from './dudemodel';

@Component({
  selector: 'mp-dude',
  template: `
    <div>
      <button (click)="toggleAutoPlay()">{{ isPlaying() ? 'Pause' : 'Play' }} Auto-Cycle</button>
      @if (dude(); as d) {
      <h2>{{ d.name }}</h2>
      <p>Age: {{ d.age }}</p>
      } @else {
      <p>Dude not found or loading...</p>
      }
      <p>
        <small>Current ID: {{ currentDudeId() }}</small>
      </p>
    </div>
  `,
})
export class Dude {
  private dudeService = inject(DudeService);
  readonly dudeId = input.required<number>();
  protected readonly isPlaying = signal<boolean>(false);
  protected readonly currentDudeId = signal<number>(1);

  private readonly timer$ = timer(0, 2000).pipe(map((tick) => (tick % 6) + 1));
  private readonly timerSignal = toSignal(this.timer$, { initialValue: 1 });

  private readonly dude$: Observable<DudeModel> = toObservable(this.currentDudeId).pipe(
    switchMap((dudeId) => this.dudeService.get(dudeId))
  );
  protected readonly dude: Signal<DudeModel | undefined> = toSignal(this.dude$);

  constructor() {
    effect(() => {
      this.currentDudeId.set(this.dudeId());
    });
    effect(() => {
      if (this.isPlaying()) {
        this.currentDudeId.set(this.timerSignal()!);
      }
    });
  }
  toggleAutoPlay() {
    this.isPlaying.update((playing) => !playing);
  }
}
```

We convert the currentDudeId signal to an observable using `toObservable()`. We then use that observable to fetch the dude data from the service. Finally, we convert the resulting observable back to a signal using `toSignal()`. This way we can use the `dude` signal directly in the template. The timer is used to automatically cycle through dude IDs when auto-play is enabled, it's just for demonstration purposes.

It may seem odd to do an Signal -> Observable -> Signal conversion, but it's the best option since RxJS makes handling asynchronous operations much easier and signals work better with templates. It's the best of both worlds and a commonly used pattern that'll you automatically pick-up on once you get familiar with it.

Note that the signal's type is `DudeModel | undefined`. This is because an Observable is _usually_ asynchronous, meaning there's no value when we're waiting for data to arrive from the server.

Here's the DudeService for completeness:

_dudeservice.ts_

```
import { inject, Injectable, signal } from '@angular/core';
import { DudeModel } from './dudemodel';
import { LogService } from './logservice';
import { Observable, of, throwError } from 'rxjs';

@Injectable()
export class DudeService {
  private readonly loggingService = inject(LogService);
  private readonly dudes = signal<DudeModel[]>([
    { id: 1, name: 'Bobby Sax', age: 45 },
    { id: 2, name: 'Jane Doe', age: 30 },
    { id: 3, name: 'Alice Cooper', age: 28 },
    { id: 4, name: 'Charlie Brown', age: 52 },
    { id: 5, name: 'Diana Prince', age: 35 },
    { id: 6, name: 'Ethan Hunt', age: 41 },
  ]);

  async list(): Promise<Array<DudeModel>> {
    this.loggingService.log('dude-service: get dudes');
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve(this.dudes());
      }, 1000);
    });
  }

  get(id: number): Observable<DudeModel> {
    this.loggingService.log(`dude-service: get dude ${id}`);
    const dude = this.dudes().find((d) => d.id === id);
    if (dude) {
      return of(dude);
    } else {
      return throwError(() => new Error('Dude not found'));
    }
  }
}
```

and

_app.ts_

```
import { Component, signal } from '@angular/core';
import { Dude } from './dude';
import { DudeService } from './dudeservice';
import { LogService } from './logservice';

@Component({
  selector: 'mp-root',
  imports: [Dude],
  providers: [DudeService, LogService],
  template: `
    <div>
      <mp-dude [dudeId]="selectedDudeId()" />
    </div>
  `,
})
export class App {
  protected readonly selectedDudeId = signal<number>(1);
}
```

## Summary

- Signals are great for simple reactive states that are synchronous.

- For more complex asynchronous states RxJS Observables are the better choice.

- You can convert between Signals and Observables using `toObservable()` and `toSignal()`.

- The combination of Signals and Observables allows you to leverage the strengths of both paradigms in your Angular applications.
