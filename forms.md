# Forms

Most non-trivial web applications have forms in one form or another (heh). Forms can also be hard with having to do input validation, display errors, fields being required (or not), depending on another field etc.

Angular generally provides two ways to handle forms, the template way and the code way. The template way is simpler and works great when you only have simple fields with little validation and no complex logic between fields. The code way is where you define a description of your form in your component and then use directives to bind the form to the inputs in your template.

Let's see an example of both.

## The template-driven approach

We'll start with the template way. Let's create a skeleton for our component.

```
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { DudeModel } from './dudemodel';

@Component({
  selector: 'mp-dude-form',
  imports: [FormsModule],
  templateUrl: './dude-form.html',
})
export class DudeForm {
  registerDude(dude: DudeModel) {
    console.log(dude);
  }
}

```

First of all, `FormsModule` does not provide standalone directives, you can't import them one by one, the whole thing has to be imported.

`FormsModule` itself provides the necessary directives for the template-driven apparoach. For the code driven approach we'll use another package later.

Let's also create our template to a separate file, forms tend to grow large because the amount of boilerplate is so high.

_dude-form.html_

```
<h2>Register a new Dude</h2>
<form (ngSubmit)="registerDude()">
  <div><label>Username</label><input name="username" ngModel /></div>
  <div><label>Password</label><input type="password" name="password" ngModel /></div>
  <button type="submit">Register Dude</button>
</form>

```

We created a button and defined an event handler for the `ngSubmit` that's on the form tag. It's emitted by the `NgForm` directive whenever `submit` is triggered.

The simplest way is to add the `ngModel` directives to your form as above. This will have Angular automatically handle the state and value of the field. These single units are represented by `FormControl`. Apart from creating the necessary `FormControl`s it'll also automatically create the `FormGroup` which represents the form as a whole. Note that the `name` tags are required as `FormGroup` uses it under the hood manage things.

Of course, we need to capture the values from the form (username and password in this case). We can define local variables and assign them to the NgForm object created by Angular for your form.

_dude-form.html_

```
<h2>Register a new Dude</h2>
<form (ngSubmit)="registerDude(dudeForm.value)" #dudeForm="ngForm">
  <div><label>Name</label><input name="username" ngModel /></div>
  <div><label>Password</label><input type="password" name="password" ngModel /></div>
  <button type="submit">Register Dude</button>
</form>

```

Submitting this form logs something in the line of `{username: 'Bro', password: password}`. Notice, that despite the `registerDude()` method taking an object of type `DudeModel`, Typescript's type definitions don't exist during runtime, so during runtime it's just `registerDude(dude)`. Template-driven forms don't offer type inference!

This only creates one way data-binding however. Your model will be updated, but your field values won't be. Let's create a two-way binding.

Let's define a new interface:

_usermodel.ts_

```
export interface UserModel {
  username: string;
  password: string;
}
```

And refactor our `dudeform.ts`

_dudeform.ts_

```
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { DudeModel } from './dudemodel';
import { UserModel } from './usermodel';

@Component({
  selector: 'mp-dude-form',
  imports: [FormsModule],
  templateUrl: './dude-form.html',
})
export class DudeForm {
  protected readonly user: UserModel = { username: '', password: '' };

  registerDude() {
    console.log(this.user);
  }
}

```

Note, that we are not using signals. Signals are not well implemented for forms as of writing this text (Late 2025/Angular v20). This will probably change in the future, so you may want to look that up.

So, now we're directly logging the `user` object, how do we access the model from the template? We can use the `ngModel` directive.

_dude-form.html_

```
<h2>Sign up</h2>
<form (ngSubmit)="registerDude()">
  <div><label>Username</label><input name="username" [(ngModel)]="user.username" /></div>
  <div>
    <label>Password</label><input type="password" name="password" [(ngModel)]="user.password" />
  </div>
  <button type="submit">Register</button>
</form>

```

Notice that the syntax used is somewhat uncommon`[(ngModel)]`, what does it mean? It's just a shortcut since this is such a common thing to handle forms, you could also write `<input name="username" [ngModel]="user.username" (ngModelChange)="user.username =
$event">`.

What's happening here is that the `ngModel` directive updates the input value every time the related model changes. Here we are binding to the `user` object with `user.username` and `user.password` to their respective fields.

Now we have two-way binding that updates whenever the model updates, as an example.

_dude-form.html_

```
<h2>Sign up</h2>
<form (ngSubmit)="registerDude()">
  <div><label>Username</label><input name="username" [(ngModel)]="user.username" /></div>
  <div>
    <label>Password</label><input type="password" name="password" [(ngModel)]="user.password" />
  </div>
  <button type="submit">Register</button>
  <div>
    <small>Just a dude named {{ user.username }}</small>
  </div>
</form>

```

This should also make it clear that the value is updated every time the input changes, not just when it's submitted. Submitting of course continues to work.

## The code-driven approach

As talked before, the other way to define forms is the code-driven way or the imperative way. Here we'll construct the form outside the template. It's a more complex approach but also much more powerful than the template-driven approach.

Unlike with template-driven approach where Angular handled `FormControl` and `FormGroup` automatically for us, we have manage them manually. However, Angular does provide a helper class called `FormBuilder` to simplify the code-driven approach.

Let's refactor both our `DudeForm` and `dude-form.html`

_dudeform.ts_

```
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule } from '@angular/forms';

@Component({
  selector: 'mp-dude-form',
  imports: [ReactiveFormsModule],
  templateUrl: './dude-form.html',
})
export class DudeForm {
  private readonly fb = inject(FormBuilder);

  readonly userForm = this.fb.group({
    username: '',
    password: '',
  });

  protected registerDude() {
    console.log(this.userForm.value);
  }
}


```

First we have to change the import. As mentioned, `FormsModule` is meant for the template-driven approach, it's directives do not work here. Instead we import `ReactiveFormsModule`. These directives start with `form` instead of `ng`.

Then we create a form with two controls, the username and the password with empty strings as initial values. We can access the form's value with the `value` attribute. The functionality is exactly the same as in the previous example.

Then the template.

_dude-form.html_

```
<h2>Sign up</h2>
<form (ngSubmit)="registerDude()" [formGroup]="userForm">
  <div><label>Username</label><input formControlName="username" /></div>
  <div><label>Password</label><input type="password" formControlName="password" /></div>
  <button type="submit">Register</button>
</form>

```

We bind the form to our `userForm` object created in the component and now manually define the `formGroup`. We then use the `formControlName` directive to bind the inputs to their controls (these being the values defined in `userForm` in `dudeform.ts`).

You can also update the value of FormControl from your component using `setValue()`

_dudeform.ts_

```
export class DudeForm {
  private readonly fb = inject(FormBuilder);

  protected readonly usernameControl = this.fb.control('');
  protected readonly passwordControl = this.fb.control('');

  readonly userForm = this.fb.group({
    username: this.usernameControl,
    password: this.passwordControl,
  });

  protected registerDude() {
    console.log(this.userForm.value);
  }

  protected somePremadeDude() {
    this.userForm.setValue({
      username: 'TheDude',
      password: 'abide',
    });
  }
}
```

_dude-form.html_

```
<h2>Sign up</h2>
<form (ngSubmit)="registerDude()" [formGroup]="userForm">
  <div><label>Username</label><input formControlName="username" /></div>
  <div><label>Password</label><input type="password" formControlName="password" /></div>
  <button type="submit">Register</button>
  <button type="button" (click)="somePremadeDude()">Use premade dude</button>
</form>
```

## Validating forms

Almost all forms need some form of validation. Fields have dependencies, some fields are required, some require a specific format, some limit their values between specific ranges etc.

We'll continue with both the template-driven forms and the code-driven forms.

Let's start by showing how to make our fields required. We'll start with code-driven forms.

### Code-driven form

Angular provides a `Validator` for this task. These include:

- `Validators.required`, makes sure the control is not empty
- `Validators.minLength(n)`, makes sure the value entered is at least _n_ characters.
- `Validators.maxLength(n)`, makes sure the the value entered is no more than _n_ characters.
- `Validators.email()`, makes sure the value entered is a valid email. This will save you the trouble of writing your own regular expression or using a random one from the internet.
- `Validators.pattern(p)`, makes sure that the value matches some regular expression _p_
- `Validators.min(n)`, makes sure the value entered is at least _n_ (not character count, value itself)
- `Validators.max(n)`, same as min but max value

You're allowed to use several validators, so let's make both of our fields required, set a max limit of 20 characters on our username and a 12 limit minimum on our password.

_dudeform.ts_

```
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({
  selector: 'mp-dude-form',
  imports: [ReactiveFormsModule],
  templateUrl: './dude-form.html',
})
export class DudeForm {
  private readonly fb = inject(FormBuilder);

  protected readonly userForm = this.fb.group({
    username: this.fb.control('', [Validators.required, Validators.maxLength(20)]),
    password: this.fb.control('', [Validators.required, Validators.minLength(12)]),
  });

  protected registerDude() {
    console.log(this.userForm.value);
  }
}
```

Note, that this doesn't produce any errors yet and the value is still printed as usual.

Let's edit the template to disable form submission if the fields don't pass the validations.

_dude-form.html_

```
<h2>Sign up</h2>
<form (ngSubmit)="registerDude()" [formGroup]="userForm">
  <div><label>Username</label><input formControlName="username" /></div>
  <div><label>Password</label><input type="password" formControlName="password" /></div>
  <button type="submit" [disabled]="userForm.invalid">Register</button>
</form>

```

We're simply using the `disabled` property provided by the button and binding it to the `userForm`'s `invalid` property.

So, now we can only submit when the all the controls are valid, but we probably want to display some user message to our users to let them know why the form can't be submitted. Let's add some basic error messages.

_dude-form.html_

```
<h2>Sign up</h2>
<form (ngSubmit)="registerDude()" [formGroup]="userForm">
  <div>
    <label>Username</label><input formControlName="username" />
    @if (userForm.controls.username.hasError('required')) {
    <div class="error">Username is required.</div>
    } @if (userForm.controls.username.hasError('maxlength')) {
    <div>Username should be no longer than 20 characters</div>
    }
  </div>
  <div>
    <label>Password</label><input type="password" formControlName="password" />
    @if (userForm.controls.password.hasError('required')) {
    <div class="error">Password is required.</div>
    } @if (userForm.controls.password.hasError('minlength')) {
    <div>Password should be at least 12 characters long</div>
    }
  </div>
  <button type="submit" [disabled]="userForm.invalid">Register</button>
</form>

```

Errors are now displayed when the validation requirements are not met, but they're also displayed immediately when the form is shown. We should hide them until the user interacts with the fields.

_dude-form.html_

```
<h2>Sign up</h2>
<form (ngSubmit)="registerDude()" [formGroup]="userForm">
  <div>
    <label>Username</label><input formControlName="username" />
    @if (userForm.controls.username.dirty && userForm.controls.username.hasError('required')) {
    <div class="error">Username is required.</div>
    } @if (userForm.controls.username.dirty && userForm.controls.username.hasError('maxlength')) {
    <div>Username should be no longer than 20 characters</div>
    }
  </div>

  <div>
    <label>Password</label><input type="password" formControlName="password" />
    @if (userForm.controls.password.dirty && userForm.controls.password.hasError('required')) {
    <div class="error">Password is required.</div>
    } @if (userForm.controls.password.dirty && userForm.controls.password.hasError('minlength')) {
    <div>Password should be at least 12 characters long</div>
    }
  </div>

  <button type="submit" [disabled]="userForm.invalid">Register</button>
</form>

```

We can use the `dirty` property of the `FormControl` to check if the user has interacted with the field. If they haven't, we don't show any errors.

The template itself is also becoming quite hard to read, we can define references in the component for the controls to make it more readable.

_dudeform.ts_

```
export class DudeForm {
  private readonly fb = inject(FormBuilder);

  protected readonly usernameCtrl = this.fb.control('', [
    Validators.required,
    Validators.maxLength(20),
  ]);

  protected passwordCtrl = this.fb.control('', [Validators.required, Validators.minLength(12)]);

  protected readonly userForm = this.fb.group({
    username: this.usernameCtrl,
    password: this.passwordCtrl,
  });

  registerDude() {
    console.log(this.userForm.value);
  }
}

```

We can then use the references in the template, ie. `(usernameCtrl.dirty && usernameCtrl.hasError('required')`.

### Template-driven forms

Now let's replicate the in the template-driven version of the form.

In the template-driven version we don't have anything in our component like the `FormGroup` to refer to, but we do have the local variable we declared. The variable allows us to access the state of the form and it's controls.

Here's the implementation.

_dude-form.html_

```
<h2>Sign up</h2>
<form (ngSubmit)="registerDude(userForm.value)" #userForm="ngForm">
  <div>
    <label>Username</label>
    <input name="username" ngModel required maxlength="20" #username="ngModel" />
    @if (username.dirty && username.hasError('required')) {
    <div class="error">Username is required.</div>
    } @if (username.dirty && username.hasError('maxlength')) {
    <div>Username should be no longer than 20 characters</div>
    }
  </div>

  <div>
    <label>Password</label>
    <input type="password" name="password" ngModel required minlength="12" #password="ngModel" />
    @if (password.dirty && password.hasError('required')) {
    <div class="error">Password is required.</div>
    } @if (password.dirty && password.hasError('minlength')) {
    <div>Password should be at least 12 characters long</div>
    }
  </div>

  <button type="submit" [disabled]="userForm.invalid">Register</button>
</form>

```

We added the necessary validators to the input fields as attributes. We also defined local variables for each field to access their state. The rest is similar to the code-driven approach. This may seem simpler, but as the form grows larger, the template can become quite messy quite fast.

## Custom validators

Of course, the predefined validators can't cover all use cases, so you often need to define your own custom validators.

We simply need to define a method that takes a `FormControl`, tests it's value as needed, and returns either an object that contains possible errors or null if it passes the validation logic.

Let's write a custom validator that makes sure the given value contains the substring `dude`.

```
import { AbstractControl, ValidationErrors } from '@angular/forms';

export const mustContainDudeInUsername = (
  control: AbstractControl<string | null>
): ValidationErrors | null => {
  const value = control.value ?? '';
  return /dude/i.test(value) ? null : { missingDude: true };
};
```

This validator checks if the value contains `dude` (case insensitive) and returns an error object if it doesn't.

To use it in our code-driven form we simply add it to the validations:

```
  protected readonly usernameCtrl = this.fb.control('', [
    Validators.required,
    Validators.maxLength(20),
    mustContainDudeInUsername,
  ]);
```

You can access it in the form with `usernameCtrl.hasError('missingDude')` to define the logic you want, just as before.

It's also possible to create asynchronous validators, maybe you want to check a the input value in the backend for some reason.

To use this in a template-driven form, you need to define it in a directive. Here's the implementation, we won't go in-depth about it.

```
import { Directive } from '@angular/core';
import {
  AbstractControl,
  NG_VALIDATORS,
  ValidationErrors,
  Validator,
  ValidatorFn
} from '@angular/forms';

export const mustContainDudeInUsername: ValidatorFn = (
  control: AbstractControl
): ValidationErrors | null => {
  const value = control.value;

  if (!value) {
      return null;
  }

  return /dude/i.test(value) ? null : { missingDude: true };
};

@Directive({
  selector: '[mustContainDude]',
  standalone: true,
  providers: [
    {
      provide: NG_VALIDATORS,
      useExisting: MustContainDudeDirective,
      multi: true,
    },
  ],
})
export class MustContainDudeDirective implements Validator {
  validate(control: AbstractControl<string | null>): ValidationErrors | null {
    return mustContainDudeInUsername(control);
  }
}
```

You'll then use it just like other validators in template-driven forms.

Generally though, code-driven forms are easier to setup and manage in the long run if you need any custom logic.

## Grouping

Until now we've had the whole form as a single group. It's also possible to declare subgroups for fields that are related. For example, if you have a password and password confirmation fields you want their values to match.

At this point using template-driven forms becomes a hassle and more complex logic like this should be limited to code-driven forms only.

Let's first define a validator so the fields match:

```
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export const passwordMatchValidator: ValidatorFn = (
  control: AbstractControl
): ValidationErrors | null => {
  const password = control.get('password')?.value;
  const confirm = control.get('confirm')?.value;
  return password && confirm && password === confirm ? null : { matchingError: true };
};
```

This function should be quite self-explanatory.

Then in our `DudeForm` we add the new definitions:

```
  protected passwordConfirmCtrl = this.fb.control('', Validators.required);

  protected readonly passwordGroup = this.fb.group(
    { password: this.passwordCtrl, confirm: this.passwordConfirmCtrl },
    { validators: passwordMatchValidator }
  );

  protected readonly userForm = this.fb.group({
    username: this.usernameCtrl,
    passwordForm: this.passwordGroup,
  });
```

We define the group `passwordGroup` and then add it to the `userForm` with the key `passwordForm`.

To use it in a template we use the `formGroupName` directive in the containing `div` element. Note that I simplified the template for better readability by removing many of errors:

_dude-form.html_

```
<h2>Sign up</h2>
<form (ngSubmit)="registerDude()" [formGroup]="userForm">
  <div>
    <label>Username</label><input formControlName="username" />
    @if (usernameCtrl.dirty && usernameCtrl.hasError('required')) {
    <div>Username is required</div>
    }
  </div>
  <div formGroupName="passwordForm">
    <div>
      <label>Password</label><input type="password" formControlName="password" />
      @if (passwordCtrl.dirty && passwordCtrl.hasError('required')) {
      <div>Password is required</div>
      }
    </div>
    <div>
      <label>Confirm password</label><input type="password" formControlName="confirm" />
      @if (passwordConfirmCtrl.dirty && passwordConfirmCtrl.hasError('required')) {
      <div>Confirm your password</div>
      }
    </div>
    @if (passwordGroup.dirty && passwordGroup.hasError('matchingError')) {
    <div>Your password does not match</div>
    }
  </div>
  <button type="submit" [disabled]="userForm.invalid">Register</button>
</form>

```

You can also define any shared validators control members of the group may have in the group itself using an array.

Template-driven forms require quite a bit of boilerplate to replicate this functionality. Leave them for simple forms requiring minimal custom validations.

## Reactivity in forms

Using code-driven forms allows you to easily react to value changes in your form. For example, it's common to show an indicator of password strength when registering based on length, used character type etc. This would be a prime example of a scenario where you need to react to every input change.

Let's implement a simple one using signals.

```
  protected readonly passwordStrength = toSignal(
    this.passwordCtrl.valueChanges.pipe(
      debounceTime(450),
      distinctUntilChanged(),
      map((newValue) => this.computePasswordStrength(newValue))
    ),
    { initialValue: 0 }
  );

  private computePasswordStrength(value: string | null): number {
    const password = value ?? '';
    let strength = 0;
    if (password.length > 0) strength++;
    if (password.length >= 12) strength++;
    if (/[A-Z]/.test(password)) strength++;
    if (/[a-z]/.test(password)) strength++;
    if (/[0-9]/.test(password)) strength++;
    if (/[^A-Za-z0-9]/.test(password)) strength++;
    return strength;
  }
```

The `passwordStrength` method is the important one here. It uses the `valueChanges` observable of the `passwordCtrl` to listen to changes. We use some RxJS operators to debounce the input and only emit when the value actually changes. Then we map the new value to a computed strength value.

- `debounceTime(450)`, waits for 450 milliseconds of silence before emitting the value. Means we're not doing calculations on every keystroke.

- `distinctUntilChanged()`, only emits when the value actually changes, not on every input event. So if the user types a character then deletes it within the debounce time, no emission occurs.

We can then simply drop this into our template:

```
<div>
  <label>Password</label>
  <input type="password" formControlName="password" />
  <div>Strength: {{ passwordStrength() }}</div>
  @if (passwordCtrl.dirty && passwordCtrl.hasError('required')) {
  <div>Password is required</div>
  }
</div>
```

# Managing value updates

You can also choose different update styles for fields or groups of fields by using the `updateOn` option.

The values it takes are:

- `change`: this is the default, value and validity are updated on every change.

- `submit`: value and validity are updated only when the form is actually submitted.

- `blur`: value and validity are updated only when the field loses focus.

Here is how you would pass it on our `usernameCtrl`:

```
protected readonly usernameCtrl = this.fb.control(
  '',
  {
    validators: [
      Validators.required,
      Validators.maxLength(20),
      mustContainDudeInUsername,
    ],
    updateOn: 'blur',
  }
);
```

It's also available on template-driven forms using `ngModelOptions`:

```
[(ngModel)]="user.username" [ngModelOptions]="{ updateOn: 'blur' }" required>
```

## Summary

- Angular provides two ways to handle forms, the template-driven and code-driven approaches.

- Template-driven forms are simpler to setup but lack type inference and become hard to manage with complex forms.

- Code-driven forms are more complex to setup but provide more power and flexibility, including type inference.

- Angular provides several built-in validators for common validation tasks.

- You can define custom validators for both code-driven and template-driven forms.

- Forms can be grouped into subgroups for better organization and shared validation logic.

- Code-driven forms allow you to easily react to value changes using observables.

- You can manage when values and validity are updated using the `updateOn` option.
