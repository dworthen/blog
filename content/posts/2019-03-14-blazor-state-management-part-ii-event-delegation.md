---
template: post
title: Blazor State Management Part II - Event Delegation
slug: /posts/blazor-state-management-2-event-delegation/
draft: false
date: '2019-03-14'
description: >-
  Data-binding is a useful concept and tool. In practice, data-binding lets
  Blazor developers manage and manipulate local component state and rest easy
  knowing the component UI is always working with, and reflecting the current
  state. The problem, data-binding does not work with state shared across
  components. In this article, we explore event delegation. A technique for
  handling state changes and for sharing state across components. 
category: Blazor
tags:
  - Blazor
---
> **This article explores event delegation in Blazor 0.7.0**
>
> **The source code for this article can be found [here](https://github.com/dworthen/BlazorStateManagement/tree/part-02-event-delegation)**

**This article is part of a Blazor state management exploration series.**

1. [Blazor State Management Part I - Data-Binding](./blazor-state-management-1-data-binding)
2. Blazor State Management Part II - Event Delegation

---

[Previously](./blazor-state-management-1-data-binding), we developed a sample application in order to explore data-binding in Blazor. Here is the current component structure of our sample application.

![data-binding component structure](/media/component-structure.png)

The index page shares a `Person` object to two components, `DisplayPerson` and `UpdatePerson`. Changes to `person`  that occur in `UpdatePerson` do not show up in `DisplayPerson`. At least, not automatically. This behavior was observed and discussed in the previous article. 

So how does one share objects across components and ensure that components are working with, and reflecting the most up-to-date state? One way to solve this problem is to use event delegation.

> **Event delegation** refers to the process of using event propagation (bubbling) to handle events at a higher level in the DOM than the element on which the event originated. 
>
> <https://learn.jquery.com/events/event-delegation/>

Instead of handling events in `UpdatePerson`, events are bubbled up to the `index` page. The `index` page will handle the event, make changes to `person` and propagate the changes to child components. `UpdatePerson` and `DisplayPerson` in this case. 

- - -

## Event handler

Add the following `HandleChange` method to the `index.cshtml` page. `HandleChange` will respond to input changes that occur in `UpdatePerson`, updating the `person`'s name and calling `StateHasChanged` to rerender the UI.

```aspnet
@* index.cshtml *@
...
@functions {
...
    private void HandleChange (UIChangeEventArgs e)
    {
        person.Name = e.Value.ToString();
        StateHasChanged();
    }
}
```

- - -

## Event Propagation (Bubbling)

The `HandleChange` event handler is in place. Now, we need to change `UpdatePerson` to bubble onchange events up to `index`/`HandleChange`. We can do this by adding a custom component parameter to `UpdatePerson`.

```apsnet
@* UpdatePerson *@
...
@functions {
    ...
    [Parameter] protected Action<UIChangeEventArgs> CustomOnChange { get; set; }

    private void OnChange(UIChangeEventArgs e)
    {
        CustomOnChange?.Invoke(e);
    }
}
```

The custom parameter, `CustomOnChange`, is defined as an `Action<UIChangeEventArgs>`. Actions are methods that except some list of parameters, `UIChangeEventArgs` in this case, and return nothing (null). Notice that this action defines a method that matches the `HandleChange` method signature. I think you can see where we are going here. 

> **[More in-depth guide to Actions](https://www.codementor.io/aydindev/delegates-func-act-in-c-sharp-du107s5mj)**

The final piece of the puzzle, wiring up the components. Update the `index` page to pass in `HandleChange` to `UpdatePerson`. 

```aspnet
@* index.cshtml *@
...
<UpdatePerson person="@person" 
    CustomOnChange="@HandleChange"></UpdatePerson>
...
```

Essentially, we are exposing the onchange event from `UpdatePerson` to `index` as a custom event, `CustomOnChange`, and handling the event with `HandleChange`. 

Launching this application and updating the name in the input field will show that `DisplayPerson` now updates to reflect the current name. Finally, we have a way to share objects across components.

- - -

## Conclusion

We started with one component structure/relationship and, through this article, and minor tweaks, we ended up with

![event delegation relationship](/media/event-delegation.png)

Event delegation provides a way to organize components in a way that allows for sharing state across components. The process for using event delegation for this purpose is to 1) identify a parent component that will "own" the data and 2) bubble events/changes from child components up to the parent component, the one that "owns" the data, to handle changes. 

There are some drawbacks to this method of sharing state. For example, how do we handle the following case?

![Event delegation with nested components.](/media/event-delegation-complicated.png)

In this case, `DisplayPerson` and `UpdatePerson` still need access to `person` but the middle components know nothing about `person` and don't need to. We could update middle components to accept `person` parameters and pass them through to the child components that need it. We would also need to pass event handlers from the parent component through middle components down to child components. A doable process but messy and cumbersome. We will explore alternative solutions for handling this case in future articles.

Event delegation is only one of many methods for sharing data across components. We will examine other methods in future articles.
