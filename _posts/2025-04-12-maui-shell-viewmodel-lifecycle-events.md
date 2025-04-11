---
title: ".NET MAUI Shell: ViewModel Lifecycle Events"
categories:
  - Blog
tags:
  - .NET MAUI
  - .NET MAUI Shell
classes: wide
---

Explore an approach to .NET MAUI Shell ViewModel initialisation and re-initialisation that covers many scenarios, and see it applied in practice.

Written in collaboration with: <i class="fab fa-github"></i> [Toby Field](https://github.com/TobyDevelopment).

## The problem

.NET MAUI does not have a built-in or opinionated way to perform ViewModel initialisation or re-initialisation.

This is a necessity in most apps for many different use cases:
 - making API calls to retrieve and display data
 - performing local database queries
 - other long running tasks

As a result, many different approaches are commonly suggested, including:
 - using `Page.OnAppearing` or `Page.OnNavigatedTo`
 - setting up a complicated `NavigationService`

However, these can all have their own drawbacks without customisation.

> **Example**<br>
> Determining whether `Page.OnNavigatedTo` was fired from switching tabs or navigating backwards, to fire a re-initialise method.

## A pretty nice solution

What does a nice solution look like?

1. We want a simple and self-contained system that can fire virtual `InitialiseAsync` and `ReinitialiseAsync` methods on the `BaseViewModel` which can be overridden in ViewModel implementations.

2. `BaseViewModel.InitialiseAsync` should be fired the first time a page/modal is navigated to.

    This includes the initial page on app launch, during relative and global navigation, the first time a tab is switched to, and each time a flyout item is navigated to.

3. `BaseViewModel.ReinitialiseAsync` should be fired when navigating back to a page.

    This can be triggered by calling `Shell.Current.GoToAsync("..")`, tapping the software/hardware back buttons, and using swipe back gestures.

4. The solution should use the existing Shell parameter passing system for forwards, backwards and root navigation.

### Implementation

The below approach uses the `ShellNavigatedEventArgs.Source` property within `Shell.OnNavigated` to determine which ViewModel lifecycle event to run:

- `ShellNavigationSource.ShellItemChanged` is fired when navigating to a root page.

    This happens when:
    - the app launches, and the first page is opened
    - navigating to a different root page using `Shell.Curent.GoToAsync("//Route")`
    - navigating to a different flyout item<br><br>

- `ShellNavigationSource.ShellSectionChanged` is fired when switching tabs.

    We track whether the ViewModel has initialised to stop tabs from being initialised more than once.

- `ShellNavigationSource.Push` is fired when a page is navigated to using relative routes.

- `ShellNavigationSource.Pop` & `ShellNavigationSource.PopToRoot` are fired when navigating back.

#### AppShell.xaml.cs

```csharp
protected override void OnNavigated(ShellNavigatedEventArgs args)
{
  base.OnNavigated(args);
  
  if (this.CurrentPage.BindingContext is not BaseViewModel viewModel)
  {
    return;
  }
      
  switch (args.Source)
  {
    case ShellNavigationSource.ShellItemChanged:
    case ShellNavigationSource.ShellSectionChanged:
      if (viewModel.HasInitialised)
      {
        return;
      }

      _ = viewModel.InitialiseAsync();
              
      viewModel.HasInitialised = true;

      return;
          
    case ShellNavigationSource.Push:
      _ = viewModel.InitialiseAsync();
              
      viewModel.HasInitialised = true;
              
      return;
          
    case ShellNavigationSource.Pop:
    case ShellNavigationSource.PopToRoot:
      _ = viewModel.ReinitialiseAsync();
      return;
  }
}
```

#### BaseViewModel.cs

```csharp
public class BaseViewModel : ObservableObject
{
    internal bool HasInitialised { get; set; } = false;
    
    public virtual Task InitialiseAsync()
    {
        return Task.CompletedTask;
    }
    
    public virtual Task ReinitialiseAsync()
    {
        return Task.CompletedTask;
    }
}
```

### Navigation service

The navigation service can be extremely simple using this solution, and leverage the existing Shell navigation parameter functionality.

> Implementing a navigation service is not required for this approach, but doing so can improve the testability of your ViewModels.

```csharp
public class NavigationService : INavigationService
{
    public async Task GoToAsync<TView>(Dictionary<string, object>? parameters = null)
        where TView : ContentPage
    {
        parameters ??= [];
        
        await Shell.Current.GoToAsync(typeof(TView).Name,
            new ShellNavigationQueryParameters(parameters));
    }
    
    public async Task GoToRootAsync<TView>(Dictionary<string, object>? parameters = null)
        where TView : ContentPage
    {
        parameters ??= [];
        
        await Shell.Current.GoToAsync($"//{typeof(TView).Name}",
            new ShellNavigationQueryParameters(parameters));
    }

    public async Task GoBackAsync(Dictionary<string, object>? parameters = null)
    {
        parameters ??= [];

        await Shell.Current.GoToAsync("..",
            new ShellNavigationQueryParameters(parameters));
    }
}
```

## Current limitations

When navigating to a root page using global routes (`//Route`), `ApplyQueryAttributes` is fired after `Shell.OnNavigated`/`virtual BaseViewModel.InitialiseAsync`.

This is caused by<br>
[dotnet/maui Issue 24241 - Maui Shell weird navigation issue with timing of ApplyQueryAttributes and Page Lifecycle](https://github.com/dotnet/maui/issues/24241).

This will hopefully be fixed in a future .NET MAUI release.

## Results

The below video showcases a sample app implementing our approach for ViewModel initialisation and reinitialisation.

It covers scenarios including normal, modal, and root navigation, as well as passing navigation data in each case.

<video width="100%" controls>
  <source src="/assets/videos/Chill-Weather-Showcase.mp4" type="video/mp4">
</video>

## Source Code

The source code for the sample app can be found in the [ChillWeather.Mvvm.Maui Repo](https://github.com/eth-ellis/ChillWeather.Mvvm.Maui).

## Feedback welcomed

We would love to hear if this helped anyone, or whether there are any use cases this does not cover.

Please raise any feedback on the the [ChillWeather.Mvvm.Maui Discussion Board](https://github.com/eth-ellis/ChillWeather.Mvvm.Maui/discussions).