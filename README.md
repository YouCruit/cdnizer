# cdnizer
[![NPM version][npm-image]][npm-url] [![Build Status][travis-image]][travis-url]

This library will replace references in HTML and other files with CDN locations.  It's flexible without being overly complicated, and handles private *and* public CDNs.  It can use your bower installation to determine file versions.

It also provides optional fallback scripts for failed file loading.  By default it can only handle failed JavaScript files, but it shouldn't be too difficult to provide a better script.

### New in version 1.0

cdnizer now can load CDN data from existing `*-cdn-data` packages, currently `google-cdn-data`, `cdnjs-cdn-data`, and `jsdelivr-cdn-data`.  Now you can [configure common public CDNs with a single line](#optionsfilescommon-cdn)!

### Possible breaking change in 1.0

I've changed the `version` field in this release.  Previously, it was the exact version as it existing within bower.  Now, `version` is the string `major(.minor)?(.patch)?`, with any trailing (`-beta*`, `-snapshot*`, etc) information removed.  (Characters not separated by a hyphen are not affected.)

You can still access the full version string via `versionFull`, which is not modified at all.

## Usage

First, install `cdnizer`:

```shell
npm install --save cdnizer
```

Then, use it like so:

```javascript
var cdnizerFactory = require("cdnizer"),
	cdnizer = cdnizerFactory({
                  defaultBase: "//my.cdn.host/base",
                  allowRev: true,
                  allowMin: true,
                  files: [
                      'js/app.js',
                      {
                          file: 'vendor/angular/angular.js',
                          package: 'angular',
                          test: 'angular',
                          cdn: '//ajax.googleapis.com/ajax/libs/angularjs/${ version }/angular.min.js'
                      },
                      {
                          file: 'vendor/firebase/firebase.js',
                          test: 'Firebase',
                          cdn: '//cdn.firebase.com/v0/firebase.js'
                      }
                  ]
              });
	var contents = fs.readFileSync('./src/index.html', 'utf8');
	contents = cdnizer(contents);
```
     
Alternatively, you can just pass in the files array if you don't need to provide any options, and only have custom files:

```js
var cdnizerFactory = require("cdnizer"),
	cdnizer = cdnizerFactory([
		            {
		                file: 'vendor/angular/angular.js',
		                package: 'angular',
		                test: 'angular',
		                // use alternate providers easily
		                cdn: '//cdnjs.cloudflare.com/ajax/libs/angularjs/${ version }/angular.min.js'
		            },
		            {
		                file: 'vendor/firebase/firebase.js',
		                test: 'Firebase',
		                cdn: '//cdn.firebase.com/v0/firebase.js'
		            }
		        ]);
```

You can also use globs to define groups of file, and dynamic filename properties:

```js
var cdnizerFactory = require("cdnizer"),
	cdnizer = cdnizerFactory([{
	                file: 'vendor/angular/*.js',
	                package: 'angular',
	                test: 'angular',
	                cdn: '//ajax.googleapis.com/ajax/libs/angularjs/${ version }/${ filenameMin }'
	            }]);
```

**New in v1.0**, you can use simplified strings for common public CDNs, like so:

```js
var cdnizerFactory = require("cdnizer"),
	cdnizer = cdnizerFactory([
					'google:angular', // that's it!
					'cdnjs:jquery',
					{
						cdn: 'jsdelivr:yui', // use a known CDN, while…
						package: 'yui3', // overriding the package name for bower, and…
						test: 'YUI' // providing a custom fallback test
					}
				]);
```

Works great on `url()`s in CSS files, too:

```js
var cdnizerFactory = require("cdnizer"),
	cdnizer = cdnizerFactory({
		            defaultCDNBase: '//my.cdn.url/',
		            relativeRoot: 'css',
		            files: ['**/*.{gif,png,jpg,jpeg}']
		        });
	var cssFile = fs.readFileSync('./css/style.css', 'utf8');
	cssFile = cdnizer(cssFile);
```

## API

### cdnizer(options|files)

Note: If options is an array, it is assumed to be the array of files to process, and the rest of the options are left as their default.

#### options.defaultCDNBase

Type: `String`  
Default: `''` (disabled)

Used for a default, custom CDN, usually for your own files.  This will be used in the defaultCDN property to define the default path for a CDN unless overridden.

#### options.defaultCDN

Type: `String`  
Default: `'${ defaultCDNBase }/${ filepathRel }'`

This is the default pattern used for generating CDN links when one isn't provided by a specific file.

#### options.relativeRoot

Type: `String`  
Default: `''`

If you are processing a file that references relative files, or is not rooted to the CDN, you can set `relativeRoot` to get correct results.

For example, if you have a CSS file under `style/`, and you reference images as `../img/foo.png`, you should set `relativeRoot` to `style/`.  Now if your `defaultCDNBase` is `//example/`, the image will be resolved to `//example/img/foo.png`.  

#### options.allowRev
Type: `Boolean`  
Default: `true`

Allow for file names with `gulp-rev` appended strings, in the form of `<file>-XXXXXXXX.<ext>`.  If you are using the `gulp-rev` plugin, this will automatically match filenames that have a rev string appeneded to them.  If you are *not* using `gulp-rev`, then you can disable this by setting `allowRev` to `false`, which will prevent possible errors in accidentally matching similar file names.

You can always manually configure your globs to include an optional rev string by using the form `?(<rev glob here>)`, such as `name?(-????????).ext` for appended revs.

#### options.allowMin

Type: `Boolean`  
Default: `true`

Allow for file names that optionally have `.min` inserted before the file extension (but after rev, if enabled).  This allows you to use the base name for a file, and have `cndizer` match the minified name.

#### options.fallbackScript

Type: `String`  
Default:

```html
<script>
function cdnizerLoad(u) {
	document.write('<scr'+'ipt src="'+encodeURIComponent(u)+'"></scr'+'ipt>';
}
</script>
```

Overwrite the default fallback script.  If any of the inputs has a fallback, this script is injected before the first occurrence of `<link`, `<script`, or `</head>` in the HTML page.  Ignored for files that don't contain `<head`.

If you already have a script loader (such as yepnope or Modernizr), you can set this to an empty string and override the `fallbackTest` below to use that instead.  Of course, this won't help if you are loading *those* scripts off a CDN and they fail!

#### options.fallbackTest

Type: `String`  
Default: `'<script>if(!(${ test })) cdnizerLoad("${ filepath }");</script>'`

Overwrite the default fallback test.  Note that all options availble to `options.files[].cdn` below are available to this template as well, along with the `options.files[].test` string.

#### options.bowerComponents

Type: `String`  
Default: null

If provided, this is the directory to look for bower components in.  If not provided, cdnizer will attempt to look for the `.bowerrc` file, and if that is not found or does not specify a directory, it falls back to `'./bower_components'`.

Once the directory is determined, the script will look for files in `<bowerComponents>/bower.json` or `<bowerComponents>/.bower.json` to try to determine the version of the installed component.

#### options.files

Type: `Array`  
Default: (none) **required**

Array of sources or objects defining sources to cdnize.  Each item in the array can be one of three types, a simple glob, a public CDN string, or object hashmap.

##### options.files.«glob»

When using a glob, if it matches a source, the `defaultCDN` template is applied.  Because there is no `test`, the script will not have a fallback.

*Examples:*
```js
'**/*.js' // matches any .js file anywhere
'js/**/*.js' // matches any *.js file under the js folder
'styles/main.css' // only matches styles/main.css, possibly rev'd or min'd based on options
'img/icon/foo??.{png,gif,jpg}' // img/icon/foo10.png, img/icon/fooAA.gif, etc
```

##### options.files.«common-cdn»

Public CDN strings make it easy to add known libraries from common public CDN repositories.  They always take the form of `'<provider>:<package>(@version)?'`.  Currently, cdnizer has built-in support for [Google](https://www.npmjs.org/package/google-cdn-data), [cdnjs](https://www.npmjs.org/package/cdnjs-cdn-data), and [jsDelivr](https://www.npmjs.org/package/jsdelivr-cdn-data), via the existing packages maintained by [Shahar Talmi](https://www.npmjs.org/~shahata).

*Examples:*

```js
'google:jquery'
'google:jquery@1.0.0' // provide a version to override bower OR if you are not using bower
'google:angular'
'cdnjs:angular.js' // you need `.js` for angular on cdnjs, but not on Google!
'jsdelivr:angularjs' // jsdelivr has it different still
```

You can also use a common cdn while still providing your own overrides by [using a common CDN with the `cdn` option within a hashmap](#optionsfilescdn).  You will need to do this if the CDN provider uses a different package name than bower, or if you want to provide a fallback test (excluding a few popular libraries).

*Important Notes:*

1. **Case matters** with these strings. Make sure you are not capitalizing the packages (such as `jQuery`, instead of `jquery`), or the output will be incorrect.

2. The packages may have different names on different public CDNs.  Make sure you look up the package name on the CDN website first.  For example, AngularJS is `angular` on Google, `angular.js` on cdnjs, and `angularjs` on jsDelivr!

3. You **must** provide the version if you are not using bower to manage your packages, or if you want to override the bower version (not recommended).

4. Due to the way versions are calculated, `-beta*`, `-unstable*`, and `-snapshot*` releases will not work with common CDNs.  This means only projects with standard 1-, 2-, or 3-part version strings will work.

5. The CDN packages available are limited to those included in the `cdn-data` packages, which means they might not always be up-to-date with the latest public packages.  However, the version information is not used at all by cdnizer, which means you can update to the latest version faster.

##### options.files.«hashmap»

The object hashmap gives you full control, using the following properties:

> ##### options.files[].file

> Type: `String`  
> Default: (none) **required**

> Glob to match against for the file to cdnize.  All properties within this object will be applied to all files that match the glob.  Globs are matched in a first-come, first-served basis, where only the first matched object hashmap is applied.

> ##### options.files[].package

> Type: `String`  
> Default: (none)

> Bower package name for this source or set of sources.  By providing the package name, cdnizer will look up the version string of the *currently installed* bower package, and provide it as a property to the `cdn` string.  This is done by looking for either the `bower.json` or `.bower.json` file within your bower components directory.

> The benefit of doing it this way is that the version used from the CDN *always* matches your local copy.  It will never automatically be updated to a newer patch version without being tested.

> ##### options.files[].cdn

> Type: `String`  
> Default: `options.defaultCDN`

> This it the template for the replacement string. It can either be a custom CDN string, or it can be a common public CDN string, using the same format as a [public CDN string](#optionsfilescommon-cdn) above.

> *Common Public CDN String:*

> Load in the default data for an existing common public CDN, using the format `'<provider>:<package>(@version)?'`. You can then customize the settings for the package, by overriding any property in this section (e.g.: providing a fallback `test`, a different `package` name, or even matching a different `file`).

> *Custom CDN String:*

> Provide a custom CDN string, which can be a simple static string, or contain one or more underscore/lodash template properties to be injected into the string:

> * `versionFull`: if [`package`](#optionsfilespackage) was provided, this is the complete version currently installed version from bower.
> * `version`: if `package` was provided, this is the `major(.minor)?(.patch)?` version number, minus any trailing information (such as `-beta*` or `-snapshot*`).
> * `major`: if `package` was provided, this is the major version number.
> * `minor`: if `package` was provided, this is the minor version number.
> * `patch`: if `package` was provided, this is the patch version number.
> * `defaultBase`: the default base provided above.  Note that this will *never* end with a trailing slash.
> * `filepath`: the full path of the source, as it exists currently.  There is no guarantee about the whether this contains a leading slash or not, so be careful.
> * `filepathRel`: the relative path of the source, guaranteed to *never* have a leading slash. The path is also processed against `options.relativeRoot` above, to try and remove any parent directory path elements.
> * `filename`: the name of the file, without any parent directories
> * `filenameMin`: the name of the file, *without* any rev tags (if `allowRev` is true), but *with* a `.min` extension added.  This won't add a min if there is one already.
> * `package`: the bower package name, as provided above.

> ##### options.files[].test

> Type: `String`  
> Default: (none)

> If provided, this string will be evaluated within a javascript block.  If the result is truthy, then we assume the CDN resource loaded properly.  If it isn't, then the original local file will be loaded.  This is ignored for files that don't get the [fallback script](#optionsfallbackscript).

> When using a common public CDN, some popular packages come with fallback tests.  The current packages that have a built-in fallback test are:

> * AngularJS
> * Backbone.js
> * Dojo
> * EmberJS
> * jQuery
> * jQuery UI
> * Lo-Dash
> * MooTools
> * Prototype
> * React
> * SwfObject
> * Underscore

> For any other packages, you'll need to provide the fallback test yourself.

> See [`options.fallbackScript`](#optionsfallbackscript) and [`options.fallbackTest`](#optionsfallbacktest) for more information.

#### options.matchers

Type: `Array`  
Default: `[]`

Array of custom matchers. Use this to add extra patterns within which you would like to cdn-ize URLs, for example if you have such URLs in data-attributes. The matchers should include regular expressions with three matching groups: 1) Leading characters 2) The actual URL to work on 3) Trailing characters.

Example (matches the ```data-src``` attribute in ```<img>``` tags):<br />
```matchers: [ { pattern: /(<img\s.*?data-src=["'])(.+?)(["'].*?>)/gi, fallback: false } ]```

You can also specify just a regular expression. In that case, fallback will default to false.

Equivalent example:<br />
```matchers: [ pattern: /(<img\s.*?data-src=["'])(.+?)(["'].*?>)/gi ]```

#### options.cdnDataLibraries

Type: `Array`
Default: `[]`

Future-proof option to add additional `*-cdn-data` packages.  These packages *must* be in the same format as [`google-cdn-data`](https://www.npmjs.org/package/google-cdn-data).  The format is to only include the leading part of the package name, for example, `cdnjs-cdn-data` would be included as simply `'cdnjs'`.

## Help Support This Project

If you'd like to support this and other OverZealous Creations (Phil DeJarnett) projects, [donate via Gittip][gittip-url]!

[![Support via Gittip][gittip-image]][gittip-url]

You can learn a little more about me and some of the [work I do for open source projects in an article at CDNify.](https://cdnify.com/blog/overzealous-creations/)

## License

[MIT License](http://en.wikipedia.org/wiki/MIT_License)

[npm-url]: https://npmjs.org/package/cdnizer
[npm-image]: https://badge.fury.io/js/cdnizer.png

[travis-url]: http://travis-ci.org/OverZealous/cdnizer
[travis-image]: https://secure.travis-ci.org/OverZealous/cdnizer.png?branch=master

[gittip-url]: https://www.gittip.com/OverZealous/
[gittip-image]: https://raw2.github.com/OverZealous/gittip-badge/0.1.2/dist/gittip.png
