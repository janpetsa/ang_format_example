# TypeScript, briefly

TypeScript is a superset of JavaScript that adds static typing and other features to the language. Angular is built with TypeScript and encourages its use for building applications.

TypeScript files have the `.ts` extension and are transpiled to JavaScript before being executed in the browser. This transpilation step allows TypeScript to provide features that are not natively available in JavaScript.

## Basic Types

To add basic type infromation to your variables, you can use type annotations. Here are some common types in TypeScript:

```
let variable: type;
```

Here are some examples of basic types:

```
const dudeNumber: number = 42;
const dudeName: string = 'Ronjson';
const isDude: boolean = true;
```

In simple cases such as this the type can be inferred by TypeScript, so you can omit the type annotation:

```
const dudeNumber = 42; // inferred as number
const dudeName = 'Ronjson'; // inferred as string
const isDude = true; // inferred as boolean
```

You types can also be based on your code, such as a class:

```
const dude: Dude = new Dude();
```

Typescript also offers generics support such as arrays:

```
const dudes: Array<Dude> = [new Dude(), new Dude()];
```

The notation `<>` indicates a generic type, in this case an array of numbers.

Having type information allows Typescript to catch errors at compile time, before the code is run. For example, the following code would produce a compile-time error:

```
dudes.push('Not a dude'); // Error: Argument of type 'string' is not assignable to parameter of type 'Dude'.
```

You can also allow multiple types using `any`:

```
let anything: any = 42;
anything = 'Now a string';
anything = true; // No error
```

However, using `any` defeats the purpose of TypeScript's type checking and should be avoided when possible. Still, it's sometimes needed when dealing with dynamic data or third-party libraries without type definitions.

If you know your variable can be one of several types, you can use union types:

```
let dudeOrString: Dude | string;
dudeOrString = new Dude(); // OK
dudeOrString = 'Just a string'; // OK
dudeOrString = 42; // Error: Type 'number' is not assignable to type 'Dude | string'.
```

## Enums

Enums allow you to define a set of named constants. Here's an example:

```
enum Color {
  Red,
  Green,
  Blue,
}

let favoriteColor: Color = Color.Green;
```

Enums can make your code more readable and maintainable by giving meaningful names to sets of related values.

In reality, the enum is actually a numeric value, where `Red` is `0`, `Green` is `1`, and `Blue` is `2`. You can also assign specific values to the enum members:

```
enum Color {
  Red = 1,
  Green = 2,
  Blue = 4,
}
```

You can even use strings:

```
enum Color {
  Red = 'RED',
  Green = 'GREEN',
  Blue = 'BLUE',
}
```

Overall though, prefer using union types or literal types over enums for better type safety and flexibility.

You can define your own types using the `type` keyword:

```
type DudeAge = "young" | "middle-aged" | "old";
let myDudeAge: DudeAge = "young";
myDudeAge = "ancient"; // Error: Type '"ancient"' is not assignable to type 'DudeAge'.
```

You can also define a return type for functions:

```
function getDudeAge(dude: Dude): DudeAge {
  if (dude.age < 30) {
    return "young";
  } else if (dude.age < 60) {
    return "middle-aged";
  } else {
    return "old";
  }
}
```

This function takes a `Dude` object as an argument and returns a `DudeAge` type. In many cases TypeScript can infer the return type, so you can omit it if you want, especially for simple functions.

## Interfaces

In JavaScript a function will work if it receives an object with the correct properties, regardless of the object's actual type.

```
function addPointsToDude(dude, points) {
  dude.points += points;
}
```

This function will work as long as the `dude` argument has a `points` property, regardless of what other properties it has or what its actual type is.

In TypeScript you can define an interface to describe the shape of an object:

```
function addPointsToDude(dude: { points: number }, points: number): void {
  dude.points += points;
}
```

This means the `dude` argument must be an object with a `points` property of type `number`.

You can also define a named interface:

```
interface HasPoints {
  points: number;
  name: string;
}
```

Then use it in your function:

```
function addPointsToDude(dude: HasPoints, points: number): void {
  dude.points += points;
}
```

Now every object that implements the `HasPoints` interface can be passed to the function.

## Optional arguments

In JavaScript arguments are optional by default. If you call a function with fewer arguments than it declares, the missing arguments will be `undefined`.

In TypeScript if we tried to call the previous function without the `points` argument:

```
addPointsToDude(dude); // Error: Expected 2 arguments, but got 1.

```

To make an argument optional in TypeScript, you can use the `?` symbol:

```
function addPointsToDude(dude: HasPoints, points?: number): void {
  if (points) {
    dude.points += points;
  }
}
```

Now you can call the function with or without the `points` argument:

```
addPointsToDude(dude); // OK
addPointsToDude(dude, 10); // OK
```

## Functions as properties

You can also define a parameter or property as a function type:

```
interface Dude {
  name: string;
  greet: (greeting: string) => void;
}
```

This means the `greet` property is a function that takes a `string` argument and returns `void`.

```
function greetDude(dude: Dude): void {
  dude.greet('Hello');
}
```

## Classes

A class can implement an interface:

```
interface Dude {
  name: string;
  age: number;
  greet: (greeting: string) => void;
}
```

```
class Person implements Dude {
  constructor(public name: string, public age: number) {}

  greet(greeting: string): void {
    console.log(`${greeting}, my name is ${this.name}`);
  }
}
```

If we tried to call the greet with a number instead of a string:

```
const dude = new Person('Ronjson', 30);
dude.greet(42); // Error: Argument of type 'number' is not assignable to parameter of type 'string'.
```

You're also allowed to implement multiple interfaces:

```
class SuperDude implments Dude, Super {
  // Implementation here
}
```

And interfaces can extend other interfaces:

```
interface Super {
  superPower: string;
}
interface SuperDude extends Dude, Super {}
```

This allows you to create complex types by combining simpler ones.

Classes can also have access modifiers for their properties and methods:

- `public`: The property or method is accessible from anywhere. This is the default.

- `private`: The property or method is only accessible within the class itself.

- `protected`: The property or method is accessible within the class and its subclasses.

Here's an example:

```
class Dude {
  public name: string;
  private secret: string;
  protected age: number;

  constructor(name: string, age: number, secret: string) {
    this.name = name;
    this.age = age;
    this.secret = secret;
  }

  public revealSecret(): void {
    console.log(`My secret is: ${this.secret}`);
  }
}
```

In this example, the `name` property is public and can be accessed from anywhere. The `secret` property is private and can only be accessed within the `Dude` class. The `age` property is protected and can be accessed within the `Dude` class and any subclasses.

## Decorators

Decorators are a special kind of declaration that can be attached to classes, methods, properties, or parameters. They are used to add metadata or modify the behavior of the decorated element.

Angular makes extensive use of decorators to define components, services, directives, and more. They're provided by Angular and are prefixed with `@`.

Here's an example of a simple Angular component using the `@Component` decorator:

```
@Component({ selector: 'ns-home', template: 'home' })
class Home {
  constructor() {
    logger.log('Home');
  }
}
```

In this example, the `@Component` decorator is used to define the `Home` class as an Angular component. The decorator takes a configuration object that specifies the component's selecto and template. When Angular sees this decorator, it knows to treat the `Home` class as a component and apply the specified configuration.

We'll look into decorators in their specific contexts later in the guide.

## Summary

- TypeScript is a superset of JavaScript that adds static typing and other features to the language.

- Type annotations allow you to specify the types of variables, function parameters, and return values.

- Interfaces define the shape of objects and can be implemented by classes.

- Classes can implement interfaces and have access modifiers for their properties and methods.

- Decorators are used to add metadata or modify the behavior of classes, methods, properties, or parameters, and are heavily used in Angular.
