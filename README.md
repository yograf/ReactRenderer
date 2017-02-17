# ReactRenderer

ReactRenderer lets you implement React.js client and server-side rendering in your PHP projects, allowing the development of universal (isomorphic) applications.

It was previously part of [ReactBundle](https://github.com/Limenius/ReactBundle) but now can be used standalone.

Features include:

* Prerrender server-side React components for SEO, faster page loading, for users that have disabled JavaScript, or for Progressive Web Applications.
* Twig integration.
* Client-side render will take the server-side rendered DOM, recognize it, and take control over it without rendering again the component until needed.
* Error and debug management for server and client side code.
* Simple integration with Webpack.

[![Build Status](https://travis-ci.org/Limenius/ReactRenderer.svg?branch=master)](https://travis-ci.org/Limenius/ReactRenderer)
[![Latest Stable Version](https://poser.pugx.org/limenius/react-renderer/v/stable)](https://packagist.org/packages/limenius/react-renderer)
[![Latest Unstable Version](https://poser.pugx.org/limenius/react-renderer/v/unstable)](https://packagist.org/packages/limenius/react-renderer)
[![License](https://poser.pugx.org/limenius/react-renderer/license)](https://packagist.org/packages/limenius/react-renderer)

## Complete example

For a complete live example, with a sensible Webpack set up, a sample application to start with and integration in a Symfony Project, check out [Symfony React Sandbox](https://github.com/Limenius/symfony-react-sandbox).

## Installation

ReactRenderer uses Composer, please checkout the [composer website](http://getcomposer.org) in case of doubt about this.

This command will install `ReactRenderer` into your project.

```bash
$ composer require limenous/react-renderer
```

> ReactRenderer follows the PSR-4 convention names for its classes so you can integrate it with your autoloader.

## Usage

### JavaScript and Webpack set up

In order to use React components you need to register them in your JavaScript. This bundle makes use of the React On Rails npm package to render React Components (don't worry, you don't need to write any Ruby code! ;) ).

Your code exposing a React component would look like this:

```js
import ReactOnRails from 'react-on-rails';
import RecipesApp from './RecipesAppServer';

ReactOnRails.register({ RecipesApp });
```

Where RecipesApp is the component we want to register in this example.

Note that it is very likely that you will need separated entry points for your server-side and client-side components, for dealing with things like routing. This is a common issue with any universal (isomorphic) application. Again, see the sandbox for an example of how to deal with this.

If you use server-side rendering, you are also expected to have a Webpack bundle for it, containing React, React on Rails and your JavaScript code that will be used to evaluate your component.

Take a look at [the Webpack configuration in the symfony-react-sandbox](https://github.com/Limenius/symfony-react-sandbox/blob/master/webpack.config.serverside.js) for more information.

### Enable Twig Extension

First, you need to configure and enable the Twig extension.

```php
use Limenius\ReactRenderer\Renderer\PhpExecJsReactRenderer;
use Limenius\ReactRenderer\Twig\ReactRenderExtension;

$renderer = new PhpExecJsReactRenderer(__DIR__.'/client/build/server-bundle.js');
$ext = new ReactRenderExtension($renderer, 'both');

$twig->addExtension(new Twig_Extension_StringLoader());
$twig->addExtension($ext);
```

`ReactRenderExtension` needs as arguments a *renderer* and a string that defines if we are rendering our React components `client_side`, `render_side` or `both`.

The renderer is one of the renders that inherit from [`AbstractReactRenderer`](ReactRenderer/src/Limenius/ReactRenderer/Renderer/AbstractReactRenderer.php).

This library provides currently two renderers:

* `PhpExecJsReactRenderer`: that uses internally [phpexecjs](https://github.com/nacmartin/phpexecjs) to autodetect the best javascript runtime available.
* `ExternalServerReactRenderer`: that relies on a external nodeJs server.

Now you can insert React components in your Twig templates with:

```twig
{{ react_component('RecipesApp', {'props': props}) }}
```

Where `RecipesApp` is, in this case, the name of our component, and `props` are the props for your component. Props can either be a JSON encoded string or an array. 

For instance, a controller action that will produce a valid props could be:

```php
/**
 * @Route("/recipes", name="recipes")
 */
public function homeAction(Request $request)
{
    $serializer = $this->get('serializer');
    return $this->render('recipe/home.html.twig', [
        'props' => $serializer->serialize(
            ['recipes' => $this->get('recipe.manager')->findAll()->recipes], 'json')
    ]);
}
```

### Server-side, client-side or both?

You can choose whether your React components will be rendered only client-side, only server-side or both, either in the configuration as stated above or per Twig tag basis.

If you set the option `rendering` of the Twig call, you can override your config (default is to render both server-side and client-side).

```twig
{{ react_component('RecipesApp', {'props': props, 'rendering': 'client_side'}) }}
```

Will render the component only client-side, whereas the following code

```twig
{{ react_component('RecipesApp', {'props': props, 'rendering': 'server_side'}) }}
```

... will render the component only server-side (and as a result the dynamic components won't work).

Or both (default):

```twig
{{ react_component('RecipesApp', {'props': props, 'rendering': 'both'}) }}
```

You can explore these options by looking at the generated HTML code.

### Debugging

One imporant point when running server-side JavaScript code from PHP is the management of debug messages thrown by `console.log`. ReactRenderer, inspired React on Rails, has means to replay `console.log` messages into the JavaScript console of your browser.

To enable tracing, you can set a config parameter, as stated above, or you can set it in your template in this way:

```twig
{{ react_component('RecipesApp', {'props': props, 'trace': true}) }}
```

Note that in this case you will probably see a React warning like

*"Warning: render(): Target node has markup rendered by React, but there are unrelated nodes as well. This is most commonly caused by white-space inserted around server-rendered markup."*

This warning is harmlesss and will go away when you disable trace in production. It means that when rendering the component client-side and comparing with the server-side equivalent, React has found extra characters. Those characters are your debug messages, so don't worry about it.

### Server-Side modes

This library supports two modes of using server-side rendering:

* Using [PhpExecJs](https://github.com/nacmartin/phpexecjs) to auto-detect a JavaScript environment (call node.js via terminal command or use V8Js PHP) and run JavaScript code through it. This is more friendly for development, as every time you change your code it will have effect immediatly, but it is also more slow, because for every request the server bundle containing React must be copied either to a file (if your runtime is node.js) or via memcpy (if you have the V8Js PHP extension enabled) and re-interpreted. It is more **suited for development**, or in environments where you can cache everything.

* Using an external node.js server ([Example](https://github.com/Limenius/symfony-react-sandbox/tree/master/app/Resources/node-server/server.js)). It will use a dummy server, that knows nothing about your logic to render React for you. This is faster but introduces more operational complexity (you have to keep the node server running). For this reason it is more **suited for production**.

### Redux

If you're using [Redux](http://redux.js.org/) you could use this library to hydrate your store's:

Use `redux_store` in your Twig file before you render your components depending on your store:

```twig
{{ redux_store('MySharedReduxStore', initialState ) }}
{{ react_component('RecipesApp') }}
```
`MySharedReduxStore` here is the identifier you're using in your javascript to get the store. The `initialState` can either be a JSON encoded string or an array. 

Then, expose your store in your bundle, just like your exposed your components:

```js
import ReactOnRails from 'react-on-rails';
import RecipesApp from './RecipesAppServer';
import configureStore from './store/configureStore';

ReactOnRails.registerStore({ configureStore });
ReactOnRails.register({ RecipesApp });
```

Finally use `ReactOnRails.getStore` where you would have used your the object you passed into `registerStore`.

```js
// Get hydrated store
const store = ReactOnRails.getStore('MySharedReduxStore');

return (
  <Provider store={store}>
    <Scorecard />
  </Provider>
);
```

Make sure you use the same identifier here (`MySharedReduxStore`) as you used in your Twig file to set up the store. 

You have an example in the [Sandbox](https://github.com/Limenius/symfony-react-sandbox).


## License

This library is under the MIT license. See the complete license in the bundle:

    LICENSE.md

## Credits

ReactRenderer is heavily inspired by the great [React On Rails](https://github.com/shakacode/react_on_rails), and uses its npm package to render React components.
