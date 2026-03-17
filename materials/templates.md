# Templating

Every component needs to have a view. As seen previously, the view can be inline or as a separate in the component's `template` or `templateUrl` property.

## Interpolation

Despite the fancy name **interpolation** is actually a simple concept. It simply means that any valid value in an Angular template that is surrounded by double curly braces `{{}}` is treated as an embedded value that needs to be evaluated and the result displayed instead. This does however come with it's own limitations from the perspective of types.

Let's say we had the following scenario where the person is populated by an asynchronous call and is initially undefined as the template is compiled:

_app.ts_

```
import { Component, signal } from '@angular/core';

@Component({
  selector: 'mp-root',
  imports: [],
  template: `<div>
    <div>Hello {{ person().name }}, age {{ person().age }}.</div>
  </div>`,
})
export class App {
  person = signal<{ name: string; age: number } | undefined>(undefined);
}
```

This will result in the error `Object is possibly 'undefined'`. Is there a way to deal with such situations? You can use `?.`, Javascript's `optional chaining operator`,

Here's a simple (and somewhat ugly) example of it's usage in an simulated asynchronous situation.

```
import { Component, signal } from '@angular/core';

@Component({
  selector: 'mp-root',
  imports: [],
  template: `<div>
    <div>Hello {{ person()?.name }}, age {{ person()?.age }}.</div>
  </div>`,
})
export class App {
  person = signal<{ name: string; age: number } | undefined>(undefined);

  constructor() {
    setTimeout(() => {
      this.person.set({ name: 'Bobby Sax', age: 45 });
    }, 3000);
  }
}

```

The initial state of the person is undefined, but because the function calls in the template are now changed from `person().name` to `person()?.name` Angular accepts that these values may be empty.

After 3 seconds `person` is set to contain a valid value and Angular's change detection system will handle rest, updating the DOM-element as expected.

## Imports

Let's create another component called `OtherCoolDudes`.

_othercooldudes.ts_

```
import { Component } from '@angular/core';

@Component({
  selector: 'ns-dudes',
  template: `<h2>Other cool dudes</h2>`,
})
export class OtherCoolDudes {}

```

We then add it to our `app.ts`.

_app.ts_

```
import { Component, signal } from '@angular/core';

@Component({
  selector: 'mp-root',
  imports: [],
  template:
  `<div>
    <div>Hello {{ person().name }}, age {{ person().age }}.</div>
    <ns-dudes />
  </div>`,
})
export class App {
  person = signal({ name: 'Bobby Sax', age: 45 });
}
```

Nothing's showing up, why? Because we didn't import the OtherCooldDudes component. Angular doesn't know it even exists at this point.

So we import the component. First by importing the component itself and then adding it to the the imports property.

```
import { Component, signal } from '@angular/core';
import { OtherCoolDudes } from './othercooldudes';

@Component({
  selector: 'mp-root',
  imports: [OtherCoolDudes],
  template: `<div>
    <div>Hello {{ person().name }}, age {{ person().age }}.</div>
    <ns-dudes />
  </div>`,
})
export class App {
  person = signal({ name: 'Bobby Sax', age: 45 });
}

```

And there, now we have a bunch of other cool dudes in the mix!

## Property binding

So far we've only seen interpolation using `{{}}` syntax. But what if we want to bind properties of HTML elements to values in our component? This is called **property binding** and is done using square brackets `[]`.

We're talking about DOM properties. Let's look at this simple HTML:

```
<input type="text" value="dude">
```

This input tag has two attributes to it, the `type` attribute and the `value` attribute. When the browser sees this tag, it parses it and creates a corresponding DOM element. The DOM element also has properties that correspond to the attributes. In this case the DOM element will have a `type` property and a `value` property.

When you use property binding in Angular, you're binding to the DOM properties, not the attributes. This is an important distinction because attributes are static and only set when the element is created, while properties can change over time.

Above we used interpolation to display a person's name like this:

```
<div>Hello {{ person().name }}, age {{ person().age }}.</div>
```

This is just syntactic sugar for property binding to the text content of the div element. We could have written it like this instead:

```
<div [textContent]="'Hello ' + person().name + ', age ' + person().age + '.'"></div>
```

DOM properties always contain up-to-date values, while attributes are only initial values. Therefore, when you bind to a property, you're always getting the current value. In the previous example the `value` attribute will always be "dude", but the `value` property will change as the user types in the input field.

## Events

So far we've only dealt with displaying values, but in reality users will interact with your web application to change those values. To handle user input the browser fires events which you can then listed and react to.

Let's go back to OtherCoolDudes and add a button that will display the amount of cool dudes available.

```
import { Component, signal } from '@angular/core';

@Component({
  selector: 'ns-dudes',
  template: `
  <h2>Other cool dudes</h2>
    <div>
      <button (click)="refreshingDudes()">Refresh the list of dudes</button>
      <p>Cool dudes available: {{ dudes().length }}</p>
    </div>`,
})
export class OtherCoolDudes {
  protected readonly dudes = signal<Array<{ name: string; age: number }>>([]);

  protected refreshingDudes(): void {
    this.dudes.set([
      { name: 'Jonnie Rollman', age: 42 },
      { name: 'Ray Gutler', age: 36 },
    ]);
  }
}
```

When initially rendered, the list is empty. The amount of available cool dudes is therefore 0. When the button is pressed Angular detects the click event and calls the `refreshingDudes()` method. The parentheses around **click** tells Angular to listen for a `click` event on that button and then trigger the method.

This is called **event binding** and it lets you react to all types of user interaction like mouse clicks, mouse movements, keyboard presses etc.

It's also good to note that events **bubble up**, that means parent elements are also listening to the events of its children. The following would also work.

```
<div (click)="refreshingDudes()">
  <button>Refresh the list of dudes</button>
  <p>Cool dudes available: {{ dudes().length }}</p>
</div>`
```

Events will keep bubbling up the DOM tree until it's handled or it reaches the root.

It's also possible to access the event itself.

```
import { Component, signal } from '@angular/core';

@Component({
  selector: 'ns-dudes',
  template: ` <h2>Other cool dudes</h2>
    <div (click)="refreshingDudes($event)">
      <button>Refresh the list of dudes</button>
      <p>Cool dudes available: {{ dudes().length }}</p>
    </div>`,
})
export class OtherCoolDudes {
  protected readonly dudes = signal<Array<{ name: string; age: number }>>([]);

  protected refreshingDudes(event: Event): void {
    console.log('Event:', event);
    this.dudes.set([
      { name: 'Jonnie Rollman', age: 42 },
      { name: 'Ray Gutler', age: 36 },
    ]);
  }
}

```

You simply pass `$event` to the method and handle it in the method. It's particularly useful for when you want to cancel default behavior or propagation:

```
  protected refreshingDudes(event: Event): void {
    event.preventDefault();
    event.stopPropagation();
    this.dudes.set([
      { name: 'Jonnie Rollman', age: 42 },
      { name: 'Ray Gutler', age: 36 },
    ]);
  }

```

If you're not familiar with preventDefault() and stopPropagation() you can read about them on MDN:

- [preventDefault()](https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault)
- [stopPropagation()](https://developer.mozilla.org/en-US/docs/Web/API/Event/stopPropagation)

## Control flow syntax

In Angular v18 a new way to handle iterating over collections and adding elements per item was introduced. In the past to iterate special **directives** such as `ngIf`, `ngFor`, `ngSwitch` were used. We'll look into directives later, but they can be thought of as components without a template and they are used to add behavior to an element. The modern Angular way to handle changes to DOM structure like this are `@if`, `@for`, and `@let`.

## @if

Let's use the previous OtherCoolDudes component as an example.

_othercooldudes.ts_

```
import { Component, signal } from '@angular/core';

@Component({
  selector: 'ns-dudes',
  template: ` <h2>Other cool dudes</h2>
    <div (click)="refreshingDudes()">
      <button>Refresh the list of dudes</button>
      <p>Cool dudes available: {{ dudes().length }}</p>
    </div>`,
})
export class OtherCoolDudes {
  protected readonly dudes = signal<Array<{ name: string; age: number }>>([]);

  protected refreshingDudes(): void {
    this.dudes.set([
      { name: 'Jonnie Rollman', age: 42 },
      { name: 'Ray Gutler', age: 36 },
    ]);
  }
}

```

Let's change it so that if the length of `dudes` is 0 we want to display an alternative text instead of the given count. We can simply use the `@if`, `@else if` `@else` instructions. They work just like the equivalent statements in other languages.

```
import { Component, signal } from '@angular/core';

@Component({
  selector: 'ns-dudes',
  template: ` <h2>Other cool dudes</h2>
    <div (click)="refreshingDudes()">
      <button>Refresh the list of dudes</button>
      @if (dudes().length === 0) {
      <p>No cool dudes available</p>
      } @else if (dudes().length === 1){
      <p>One cool dude available</p>
      } @else {
      <p>Cool dudes available: {{ dudes().length }}</p>
      }
    </div>`,
})
export class OtherCoolDudes {
  protected readonly dudes = signal<Array<{ name: string; age: number }>>([]);

  protected refreshingDudes(): void {
    this.dudes.set([
      { name: 'Jonnie Rollman', age: 42 },
      { name: 'Ray Gutler', age: 36 },
    ]);
  }
}
```

It also possible to use an alias as condition by storing it in a local variable in case you want to use it multiple times. Here we use `dudeMound` as a local variable to store `dudes().length`.

```
@if (dudes().length; as dudeMound) {
  <p>Cool dudes available: {{ dudeMound }}</p>
} @else {
  < p>No cool dudes available</p>
}
```

## @for

Currently we're only displaying the amount of dudes available, but often you want to display the actual list of the data you have as separate elements. This is where `@for` comes in. It allows you display an element for every item in your collection. Let's display a list of names.

_othercooldudes.ts_

```
import { Component, signal } from '@angular/core';

@Component({
  selector: 'ns-dudes',
  template: ` <h2>Other cool dudes</h2>
    <div (click)="refreshingDudes()">
      <button>Refresh the list of dudes</button>
      @if (dudes().length === 0) {
      <p>No cool dudes available</p>
      } @else {
      <ul>
        @for (dude of dudes(); track dude.id) {
        <li>{{ dude.name }}</li>
        }
      </ul>
      }
    </div>`,
})
export class OtherCoolDudes {
  protected readonly dudes = signal<Array<{ id: number; name: string; age: number }>>([]);

  protected refreshingDudes(): void {
    this.dudes.set([
      { id: 1, name: 'Jonnie Rollman', age: 42 },
      { id: 2, name: 'Ray Gutler', age: 36 },
    ]);
  }
}

```

Notice that we modified the type definition of the object inside the dudes signal. We added an `id` field. We also modified the setter inside `refreshingDudes()` to include an id for every dude.

We're using `@for` on every item in the `dudes` array and create a corresponding list (`li`) item displaying the name of the dude.

But why did we add the `id` field to the type definition? As can be seen, Angular requires that a `@for` instruction is given a `track` parameter. This is added so that Angular can efficiently track the items in the collection and update the DOM as needed. The tracked value needs to be unique (ie. `id` here), but you can also track the item itself (ie. `track dude`) if you are working with data where you have no unique identifying fields. This does come with a performance overhead because of the complexity of the tracked object in comparison to a simple number value.

`@for` also comes up with some useful variables of it's own.

- `$index`, index of the current item
- `$odd`, boolean that the item index is odd
- `$even`, boolean that the item index is even
- `$first`, boolean that the item is first in the collection
- `$lastm`, boolean boolean that the item is last in the collection

Here we use `$odd` to apply an inline style to every even member of the collection.

```
<ul>
  @for (dude of dudes(); track dude.id) {
  <li [style.color]="$odd ? 'blue' : 'black'">{{ dude.name }}</li>
  }
</ul>
```

You can also use an alias.

```
<ul>
  @for (dude of dudes(); let isOdd = $odd;  track dude.id) {
  <li [style.color]="isOdd ? 'blue' : 'black'">{{ dude.name }}</li>
  }
</ul>
```

## @let

You can also define template variables using `@let` without having declaring it in the component class itself.

It can be useful in cases where you have to reuse a value several times in your template. Let's imagine our dudes data also stored an address field with multiple subfields:

```
@for (dude of dudes(); track dude.id) {
  <div class="name">{{ dude.name }}</div>
  <div class="age">{{ dude.age }}</div>
  <div class="address">
    <span>{{ dude.address.default.street }}</span>
    <span>{{ dude.address.default.number }}</span>
    <span>{{ dude.address.default.zip }}</span>
    <span>{{ dude.address.default.city }}</span>
  </div>
}
```

Instead of repeating ourselves with every call we can write it more clearly using a template variable:

```
@for (dude of dudes(); track dude.id) {
  <div class="name">{{ dude.name }}</div>
  <div class="age">{{ dude.age }}</div>
  <div class="address">
    @let address = dude.address.default;
    <span>{{ address.street }}</span>
    <span>{{ address.number }}</span>
    <span>{{ address.zip }}</span>
    <span>{{ address.city }}</span>
  </div>
}
```

## Structural directives

In older versions of Angular (before v18) `@let`, `for`, and `switch` control flow syntax did not exist. Instead structural directives such as `NgIf`, `NgFor`, and `NgSwitch` were used.

We will not cover those in depth, but since the modern control flow syntax is quite recent and structural directives have been the standard way to handle control flow for most of Angular's history, you may run into them in online materials, AI-generated code etc,. so it's still important to be aware of them. While they're still valid and work just fine, they are considered deprecated and you should avoid using them in new projects.

## Summary

- Templates define the view of a component.

- Interpolation is done using `{{}}` syntax.

- Event binding is done using `(eventName)="handler()"` syntax.

- Control flow syntax includes `@if`, `@for`, and `@let` for handling conditional rendering and iteration.

- Structural directives are the older way to handle control flow and should be avoided in new projects.
