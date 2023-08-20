# Migrate To Angular inject Using TypeScript Compiler

## Table Of Content
1. Introduction
2. Problem
3. Solution
4. TypesScript Compiler
5. AST
6. Nodes
7. Visitor
8. Tree Reversal
9. Angular DI
   - SkipSelf
   - Self
   - Optional
   - Host
   - Inject
10. The Migration Script

## Introduction
So one of the cool features of Angular 14 is the `inject` function -just another way to resolve a dependency- but there was no auto migration for it.

Before that, in order to resolve a dependency in a decorated class -Angular class- you'd have to do the following

```ts
@Component({ ... })
export class ConstructorInjectionComponent {
    constructor(private _service: Service) { }
}
```

Now we can also do
```ts
@Component({ ... })
export class UsingInjectFnComponent {
   private _service = inject(Service);
// you can use javascript native private modifier as well!
// #service = inject(Service);

}
```

I won't get into how DI works, if you're new to this stuff I recommend [DI](https://angular.io/guide/dependency-injection-overview).

In this article, we're going to learn how we can create a migration script utilising a typescript compiler

## Problem
There's actually no problem with constructor injection, I believe it is a matter of style, folks with C# or Java might not like the new approach as it's no longer **simple** to know what dependency a class is requesting, also it might create confusion for the developers 

## Bouns section
an ESLint rule that will prevent constructor injection
