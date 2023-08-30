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
  - Reading TsConfig
  - TypeScript Program
  - Transform Function
  - The Dependency Line
  - The Migrator
  - Apply Changes
  - Considering Inheritance
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
1. The access modifier whether public, private or protected - _Optional_
2. The Token (dependency) - _Mandatory_ 
3. The dependency name - _Mandatory_ but it can be either an identifier or a destructured object

```ts
@Component({ ... })
export class ConstructorInjectionComponent {
    constructor(private _service: Service) { }
}
```

```ts
@Component({ ... })
export class ConstructorInjectionComponent {
    constructor({someValue}: Service) { }
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

One more thing besides DI, typescript allows us to use dependency parameter name within its block without having to use `this`

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

### Reading TsConfig

Although this part of the code may appear as a boilerplate, I'm including it here for reference to prevent any confusion when we reference it later in the article. The getFilesFromTsConfig function essentially reads the tsconfig.json file and parses its content, storing the parsed information in the result variable.

```ts
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
Instead of looking for a function node, we need to look for a class node and check if it is qualified for migration and if not we need to break the visit chain by returning the node as is as explain before.

A class is qualified for migration if 
1. The class node should have an Angular decorator.
2. The class node should have a constructor present.
3. The constructor should have parameters (dependency lines)

```ts
if (ts.isClassDeclaration(node)) {
	const cstr = node.members.find(ts.isConstructorDeclaration);
	if (!cstr || cstr.parameters.length === 0) {
		return node;
	}

	const angularDecorators = ['NgModule', 'Component', 'Directive', 'Injectable', 'Pipe'];
	const isAngularClass = angularDecorators.some((it) => getDecorator(node, it))
	if (!isAngularClass) {
		return node;
	}

	// the class node is already qualified at this point
}
```

The `return node` breaks the visit chain as we described in the earlier section. Keep in mind that if you didn't do a return the code will move till this line `return ts.visitEachChild(node, visit, context);` which means visit the current node children. in our context, that means we're only visiting `ClassDeclaration` node children.

Moving forward, for the `constructor` we need to visit its children, specifically its parameters to convert them to the new syntax and its body to ensure that the code is still working as expected (prefix used dependencies with `this`).

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

Let's visit the constructor parameter

### The Dependency Line

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

	const tokenMetadata = makeTokenMetadata(node, node.name);
	const parameterMetadata = makeParameterMetadata(node);

	if (!tokenMetadata) {
		// ignore parameters without token -Deps Type-
		return node;
	}

	tokensMap[node.name.getText(currentSourceFile)] = tokenMetadata;
	changes.push(convertToInjectSyntax(parameterMetadata, tokenMetadata));
	return undefined;
}
```

The thing about the parameter node is that not only the constructor can have it, class instance and static methods can have it as well, however, there is a convenient way to distinguish constructor parameters from method parameters by checking the presence of `modifiers` on the parameter node.

So, a parameter can be migrated if
1. It doesn't have `modifiers`.
2. It has a name! if you recall from above, the dependency line can have a destructure syntax instead of a name.
3. It has a token (Dependency Type) - we can know that by checking if the parameter type is `TypeReference`, more on that later.

```ts
constructor(private {someValue}: Service) { } // no name
constructor(private _service: any) { } // no token
constructor(private _service) { } // no type at all
```

Assuming the node is a constructor parameter node then we need to
1. Extract some details from it and that is done by calling `makeTokenMetadata` and `makeParameterMetadata` on the node.
2. Save the token for later because we're going to need it in different visit stages.
3. Convert the node to the new syntax and store it for later. We'll be using the `changes` array in the visit `ClassDeclaration` stage as we cannot return `PropertyDeclaration` node -The new node from the migrate function- in the `ParameterDeclaration` visit stage.
4. Finally, return undefined to remove the parameter from the constructor.

Moving on, let's write the `makeTokenMetadata` function. the function should return three properties:
- name: the dependency line name.
- token: the dependency token.
- type: the dependency full type.

The difference between token and type could be demonstrated in the following code

```ts
constructor(private _elementRef: ElementRef) {}
constructor(private _elementRef: ElementRef<InputHtmlElement>) {}
```

The token and type in the first constructor are the same but in the second the token is `ElementRef` but the type is `ElementRef<InputHtmlElement>`

There are two ways to specify the dependency line token

1. Using a dependency type as a token
2. Using `@Inject` to specify the token

```ts
constructor(private _service: Service) { }
constructor(@Inject(FAST_TOKEN) private _measure: IFast) { }
```

Resuming `makeTokenMetadata`

To extract details from the first way (without `@Inject`)

```ts
function makeTokenMetadata(
	param: ts.ParameterDeclaration,
	paramName: ts.Identifier
) {
	const token = // -> 1
		param.type &&
		ts.isTypeReferenceNode(param.type) &&
		ts.isIdentifier(param.type.typeName)
			? param.type.typeName.text
			: undefined;

	if (!token) { // -> 2
		return undefined;
	}

	return {
		name: paramName.text,
		token: token,
		get genericType() { // -> 3
			// the type reference could be ElementRef<HTMLElement>
			// but the token can only be ElementRef
			// so if the type is the same as the token
			// we don't need to specify it as generic in the inject function
			const type = param.type?.getText(currentSourceFile);
			return type === token ? undefined : type;
		},
	};
}
```

1. A parameter doesn't necessarily have a type so we need to make sure it does and it is a `TypeReference`. Once we get there we only need `typeName` and that is our token. We said before that `TypeReference` can be `ElementRef<InputHtmlElement>` therefore the `typeName` here is `ElementRef` only.
2. When we visited the parameter node before we added a check that says if there is no token then we pass that parameter.
3. The `genericType` is the full `TypeReference`. The common case is that a token is the same as the type, in that case, we don't need to add it as a generic type to the `inject` function.  

_Note: I'm assuming that the line of dependency has a valid TypeReference (non-valid could be union or primitive type). Angular already validates that on startup._

Let's extract details from the `Inject` way

```ts
let injectDecorator = getDecorator(param, 'Inject'); // -> 1
if (injectDecorator) {
	const args = (injectDecorator.expression as ts.CallExpression).arguments;
	return {
		name: paramName.text,
		token: (args[0] as ts.Identifier).text, // -> 2
		get genericType() { // -> 3
			// We need the full type regardless of what it is.
			return param.type?.getText(currentSourceFile);
		},
	};
}
```

1. The `getDecorator` is a utility function to get a decorator from a node if there is one.
2. The `@Inject()` decorator accepts the token as the first argument, I'll assume it is there because TypeScript won't allow it otherwise.
3. We need to return type as is because that line of dependency allows any type.

Now that we have got the information we need about the token let's examine the parameter modifiers

```ts
function makeParameterMetadata(param: ts.ParameterDeclaration) {
	const hasModifier = (modifier: ts.Modifier['kind']) =>
		(param.modifiers ?? []).some((m) => m.kind === modifier);

	return {
		isPublic: hasModifier(ts.SyntaxKind.PublicKeyword),
		isPrivate: hasModifier(ts.SyntaxKind.PrivateKeyword),
		isProtected: hasModifier(ts.SyntaxKind.ProtectedKeyword),
		isReadonly: hasModifier(ts.SyntaxKind.ReadonlyKeyword),
		isOptional: getDecorator(param, 'Optional'),
		isSelf: getDecorator(param, 'Self'),
		isSkipSelf: getDecorator(param, 'SkipSelf'),
		isHost: getDecorator(param, 'Host'),
	};
}
```
Nothing fancy here; both modifiers and decorators are arrays, so a simple array search operation is performed to find the required metadata. The function makeParameterMetadata returns an object containing boolean flags indicating which modifiers and decorators are present.

Here's the `getDecorator` utility function, the same as `hasModifier` but it also makes sure that a decorator is a `CallExpression` -invoked as you're invoking a function `@Self()` with parentheses-

```ts
function getDecorator(node: ts.HasDecorators, decoratorName: string) {
	const decorators = ts.getDecorators(node) ?? [];
	return decorators.find((it) => {
		if (ts.isCallExpression(it.expression)) {
			return (
				ts.isIdentifier(it.expression.expression) &&
				it.expression.expression.text === decoratorName
			);
		}
		return false;
	});
}
```

### The Migrator

We've all been waiting for this function, lucky you it is simple to digest.
Recall how we request dependency using `inject` function.

Syntax
```
[public/private/protected/#] [readonly/override] <name> = inject(<Token>, [{
	optional: true,
	skipSelf: true,
	self: true,
	host: true
}])
```

Example
```ts
private _service = inject(Service);
#service = inject(Service, {
	host: true
});
```

Essentially, you construct a new property declaration that incorporates all the previous settings like visibility (public, private, etc.) and injection flags/options

First, let's create the inject function arguments.
```ts
function convertToInjectSyntax(
	parameterMetadata: ReturnType<typeof makeParameterMetadata>,
	tokenMetadata: NonNullable<ReturnType<typeof makeTokenMetadata>>
) {
	const injectArgs: ts.Expression[] = [
		ts.factory.createIdentifier(tokenMetadata.token),
	];
}
```
See, told you, simple. A function that accepts both token and parameter metadata, the `injectArgs` are the arguments for the `inject` function, in this sample we add the token argument. Let's add the options argument

```ts
const optionsProperties: ts.PropertyAssignment[] = [];
if (parameterMetadata.isOptional) {
	optionsProperties.push(
		ts.factory.createPropertyAssignment('optional', ts.factory.createTrue())
	);
}
if (parameterMetadata.isSelf) {
	optionsProperties.push(
		ts.factory.createPropertyAssignment('self', ts.factory.createTrue())
	);
}
if (parameterMetadata.isSkipSelf) {
	optionsProperties.push(
		ts.factory.createPropertyAssignment('skipSelf', ts.factory.createTrue())
	);
}
if (parameterMetadata.isHost) {
	optionsProperties.push(
		ts.factory.createPropertyAssignment('host', ts.factory.createTrue())
	);
}
// append the options object only if it has options
if (optionsProperties.length) {
	injectArgs.push(
		ts.factory.createObjectLiteralExpression(optionsProperties)
	);
}
```

An object property has two things, key and value -initializer-. the `createPropertyAssignment` function accepts the key as the first argument and value -Expression- as the second argument.
The `options` object will only appear if there is a need for it.

_Note: you might want to consider making the inject type to be token type or null when `isOptional` is true to catch errors at compile time_

It's time to create the inject function

```ts
const depsType = tokenMetadata.genericType
	? [ts.factory.createTypeReferenceNode(tokenMetadata.genericType, undefined)]
	: undefined;

const injectFn = ts.factory.createCallExpression(
	ts.factory.createIdentifier('inject'),
	depsType,
	injectArgs
);
```
With that in place, the `inject` function is complete, we have got the token, the generic type and arguments.

Time for the left part of the statement.

```ts
const modifiers: ts.Modifier[] = [
	ts.factory.createModifier(
		parameterMetadata.isPublic
			? ts.SyntaxKind.PublicKeyword
			: parameterMetadata.isProtected
			? ts.SyntaxKind.ProtectedKeyword
			: ts.SyntaxKind.PrivateKeyword
	),
];

if (parameterMetadata.isReadonly) {
	modifiers.push(ts.factory.createModifier(ts.SyntaxKind.ReadonlyKeyword));
}

const propertyName = ts.factory.createIdentifier(tokenMetadata.name);

```

In the snippet, we're adding things back as they were, with one exception we default to private modifier as a last resort. You might be thinking why not use JavaScript native private identifier In that case, sure thing we can

```ts
const modifiers: ts.Modifier[] = [];
const accessModifier = parameterMetadata.isPublic
	? ts.SyntaxKind.PublicKeyword
	: parameterMetadata.isProtected
	? ts.SyntaxKind.ProtectedKeyword
	: null;

if (accessModifier) {
	modifiers.push(ts.factory.createModifier(accessModifier));
}

if (parameterMetadata.isReadonly) {
	modifiers.push(ts.factory.createModifier(ts.SyntaxKind.ReadonlyKeyword));
}

const propertyName = accessModifier
	? ts.factory.createIdentifier(tokenMetadata.name)
	: ts.factory.createPrivateIdentifier(tokenMetadata.name);
```
We had to make a slight modification to the logic where we default to `null` if the parameter isn't `public` or `protected`.

The statement is complete left and right parts. Combining everything to create class property.

```ts
return ts.factory.createPropertyDeclaration(
	modifiers,
	propertyName,
	undefined, // question or exclamation token (? !)
	undefined, // property type. Not needed as we're relying on the inject generic type
	injectFn
);
```

Congrats! The migrator function is done.

At this point, you might wonder how to integrate this inject function into Angular classes. Well, that's the next big step! You'll replace the original constructor parameter declaration with this newly formed inject function call. Recall that when we visited the parameter node -`ts.isParameter(node)`-, we stored the result of the migrator function in the `changes` array. It is the time to make use of it.

### Apply Changes
Let's go back to visiting `ClassDeclaration` node and complete where we left off

```ts
if (ts.isClassDeclaration(node)) {
	// the class node is already qualified at this point
}
```



## Todo: 
1. Handle inheritance.
2. Handle injection token with type any or not type at all.
3. Migrating more than one class in the same file.
4. Adding `inject` import if it is not already imported or updating current import if there was one.

## Optimisation
The code does work but it still can be optimised further, for instance, we can parallelise the migration so every 20 files, for instance, are run in a different worker_thread

## Outroduction
Throughout the writing, we assumed your code is already valid and can be compiled and works on runtime, if that is not the case then you'll have to adjust the code a bit to handle the error, a clear example is this line
```ts
const args = (injectDecorator.expression as ts.CallExpression).arguments;
```
Here, we assume the decorator is invoked `@Inject(SOME_TOKEN)` but if it is written like `@Inject` then it won't work unless you avoid the parameter completely by checking if the `injectDecorator` node is invoked by calling `ts.isCallExpression`. That is one example, however, there are a lot of such.

Another thing, if you're using custom decorators along with Angular ones you'd need to make some adjustments to keep the same behaviour, like moving the custom decorators along with other properties

I hope you learned something new today about TypeScript Compiler, It works wonders for such a huge change. Look at your codebase I'm certain that you can find a use case somewhere. The compiler API can be used to accomplish different things as well like code generation or ensuring specific criteria are met before/after the build script run or [transforming code into fly](https://github.com/TypeStrong/ts-loader#getcustomtransformers).

## Bouns section

an ESLint rule that will prevent constructor injection
