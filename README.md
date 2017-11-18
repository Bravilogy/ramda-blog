# Functional programming with Ramda
* Immutable.
* Stateless.
* Declarative.
* Pure.
* Compositions.
* Functions.
 
Just a few things that come to one's mind when thinking about functional programming.

In this post I'd like to delve a bit deeper into the functional world and specifically,
explore the awesomeness of currying and function compositions.
 
## Currying
What is currying? It is a way to partially apply arguments to a function.
Let's start with the basics:
 
 ```js
const add = (a, b) => a + b;
add(2, 3); // => 5
 ```
Here, our `add` function accepts two arguments and for it to actually do something,
it needs both arguments to be supplied at the same time. What if we wanted to apply just one argument
and pass another later on in the application? Well, we can do this by returning a closure function like so:
```js
const add = a => b => a + b;
const addTwo = add(2);
```
and later on...
```js
addTwo(3); // => 5
```
But now we've lost the original functionality to pass both arguments at the same time. And also, if we wanted to add more arguments,
then we would end up with something like this:
```js
const add = a => b => c => d => a + b + c + d;
```
Also, what if we wanted to pass just 1 argument on a first call and then the rest (3 in this case), afterwards? It's impossible to do so as it is now.

This is where `currying` comes in. So in theory, we want to create a function that accepts a function (our `add` in this case),
then remembers the number of arguments it expects and returns a function, that when called with `n` number arguments,
will check if the number of arguments is `>=` the arguments that we're expecting and if `true`, then it will execute that original function
with these arguments, otherwise it will return another function, that will accept some other arguments and when called,
it will `concat` the top arguments with these `other` arguments and loop back to this whole functionality again.
A bit confusing? Let's look at some code:
```js
const curry = fn => {
  // here we save the number of arguments we need
  const totalArgsRequired = fn.length;

  // then we return our partial function that accepts n arguments
  return function basePartial (...args) {
    // and when called, it checks if our
    // original function is satisfied
    if (args.length >= totalArgsRequired) {
      // if so, then we call it and pass these args
      return fn.apply(null, args);
    }
    // otherwise, we return a function, that
    // accepts some otherArguments
    return function partial (...otherArgs) {
      // and when called, we loop back to the same thing,
      // passing all the arguments that we have so far
      return basePartial.apply(null, args.concat(otherArgs));
    };
  };
};
```
 
For a better understanding, I have named the inner functions. Let's break this down by the following example:
```js
const add = curry((a, b, c) => a + b + c); // => here we get a "basePartial" function
const firstLoop = add(2); // => here, firstLoop is a "partial" function
const secondLoop = firstLoop(3); // => "partial" again
const thirdLoop = secondLoop(4); // => the "if" statement is fulfilled and we get the result - 9.
```
 
Now, what good is this? Well, we'll get there, but let's look at some partial function applications we can start implementing already,
just like these two generic functions here:
```js
const prop = curry((property, obj) => obj[property]);
const filter = curry((predicate, arr) => arr.filter(predicate));
```
And then let's see a few use cases:
```js
const users = [
  { id: 1, name: 'Ed', online: true },
  { id: 2, name: 'Edd', online: true },
  { id: 3, name: 'Eddy', online: false },
];

const getName = prop('name');
users.map(getName); // => ['Ed', 'Edd', 'Eddy']
 
const getOnlineUsers = filter(prop('online'));
getOnlineUsers(users); // => [ { id: 1, name: 'Ed', online: true }, { id: 2, name: 'Edd', online: true } ]
```
## Point-free
And this has seemlessly led us to another concept that comes with currying (free of charge) - point-free programming.

Notice how our `getName` or `getOnlineUsers` functions don't really say anything about `data` they need? They just describe what they're doing
in a very descriptive, generic way. Thanks to currying we can completely get rid of the `data` part. But to achieve this, we strategically
placed `obj` and `arr` as the last arguments accepted by our generic functions above. What if we had done the other way round?
```js
const prop = curry((obj, property) => obj[property]);
const filter = curry((arr, predicate) => arr.filter(predicate));
 
const getName = user => prop(user, 'name');
const getOnlineUsers = users => filter(users, user => prop(user, 'online'));
```
What a mess. We would just be creating some very unnecessary and useless wrapper functions.

_But if I don't specify the *data* in my arguments list, I might not understand what a function expects._

Good point, dear reader. And that is exactly why most functional programming languages have a robust type systems.

But let's not clutter our minds with extras for now.
 
Now that we've seen currying in action, let's continue with the idea of point-free programming here and implement a `map` too.
```js
const map = curry((fn, data) => data.map(fn));
const prop = curry((property, obj) => obj[property]);
const filter = curry((predicate, arr) => arr.filter(predicate));
 
const getNames = map(prop('name'));
const getOnlineUsers = filter(prop('online'));
```
And to get the names of just online users, we could:
```js
const getJustOnlineUsersNames = users =>
  getNames(getOnlineUsers(users));

getJustOnlineUsersNames(users); // => ['Ed', 'Edd']
``` 
But now we've lost the point-free programming badge, because we explicitly specified what `data` our function expects and we're no longer
invited to the functional programming party. Shame.
 
Well, this brings us to the next topic - function compositions.
## Function compositions
Wouldn't it be awesome to keep the `data` out of the way all the way through and keep creating these generic, descriptive functions?

Let's sit back, relax and find an inner functional composer in us and create a little helper function that will accept some `n` number
of functions and return a new function, that will wait for some `data`. Once called with data (`n` number of arguments), it will 
reduce on functions supplied, passing the result of one function as an argument to another, starting off with that initial data. 
So to sketch out what we're trying to achieve, is this:
```js
compose(getNames, getOnlineUsers);
```
_Umm.. why is `getOnlineUsers` function placed as a second argument here? Shouldn't it be run as a first step in our composition?_

Yes, of course, but we're just following the same thing we've always done, in our daily coding - executing functions from right-to-left:
```js
getNames(getOnlineUsers(users)); // => the rightmost function is run first, going to the left direction
```
Let's implement our compose function here:
```js
const compose = (fn, ...fns) => (...args) =>
  fns.reduce((result, x) => x(result), fn(...args));
```
But we're not done yet, because our `reduce` method will start with the very first function and do left-to-right computation as it is now.
 
However, before we fix that and make it right-to-left, let's look at the the reason why we're breaking down our first batch of 
arguments into `fn` and `fns`. As we know, `reduce` method accepts only 1 argument as an initial data. But we need to be able to 
pass multiple arguments to our composed function, so `fn` will produce the initial data for our reducer here.
 
For example, if we do this (left-to-right computation):
 ```js
const doMaths = compose(Math.max, Math.round);
doMaths(1, 2, 0.3, 3.4);
```
Our `reduce` method would be confused taking `1, 2, 0.3, 3.4` as initial arguments. So we let `Math.max` produce that initial data first.

### dilemma
Do we keep the left-to-right computation or switch to right-to-left instead?

### solution
Really, it comes down to whatever you find more readable, so why not have both versions?
```js
const pipe = (fn, ...fns) => (...args) =>
  fns.reduce((result, x) => x(result), fn(...args)); // => left-to-right computation
 
const compose = (...fns) =>
  pipe.apply(null, fns.reverse()); // => right-to-lefty
```
And now going back to our initial issue, here is the full version:
```js
const curry = fn => {
  const totalArgsRequired = fn.length;

  return function basePartial (...args) {
    if (args.length >= totalArgsRequired) {
      return fn.apply(null, args);
    }
    return function partial (...otherArgs) {
      return basePartial.apply(null, args.concat(otherArgs));
    };
  };
};

const map = curry((fn, data) => data.map(fn));
const prop = curry((property, obj) => obj[property]);
const filter = curry((predicate, arr) => arr.filter(predicate));

const pipe = (fn, ...fns) => (...args) =>
  fns.reduce((result, x) => x(result), fn(...args));
 
const compose = (...fns) =>
  pipe.apply(null, fns.reverse());
 
const getNames = map(prop('name'));
const getOnlineUsers = filter(prop('online'));

const getOnlineUsersNames = compose(getNames, getOnlineUsers);

const users = [
  { id: 1, name: 'Ed', online: true },
  { id: 2, name: 'Edd', online: true },
  { id: 3, name: 'Eddy', online: false },
];

getOnlineUsersNames(users); // => ['Ed', 'Edd']
```
 
But now, 90% of our code is just boilerplate. And this is where [ramda](http://ramdajs.com) comes in. In ramda, every single function is 
automatically curried. So if we were to re-implement this using ramda, we would simply do:
 
```js
import { pipe, prop, map, filter } from 'ramda';

const getOnlineUsersNames = pipe(
  filter(prop('online')),
  map(prop('name')),
);

const users = [
  { id: 1, name: 'Ed', online: true },
  { id: 2, name: 'Edd', online: true },
  { id: 3, name: 'Eddy', online: false },
];

getOnlineUsersNames(users); // => ['Ed', 'Edd']
```
 
Ramda offers tons of little utility functions that can be used to write very descriptive, point-free code.

## Ramda vs. ...?
Another `lodash` library one might say. But no, Ramda specifically targets purity and functional programming concepts, meaning there is
not a single function here that could have a side effect (like `Math.random`, `debounce` and etc). There is however `lodash-fp`, which is very similar to Ramda at first, but when digging deeper, Ramda not only comes with loads of
tiny utility functions, but also plays nicely with `Monadic` interfaces in JS, that will be covered in the next chapter.

_P.S. There is a nice collection of recipes for `Ramda` to get you started:
[Ramda cookbook](https://github.com/ramda/ramda/wiki/Cookbook)_

## Few examples
### episode 1: spread
Suppose we have an object and we need to take one of its properties and spread it at a current level:
 
```js
const state = {
  page: {
    title: 'Home page',
    url: '/home',
    assets: [],
    links: [],
  },
  fetchUsers: () => {},
};
```

desired result:
```js
{
  title: 'Home page',
  url: '/home',
  assets: [],
  links: [],
  fetchUsers: () => {},
}
```
 
We can create a generic `spread` function like so:
```js
import { converge, merge, dissoc, prop } from 'ramda';

const spread = converge(merge, [dissoc, prop]);

spread('page', state);
// => { title: 'Home page', url: '/home', assets: [], links: [], fetchUsers: () => {} }
```

### episode 2: reject and transform
Or perhaps we're receiving an array of `page` objects from the server that look like this:

```js
{
  id: 1,
  title: 'home',
  url: '/home',
  resources: [],
  active: null,
  archived: true,
  meta: {
    views: 33,
    links: [],
    comments: null,
  },
}
```

We want to reject the ones that are `archived`, and transform the rest with the following logic:
1. Make `title` uppercase
2. prepend *//test.amido.com* to the `url`
3. default `active` property to false if empty,
4. increment `meta -> views`
5. default `meta -> comments` to 0 if empty

```js
import { evolve, toUpper, inc, defaultTo, concat, map, reject, compose } from 'ramda';

const transformer = evolve({
  title: toUpper,
  url: concat('//test.amido.com'),
  active: defaultTo(false),
  meta: {
    views: inc,
    comments: defaultTo(0),
  },
});

const transformAndReject = compose(map(transformer), reject(prop('archived')));

api.get('/pages').then(transformAndReject);
```

### episode 3: format user
We've just received a `user` object from one endpoint
```js
const user = {
  profile: {
    firstName: 'One',
    lastName: 'Two',
  },
  skills: ['js', 'elm', 'clojure'],
  social: {
    facebook: 'onetwo.facebook.com',
    twitter: '@twitter',
  },
};
```
some posts array from another
```js
const posts = [
  { id: 1, body: 'hello world', created_at: 138201389 },
  { id: 2, body: 'functional world', created_at: 123928838 },
];
```
and we decided that we want to format the user in a very specific way - we want to merge `firstName` and `lastName` in a single, 
`fullName` property and then add `posts` array to the user object under `posts` key.

```js
import { pipe, props, join, objOf, over, lensProp, useWith, merge } from 'ramda';

const createFullName = pipe(
  props(['firstName', 'lastName']),
  join(' '),
  objOf('fullName'),
);

const formatUser = useWith(merge, [
  over(lensProp('profile'), createFullName),
  objOf('posts'),
]);

formatUser(user, posts);
```

# Conclusion
Since in functional programming, an ideal application is just one big function, consisting of these tiny building block functions, can we keep
creating these function compositions and end up with just one big composition in the end?

In an ideal world, we can. However, there are some `undefined` and `null` monsters lurking in the shadows, that want to completely ruin
our code execution and throw loads of errors on the way. There are also side effects that need to be dealt with in a functional way.
We can overcome these obstacles with the help of a few, beautiful monadic interfaces in the next chapter and keep composing the life.

