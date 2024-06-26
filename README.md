# TypeScript Signals

A simple implementation of Signals in TypeScript.

This repository is purely for fun and educational purposes.
It is not intended to be used in production code.

## Live Demo

[![Netlify Status](https://api.netlify.com/api/v1/badges/a782e12b-6f39-48f2-b8c0-ced9cd483866/deploy-status)](https://ts-signals.netlify.app)

[ts-signals.netlify.app](https://ts-signals.netlify.app)

## What are Signals?

Signals are a way to manage state in a reactive way. They are similar to observables, but with a simpler API.

You can learn more about them here:

-   [SolidJS - Introduction/Signals](https://www.solidjs.com/tutorial/introduction_signals)

-   [YouTube - Ryan K Carniato - Revolutionary Signals](https://www.youtube.com/watch?v=Jp7QBjY5K34)

-   [TC39/proposal-signals](https://github.com/tc39/proposal-signals)

## API

The Signals API is lean and contains only 2 building blocks:

-   `createSignal(initialValue: T): [get, set]` - Creates a Signal object and returns a getter and a setter.

-   `createEffect(cb: () => void)` - Creates an effect that runs the callback function _automagically™_ whenever the Signals that are used inside the callback change.

## Usage

Let's see a simple example of a counter signal that logs the count whenever it changes:

```typescript
import { createSignal, createEffect } from './signals';

const [count, setCount] = createSignal(0);

const increment = () => setCount(count() + 1);

createEffect(() => {
	console.log('Count:', count());
});

increment(); // Count: 1
increment(); // Count: 2

// ...
```

We create a Signal with default value of `0`, and add an effect that logs the count whenever it changes.
The effect automatically infers its dependencies, and runs whenever the `count` Signal changes.

Magic! 🎩✨

You can also use Signals to create derived state:

```typescript
import { createSignal, createEffect } from './signals';

const [count, setCount] = createSignal(0);
const doubled = () => count() * 2;

createEffect(() => {
	console.log('Doubled:', doubled());
});

setCount(1); // Doubled: 2
setCount(2); // Doubled: 4
```

A derived Signal is just a function that returns a value based on other Signals. It has to be a function, so that it can be re-evaluated whenever its dependencies change.

If the derived state calculation is expensive, you can use `createMemo` to memoize the result:

```typescript
import { createSignal, createEffect, createMemo } from './signals';

const [count, setCount] = createSignal(0);
const doubled = createMemo(() => {
	console.log('Calculating');

	return count() * 2;
});

createEffect(() => {
	console.log('Doubled:', doubled());
});

createEffect(() => {
	console.log('Again:', doubled());
});

setCount(1); // Logs: "Calculating", "Doubled: 2", "Again: 2"
setCount(2); // Logs: "Calculating", "Doubled: 4", "Again: 4"
```

The `doubled` value calculation will run only once per change of `count`, even if multiple effects depend on it.

## How it Works Under the Hood?

A "Signal" is just an object that holds a value and gives you the option to read and write to it. Roughly something like this:

```typescript
const signal = {
	value: 0,

	get() {
		return this.value;
	},

	set(newValue) {
		this.value = newValue;
	},
};
```

The actual magic happens when combining the `createSignal` and `createEffect` functions.

We utilize JavaScript's call stack to keep track of the dependencies of each effect.
Whenever a Signal is read inside an effect, we add the effect to the Signal's list of subscribers, and whenever the Signal is written to, we notify all the subscribers to run their effects:

```typescript
let currentEffect = null;

function createEffect(cb) {
	currentEffect = cb;

	// This will trigger the `get` method of any Signal that's
	// being accessed inside the effect, which in turn,
	// add the effect to the Signal's subscribers list.
	cb();

	currentEffect = null;
}

function createSignal() {
	const signal = {
		// ...

		subscribers: new Set(),

		get() {
			this.subscribers.add(currentEffect);

			return this.value;
		},

		set(newValue) {
			this.value = newValue;

			this.subscribers.forEach((effect) => effect());
		},
	};

	// ...
}
```

Simple yet powerful! 🚀
