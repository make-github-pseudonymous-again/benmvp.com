---
date: 2020-12-20
title: Quick way to sort a JavaScript array by multiple fields
shortDescription: How to avoid using if statements when sorting arrays by multiple keys in JavaScript
category: JavaScript
tags: [arrays, typescript]
hero: ./shell-game.jpg
heroAlt: Photo of a shell game in action
heroCredit: 'Photo by [HCLDR](https://hcldr.wordpress.com/)'
---

Let's say we have an array of structured objects, like an array of data about the 30 NBA teams:

```js
const nbaTeams = [
  { name: 'Houston Rockets', championships: 2 },
  { name: 'Phoenix Suns', championships: 0 },
  { name: 'Los Angeles Lakers', championships: 17 },
  { name: 'Utah Jazz', championships: 0 },
  { name: 'Golden State Warriors', championships: 6 },
  { name: 'Denver Nuggets', championships: 0 },
  // ...the other 24
]
```

And now we would like to sort the data by the number of championships they've won from highest to lowest. That's pretty straightforward. We can use the [`.sort`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort) method on arrays:

```js
const winningestTeams = nbaTeams.sort(
  (teamA, teamB) => teamB.championships - teamA.championships,
)
```

The way the `.sort` method works is that we give it a comparison function that will take two elements in the array, in our case two team objects. I've called them `teamA` & `teamB`. **It's now our responsibility to tell the method which object should go first based on the number we return** and `.sort` does the rest.

> Keep in mind. Even though `.sort` returns an array it actually mutates the source array by sorting in place.

If we return a negative number, we're saying `teamA` should come before `teamB` in our sorted list. If we return a positive number, we're saying `teamB` should come before `teamA`. And if we return `0`, we're saying they're equal, so leave their order unchanged.

```js
const winningestTeams = nbaTeams.sort(
  // highlight-next-line
  (teamA, teamB) => teamB.championships - teamA.championships,
)
```

Let's say `teamA` is the **Utah Jazz with 0 championships** and `teamB` is the **Houston Rockets with 2 championships**. We can use subtraction to determine our sort order. 2 minus 0 equals 2, a positive number. Therefore `teamB` (Houston Rockets) should come before `teamA` (Utah Jazz) in the sorted list. The `.sort` method calls the comparison function over and over until the array is sorted.

> By the way, if we wanted to sort from lowest to highest, we would do `teamA.championships - teamB.championships` instead.

So this works fine. But what we'll notice is that while the 0-championship teams (Utah Jazz, Phoenix Suns, Denver Nuggets, etc) are at the end of the list, they're not in any particular order. In fact, they're actually in their original order in the list.

Well what if we wanted to sort the array so that if two teams have the same number of championships, they'll be sorted alphabetically? **Now we're trying to sort by two keys or properties of the objects.** There are many ways to accomplish this, including verbose ways using `if` statements, but how about we keep sorting by multiple fields just as succinct as a single field.

```js
const winningestTeams = nbaTeams.sort(
  (teamA, teamB) =>
    teamB.championships - teamA.championships ||
    teamA.name.localeCompare(teamB.name),
)
```

Whoa! 🤯 When I first learned this approach while trying to solve a similar problem, I was amazed at how terse the solution can be. And while it is terse, I think it's still kinda readable. But you can be the judge. 😄 So how does it work?

```js
const winningestTeams = nbaTeams.sort(
  (teamA, teamB) =>
    // highlight-next-line
    teamB.championships - teamA.championships ||
    teamA.name.localeCompare(teamB.name),
)
```

Well it operates on the same principle of the sign of the number returned from the comparison function. But it leverages the fact that `0` means that the values are equal. Since `0` is a falsy value, the expression moves on to the second half of the condition (because that's how `||` works) and compares the team names.

```js
const winningestTeams = nbaTeams.sort(
  (teamA, teamB) =>
    teamB.championships - teamA.championships ||
    // highlight-next-line
    teamA.name.localeCompare(teamB.name),
)
```

We can't subtract two strings from each other. That returns `NaN`. But strings have this super handy [`.localeCompare`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/localeCompare) method which follows the same rules as our comparison function. So when the championships are equal, it will sort the teams alphabetically.

As a result, using our handy dandy comparison function, the sorted array (`winningestTeams`) puts the Los Angeles Lakers at the front and the Utah Jazz wayyyyyy at the end. 😂

```json
[
  { "name": "Los Angeles Lakers", "championships": 17 },
  // ...
  { "name": "Golden State Warriors", "championships": 6 },
  // ...
  { "name": "Houston Rockets", "championships": 2 },
  // ...
  { "name": "Denver Nuggets", "championships": 0 },
  // ... more losers
  { "name": "Phoenix Suns", "championships": 0 },
  { "name": "Utah Jazz", "championships": 0 }
]
```

One more thing. If you've read any of my [other posts](/blog/), you know by now that I'm a huge proponent of TypeScript. Well the best thing about this solution, is that even though it's terse, it still doesn't require any additional TypeScript typing. TypeScript is able to figure out everything using [type inference](https://www.typescriptlang.org/docs/handbook/type-inference.html). It's a win-win.

Keep learning my friends. 🤓
