# Pipes

Often times you have to transform the raw data you have to something that fits your view better, it may need formatting, combinining etc. To do this Angular provides a functionality called `pipes`.

Pipes provided by Angular reside in the `CommonModule` module, you can either import single pipes or simply the whole `CommonModule`.

Let's go over some of the more commonly used pipes in Angular.

## slice

`slice` works just like the `slice` method in Javascript.

```
import { Component } from '@angular/core';
import { SlicePipe } from '@angular/common';

@Component({
  selector: 'mp-root',
  imports: [SlicePipe],
  template: `
    <div>
      <p>{{ 'Cool dudes in the summer' | slice : 0 : 4 }}</p>
    </div>
  `,
})
export class App {}

```

Here we import `SlicePipe` and use it in the template. Usage of the pipe is simple, you simply add the pipe character `|` after any data you want to use. The data on the left will be passed on to the pipe on the right.

You could also store the string in a signal and use it like this:

```
<p>{{ dudeString() | slice : 0 : 4 }}</p>
```

Produces the same result.

If given only one index like this:

```
<p>{{ dudeString() | slice : 4 }}</p>
```

It will take the elements from `n` to the end, in this case the output is: `dudes in the summer`

Negative `n` will take the last `n` elements:

```
<p>{{ dudeString() | slice : -6 }}</p>
```

Output: `summer`

If the end index is negative, it counts backwards from the end of the string. For example:

```
<p>{{ dudeString() | slice : 5 : -7 }}</p>
```

Output: `dudes in the`

You can also chain pipes:

```
import { Component, signal } from '@angular/core';
import { SlicePipe } from '@angular/common';

@Component({
  selector: 'mp-root',
  imports: [SlicePipe],
  template: `
    <div>
      <p>Original: {{ dudeString() }}</p>
      <p>First slice: {{ dudeString() | slice : 5 : -7 }}</p>
      <p>
        Chained slices: {{ dudeString() | slice : 5 : -7 | slice : 0 : 5 }}
      </p>
    </div>
  `,
})
export class App {
  readonly dudeString = signal('Cool dudes in the summer');
}

```

This outputs:

```
Original: Cool dudes in the summer

First slice: dudes in the

Chained slices: dudes
```

Slices can be used in all expressions, here's an example of using it with `@for`:

```
@Component({
  selector: 'mp-root',
  imports: [SlicePipe],
  template: `
    <div>
      @for (dude of dudes() | slice : 0 : 2; track dude.id) {
      <div>{{ dude.name }} ({{ dude.age }})</div>
      }
    </div>
  `,
})
export class App {
  protected readonly dudes = signal<DudeModel[]>([
    { id: 1, name: 'Bobby Sax', age: 45 },
    { id: 2, name: 'Ray Gutler', age: 30 },
    { id: 3, name: 'Jonny Rollman', age: 25 },
  ]);
}

```

This will only create corresponding elements to the first two elements in the `dudes` array as we applied the slice pipe to the collection.

## json

The `json` pipe is mostly used for debugging during development. Let's say we have some data, like a signal containing an array of dudes, and during development you want to quickly render what's inside for one reason or another.

You drop this bad boy into your code.

```
<div>
  {{ dudes() }}
</div>
```

But that just ends up displaying something like`[object Object],[object Object],[object Object]`.

To handle this, you can easily pipe the data to `JsonPipe` as follows:

```
<div>
  {{ dudes() | json }}
</div>

```

This will then render the data in it's json form `[ { "id": 1, "name": "Bobby Sax", "age": 45 }, { "id": 2, "name": "Ray Gutler", "age": 30 }, { "id": 3, "name": "Jonny Rollman", "age": 25 } ]`.

You could also chain pipes, ie. `{{ dudes() | slice : 0 : 2 | json }}`, which would result in only the first 2 items to be displayed.

## keyvalue

The `keyvalue` pipe allows you to display the keys and/or values of iterables such as Maps or objects.

Here's an example.

```
import { Component, signal } from '@angular/core';
import { KeyValuePipe } from '@angular/common';

@Component({
  selector: 'mp-root',
  imports: [KeyValuePipe],
  template: `
    <div>
      @for (dude of dudes() | keyvalue; track dude) {
      <p>{{ dude.key }} - {{ dude.value }}</p>
      }
    </div>
  `,
})
export class App {
  protected readonly dudes = signal(
    new Map<number, string>([
      [1, 'Bobby Sax'],
      [2, 'Ray Gutler'],
      [3, 'Jonny Rollman'],
    ])
  );
}

```

This will render a list on names in the form `key - value`, ie. `1 - Bobby Sax`. Any null or undefined keys will be displayed at the end, but as long as you keep your type definitions in check this shouldn't be a problem ;).

The keys are sorted by the following rules:

- If both a strings, lexicographically
- If both are numbers, by their value
- If both are booleans, false before true

If keys have different types they'll be cast to strings and then compared lexicographically.

If you want to change the sorting order you can define your own comparator function. Here we'll define a custom comparator `dudeComparator` that will sort the keys from largest to the smallest.

```
import { Component, signal } from '@angular/core';
import { KeyValuePipe, KeyValue } from '@angular/common';

@Component({
  selector: 'mp-root',
  imports: [KeyValuePipe],
  template: `
    <div>
      @for (dude of dudes() | keyvalue: dudeComparator; track dude) {
      <p>{{ dude.key }} - {{ dude.value }}</p>
      }
    </div>
  `,
})
export class App {
  protected readonly dudes = signal(
    new Map<number, string>([
      [1, 'Bobby Sax'],
      [2, 'Ray Gutler'],
      [3, 'Jonny Rollman'],
    ])
  );
  protected readonly dudeComparator = (
    a: KeyValue<number, string>,
    b: KeyValue<number, string>
  ): number => {
    return b.key - a.key;
  };
}

```

Note that the comparator's type need to be of `KeyValue`.

## async

The `async` pipe is for displaying asynchronously obtained data. It returns `null` until the data is available. Once the promise is resolved the resolved value is returned and change detection check run.

Let's create a promise that resolves after 5 seconds and use the `async` pipe on it.

```
import { Component, signal } from '@angular/core';
import { AsyncPipe } from '@angular/common';

@Component({
  selector: 'mp-root',
  imports: [AsyncPipe],
  template: ` <div>{{ asyncTest | async }}</div> `,
})
export class App {
  protected readonly asyncTest = new Promise((resolve) => {
    window.setTimeout(() => resolve("I've arrived"), 5000);
  });
}

```

This renders nothing until the `promise` is resolved. After resolving, the resolved value is stored in `aSyncTest` and the `async` pipe handles rest.

If you're working with observables (as you often are with Angular) the pipe will handle unsubscribing when the pipe is destroyed. We'll go over observables a bit later.

## Custom pipes

You can also define your own pipes which can obviously be quite useful. Let's define a simple custom pipe for truncating strings to a specified length and adds `...` if the string was truncated.

We need to define a new class that implements the `PipeTransform` interface. This in turn makes sure we use the `transform()` method provided by Angular that'll handle the hard work for us. We also add the `@Pipe` decorator so that Angular knows we're defining a pipe.

Here's our custom `truncate` pipe defined.

```
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
  standalone: true,
})
export class TruncatePipe implements PipeTransform {
  transform(value: string | null, limit: number = 10): string {
    if (!value) return '';
    return value.length > limit ? value.substring(0, limit) + '...' : value;
  }
}
```

We simply check that the value is not an empty string and then use a lambda expression to check whether the given string is a longer than the value we've set (default is 10). If it is, we'll return a substring containing the first 10 letters and append `...` to the end, otherwise return the original string. Note that the type of `value` is the union type `string | null` because we want to use this with the `async` pipe shown earlier which emits `null` while the promise is not resolved.

Usage:

```
import { Component, signal } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { TruncatePipe } from './truncatepipe';

@Component({
  selector: 'mp-root',
  imports: [AsyncPipe, TruncatePipe],
  template: ` <div>{{ asyncTest | async | truncate }}</div> `,
})
export class App {
  protected readonly asyncTest = new Promise<string>((resolve) => {
    window.setTimeout(() => resolve("I've arrived"), 5000);
  });
}
```

It's imported like any other pipe and used like any other pipe. You'll notice that the `name` paraemeter we defined in the `TruncatePipe` is used to call the pipe. That should also clear up why built-in pipes use differing name compared to the import when used in the template.

## Other built-in pipes

A full list of built-in pipes offered by Angular can be found [here](https://angular.dev/guide/templates/pipes#built-in-pipes).
