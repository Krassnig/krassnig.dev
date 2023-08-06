# You Might Not Need A useEffectAsync Hook

**Or alternatively "How to use async functions in React".**

There is a large number of attempts at handling async programming in React.
Many try to introduce a new hook that is named some version of `useAsync`
which has the big downside of not being statically verifiable by eslint because of the custom dependency array.
In this article I would like to convince to do something much simpler.

## Converting To Async

Instead of creating yet another `useAsync` hook use the `Async` function to convert the effect hook into an async hook.

```tsx
const PersonComponent: React.FC = ({ personId }) => {
    const [name, setName] = useState<string | undefined>(undefined);

    useEffect(() => Async(async signal => {
        await delay(1000, signal);
        const person = await personEndpoint.findById(personId, signal);
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
Since `Async` is just another function called inside `useEffect` eslint verifies missing dependencies like it would for any other function inside `useEffect`.
For example, if the `personId` were to be forgotten inside the `useEffect` dependency array,
eslint would warn you with the message:
`React Hook useEffect has a missing dependency: 'personId'. Either include it or remove the dependency array  react-hooks/exhaustive-deps`

## Converting To Effect

The inverse function to `Async` is `Effect` and is used to convert a callback style function into a promise.

```tsx
const delay = (milliseconds: number, signal: AbortSignal): Promise<void> => {
    return Effect<void>(resolve => {
        const timeoutId = setTimeout(() => resolve(), milliseconds);
        return () => clearTimeout(timeoutId);
    }, signal);
}
```

More on that later in the article.

## Back to Async

### Why the extra `() => ` in front of `Async`

```typescript
// DO NOT do this, theoretically possible
useEffect(Async(async signal => {
    
}), [...])

type TheoreticalAsync = (promise: (...) => Promise): (() => Destructor);
type Destructor = () => void;
```

Although you could encapsulate the outermost lambda as seen in the `TheoreticalAsync` type, eslint would give the following error
`React Hook useEffect received a function whose dependencies are unknown. Pass an inline function instead react-hooks/exhaustive-deps`.
This forces the `Async` function to be implemented in a way that immediately execute a promise
and forces you to add the `() =>` in front of `Async` to allow React to start the effect at any point in time.

```typescript
// do this
useEffect(() => Async(async signal => {
    
}), [...])

type ActualAsync = (promise: (...) => Promise): Destructor;
type Destructor = () => void;
```

## AbortSignal

An important part of asynchronous programming is the capability to cancel async functions at any given moment.
Without it an async function is forced to run to completion and cannot be stopped.
Even without async functions React has a build in way to cancel `useEffect` by letting the user return a cleanup function.
Inside this cleanup function any resource that is created is cancelled and cleaned up.

```tsx
const Timer: React.FC = () => {
    const [secondsPassed, setSecondsPassed] = useState(0);
    const [rerender, setRerender] = useState(0);
    const reset = () => {
        setSecondsPassed(0);
        setRerender(n => n + 1);
    }

    useEffect(() => {
        const intervalId = setInterval(() => setSecondsPassed(n => n + 1), 1000);
        return () => clearInterval(intervalId);
    }), [rerender]);

    return (
        <div>
            <p>{n} {n === 1 ? 'second' : 'seconds'} have passed.</p>
            <button type="button" onClick={() => reset()}>Reset</button>
        </div>
    );
}
```

Similarly the
[`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)
is used to abort and cleanup promises.
This `AbortSignal` is provided to every async function that is called.
The caller who holds the
[`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
can then interrupt the promise by calling `controller.abort(reason: any)`.
Rewriting the `Timer` above should result in the example below:

```typescript
useEffect(() => {
    const controller = new AbortController();
    const signal = abortController.signal;
    const promiseAborted = 'promise aborted';

    const promise = async () => {
        try {
            for (let i = 0; true; i++) {
                setSecondsPassed(i);
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
Otherwise each time a promise is aborted there will be an error printed to the console.
But importantly we still want to rethrow any non-cancellation errors since they contain relevant information about potential errors in our programs.

The example above obviously had quite lot of boilerplate that is going to repeated for every async `useEffect`.
This boilerplate is what `Async` allows you to abstracts away.

```tsx
const Timer: React.FC = () => {
    const [secondsPassed, setSecondsPassed] = useState(0);
    const [rerender, setRerender] = useState(0);
    const reset = () => setRerender(n => n + 1);

    useEffect(() => Async(async (signal: AbortSignal) => {
        for (let i = 0; true; i++) {
            setSecondsPassed(i);
            await delay(1000, signal);
        }
    })), [rerender]);

    return (
        <div>
            <p>{n} {n === 1 ? 'second' : 'seconds'} have passed.</p>
            <button type="button" onClick={() => reset()}>Reset</button>
        </div>
    );
}
```

### How to implement abortable Promises

At the promise side of things we need to use the `AbortSignal` to properly abort the promise.
When `controller.abort(reason: any)` is called the associated `controller.signal` is aborted
and the `reason` provided should (theoretically) be thrown from the inside async function
that has been given the corresponding signal as a parameter.

```typescript
const myAsyncFunction = async (signal: AbortSignal): Promise<any> => {
    for (let i = 0; i <= 20; i++) {
        if (signal.aborted) {
            throw signal.reason; // <-- abort with signal.reason
        }

        // allows the browser to rerender
        await new Promise(resolve => setTimeout(() => resolve(), 0));

        // split up synchronous work
        doExpensiveWork(i);
    }
}
```

Inside the async function you can either test whether the signal has been aborted or subscribe to the `abort` event depending on your use case.
The example below implements a delay function by subscribing to the `'abort'` event.

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

## Support for AbortSignal

Abort Signals are supported by both
[fetch](https://developer.mozilla.org/en-US/docs/Web/API/Request/signal)
and
[axios](https://axios-http.com/docs/cancellation).
Since
[XHR](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/abort)
is not async it does not support `AbortSignal`s but it has an `abort` method to achieve the same effect.

Sadly both `fetch` and `axios` throw custom errors when cancelling http requests.
There is however a work-around.

### How to handle errors with AbortSignal

Cancellation errors thrown inside async function should not be caught in general.
Instead they should be rethrown until they reach the orignal caller of the async function.

However, checking for a specific error is difficult if you neither have controll over the error thrown by the callee nor the error requested by the caller.
To work around that limitation the signal can be checked to see if it has already been aborted.
Through this you can infer that the error is *almost* centrainly a cancellation error without checking for a specific value.

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

## Reacts Documentation to Fechting Data

Although Reacts documentation concerning
[Fetching Data](https://react.dev/learn/you-might-not-need-an-effect#fetching-data)
is correct in the sense that race condition are created if you do not clean up your `useEffect`.
It is incorrect in that if you were to properly cleanup requests with `AbortSignal`s there would be no race condition in the first place.

Reacts solution is to include a boolean "signal" which lets you detect whether a request that has been made is still relevant or should be corsidered "stale" and therefore be ignored.

This example is taken from the React docs:

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

The `ignore` variable also cannot be used inside other async functions. If you were to pass `ignore` onto `fetchResults` it would not change its value inside `fetchResults` once the cleanup function is executed.

In the general case, it also seem absurd to let requests run to conclusion even though they are no longer needed.
If the user still wanted to keep those requests running such behavior could still be achieved by using an alternative `AbortSignal` which only cleans up once the component unmounts.

[Still love you Dan ;)](https://github.com/TodePond/C/blob/v0.9.9.9.9.9.9.9.9d/wallpaper_dont_upload.png)

## Effect

The inverse function to `Async` is `Effect` and is used to convert a callback style function into a promise.

```typescript
useEffect(() => Async(async signal => {
    await Effect(resolve => {
        const timeoutId = setTimeout(() => resolve(), 1000);
        return () => clearTimeout(timeoutId);
    }, signal);
    const response = await endpoint.findById(..., signal);
}), [...]);
```

For example awaiting a timeout can be written in a callback-style-function but the remainder of the `useEffect` is still written in an async style.

```typescript
const delay = (milliseconds: number, signal: AbortSignal): Promise<void> => {
    return Effect<void>(() => {
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

`Effect` can also `resolve` values or throw exceptions via `reject`.

```typescript
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
                    reject('status: ' + status + ' message: ' + request.responseText);
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

    await Effect(resolve => () => resolve(), signal);

    console.log('execute code after cleanup here.');
}). [...])
```

## Nesting

If you are unsure which paradigm to use after reading this article feel free to express that uncertainty by switching between both paradigms multiple times!

```tsx
useEffect(
    () => Async(
        async signal => await Effect(
            () => Async(
                async signal => await Effect(
                    () => Async(
                        async signal => await Effect(
                            () => Async(
                                async signal => { console.log('Hello World!'); }
                            ), signal
                        )
                    ), signal
                )
            ), signal
        )
    )
);
```