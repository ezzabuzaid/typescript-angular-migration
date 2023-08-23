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
You might think about changing components on the go, changing a couple of classes now and then, or changing all of them at once! Today, we're going to see how we can migrate them all at once.

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
  "flags": 0,
  "kind": 308,
  "statements": [ // -> Node[]
    { // -> Function Declaration Node
      "kind": 259,
      "flags": 0,
      "name": { // -> Node
        "flags": 0,
        "kind": 79,
        "escapedText": "whatAmI"
      },
      "parameters": [], // Node[]
      "body": { // Block Node
        "flags": 0,
        "kind": 238,
        "statements": []
      }
    }
  ]
```

### Node

In the above tree, the root node is called SourceFile.

You can see that each Node has two common properties
- kind: is a numeric value that represents the specific type or category of that node.
- flags: used to store specific binary attributes that give additional information about the node. This might include characteristics like visibility modifiers, whether a node is in a certain state, or other special properties.

We can know what the node is by looking at its kind
- FunctionDeclaration (kind 259)
- Block (kind 238)

Flags also are useful to know if a node has a specific attribute. consider a class field, using flags we can check whether it is a read-only or not. Flags are optimisation technique used to store additional information in a node.

### Migrate Sample
Let's try to change the function name using the TypeScript compiler
```ts
import * as ts from 'typescript';

const code = `function whatAmI() { }`;

const sourceFile = ts.createSourceFile(
  'index.ts', // any file name would do
  code, // the source code 
  ts.ScriptTarget.Latest // ES version
);

const result = ts.transform<ts.SourceFile>(sourceFile, [
  (context) => {
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
  }
]);
```

It doesn't look easy, does it? let's break it down!

We created a source file first to encapsulate our code in a node, and then we used the typescript transform function to **visit** each nodes, Once we found the FunctionDeclaration node we stopped there and **changed** the function name.

To update a node we use `ts.factory` object that provides various factory methods to create or update nodes within the AST. In our specific example, we used `ts.factory.updateFunctionDeclaration` to update the function declaration node with a new name. We kept all other parts of the function declaration the same, including the modifiers, asterisk token (if present), type parameters, parameters, type, and body. These were all copied from the original node.

The visit process allows us to traverse the AST, inspecting each node to find the ones we're interested in. Once we find the right node, we can make the necessary changes,

Remember we can know the node we are interested in using the `kind` property so instead of `if (ts.isFunctionDeclaration(node))` we can say `if(node.kind === ts.SyntaxKind.FunctionDeclaration)` but the benefit of using is{Kind} function is that it casts the node type to the appropriate kind.


## Conclusion
There 

## Bouns section
an ESLint rule that will prevent constructor injection
