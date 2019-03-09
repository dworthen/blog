---
template: post
title: TypeScript and Runtime Type Safety
slug: /posts/typescript-and-runtime-type-safety/
draft: false
date: '2018-05-24'
description: >-
  Intellisense, type definitions, tooling, and documentation. A few of the
  benefits that follow from TypeScript's compile-time static type system but
  it's not all good news. There are some drawbacks. For starters, the type
  checking, and thus type safety, occur at compile time. What about Runtime type
  safety?
category: TypeScript
tags:
  - TypeScript
---

I began programming in TypeScript by accident. I was exploring Microsoft's new ASP.NET Core framework and before I knew it, I was writing code in TypeScript.

It wasn't my fault. My IDE of choice made the decision for me, without my consent. Visual Studio had decided to scaffold out a project with TypeScript. I thought, "why not?". After all, learning TypeScript was a longtime desire of mine and a chore I put off for long enough. It was time to learn TypeScript.

I was immediately drawn to the type system. The compiler provides type safety and valuable feedback. Take the following code for example:

```TypeScript
export function add(a: number, b: number): number {
    return a + b;
}
```

Valid TypeScript. Now let's call, or should I say miscall, `add`.

```TypeScript
add('Hello', { name: "World" });
```

Not so valid. Don't be mistaken, this is valid JavaScript but not valid TypeScript. `'Hello'` and `{ name: "World" }` are invalid arguments for `add` since they are not numbers. The compiler will inform us of this blunder.

Writing application code in TypeScript adds type safety, a great benefit to developers. Editing `add` to accept strings instead of numbers will cause the compiler to explode. That is until we update all calls of `add` to use strings. A valuable feedback loop useful for refactoring. The compiler is the first line of defense.

Writing reusable JavaScript libraries in TypeScript is a different matter. What if we were to package and distribute `add` as a JavaScript library? The following is the compiled JavaScript version of `add`.

```JavaScript
export function add(a, b) {
    return a + b;
}
```

That's right. TypeScript's compiler removes all information about types. This makes sense. TypeScript compiles to valid JavaScript, which does not have a static type system. Though it provides compile-time type safety, TypeScript **does not** provide runtime type safety.

Compilation removes type information and restraints on `add`. Developers using the compiled library may call `add` with strings instead of numbers. Nothing prevents them from doing so.

---

## Solution

As always with library development, consider end users and write unit tests for all use cases. On top of unit tests, consider writing runtime type checks. This will prevent misuse of code or provide relevant feedback when misuse occurs.

```TypeScript
export function add(a: number, b: number): number {
    if(typeof a !== 'number') {
        throw new TypeError('....');
    }
    if(typeof b !== 'number') {
        throw new TypeError('....');
    }
    return a + b;
}
```

The above is still valid, and beautiful, TypeScript. The compiler has no problems with the extra type guards.
