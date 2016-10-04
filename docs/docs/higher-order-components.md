# Higher-order components

A higher-order component (HOC) is an advanced technique in React for reusing component logic. HOCs are not part of the React API, per se. They are a pattern that emerges from React's compositional nature.

Concretely, **a higher-order component is a function that takes a component and returns a new component.** An HOC accepts a component as one of its arguments, and returns an enhanced version of that component:

```js
let EnhancedComponent = higherOrderComponent(ChildComponent)
```

Whereas a component transforms props into UI, a higher-order component transforms one component into another component.

HOCs are common in third-party React libraries, such as Redux's `connect()` and Relay's `createContainer()`.

In this document, we'll discuss why higher-order components are useful, and how to write your own.

## Use HOCs for cross-cutting concerns

Components are the primary unit of code reuse in React. However, you'll find that some patterns aren't a straightforward fit for traditional components.

For example, say you have a CommentList component that subscribes to an external data source to render a list of comments:

```js
class CommentList extends React.Component {
  constructor(props) {
    super();
    this.handleChange = this.handleChange.bind(this);
    this.state = {
      // Store data in state
      comments: DataSource.getComments()
    };
  }

  componentDidMount() {
    // Subscribe to changes
    DataSource.addChangeListener(this.handleChange);
  }

  componentWillUnmount() {
    // Clean up listener
    DataSource.removeChangeListener(this.handleChange);
  }

  handleChange() {
    // Update component state whenever the data source changes
    this.setState({
      comments: DataSource.getComments()
    });
  }

  render() {
    return (
      <div>
        {comments.map((comment) => (
          <Comment comment={comment} key={comment.id} />
        ))}
      </div>
    );
  }}
```

Later, you write a component for subscribing to a single blog post, which follows a similar pattern:

```js
class BlogPost extends React.Component {
  constructor(props) {
    super();
    this.handleChange = this.handleChange.bind(this);
    this.state = {
      blogPost: DataSource.getBlogPost(props.id)
    };
  }

  componentDidMount() {
    DataSource.addChangeListener(this.handleChange);
  }

  componentWillUnmount() {
    DataSource.removeChangeListener(this.handleChange);
  }

  handleChange() {
    this.setState({
      blogPost: DataSource.getBlogPost(this.props.id)
    });
  }

  render() {
    return <BlogPost blogPost={this.state.blogPost} />;
  }}
```

`CommentList` and `BlogPost` aren't identical—they call different methods on `DataSource`, and they render different output. But much of their implementation is the same:

- On mount, add a change listener to `DataSource`.
- Inside the listener, call `setState` whenever the data source changes.
- On unmount, remove the change listener.

You can imagine that in large app, this same pattern of subscribing to `DataSource` and calling `setState` will occur over and over again. We want an abstraction that allows us to define this logic in a single place and share them across many components. This is where HOCs excel.

We'll create a function called `withSubscription` that works like this:

```js
let CommentListWithSubscription = withSubscription(
  // The first parameter is the wrapped component
  CommentList,
  // The second parameter retrieves the data we're interested in
  function selectData(DataSource) {
    return DataSource.getComments();
  }
);

let BlogPostWithSubscription = withSubscription(
  CommentList,
  // An optional second argument passes the current props
  function selectData(DataSource, props) {
    return DataSource.getBlogPost(props.id);
  }
});
```

And here's what the implementation of `withSubscription` looks like:

```js
// This function takes a component...
function withSubscription(WrappedComponent, selectData) {
  // ...and returns another component...
  return class extends React.Component {
    constructor(props) {
      this.state = {
        data: selectData(DataSource, props)
      };
    }

    componentDidMount() {
      // ... that takes care of the subscription...
      DataSource.addChangeListener(this.handleChange);
    }

    componentWillUnmount() {
      DataSource.removeChangeListener(this.handleChange);
    }

    handleChange: function() {
      this.setState({
        comments: selectData(DataSource, this.props)
      });
    }

    render: function() {
      // ... and renders the wrapped component with the fresh data!
      // Notice that we pass through any additional props
      return <WrappedComponent data={this.state.data} {...this.props} />;
    }
  });
}
```

And that's it! The wrapped component receives a single prop, `data`, which it uses to render its output. The HOC isn't concerned with how or why the data is used, and the wrapped component isn't concerned with where the data came from.

Because `withSubscription` is a normal function, you can add as many or as few arguments as you like. For example, you may want to make the name data prop configurable, to further isolate the HOC from the wrapped component. Or you could accept an argument that configures `shouldComponentUpdate`, or one that changes the data source. These are all possible because the HOC has full control over how the component is defined.

Note that like components, the contract between `withDataSubscription` and the wrapped component is entirely props-based. This makes it easy to swap one HOC for a different one, as long as they provide the same props to the wrapped component. An example of when this may be useful is if you change data-fetching libraries.

## Why not use a component?

Components are a good choice when you want to vary the output based on the given props. However, in our example, the difference between `CommentList` and `BlogPost` isn't limited to their output. They also differ in the implementation of their lifecycle methods.

Because HOCs control the definition of the component class itself, they have the power to configure all aspects of a component's behavior, not just what goes in the `render` method.

Note: It's possible to use normal components in place of certain types of HOCs. See the document on render callbacks to learn more about this technique and how it compares.

## Conventions for idiomatic higher-order components

Although HOCs are a pattern rather than a first-class part of React's API, there are some conventions that we recommend most HOCs follow.

### Don't mutate the original component. Use composition.

Resist the temptation to modify a component's prototype (or otherwise mutate it) inside an HOC.

```js
function onlyUpdateForKeys(InputComponent, keys) {
  // Mutates the input component's prototype
  // *Don't do this!*
  InputComponent.prototype.shouldComponentUpdate = function(nextProps) {
    for (let i; i < keys.length; i++) {
      let key = keys[i];
      if (
        nextProps.hasOwnProperty(key) &&
        this.props.hasOwnProperty(key) &&
        nextProps[key] !== this.props[key]
      ) {
        return false
      }
    }
    return true;
  }
  // The fact that we're returning the original input is a hint that it has
  // been mutated.
  return InputComponent;
}

// EnhancedComponent will only update when the `foo` or `bar` props change
let EnhancedComponent = onlyUpdateForKeys(InputComponent, ['foo', 'bar'])
```

There are a few problems with this. One is that the input component cannot be reused separately from the enhanced component. More crucially, if you apply another HOC to `EnhancedComponent` that *also* mutates `shouldComponentUpdate`, the first HOC's functionality will be overridden! Mutating HOCs are a leaky abstraction—the consumer must know how they are implemented in order to avoid conflicts with other HOCs.

Instead of mutation, HOCs should use composition, by wrapping the input component in a container component:

```js
function onlyUpdateForKeys(WrappedComponent, keys) {
  return class extends React.Component {
    shouldComponentUpdate(nextProps) {
      for (let i; i < keys.length; i++) {
        let key = keys[i];
        if (
          nextProps.hasOwnProperty(key) &&
          this.props.hasOwnProperty(key) &&
          nextProps[key] !== this.props[key]
        ) {
          return false;
        }
      }
      return true;
    }
    render() {
      // Wraps the input component in a container, without mutating it. Good!
      return <WrappedComponent {...this.props} />;
    }
  }
}
```

This HOC has the same functionality as the mutating version while avoiding the potential for clashes. Because it's a pure function, it's composable with other HOCs, or even with itself:

```js
// EnhancedComponent will only update when the `foo`, `bar`, or `baz` props change
let EnhancedComponent = onlyUpdateForKeys(InputComponent, ['foo', 'bar']);
EnhancedComponent = onlyUpdateForKeys(EnhancedComponent, ['baz']);
```

You may have noticed similarities between HOCs and a pattern called **container components**. Container components are a pattern for separating responsibility between high-level and low-level concerns. Containers manage things like subscriptions and state, and pass props to components that handle things like rendering UI. HOCs use containers as part of their implementation. You can think of HOCs as a paramaterized container component definition.

### Pass unrelated props through to the wrapped component

HOCs add features to a component. They shouldn't drastically alter its contract. It's expected that the component returned from an HOC has a similar interface to the wrapped component.

HOCs should pass through props that are unrelated to its specific concern. Most HOCs contain a render method that looks something like this:

```js
render() {
  // Filter out extra props that are specific to this HOC and shouldn't be
  // passed through
  const { extraProp, ...passThroughProps } = this.props;

  // Inject props into the wrapped component. These are usually state values or
  // instance methods.
  let injectedProp = someStateOrInstanceMethod;

  // Pass props to wrapped component
  return (
    <WrappedComponent
      injectedProp={injectedProp}
      {...passThroughProps}
    />
  );
}
```

This convention helps ensure that HOCs are as flexible and reusable as possible.

### Optional: Maximizing composability

Not all HOCs look the same. Sometimes they accept only a single argument, the wrapped component:

```js
let NavbarWithRouter = withRouter(Navbar);
```

Usually, HOCs accept additional arguments. In this example from Relay, a config object is used to specific a component's data dependencies:

```js
let CommentWithRelay = Relay.createContainer(CommentWithRelay, config);
```

The most common signature for HOCs looks like this:

```js
// React Redux's `connect`
let ConnectedComment = connect(commentSelector, commentActions)(Comment);
```

*What?!* If you break it apart, it's easier to see what's going on.

```js
// connect is a function that returns another function
let enhance = connect(commentListSelector, commentListActions);
// The returned function is an HOC, which returns a component that is connected
// to the Redux store
let ConnectedComment = enhance(CommentList);
```
In other words, `connect` is a higher-order function that returns a higher-order component!

This may seem confusing or unnecessary, but this form has a useful property. Single-argument HOCs like the one returned by the `connect` function have the form `Component => Component`. Functions that whose output type is the same as its input type are really easy to compose together.

```js
// compose performs function composition
// compose(f, g, h) is the same as (...args) => f(g(h(...args)))
let enhance = compose(
  // These are both single-argument HOCs
  connect(commentSelector),
  withRouter
)
let EnhancedComponent = enhance(WrappedComponent)

// This is the same as
let EnhancedComponent = connect(commentSelector)(withRouter(WrappedComponent))
```

(This same property also allows `connect` and other enhancer-style HOCs to be used as decorators, an experimental JavaScript proposal.)

### Optional: Wrap the display name for easy debugging

TODO

## Caveats

TODO

- Can't use HOCs inside render.
- Composing multiple HOCs together can be confusing, especially if you're not used to using function composition.
- Instance methods don't work; static methods have to be explicitly copied over.
- Refs are complicated.

## More examples

TODO

https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html
