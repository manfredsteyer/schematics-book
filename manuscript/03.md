# Generating custom Angular Code with the CLI and Schematics, Part III: Extending existing Code with TypeScript's Compiler API

In my two previous blog posts, I've shown how to leverage Schematics to [generate custom code with the Angular CLI](https://softwarearchitekt.at/post/2017/10/29/generating-custom-code-with-the-angular-cli-and-schematics.aspx) as well as to [update an existing NgModules](https://softwarearchitekt.at/post/2017/12/01/generating-angular-code-with-schematics-part-ii-modifying-ngmodules.aspx) with declarations for generated components. The latter one was not that difficult because this is a task the CLI performs too and hence there are already helper functions we can use.

But, as one can imagine, we are not always that lucky and find existing helper functions. In these cases we need to do the heavy lifting by ourselves and this is what this post is about: Showing how to directly modify existing source code in a safe way. 

When we look into the helper functions used in the previous article, we see that they are using the [TypeScript Compiler API](https://github.com/Microsoft/TypeScript/wiki/Using-the-Compiler-API) which e. g. gives us a syntax tree for TypeScript files. By traversing this tree and looking at its nodes we can analyse existing code and find out where a modification is needed. 

Using this approach, this post extends the schematic from the last article so that the generated Service is injected into the ``AppComponent`` where it can be configured:

```TypeScript
[...]
import { SideMenuService } from './core/side-menu/side-menu.service';

@Component({ [...] })
export class AppComponent {

  constructor(
    private sideMenuService: SideMenuService) {
        // sideMenuService.show = true;
  }
}
```

I think, providing boilerplate for configuring a library that way can lower the barrier for getting started with it. However, please note that this simple example represents a lot of situations where modifying existing code provides more convenience. 

The [source code](https://github.com/manfredsteyer/schematics-modify-typescript-ast) for the examples used for this can be found [here in my GitHub repository](https://github.com/manfredsteyer/schematics-modify-typescript-ast).

> Schematics is currently an Angular Labs project. Its public API is experimental and can change in future.
> ![Angular Labs](https://i.imgur.com/y8LIiVg.png)

## Walking a Syntax Tree with the TypeScript Compiler API

To get familiar with the TypeScript Compiler API, let's start with a simple NodeJS example that demonstrates its fundamental usage. All we need for this is TypeScript itself. As I'm going to use it within an simple NodeJS application, let's also install the typings for it. For this, we can use the following commands in a new folder:

```
npm init
npm install typescript --save
npm install @types/node --save-dev
```

In addition to that, we need a ``tsconfig.json`` with respective compiler settings:

```JSON
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
    "lib": ["dom", "es2017"],
    "moduleResolution": "node"
  }
}
```

Now we have everything in place for our first experiment with the Compiler CLI. Let's create a new file ``index.ts``:

```TypeScript
import * as ts from 'typescript';
import * as fs from 'fs';

function showTree(node: ts.Node, indent: string = '    '): void {

    console.log(indent + ts.SyntaxKind[node.kind]);
    
    if (node.getChildCount() === 0) {
        console.log(indent + '    Text: ' + node.getText());
    }

    for(let child of node.getChildren()) {
        showTree(child, indent + '    ');
    }
}

let buffer = fs.readFileSync('demo.ts');
let content = buffer.toString('utf-8');
let node = ts.createSourceFile('demo.ts', content, ts.ScriptTarget.Latest, true);

showTree(node);
```

The ``showTree`` function recursively traverses the syntax tree beginning with the passed node. For this it logs the node's ``kind`` to the console. This property tells us whether the node represents for instance a class name, a constructor or a parameter list. If the node doesn't have any children, the program is also printing out the node's textual content, e. g. the represented class name. The function repeats this for each child node with an increased indent.

At the end, the program is reading a TypeScript file and constructing a new ``SourceFile`` object with it's content. As the type ``SourceFile`` is also a node, we can pass it to ``showTree``.

In addition to this, we also need the ``demo.ts`` file the application is loading. For the sake of simplicity, let's go with the following simple class:

```TypeScript
class Demo {
    constructor(otherDemo: Demo) {}
}
```

To compile and run the application, we can use the following commands:

```
tsc index.ts
node index.js
```

Of course, it would make sense to create a npm script for this.

When running, the application should show the following syntax tree:

```
SourceFile
    SyntaxList
        ClassDeclaration
            ClassKeyword
                Text: class
            Identifier
                Text: Demo
            FirstPunctuation
                Text: {
            SyntaxList
                Constructor
                    ConstructorKeyword
                        Text: constructor
                    OpenParenToken
                        Text: (
                    SyntaxList
                        Parameter
                            Identifier
                                Text: otherDemo
                            ColonToken
                                Text: :
                            TypeReference
                                Identifier
                                    Text: Demo
                    CloseParenToken
                        Text: )
                    Block
                        FirstPunctuation
                            Text: {
                        SyntaxList
                            Text: 
                        CloseBraceToken
                            Text: }
            CloseBraceToken
                Text: }
    EndOfFileToken
        Text: 
```

Take some time to look at this tree. As you see, it contains a node for every aspect of our ``demo.ts``. For instance, there is a node with of the kind ``ClassDeclaration`` for our class and it contains a ``ClassKeyword`` and an ``Identifier`` with the text ``Demo``. You also see a ``Constructor`` with nodes that represent all the pieces a constructor consists of. It contains a ``SyntaxList`` with a sub tree for the constructor argument ``otherDemo``.

When we combine what we've learned when writing this example with the things we already know about Schematics from the previous articles, we have everything to implement the initially described endeavor. The next sections describe the necessary steps.

## Providing Key Data

When writing a Schematics rule, a first good step is thinking about all the data it needs and creating a class for it. In our case, this class looks like this:

```TypeScript
export interface AddInjectionContext {
    appComponentFileName: string;       
        // e. g. /src/app/app.component.ts

    relativeServiceFileName: string;    
        // e. g. ./core/side-menu/side-menu.service
    
    serviceName: string;
        // e. g. SideMenuService
}
```

To get this data, let's create a function ``createAddInjectionContext``:

```TypeScript
function createAddInjectionContext(options: ModuleOptions): AddInjectionContext {
    
    let appComponentFileName = '/' +  options.sourceDir + '/' + options.appRoot + '/app.component.ts';
    let destinationPath = constructDestinationPath(options);
    let serviceName = classify(`${options.name}Service`);
    let serviceFileName = join(normalize(destinationPath), `${dasherize(options.name)}.service`);

    let relativeServiceFileName = buildRelativePath(appComponentFileName, serviceFileName);

    return {
        appComponentFileName,
        relativeServiceFileName,
        serviceName
    }
}
```

As this listing shows, ``createAddInjectionContext`` takes an instance of the class ``ModuleOptions``. It is part of the utils Schematics contains and represents the parameters the CLI passes. The three needed fields are inferred from those instance. To find out in which folder the generated files are placed, it uses the custom helper ``constructDestinationPath``:

```TypeScript
export function constructDestinationPath(options: ModuleOptions): string {
    
    return '/' + (options.sourceDir? options.sourceDir + '/' : '') + (options.path || '')
                + (options.flat ? '' : '/' + dasherize(options.name));
}
```

In addition to this, it uses further helper functions Schematics provides us:

- ``classify``: Creates a class name, e. g. ``SideMenu`` when passing ``side-menu``.
- ``normalize``: Normalizes a path in order to compensate for platform specific characters like \ under Windows.
- ``dasherize``: Converts to Kebab case, e. g. it returns ``side-menu`` for ``SideMenu``.
- ``join``: Combines two paths.
- ``buildRelativePath``: Builds a relative path that points from the first passed absolute path to the second one.

Please note, that some of the helper functions used here are not part of the public API. To prevent breaking changes I've copied the respective files. [More about this wrinkle](https://softwarearchitekt.at/post/2017/12/01/generating-angular-code-with-schematics-part-ii-modifying-ngmodules.aspx) can be found in my [previous article](https://softwarearchitekt.at/post/2017/12/01/generating-angular-code-with-schematics-part-ii-modifying-ngmodules.aspx) about this topic.

## Adding a new constructor

In cases where the ``AppComponent`` does not have a constructor, we have to create one. The Schematics way of doing this is creating a ``Change``-Object that describes this modification. For this task, I've created a function ``createConstructorForInjection``. Although it is a bit long because we have to include several null/undefined checks, it is quite straight:

```TypeScript
function createConstructorForInjection(context: AddInjectionContext, nodes: ts.Node[], options: ModuleOptions): Change {
    let classNode = nodes.find(n => n.kind === ts.SyntaxKind.ClassKeyword);
    
    if (!classNode) {
        throw new SchematicsException(`expected class in ${context.appComponentFileName}`);
    }
    
    if (!classNode.parent) {
        throw new SchematicsException(`expected constructor in ${context.appComponentFileName} to have a parent node`);
    }

    let siblings = classNode.parent.getChildren();
    let classIndex = siblings.indexOf(classNode);

    siblings = siblings.slice(classIndex);

    let classIdentifierNode = siblings.find(n => n.kind === ts.SyntaxKind.Identifier);

    if (!classIdentifierNode) {
        throw new SchematicsException(`expected class in ${context.appComponentFileName} to have an identifier`);
    }

    if (classIdentifierNode.getText() !== 'AppComponent') {
        throw new SchematicsException(`expected first class in ${context.appComponentFileName} to have the name AppComponent`);
    }

    // Find opening cury braces (FirstPunctuation means '{' here).
    let curlyNodeIndex = siblings.findIndex(n => n.kind === ts.SyntaxKind.FirstPunctuation); 
        
    siblings = siblings.slice(curlyNodeIndex);

    let listNode = siblings.find(n => n.kind === ts.SyntaxKind.SyntaxList);

    if (!listNode) {
        throw new SchematicsException(`expected first class in ${context.appComponentFileName} to have a body`);
    }

    let toAdd = `
  constructor(private ${camelize(context.serviceName)}: ${classify(context.serviceName)}) {
    // ${camelize(context.serviceName)}.show = true;
  }
`;
    return new InsertChange(context.appComponentFileName, listNode.pos+1, toAdd);

}
```

The parameter ``nodes`` contains all nodes of the syntax tree in a flat way. This structure is also used by some default rules Schematics comes with and allows to easily search the tree with Array methods. The function looks for the first node of the kind ``ClassKeyword`` which contains the ``class`` keyword. Compare this with the syntax tree above which was displayed by the first example.

After this it gets an array with the ``ClassKeyword``'s siblings (=its parent's children) and searches it from left to right in order to find a position for the new constructor. To search from left to right, it truncates everything that is on the left of the current position using ``slice`` several times. To be honest, this is not the best decision in view of performance, but it should be fast enough and I think that it makes the code more readable.

Using this approach, the functions walks to the right until it finds a ``SyntaxList`` (= class body) that follows a ``FirstPunctuation`` node (= the character '{' in this case) which in turn follows an ``Identifier`` (= the class name). Then it uses the position of this ``SyntaxList`` to create an ``InsertChange`` object that describes that a constructor should be inserted there.

Of course, we could also search the body of the class to find a more fitting place for the constructor -- e. g. between the property declarations and the method declarations -- but for the sake of simplicity and demonstration, I've dropped this idea.

## Adding a constructor argument

If there already is a constructor, we have to add another argument for our service. The following function is taking care about this task. Among other parameters, it takes the node that represents the constructor. You can also compare this with the syntax tree of our first example at the beginning.

```TypeScript
function addConstructorArgument(context: AddInjectionContext, ctorNode: ts.Node, options: ModuleOptions): Change {

    let siblings = ctorNode.getChildren();

    let parameterListNode = siblings.find(n => n.kind === ts.SyntaxKind.SyntaxList);
    
    if (!parameterListNode) {
        throw new SchematicsException(`expected constructor in ${context.appComponentFileName} to have a parameter list`);
    }

    let parameterNodes = parameterListNode.getChildren();

    let paramNode = parameterNodes.find(p => {
        let typeNode = findSuccessor(p, [ts.SyntaxKind.TypeReference, ts.SyntaxKind.Identifier]);
        if (!typeNode) return false;
        return typeNode.getText() === context.serviceName;
    });

    // There is already a respective constructor argument --> nothing to do for us here ...
    if (paramNode) return new NoopChange();

    // Is the new argument the first one?
    if (!paramNode && parameterNodes.length == 0) {
        let toAdd = `private ${camelize(context.serviceName)}: ${classify(context.serviceName)}`;
        return new InsertChange(context.appComponentFileName, parameterListNode.pos, toAdd);
    }
    else if (!paramNode && parameterNodes.length > 0) {
        let toAdd = `,
    private ${camelize(context.serviceName)}: ${classify(context.serviceName)}`;
        let lastParameter = parameterNodes[parameterNodes.length-1];
        return new InsertChange(context.appComponentFileName, lastParameter.end, toAdd);
    }

    return new NoopChange();
}
```

This function retrieves all child nodes of the constructor and searches for a ``SyntaxList`` (=the parameter list) node having a ``TypeReference`` child which in turn has a ``Identifier`` child. For this, it uses the helper function ``findSuccessor`` displayed below. The found identifier holds the type of the argument in question. If there is already an argument that points to the type of our service, we don't need to do anything. Otherwise the function checks wether we are inserting the first argument or a subsequent one. In each case, the correct position for the new argument is located and then the function returns a respective ``InsertChange``-Object for the needed modification.

```TypeScript
function findSuccessor(node: ts.Node, searchPath: ts.SyntaxKind[] ) {
    let children = node.getChildren();
    let next: ts.Node | undefined = undefined;

    for(let syntaxKind of searchPath) {
        next = children.find(n => n.kind == syntaxKind);
        if (!next) return null;
        children = next.getChildren();
    }
    return next;
}
```

## Deciding whether to create or modify a Constructor

The good message first: We've done the heavy work. What we need now is a function that decides which of the two possible changes -- adding a constructor or modifying it -- needs to be done:

```TypeScript
function buildInjectionChanges(context: AddInjectionContext, host: Tree, options: ModuleOptions): Change[] {

    let text = host.read(context.appComponentFileName);
    if (!text) throw new SchematicsException(`File ${options.module} does not exist.`);
    let sourceText = text.toString('utf-8');

    let sourceFile = ts.createSourceFile(context.appComponentFileName, sourceText, ts.ScriptTarget.Latest, true);

    let nodes = getSourceNodes(sourceFile);
    let ctorNode = nodes.find(n => n.kind == ts.SyntaxKind.Constructor);
    
    let constructorChange: Change;

    if (!ctorNode) {
        // No constructor found
        constructorChange = createConstructorForInjection(context, nodes, options);
    } 
    else { 
        constructorChange = addConstructorArgument(context, ctorNode, options);
    }

    return [
        constructorChange,
        insertImport(sourceFile, context.appComponentFileName, context.serviceName, context.relativeServiceFileName) 
    ];

}
```

As the first sample in this post, it uses the TypeScript Compiler API to create a ``SourceFile`` object for the file containing the ``AppComponent``. Then it uses the function ``getSourceNodes`` which is part of Schematics to traverse the whole tree and creates a flat array with all nodes. These nodes are searched for a constructor. If there is none, we are using our function ``createConstructorForInjection`` to create a ``Change`` object; otherwise we are going with ``addConstructorArgument``. At the end, the function returns this ``Change`` together with another ``Change`` created by ``insertImport`` which also comes with Schematics and creates the needed ``import`` statement at the beginning of the TypeScript file.

Please note that the order of these two changes is vital because they are adding lines to the source file which is forging the position information within the node objects.

## Putting all together

Now, we just need a factory function for a rule that is calling ``buildInjectionChanges`` and applying the returned changes:

```TypeScript
export function injectServiceIntoAppComponent(options: ModuleOptions): Rule {
    return (host: Tree) => {

        let context = createAddInjectionContext(options);
        let changes = buildInjectionChanges(context, host, options);
        
        const declarationRecorder = host.beginUpdate(context.appComponentFileName);
        for (let change of changes) {
            if (change instanceof InsertChange) {
                declarationRecorder.insertLeft(change.pos, change.toAdd);
            }
        }
        host.commitUpdate(declarationRecorder);
    
        return host;
    };
};
```

This function takes the ``ModuleOptions`` holding the parameters the CLI passes and returns a ``Rule`` function. It creates the context object with the key data and delegates to ``buildInjectionChanges``. The received rules are iterated and applied.

## Adding Rule to Schematic

To get our new ``injectServiceIntoAppComponent`` rule called, we have to call it in its `index.ts`:

```TypeScript
[...]
export default function (options: MenuOptions): Rule {

    return (host: Tree, context: SchematicContext) => {

      [...]

      const rule = chain([
        branchAndMerge(chain([
          mergeWith(templateSource),
          addDeclarationToNgModule(options, options.export),
          injectServiceIntoAppComponent(options)
        ]))
      ]);

      return rule(host, context);
    }
}
```

## Testing the extended Schematic

To try the modified Schematic out, compile it and copy everything to the ``node_modules`` folder of an example application. As in the [former blog article](https://softwarearchitekt.at/post/2017/10/29/generating-custom-code-with-the-angular-cli-and-schematics.aspx), I've decided to copy it to ``node_modules/nav``. Please make sure to exclude the Schematic Collection's ``node_modules`` folder, so that there is no folder ``node_modules/nav/node_modules``. 

After this, switch to the example application's root and call the Schematic:

![Calling Schematic which generated component and registers it with the module](https://i.imgur.com/gu5ZuNA.png)

This not only created the ``SideMenu`` but also injects its service into the ``AppComponent``:

```TypeScript
import { Component } from '@angular/core';
import { OnChanges, OnInit } from '@angular/core';
import { SideMenuService } from './core/side-menu/side-menu.service';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {

  constructor(private sideMenuService: SideMenuService) {
      // sideMenuService.show = true;
  }

  title = 'app';
}
```