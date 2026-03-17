# Directives

We've built few simple components so far and implemented some basic functionality, but how do you do add more complex interaction between them? Most of the time components will pass data between each other. There also has to be some form of lifecycle management for components to keep your app performant. How does Angular handle such basic issues? Using **directives**.

Directives are structurally very similar to components. Instead of merely managing what's displayed and managing the DOM for you, directives allow you to attach custom behavior to HTML elements. This applies to both existing HTML elements and the custom component based ones such as `<ns-dudes>`. You could for example have a draggable directive that allows you make any element it's applied to be draggable.

The basic structure of a directive is familiar:

_behaviorless.ts_

```
import { Directive } from '@angular/core';

@Directive({
  selector: '[Behaviorless]',
})
export class Behaviorless {
  constructor() {
    console.log('Behaviorless directive');
  }
}

```

Instead of the `@Component` decorator found in components, we use `@Directive`.

The `selector` parameter is used to identify components or directives inside HTML templates.

Selectors can be:

- elements, such as `ns-dudes` created by the OtherCoolDudes component
- classes, such as `.alert`
- attributes, such as `[color]` or `[color=red]`
- combinations of all above

You could use the preceding structural directive like this:

_app.ts_

```
import { Component } from '@angular/core';
import { Behaviorless } from './behaviorless';

@Component({
  selector: 'mp-root',
  template: '<div behaviorless>Nothing.</div>',
  imports: [Behaviorless],
})
export class App {}

```

This would simply log `"Behaviorless directive"` on the console whenever the element it's targeting is created or recreated (rendered or re-rendered). In this case the div with the `behaviorless` attribute. Note that HTML elements are case-insensitive, so `behaviorless`, `Behaviorless`, `BeHaViOrLeSs` etc., all work regardless.

## Using input()

Directives and components need to be able to receive data from their parent components. To pass an input to a directive or a component, we can use property bindings.

Let's create a simple component that receives and displays the input.

_passdown.ts_

```
import { Component, input } from '@angular/core';

@Component({
  selector: 'mp-passdown',
  template: '<div>Input is {{ passedDown() }}</div>',
})
export class Passdown {
  readonly passedDown = input<string>();
}


```

And the parent component that inputs the data.

_app.ts_

```
import { Component, input } from '@angular/core';
import { Passdown } from './passdown';

@Component({
  selector: 'mp-root',
  imports: [Passdown],
  template:
  `<div>
    <mp-passdown [passedDown]="passDown()" />
  </div>`,
})
export class App {
  passDown = input<string>('passed down');
}

```

The `passedDown` property defined by `input` is a signal, `InputSignal`. It's not writable (hence the Typescript readOnly property) and the only way to change it is bind another value from the parent component. Note that the the property binding is of the form `[passedDown]="passDown()"` because we're passing the value of the expression `passDown()`. If we had instead wanted to pass a string literal we would've used `passedDown="passed down"`, removing the square brackets from property name.

In the example above, the input is optional, so `passedDown` has the type `InputSignal<string | undefined>`. Angular compiler will not throw an error and an empty string will be shown instead.

To avoid this you have two options:

1. Define the input with a default value: `readonly passedDown = input<string>('default value');`. The default value will be shown when no input is passed down.

2. Make the input `required`, ie. `readonly passedDown = input.required<string>();`. This will cause the Angular compiler to throw an error. Required inputs don't allow default values (for obvious reasons).

Before `input()`, the standard way to handle input was using the `@Input` decorator. As before, it's better to avoid this style if you run into it elsewhere.

## Using output()

For sending data the other way Angular provides the `output()` function. Unlike `input()` which is a form of signal using property bindings, `output()` instead **emits custom events**. The type of event is can be anything from an undefined value or your own objects. In the following example we'll emit a an event that contains an object of the type `DudeModel`. Note, that the following file is a `Component` and not a `Directive` as it does contain a template.

_passup.ts_

```
import { Component, input, output } from '@angular/core';
import { DudeModel } from './dudemodel';

@Component({
  selector: 'mp-passup',
  template: `
    <div>
      <button (click)="selectDude()">{{ dude().name }}</button>
    </div>
  `,
})
export class PassUp {
  readonly dude = input.required<DudeModel>();

  readonly selectedDude = output<DudeModel>();

  protected selectDude() {
    this.selectedDude.emit(this.dude());
  }
}
```

Here we define a property an output property `selectedDude()` with the event containing an object of the type `DudeModel` (you should be able to figure out the type definition on your own). When `this.selectedDude.emit()` is called, an event containing the object is emitted. In this case we simply bind a click event to call the `selectDude()` function whenever the button is pressed to emit.

You could call the output's emit straight from the template, but it's harder to read, extend, and maintain. Not to mention that having complex expressions in templates is generally discouraged. It's better to use a wrapper method such as `selectDude()` even in simple cases such as this.

Next we define the parent component that receives the emitted event.

_app.ts_

```
import { Component, signal } from '@angular/core';
import { DudeModel } from './dudemodel';
import { PassUp } from './passup';

@Component({
  selector: 'mp-root',
  imports: [PassUp],
  template: `<div>
    <div>Selected dude: {{ selectedDude().name }}</div>
    <mp-passup [dude]="dude()" (selectedDude)="onSelectedDude($event)" />
  </div>`,
})
export class App {
  protected readonly dude = signal<DudeModel>({ id: 1, name: 'Bobby Sax', age: 45 });

  protected readonly selectedDude = signal<DudeModel>({ id: 0, name: 'Not a real dude', age: 0 });

  protected onSelectedDude(event: DudeModel) {
    this.selectedDude.set(event);
  }
}
```

Here we are binding both an input property binding and an event binding, you can tell the difference from the brackers `[...]` (property binding, passing an input value) and `(...)` (event binding). With `(selectedDude)="onSelectedDude($event)"` we're telling Angular to listen for the event `selectedDude` and call the function `onSelectedDude` with `$event` (they **payload**). Despite it being a custom event the syntax is no different from standard events, ie. `(click)="f($event)"`.

Note, that `$event` is a special template variable, it can't be renamed. Of course, you don't need to necessary use the payload, it could just be `unidentified` and use to signal that something happened. In this case you could simply write `(theHappening)="somethingHappened()" `.

The function parameter is not tied to `event` however, you can name it as you see fit. ie. `onSelectedDude(dude: DudeModel)`.

Not much of a surprise, but before `output()` there was the `@output` decorator. Again, don't use it if possible.

## Component and directive lifecycles

When Angular needs to display component it constructs it. This where the class constructor gets called. After this any input values are passed to the to the component. When the component is no longer needed it will be destroyed. In this case Javascript virtual machine's garbage collector should deal with object.

We will not go too deep into lifecycles, but there's few things that are worth knowing.

This won't work:

```
export class Dude {
  readonly dude = input.required<Dude>();
  constructor() {
    console.log(`My name is ${this.dude().name}`);
  }
}

```

Why? Because inputs are initialized after the constructor has run, `dude` is therefore not initialized inside the constructor call.

Instead you can use the `ngOnInit` hook that's run once after the component is first initialized.

```
import { OnInit, input } from '@angular/core';

export class Dude implements OnInit {
  readonly dude = input.required<Dude>();
  ngOnInit() {
    console.log(`My name is ${this.dude().name}`);
  }
}
```

Note, that having the class implement the OnInit interface is not strictly required, but you should do so anyway for Typescript's compile time checks and documenting intent.

If you want something to happen every single time an input changes, you can use `ngOnChanges`.

_lifecycle.ts_

```
import { Directive, OnChanges, SimpleChanges, input } from '@angular/core';
import { DudeModel } from './dudemodel';

@Directive({
  selector: '[mp-lifecycle]',
})
export class Lifecycle implements OnChanges {
  readonly dude = input.required<DudeModel>();

  ngOnChanges(changes: SimpleChanges) {
    const dudeHistory = changes['dude'];
    console.log(`I was ${dudeHistory.previousValue?.name}`);
    console.log(`Now I'm ${dudeHistory.currentValue.name}`);
    console.log(`Was this the first time = ${dudeHistory.isFirstChange()}`);
  }
}
```

_app.ts_

```
import { Component, signal } from '@angular/core';
import { DudeModel } from './dudemodel';
import { Lifecycle } from './lifecycle';

@Component({
  selector: 'mp-root',
  imports: [Lifecycle],
  template: `<div>
    <button (click)="changeDudeName()">Change dude name</button>
    <div mp-lifecycle [dude]="dude()"></div>
  </div>`,
})
export class App {
  protected readonly dude = signal<DudeModel>({ id: 1, name: 'Bobby Sax', age: 45 });

  protected changeDudeName(): void {
    this.dude.update((d) => ({ ...d, name: 'still Bobby Sax' }));
  }
}

```

`changes` is a map-like object where each key is the name of an input property (in this case `dude`) and the corresponding value is `SimpleChange` object containing the details such as `previousValue`, `currentValue`, and `firstChange`. The names should be quite self-explanatory.

Lastly, we have the `ngOnDestroy` hook for cleaning up after the component.

```
import { Directive, OnDestroy, input } from '@angular/core';
import { DudeModel } from './dudemodel';

@Directive({
  selector: '[mp-lifecycle]',
})
export class Lifecycle implements OnDestroy {
  readonly dude = input.required<DudeModel>();
  private readonly interval: number;

  constructor() {
    this.interval = window.setInterval(() => console.log(`My name is ${this.dude().name}`), 1000);
  }

  ngOnDestroy(): void {
    window.clearInterval(this.interval);
  }
}
```

If `ngOnDestroy` isn't called the timer would continue running on the background even after this version of the component was destroyed. Every time the component was recreated a new timer would be created. The different timers would simply continue existing and logging on the background, leaking memory.

## Summary

- Directives allow you to attach custom behavior to HTML elements.

- Use `input()` to define input properties for directives and components.

- Use `output()` to define output events for directives and components.

- Lifecycle hooks such as `ngOnInit`, `ngOnChanges`, and `ngOnDestroy` allow you to manage component lifecycles effectively.

- Directives and components can be used together to create dynamic and interactive user interfaces in Angular applications.

- Avoid using the older `@Input` and `@Output` decorators in favor of `input()` and `output()`.
