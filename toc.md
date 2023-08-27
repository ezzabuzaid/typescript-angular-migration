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
  - `useFactory` and the `deps` array

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
		return ts.visitEachChild(node, visit, context);
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

A "visitor" is basically a function you define to be invoked for each node in the AST during the traversal. The function is called with the current node and has few return choices.

- Return the node as is, meaning no changes.
- Return a new node of the same kind (otherwise might disrupt the AST) to replace it.
- Return undefined to remove the node entirely.

Regardless of what the choice is, the visit chain will stop right after and this is how we perform manipulations on the AST!

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

Also, we can add other modifiers and combine them

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

One more thing beside DI, typescript allows us to use dependency parameter name within its block without having to use `this`

```ts
@Component({ ... })
export class ConstructorInjectionComponent extends SuperComponent {
    constructor(private _service: Service) {
	_service.someLogic()
    }
}
```

It is valid and we need to keep that in mind as well!

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

## The Migration Script
We will go through a few steps
1. Fetch all files under `tsconfig.json`
2. Encapsulate all files under TypeScript **Program**
4. Prepare the transform function.
5. Run the transform function over each file.

### Reading tsconfig.json

Although this part of the code may appear as a boilerplate, I'm including it here for reference to prevent any confusion when we reference it later in the article. The getFilesFromTsConfig function essentially reads the tsconfig.json file and parses its content, storing the parsed information in the result variable.

```
function getFilesFromTsConfig(tsconfigPath: string) {
	const parseConfigHost: ts.ParseConfigHost = {
		fileExists: ts.sys.fileExists,
		readDirectory: ts.sys.readDirectory,
		readFile: ts.sys.readFile,
		useCaseSensitiveFileNames: true,
	};
	const result = ts.parseJsonConfigFileContent(
		ts.readConfigFile(tsconfigPath, ts.sys.readFile).config,
		parseConfigHost,
		path.dirname(tsconfigPath)
	);
	return result;
}
```

The result variable contains important information extracted from the tsconfig.json file, such as file names and compiler options.

### TypeScript Program

When working with the TypeScript Compiler, one of the central elements you'll encounter is the Program object. This object serves as the starting point for many of the operations you might want to perform, like type checking, emitting output files, or transforming the source code. The Program is created using the `ts.createProgram` function, which can accept a variety of configuration options, such as

- options: These are the compiler options that guide how the TypeScript Compiler will behave. This could include settings like the target ECMAScript version, module resolution strategy, and whether to include type-checking errors, among others.
- rootNames: This property specifies the entry files for the program. It usually contains an array of filenames that act as the roots from which the TypeScript Compiler will begin its operations. These are often the .ts or .tsx files that serve as entry points to your application or library.
- projectReferences: If your TypeScript project consists of multiple sub-projects that reference each other, this property is used to manage those relationships.
- configFileParsingDiagnostics: This property is an array that will capture any diagnostic information or errors that arise when parsing the tsconfig.json file.

```ts
const tsconfigPath = './tsconfig.json'; // path to your tsconfig.json
const tsConfigParseResult = getFilesFromTsConfig(tsconfigPath);

const program = ts.createProgram({
  options: tsConfigParseResult.options,
  rootNames: tsConfigParseResult.fileNames,
  projectReferences: tsConfigParseResult.projectReferences,
  configFileParsingDiagnostics: tsConfigParseResult.errors,
});
```
In this sample, we read all files from a specific `tsconfig.json` and created a TypeScript program from the tsconfig parsing results.

### The Transform Function
If you read the article from the start you've already seen what a transformer function looks like

```ts
const transformer: ts.TransformerFactory<ts.SourceFile> = (context) => {
	const visit: ts.Visitor = (node) => {
		if (ts.isFunctionDeclaration(node)) {
			// do some stuff to a function declaration
		}
		return ts.visitEachChild(node, visit, context);
	};
	return (node) => ts.visitEachChild(node, visit, context);
};
```
Instead of looking for a function node, we need to look for a class node and if a class node doesn't have an Angular decorator then we break the visit chain by returning something

```ts
if (ts.isClassDeclaration(node)) {
const angularDecorators = ['NgModule', 'Component', 'Directive', 'Injectable', 'Pipe'];
  if (angularDecorators.some((it) => getDecorator(node, it) === false) {
	return node // break the visit chain by returning the node as we described in the earlier section
  }
}
```

Keep in mind that if you didn't do a return the code will move till this line `return ts.visitEachChild(node, visit, context);` which means visit the current node children. in our context, that means we're only visiting `ClassDeclaration` node children.

Moving forward, for the `constructor` we need to visit its children, specifically its parameters to convert them to the new syntax and its body to ensure that the code is still working as expected.

```ts
if (ts.isConstructorDeclaration(node)) {
  const updatedNode = ts.visitEachChild(node, visit, context);

  if (
    updatedNode.parameters.length ||
    updatedNode.body?.statements.length
  ) {
    return ts.factory.createConstructorDeclaration(
      node.modifiers,
      updatedNode.parameters,
      updatedNode.body
    );
  }

  return undefined;
}
```
After that let's assume that the constructor no longer has parameters and a body then there is no use for it; That's why you see `return undefined;` which means remove the constructor from the class. Of course, this is up to you, If you prefer to keep an empty constructor then just replace it with `return updatedNode;`

Let's visit the constructor parameter, 

```ts
if (ts.isParameter(node)) {
	if (!node.modifiers) {
		// ignore non constructor parameters
		return node;
	}

	if (!ts.isIdentifier(node.name)) {
		// ignore properties with destructuring
		// @Inject(TOKEN) { someValue }: Interface
		return node;
	}

	const tokenMetadata = makeTokenMetadata(node);
	const parameterMetadata = makeParameterMetadata(node);
	tokensMap[node.name.getText(currentSourceFile)] = tokenMetadata;
	if (tokenMetadata.ignore) {
		// return the node as is since it cannot be migrated
		return node;
	}
	changes.push(convertToInjectSyntax(parameterMetadata, tokenMetadata));
	return undefined;
}
```

The thing about the parameter node is that not only the constructor can have it, class instance and static methods can have it as well, however, there is a convenient way to distinguish constructor parameters from method parameters by checking for the presence of modifiers on the parameter node. if it is undefined then keep the node as is. Another case where the parameter doesn't have a name! if you recall from above, a dependency line can have no name if `InjectionToken` is used or if we didn't prefix it with a modifier.

Assuming the node is a constructor node then we need to
1. Extract some details from it and that is done by calling `makeTokenMetadata` on the node.
2. Save the token for later because we're going to need it in different visit stages.
3. If the node should ignored let it pass, We'll know soon what could be the reasons a parameter node cannot be migrated.
4. Convert the node to the new syntax and store it for later. We'll be using the `changes` array in the visit `ClassDeclaration` stage as we cannot return `PropertyDeclaration` node -The new node from the to migrate to syntax- in the `ParameterDeclaration` visit stage.
5. Finally, return undefined to remove the parameter from the constructor.



## Todo: 
1. Handle inheritance.
2. Handle injection token with type any or not type at all.
3. Migrating more than one class in the same file.
4. Adding `inject` import if it not already imported or update current import if there was one.

## Optimisation
The code does work but it still can be optimised further, for instance, we can parallelise the migration so every 20 files, for instance, are run in a different worker_thread

## Bouns section

an ESLint rule that will prevent constructor injection
