# Embedding multiple ESModules within the same HTML Document

First introduced in 2015, **ESModules** enable JavaScript to be written using _modular script architecture_ (*).

### From HTML
From within an HTML document, you can call an external **ESModule** via `<script src="/scripts/my-module.js" type="module">`

### From an ESModule
From an **ESModule** - either an external `.js` file or one embedded in an HTML Document - you may `import` another **ESModule** via two methods:

#### static `import`

`import { export1 as alias1, export2 as alias2 } from '/my-module.js';`

#### dynamic `import()`

```
let alias1, alias2;
import('/my-module.js').then((myModule) => {
  alias1 = myModule.export1;
  alias2 = myModule.export2;
});
```

### From a Script
You may also `import()` an **ESModule** from a (non-module) JavaScript - either an external `.js` file or one embedded in an HTML Document - but _only_ via dynamic `import()`.

______

## However...

_However_.

What you _can't_ do - at least not natively - is embed **ESModule** scripts into the same HTML document and have those **ESModules** `import` and `export` from each other, resulting in a _modular script architecture_ which runs entirely within a _same-document-environment_.

The following custom-built approach, involving `<script type="module/embedded" id="example-id">` and the asynchronous function `parseEmbeddedModule()` makes this possible.

________

## Example of ESModules importing and exporting within a single HTML Document

 - https://rouninmedia.github.io/Embedding-multiple-ESModules-within-the-same-HTML-Document/

________

## Three Steps to embedding multiple ESModules within the same HTML Document

#### Step One: Add two or more `<script type="module/embedded" id="example-id">` scripts to the HTML Document

**What is `type="module/embedded"`?**

**TLDR:** _It's unofficial._

**Longer explanation:** A long time ago, it was standard to include MIME type attributes in some elements:
 - `<script type="text/javascript">`
 - `<style type="text/css">`

As time progressed, these attributes went from _required_ (in **HTML 4.01**) to _optional_ (in **HTML5**) which led to their falling out of use:
 - `<script>`
 - `<style>`

In 2015, the `type` attribute made a comeback, this time to distinguish **ESModules** from conventional **JavaScript**:
 - `<script type="module">`

But, importantly, unlike the values from a decade earlier, the value `module` is _not_ a MIME type.

Instead, it's just an indication of what type of script we're dealing with.

Following this model, embedded modules look like this:

 - `<script type="module/embedded" id="example-id">`

Like `module`, even though it looks vaguely like it might be one, `module/embedded` is also not a MIME type.

Instead, it's just an indication of what type of script we're dealing with.

Two further things to note at this point:

1. Fairly importantly, `<script type="module/embedded" id="example-id">` is unofficial and it is unrecognised by the browser. Not being recognised by the browser means the element's contents will remain as unparsed `plaintext`. Since the element's contents are not parsed, they cannot be executed, either.

2. `<script type="module/embedded" id="example-id">` **must** include an `id` attribute. If it doesn't have one, it cannot be parsed by the `parseEmbeddedModule()` function.

_______

#### Step Two: Activate any Embedded Module via `parseEmbeddedModule()`

Once the **HTML Document** contains one or more `<script type="module/embedded" id="example-id">` elements, any of them may be explicitly activated via the asynchronous `parseEmbeddedModule()` function:

##### `parseEmbeddedModule()` function
```js
window.embeddedModules = {};

const parseEmbeddedModule = async (moduleName, exportNames = []) => {
  const embeddedModule = document.querySelector(`#${moduleName}`);
  if (embeddedModule?.getAttribute('type') !== 'module/embedded') throw new Error(`<script type="module/embedded" id="${moduleName}"> not found in document`);
  embeddedModules[moduleName] = (Object.hasOwn(embeddedModules, moduleName))
    ? embeddedModules[moduleName]
    : await import(URL.createObjectURL(new Blob([document.querySelector(`#${moduleName}`).textContent], {type: 'application/javascript'})));
  return (typeof exportNames === 'string') 
    ? embeddedModules[moduleName][exportNames]
    : ((Object.keys(embeddedModules[moduleName]).length === 1) && (Object.keys(embeddedModules[moduleName])[0] === 'default'))
      ? embeddedModules[moduleName]['default']
      : (exportNames.length > 0) 
        ? Object.fromEntries(Object.entries(embeddedModules[moduleName]).filter(([key, value]) => exportNames.includes(key)))
        : embeddedModules[moduleName];
}  
```
The function anticipates the following parameters:

 - a `moduleName` _(required)_
 - a list of one or more `exportNames` _(optional)_

If the function cannot find the **Embedded Module** by `id` in the same **HTML Document** it throws an error:

> `<script type="module/embedded" id="${moduleName}"> not found in document`

If the function can find the **Embedded Module**, it will check to see if the `exports` of that **Embedded Module** have already been imported into the `embeddedModules` object and, if not, it will _activate_ the **Embedded Module** and `import` them.

Once the `embeddedModules` object contains the **Embedded Module**'s `exports`, the function will return one, several or all of the exports, according to the `exportNames` parameter.

After returning one or more  **Embedded Module** `exports`, the function terminates.

**N.B.** Note that the function is _explicitly_ attached to `window` as a property, via the line `window.parseEmbeddedModule = parseEmbeddedModule;`.

This is intentional.

The assignment is required in order that _any type of script in the document_, be it:

 - `<script>`
 - `<script type="module">`
 - `<script type="module/embedded">`

will have access to the `parseEmbeddedModule()` function without it needing to be re-declared.

_______

#### Step Three: The Embedded Module's `exports` are now retrievable from the `embeddedModules` object

An **Embedded Module** only needs to be _activated once_ by `parseEmbeddedModule()`.

When the **Embedded Module** is first _activated_, two things will happen:
 1. the **Embedded Module**'s `exports` will be asynchronously imported (via dynamic `import()`) into the `embeddedModules` object;
 2. one, several or all `exports` will then be retrieved from the `embeddedModules` object. 

From that point onwards, whenever requested, the **Embedded Module**'s `exports` will be synchronously retrieved from the `embeddedModules` object, with no further `import()`-ing necessary.

Since `parseEmbeddedModule()` is asynchronous, we can either use it in conjunction with `.then()` or else we can use it with with `await`, within an `async` function.

##### TWO EMBEDDED MODULES
```js
<script type="module/embedded" id="testModule1">
  const displayParagraph_1 = (text) => {
    const paragraph = document.createElement('p');
    paragraph.textContent = `1: ${text}`;
    document.body.appendChild(paragraph);
  }

  const paragraph_1 = 'This is a paragraph export PLUS a function export';
  export {displayParagraph_1, paragraph_1};
</script>

<script type="module/embedded" id="testModule2">
  const displayParagraph_2 = (text) => {
    const paragraph = document.createElement('p');
    paragraph.textContent = `2: ${text}`;
    document.body.appendChild(paragraph);
  }

  const paragraph_2 = 'This is also a paragraph export PLUS a function export';
  // export default paragraph_2;
  export {displayParagraph_2, paragraph_2};
</script>

```

##### `parseEmbeddedModule()` FUNCTION
```js
window.embeddedModules = {};

const parseEmbeddedModule = async (moduleName, exportNames = []) => {
  const embeddedModule = document.querySelector(`#${moduleName}`);
  if (embeddedModule?.getAttribute('type') !== 'module/embedded') throw new Error(`<script type="module/embedded" id="${moduleName}"> not found in document`);
  embeddedModules[moduleName] = (Object.hasOwn(embeddedModules, moduleName))
    ? embeddedModules[moduleName]
    : await import(URL.createObjectURL(new Blob([document.querySelector(`#${moduleName}`).textContent], {type: 'application/javascript'})));
  return (typeof exportNames === 'string') 
    ? embeddedModules[moduleName][exportNames]
    : ((Object.keys(embeddedModules[moduleName]).length === 1) && (Object.keys(embeddedModules[moduleName])[0] === 'default'))
      ? embeddedModules[moduleName]['default']
      : (exportNames.length > 0) 
        ? Object.fromEntries(Object.entries(embeddedModules[moduleName]).filter(([key, value]) => exportNames.includes(key)))
        : embeddedModules[moduleName];
}
```

###### USING `parseEmbeddedModule()` WITH ASYNC / AWAIT
We can straightforwardly use the `parseEmbeddedModule()` function with `async` / `await` syntax:

```js
(async () => {

  // ALL EXPORTS (VIA DEFAULT PARAMETER)
  const test_A1 = await parseEmbeddedModule('testModule1'); // INITIALISES testModule_1 FOR ALL FUTURE EXPORTS UNTIL PAGE IS RELOADED
  const test_A2 = await parseEmbeddedModule('testModule2'); // INITIALISES testModule_2 FOR ALL FUTURE EXPORTS UNTIL PAGE IS RELOADED
  test_A1.displayParagraph_1(`test_A1: ${test_A1.paragraph_1}`);
  test_A2.displayParagraph_2(`test_A2: ${test_A2.paragraph_2}`);

  // SINGLE EXPORT (VIA STRING PARAMETER)
  const test_B1 = await parseEmbeddedModule('testModule1', 'paragraph_1');
  console.log(`test_B1: ${test_B1.paragraph_1}`);

  const test_B2 = await parseEmbeddedModule('testModule2', 'paragraph_2');
  console.log(`test_B2: ${test_B2.paragraph_2}`);

  // MULTIPLE EXPORTS (VIA ARRAY PARAMETER)
  const test_C1 = await parseEmbeddedModule('testModule1', ['displayParagraph_1', 'paragraph_1']);
  const test_C2 = await parseEmbeddedModule('testModule2', ['displayParagraph_2', 'paragraph_2']);
  test_C1.displayParagraph_1(`test_C1: ${test_C1.paragraph_1}`);
  test_C2.displayParagraph_2(`test_C2: ${test_C2.paragraph_2}`);

  // SYNCHRONOUS RETRIEVAL (ONLY POSSIBLE AFTER EMBEDDED MODULE ACTIVATION)
  const test_D1 = embeddedModules['testModule1'];
  const test_D2 = embeddedModules['testModule2'];
  test_D1.displayParagraph_1(`test_D1: ${test_D1.paragraph_1}`);
  test_D2.displayParagraph_2(`test_D2: ${test_D2.paragraph_2}`);

  // THIS WILL NOT WORK
  test_B1.displayParagraph_1(`test_B1: ${test_B1.paragraph_1}`); // test_B1.displayParagraph_1 is not a function
  test_B2.displayParagraph_2(`test_B2: ${test_B2.paragraph_2}`); // test_B2.displayParagraph_1 is not a function

})();
```

Let's take a closer look at the `async / await` code above to get a better idea about what's happening in each section:

 - **All Exports:** The easiest way to assign the exports of an **embedded module** to a variable is to deploy the `parseEmbeddedModule()` function with a single `string` argument. That argument represents the `id` of the **embedded module** about to be activated. Since there are no subsequent arguments detailing which named exports are to be assigned to the variable, _all exports_ from the **embedded module** will be assigned. From then on until page reload, _all named exports_ from the activated module will remain synchronously available _both_ from the global `embeddedModules` object _and_ from the variable to which the embedded module exports have been assigned.
 - **Single Export:** The _second_ easiest way to deploy the `parseEmbeddedModule()` function is to introduce, as the function's second argument, a `string` which indicates the name of _a single named export_ from the **embedded module**. In this case (as above) _all named exports_ from the activated module will become available from the global `embeddedModules` object but, in contrast, _only that specific named export_ will be available from the assignment variable. Note that this behaviour might be desirable in only a limited set of situations.
 - **Multiple Exports:** Thirdly, instead of deploying a `string`, we can introduce a second argument which is an `array`. Each array element represents the name of an export from the embedded module about to be imported. Each of these _named exports_ will become available from the variable to which they have been assigned, though (as above) _**all** named exports_ from the activated module will be available from the global `embeddedModules` object.
 - **Synchronous Retrieval:** Once an embedded module's named exports have been made synchronously available, it won't be necessary to run the `parseEmbeddedModule()` function again until page reload. In the code above we can see that ``test_D1.displayParagraph_1(`test_D1: ${test_D1.paragraph_1}`);`` works, even in the absence of the `parseEmbeddedModule()` function being invoked. Why? Because, from the point when ``const test_A1 = await parseEmbeddedModule('testModule1');`` (higher up the script) first returned a asynchronous response, the named exports `displayParagraph_1` and `paragraph_1` have been persistently available on a synchronous basis from `embeddedModules.testModule1`.
 - **This will not work:** We can try the same thing with ``test_B1.displayParagraph_1(`test_B1: ${test_B1.paragraph_1}`);`` but, this time, it won't work. Why not? We know that `testModule1` has already been parsed and that, consequently, `displayParagraph_1` and `paragraph_1` are both synchronously retrievable from `embeddedModules.testModule1`, so it's not that. Instead, it's simply the case that the variable `test_B1` was declared with the assignment `const test_B1 = await parseEmbeddedModule('testModule1', 'paragraph_1');` which means that, regardless of the fact that `displayParagraph_1` exists as a property of `embeddedModules.testModule1`, it _doesn't_ exist as a property of `test_B1`. In this case, we would need to run the synchronous line `test_B1.displayParagraph_1 = embeddedModules.testModule1.displayParagraph_1` _first_ and then ``test_B1.displayParagraph_1(`test_B1: ${test_B1.paragraph_1}`);`` will run as expected.

From this last observation we might conclude, that although we always have the option to declare one or several exports explicitly (by invoking the function with a second argument), it's usually the best approach to _omit_ the second argument from the `parseEmbeddedModule()` function, since we may then be sure that any variable which has an embedded module assigned to it will include access to _all_ the exports from that embedded module.

###### USING `parseEmbeddedModule()` WITH .THEN()
We can also perform the same series of `async / await` operations above using `.then()` syntax.

```js
// ALL EXPORTS (VIA DEFAULT PARAMETER)
parseEmbeddedModule('testModule1').then((module) => module.displayParagraph_1(`test_A1: ${module.paragraph_1}`));
parseEmbeddedModule('testModule2').then((module) => module.displayParagraph_2(`test_A2: ${module.paragraph_2}`));

// SINGLE EXPORT (VIA ARRAY PARAMETER)
parseEmbeddedModule('testModule1', ['paragraph_1']).then((module) => console.log(`test_B1: ${module.paragraph_1}`));
parseEmbeddedModule('testModule2', ['paragraph_2']).then((module) => console.log(`test_B2: ${module.paragraph_2}`));

// MULTIPLE EXPORTS (VIA ARRAY PARAMETER)
parseEmbeddedModule('testModule1', ['displayParagraph_1', 'paragraph_1']).then((module) => module.displayParagraph_1(`test_C1: ${module.paragraph_1}`));
parseEmbeddedModule('testModule2', ['displayParagraph_2', 'paragraph_2']).then((module) => module.displayParagraph_2(`test_C2: ${module.paragraph_2}`));

// SYNCHRONOUS RETRIEVAL (ONLY POSSIBLE AFTER EMBEDDED MODULE ACTIVATION, WHICH, IN THIS CASE, NECESSITATES A ZERO SECOND TIMEOUT)
setTimeout(() => {
  const test_D1 = embeddedModules['testModule1'];
  const test_D2 = embeddedModules['testModule2'];
  test_D1.displayParagraph_1(`test_D1: ${test_D1.paragraph_1}`);
  test_D2.displayParagraph_2(`test_D2: ${test_D2.paragraph_2}`);
}, 0);

// THIS WILL NOT WORK
parseEmbeddedModule('testModule1', ['paragraph_1']).then((module) => module.displayParagraph_1(`test_B1: ${module.paragraph_1}`));
parseEmbeddedModule('testModule2', ['paragraph_2']).then((module) => module.displayParagraph_2(`test_B2: ${module.paragraph_2}`));
```

This time it's clearer why the final pair of operations won't work. Evidently, if the second argument of the `parseEmbeddedModule()` function is `['paragraph_1']`, then the function's response object will only include a single property: `.paragraph_1`. It will lack the property `displayParagraph_1` - and it will have no way of accessing such a property.

Note, also, that in the examples immediately above, the single-export operations reference a single-element array as the second argument of the `parseEmbeddedModule()` function rather than a string. In the context of the second parameter, a single-element array is functionally identical to a string.

______

## Passing updatable objects around, between native ESModules

When we write architectures based on **ESModules**, every modification made to an exported `object` persists.

If, subsequently, the `object` is imported by another **ESModule**, it is the _modified_ `object` which is imported, not the initial object.

A simple example, using **ESModules**, which illustrates this:

#### initialise.js
```js
const userProfile = {
  loginTime: Date.now(),
  userType: standard
};

const greetUser = (userProfile) => console.log(`Hi, ${userProfile.userName}!`);

export { userProfile, greetUser }
```

#### add-username.js
```js
import { userProfile } from '/initialise.js';

userProfile.userName = 'Aimée';
```

#### HTML DOCUMENT
```html

  <script type="module">
    import { userProfile, greetUser } from '/initialise.js';

    greetUser(userProfile); // 'Hi undefined!'
  </script>


  <script type="module">
    import '/add-username.js';
  </script>


  <script type="module">
    import { userProfile, greetUser } from '/initialise.js';

    greetUser(userProfile); // 'Hi Aimée!'
  </script>

```

In the HTML document we can see that `initialise.js` is imported and run twice: once before `add-username.js` is imported and once after.

We can also see that the first time `initialise.js` is imported (_before_ `add-username.js` is imported), the imported `greetUser` function _cannot find_ the `userName` property in the imported `userProfile` object. This is because the object in question does not yet have such a property.

The second time `initialise.js` is imported (_after_ `add-username.js` is imported), although we know that `userProfile` is never given a `userName` property in `initialise.js`, suddenly it seems the property _does_ exist in the object and now the `greetUser` function works perfectly.

What we are observing is that importing and running `add-username.js` means that the `userProfile` `export` is imported and modified and, _critically_, these modifications then **persist**.

We should want _Embedded Modules_ to work the same way - and they do.

## Passing updatable objects around, between _embedded modules_

Here is the same example as above showing that when we write the same architectures using **Embedded Modules**, every modification made to an exported `object` persists, just as it did before.

That is, when an object is imported and then modified, any subsequent import of the `object` by another **Embedded Module** will import the _modified_ `object`, not the initial object.

______

## Conclusion

That's it.

We can now add **embedded modules** to a single document using `<script type="module/embedded">`.

And we can `import` the `exports` from those **embedded modules** into one or more:

 - `<script>`
 - `<script type="module">`
 - `<script type="module/embedded">`

elements in the same document.

__________

(*) Though it's also true that, long before **ESModules** were officially introduced, unofficial module-like design patterns already existed:

 - **Module Pattern (2007):** http://web.archive.org/web/20140903025035/http://yuiblog.com/blog/2007/06/12/module-pattern/
 - **Revealing Module Pattern (2007):** https://christianheilmann.com/2007/08/22/again-with-the-module-pattern-reveal-something-to-the-world/
 - **Definitive Module Pattern (2014):** http://inniepress.blogspot.com/2014/07/definitive-module-pattern.html
