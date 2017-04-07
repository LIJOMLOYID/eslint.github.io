---
title: Working with Rules
layout: doc
---
<!-- Note: No pull requests accepted for this file. See README.md in the root directory for details. -->

# Working with Rules

**Note:** This page covers the most recent rule format for ESLint >= 3.0.0. There is also a [deprecated rule format](./working-with-rules-deprecated).

Each rule in ESLint has two files named with its identifier (for example, `no-extra-semi`).

* in the `lib/rules` directory: a source file (for example, `no-extra-semi.js`)
* in the `tests/lib/rules` directory: a test file (for example, `no-extra-semi.js`)

**Important:** If you submit a **core** rule to the ESLint repository, you **must** follow some conventions explained below.

Here is the basic format of the source file for a rule:

```js
/**
 * @fileoverview Rule to disallow unnecessary semicolons
 * @author Nicholas C. Zakas
 */

"use strict";

//------------------------------------------------------------------------------
// Rule Definition
//------------------------------------------------------------------------------

module.exports = {
    meta: {
        docs: {
            description: "disallow unnecessary semicolons",
            category: "Possible Errors",
            recommended: true
        },
        fixable: "code",
        schema: [] // no options
    },
    create: function(context) {
        return {
            // callback functions
        };
    }
};
```

## Rule Basics

The source file for a rule exports an object with the following properties.

`meta` (object) contains metadata for the rule:

* `docs` (object) is required for core rules of ESLint:

    * `description` (string) provides the short description of the rule in the [rules index](../rules/)
    * `category` (string) specifies the heading under which the rule is listed in the [rules index](../rules/)
    * `recommended` (boolean) is whether the `"extends": "eslint:recommended"` property in a [configuration file](../user-guide/configuring#extending-configuration-files) enables the rule

    In a custom rule or plugin, you can omit `docs` or include any properties that you need in it.

* `fixable` (string) is either `"code"` or `"whitespace"` if the `--fix` option on the [command line](../user-guide/command-line-interface#fix) automatically fixes problems reported by the rule

    **Important:** Without the `fixable` property, ESLint does not [apply fixes](#applying-fixes) even if the rule implements `fix` functions. Omit the `fixable` property if the rule is not fixable.

* `schema` (array) specifies the [options](#options-schemas) so ESLint can prevent invalid [rule configurations](../user-guide/configuring#configuring-rules)

* `deprecated` (boolean) indicates whether the rule has been deprecated.  You may omit the `deprecated` property if the rule has not been deprecated.

`create` (function) returns an object with methods that ESLint calls to "visit" nodes while traversing the abstract syntax tree (AST as defined by [ESTree](https://github.com/estree/estree)) of JavaScript code:

* if a key is a node type or a [selector](./selectors), ESLint calls that **visitor** function while going **down** the tree
* if a key is a node type or a [selector](./selectors) plus `:exit`, ESLint calls that **visitor** function while going **up** the tree
* if a key is an event name, ESLint calls that **handler** function for [code path analysis](./code-path-analysis)

A rule can use the current node and its surrounding tree to report or fix problems.

Here are methods for the [array-callback-return](../rules/array-callback-return) rule:

```js
function checkLastSegment (node) {
    // report problem for function if last code path segment is reachable
}

module.exports = {
    meta: { ... },
    create: function(context) {
        // declare the state of the rule
        return {
            ReturnStatement: function(node) {
                // at a ReturnStatement node while going down
            },
            // at a function expression node while going up:
            "FunctionExpression:exit": checkLastSegment,
            "ArrowFunctionExpression:exit": checkLastSegment,
            onCodePathStart: function (codePath, node) {
                // at the start of analyzing a code path
            },
            onCodePathEnd: function(codePath, node) {
                // at the end of analyzing a code path
            }
        };
    }
};
```

## The Context Object

The `context` object contains additional functionality that is helpful for rules to do their jobs. As the name implies, the `context` object contains information that is relevant to the context of the rule. The `context` object has the following properties:

* `parserOptions` - the parser options configured for this run (more details [here](../user-guide/configuring#specifying-parser-options)).
* `id` - the rule ID.
* `options` - an array of rule options.
* `settings` - the `settings` from configuration.
* `parserPath` - the full path to the `parser` from configuration.

Additionally, the `context` object has the following methods:

* `getAncestors()` - returns an array of ancestor nodes based on the current traversal.
* `getDeclaredVariables(node)` - returns the declared variables on the given node.
* `getFilename()` - returns the filename associated with the source.
* `getScope()` - returns the current scope.
* `getSourceCode()` - returns a `SourceCode` object that you can use to work with the source that was passed to ESLint
* `markVariableAsUsed(name)` - marks the named variable in scope as used. This affects the [no-unused-vars](../rules/no-unused-vars) rule.
* `report(descriptor)` - reports a problem in the code.

**Note:** Earlier versions of ESLint supported additional methods on the `context` object. Those methods were removed in the new format and should not be relied upon.

### context.report()

The main method you'll use is `context.report()`, which publishes a warning or error (depending on the configuration being used). This method accepts a single argument, which is an object containing the following properties:

* `message` - the problem message.
* `node` - (optional)  the AST node related to the problem. If present and `loc` is not specified, then the starting location of the node is used as the location of the problem.
* `loc` - (optional) an object specifying the location of the problem. If both `loc` and `node` are specified, then the location is used from `loc` instead of `node`.
    * `start` - An object of the start location.
        * `line` - the 1-based line number at which the problem occurred.
        * `column` - the 0-based column number at which the problem occurred.
    * `end` - An object of the end location.
        * `line` - the 1-based line number at which the problem occurred.
        * `column` - the 0-based column number at which the problem occurred.
* `data` - (optional) placeholder data for `message`.
* `fix` - (optional) a function that applies a fix to resolve the problem.

Note that at least one of `node` or `loc` is required.

The simplest example is to use just `node` and `message`:

```js
context.report({
    node: node,
    message: "Unexpected identifier"
});
```

The node contains all of the information necessary to figure out the line and column number of the offending text as well the source text representing the node.

You can also use placeholders in the message and provide `data`:

```js
{% raw %}
context.report({
    node: node,
    message: "Unexpected identifier: {{ identifier }}",
    data: {
        identifier: node.name
    }
});
{% endraw %}
```

Note that leading and trailing whitespace is optional in message parameters.

The node contains all of the information necessary to figure out the line and column number of the offending text as well the source text representing the node.

### Applying Fixes

If you'd like ESLint to attempt to fix the problem you're reporting, you can do so by specifying the `fix` function when using `context.report()`. The `fix` function receives a single argument, a `fixer` object, that you can use to apply a fix. For example:

```js
context.report({
    node: node,
    message: "Missing semicolon",
    fix: function(fixer) {
        return fixer.insertTextAfter(node, ";");
    }
});
```

Here, the `fix()` function is used to insert a semicolon after the node. Note that a fix is not immediately applied, and may not be applied at all if there are conflicts with other fixes. After applying fixes, ESLint will run all of the enabled rules again on the fixed code, potentially applying more fixes. This process will repeat up to 10 times, or until no more fixable problems are found. Afterwards, any remaining problems will be reported as usual.

**Important:** Unless the rule [exports](#rule-basics) the `meta.fixable` property, ESLint does not apply fixes even if the rule implements `fix` functions.

The `fixer` object has the following methods:

* `insertTextAfter(nodeOrToken, text)` - inserts text after the given node or token
* `insertTextAfterRange(range, text)` - inserts text after the given range
* `insertTextBefore(nodeOrToken, text)` - inserts text before the given node or token
* `insertTextBeforeRange(range, text)` - inserts text before the given range
* `remove(nodeOrToken)` - removes the given node or token
* `removeRange(range)` - removes text in the given range
* `replaceText(nodeOrToken, text)` - replaces the text in the given node or token
* `replaceTextRange(range, text)` - replaces the text in the given range

Best practices for fixes:

1. Avoid any fixes that could change the runtime behavior of code and cause it to stop working.
1. Make fixes as small as possible. Fixes that are unnecessarily large could conflict with other fixes, and prevent them from being applied.
1. Only make one fix per message. This is enforced because you must return the result of the fixer operation from `fix()`.
1. Since all rules are run again after the initial round of fixes is applied, it's not necessary for a rule to check whether the code style of a fix will cause errors to be reported by another rule.
    * For example, suppose a fixer would like to surround an object key with quotes, but it's not sure whether the user would prefer single or double quotes.

        ```js
        ({ foo : 1 })

        // should get fixed to either

        ({ 'foo': 1 })

        // or

        ({ "foo": 1 })
        ```
    * This fixer can just select a quote type arbitrarily. If it guesses wrong, the resulting code will be automatically reported and fixed by the [`quotes`](/docs/rules/quotes) rule.

### context.options

Some rules require options in order to function correctly. These options appear in configuration (`.eslintrc`, command line, or in comments). For example:

```json
{
    "quotes": ["error", "double"]
}
```

The `quotes` rule in this example has one option, `"double"` (the `error` is the error level). You can retrieve the options for a rule by using `context.options`, which is an array containing every configured option for the rule. In this case, `context.options[0]` would contain `"double"`:

```js
module.exports = {
    create: function(context) {
        var isDouble = (context.options[0] === "double");

        // ...
    }
};
```

Since `context.options` is just an array, you can use it to determine how many options have been passed as well as retrieving the actual options themselves. Keep in mind that the error level is not part of `context.options`, as the error level cannot be known or modified from inside a rule.

When using options, make sure that your rule has some logic defaults in case the options are not provided.

### context.getSourceCode()

The `SourceCode` object is the main object for getting more information about the source code being linted. You can retrieve the `SourceCode` object at any time by using the `getSourceCode()` method:

```js
module.exports = {
    create: function(context) {
        var sourceCode = context.getSourceCode();

        // ...
    }
};
```

Once you have an instance of `SourceCode`, you can use the methods on it to work with the code:

* `getText(node)` - returns the source code for the given node. Omit `node` to get the whole source.
* `getAllComments()` - returns an array of all comments in the source.
* `getComments(node)` - returns the leading and trailing comments arrays for the given node.
* `getJSDocComment(node)` - returns the JSDoc comment for a given node or `null` if there is none.
* `isSpaceBetweenTokens(first, second)` - returns true if there is a whitespace character between the two tokens.
* `getFirstToken(node, skipOptions)` - returns the first token representing the given node.
* `getFirstTokens(node, countOptions)` - returns the first `count` tokens representing the given node.
* `getLastToken(node, skipOptions)` - returns the last token representing the given node.
* `getLastTokens(node, countOptions)` - returns the last `count` tokens representing the given node.
* `getTokenAfter(nodeOrToken, skipOptions)` - returns the first token after the given node or token.
* `getTokensAfter(nodeOrToken, countOptions)` - returns `count` tokens after the given node or token.
* `getTokenBefore(nodeOrToken, skipOptions)` - returns the first token before the given node or token.
* `getTokensBefore(nodeOrToken, countOptions)` - returns `count` tokens before the given node or token.
* `getFirstTokenBetween(nodeOrToken1, nodeOrToken2, skipOptions)` - returns the first token between two nodes or tokens.
* `getFirstTokensBetween(nodeOrToken1, nodeOrToken2, countOptions)` - returns the first `count` tokens between two nodes or tokens.
* `getLastTokenBetween(nodeOrToken1, nodeOrToken2, skipOptions)` - returns the last token between two nodes or tokens.
* `getLastTokensBetween(nodeOrToken1, nodeOrToken2, countOptions)` - returns the last `count` tokens between two nodes or tokens.
* `getTokens(node)` - returns all tokens for the given node.
* `getTokensBetween(nodeOrToken1, nodeOrToken2)` - returns all tokens between two nodes.
* `getTokenByRangeStart(index, rangeOptions)` - returns the token whose range starts at the given index in the source.
* `getNodeByRangeIndex(index)` - returns the deepest node in the AST containing the given source index.
* `getLocFromIndex(index)` - returns an object with `line` and `column` properties, corresponding to the location of the given source index. `line` is 1-based and `column` is 0-based.
* `getIndexFromLoc(loc)` - returns the index of a given location in the source code, where `loc` is an object with a 1-based `line` key and a 0-based `column` key.

> `skipOptions` is an object which has 3 properties; `skip`, `includeComments`, and `filter`. Default is `{skip: 0, includeComments: false, filter: null}`.
> - `skip` is a positive integer, the number of skipping tokens. If `filter` option is given at the same time, it doesn't count filtered tokens as skipped.
> - `includeComments` is a boolean value, the flag to include comment tokens into the result.
> - `filter` is a function which gets a token as the first argument, if the function returns `false` then the result excludes the token.
>
> `countOptions` is an object which has 3 properties; `count`, `includeComments`, and `filter`. Default is `{count: 0, includeComments: false, filter: null}`.
> - `count` is a positive integer, the maximum number of returning tokens.
> - `includeComments` is a boolean value, the flag to include comment tokens into the result.
> - `filter` is a function which gets a token as the first argument, if the function returns `false` then the result excludes the token.
>
> `rangeOptions` is an object which has 1 property: `includeComments`.
> - `includeComments` is a boolean value, the flag to include comment tokens into the result.

There are also some properties you can access:

* `hasBOM` - the flag to indicate whether or not the source code has Unicode BOM.
* `text` - the full text of the code being linted. Unicode BOM has been stripped from this text.
* `ast` - the `Program` node of the AST for the code being linted.
* `lines` - an array of lines, split according to the specification's definition of line breaks.

You should use a `SourceCode` object whenever you need to get more information about the code being linted.

### Options Schemas

Rules may export a `schema` property, which is a [JSON schema](http://json-schema.org/) format description of a rule's options which will be used by ESLint to validate configuration options and prevent invalid or unexpected inputs before they are passed to the rule in `context.options`.

There are two formats for a rule's exported `schema`. The first is a full JSON Schema object describing all possible options the rule accepts, including the rule's error level as the first argument and any optional arguments thereafter.

However, to simplify schema creation, rules may also export an array of schemas for each optional positional argument, and ESLint will automatically validate the required error level first. For example, the `yoda` rule accepts a primary mode argument, as well as an extra options object with named properties.

```js
// "yoda": [2, "never", { "exceptRange": true }]
module.exports = {
    meta: {
        schema: [
            {
                "enum": ["always", "never"]
            },
            {
                "type": "object",
                "properties": {
                    "exceptRange": {
                        "type": "boolean"
                    }
                },
                "additionalProperties": false
            }
        ];
    },
};
```

In the preceding example, the error level is assumed to be the first argument. It is followed by the first optional argument, a string which may be either `"always"` or `"never"`. The final optional argument is an object, which may have a Boolean property named `exceptRange`.

To learn more about JSON Schema, we recommend looking at some [examples](http://json-schema.org/examples.html) to start, and also reading [Understanding JSON Schema](http://spacetelescope.github.io/understanding-json-schema/) (a free ebook).

### Getting the Source

If your rule needs to get the actual JavaScript source to work with, then use the `sourceCode.getText()` method. This method works as follows:

```js

// get all source
var source = sourceCode.getText();

// get source for just this AST node
var nodeSource = sourceCode.getText(node);

// get source for AST node plus previous two characters
var nodeSourceWithPrev = sourceCode.getText(node, 2);

// get source for AST node plus following two characters
var nodeSourceWithFollowing = sourceCode.getText(node, 0, 2);
```

In this way, you can look for patterns in the JavaScript text itself when the AST isn't providing the appropriate data (such as location of commas, semicolons, parentheses, etc.).

### Accessing Comments

While comments are not technically part of the AST, ESLint provides a few ways for rules to access them:

#### sourceCode.getAllComments()

This method returns an array of all the comments found in the program. This is useful for rules that need to check all comments regardless of location.

#### sourceCode.getComments(node)

This method returns comments for a specific node in the form of an object containing arrays of "leading" (occurring before the given node) and "trailing" (occurring after the given node) comment tokens. This is useful for rules that need to check comments around a given node.

Keep in mind that the results of this method are calculated on demand.

#### Token traversal methods

Finally, comments can be accessed through many of `sourceCode`'s methods using the `includeComments` option.

### Accessing Shebangs

Shebangs are represented by tokens of type `"Shebang"`. They are treated as comments and can be accessed by the methods outlined above.

### Accessing Code Paths

ESLint analyzes code paths while traversing AST.
You can access that code path objects with five events related to code paths.

[details here](./code-path-analysis)

## Rule Unit Tests

Each rule must have a set of unit tests submitted with it to be accepted. The test file is named the same as the source file but lives in `tests/lib/`. For example, if your rule source file is `lib/rules/foo.js` then your test file should be `tests/lib/rules/foo.js`.

For your rule, be sure to test:

1. All instances that should be flagged as warnings.
1. At least one pattern that should **not** be flagged as a warning.

The basic pattern for a rule unit test file is:

```js
/**
 * @fileoverview Tests for no-with rule.
 * @author Nicholas C. Zakas
 */

"use strict";

//------------------------------------------------------------------------------
// Requirements
//------------------------------------------------------------------------------

var rule = require("../../../lib/rules/no-with"),
    RuleTester = require("../../../lib/testers/rule-tester");

//------------------------------------------------------------------------------
// Tests
//------------------------------------------------------------------------------

var ruleTester = new RuleTester();
ruleTester.run("no-with", rule, {
    valid: [
        "foo.bar()"
    ],
    invalid: [
        {
            code: "with(foo) { bar() }",
            errors: [{ message: "Unexpected use of 'with' statement.", type: "WithStatement"}]
        }
    ]
});
```

Be sure to replace the value of `"no-with"` with your rule's ID. There are plenty of examples in the `tests/lib/rules/` directory.

### Valid Code

Each valid case can be either a string or an object. The object form is used when you need to specify additional global variables or arguments for the rule. For example, the following defines `window` as a global variable for code that should not trigger the rule being tested:

```js
valid: [
    {
        code: "window.alert()",
        globals: [ "window" ]
    }
]
```

You can also pass options to the rule (if it accepts them). These arguments are equivalent to how people can configure rules in their `.eslintrc` file. For example:

```js
valid: [
    {
        code: "var msg = 'Hello';",
        options: [ "single" ]
    }
]
```

The `options` property must be an array of options. This gets passed through to `context.options` in the rule.

### Invalid Code

Each invalid case must be an object containing the code to test and at least one message that is produced by the rule. The `errors` key specifies an array of objects, each containing a message (your rule may trigger multiple messages for the same code). You should also specify the type of AST node you expect to receive back using the `type` key. The AST node should represent the actual spot in the code where there is a problem. For example:

```js
invalid: [
    {
        code: "function doSomething() { var f; if (true) { var build = true; } f = build; }",
        errors: [
            { message: "build used outside of binding context.", type: "Identifier" }
        ]
    }
]
```

In this case, the message is specific to the variable being used and the AST node type is `Identifier`.

You can also check that the rule returns the correct line and column numbers for the message by adding `line` and `column` properties as needed (both are optional, but highly recommend):

```js
invalid: [
    {
        code: "function doSomething() { var f; if (true) { var build = true; } f = build; }",
        errors: [
            {
                message: "build used outside of binding context.",
                type: "Identifier",
                line: 1,
                column: 68
            }
        ]
    }
]
```

The test fails if the line or column reported by the rule doesn't match the options specified in the test.

Similar to the valid cases, you can also specify `options` to be passed to the rule:

```js
invalid: [
    {
        code: "function doSomething() { var f; if (true) { var build = true; } f = build; }",
        options: [ "double" ],
        errors: [
            { message: "build used outside of binding context.", type: "Identifier" }
        ]
    }
]
```

For simpler cases where the only thing that really matters is the error message, you can also specify any `errors` as strings. You can also have some strings and some objects, if you like.

```js
invalid: [
    {
        code: "'single quotes'",
        options: ["double"],
        errors: ["Strings must use doublequote."]
    }
]
```

### Specifying Globals

If your rule relies on globals to be specified, you can provide global variable declarations by using the `globals` property. For example:

```js
valid: [
    {
        code: "for (x of a) doSomething();",
        globals: { window: true }
    }
]
```

The same works on invalid tests:

```js
invalid: [
    {
        code: "'single quotes'",
        globals: { window: true },
        errors: ["Strings must use doublequote."]
    }
]
```

### Specifying Settings

If your rule relies on `context.settings` to be specified, you can provide those settings by using the `settings` property. For example:

```js
valid: [
    {
        code: "for (x of a) doSomething();",
        settings: { message: "hi" }
    }
]
```

The same works on invalid tests:

```js
invalid: [
    {
        code: "'single quotes'",
        settings: { message: "hi" },
        errors: ["Strings must use doublequote."]
    }
]
```

You can then access `context.settings.message` inside of the rule.

### Specifying Filename

If your rule relies on `context.getFilename()` to be specified, you can provide the filename by using the `filename` property. For example:

```js
valid: [
    {
        code: "for (x of a) doSomething();",
        filename: "foo/bar.js"
    }
]
```

The same works on invalid tests:

```js
invalid: [
    {
        code: "'single quotes'",
        filename: "foo/bar.js",
        errors: ["Strings must use doublequote."]
    }
]
```

You can then access `context.getFilename()` inside of the rule.

### Specifying Parser and Parser Options

Some tests require that a certain parser configuration must be used. This can be specified in test specifications via the `parser` and `parserOptions` properties. While the following examples show usage in `valid` tests, you can use the same options in `invalid` tests as well.

For example, to set `ecmaVersion` to 6 (in order to use constructs like `for-of`):

```js
valid: [
    {
        code: "for (x of a) doSomething();",
        parserOptions: { ecmaVersion: 6 }
    }
]
```

If you are working with ES6 modules:

```js
valid: [
    {
        code: "export default function () {};",
        parserOptions: { ecmaVersion: 6, sourceType: "module" }
    }
]
```

For non-version specific features such as JSX:

```js
valid: [
    {
        code: "var foo = <div>{bar}</div>",
        parserOptions: { ecmaFeatures: { jsx: true } }
    }
]
```

To use a different parser:

```js
valid: [
    {
        code: "var foo = <div>{bar}</div>",
        parser: "my-custom-parser"
    }
]
```

The options available and the expected syntax for `parserOptions` is the same as those used in [configuration](../user-guide/configuring#specifying-parser-options).

## Performance Testing

To keep the linting process efficient and unobtrusive, it is useful to verify the performance impact of new rules or modifications to existing rules.

### Overall Performance

The `npm run perf` command gives a high-level overview of ESLint running time with default rules (`eslint:recommended`) enabled.

```bash
$ git checkout master
Switched to branch 'master'

$ npm run perf
CPU Speed is 2200 with multiplier 7500000
Performance Run #1:  1394.689313ms
Performance Run #2:  1423.295351ms
Performance Run #3:  1385.09515ms
Performance Run #4:  1382.406982ms
Performance Run #5:  1409.68566ms
Performance budget ok:  1394.689313ms (limit: 3409.090909090909ms)

$ git checkout my-rule-branch
Switched to branch 'my-rule-branch'

$ npm run perf
CPU Speed is 2200 with multiplier 7500000
Performance Run #1:  1443.736547ms
Performance Run #2:  1419.193291ms
Performance Run #3:  1436.018228ms
Performance Run #4:  1473.605485ms
Performance Run #5:  1457.455283ms
Performance budget ok:  1443.736547ms (limit: 3409.090909090909ms)
```

### Per-rule Performance

ESLint has a built-in method to track performance of individual rules. Setting the `TIMING` environment variable will trigger the display, upon linting completion, of the ten longest-running rules, along with their individual running time and relative performance impact as a percentage of total rule processing time.

```bash
$ TIMING=1 eslint lib
Rule                    | Time (ms) | Relative
:-----------------------|----------:|--------:
no-multi-spaces         |    52.472 |     6.1%
camelcase               |    48.684 |     5.7%
no-irregular-whitespace |    43.847 |     5.1%
valid-jsdoc             |    40.346 |     4.7%
handle-callback-err     |    39.153 |     4.6%
space-infix-ops         |    35.444 |     4.1%
no-undefined            |    25.693 |     3.0%
no-shadow               |    22.759 |     2.7%
no-empty-class          |    21.976 |     2.6%
semi                    |    19.359 |     2.3%
```

To test one rule explicitly, combine the `--no-eslintrc`, and `--rule` options:

```bash
$ TIMING=1 eslint --no-eslintrc --rule "quotes: [2, 'double']" lib
Rule   | Time (ms) | Relative
:------|----------:|--------:
quotes |    18.066 |   100.0%
```

## Rule Naming Conventions

The rule naming conventions for ESLint are fairly simple:

* If your rule is disallowing something, prefix it with `no-` such as `no-eval` for disallowing `eval()` and `no-debugger` for disallowing `debugger`.
* If your rule is enforcing the inclusion of something, use a short name without a special prefix.
* Keep your rule names as short as possible, use abbreviations where appropriate, and no more than four words.
* Use dashes between words.

## Runtime Rules

The thing that makes ESLint different from other linters is the ability to define custom rules at runtime. This is perfect for rules that are specific to your project or company and wouldn't make sense for ESLint to ship with. With runtime rules, you don't have to wait for the next version of ESLint or be disappointed that your rule isn't general enough to apply to the larger JavaScript community, just write your rules and include them at runtime.

Runtime rules are written in the same format as all other rules. Create your rule as you would any other and then follow these steps:

1. Place all of your runtime rules in the same directory (i.e., `eslint_rules`).
2. Create a [configuration file](../user-guide/configuring) and specify your rule ID error level under the `rules` key. Your rule will not run unless it has a value of `1` or `2` in the configuration file.
3. Run the [command line interface](../user-guide/command-line-interface) using the `--rulesdir` option to specify the location of your runtime rules.