# Signal reactivity

We ended the previous part by using `ngOnChanges` to react to input changes. However, `ngOnChanges` can only react to inputs and doesn't offer much atomicity in terms of what to react to. To offer more granular control of your refreshes, Angular provides two functions called `computed()` and `effect()`.

## Computed signals

Let's say we want to have a value that is composed of multiple other values. These values could be stored in either a single signal or multiple. Point being, you don't need every value stored.

The naive approach is as follows:

```
import { Component, input } from '@angular/core';
import { DudeModel } from './dudemodel';

@Component({
  selector: '[mp-computed]',
  template: ` <div>{{ dude().name }} ({{ dude().age }})</div> `,
})
export class ComputedName {
  readonly dude = input.required<DudeModel>();
}

```

This works fine, but let's say our `DudeModel` had a field called `height`. If `height` was updated it would still rerender the template even if we're not using it as Angular's change detection will react to the signal changing.

We could also use `ngOnChanges` like this:

```
@Component({
  selector: '[mp-computed]',
  template: ` <div>{{ nameAge() }}</div> `,
})
export class ComputedName implements OnChanges {
  readonly dude = input.required<DudeModel>();
  protected readonly nameAge = signal<string>('');

  ngOnChanges() {
    this.nameAge.set(`${this.dude().name} (${this.dude().age})`);
  }
}

```

This again works, but introducing another input would require us to recompute the nameAge whenever this unrelated input changes. We also need to initialize nameAge with a default value.

To achieve both the reactivity and the granularity, Angular offers `computed` signals. It's a read-only signal whose value is derived from other signals. It's value can be derived from any number of signals and any number of their fields. It will be recomputed only when the signals and values of it's dependencies change. That means it's **lazy** and **memoized**.

Let's first reimplement the previous example using a `computed` signal:

```
import { Component, input, computed } from '@angular/core';
import { DudeModel } from './dudemodel';

@Component({
  selector: '[mp-computed]',
  template: ` <div>{{ nameAge() }}</div> `,
})
export class ComputedName {
  readonly dude = input.required<DudeModel>();
  protected readonly nameAge = computed(() => `${this.dude().name} (${this.dude().age})`);
}
```

Simply put, the value of `nameAge` will be only recomputed when accessed and only if their dependencies have changed since the last computation. This becomes extremely valuable if you had a value that's based on multiple other signals.

Computed signals can also track other computed signals for chaining. Let's use an example of calculating a (completely fictional) fitness score.

```
const steps = signal(10000);
const caloriesBurned = signal(500);
const activityLevel = signal(0.8);

const baseScore = computed(() => steps() * 0.01 + caloriesBurned() * 0.1);
const adjustedScore = computed(() => baseScore() * activityLevel());
const finalFitnessScore = computed(() => adjustedScore() * 1.2);

```

Here `finalFitnessScore` depends on `adjustedScore`, which depends on `baseCore` etc. Each is recomputed only when accessed and if the dependencies have changed.

## Effects

Another way to react to signal changes is using effects. Instead of deriving a value like computed signals do, effects trigger a side effect when a signal changes. Generally you want to limit the usage of effects to things such as storing a value in local/session storage or updating the DOM shen a signal value. However, you want to avoid using them for updating other signals as that can easily cause infinite loops, just use `computed` signals for those cases.

Here's a simple example of using `effect` to log into the console.

```
import { input, effect, Directive } from '@angular/core';
import { DudeModel } from './dudemodel';

@Directive({
  selector: '[mp-effect]',
})
export class EffectLog {
  readonly dude = input.required<DudeModel>();

  constructor() {
    effect(() => {
      console.log(`Dude info: ${this.dude().name}, Age: ${this.dude().age}`);
    });
  }
}

```

The effect will run every time the parent component passes `dude` as an input. While effects can be defined outside the constructor, you generally want to create them in your constructor. You also get the benefit of automatic lifetime management as Angular will destroy the effect since it was created in the component constructor.

Effects are not run synchronously and will not run immediately when one of their source signals change. They're instead called before change detection and will therefore not notice any intermediate values they signal may have had.

Here's a small example:

_app.ts_

```
import { Component, signal, effect } from '@angular/core';

@Component({
  selector: 'mp-root',
  template: `
    <div>
      <p>Score: {{ number() }}</p>
      <button (click)="addOne()">+1</button>
      <button (click)="addTwo()">+2</button>
    </div>
  `,
})
export class App {
  protected readonly number = signal(0);

  constructor() {
    effect(() => console.log(`Number: ${this.number()}`));
  }

  protected addOne() {
    this.number.update((s) => s + 1);
  }

  protected addTwo() {
    this.addOne();
    this.addOne();
  }
}

```

When you trigger `addTwo()`, it does not log ie. `1` and `2` but simply `2`. This is because `1` is an intermediate value.

This covers most of what'll you need to use signals in basic work.

## Summary

- `computed()` signals are read-only signals whose values are derived from other signals. They are lazy and memoized, recomputing only when accessed and when their dependencies change.

- `effect()` functions are used to trigger side effects in response to signal changes. They are typically used for operations like logging or updating the DOM, and should be created in the component constructor for automatic lifetime management.

- Both `computed` signals and `effects` enhance the reactivity of Angular applications, allowing for more granular control over state changes and side effects.

- Signals provide a powerful way to manage state and reactivity in Angular applications, complementing existing mechanisms like RxJS Observables.

- By leveraging `computed` signals and `effects`, developers can create more efficient and responsive applications.

- Understanding when to use each tool is key to building maintainable and performant Angular applications.
