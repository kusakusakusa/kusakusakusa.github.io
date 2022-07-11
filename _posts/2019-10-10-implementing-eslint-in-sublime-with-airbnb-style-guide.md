---
title:  "Implementing Eslint In Sublime With Airbnb Style Guide"
date:   2019-10-10 09:00:00
permalink: implementing-eslint-in-sublime-with-airbnb-style-guide
---

This is a documentation on how to setup eslinting on sublime text editor, bootstrapped with using the style guide set up by Airbnb.

## Why Is Eslinting Necessary

Linting your javascript code catches syntax errors, and possibly some runtime errors, while you are coding. It helps you debug faster, putting things like missing ; or wrong closures out of the way from the work. No more rolling your eyes and reducing your lifespan over things that shouldn’t matter.

🙄

It will also be helpful if we are working together with other developers. It helps to keep the flavor of the `JavaScript` code same across the code bases, which can speed up development work and make it enjoyable at the very least.

Ok, maybe a teeny weeny bit better since we are talking about JavaScript here.

INSERT JAVASCRIPT CRYING MEME

## Why Airbnb Style Guide

Linting can also be a good teacher. We can follow style guides, or more technically `eslint` configurations, setup by other senpais with telling experience or hailing from reputable tech companies, and when we write codes that do no adhere to  what they deem as best practice, we can learn why and change accordingly.

Google, for instance, has it own set of [JavaScript style guide](https://github.com/google/eslint-config-google), which comes with a shareable `eslint` config that we can just plug and play into our codebase.

However, I would personally suggest [Airbnb's style guide](https://github.com/airbnb/javascript) because of its well-documented reasons for implementing each rule. In its style guide page, it states the reason for implementing certain rules as well as give examples of good and bad coding examples. Here is an image to better explain this point.

This is an accumulation of a wealth of knowledge from brilliant programmers that is made easily accessible to us.

Sometimes, we may even learn some quirks of `JavaScript` that we never knew existed because we have not experienced the problem before. It is like time traveling to the future by leveraging on the past lessons of those before us.

## Installing eslinting

Start off by installing the packages required for eslinting with Airbnb style.

```bash
npm i -D eslint 
    babel-eslint 
    eslint-config-airbnb 
    eslint-plugin-import 
    eslint-plugin-jsx-a11y 
    eslint-plugin-react 
    eslint-plugin-react-hooks
```

I will attempt to explain what each line does.

### -D flag

The `-D` flag installs the packages as development dependencies for the project at hand. It makes sense to install it under the project instead of globally because the same configurations can be shared to other developers working on the same project. And install it only as a development dependency as it is only required during development work.

### eslint

The eslint package is the main package that will handle the linting on Vanilla JavaScript and Vanilla JavaScript only. It is crucial, for the case of JavaScript, to understand the significance of Vanilla JavaScript, and that will require a bit of a history lesson.

### babel-eslint

`JavaScript` is a fairly senior language. It is one of the pioneers of computing languages, which meant that it did inevitably made some bad mistakes in its syntax design. Over the years, people, or should I say geniuses, are unhappy about it and have come up with more efficient ways to write it.
CoffeeScript is one example. To define a `JavaScript` function that looks like this:

```javascript
var greet = function(name) {
  return console.log(Hello, ${name});
};
```

`CoffeeScript` and its gang of caffeine addicts have decided to shrink the amount of code require with the use of indentations. The same definition can be written in CoffeeScript like this:

```coffee
greet = (name) ->
  console.log `Hello, ${name}`
```

`TypeScript` is another example and their side of the camp stresses the need for JavaScript variables to be type safe, the lack of which has caused much of the JavaScript errors throughout its history. The same function defined in TypeScript looks like this:

```ts
var greet = function (name: string): void {
  console.log(`Hello ${name}`);
}
```

The official standard for `JavaScript`, however, is the ECMAScript specification, or ES for short (hope that answer the naming of the eslint package). And its official compiler is Babel. The specification has undergone multiple improvements over the years and new standards were iterated. Babel has also evolved with it. The same function can be written in Babel as such:

```javascript
var greet = (name) => {
  console.log(`Hello ${name}`);
}
```

All these various compilers mean that we need a different set of linting depending on the compiler you are using to catch the corresponding syntax errors. Airbnb style guide uses Babel and follows the official modern JavaScript syntax. Hence we are installing the `babel-eslint` package.

### eslint-config-airbnb

This package  consist of all the rules that the engineers in Airbnb have deemed as best practices and which their company follows.

### eslint-plugin-import

Next is the `eslint-plugin-import` package which supports linting of file imports using `ECMAScript`. While the previous mentioned packages watch out for syntax errors in your code, this packages looks out for erroneous file imports. These errors may be due forgetting to export modules that exist in another file, or a wrong spelling in the file name to be imported.

## The Other Packages

The other packages are react specific. Airbnb use react as the main `JavaScript` framework for their front end. Hence their default linting config requires the remaining packages to work, namely `eslint-plugin-react`, `eslint-plugin-react-hooks`, and `eslint-plugin-jsx-a11y`. Without them, you will get missing dependency errors.

Note that it also requires the eslint and eslint-plugin-import packages to work, but since its not specific to react and is a good-to-have tool for development work, I am explained a little more about them.

If you are not using react, but still want to use Airbnb style guide, you will be interested in their base configurations in the `eslint-config-airbnb` package.

### The .eslintrc

What we have done so far is only downloaded the linting packages. To utilise them, some configurations need to be setup, and I will be setting it up using `.eslintrc`. There are a number different ways to setup the configurations.

Write the `.eslintrc` as such:

```
{
  "parser": "babel-eslint",
  "extends": "airbnb",
  "rules": {
    "camelcase": "off",
  }
}
```

The parser option specifies that we are going to write the latest `ECMAScript` syntax and using `Babel` as the compiler, as what the rules in the Airbnb configuration file is expecting. More parser options can be found here.

The extends options tells the linter to use the rules setup by the Airbnb configuration. The rules will be empty if this is not specified, and you will be essentially writing code with the recommended setting in `babel-eslint` package.

The rules options allows you to overwrite rules that might not fit your workflow. I added the example for ignoring warnings on non camel case variables which I am using right now. I do not follow the camel case practice when naming `JavaScript` variables because I am working with a `Ruby On Rails` backend. The `Rails` framework advocates snake cased variable naming which is different from that of `JavaScript`. Hence, I decide to keep the same casing for the variable naming so that it is easy to receive data from the API responses, as well as to package requests to send to the backend..

### Text Editor Linter

Lastly, we need a package on the text editor you are using to read the `eslint` packages and configurations to highlight the relevant syntax errors accordingly. For sublime text editor, you need to install these packages.

```
SublimeLinter
SublimeLinter-eslint
```

The steps to install sublime text packages can be easily achieved with the `Sublime Package Control`.

### Restarting Sublime Text

The last step is to restart your Sublime Text editor. Open a file with the `js` extension, and start writing some erroneous code to see the linting in effect!

