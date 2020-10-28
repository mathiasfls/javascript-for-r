# Packer Advanced {#packer-adv}

We're about to cover slightly more advanced uses of NPM and webpack with R using packer. These involve using an NPM dependency to develop a widget and use [Vue.js](https://vuejs.org/) to power the front-end of a shiny application.

Those will make for more concrete cases to bring webpack into your workflow, and also enable explaining more advanced topics only thus far briefly touched upon, such as transpiling.

## Widgets {#packer-adv-widgets}

The widget scaffold, like all other scaffolds, must be run from within the root of a package. To demonstrate we'll write a widget for the [countup](https://github.com/inorganik/countUp.js/) library that allows animating numbers.

First, we create a package which we name `counter`. You can name the package different but avoid naming it `countup`. We will later have to install the external dependency also named `countup` and NPM does not allow a project named X to use a dependency also named X.

```r
usethis::create_package("counter")
```

From the root of the package, we scaffold the widget with `scaffold_widget`, which prints out some information on what packer exactly does.

```r
packer::scaffold_widget("countup")
```

```
── Scaffolding widget ──────────────────────────────── countup ── 
✔ Bare widget setup
✔ Created srcjs directory
✔ Initialiased npm
✔ webpack, webpack-cli, webpack-merge installed with scope dev
✔ Created srcjs/config directory
✔ Created webpack config files
✔ Created srcjs/modules directory
✔ Created srcjs/widgets directory
✔ Created srcjs/index.js
✔ Moved bare widget to srcjs
✔ Added npm scripts

── Adding files to '.gitignore' and '.Rbuildignore' ──

✔ Setting active project to '/Projects/countup'
✔ Adding '^srcjs$' to '.Rbuildignore'
✔ Adding '^node_modules$' to '.Rbuildignore'
✔ Adding '^package\\.json$' to '.Rbuildignore'
✔ Adding '^package-lock\\.json$' to '.Rbuildignore'
✔ Adding '^webpack\\.dev\\.js$' to '.Rbuildignore'
✔ Adding '^webpack\\.prod\\.js$' to '.Rbuildignore'
✔ Adding '^webpack\\.common\\.js$' to '.Rbuildignore'
✔ Adding 'node_modules' to '.gitignore'

── Adding packages to Imports ──

✔ Adding 'htmlwidgets' to Imports field in DESCRIPTION
● Refer to functions with `htmlwidgets::fun()`

── Scaffold built ──

ℹ Run `bundle` to build the JavaScript files
```

Importantly, it runs `htmlwidgets::scaffoldWidget` internally, there is thus no need to run this function. About the widget itself, there is very little difference between what `htmlwidgets::scaffoldWidget` and `packer::scaffold_widget`. While, if you remember, the initial scaffold of htmlwidgets includes a simple function to display a message in HTML using `innerText`. The scaffold produced by packer differs only in that this message is displayed in `<h1>` HTML tags. That is so it can, from the get-go, demonstrate how to modularise a widget. We'll cover that in just a minute before we do so, bundle the JavaScript and run the `counter` function to observe the output it generates.

```r
packer::bundle()
devtools::load_all()
countup("Hello widgets!")
```

This indeed displays the message in `<h1>` HTML tags, now onto unpacking the structure generated by packer. We'll skip the R code to keep this concise as nothing differs from a standard widget on that side. Instead, we'll focus on the JavaScript code in the `srcjs` directory. First, in the `srcjs/widgets` directory, one will find the file `countup.js`, this file contains the code that produces the widget.

At the top of the file are the imports. First, it imports `widgets`, which is an _external dependency,_ second it imports the function `asHeader` from the `header.js` file in the `modules` directory.

```js
import 'widgets';
import { asHeader } from '../modules/header.js'; 

HTMLWidgets.widget({

  name: 'countup',

  type: 'output',

  factory: function(el, width, height) {

    // TODO: define shared variables for this instance

    return {

      renderValue: function(x) {

        // TODO: code to render the widget, e.g.
        el.innerHTML = asHeader(x);

      },

      resize: function(width, height) {

        // TODO: code to re-render the widget with a new size

      }

    };
  }
});
```

The `header.js` file includes the `asHeader` function which accepts an `x` argument that is used to create the `<h1>` message. This function is exported.

```js
const asHeader = (x) => {
  return '<h1>' + x.message + '</h1>';
}

export { asHeader };
```

We will make changes to the JavaScript so that instead of displaying the message as text, uses the aforementioned countup library to animate a number. The first order of business is to install countup. Here we use packer to do so, the function call below is identical to running `npm install countup --save` from the terminal.

```r
packer::npm_install("countup", scope = "prod") 
```

We will not need the `header.js` file, we can delete it and in its stead create another file called `count.js`. This file will include a function that uses countup to animate the numbers; it should accept 1) the id of the element where countup should be used, and 2) the value that countup should animate. This function called `counter` is, at the end of the file, exported.

```js
import { CountUp } from 'countup.js';

function counter(id, value){
  var countUp = new CountUp(id, value);
  countUp.start();
}

export { counter };
```

We need to add the import statement to bring in the `counter` function and run it in the `renderValue` method. Packer also added the htmlwidgets external dependency which is imported below with `import 'widgets'`.

Because we left the _R function_ `countup` untouched, we have to use the default `message` variable it accepts. Ideally, this argument in the R function should be renamed to something more adequate. 

```js
import 'widgets';
import { counter } from '../modules/count.js'; 

HTMLWidgets.widget({

  name: 'countup',

  type: 'output',

  factory: function(el, width, height) {

    // TODO: define shared variables for this instance

    return {

      renderValue: function(x) {

        counter(el.id, x.message);

      },

      resize: function(width, height) {

        // TODO: code to re-render the widget with a new size

      }

    };
  }
});
```

Finally, the JavaScript bundle can be generated with `packer::bundle()`, install the package or run `devtools::load_all()`, and test that the widget works!

```r
countup(12345)
```

That hopefully is a compelling example to use NPM and webpack to build widgets. It could even be argued that it is easier to set up; dependencies are much more manageable; nothing has to be manually downloaded; it will be easier to update them in the future, etc.

## Shiny with Vue {#packer-adv-shiny-vue}

In this example, we create a shiny application that uses [Vue.js](https://vuejs.org/) in the front-end. If you prefer using [React](https://reactjs.org/) know that it is also supported by webpack and packer. Vue is a framework to create user interfaces which, like React, makes much of the front-end work much more straightforward. It reduces the amount of code one has to write, simplifies business logic, enables to easily include more reactivity, and much more.

Since packer only allows placing scaffolds in R packages the way one can build shiny applications is using the golem package. Golem is an opinionated framework for build applications _as R packages._ Writing shiny applications as R packages brings many of the advantages that packages have to shiny applications: ease of installation, unit testing, dependency management, etc.

```r
install.packages("golem")
```

After installing golem from CRAN, we can create an application with the `golem::create_golem` function.

```r
golem::create_golem("vuer")
```

From within a golem application one use a scaffold specifically designed for this with `scaffold_golem`. Note that this does not mean other scaffolds will not work, custom shiny inputs and outputs can also be created with `scaffold_input`, and `scaffold_output` respectively. The `scaffold_golem` function takes two core arguments; `vue` and `react`. Setting either of these to `TRUE` will prepare a scaffold specifically designed to support either Vue or React.

The reason these arguments exist is because webpack requires further configuration that can be tricky to set up manually. Moreover, Vue supports (but does not require) a `.vue` files which can hold HTML, JavaScript, and CSS.

When the `vue` argument is set to `TRUE` in `scaffold_golem`, the function does follow the usual procedure or initialising NPM, creating the various files, and directories, but in addition configures two loaders and the vue plugin.

Loaders are transformers, they scan for files in the `srcjs` directory and pre-process them. That allows using, for instance, using the Babel compiler that will transform the latest version of JavaScript into code that every browsers can run. This compiler is very often used, including here to compile Vue code. Since Vue allows placing CSS in `.vue` files another loader is required; one that will look for CSS and bundle it within the JavaScript file.

Plugins are a feature of webpack that allow extending its functionalities, there is one for Vue which the function will install and configure for you.

```r
packer::scaffold_golem(vue = TRUE)
```