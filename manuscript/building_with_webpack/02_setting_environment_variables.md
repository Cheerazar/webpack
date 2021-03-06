# Setting Environment Variables

React relies on `process.env.NODE_ENV` based optimizations. If we force it to `production`, React will get built in an optimized manner. This will disable some checks (e.g., property type checks). Most importantly it will give you a smaller build and improved performance.

## The Basic Idea of `DefinePlugin`

Webpack provides `DefinePlugin` that is able to rewrite matching **free variables**. To understand the idea better, consider the example below:

```javascript
var foo;

// Not free due to "foo" above, not ok to replace
if (foo === 'bar') {
  console.log('bar');
}

// Free since we don't refer to "bar", ok to replace
if (bar === 'bar') {
  console.log('bar');
}
```

If we replaced `bar` with a string like `'foobar'`, then we would end up with code like this:

```javascript
var foo;

// Not free due to "foo" above, not ok to replace
if (foo === 'bar') {
  console.log('bar');
}

// Free since we don't refer to "bar", ok to replace
if ('foobar' === 'bar') {
  console.log('bar');
}
```

Further analysis shows that `'bar' === 'bar'` equals `true` so UglifyJS gives us:

```javascript
var foo;

// Not free due to "foo" above, not ok to replace
if (foo === 'bar') {
  console.log('bar');
}

// Free since we don't refer to "bar", ok to replace
if (false) {
  console.log('bar');
}
```

And based on this UglifyJS can eliminate the `if` statement:

```javascript
var foo;

// Not free, not ok to replace
if (foo === 'bar') {
  console.log('bar');
}

// if (false) means the block can be dropped entirely!
```

This is the core idea of `DefinePlugin`. We can toggle parts of code using it using this kind of mechanism. UglifyJS is able to perform the analysis for us and enable/disable entire portions of it as we prefer.

## Setting `process.env.NODE_ENV`

To show you the idea in practice, we could have a declaration like `if (process.env.NODE_ENV === 'development')` within our code. Using `DefinePlugin` we could replace `process.env.NODE_ENV` with `'production'` to make our statement evaluate as false just like above and eliminate the related code as a result.

As before, we can encapsulate this idea to a function:

**webpack.parts.js**

```javascript
...

leanpub-start-insert
exports.setFreeVariable = function(key, value) {
  const env = {};
  env[key] = JSON.stringify(value);

  return {
    plugins: [
      new webpack.DefinePlugin(env)
    ]
  };
}
leanpub-end-insert
```

We can connect this with our configuration like this:

**webpack.config.js**

```javascript
...

module.exports = function(env) {
  if (env === 'build') {
    return merge(
      common,
      {
        devtool: 'source-map'
      },
leanpub-start-insert
      parts.setFreeVariable(
        'process.env.NODE_ENV',
        'production'
      ),
leanpub-end-insert
      parts.minify(),
      parts.setupCSS(PATHS.app)
    );
  }

  ...
};
```

Execute `npm run build` again, and you should see improved results:

```bash
Hash: 796b5780b9701387d6cb
Version: webpack 2.2.0-rc.1
Time: 1211ms
     Asset       Size  Chunks           Chunk Names
    app.js    24.1 kB  0[emitted]  app
index.html  180 bytes  [emitted]
  [14] ./app/component.js 136 bytes {0} [built]
  [16] ./app/main.css 904 bytes {0} [built]
  [17] ./~/css-loader!./app/main.css 190 bytes {0} [built]
  [32] ./app/index.js 124 bytes {0} [built]
    + 29 hidden modules
Child html-webpack-plugin for "index.html":
        + 4 hidden modules
```

So we went from 148 kB to 45 kB, and finally, to 24.1 kB. The final build is a little faster than the previous one. As that 24.1 kB can be served gzipped, it is quite reasonable. gzipping will drop around another 40% and it is well supported by browsers.

T> [babel-plugin-transform-inline-environment-variables](https://www.npmjs.com/package/babel-plugin-transform-inline-environment-variables) Babel plugin can be used to achieve the same effect. See [the official documentation](https://babeljs.io/docs/plugins/transform-inline-environment-variables/) for details.

T> Note that we are missing [react-dom](https://www.npmjs.com/package/react-dom) from our build. In practice our React application would be significantly larger unless we are using a lighter version such as [preact](https://www.npmjs.com/package/preact) or [react-lite](https://www.npmjs.com/package/react-lite). These libraries might be missing some features, but they are worth knowing about if you use React.

T> `webpack.EnvironmentPlugin(['NODE_ENV'])` is a shortcut that allows you to refer to environment variables. It uses `DefinePlugin` internally and can be useful by itself in more limited cases. You can achieve the same effect by passing `process.env.NODE_ENV` to the custom function we made.

## Generating gzips with Webpack

Even though you can let your server to gzip the files using a suitable middleware, you can also setup webpack to generate the gzips for you using [compression-webpack-plugin](https://www.npmjs.com/package/compression-webpack-plugin). This can save some processing time on the server.

## Conclusion

Even though simply setting `process.env.NODE_ENV` the right way can help a lot especially with React related code, we can do better. We can split `app` and `vendor` bundles and add hashes to their filenames to benefit from browser caching. After all, the data that you don't need to fetch loads the fastest.
