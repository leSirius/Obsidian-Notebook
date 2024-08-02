# Fiber
## Background
Fiber’s architecture has two major phases: 
- reconciliation/render 
    - updates state and props,
    - calls lifecycle hooks,
    - retrieves the children from the component,
	- compares them to the previous children,
	- and figures out the DOM updates that need to be performed.
- commit.
All activities in the 1st phase are referred to as **work** inside Fiber. All kinds of work are listed [here](https://github.com/facebook/react/blob/340bfd9393e8173adca5380e6587e1ea1a23cefa/packages/shared/ReactWorkTags.js?source=post_page---------------------------#L29-L28) . 
### The Problem
> When dealing with UIs, the problem is that if **too much work is executed all at once**, it can cause animations to drop frames…

 ‘all at once’ means the case where React walks the entire tree of components synchronously and perform work for each component, and it may run over 16ms available for an application code to execute its logic.
### `requestIdleCallback` Function
New [`requestIdleCallback`](https://developer.chrome.com/blog/using-requestidlecallback?source=post_page---------------------------) function
``` js
requestIdleCallback((deadline)=>{ 
	console.log(deadline.timeRemaining(), deadline.didTimeout)      // 49.9 false
});
```
>`requestIdleCallback` is actually a bit too restrictive and [is not executed often enough](https://github.com/facebook/react/issues/13206?source=post_page---------------------------#issuecomment-418923831) to implement smooth UI rendering, so React team [had to implement their own version](https://github.com/facebook/react/blob/eeb817785c771362416fd87ea7d2a1a32dde9842/packages/scheduler/src/Scheduler.js?source=post_page---------------------------#L212-L222).

If we put activities into function `performWork`, and use this function to schedule:
```js
requestIdleCallback((deadline) => { 
	// while we have time, perform work for a part of the components tree 
	while ((deadline.timeRemaining()>0 || deadline.didTimeout) && nextComponent){ 
		nextComponent = performWork(nextComponent); 
	}
});
```
This would work, if not for one thing. You can’t process the entire tree of components synchronously (walk the component tree all at once). And that’s the problem Andrew talks about here:

> in order to use those APIs, you need a way to break rendering work into incremental units.

So React had to re-implement the algorithm for walking the tree from the synchronous recursive model that relied on the built-in stack to an asynchronous model with linked list and pointers.
## React Examples and Solutions
## A Recursive Example
![[Pasted image 20240731171456.png]]
``` js
const a1 = {name: 'a1'}; 
const b1 = {name: 'b1'}; 
const b2 = {name: 'b2'}; 
const b3 = {name: 'b3'}; 
const c1 = {name: 'c1'}; 
const c2 = {name: 'c2'}; 
const d1 = {name: 'd1'}; 
const d2 = {name: 'd2'}; 

a1.render = () => [b1, b2, b3]; 
b1.render = () => []; 
b2.render = () => [c1]; 
b3.render = () => [c2]; 
c1.render = () => [d1, d2]; 
c2.render = () => []; 
d1.render = () => []; 
d2.render = () => [];
```
```js
walk(a1); 
function walk(instance) { 
	doWork(instance); 
	const children = instance.render(); 
	children.forEach(walk); 
} 
function doWork(o) { 
	console.log(o.name); 
}
```
The biggest issue is that we can’t break the work into incremental units. We can’t pause the work at a particular component and resume it later. With this approach React just keeps iterating until it processed all components and the stack is empty.

## Linked list traversal

![[Pasted image 20240731172219.png]]
In the context of the new reconciliation algorithm in React, the data structure with following fields, child, sibling and return, is called **Fiber**. Under the hood it’s the representation of a React Element that keeps a queue of work to do.
```js
class Node { 
	constructor(instance) { 
		this.instance = instance; 
		this.child = null; 
		this.sibling = null; 
		this.return = null; 
	} 
}
```
The function that takes an array of nodes and links them together. We’re going to use it to link children returned by the `render` method:

```js
function link(parent, elements) {
    if (elements === null) elements = [];
 
    parent.child = elements.reduceRight((previous, current) => {
        const node = new Node(current);
        node.return = parent;
        node.sibling = previous;
        return node;
    }, null);
 
    return parent.child;
}
// how the funcition works
const children = [{name: 'b1'}, {name: 'b2'}]; 
const parent = new Node({name: 'a1'}); 
const child = link(parent, children);
```
We’ll also implement a helper function that performs some work for a node. In our case, it’s going to log the name of a component. But besides that it also retrieves the children of a component and links them together:

```js
function doWork(node) {
    console.log(node.instance.name);
    const children = node.instance.render();
    return link(node, children);
}
```
Now we can get a new version of parent-first, depth-first traversal: (or a better [view](https://stackblitz.com/edit/js-tle1wr?file=index.js&source=post_page---------------------------))
```js
function walk(o) {
    let root = o;
    let current = o;
 
    while (true) {
        // perform work for a node, retrieve & link the children
        let child = doWork(current);
 
        // if there's a child, set it as the current active node
        if (child) {
            current = child;
            continue;
        }
 
        // if we've returned to the top, exit the function
        if (current === root) {
            return;
        }
 
        // keep going up until we find the sibling
        while (!current.sibling) {
 
            // if we've returned to the top, exit the function
            if (!current.return || current.return === root) {
                return;
            }
 
            // set the parent as the current active node
            current = current.return;
        }
 
        // if found, set the sibling as the current active node
        current = current.sibling;
    }
}

const hostNode = new Node(a1);
walk(hostNode);
```

![[content-image4.gif]]
The above call stack doesn't grow as we walk down the tree. But if now put the debugger into the `doWork` function and log node names, we’re going to see the following:
![[content-image5.gif]]
It looks like a call stack in a browser. So with this algorithm, we’re effectively replacing the browser’s implementation of the call stack with our own implementation. That’s what Andrew describes here:

> Fiber is re-implementation of the stack, specialized for React components. You can think of a single fiber as a virtual stack frame.

Since we’re now controlling the stack by keeping the reference to the node that acts as a top frame:
```
function walk(o) {
    let root = o;
    let current = o;
 
    while (true) {
            ...
 
            current = child;
            ...
            
            current = current.return;
            ...
 
            current = current.sibling;
    }
}
```
  
we can stop the traversal at any time and resume to it later. That’s exactly the condition we wanted to achieve to be able to use the new `requestIdleCallback` API.

### Work loop in React
```js
function workLoop(isYieldy) {
    if (!isYieldy) {
        // Flush work without yielding
        while (nextUnitOfWork !== null) {
            nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
        }
    } else {
        // Flush asynchronous work until the deadline runs out of time.
        while (nextUnitOfWork !== null && !shouldYield()) {
            nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
        }
    }
}
```
The [code](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js?source=post_page---------------------------#L1118) implementing work loop in React, maps to the algorithm presented above. It keeps the reference to the current fiber node in the `nextUnitOfWork` variable that acts as a top frame.

The algorithm can walk the components tree synchronously and perform the work for each fiber node in the tree (`nextUnitOfWork`). This is usually the case for so-called **interactive updates** caused by UI events (click, input etc.). 

Or it can walk the components tree asynchronously checking if there’s time left after performing work for a Fiber node. The function `shouldYield` returns the result based on [deadlineDidExpire](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js?source=post_page---------------------------#L1806) and [deadline](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js?source=post_page---------------------------#L1809) variables that are constantly updated as React performs work for a fiber node.

# reconciliation algorithm
## Background
![](https://angularindepth.com/api/posts/1008/assets/content-image1.gif)

And here’s the implementation:

```js
class ClickCounter extends React.Component {
    constructor(props) {
        super(props);
        this.state = {count: 0};
        this.handleClick = this.handleClick.bind(this);
    }
 
    handleClick() {
        this.setState((state) => {
            return {count: state.count + 1};
        });
    }
 
 
    render() {
        return [
            <button key="1" onClick={this.handleClick}>Update counter</button>,
            <span key="2">{this.state.count}</span>
        ]
    }
}
```
Here are the high-level operations React performs during the first render and after state update in our simple application:
- updates `count` property in the `state` of `ClickCounter`
- retrieves and compares children of `ClickCounter` and their props
- updates props for the `span` element
All these activities are collectively referred to as “work” in the Fiber architecture.

## From React Elements to Fiber nodes

### React Elements
Every component in React has a UI representation we can call a view or a template that’s returned from the `render` method. Here’s the template for our `ClickCounter` component:

```jsx
<button key="1" onClick={this.onClick}>Update counter</button>
<span key="2">{this.state.count}</span>
```
Once a template goes through the JSX compiler, you end up with a bunch of React elements, what we really get from `render` method. We can re-write `render` like this:
``` js
class ClickCounter {
    ...
    render() {
        return [
            React.createElement(
                'button',
                {
                    key: '1',
                    onClick: this.onClick
                },
                'Update counter'
            ),
            React.createElement(
                'span',
                {
                    key: '2'
                },
                this.state.count
            )
        ]
    }
}
```
The calls to `React.createElement` in the `render` method will create two data structures like this:
```js
[ 
    {
        $$typeof: Symbol(react.element),
        type: 'button',
        key: "1",
        props: {
            children: 'Update counter',
            onClick: () => { ... }
        }
    },
    {
        $$typeof: Symbol(react.element),
        type: 'span',
        key: "2",
        props: {
            children: 0
        }
    }
]
```
[$$typeof](https://overreacted.io/why-do-react-elements-have-typeof-property/) is used to uniquely identify these objects as React elements. 
### Fiber nodes
During **reconciliation**, data from every React element returned from the `render` method is merged into the tree of fiber nodes. Every React element is converted into a fiber node of corresponding type,  describing the work that needs to be done. Unlike React elements, fibers aren’t re-created on every render. These are **mutable** data structures that hold components state and DOM.

Fiber’s architecture also provides a convenient way to track, schedule, pause and abort the work.

When a React element is converted into a fiber node for the first time, React uses the data from the element to create a fiber in the [`createFiberFromTypeAndProps`](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414) function. In the consequent updates React reuses the fiber node and just updates the necessary properties using data from a corresponding React element. React may also need to move the node in the hierarchy based on the `key` prop or delete it if the corresponding React element is no longer returned from the `render` method.

Fiber tree:
![[Pasted image 20240731192427.png]]

### Current and work in progress trees
After the first render, React ends up with a fiber tree that reflects the state of the application that was used to render the UI. This tree is often referred to as `current` tree. When React starts working on updates it builds a so-called `workInProgress` tree that reflects the future state to be flushed to the screen. 

All work is performed on fibers from the `workInProgress` tree. As React goes through the `current` tree, for each existing fiber node it creates an alternate node that constitutes the `workInProgress` tree. This node is created using the data from React elements returned by the `render` method. Once the updates are processed and all related work is completed, React will have an alternate tree ready to be flushed to the screen. Once this `workInProgress` tree is rendered on the screen, it becomes the `current` tree.

One of React’s core principles is consistency. React always updates the DOM in one go — it doesn’t show partial results. The `workInProgress` tree serves as a “draft” that’s not visible to the user, so that React can process all components first, and then flush their changes to the screen.

In the sources you’ll see a lot of functions that take fiber nodes from both the `current` and `workInProgress` trees. Here’s the signature of one such function:

```js
function updateHostComponent(current, workInProgress, renderExpirationTime) {...}
```

Each fiber node holds a reference to its counterpart from the other tree in the **alternate** field. A node from the `current` tree points to the node from the `workInProgress` tree and vice versa.
### Side-effects
We can think of a component in React as a function that uses the state and props to compute the UI representation. Every other activity like mutating the DOM or calling lifecycle methods should be considered a side-effect or, simply, an effect. Effects are also mentioned [in the docs](https://reactjs.org/docs/hooks-overview.html#%EF%B8%8F-effect-hook):

> You’ve likely performed data fetching, subscriptions, or manually ****changing the DOM**** from React components before. We call these operations “side effects” (or “effects” for short) because they can affect other components and can’t be done during rendering.**

You can see how most state and props updates will lead to side-effects. And since applying effects is a type of work, a fiber node is a convenient mechanism to track effects in addition to updates. Each fiber node can have effects associated with it. They are encoded in the `effectTag` field.

So effects in Fiber basically define the [work](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js) that needs to be done for instances after updates have been processed. For host components (DOM elements) the work consists of adding, updating or removing elements. For class components React may need to update refs and call the `componentDidMount` and `componentDidUpdate` lifecycle methods. There are also other effects corresponding to other types of fibers.
#### Effects list
An optimization for performance is building a linear list of fiber nodes with effects for quick iteration. Iterating the linear list is much faster than a tree, and there’s no need to spend time on nodes without side-effects.

The goal of this list is to mark nodes that have DOM updates or other effects associated with them. This list is a subset of the `finishedWork` tree (the result of render phase) and is linked using the `nextEffect` property instead of the `child` property used in the `current` and `workInProgress` trees.

[Dan Abramov](https://medium.com/u/a3a8af6addc1?source=post_page---------------------------) offered an analogy for an effects list. He likes to think of it as a Christmas tree, with “Christmas lights” binding all effectful nodes together. For example, our update caused `c2` to be inserted into the DOM, `d2` and `c1` to change attributes, and `b2` to fire a lifecycle method. The effect list will link them together so React can skip other nodes later:

![[Pasted image 20240731200818.png]]

When going over the nodes, React uses the `firstEffect` pointer to figure out where the list starts. So the diagram above can be represented as a linear list like this:

![](https://angularindepth.com/api/posts/1008/assets/content-image4.png)

### Fiber node structure

Let’s now take a look at the structure of fiber nodes created for the `ClickCounter` component:

```js
{
    stateNode: new ClickCounter,
    type: ClickCounter,
    alternate: null,
    key: null,
    updateQueue: null,
    memoizedState: {count: 0},
    pendingProps: {},
    memoizedProps: {},
    tag: 1,
    effectTag: 0,
    nextEffect: null
}
```

and the `span` DOM element:

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    alternate: null,
    key: "2",
    updateQueue: null,
    memoizedState: null,
    pendingProps: {children: 0},
    memoizedProps: {children: 0},
    tag: 5,
    effectTag: 0,
    nextEffect: null
}
```
## General algorithm

React performs work in two main phases: **render** and **commit**

During the first `render` phase React applies updates to components scheduled through `setState` or `React.render` and figures out what needs to be updated in the UI. If it’s the initial render, React creates a new fiber node for each element returned from the `render` method. In the following updates, fibers for existing React elements are re-used and updated. **The result of the phase is a tree of fiber nodes marked with side-effects.** The effects describe the work that needs to be done during the following `commit` phase. During this phase React takes a fiber tree marked with effects and applies them to instances. It goes over the list of effects and performs DOM updates and other changes visible to a user.

**It’s important to understand that the work during the first `render` phase can be performed asynchronously.** React can process one or more fiber nodes depending on the available time, then stop to stash the work done and yield to some event. It then continues from where it left off. Sometimes though, it may need to discard the work done and start from the top again. These pauses are made possible by the fact that the work performed during this phase doesn’t lead to any user-visible changes, like DOM updates. **In contrast, the following `commit` phase is always synchronous.** This is because the work performed during this stage leads to changes visible to the user, e.g. DOM updates. That’s why React needs to do them in a single pass.

Calling lifecycle methods is one type of work performed by React. Some methods are called during the `render` phase and others during the `commit` phase. Here’s the list of lifecycles called when working through the first `render` phase:
- `getDerivedStateFromProps`
- `shouldComponentUpdate`
- `render`

Here’s the list of lifecycle methods executed during the second `commit` phase:

- `getSnapshotBeforeUpdate`
- `componentDidMount`
- `componentDidUpdate`
- `componentWillUnmount`

Because these methods execute in the synchronous `commit` phase, they may contain side effects and touch the DOM.
### Render phase
  
The reconciliation algorithm always starts from the topmost `HostRoot` fiber node using the [renderRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1132) function. However, React bails out of (skips) already processed fiber nodes until it finds the node with unfinished work. For example, if you call `setState` deep in the components tree, React will start from the top but quickly skip over the parents until it gets to the component that had its `setState` method called.
#### Main steps of the work loop
  
All fiber nodes are processed [in the work loop](https://github.com/facebook/react/blob/f765f022534958bcf49120bf23bc1aa665e8f651/packages/react-reconciler/src/ReactFiberScheduler.js#L1136). Here is the implementation of the synchronous part of the loop:
```js
function workLoop(isYieldy) {
  if (!isYieldy) {
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {...}
}
```

In the code above, the `nextUnitOfWork` holds a reference to the fiber node from the `workInProgress` tree that has some work to do. As React traverses the tree of Fibers, it uses this variable to know if there’s any other fiber node with unfinished work. After the current fiber is processed, the variable will either contain the reference to the next fiber node in a tree or `null`. In that case React exits the work loop and is ready to commit the changes.

There are 4 main functions that are used to traverse the tree and initiate or complete the work:
- [`performUnitOfWork`](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1056)
- [`beginWork`](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L1489)
- [`completeUnitOfWork`](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L879)
- [`completeWork`](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberCompleteWork.js#L532)

To demonstrate how they are used, take a look at the following animation of traversing a fiber tree. I’ve used the simplified implementation of these functions for the demo. Each function takes a fiber node to process and as React goes down the tree you can see the currently active fiber node changes. You can clearly see on the video how the algorithm goes from one branch to the other. It first completes the work for children before moving to parents.
[video](https://vimeo.com/302222454)
![[content-image5 1.gif]]

Let’s start with the first two functions `performUnitOfWork` and `beginWork`:

```js
function performUnitOfWork(workInProgress) {
    let next = beginWork(workInProgress);
    if (next === null) {
        next = completeUnitOfWork(workInProgress);
    }
    return next;
}
 
function beginWork(workInProgress) {
    console.log('work performed for ' + workInProgress.name);
    return workInProgress.child;
}
```
The function `performUnitOfWork` receives a fiber node from the `workInProgress` tree and starts the work by calling `beginWork` function. This is the function that will start all the activities that need to be performed for a fiber. For the purposes of this demonstration, we simply log the name of the fiber to denote that the work has been done. ****The function**** `beginWork` ****always returns a pointer to the next child to process in the loop or**** `null`****.****

If there’s a next child, it will be assigned to the variable `nextUnitOfWork` in the `workLoop` function. However, if there’s no child, React knows that it reached the end of the branch and so it can complete the current node. ****Once the node is completed, it’ll need to perform work for siblings and backtrack to the parent after that.**** This is done in the `completeUnitOfWork` function:
```js
function completeUnitOfWork(workInProgress) {
    while (true) {
        let returnFiber = workInProgress.return;
        let siblingFiber = workInProgress.sibling;
 
        nextUnitOfWork = completeWork(workInProgress);
 
        if (siblingFiber !== null) {
            // If there is a sibling, return it
            // to perform work for this sibling
            return siblingFiber;
        } else if (returnFiber !== null) {
            // If there's no more work in this returnFiber,
            // continue the loop to complete the parent.
            workInProgress = returnFiber;
            continue;
        } else {
            // We've reached the root.
            return null;
        }
    }
}
 
function completeWork(workInProgress) {
    console.log('work completed for ' + workInProgress.name);
    return null;
}
```

You can see that the gist of the function is a big `while` loop. React gets into this function when a `workInProgress` node has no children. After completing the work for the current fiber, it checks if there’s a sibling. If found, React exits the function and returns the pointer to the sibling. It will be assigned to the `nextUnitOfWork` variable and React will perform the work for the branch starting with this sibling. It’s important to understand that at this point React has only completed work for the preceding siblings. It hasn’t completed work for the parent node. **Only once all branches starting with child nodes are completed does it complete the work for the parent node and backtracks**

As you can see from the implementation, both and `completeUnitOfWork` are used mostly for iteration purposes, whereas the main activities take place in the `beginWork` and `completeWork` functions. In the following articles in the series we’ll learn what happens for the `ClickCounter` component and the `span` node as React steps into `beginWork` and `completeWork` functions.

### Commit phase
The phase begins with the function [completeRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L2306). This is where React updates the DOM and calls pre and post mutation lifecycle methods.

When React gets to this phase, it has 2 trees and the effects list. The first tree represents the state currently rendered on the screen. Then there’s an alternate tree built during the `render` phase. It’s called `finishedWork` or `workInProgress` in the sources and represents the state that needs to be reflected on the screen. This alternate tree is linked similarly to the current tree through the `child` and `sibling` pointers.

And then, there’s an effects list — a subset of nodes from the `finishedWork` tree linked through the `nextEffect` pointer. Remember that the effect list is the **result** of running the `render` phase. **The whole point of rendering was to determine which nodes need to be inserted, updated, or deleted, and which components need to have their lifecycle methods called.** And that’s what the effect list tells us. **And it’s exactly the set of nodes that’s iterated during the commit phase.**

> For debugging purposes, the `current` tree can be accessed through the `current` property of the fiber root. The `finishedWork` tree can be accessed through the `alternate` property of the `HostFiber` node in the current tree.

The main function that runs during the commit phase is [commitRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L523). Basically, it does the following:

- Calls the `getSnapshotBeforeUpdate` lifecycle method on nodes tagged with the `Snapshot` effect
- Calls the `componentWillUnmount` lifecycle method on nodes tagged with the `Deletion` effect
- Performs all the DOM insertions, updates and deletions
- Sets the `finishedWork` tree as current
- Calls `componentDidMount` lifecycle method on nodes tagged with the `Placement` effect
- Calls `componentDidUpdate` lifecycle method on nodes tagged with the `Update` effect

After calling the pre-mutation method `getSnapshotBeforeUpdate`, React commits all the side-effects within a tree. It does it in two passes. The first pass performs all DOM (host) insertions, updates, deletions and ref unmounts. Then React assigns the `finishedWork` tree to the `FiberRoot` marking the `workInProgress` tree as the `current` tree. This is done after the first pass of the commit phase, so that the previous tree is still current during `componentWillUnmount`, but before the second pass, so that the finished work is current during `componentDidMount****/****Update`. In the second pass React calls all other lifecycle methods and ref callbacks. These methods are executed as a separate pass so that all placements, updates, and deletions in the entire tree have already been invoked.

Here’s the gist of the function that runs the steps described above:

```
function commitRoot(root, finishedWork) {
    commitBeforeMutationLifecycles()
    commitAllHostEffects();
    root.current = finishedWork;
    commitAllLifeCycles();
}
```

Each of those sub-functions implements a loop that iterates over the list of effects and checks the type of effects. When it finds the effect pertaining to the function’s purpose, it applies it.
#### Pre-mutation lifecycle methods

Here is, for example, the code that iterates over an effects tree and checks if a node has the `Snapshot` effect:

```js
function commitBeforeMutationLifecycles() {
    while (nextEffect !== null) {
        const effectTag = nextEffect.effectTag;
        if (effectTag & Snapshot) {
            const current = nextEffect.alternate;
            commitBeforeMutationLifeCycles(current, nextEffect);
        }
        nextEffect = nextEffect.nextEffect;
    }
}
```

For a class component, this effect means calling the `getSnapshotBeforeUpdate` lifecycle method.

#### DOM updates

[`commitAllHostEffects`](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L376) is the function where React performs DOM updates. The function basically defines the type of operation that needs to be done for a node and executes it:

```
function commitAllHostEffects() {
    switch (primaryEffectTag) {
        case Placement: {
            commitPlacement(nextEffect);
            ...
        }
        case PlacementAndUpdate: {
            commitPlacement(nextEffect);
            commitWork(current, nextEffect);
            ...
        }
        case Update: {
            commitWork(current, nextEffect);
            ...
        }
        case Deletion: {
            commitDeletion(nextEffect);
            ...
        }
    }
}
```

It’s interesting that React calls the `componentWillUnmount` method as part of the deletion process in the `commitDeletion` function.

### Post-mutation lifecycle methods

[commitAllLifecycles](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L465) is the function where React calls all remaining lifecycle methods `componentDidUpdate` and `componentDidMount`.
















Contents from:
https://angularindepth.com/posts/1007/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-to-walk-the-components-tree

More readings:
https://github.com/acdlite/react-fiber-architecture?source=post_page---------------------------
https://www.youtube.com/watch?v=ZCuYPiUIONs&source=post_page---------------------------