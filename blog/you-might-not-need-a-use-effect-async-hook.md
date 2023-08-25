# You Might Not Need A useEffectAsync Hook

There are numerous attempts at handling async programming in React.
Many of those attempts introduce a new hook named something along the lines of `useAsync`.
However, creating a custom hook with dependencies come with a significant drawback:
the dependency array can no longer be statically verified by ESLint.

This article describes how to overcome that problem and demonstrate how straightforward using async programming can be in React.

The source code for the code used in this article can be found in this
[Github repository](https://github.com/Krassnig/no-async-hook) and
[NPM package](https://www.npmjs.com/package/no-async-hook).

## Async

The `Async` function enables you to write a typical async functions.
That function is passed to `Async` and then transformed into a function that can be used by `useEffect`.

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

Wihtin the async function, you can perform any async operations as you normally would.
And since `Async` is essentially just another function called inside `useEffect`,
ESLint verifies missing dependencies the same as it would for any other function inside `useEffect`.
For example, if the `personId` were to be forgotten in the `useEffect` dependency array,
ESLint would warn you with the message:
`React Hook useEffect has a missing dependency: 'personId'. Either include it or remove the dependency array  react-hooks/exhaustive-deps`.

## Effect

The inverse function of `Async` is `Effect` and is used to convert callback style functions into promises.
If you are already familiar with `useEffect`, you can easily create an async/await function from pre-existing non-async functions.

```tsx
import { Effect } from "no-async-hook";

const delay = (milliseconds: number, signal: AbortSignal): Promise<void> => {
    return Effect<void>(resolve => {
        const timeoutId = setTimeout(() => resolve(), milliseconds);
        return () => clearTimeout(timeoutId);
    }, signal);
}
```

`Effect` is explained in more detail in the [Effect Section](#effect) of this article.

## Back To Async

### Why The Extra `() => ` In Front Of `Async`

```typescript
// DO NOT do this, could be a possible implementation
useEffect(Async(async signal => {
    
}), [...]);

type TheoreticalAsync = (promise: (...) => Promise<void>) => (() => Destructor);
type Destructor = () => void;
```

Although the outermost lambda could be encapsulated, ESLint would give the following error
`React Hook useEffect received a function whose dependencies are unknown. Pass an inline function instead react-hooks/exhaustive-deps`.
Since ESLint does not verify that version of `Async`, it must be implemented in a way that immediately execute the promise.
ESLint probably enforces this rule because a developer could match the `useEffect` signature while inadvertently breaking its intended behavior.

```typescript
// do this
useEffect(() => Async(async signal => {
    
}), [...])

type ActualAsync = (promise: (...) => Promise<void>) => Destructor;
type Destructor = () => void;
```

## AbortSignal

An important aspect of asynchronous programming is the ability to cancel async functions at any given moment.
Without this capability, an async function is forced to run to completion and cannot be stopped.
Even without async functions, React offers a build in way to cancel effects by allowing you to return a cleanup function.
Inside this cleanup function any resource that is created is cancelled and cleaned up.
For example, the `Timer` component below creates an effect with `setInteval` and removes the effect with `clearInterval`.
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

Likewise, when using `Async` the
[`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)
is used to abort and cleanup promises.
This `AbortSignal` is supplied to each invoked async function.
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

To successfully interrupt a promise the error should be caught in the function calling the promise.
Failing to do so, would result in a print to the console for each aborted promise.
Yet we still want to rethrow any non-cancellation errors, as they contain relevant information about potential errors in our programs.

The example above obviously has quite a lot of boilerplate that would be repeated for every async `useEffect`.
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

By using `Async` in the `Timer` component we only see the async code we really care about:
the for loop that updates the seconds.

### How To Implement Abortable Promises

Inside promises we need to use the `AbortSignal` to effectively abort promises.
When `controller.abort(reason: any)` is called the associated `controller.signal` is aborted
Ideally, the `reason` should be thrown from the inside async function.

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

In async functions, you either have the option to test whether the signal has been aborted or subscribe to the `abort` event.
If feasible, prefer the latter.
The example below demostrates how a promise could be implemented by subscribing to an `'abort'` event.
However, as you can see it is quite complicated and easy to get wrong.

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

As demonstrated earlier, the `Effect` function significantly simplifies the implentation of such Promises.

### Support For AbortSignal

Abort Signals are supported by both
[fetch](https://developer.mozilla.org/en-US/docs/Web/API/Request/signal)
and
[axios](https://axios-http.com/docs/cancellation).
However, since
[XHR](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/abort)
is not implemented through promises, it does not support `AbortSignal`s.
Instead, it offers an `abort` method to achieve the same effect.


### How To Handle Errors With AbortSignal

Cancellation errors thrown inside async function should not be caught.
Rather, they should be rethrown until they reach the initial caller of the async function.

Unfortunately, both `fetch` and `axios` throw custom errors when HTTP requests are cancelled.
However, targeting a specific error reduces the overall generality of your async functions.
Thankfully, there is a work around even when you have no controll over the error thrown by the callee and the error provided by the caller.

Within each `catch` block, check whether the signal has already been aborted.
Through this approach you can infer that the error is *almost* centrainly a cancellation error.

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
contains a section dedicated to asynchronous programming called
[Fetching Data](https://react.dev/learn/you-might-not-need-an-effect#fetching-data).
In this section it is suggested, that a boolean value can be used to track whether a requests response is considered "stale" or if it should still call the `setState` function.

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

However, using this approach has a few downsides.
The `ignore` variable can only be used inside the `useEffect` lambda.
Should you attempt to pass the `ignore` variable onto `fetchResults`, its value would remain unaffected inside `fetchResults`.
Also, as this solution does not utilize `AbortSignal`s, request have to run to completion, wasting resources in the process.
Lastly, as the complexity of the effect grows, it becomes harder to implement correctly and consequently also harder to read.

[Still love you Dan ;)](https://github.com/TodePond/C/blob/v0.9.9.9.9.9.9.9.9d/wallpaper_dont_upload.png)

## Effect

The inverse function of `Async` is `Effect` and is used to convert a callback style functions into promises.
It has almost the exact same signature as `useEffect`. Consequently, implementing an `Effect` should not need an explanation.

```typescript
useEffect(() => Async(async signal => {
    await Effect<void>(resolve => {
        const timeoutId = setTimeout(() => resolve(), 1000);
        return () => clearTimeout(timeoutId);
    }, signal);
    const response = await endpoint.findById(..., signal);
}), [...]);
```

The main differences between `useEffect` and `Effect` are that instead of passing a dependency array it accepts an `AbortSignal` and it provides a `resolve` function to fulfill the returned promise.
Additionally, you can improve the readability of your code by wrapping `Effect` calls within async functions.

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

`Effect` can also be used for awaiting values and rejecting promises.
Here is an example that converts an `XMLHttpRequest` into a promise and rejects that promise should the HTTP request err.

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

`Effect` can even be used to execute code once the `useEffect` cleaned up has been triggered.
You can achieve that by simply resolving the `Effect` inside its cleanup function.

```typescript
useEffect(() => Async(async signal => {
    console.log('applying the effect.');

    await Effect<void>(resolve => () => resolve(), signal);

    console.log('execute code after cleanup here.');
}). [...])
```

## Why The Uppercase Naming?

Honestly, I am not too sure myself.
I am well aware that I am deviating from Javascript's convention, but it just kind of has a nice aestethic.

If I were to make up a justification:
In React, hooks are usually named `use` followed by a noun.
Now, what if we to strip away the aspect that makes it a hook?
You would end up with just a noun in uppercase.
Ultimately, if you feel like this rebellious convention breaking by me is too much, feel free to just copy the code from the
[Github repository](https://github.com/Krassnig/no-async-hook/blob/main/no-async-hook.ts)
and modify it for your own needs 😉.

## Nesting

If you are unsure which style of function to use after reading this article,
feel free to express that uncertainty by switching between both styles multiple times!

```tsx
useEffect(() => Async(
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
), []);
```

## Contribute

To leave a comment please use the
[krassnig.dev repository issue](https://github.com/Krassnig/krassnig.dev/issues/1).
If you like this library, I would really appreciate a github star in the
[no-async-hook repository](https://github.com/Krassnig/no-async-hook).

Thanks for reading ^^