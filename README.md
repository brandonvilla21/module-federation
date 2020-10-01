# Micro-frontends: Module Federation with WebPack 5

### What is Module Federation?

It is basically a JavaScript architecture. It allows a JavaScript application to dynamically load code from another application (a different Webpack build).

### This is how you normally use Webpack

You would use Webpack to generate a bundle for production or development, let's say Webpack helps you to generate a folder called `dist` and a file `main.js` within this folder. This is the result of all of your JavaScript code that you normally have in a folder called `src`

The more you add code into your `src` folder the heavier is this `main.js` file which Webpack generates. Remember that this is the file you take to your production environment and clients download in their browsers, if this file is heavy that means it will take longer for the users to load your page.

That means we care about the size of our bundle but we also want to keep adding new features to our projects

### Is there a solution to this problem?

There is, there are strategies to break that `main.js` file into chunks of smaller files in order to avoid loading all your code at first render. This is called Code Splitting ([https://webpack.js.org/guides/code-splitting/](https://webpack.js.org/guides/code-splitting/))

There are different techniques to accomplish this, one is defining more than one entry point into your Webpack configuration but it comes with some pitfalls, sometimes you will have duplicated modules between chunks and both chunks will include these modules so it will increase the size of your chunks.

There's another popular and more accepted way, this consists in using the `import()` syntax which conforms to the ES Proposal in order to have dynamic imports in JS ([https://github.com/tc39/proposal-dynamic-import](https://github.com/tc39/proposal-dynamic-import))

Using this approach looks something like this:

```jsx
function test() {
  import('./some-file-inside-my-project.js')
    .then(module => module.loadItemsInPage())
    .catch(error => alert('There was an error'))
}
```

We can lazy load the elements to our page using `import()` syntax and also this will create a new chunk which will get loaded on demand

**But what if I told you that there's another way to break this main.js file not only into different chunks but into different projects?**

### Here is where Module Federation comes

With Module Federation you can import remote Webpack builds to your application. Currently, you could import these chunks but they would have to come from your same project. Now, you can have these chunks (Webpack builds) from a different origin, which means, a different project!

### Module Federation in action

To explain what all of this is about, we will see some code samples of a Webpack configuration using `ModuleFederationPlugin`  and some React.js code

For this, we will use Webpack 5 which currently is on version beta. This is how the `package.json` file looks like:

```jsx
// package.json (fragment)

...

  "scripts": {
   "start": "webpack-dev-server --open",
   "build": "webpack --mode production"
  },
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@babel/core": "7.10.3",
    "@babel/preset-react": "7.10.1",
    "babel-loader": "8.1.0",
    "html-webpack-plugin": "^4.3.0",
    "webpack": "5.0.0-beta.24",
    "webpack-cli": "3.3.11",
    "webpack-dev-server": "^3.11.0"
  },
  "dependencies": {
    "react": "^16.13.1",
    "react-dom": "^16.13.1"
  }

...
```

We have included all the Webpack modules to create a basic setup for a React application

This is how the `webpack.config.js` looks so far:

```jsx
// webpack.config.js

const HtmlWebpackPlugin = require('html-webpack-plugin');
const path = require('path');

module.exports = {
  entry: './src/index',
  mode: 'development',
  devServer: {
    contentBase: path.join(__dirname, 'dist'),
    port: 3000,
  },
	output: {
    publicPath: "http://localhost:3000/",
  },
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        loader: 'babel-loader',
        exclude: /node_modules/,
        options: {
          presets: ['@babel/preset-react'],
        },
      },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './public/index.html',
    }),
  ],
};
```

This is a normal configuration of Webpack

Let's add a react component to the project:

```jsx
// src/index.js

import React from 'react';
import ReactDOM from 'react-dom';

function App() {
  return (
    <h1>Hello from React component</h1>
  )
}

ReactDOM.render(<App />, document.getElementById('root'));
```

At this point if you run this project, you will get a page which will show a message saying "Hello from React component". Until now, there's nothing new here.

The code of this project until this step is here:[https://github.com/brandonvilla21/module-federation/tree/initial-project](https://github.com/brandonvilla21/module-federation/tree/initial-project)

## Creating a second project

Now, we will create a second project with the same `package.json` file but with some differences under the Webpack configuration:

```jsx
const HtmlWebpackPlugin = require('html-webpack-plugin');
const path = require('path');

// Import Plugin
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  entry: './src/index',
  mode: 'development',
  devServer: {
    contentBase: path.join(__dirname, 'dist'),
    // Change port to 3001
    port: 3001,
  },
	output: {
    publicPath: "http://localhost:3000/",
  },
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        loader: 'babel-loader',
        exclude: /node_modules/,
        options: {
          presets: ['@babel/preset-react'],
        },
      },
    ],
  },
  plugins: [
    // Use Plugin
    new ModuleFederationPlugin({
      name: 'app2',
      library: { type: 'var', name: 'app2' },
      filename: 'remoteEntry.js',
      exposes: {
        // expose each component you want 
        './Counter': './src/components/Counter',
      },
      shared: ['react', 'react-dom'],
    }),
    new HtmlWebpackPlugin({
      template: './public/index.html',
    }),
  ],
};
```

We're importing the ModuleFederationPlugin on top of the configuration

```jsx
const { ModuleFederationPlugin } = require('webpack').container;
```

We also need to change the port since we will be running both applications at the same time

```jsx
port: 3001,
```

And this is how the Plugin config looks like:

```jsx
new ModuleFederationPlugin({
  name: 'app2', // We need to give it a name as an identifier
  library: { type: 'var', name: 'app2' },
  filename: 'remoteEntry.js', // Name of the remote file
  exposes: {
    './Counter': './src/components/Counter', // expose each component you want 
  },
  shared: ['react', 'react-dom'], // If the consumer application already has these libraries loaded, it won't load them twice
}),
```

This is the main piece of configuration in order to share the dependencies of this second project with the first one.

Before consuming this second application from the first one, let's create the Counter component:

```jsx
// src/components/Counter.js

import React from 'react'

function Counter(props) {
  return (
     <>
       <p>Count: {props.count}</p>
       <button onClick={props.onIncrement}>Increment</button>
       <button onClick={props.onDecrement}>Decrement</button>
     </>
  )
}

export default Counter
```

This is a very common example but the point here is to show how can we use this component and pass some props from the first application

If you try to run the second app at this point adding a basic `index.js` like what we did on the first application, you will likely get a message saying the following:

```bash
Uncaught Error: Shared module is not available for eager consumption
```

As the error says, you're eagerly executing your application. In order to provide an async way to load the application we can do the following:

Create a `bootstrap.js` file and move all your code from `index.js` to this file

```jsx
// src/bootstrap.js

import React from 'react';
import ReactDOM from 'react-dom';

function App() {
  return <h1>Hello from second app</h1>;
}

ReactDOM.render(<App />, document.getElementById('root'));
```

And import it in `index.js` like this: (*notice we're using `import()` syntax here*)

```jsx
// src/index.js

import('./bootstrap')
```

Now if you run the second project at this point, you will be able to see the message **Hello from second app** 

### Importing Counter component to the first project

We will need to update the `webpack.config.js` file first, in order to consume the Counter component from the second app

```jsx
// webpack.config.js (fragment)

...
plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      library: { type: 'var', name: 'app1' },
      remotes: {
        app2: 'app2', // Add remote (Second project)
      },
      shared: ['react', 'react-dom'],
    }),
    new HtmlWebpackPlugin({
      template: './public/index.html',
    }),
  ],
...
```

The difference between this Webpack config and the other relies on `expose` and `remote`. Where in the first app, we expose the component that we want to take from the first app, so in this app, we specify the name of the remote app

We also need to specify the `remoteEntry.js` file from the remote host:

```html
<!-- public/index.html (fragment)-->

...
<body>
  <div id="root"></div>
  <script src="http://localhost:3001/remoteEntry.js"></script>
</body>
...
```

### Importing React component from a remote project

Now it's time to use the Counter component from the second project into the first project:

```jsx
// src/bootstrap.js

import React, { useState } from 'react';
import ReactDOM from 'react-dom';

const Counter = React.lazy(() => import('app2/Counter'));

function App() {
  const [count, setCount] = useState(0);
  return (
    <>
      <h1>Hello from React component</h1>
      <React.Suspense fallback='Loading Counter...'>
        <Counter
          count={count}
          onIncrement={() => setCount(count + 1)}
          onDecrement={() => setCount(count - 1)}
        />
      </React.Suspense>
    </>
  );
}

ReactDOM.render(<App />, document.getElementById('root'));
```

We will need to lazy load the Counter component and then we can use React Suspense for loading the component with a fallback

That's it! You should be able to load the counter component from the first project

## Conclusions

The possibility to load remote Webpack builds into your applications opens up a new world of possibilities for creating new Frontend architectures. It will be possible to create:

### Micro Frontends

Since we can have separate bundles of JavaScript into separate projects, it gives us the possibility to have separate build processes for each application.

You will be able to have totally independent applications with the feeling of a single website. This allows big teams to break down into smaller and more efficient teams which will scale vertically from the Frontend to the BE and DevOps team.

This way we will have autonomous teams which won't depend on others in order to deliver new features

It could be represented like this:

![Micro-frontends%20Module%20Federation%20with%20WebPack%205%20dcf6c7f5c47a4cb9904b822b96a32d0e/Untitled.png](img/e2e-microfrontend-teams.png)

[Source Image](https://micro-frontends.org/)

### Design system incorporation at runtime

Currently, there are multiple ways to implement a design system at build time (npm/yarn packages, GitHub packages, Bit.dev) but this could represent an issue for some projects. Whenever you need to update some components from your design system, you will have to re-build your application and deploy it again in order to have the latest version of your design system in production.

With a design system at runtime, you will be able to get the latest version of your design system into your application without going through the build and re-deploy process of your entire application since you will get the components from a different origin and at runtime.

These two are just a few of the possibilities with Federated Modules.

## Repository of the complete example

[github.com/brandonvilla21/module-federation](http://github.com/brandonvilla21/module-federation)
