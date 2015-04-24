# Examples

```js
// Simple config to run jshint any time a file is added, changed or deleted
grunt.initConfig({
  watchChokidar: {
    files: ['**/*'],
    tasks: ['jshint'],
  },
});
```

```js
// Advanced config. Run specific tasks when specific files are added, changed or deleted.
grunt.initConfig({
  watchChokidar: {
    gruntfile: {
      files: 'Gruntfile.js',
      tasks: ['jshint:gruntfile'],
    },
    src: {
      files: ['lib/*.js', 'css/**/*.scss', '!lib/dontwatch.js'],
      tasks: ['default'],
    },
    test: {
      files: '<%= jshint.test.src %>',
      tasks: ['jshint:test', 'qunit'],
    },
  },
});
```

## Using the `watchChokidar` event
This task will emit a `watchChokidar` event when watched files are modified. This is useful if you would like a simple notification when files are edited or if you're using this task in tandem with another task. Here is a simple example using the `watchChokidar` event:

```js
grunt.initConfig({
  watchChokidar: {
    scripts: {
      files: ['lib/*.js'],
    },
  },
});
grunt.event.on('watchChokidar', function(action, filepath, target) {
  grunt.log.writeln(target + ': ' + filepath + ' has ' + action);
});
```

**The `watchChokidar` event is not intended for replacing the standard Grunt API for configuring and running tasks. If you're trying to run tasks from within the `watchChokidar` event you're more than likely doing it wrong. Please read [configuring tasks](http://gruntjs.com/configuring-tasks).**

### Compiling Files As Needed
A very common request is to only compile files as needed. Here is an example that will only lint changed files with the `jshint` task:

```js
grunt.initConfig({
  watchChokidar: {
    scripts: {
      files: ['lib/*.js'],
      tasks: ['jshint'],
      options: {
        spawn: false,
      },
    },
  },
  jshint: {
    all: {
      src: ['lib/*.js'],
    },
  },
});

// on watch events configure jshint:all to only run on changed file
grunt.event.on('watchChokidar', function(action, filepath) {
  grunt.config('jshint.all.src', filepath);
});
```

If you need to dynamically modify your config, the `spawn` option must be disabled to keep the watch running under the same context.

If you save multiple files simultaneously you may opt for a more robust method:

```js
var changedFiles = Object.create(null);
var onChange = grunt.util._.debounce(function() {
  grunt.config('jshint.all.src', Object.keys(changedFiles));
  changedFiles = Object.create(null);
}, 200);
grunt.event.on('watchChokidar', function(action, filepath) {
  changedFiles[filepath] = action;
  onChange();
});
```

## Live Reloading
Live reloading is built into the watch task. Set the option `livereload` to `true` to enable on the default port `35729` or set to a custom port: `livereload: 1337`.

The simplest way to add live reloading to all your watch targets is by setting `livereload` to `true` at the task level. This will run a single live reload server and trigger the live reload for all your watch targets:

```js
grunt.initConfig({
  watchChokidar: {
    options: {
      livereload: true,
    },
    css: {
      files: ['public/scss/*.scss'],
      tasks: ['compass'],
    },
  },
});
```

You can also configure live reload for individual watch targets or run multiple live reload servers. Just be sure if you're starting multiple servers they operate on different ports:

```js
grunt.initConfig({
  watchChokidar: {
    css: {
      files: ['public/scss/*.scss'],
      tasks: ['compass'],
      options: {
        // Start a live reload server on the default port 35729
        livereload: true,
      },
    },
    another: {
      files: ['lib/*.js'],
      tasks: ['anothertask'],
      options: {
        // Start another live reload server on port 1337
        livereload: 1337,
      },
    },
    dont: {
      files: ['other/stuff/*'],
      tasks: ['dostuff'],
    },
  },
});
```

### Enabling Live Reload in Your HTML
Once you've started a live reload server you'll be able to access the live reload script. To enable live reload on your page, add a script tag before your closing `</body>` tag pointing to the `livereload.js` script:

```html
<script src="//localhost:35729/livereload.js"></script>
```

Feel free to add this script to your template situation and toggle with some sort of `dev` flag.

### Using Live Reload with the Browser Extension
Instead of adding a script tag to your page, you can live reload your page by installing a browser extension. Please visit [how do I install and use the browser extensions](http://feedback.livereload.com/knowledgebase/articles/86242-how-do-i-install-and-use-the-browser-extensions-) for help installing an extension for your browser.

Once installed please use the default live reload port `35729` and the browser extension will automatically reload your page without needing the `<script>` tag.

### Using Connect Middleware
Since live reloading is used when developing, you may want to disable building for production (and are not using the browser extension). One method is to use Connect middleware to inject the script tag into your page. Try the [connect-livereload](https://github.com/intesso/connect-livereload) middleware for injecting the live reload script into your page.

### Rolling Your Own Live Reload
Live reloading is made easy by the library [tiny-lr](https://github.com/mklabs/tiny-lr). It is encouraged to read the documentation for `tiny-lr`. If you would like to trigger the live reload server yourself, simply POST files to the URL: `http://localhost:35729/changed`. Or if you rather roll your own live reload implementation use the following example:

```js
// Create a live reload server instance
var lrserver = require('tiny-lr')();

// Listen on port 35729
lrserver.listen(35729, function(err) { console.log('LR Server Started'); });

// Then later trigger files or POST to localhost:35729/changed
lrserver.changed({body:{files:['public/css/changed.css']}});
```

### Live Reload with Preprocessors
Any time a watched file is edited with the `livereload` option enabled, the file will be sent to the live reload server. Some edited files you may desire to have sent to the live reload server, such as when preprocessing (`sass`, `less`, `coffeescript`, etc). As any file not recognized will reload the entire page as opposed to just the `css` or `javascript`.

The solution is to point a `livereload` watch target to your destination files:

```js
grunt.initConfig({
  sass: {
    dev: {
      src: ['src/sass/*.sass'],
      dest: 'dest/css/index.css',
    },
  },
  watchChokidar: {
    sass: {
      // We watch and compile sass files as normal but don't live reload here
      files: ['src/sass/*.sass'],
      tasks: ['sass'],
    },
    livereload: {
      // Here we watch the files the sass task will compile to
      // These files are sent to the live reload server after sass compiles to them
      options: { livereload: true },
      files: ['dest/**/*'],
    },
  },
});
```