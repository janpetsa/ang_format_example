# Angular Basics

Let's take a look into Angular basics by creating an Angular app with Angular CLI.

Angular CLI is a command line utility is a tool that allows you easily to initialize, develop, test, maintain and deploy Angular applications. As is the standard for modern Javascript tooling both Node.js and NPM are required. If you don't have them installed follow the instructions on the [official Node.js website](https://nodejs.org/en).

You can verify your installation by executing **node --version** in the terminal. Actively supported versions are listed [here](https://angular.dev/reference/versions).

With that out of the way you can install Angular CLI with the following command:

```
npm install -g @angular/cli
```

After that, you can initialize a new project with:

```
ng new appname --defaults --no-routing --prefix mp
```

This will create a default project skeleton. We don't have to worry about the details of the creation command just yet.

You can run the following command to run the app locally with a local HTTP development server:

```
ng serve
```

If everything has been installed correctly you should be able to visit the default app in your browser.

## Project structure

Opening the created project in your IDE (VS Code, Webstorm etc.) shows you the basic project structure. **package.json** lists the default dependencies for an Angular application, mainly:

- @angular core packages
- zone.js, detection change library
- rxjs, a reactive programming heavily used in Angular applications. We'll cover [RxJS](https://rxjs.dev/) and how it fits in with modern Angular and it's newer built-in mechanism that offer a partial replacement to some the RxJS functionality.

Angular's own configuration file is **angular.json**. You can for example add or rename commands (such as the previously used _ng serve_) if you want, but this course material will use the defaults as they are.

## A simple component

Let's look at basic component in Angular. The project skeleton already contains a very basic one, it can be found at: **src/app/app.ts**.

_app.ts_

```
import { Component, signal } from '@angular/core';

@Component({
  selector: 'mp-root',
  imports: [],
  templateUrl: './app.html',
  styleUrl: './app.css'
})
export class App {
  protected readonly title = signal('ang-basics');
}
```

It's about the simplest component imaginable, but it does contain all the building the basic building blocks. Let's go over the details.

For Angular to know that we're with a component you need to use the **@Component** decorator. It needs to be imported from the **@angular/core** module. The decorator itself needs a configuration object, we'll explain the required properties here and expand on these options later as needed.

### selector

Angular uses the selector to look for a specific element in the HTML-template. In this case, when it finds an element with the name **`</mp-root>`** in the HTML, it will create a new instance of the App-class in its place.

Note that there's a general recommendation to use a prefix in component selectors to avoid issues with name classes when using external libraries. When we created the project we used the `prefix` option with the argument `mp`. This configures a default prefix that is automatically added if you use the CLI to generate new components, ie. `ng generate component name` generates a component where the selector is automatically set to `mp-name`.

You can find the **`</mp-root>`** used in **index.html**.

### templateUrl and template

A component is required to have a template. Templates are either inline or in external files.

For external templates you use the `templateUrl` option. If you generate components using the CLI, it will by default generate a HTML file for the component (along with a CSS file) to accompany it.

You can also use the `template` option and define your HTML inline. For example:

_app.ts_

```
import { Component, signal } from '@angular/core';

@Component({
  selector: 'mp-root',
  imports: [],
  template:
  `<div>
    <h1>Hello, {{ title() }}</h1>
    <p>Congratulations! Your app is running. 🎉</p>
  </div>`,
  styleUrl: './app.css',
})
export class App {
  protected readonly title = signal('ang-basics');
}

```

### styleUrl, styles, imports

While not strictly required, these options are found in the majority of Angular components.

`styleUrl/styles` options are the CSS equivalent of templateUrl/template and should be rather self-explanatory. We'll look into styling later.

`imports` option exists to tell Angular which components, directives, and pipes are used in component template. Almost all non-trivial components will use a subset of Angular's built-in functionalities and they can be imported from `CommonModule` by using `imports: [CommonModule]`. This will import all of them by default or you can explicitly import the ones that are needed. We'll go over these built-in pipes and directives in more detail later.

## Bootstrapping

How is an Angular application then bootstrapped? Using the `bootstrapApplication` function. It can be found in `src/main.ts`:

_main.ts_

```
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { App } from './app/app';

bootstrapApplication(App, appConfig)
  .catch((err) => console.error(err));
```

The root component `App` is imported and used as the input to the bootstrapping function. Note, that using `App` is generally the norm, but in not strictly required.

What about the HTML file? That's found in `src/index.html`. That's our **Single-Page Application (SPA)**. Readers familiar with web development may have noticed that it's missing the script-tag, how could it possibly have any Javascript functionality? When you use `ng serve` the CLI adds handles all of that automatically.

That's an exceedingly simple overview of Angular and everything done here would've been much easier to implement using a static HTML page. But Angular has more to offer. Next we'll dive deeper into templates and the basics of Angular's answer to state management, signals.

## Summary

- Angular applications are built using components.

- Components are defined using the `@Component` decorator.

- Components require a selector and a template.

- Components can also have styles and import other components, directives, and pipes.

- Angular applications are bootstrapped using the `bootstrapApplication` function.
