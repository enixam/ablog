---
title: Developing Htmlwidgets in R with TypeScript and Esbuild
author: Qiushi Yan
date: '2022-01-29'
slug: r-htmlwidgets-ts
categories: []
tags: []
subtitle: ''
summary: "Developing htmlwidgets in R is easy and fun. Let's add TypeScript and Esbuild to have more fun!"
authors: []
lastmod: '2022-01-29T12:44:04-06:00'
draft: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
---



## Htmlwidgets in R

<div class = "note">
This post assumes you know the basics of R Package development, JavaScript and node. I recommend the second chapter of [JavaScript for R](https://book.javascript-for-r.com/) as a starter.
</div>

The R `htmlwidgets` package provides a friendly framework for developing R packages that wraps JavaScript libraries. An htmlwidget is nothing more than a normal R plot plus interactivity powered by JavaScript. The package abstracts away many of the details of juggling with both JavaScript and R, most notable of which being dependecy managment.

An example from the [JavaScript for R](https://book.javascript-for-r.com/) book shows the development of the [`gior`](https://github.com/JohnCoene/gior) package, which corresponds to the [`gio.js`](https://giojs.org/) javascirpt library. The [`inst/htmlwidgets`](https://github.com/JohnCoene/gior/tree/master/inst/htmlwidgets) directory contains necessary dependencies required by `gio.js`. This file is the entrypoint of creating the widget. It depends on javasciprt libraries including `gio.js`, `three.js` and `HTMLWidgets`, `Shiny`. We don't need to worry about including `HTMLWidgets` or `Shiny` ourselves, since R will do it for us. For the first two dependencies, we can download it from cdn and include it in the [`inst/htmlwidgets/lib`](https://github.com/JohnCoene/gior/tree/master/inst/htmlwidgets/lib) directory. Lastly, we include a file [`gior.yaml`](https://github.com/JohnCoene/gior/blob/master/inst/htmlwidgets/gior.yaml) to declare locations of the dependencies that looks like:

```yaml
dependencies:
 - name: three
   version: 97
   src: htmlwidgets/lib/three
   script: three.min.js
 - name: gio
   version: 2.0
   src: htmlwidgets/lib/gio-2.0
   script: gio.min.js
```

Now, whenever we create a widget from R, the rendering context will automatically serve all the javascript files. This workflow is convenient for developing pacakges that does not require much work on the JavaScript side, all we need to do is call some initialization functions in `gior.js`. However, if we need to interact more with the javascript library, more than just passing a few lines of options, this setup is not sufficient. Since javascript dependencies is managed from R and never decalred in `gior.js`, we won't be getting all the nice features a modern text editor can provide, such as autocompletion, snippets, linking and intellisense. Moreover, when our package gets larger we might want to split the javascript code into separate modules rather than cluttering the `gior.js` file, and it's not so fun to do bundling ourselves.

For this reason, it make sense to have more control over how javascript dependencies is managed, rather than just downloding and including a dist file. The end result is still the same, we need to include one or several javascript files for the plot. It's just we will not be using files already provided by cdn but to download the javascript package and do the bundling ourselves. The [`packer`](https://github.com/JohnCoene/packer) package provides an solution to this.

## The `packer` package

In the JavaScript world, dependecy management is done through node and a package manager of choice, like npm, yarn or pnpm. These package managers call be used to create a project-specific environment into which various packages will be insallted.  Then, we would use a bundler like webpack to extract all files into a single file, which is provided everytime a widget is created from R. `packer` can be used to scaffold a project structure for this need, and provides an R interface so that we can still do all the work through R commands. The following two commands scaffold an htmlwidgets package powered by packer:

```r
usethis::create_package("<package-name>")
packer::scaffold_widget("<widget-name>")
```

The project directory tree is generated as

```
├── DESCRIPTION
├── NAMESPACE
├── R
│   ├── <widget-name>.R
├── inst
│   └── packer
├── node_modules
│   └── ...
├── package.json
├── srcjs
│   ├── config
│   ├── inputs
│   └── index.js
├── webpack.common.js
├── webpack.dev.js
└── webpack.prod.js
```

A `node_modules` folder is created for storing javascript dependencies. Note that we are managing javascript dependecies ourselves now, and we can install them with `packer::yarn_install` from R or simply `yarn add` from the command line.

The three files started with `webpack` are webpack configurations for bundling. The `webpack.common.js` file is are common options for both development and production. The `webpack.dev.js` is used for development, and the `webpack.prod.js` is used for production. There are 3 most important webpack options for our purposes, which packer sets in the `srcjs/config`
directory.


- `output` will determine the dist file name and location, this should be named `<widget-name>.js` in the `inst/htmlwidgets` directory so that R knows to include it.

- `entryPoints` determins the starting point of the bundling process. This can be set to any top-level file that imports other dependencies that calls `HTMLWidgets.widget()`. packer use the convention of `srcjs/widgets/<widget-name>.js` as the entry point.

- `externals` declares the dependencies that we dont need webpack to resolve. This includes `Shiny` and `HTMLWidgets` which is outside of the `node_modules` folder and added by R. If we don't declare them webpack will be stucked as it can't find them.

Besdies, there is also a `loaders` option that tells webpack how to preprocess each file type. If we are developing a regular javascript website, this will include different preprocessors for javascript, css, scss, etc. Though in the context of htmlwidgets it's all setup by packer.

Now, if we run `packer::bundle_dev()`, it will invoke `npm run development` specified in the `scripts` section in `package.json`, whcih then runs webpack with development configurations. webpack will include all necessary files and bundle them into a `inst/htmlwidgets`. Anytime we make a change to the `srcjs` directory, we need to run `packer::bundle_dev()` to update the dist file.

This time, since our project follows standard javascript project structure with `package.json` and `node_modules`. When we are writing JavaScript code, our text editor will be able to resolve them and provide intellisense. And we can have arbitrary code structure to to better organize our code, as long as it is imported by the entry file.

## Using TypeScript and Esbuild


The boilerplate produced by packer is satisfactory enough if you are happy with simple JavaScript libraries and webpack. However, if you need to include TypeScript, Sass or framworks like React and Svelte, webpack configurations can be notoriously time-consuming. Although packer also provides templates for the JavaScript version of React and Vue, but they still require a handful of customization in my opinion. Further, webpack is sometimes considered outdated with bigger bundle size and slow bundling speed.

So if you are like me who goes out of his way to have an as "optimized" package as possible, it may be better off to have a personal setup similar to packer with optimized replacements. In essence it's just a matter of producing a dist file in the `inst/htmlwidgets/` directory that guarantees the best development experience, and I will share one combo I find most comfortable. TypeScript is used to replace javascript for static typing, and esbuild to replace webpack with hundres times faster performance , simpler configurations, and native support for TypeScript.

During my recent development of the [xkcd](https://github.com/qiushiyan/xkcd) htmlwidgets package, I migrate a packer-generated setup to one with TypeScript and esbuild.

The first thing is to remove webpack related dependencies in `package.json` and run `yarn update`. Then we can install TypeScript, esbuild and whatever JavaScript library you want to work with

```bash
yarn add -D typescript esbuild @types/node
yarn add <target-package>
```

 We can also remove all the webpack configurations in `srcjs/config` and `webpack.*.js` files in the root directory.

At this point, our `package.json` file should look like this

```json
// ./package.json
{
	"devDependencies": {
		"@types/node": "^17.0.12",
		"esbuild": "^0.14.14",
		"typescript": "^4.5.5"
	},
	"dependencies": {
		"chart.xkcd": "^1.1.13" // this is your widget library
	}
}
```

Next, let's create our entry point file, I like to name it `index.ts` under the `srcts` directory:

```javascript
// ./srcts/index.ts
import * as chartXkcd from "chart.xkcd";

HTMLWidgets.widget({
  name: "xkcd",
  type: "output",
  factory: function (el: HTMLElement, width: number, height: number) {
    // TODO: define shared variables for this instance
    return {
      renderValue: function (x: any) {
        // plotting logic
      },
      resize: function (width: number, height: number) {
        // resizing logic
      },
    };
  },
});
```

Now, let's configure esbuild to meet the requirments of htmlwidgets. Since esbuild does not have a configuration file that will be automatically pick up when invoked from the command line, we'll create a normal `esbuild.js` file in the root directory  and then run it through `node`.

```javascript
// ./esbuild.js
const esbuild = require("esbuild");
const path = require("path");

esbuild
  .build({
    entryPoints: [path.join(__dirname, "srcts/index.ts")],
    bundle: true,
    outfile: path.join(__dirname, "inst/htmlwidgets/xkcd.js"),
    platform: "node",
    format: "cjs",
    external: ["Shiny", "HTMLWidgets"],
    watch: {
      onRebuild(error, result) {
        if (error) console.error("watch build failed:", error);
        else console.log("watch build succeeded:", result);
      },
    },
  })
  .catch((err) => {
    process.stderr.write(err.stderr);
    process.exit(1);
  });
```

Note that esbuild share similar configurations with webpack, we are again decalring entry file (`entryPoints`), where the bundled file should go (`outfile`),  and external dependencies (`external`). The last step is adding a command that invokes this script and do the bundling:

```json
./package.json
{
  "scripts": {
		"watch": "node esbuild.js"
	},
	"devDependencies": {
		"@types/node": "^17.0.12",
		"esbuild": "^0.14.14",
		"typescript": "^4.5.5"
	},
	"dependencies": {
		"chart.xkcd": "^1.1.13"
	}
}
```

After run `yarn watch` from the command line, esbuild starts with the message:

```bash
❯ yarn watch
yarn run v1.22.17
$ node esbuild.js
watch build succeeded: { errors: [], warnings: [], stop: [Function: stop] }
```

This will create the `<widget-name>.js` dist file under `inst/htmlwidgets/`. Because we set `watch` in `esbuild.js`, esbuild will watch for changes in the entry file and related modules and rebuild the bundle whenever it changes. This means if we are making only making a change on the JavaScript side, the widget should update automatically next time it's created. So there is no similar need to call `packer::bundle_dev()` again.

With this setup, it's easy to include any additional libraries you like to use. For example, if you want to include [tailwindcss](https://tailwindcss.com/) in your widget, you can do a yarn install and look up the corresponding tailwind-esbuild configuration.

This concludes the introduction of my custom setup, over time I may come to this and add more tips. Thanks for reading!