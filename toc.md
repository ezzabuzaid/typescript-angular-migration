# Migrate To Angular `inject` Using TypeScript Compiler

## Table Of Content
1. Introduction
2. Problem
3. Solution
4. TypeScript Compiler
  - AST
  - Node
  - Transformation
  - Visitor
5. Angular DI
6. The Migration Script
  - Simple Injection
  - Injection Token
  - Considering inheritance
  - useFactory and the deps array

## Introduction

Transitioning to Angular's new `inject` function can seem daunting, especially with a large codebase. But thanks to the TypeScript Compiler API, we can automate this task and make our code more maintainable and future-proof.

The TypeScript compiler is not just for compiling your TypeScript code into JavaScript. It provides APIs that allow you to parse, analyze, and even modify your TypeScript code programmatically. We can utilize this feature to create a migration script that will automate the process of updating our Angular classes to use the new `inject` function.

If you have a codebase that's been around for a while, you've likely implemented dependency injection the traditional way. Your Angular components probably look like this:

```ts
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
  #service = inject(Service);

}
```
 

I won't get into how DI works, if you're new to this stuff I recommend [DI](https://angular.io/guide/dependency-injection-overview).

## Problem
There's no problem with constructor injection, I believe it is a matter of style, folks with C# or Java might not like the new approach as it's no longer **simple** to know what dependency a class is requesting, however, in my opinion failing to adopt new styles can lead to deprecated code, technical debt, and missed opportunities for optimization.

## Solution
You could manually update each component. A few here, a few there, and you'll eventually get the job done. But why go through that hassle when TypeScript's Compiler API allows us to automate this task? You've already seen such a thing when doing upgrades using `ng update`, Angular does some sort of code migration to keep your codebase up to date.

Leaving opinions aside, we're here to learn about using TypeScript compiler to beyond and not specifically about Angular. So let's start!

## TypeScript Compiler

The TypeScript Compiler (tsc) takes TypeScript code, which includes type information, and compiles it into plain JavaScript. It also performs type checking to catch errors at compile time rather than at runtime. The compiler utilizes an Abstract Syntax Tree (AST) to understand and transform the source code.

### Abstract Syntax Tree (AST)
_It's the code you wrote but in a form that can be utilised._ In other words, the code you write is essentially a text that isn't useful unless it can be parsed. That parsing process produces a tree data structure called AST. The AST nodes contain a lot of information like their type and location.


Taking the following text -code-
```ts
function whatAmI() { }
```

Will turn into 

```json
{ // -> Source File Node
  "kind": 308,
  "statements": [ // -> Node[]
    { // -> Function Declaration Node
      "kind": 259,
      "name": { // -> Node
        "kind": 79,
        "escapedText": "whatAmI"
      },
      "parameters": [], // Node[]
      "body": { // Block Node
        "kind": 238,
        "statements": []
      }
    }
  ]
```

### Node

In AST, the fundamental unit is called a Node. In the example above, the root node is called SourceFile which has **kind** 308

**Kind** is a numeric value that represents the specific type or category of that node. For instance:
- FunctionDeclaration has kind 259
- Block has kind 238
These numbers are exported in an enum called `SyntaxKind` 

Important to know that if you want to make a change, you usually create a new node that represents the modified version and replace the original node. This is something to keep in mind as we move forward.

The node object has more than just these properties but right now we're only interested in a few, nonetheless, two additional important properties you might want to know about are:
- Parent: This property points to the node that is the parent of the current node in the AST.
- Flags: These are binary attributes stored as flags in the node. They can tell you various properties of the node, such as whether it's a read-only field if it has certain modifiers.

### Transformation
Let's try to change the function name using the TypeScript compiler

```ts
import * as ts from 'typescript';

const code = `function whatAmI() { }`;

const sourceFile = ts.createSourceFile(
	'index.ts', // any file name would do
	code, // the source code
	ts.ScriptTarget.Latest // ES version
);

const transformer: ts.TransformerFactory<ts.SourceFile> = (context) => {
	const visit: ts.Visitor = (node) => {
		if (ts.isFunctionDeclaration(node)) {
			return ts.factory.updateFunctionDeclaration(
				node,
				node.modifiers,
				node.asteriskToken,
				ts.factory.createIdentifier('newFnName'), // -> the new function name
				node.typeParameters,
				node.parameters,
				node.type,
				node.body
			);
		}
		return node;
	};
	return (node) => ts.visitEachChild(node, visit, context);
};

const result = ts.transform<ts.SourceFile>(sourceFile, [transformer]);
printCode(result);

function printCode(result: ts.TransformationResult<ts.SourceFile>) {
	const printer = ts.createPrinter();
	const transformedSource = printer.printFile(result.transformed[0]);
	console.log(transformedSource);
}
```

It doesn't look easy, does it? let's break it down!

We created a source file first to encapsulate our code in a node, and then we used the typescript transform function to **visit** each node, Once we found the `FunctionDeclaration` node we stopped there and **changed** the function name.

To update a node we use `ts.factory` object that provides various factory methods to create or update nodes within the AST. In our specific example, we used `ts.factory.updateFunctionDeclaration` to update the function declaration node with a new name. We kept all other parts of the function declaration the same, including the modifiers, asterisk token (if present), type parameters, parameters, type, and body these were all copied from the original node.

The visit process allows us to traverse the AST, inspecting each node to find the ones we're interested in. Once we find the right node, we can make the necessary changes,

Remember we can know the node we are interested in using the `kind` property so instead of `if (ts.isFunctionDeclaration(node))` we can say `if(node.kind === ts.SyntaxKind.FunctionDeclaration)` but the benefit of using is{Kind} function is that it casts the node type to the appropriate kind.

The `printCode` function it's a utility function that transforms the AST back to code.


### Visitor

I hope you noticed the `visit` function 😁, Let's talk about it, it is a simpler version of what is called the Visitor Pattern. An essential part of how the TypeScript Compiler API works. Actually, you'll see that design pattern whenever you work with AST, Hey at least I did!

A "visitor" is basically a function you define. This function will be invoked for each node in the AST during the traversal. The function is called with the current node and returns a new node that replaces it. This is how we perform manipulations on the AST.

You have a few choices:

- Return the node as is, meaning no changes.
- Return a new node to replace it.
- Return undefined to remove the node entirely.

Okay, time to run the code!

{% replit @EzzAbuzaid/Migration %}


Time to talk about Angular then, do you think so?

## Angular
If you're still new to [how DI works](https://angular.io/guide/dependency-injection-overview) I advise you to read more about it, but if you're comfortable with it let's wake up your memory

There are 3 main parts to injecting a dependency in an Angular class
1. The access modifier whether public, private and protected
2. The Token -dependency-
3. The dependency name

```ts
@Component({ ... })
export class ConstructorInjectionComponent {
    constructor(private _service: Service) { }
}
```

In this sample
1. Access Modifier: private
2. Token: Service
3. Dependency Name: _service

Also, we can add other modifiers and combine them like

- @Self()
- @SkipSelf()
- @Host()
- @Optional()

And be used like this

```ts
@Component({ ... })
export class ConstructorInjectionComponent {
    constructor(
        @Optional() private _optionalService: Service|null,
        @Self() private _sandboxedService: Service,
        @Optional() @Host() private _optionalHostService: Service|null,
        @SkipSelf() private _ignoreCurrentInjectorService: Service,
    ) { }
}
```

Of course the `@Inject` as well

```ts
const APP_VERSION = new InjectionToken<string>('App version token.');

@Component({ ... })
export class ConstructorInjectionComponent {
    constructor(
        @Inject(APP_VERSION) private _appVersion: string,
    ) { }
}
```

Using `InjectionToken` allows us to set arbitrary interface and primitive types like string and `{arbitaryProperty: 'some value'}`. That is important and we will get later to it.

You can prefix the dependency line with as many applicable decorators as we did above.

Still not the end, with a newer typescript version we can use the `override` modifier to say that a dependency is overridden as such

```ts
@Component({ ... })
export class ConstructorInjectionComponent extends SuperComponent {
    constructor(private override _service: Service) { }
}
```

Other modifiers as well like `readonly` should be taken into consideration

_Note: We will call these decorators modifiers from now on._

That is all that you need to recall about Angular DI; we need to know this stuff so we can handle them properly later on, now we are going to rewrite these cases using the `inject` function.

**Old**
```ts
@Component({ ... })
export class ConstructorInjectionComponent {
    constructor(private _service: Service) { }
}
```
**New**
```ts
@Component({ ... })
export class ConstructorInjectionComponent {
  private _service = inject(Service)
}
```

**Old**
```ts
@Component({ ... })
export class ConstructorInjectionComponent {
    constructor(
        @Optional() private _optionalService: Service|null,
        @Self() private _sandboxedService: Service,
        @Optional() @Host() private _optionalHostService: Service|null,
        @SkipSelf() private readonly _ignoreCurrentInjectorService: Service,
    ) { }
}
```
**New**
```ts
@Component({ ... })
export class ConstructorInjectionComponent {
  private _optionalService = inject<Service|null>(Service, {optional: true});
  private _sandboxedService = inject(Service, {self: true});
  private _optionalHostService = inject<Service|null>(Service, {host: true, optional: true});
  private readonly _ignoreCurrentInjectorService = inject(Service, {skipSelf: true});
}
```

**Old**
```ts
@Component({ ... })
export class ConstructorInjectionComponent extends SuperComponent {
    constructor(private override _service: Service) { }
}
```
**New**
```ts
@Component({ ... })
export class ConstructorInjectionComponent extends SuperComponent {
  private override _service = inject(Service)
}
```


 
## Bouns section
an ESLint rule that will prevent constructor injection
