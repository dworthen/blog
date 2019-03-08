---
template: post
title: Blazor State Management Part I - Data-Binding
slug: /posts/blazor-state-management-1-data-binding/
draft: false
date: '2019-03-08'
description: >-
  Blazor.net supports data-binding but what does that mean? Are there
  limitations to the data-binding implementation? What about the
  Model-View-ViewModel design pattern (MVVM)? Does Blazor support the MVVM
  pattern? In this article, I explore data-binding in Blazor.
category: Blazor
tags:
  - Blazor
---
> **This article explores data-binding in Blazor 0.7.0**
>
> **The source code for this article can be found [here](https://github.com/dworthen/BlazorStateManagement/tree/part-01-data-binding).**

MVVM (Model-View-ViewModel) is a UI design pattern that separates the data layer, model, from the presentation layer, view. The pattern bridges the two layers with the ViewModel which is responsible for converting the model into a view-friendly form and for relaying view-driven updates back to the model.

![MVVM Diagram](/media/mvvm-concrete.png)

In the above example, notice how the model stores the date compared to how the view displays the date. The ViewModel converts the date back and forth between the model and the view.

An important part of the MVVM pattern is the communication between the view and ViewModel. When the ViewModel updates data the view automatically updates to reflect the changes. The opposite is also true. When the view updates data through user interactions, the ViewModel references the most up-to-date data. This is known as data-binding.

In short, data-binding means that the view always reflects the current ViewModel and the ViewModel stays in sync with updates that occur in the view, user-driven or otherwise.

Here is an example of data-binding from the Blazor documentation.

```aspnet
<input type="checkbox" class="form-check-input" id="italicsCheck" bind="@_italicsCheck" />
Current Value: @_italicsCheck

@functions {
	private bool _italicsCheck { get; set; } = false;
}
```

The above is a simple component. The properties defined in the functions section act as our ViewModel and are accessible to the above HTML, the view, through razor syntax. In this case, the view displays a checked checkbox when the ViewModel's `_italicsCheck` is true. Furthermore, the value stored in `_italicsCheck` updates as users toggle the checkbox in the UI to match the new toggled state. This functionality, data-binding, is achieved with the `bind` attribute.

Two-way data-binding out of the box! So Blazor supports the MVVM design pattern, right? Not quite. Let's examine what happens when data is used across components.

```aspnet
@* UpdateMessage.cshtml *@
<div>UpdateMessage Component Current Value: @Message</div>

<div>
    <input type="text" bind="@Message" />
</div>

@functions {
    [Parameter] string Message { get; set; }
}
```

```aspnet
@* DisplayMessage.cshtml *@
<div>Display Message Component Current Value: @Message</div>

@functions {
    [Parameter] string Message { get; set; }
}
```

And update `index.cshtml`

```aspnet
@* index.cshtml *@
@page "/"

<DisplayMessage Message="@Message"></DisplayMessage>
<UpdateMessage Message="@Message"></UpdateMessage>

@functions {
    private string Message { get; set; } = "Hello World";
}
```

Now, two components are using the same data. Let's see what happens when we update the message in UpdateMessage. Do updates propagate to the DisplayMessage component?

![data-binding a string](/media/data-bind-1.png)

Unfortunately, no. Data-binding is limited to the current component and child components. But wait! Like two-way binding on input tags, Blazor supports two-way binding on custom component parameters using the `bind-` prefix attribute. Let's implement that now.

```aspnet
@* index.cshtml *@
...
<DisplayMessage bind-Message="@Message"></DisplayMessage>
<UpdateMessage bind-Message="@Message"></UpdateMessage>
...
```

The `bind-` prefix allows one to bind data to custom-component parameters. Before refreshing the application, we need to add the following `Action` parameter to UpdateMessage and DisplayMessage.

```aspnet
@* DisplayMessage.cshtml and UpdateMessage.cshtml *@
...
@functions {
	...
	[Parameter] Action<string> MessageChanged { get; set; }
}
```

We will examine how this works in a future article. For now, cross your fingers!

![data-binding with bind attribute](/media/data-bind-2.png)

No luck! Surely, the issue is that we are sharing a string across components. Passing a string is to pass by value. So each component is receiving its own copy of the string. AHA! Let's try using a POCO as our state object.

```aspnet
@* Person.cs *@
public class Person
{
    public string Name { get; set; }
}
```

And repeat the same pattern from before...

```aspnet
@* DisplayPerson *@
<div>DisplayPerson Component - Person's Name: @person.Name</div>

@functions {
    [Parameter] Person person { get; set; }
}
```

```aspnet
@* UpdatePerson Component *@
<div>UpdatePerson Component - Person's Name: @person.Name</div>

<div>
    <input type="text" bind="@person.Name" />
</div>

@functions {
    [Parameter] Person person { get; set; }
}
```

Let's not forget to update `index.cshtml` to use the new Person components.

```aspnet
@* index.cshtml *@
...
<DisplayPerson person="@person"></DisplayPerson>
<UpdatePerson person="@person"></UpdatePerson>

@functions {
    ...
    private Person person { get; set; } = new Person { Name = "Derek" };
}
```

Refreshing our browser we get

![Data-binding with poco](/media/data-binding-with-poco.png)

Don't panic! Hope is not lost. Instead of relying on Blazor's data-binding let's try responding to the `onchange` event manually.

```aspnet
@* UpdatePerson.cshtml *@
...
	<input type="text" value="@person.Name" onchange="@OnChange" />
...
@functions {
    ...
    private void OnChange(UIChangeEventArgs e)
    {
        person.Name = (string)e.Value;
    }
}
```

If you refresh the browser, you will see that we still have the same functionality. Well, we didn't break anything. We still have component-scoped data-binding. Why use an event listener instead of bind? Using an event listener allows us to have side effects, such as calling `StateHasChanged`.

```aspnet
@* UpdatePerson.cshtml *@
private void OnChange(UIChangeEventArgs e)
{
    person.Name = (string)e.Value;
    StateHasChanged();
}
```

`StateHasChanged`, if you haven't guessed, is a Blazor function that tells the system that the state has changed which, in turn, triggers a rerender. Blazor renders UI similar to many popular JS frameworks. It maintains a virtual dom. When a rerender occurs, Blazor generates a new virtual dom, diffs it with the previous virtual dom and then minimally updates the real dom.

Maybe, just maybe, manually calling `StateHasChanged` causes Blazor to rerender and diff the entire virtual dom and not just the local component dom. And...

![Data-binding with statehaschanged](/media/data-binding-with-satehaschanged.png)

Turns out, `StateHasChanged` is scoped to the current component and child components. No different from bind.

What gives? DisplayPerson and UpdatePerson receive the same Person object. It is an object! It has to be passed by reference, right? This is true. DisplayPerson and UpdatePerson receive a reference to the same object. The problem lies in how rerendering works in Blazor.

Let's prove that the shortcoming lies within the rendering mechanism. Add the following code to DisplayPerson

```aspnet
...
@functions {
...
    protected async override Task OnInitAsync()
        {
            await base.OnInitAsync();
            System.Timers.Timer timer = new System.Timers.Timer(10000);
            await Task.Delay(6000);
            StateHasChanged();
        }
...
}
```

After 6 seconds, DisplayPerson calls `StatehasChanged`, triggering a rerender. This should give us enough time to load the page, update the state within the UpdatePerson component and wait and see if DisplayPerson will display the updated name after StateHasChanged has been called. Go quick. You have 6 seconds. If all goes well, DisplayPerson, after 6 seconds, should display the updated name since it is referencing the same object.

![Data-binding with a delay](/media/data-binding-with-a-delay.gif)

Notice that the name is updating to match user input

- - -

## Conclusion

In summary, Blazor supports component-based data-binding, a process for keeping the ViewModel and view in sync. Data-binding is component based and will not work across components. In future articles, I will examine methods for sharing data across components and ensuring components use and reflect the most up-to-date data.
