---
layout: series-post
title: "Working Through cpp/winrt Part 9 - Commands"
date: 
series: "cpp/winrt"
published: false
---

## WinRT and XAML Commands ##

### The Problem with Click Handlers ###

In [a previous entry]({% post_url 2026-09-30-winrt-3 %}) we discussed click handlers in the context of setting up menu actions. The basic workflow was to create a function in our `MainWindow.xaml.cpp` file and bind it to the relevant xaml object in `MainWindow.xaml` using the `Click=` attribute. The click handler can either be a standard synchronous function for quick actions that won't impede the main UI thread or `winrt::fire_and_forget` coroutine functions that suspend and return to the main thread until they perform some action in the background and are then resumed. Pretty standard fare and easy to grok. Very similar to writing callback functions in good 'ol Win32.

So what's the problem then? 

Well, one problem is you end up with a bunch of functions tightly integrated into your main window class. You can't really use them with other windows or objects without re-writing them and your window class can become bloated with functions that it would be nice to place elsewhere for maintenance.

Perhaps a more challenging issue with click handlers is how to *disable* a menu command (ie, grey it out on the menu) when its action is not possible to complete. For example, disabling `Save` on the menu until changes are made in a file. With standard click handlers, you have to manually update the xaml object whenever state changes to change the appearance and disable the click functionality.

Wouldn't it be great if we could create independent objects that perform actions and can be linked to xaml objects? Not only that, but could even hold properties like icons, labels, keyboard shortcuts, availability indicators, and descriptions. Say hello to my little friend, **XAML Commands**!

### Command and Conquer ###

[Documentation Here](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/commanding)






2. ICommand Basics
   What Execute, CanExecute, and CanExecuteChanged do, using
   Microsoft::UI::Xaml::Input (not the UWP Windows::UI::Xaml namespace).

3. Built-in Options: XamlUICommand and StandardUICommand
   Bundling label, icon, accelerator, and description; handling
   ExecuteRequested / CanExecuteRequested; using predefined commands
   like Cut, Copy, and Delete for menus.

4. Rolling Your Own: A Reusable DelegateCommand
   C++/WinRT has no RelayCommand equivalent, so build one with
   winrt::implements<..., ICommand> that wraps lambdas.

5. Wiring It Up: IDL, Binding, and Parameters
   Declaring the command property in .idl, binding with {x:Bind MyCommand},
   and unpacking CommandParameter from IInspectable with unbox_value / try_as.

6. Keeping UI State in Sync
   Raising CanExecuteChanged (or calling NotifyCanExecuteChanged) so menu
   items and buttons enable and disable correctly.
