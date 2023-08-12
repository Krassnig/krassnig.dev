# You Might Not Need A useEffectAsync Hook

### Or Alternatively: How To Use Async Functions In React

There is numerous attempts at handling async programming in React.
Many of those attempts introduce a new hook that is named some version of `useAsync`,
but creating a custom hook with dependencies has one big downside.
The depency array is no longer statically verifiable by eslint.

This article describes how to work around that problem and how simple using async can be in React.

The source code for the library used in this article can be found in this
[Github repository](https://github.com/Krassnig/no-async-hook) and
[NPM package](https://www.npmjs.com/package/no-async-hook).

## Converting To Async

The `Async` function allows you to write a normal async function and converts that async function into a function that `useEffect` can control.

```tsx
import { Async } from "no-async-hook";
import { useEffect, useState } from "react";

const PersonComponent: React.FC = ({ personId }) => {
    const [name, setName] = useState<string | undefined>(undefined);

    useEffect(() => Async(async signal => {
        await delay(1000, signal);
        const person = await findPersonById(personId, signal);
        const fullName = person.firstName + ' ' + person.lastName;
        setName(fullName);
    }), [personId]);

    return name === undefined ? (
        <p>Loading person...</p>
    ) : (
        <p>The person is named: { name }</p>
    );
}
```

Inside the async function you can do async operations as you normally would.
Since `Async` is just another function called inside `useEffect`,
eslint verifies missing dependencies like it would for any other function inside `useEffect`.
For example, if the `personId` were to be forgotten inside the `useEffect` dependency array,
eslint would warn you with the message:
`React Hook useEffect has a missing dependency: 'personId'. Either include it or remove the dependency array  react-hooks/exhaustive-deps`.

## Converting To Effect

The inverse function of `Async` is `Effect` and is used to convert callback style functions into promises.
If you know how to implement a timer with `useEffect` already, you can easily implement an async/await implementation through the `Effect` function.

```tsx
import { Effect } from "no-async-hook";

const delay = (milliseconds: number, signal: AbortSignal): Promise<void> => {
    return Effect<void>(resolve => {
        const timeoutId = setTimeout(() => resolve(), milliseconds);
        return () => clearTimeout(timeoutId);
    }, signal);
}
```

You can find more on `Effect` in the [Effect Section](#effect) of this article.

## Back To Async

### Why The Extra `() => ` In Front Of `Async`

```typescript
// DO NOT do this, theoretically possible
useEffect(Async(async signal => {
    
}), [...]);

type TheoreticalAsync = (promise: (...) => Promise<void>) => (() => Destructor);
type Destructor = () => void;
```

Although you could encapsulate the outermost lambda as well,
eslint would give the following error
`React Hook useEffect received a function whose dependencies are unknown. Pass an inline function instead react-hooks/exhaustive-deps`.
Since eslint does not verify this function,
`Async` has to be implemented in a way that immediately execute the promise.
Eslint probably enforces this rule because a developer might match the signature of `useEffect`,
yet break the expected behavior of `useEffect`.

```typescript
// do this
useEffect(() => Async(async signal => {
    
}), [...])

type ActualAsync = (promise: (...) => Promise<void>) => Destructor;
type Destructor = () => void;
```

## AbortSignal

An important aspect of asynchronous programming is the capability to cancel async functions at any given moment.
Without it, an async function is forced to run to completion and cannot be stopped.
Even without async functions React has a build in way to cancel effects by letting the user return a cleanup function.
Inside this cleanup function any resource that is created is cancelled and cleaned up.
For example, the `Timer` component creates an effect with `setInteval` and removes the effect with `clearInterval`.
By providing React with a callback to `clearInterval` the effect can be cancelled at any moment.

```tsx
import { useEffect, useState } from "react";

const Timer: React.FC = () => {
    const [seconds, setSeconds] = useState(0);
    const [rerender, setRerender] = useState(0);
    const reset = () => {
        setSeconds(0);
        setRerender(n => n + 1);
    }

    useEffect(() => {
        const intervalId = setInterval(() => setSeconds(n => n + 1), 1000);
        return () => clearInterval(intervalId);
    }, [rerender]);

    return (
        <div>
            <p>{seconds} {seconds === 1 ? 'second' : 'seconds'} have passed.</p>
            <button type="button" onClick={() => reset()}>Reset</button>
        </div>
    );
}
```

Similarly, when using `Async` the
[`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)
is used to abort and cleanup promises.
This `AbortSignal` is provided to every async function that is called.
The caller who holds the
[`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
can then interrupt the promise by calling `controller.abort(reason: any)`.
Rewriting the `Timer` using `AbortSignal` with correct error handling would result in the example below.

```typescript
useEffect(() => {
    const controller = new AbortController();
    const signal = controller.signal;
    const promiseAborted = 'promise aborted';

    const promise = async () => {
        try {
            for (let i = 0; true; i++) {
                setSeconds(i);
                await delay(1000, signal);
            }
        }
        catch (error) {
            if (error === promiseAborted) {
                // promise was aborted
                return;
            }
            else {
                // not promise related
                throw error;
            }
        }
    };

    promise();

    return () => controller.abort(promiseAborted);
}, [rerender]);
```

To successfully interrupt a promise the error should be caught at the top level function.
Otherwise, each time a promise is aborted an error is printed to the console.
Yet we still want to rethrow any non-cancellation errors since they contain relevant information about potential errors in our programs.

The example above obviously has quite lot of boilerplate that would be repeated for every async `useEffect`.
This boilerplate is what `Async` allows you to abstracts.

```tsx
import { Async } from "no-async-hook";
import { useEffect, useState } from "react";

const Timer: React.FC = () => {
    const [seconds, setSeconds] = useState(0);
    const [rerender, setRerender] = useState(0);
    const reset = () => setRerender(n => n + 1);

    useEffect(() => Async(async (signal: AbortSignal) => {
        for (let i = 0; true; i++) {
            setSeconds(i);
            await delay(1000, signal);
        }
    }), [rerender]);

    return (
        <div>
            <p>{seconds} {seconds === 1 ? 'second' : 'seconds'} have passed.</p>
            <button type="button" onClick={() => reset()}>Reset</button>
        </div>
    );
}
```

By using `Async` in the `Timer` component we only see the async code we care about,
which is the for loop which updates the seconds that have passed.

### How To Implement Abortable Promises

Inside promises we need to use the `AbortSignal` to properly abort promises.
When `controller.abort(reason: any)` is called the associated `controller.signal` is aborted
and the `reason` provided should (ideally) be thrown from the inside async function.

```typescript
const myAsyncFunction = async (signal: AbortSignal): Promise<any> => {
    for (let i = 0; i <= 20; i++) {
        if (signal.aborted) {
            throw signal.reason; // <-- abort with signal.reason
        }

        // allows the browser to rerender
        await new Promise<void>(resolve => setTimeout(() => resolve(), 0));

        // split up synchronous work
        doExpensiveWork(i);
    }
}
```

Inside the async function you can either test whether the signal has been aborted or subscribe to the `abort` event depending on your use case.
You should prefer the latter if possible.
The example below is implemented by subscribing to the `'abort'` event, but it is quite complicated.

```typescript
export const delay = (milliSeconds: number, signal: AbortSignal): Promise<void> => {
    return new Promise((resolve, reject) => {
        const cleanUp = () => {
            signal.removeEventListener('abort', onAbort);
            clearTimeout(timeoutId);
        }

        const onAbort = () => {
            cleanUp();
            reject(signal.reason); // <-- reject with signal.reason
        };

        signal.addEventListener('abort', onAbort);

        const timeoutId = setTimeout(() => {
            cleanUp();
            resolve();
        }, milliSeconds);
    });
}
```

As you have already seen in the [Introduction](#converting-to-effect) of this article there is an easier way of implementing this function using the `Effect` function instead.

### Support For AbortSignal

Abort Signals are supported by both
[fetch](https://developer.mozilla.org/en-US/docs/Web/API/Request/signal)
and
[axios](https://axios-http.com/docs/cancellation).
Since
[XHR](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/abort)
is not async it does not support `AbortSignal`s but it has an `abort` method to achieve the same effect.

Sadly, both `fetch` and `axios` throw custom errors when cancelling http requests.
However, there is a work-around.

### How To Handle Errors With AbortSignal

Cancellation errors thrown inside async function should not be caught in general.
Instead, they should be rethrown until they reach the orignal caller of the async function.

However, checking for a specific error is difficult if you neither have controll over the error thrown by the callee nor the error provided by the caller.
To work around that limitation the signal can be checked to see if it has already been aborted.
Through this you can infer that the error is *almost* centrainly a cancellation error without having to check for a specific value.

```typescript
const myAsyncFunction = async (signal: AbortSignal): Promise<any> => {
    try {
        return await anotherAsyncFunction(signal);
    }
    catch (error) {
        if (signal.aborted) { // DO NOT catch cancellations!
            throw error; // The call stack continues to collapse
        }
        else {
            // handle error in some way
            console.error(error);
        }
    }
}
```

## Reacts Documentation For Fechting Data

The React documentation on
[You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) 
contains a section on how to handle asynchronous calls inside the
[Fetching Data](https://react.dev/learn/you-might-not-need-an-effect#fetching-data)
section.
In it they recommend setting a boolean value to track whether a request is considered "stale" or if it should call the `setState` function.

The following example is given:

```typescript
useEffect(() => {
    let ignore = false;
    fetchResults(query, page).then(json => {
        if (!ignore) {
            setResults(json);
        }
    });
    return () => {
        ignore = true;
    };
}, [query, page]);
```

Using this approach has a few downsides.
The `ignore` variable can only be used inside the `useEffect` lambda.
If you tried to pass `ignore` onto `fetchResults` it would not change its value inside `fetchResults` once the cleanup function is executed.
Since this solution does not use `AbortSignal`s request also has to run to completion.
And lastly, once the effect grows in complexity it will become very hard to read.

[Still love you Dan ;)](https://github.com/TodePond/C/blob/v0.9.9.9.9.9.9.9.9d/wallpaper_dont_upload.png)

## Effect

The inverse function of `Async` is `Effect` and is used to convert a callback style function into a promise.
Inside `Effect` you can write code just like inside a `useEffect` lambda.

```typescript
useEffect(() => Async(async signal => {
    await Effect<void>(resolve => {
        const timeoutId = setTimeout(() => resolve(), 1000);
        return () => clearTimeout(timeoutId);
    }, signal);
    const response = await endpoint.findById(..., signal);
}), [...]);
```

The main difference between `useEffect` and `Effect` is that instead of passing a dependency array to `Effect` it accepts an `AbortSignal` and provides a `resolve` function to fulfill the promise.
With `Effect` any callback style function can be converted into a promise easily.
Furthermore, these promises can be wrapped inside async functions to simplify async `useEffect` even more.

```typescript
const delay = (milliseconds: number, signal: AbortSignal): Promise<void> => {
    return Effect<void>(resolve => {
        const timeoutId = setTimeout(() => resolve(), milliseconds);
        return () => clearTimeout(timeoutId);
    }, signal);
}

useEffect(() => Async(async signal => {
    await delay(1000, signal);
    const response = await endpoint.findById(..., signal);
}), [...]);
```

### Awaiting Values

`Effect` can also await values and `reject` promises.
Here is an example that converts an `XMLHttpRequest` into a promise.

```typescript
import { Effect } from "no-async-hook";

const xhrGet = (url: string, signal: AbortSignal): Promise<string> => {
    return Effect<string>((resolve, reject) => {
        const request = new XMLHttpRequest();

        request.onreadystatechange = function() {
            if (request.readyState === XMLHttpRequest.DONE) {
                const status = request.status;
                if (200 <= status && status < 400) {
                    resolve(request.responseText);
                }
                else {
                    reject(`status: ${status} message: ${request.responseText}`);
                }
            }
        }

        request.open('GET', url);
        request.send(null);

        return () => request.abort();
    }, signal);
}
```

### Awaiting Cleanup

`Effect` can be used to execute code after a `useEffect` is cleaned up.
Simply resolve the `Effect` inside the `Effect`s cleanup function.

```typescript
useEffect(() => Async(async signal => {
    console.log('applying the effect.');

    await Effect<void>(resolve => () => resolve(), signal);

    console.log('execute code after cleanup here.');
}). [...])
```

## Why The Uppercase Naming?

Honestly, I am not too sure myself.
I am well aware that I am breaking Javascripts convention, but it just kind of feels and looks nice.

If I had to make up a reason:
Hooks in React are usually named `use` plus a noun, so what happens if you remove the hook part? You end up with just a noun in uppercase.
Ultimately, if you feel like this rebellious convention breaking by me is too much, feel free to just copy the code from the
[Github repository](https://github.com/Krassnig/no-async-hook)
and modify it for you own needs 😉.

## Nesting

If you are unsure which paradigm to use after reading this article,
feel free to express that uncertainty by switching between both paradigms multiple times!

```tsx
useEffect(
    () => Async(
        async signal => await Effect(
            () => Async(
                async signal => await Effect(
                    () => Async(
                        async signal => await Effect(
                            () => Async(
                                async () => { console.log('Hello World!'); }
                            ), signal
                        )
                    ), signal
                )
            ), signal
        )
    )
);
```

## Contribute

To leave a comment please use the
[krassnig.dev repository issue](https://github.com/Krassnig/krassnig.dev/issues/1).
If you like this library I would really appreciate a github star in the
[no-async-hook repository](https://github.com/Krassnig/no-async-hook).

Thanks for reading ^^