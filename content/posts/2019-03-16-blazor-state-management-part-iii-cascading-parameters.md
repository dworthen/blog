---
template: post
title: Blazor State Management Part III - Cascading Parameters
slug: /posts/blazor-state-management-3-cascading-parameters/
draft: false
date: '2019-03-16'
description: >-
  Cascading parameters provides a way to share data and behavior from ancestor
  components to child components without having to manually pass parameters down
  in intermediate components. Components can access the data it needs without
  requiring parent components to manage data dependencies. In this article, we
  will explore using cascading parameters, along with event delegation, in order
  to share data throughout a component tree.
category: Blazor
tags:
  - Blazor
---
> **This article explores cascading values in Blazor 0.7.0**
>
> **The source code for this article can be found [here](https://github.com/dworthen/BlazorStateManagement/tree/part-03-cascading-parameters)**

**This article is part of a Blazor state management exploration series.**

1. [Blazor State Management Part I - Data-Binding](/posts/blazor-state-management-1-data-binding)
2. [Blazor State Management Part II - Event Delegation](/posts/blazor-state-management-2-event-delegation)
3. Blazor State Management Part III - Cascading Parameters

- - -

[The last article](/posts/blazor-state-management-2-event-delegation) ended with a question. How do we share data within the following component structure?

![Cascading parameters.](/media/event-delegation-complicated.png)

With event delegation, we could update middle components to accept a `person` parameter and pass it through to child components that need it. We would also need to pass event handlers through the middle components. A doable, but a messy and cumbersome process.

Ideally, we would pass parameters from the parent component directly to child components that need them, no matter how nested the child components are. Well, it turns out, we can. [CascadingValue](https://docs.microsoft.com/en-us/aspnet/core/razor-components/components?view=aspnetcore-3.0#cascading-values-and-parameters) is a unique tag that allows us to hoist parameters for child components, even nested ones, to acccess. This should feel familiar to anyone who has used [React context](https://reactjs.org/docs/context.html).

Let's create a `MiddleComponent` to contain the `DisplayPerson` and `UpdatePerson` components.

```aspnet
@* MiddleComponent.cshtml *@
<div class="boxed">
    <h4>DisplayPerson Component</h4>
    <DisplayPerson></DisplayPerson3>
</div>
<div class="boxed">
    <h4>UpdatePerson Componet</h4>
    <UpdatePerson></UpdatePerson3>
</div>
```

Notice that we are no longer passing parameters to `DisplayPerson` or `UpdatePerson`. This is because the components will check the hoisted space for the values instead of receiving them directly as parameters. Update `index.cshtml` to use the newly created `MiddleComponent`.

```aspnet
@* index.cshtml *@
...
<MiddleComponent></MiddleComponent>

@functions {
    protected Person person { get; set; } = new Person { Name = "Derek" };
    ...
}
```

This will not work. Remember, `DisplayPerson` and `UpdatePerson` need a `Person` object to work with. So let's wrap `MiddleComponent` in a `CascadingValue` component to hoist our `person` object.

```aspnet
@* index.cshtml *@
...
<CascadingValue Value="@person">
 	<MiddleComponent></MiddleComponent>
</CascadingValue>
...
```

This is still not enough. `UpdatePerson` also expects a function parameter. A function that will handle updating the state when necessary. Each `CascadingValue` can only hoist one value so we will need to nest tags. 

```aspnet
@* index.cshtml *@
<CascadingValue Value="@person">
    <CascadingValue 
        Value="@HandleChange" 
        Name="HandleChange" 
        T="Action<UIChangeEventArgs>">
        <MiddleComponent></MiddleComponent>
    </CascadingValue>
</CascadingValue>

@functions {
    protected Person person { get; set; } = new Person { Name = "Derek" };
    protected void HandleChange (UIChangeEventArgs e)
    {
        person.Name = e.Value.ToString();
        StateHasChanged();
    }
}
```

The second `CascadingValue` component is hoisting a named parameter. Since the parameter is a method, we need to help Blazor identify the parameter type by specifying the type with the `T` parameter.

Next, we need to update `UpdatePerson` and `DisplayPerson` to use `CascadingParameters` instead of normal parameters. We only need to update the decorators to specify `CascadingParameters`. Nothing else needs to change.

```apsnet
@* UpdatePerson.cshtml *@
...
@functions {
    [CascadingParameter] 
    protected Person person { get; set; }
    [CascadingParameter(Name = "HandleChange")] 
    protected Action<UIChangeEventArgs> CustomOnChange { get; set; }
    ...
}
```

```aspnet
@* DisplayPerson.cshtml *@
...
@functions {
    [CascadingParameter] protected Person person { get; set; }
}
```

Our application should work as it did at the end of the event delegation article. Updating the person's name using the input field should update the name displayed in all components. 

- - -

## Conclusion

Yes, it is possible to use cascading parameters to share state throughout an application, but it is messy. The Blazor docs use CSS style rules or theme information as an example for using cascading parameters. read-only UI state or theme information are good candidates for cascading parameters. Otherwise, not only do the values need to be cascaded but so do the methods that will be responsible for updating the hoisted values. Not to mention, `CascadingValue` can only hoist one value at a time, requiring nested tags for each cascading value.
