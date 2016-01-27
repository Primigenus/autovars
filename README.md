# AutoVars
Reactive state management for Meteor + React

## Why should I use this?
Meteor and React are both great frameworks but integrated you may run into
the following issues when you build a complex, data heavy, highly interactive
app.

### 1. Performance issues due to rerendering of entire subtrees
React is a great addition to Meteor for several reasons, one of them is it's
components which allow you to encapsulate both view and logic. This is always
a bit of a pain when you use Blaze.

Unfortunately if you have an app with many nested components, each with their
own logic and derived state, rerendering those components may be too costly.

React was built with a nice shadow DOM to be able to quickly execute view
changes. But that doesn't help when child components have their own set of
calculations to run before they can render themselves. Those still have to run
before every render.

One approach is to separate logic and view and keep the React components as dumb
as possible. This is the approach Flux takes.

AutoVars goes in the other direction by making the components smarter so
renders become more fine grained and less frequent. It relies heavily on core
Meteor reactivity.

### 2. Bugs due to multiple state mutation flows: data, props and state
React has props to pass data between components and state to manage internal
component state. The integration with Meteor adds `.data` and `getMeteorData`.
Both `render` and `getMeteorData` become reactive and are executed when a
reactive dependency they use changes.

This means both React and Meteor's Tracker can execute your code. They do so
when they see fit. And they don't sync their timing.

This does not have to be a problem when you only use `getMeteorData` to pull in
data from a Meteor Mongo collection and from there on program React style. This
can be achieved with a strict code convention in your team.

Unfortunately when the team grows and you have more Meteor experts on the team,
`ReactiveVar`'s and `autoruns` may enter your project, possibly from other
packages. The only way to connect those to the React world is through
`getMeteorData` and then you will come to a point where it is very hard to
understand what triggers executing your code.

AutoVars solves this problem by replacing `.data` and `.state` as well as most
of the React component lifecycle with automatically executing functions, the
results of which can be accessed through `.autovars`.

## Show me an example!
An AutoVars version of the simple-todos example:

```javascript
// App component - represents the whole app
App = React.createClass({
  mixins: [AutoVarMixin],

  // Here goes all logic previously in getMeteorData and getInitialState
  constructAutoVars() {
    // These cursors never change, so create them once (constructAutoVars is
    // called once in the React component lifecycle).
    const sortBy = { sort: { createdAt: -1 } };
    const allCursor = Tasks.find({}, sortBy);
    const incompleteCursor = Tasks.find({ checked: { $ne: true } }, sortBy);

    return {
      // Creates this.autovars.hideCompleted that can be set in code below
      hideCompleted: false,

      // Executed when hideCompleted changes or when the currently used cursor
      // updates.
      tasks: () => this.autovals.hideCompleted ?
        incompleteCursor.fetch() :
        allCursor.fetch(),

      // Executed when the underlying collection changes
      incompleteCount: () => incompleteCursor.count()
    }
  },

  ...

  toggleHideCompleted() {
    // Toggle the boolean, will cause all depending autovars to be reexecuted
    this.autovals.hideCompleted = !this.autovals.hideCompleted;
  },

  ...

  render() {
    // Render will depend on incompleteCount, hideCompleted and tasks (through
    // renderTasks). If any or all of those change, render will be executed
    // exactly once.
    return (
      <div className="container">
        <header>
          <h1>Todo List ({this.autovals.incompleteCount})</h1>

          <label className="hide-completed">
            <input
              type="checkbox"
              readOnly={true}
              checked={this.autovals.hideCompleted}
              onClick={this.toggleHideCompleted} />
            Hide Completed Tasks
          </label>
  ...
```  

See the examples directory for more examples.

## API
When you add the `AutoVarMixin` to your React component, the following will be
added to the component.

### `this.constructAutoVars`
This function is called once (during `componentWillMount`) and is the place
where you can declare your Autovars. Autovars come in two flavors: primitive
values that can be updated from other code and functions that are automatically
rerun when the autovars (or other reactive dependencies) they use have changed.

```javascript
App = React.createClass({
  mixins: [AutoVarMixin],

  constructAutoVars() {
    return {
      count: 0,
      oddOrEven: () => this.autovals.count % 2 === 0 ? 'even' : 'odd'
    }
  }
```

The functions declared in `constructAutoVars` are executed in order and can
therefore use the output of preceding sibling vars.

### `this.autovars`
All Autovars declared in `constructAutoVars` are exposed through
`this.autovars`. An Autovar is in fact a ReactiveVar and `this.autovars` gives
you a handle to that ReactiveVar. This is particularly useful if you want to
pass the var through React props, because with Autovars, only components
actually reading from the ReactiveVar will be rerendered. With plain React, all
components in the hierarchy passing on props are rerendered.

Example:

```javascript
  click() {
    this.autovars.count.set(this.autovars.count.get() + 1);
  },

  render() {
    const count = this.autovars.count.get();
    const oddOrEven = this.autovars.oddOrEven.get();
    return (
      <div>
        <button onClick={this.click()}>Click Me</button>
        <p>{count} clicks ({oddOrEven}).</p>
      </div>)
  }
```

### `this.autovals`
`this.autovals` is a shorthand for `this.autovars` exposing the same vars, but
now with getters and setters so your code becomes more concise.

Example:

```javascript
  render() {
    return (
      <div>
        <button onClick={_ => this.autovals.count++}>Click Me</button>
        <p>{this.autovals.count} clicks ({this.autovals.oddOrEven}).</p>
      </div>)
  }
```
