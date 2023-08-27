# Use Destructuring To Handle Forms In React!

There have been numerous attempts at handling forms in React such as
[Formik](https://formik.org/),
[React Hook Form](https://www.react-hook-form.com/)
and
[Redux Form](https://redux-form.com/).
All of these libraries assist you by managing your form state for you.

However, `useDestructuring` takes a different approach.
Instead of managing your state for you, it allows you to manage your own state,
resulting in simplified forms and greater decoupling between components.

The source code of `useDestructuring` can be found in the
[Github repository](https://github.com/Krassnig/use-destructuring) and
[NPM package](https://www.npmjs.com/package/use-destructuring).

## How To useDestructuring

`useDestructuring` generates corresponding `setState` functions for every property in the `person` object.
Initially, this appears to just free you from writing `setPerson(oldState => { ...oldState, 'firstName': newName })`.
However, when you start separating the form into individual components, this function will massively simplify the entire component tree and make each of those component reusable across your application.

```tsx
import { useState } from "react";
import useDestructuring from "use-destructuring";

interface Person {
    firstName: string;
    lastName: string;
}

export const PersonForm: React.FC = () => {
    
    const [person, setPerson] = useState<Person>({ firstName: 'John', lastName: 'Smith' });

    const { firstName, lastName } = person; // Javascript object destructuring
    const { setFirstName, setLastName } = useDestructuring(person, setPerson);

    return (
        <form>
            <input type="text" value={firstName} onChange={e => setFirstName(e.target.value)} />
            <input type="text" value={lastName} onChange={e => setLastName(e.target.value)} />
        </form>
    );
};
```

## Abstraction

To share state between a parent and a child component two arguments must be passed from the parent to the child component, one for reading state and one for updating state.
Reacts type definition for updating state is not very straightforward and implementing the type definition is even more complicated.
```ts
type ReactSetState<T> = React.Dispatch<React.SetStateAction<T>>;
type ReactSetState<T> = (value: (T | ((prevState: T) => T))) => void;
```
However, it allows you to hold state in a parent component while retaining full control over that state in its child component.
For instance, what if were to extract the `<input />` from the previous example and place it into its own component.

```tsx
type Props = {
    text: string,
    setText: React.Dispatch<React.SetStateAction<string>>
}

const TextInput: React.FC<Props> = ({ text, setText }) => {
    return (
        <input type="text" value={text} onChange={e => setText(e.target.value)} />
    );
};
```

Inside the `TextInput` component, our main concern is the text string.
However, in the `PersonForm`, we might have a more complex object like `Person`.
This is precisely where the utility `useDestructuring` becomes apparent.
It automatically generates all the necessary `ReactSetState` functions,
enabling you to keep the `TextInput` component decoupled from the more complex `PersonForm` component.
The only way these two components remain coupled is through their shared state which follows Reacts convention for two way binding given by useState.

```tsx
import { useState } from "react";
import useDestructuring from "use-destructuring";

interface Person {
    firstName: string;
    lastName: string;
}

export const PersonForm: React.FC = () => {

    const [person, setPerson] = useState<Person>({ firstName: 'John', lastName: 'Smith' });

    const { firstName, lastName } = person; // Javascript object destructuring
    const { setFirstName, setLastName } = useDestructuring(person, setPerson);

    return (
        <form>
            <TextInput text={firstName} setText={setFirstName} />
            <TextInput text={lastName} setText={setLastName} />
        </form>
    );
};
```

## Arrays

`useDestructuring` can also be utilized to destructure arrays.
For instance, let's extend the `Person` object by including a list of phone numbers.
To keep the example component comprehensible a new component `PhoneNumbersInput` is created.

```tsx
import { useState } from "react";
import useDestructuring from "use-destructuring";

interface Person {
    firstName: string;
    lastName: string;
    phoneNumbers: string[];
}

export const PersonForm: React.FC = () => {
    
    const [person, setPerson] = useState<Person>({
        firstName: 'John',
        lastName: 'Smith',
        phoneNumbers: []
    });

    const { firstName, lastName, phoneNumbers } = person;
    const { setFirstName, setLastName, setPhoneNumbers } = useDestructuring(person, setPerson);

    return (
        <form>
            <TextInput text={firstName} setText={setFirstName} />
            <TextInput text={lastName} setText={setLastName} />
            <PhoneNumbersInput phoneNumbers={phoneNumbers} setPhoneNumbers={setPhoneNumbers} />
        </form>
    );
};
```

Within the `PhoneNumbersInput` component, the array of phone numbers and the corresponding set state function is destructured into an array of 3-tuples.
To generate a list of html elements, call the `.map()` function on the `destructuredPhoneNumbers`.
Inside the `.map()` function callback, you can further destructure the first argument.
This argument is then the 3-tuple, where the first two elements contain the `state` and `setState` function.
Optionally, the third element can be used to remove the associated entry from the phone numbers array.

```typescript
type DestructuredPhoneNumbers = [phoneNumber, setPhoneNumber, removePhoneNumber][];
```

```tsx
import useDestructuring from "use-destructuring";

type Props = {
    phoneNumbers: string[];
    setPhoneNumbers: React.Dispatch<React.SetStateAction<string[]>>;
}

export const PhoneNumbersInput: React.FC<Props> = ({ phoneNumbers, setPhoneNumbers }) => {

    const destructuredPhoneNumbers = useDestructuring(phoneNumbers, setPhoneNumbers);

    return (
        <div>
            { destructuredPhoneNumbers.map(([phoneNumber, setPhoneNumber, removePhoneNumber], i) => (
                <span key={i}>
                    <TextInput text={phoneNumber} setText={setPhoneNumber} >
                    <button type="button" onClick={() => removePhoneNumber()}>X</button>
                </span>
            )) }
            <button
                type="button"
                onClick={() => setPhoneNumbers(oldPhoneNumbers => [...oldPhoneNumbers, '+01234567'])}
            >
                Add New Telephone Number
            </button>
        </div>
    );
}
```

The only remaining aspect is the capability of adding new values to the phone numbers array.
To add phone numbers, simply use the `setPhoneNumbers` function that is already present.

## Validation

Validation does not require any specialized logic when using `useDestructuring`.
Simply writing the validating logic inside your component using the state.
For example:

```tsx
type Props = {
    phoneNumber: string;
    setPhoneNumber: React.Dispatch<React.SetStateAction<string>>;
}

const PhoneNumberInput: React.FC<Props> = ({ phoneNumber, setPhoneNumber }) => {
    return (
        <label>
            Phone Number:
            <input type="text" value={phoneNumber} onChange={e => setPhoneNumber(e.target.value)} />
            <span className="error">
                { !isValidPhoneNumber(phoneNumber)
                    && 'A phone number must start with a + and have at least 6 digits.' }
            </span>
        </label>
    );
}

const isValidPhoneNumber = (phoneNumber: string): boolean => {
    return /\+[0-9]{6,}/.test(phoneNumber);
}
```

To validate an entire form object, you could combine the individual property validators into one complete form object validator.

## Performance

By default, every time a React component rerenders **ALL** its child components also rerender.
The video below illustrates the React rerending process.
With each rerender, the components color becomes more red.

<video src="./assets/video-01-without-memo.mp4" controls>VIDEO - Without Memo</video>

It is apparent that the entire form rerenders completely each time a key is pressed in any component.
While this is fine most of the time, once a form grows in size and complexity it can become unresponsive and un*react*ive.

To only rerender components with changed props, React components can be wrapped in [React.memo](https://react.dev/reference/react/memo) calls.
However, if functions are defined within React components, they are re-generated each time a component rerenders.
Consequently, this would force child components that receive those functions to rerender, regardless of whether they are wrapped in a memo call or not.
Even if the data passed onto them is not changed.
To address this, the functions generated by `useDestructuring` are all memoized, meaning they do not change from one rerender to the next.
For additional details on how to memoize your own functions, use the [useCallback](https://react.dev/reference/react/useCallback) hook.

In the example below, all components in the component tree are memoized with `React.memo`.
As a consequence, only the leaf component that update state and all its parent components are rerendered.

<video src="./assets/video-02-with-memo.mp4" controls>VIDEO - With Memo</video>

The state and setState generated by `useDestructuring` can also be passed onto multiple child components that will both rerender when either one is changed.
In the video below, multiple distant components rerender because they share same state.

<video src="./assets/video-03-twins.mp4" controls>VIDEO - Twins</video>

Rerender counts can be reduced further.
Forms often contain components that rerender frequently, such as text components that rerender with each key press.
To minimize the rerendering of parents components for each key press, the propagation of the state update can be delayed by a few hundred milliseconds.
This approach ensures, that state update only propagate once the user stops typing.

<video src="./assets/video-04-debounce.mp4" controls>VIDEO - Debounce</video>

You can try it out yourself
[here](https://krassnig.github.io/use-destructuring-performance/)
and view the source code in the
[use-destructuring-performance repository](https://github.com/Krassnig/use-destructuring-performance).

## Contribute

To leave a comment please use the
[krassnig.dev repository issue](https://github.com/Krassnig/krassnig.dev/issues/2).

If you like this library, I would really appreciate a Github star in the
[use-destructuring repository](https://github.com/Krassnig/use-destructuring).

Thanks for reading ^^