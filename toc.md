# Migrate To Angular inject Using TypeScript Compiler

## Table Of Content
1. Introduction
2. Problem
3. Solution
4. TypeScript Compiler
  - AST
  - Node
  - Visitor
9. Tree Reversal
10. Angular DI
   - SkipSelf
   - Self
   - Optional
   - Host
   - Inject
11. The Migration Script

## Introduction
So one of the cool features of Angular 14 is the `inject` function -just another way to resolve a dependency- but there was no auto migration for it.

Before that, in order to resolve a dependency in a decorated class -Angular class- you'd have to do the following

``` ts
@Component({ ... })
export class ConstructorInjectionComponent {
    constructor(private _service: Service) { }
}
```

Now we can also do
``` ts
@Component({ ... })
export class UsingInjectFnComponent {
   private _service = inject(Service);
//You can use javascript native private modifier as well!
// #service = inject(Service);

}
```

I won't get into how DI works, if you're new to this stuff I recommend [DI](https://angular.io/guide/dependency-injection-overview).

In this article, we're going to learn how we can create a migration script utilising a typescript compiler

## Problem
There's no problem with constructor injection, I believe it is a matter of style, folks with C# or Java might not like the new approach as it's no longer **simple** to know what dependency a class is requesting, however, assuming it is problem, we need to migrate to the new style to keep up with Angular changes.

## Solution
You might think to change components on the go, change a couple of classes now and then, or change all of them at once! depending on your team and your flavour of injecting dependencies 

## Bouns section
an ESLint rule that will prevent constructor injection

## TypeScript Compiler

### AST
_It's the code you wrote but in a form that can be utilised._ In other words, the code you write is essentially a text that ain't useful unless it can be parsed. That parsing process produces a tree data structure called AST.

Understanding AST is crucial as the rest of the writing is all about it

Taking the following text -code-
```ts
function whatAmI() { }
```

Will turn into 

```json
{ // -> Node
  "flags": 0,
  "kind": 308,
  "statements": [
    { // -> Node
      "kind": 259,
      "flags": 0,
      "name": { // -> Node
        "flags": 0,
        "kind": 79,
        "escapedText": "whatAmI"
      },
      "parameters": [], // Node[]
      "body": { // Node
        "flags": 0,
        "kind": 238,
        "statements": []
      }
    }
  ]
```

Since it's a tree that means in our language each object we see is a **Node**

Some data are easy to understand, but others like flags and kinds might be new to you so let's break them down.

### Flags

