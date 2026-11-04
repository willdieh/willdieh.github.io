---
layout: series-post
title: "Working Through cpp/winrt Part 9 - Commands"
date: 
series: "cpp/winrt"
published: false
---

## WinRT and XAML Commands ##

### The Problem with Click Handlers ###

In [a previous entry]({% post_url 2026-09-30-winrt-3 %}) we discussed Click Handlers in the context of setting up menu actions. The basic workflow was to create a function in our `MainWindow.xaml.cpp` file and bind it to the relevant XAML object in `MainWindow.xaml` using the `Click=` attribute. The Click Handler can either be standard synchronous functions for quick actions that won't impede the main UI thread or `winrt::fire_and_forget` coroutine functions that suspend and return to the main thread until they perform some action in the background and are then resumed. Pretty standard fare and easy to grok. Very similar to writing callback functions in good 'ol Win32.

So what's the problem then? 

Well, one problem is you end up with a bunch of functions tightly integrated into your main window class. You can't really use them with other windows or objects without re-writing them and your window class can become bloated with functions that it would be nice to place elsewhere for maintenance.

Perhaps a more challenging issue with Click Handlers is how to *disable* a menu command (ie, grey it out on the menu) when its action is not possible to complete. For example, disabling `Save` on the menu until changes are made in a file. With standard Click Handlers, you have to manually update the XAML object whenever state changes to change the appearance and disable the click functionality.

Wouldn't it be great if we could create independent objects that perform actions and can be linked to XAML objects? Not only that, but could even hold properties like icons, labels, keyboard shortcuts, availability indicators, and descriptions. Say hello to my little friend, **XAML Commands**!

[Documentation Here](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/commanding)

## Command Basics ##

XAML Commands come in three flavors, from pre-defined to completely customizable. Each class derives from its more basic parent class:

- StandardUICommand: Preconfigured to support several basic system commands
- XamlUICommand: Provides preset customizable functions to create custom commands
- ICommand: Completely customizable base Command class

### StandardUICommand ###

We'll start off with the `StandardUICommand` as it offers built-in functionality for most basic system functions, including labels, icons, and keyboard shortcuts:

| Command/Label | Description | Icon | Keyboard shortcut |
| --- | --- | --- | --- |
| Cut | Delete the selected content and copy it to the clipboard. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e8c6.png)E8C6 | Ctrl + X |
| Copy | Copy the selected content to the clipboard. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e8c8.png)E8C8 | Ctrl + C |
| Paste | Insert the the clipboard content. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e77f.png)E77F | Ctrl + P |
| Select All | Select all content. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e8b3.png)E8B3 | Ctrl + A |
| Delete | Delete the selected content. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e74d.png)E74D | Delete |
| Share | Share the selected content. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e72d.png)E72D | n/a |
| Save | Save changes. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e74e.png)E74E | Ctrl + S |
| Open | Open panel. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e8a0.png)E8A0 | Ctrl + O |
| Close | Close panel. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e89f.png)E89F | Alt + F4 |
| Pause | Pause currently playing media. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e769.png)E769 | Spacebar |
| Play | Play currently paused media. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e768.png)E768 | Spacebar |
| Stop | Stop media. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e71a.png)E71A | Escape |
| Forward | Skip forward. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e72a.png)E72A | Alt + Right arrow |
| Back | Skip back. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e72b.png)E72B | Alt + Left arrow |
| Undo | Revert the most recent action. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e7a7.png)E7A7 | Ctrl + Z |
| Redo | Repeat the most recently undone action. | ![](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.input/images/segoe-fluent-icons/e7a6.png)E7A6 | Ctrl + Y |

See that big shiny **Open** Command? That's what we'll use in our project to replace our current Click Handler! 

**Note**
>Before we continue, I just wanted to point out that our test application is really basic so implementing a XAML Command might be a bit overkill when our Click Handler is working just fine. The [Documentation](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/commanding#command-experiences-using-the-icommand-interface) even says we might be fine with just the Click Handler and a bound variable to update the Open menu item `IsEnabled` attribute.
>
>Since we never updated the `IsEnabled` attribute on the Open menu item when executing the Click Handler, we technically have a bug. While the `FileOpenPicker` window is modal and will prevent input on the parent `MainWindow` while open, once the coroutine suspends and the background file load process initiates, there might be a millisecond or two when another Open request could be submitted. 
>
>So we accidentally set ourselves up with an excuse to implement the XAML Open Command because it will give us an easy way to take care of all that while teaching us the "best" way to do things.

Let's start off by updating our `MainWindow.xaml` file:

**`MainWindow.xaml`**
```xaml
<?xml version="1.0" encoding="utf-8"?>
<Window
   ...
    <Grid>
        <Grid.Resources>
            <StandardUICommand
               x:Name="OpenCommand"
               Kind="Open"
               ExecuteRequested="OnOpenExecuteRequested"/>
        </Grid.Resources>
        ...
        <MenuBar Grid.Row="0">
            <MenuBarItem Title="File" AccessKey="F" >
                <MenuFlyoutItem Command="{StaticResource OpenCommand}" AccessKey="O"/>                
                <MenuFlyoutItem Text="Exit" Click="OnExitClick" AccessKey="X"/>
            </MenuBarItem>
            ...
        </MenuBar>
        ...
    </Grid>
</Window>
```

To use the `StandardUICommand`, we first added it to the `Grid`'s local `Resources`. We set its `x:Name` and `Kind` attributes followed by the `ExecuteRequested` function name attribute. By setting `x:Name` we will now have a way to access this object in our cpp code using something like 
```c++
auto c = OpenCommand();
```

Once defined as a resource, it can be used by name within the `Grid` so we use it in the Open `MenuFlyoutItem`. Notice how we were able to remove the `Text=` and `Click=` attributes and even the entire `KeyboardAccelerator` section - all these will now be set from `OpenCommand`. We still had to define the `AccessKey` which is a bummer, but that's a menu specific setting so I guess it's understandable.

It’s still possible to override what `OpenCommand` provides. If we wanted a label other than the default “Open,” we could set `Text=` on the `MenuFlyoutItem` to change it just for that item, or set `Label=` on the `StandardUICommand` itself to change it everywhere the command is used.

We still need to create our `OnOpenExecuteRequested()` function in cpp to match the `ExecuteRequested` attribute. Let's do that now:

**`MainWindow.xaml.h`**
```c++
...
#include <winrt/Microsoft.UI.Xaml.Input.h>  // For XamlUICommand, ExecuteRequestedEventArgs
...
namespace winrt::WinUIApp::implementation
{
   struct MainWindow : MainWindowT<MainWindow>
   {
   ...
   // Command ExecuteRequested Handlers
   winrt::fire_and_forget OnOpenExecuteRequested(
    winrt::Microsoft::UI::Xaml::Input::XamlUICommand const& sender,  
    winrt::Microsoft::UI::Xaml::Input::ExecuteRequestedEventArgs const& args);
   
   // Click handlers
   void OnExitClick(IInspectable const& sender, RoutedEventArgs const& e);
   void OnOrientationClick(IInspectable const& sender, RoutedEventArgs const& e);
   void OnSizeClick(IInspectable const& sender, RoutedEventArgs const& e);
   ...
   }
}
...
```

**`MainWindow.xaml.cpp`**
```c++
...
   namespace winrt::WinUIApp::implementation
   {
       winrt::fire_and_forget MainWindow::OnOpenExecuteRequested(
           [[maybe_unused]] winrt::Microsoft::UI::Xaml::Input::XamlUICommand const& sender, 
           [[maybe_unused]] winrt::Microsoft::UI::Xaml::Input::ExecuteRequestedEventArgs const& args)
       {
       ...   // Same code as original OnOpenClick()
       }
   ...
   }
...
```

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
