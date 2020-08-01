---
title: React学习笔记
date: 2020-7-24
tags: web, react
---


## 简介

React是用于构建用户界面的javascript库

- 特性
    * 声明式编程
        + 通过代码告诉计算机，你想的是什么，让计算机想出如何去做
        + 命令式编程：告诉计算机去做什么
    * 组件化
    * 一次学会，随处编写


## Main Concepts

### Rendering Elements

To rendering an element into a root DOM node like `<div id="root"></div>`, pass both to `ReactDOM.render()`:

``` javascript
const element = <h1>Hello, world</h1>;
ReactDOM.render(element, document.getElementById('root'));
```

React elements are immutable. Once you create an element, you can’t change its children or attributes. React Only Updates What's Necessary.

With our knowledge so far, the only way to update the UI is to create a new element, and pass it to `ReactDOM.render()`.


### Components and Props

Components are like JavaScript functions. **They accept arbitrary inputs (called “props”) and return React elements**.

``` javascript
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
```

This function is valid React component because it accept a single "props" object and return a React element.

You can also use an `ES6 class` to define a component:

``` javascript
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```


#### Rendering a Component

React element not only can represent DOM tags, but also can represent user-defined components.

**When React sees an element representing a user-defined component, it pass JSX attributes and children to this component as a single object**. We call this obj "props".

For example:

``` javascript
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}

const element = <Welcome name="Sara" />;
ReactDOM.render(
  element,
  document.getElementById('root')
);
```


#### Props are Read-Only

React is pretty flexible but it has a single strict rule:

**All React components must act like pure functions with respect to their props**


### State and Lifecycle

State is similar to props, but it it private and fully controlled by the component.

- Converting a Function to a Class
    * 1. Create an `ES6 class`, with the same name, that extends `React.Component`
    * 2. Add a single empty method to it called `render()`
    * 3. Move the body of the function into the `render()` method
    * 4. Replace `props` with `this.props` in the `render()` body
    * 5. Delete the remaining empty function declaration

``` javascript
class Clock extends React.Component {
  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.props.date.toLocaleTimeString()}.</h2>
      </div>
    );
  }
}
```


#### Adding Local State to a Class

- 1. Add a `class constructor` that assigns the initial `this.state`
    * Class components should always call the base constructor with `props`
    ``` javascript
    constructor(props) {
        super(props);
        this.state = {date: new Date()}
    }
    ```
- 2. Replace `this.props.date` with `this.state.date`


#### Adding Lifecycle Methods to a Class

We can declare special methods on the component class to run some code when a component mounts and unmounts:

- `componentDidMount()` method runs after the component output has been rendered to the DOM
- `componentWillUnmount()`
- Use `this.setState()` to schedule updates to the component local state:
    * like: `this.setState({date: new Date()})`


#### Using State Correctly

- **Do Not Modify State Directly** 
    * WRONG: `this.state.comment = "hello"`
    * CORRECT: `this.setState({comment: "hello"})`
    * the only place where you can assign `this.state` is the constructor
- **State Updates May Be Asynchronous**
    * React may batch multiple `setState()` calls into a single update for performance
    * Because of that, you should not rely on their values for calculating the next state
    * **To fix it, use a second form of `setState()` that accepts a function rather than an object**
        + the function will receive the previous state as the first argument, and the props as the time update is applied as the second argument
        + `this.setState((state, props)=>{})`


#### State Updates are Merged

[???](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-are-merged)


#### The Date Flows Down

Both parent and children should not care if a certain component is stateful(class) or stateless(functionf). It is not accessible to any component other than the one that owns and sets it.

A component may choose to pass its state down as props to its child components: `<FormattedDate date={this.state.date} />`

This is commonly called a “top-down” or “unidirectional” data flow.


### Handling Events

https://reactjs.org/docs/handling-events.html






