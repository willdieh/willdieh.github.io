---
layout: series-post
title: "Working Through cpp/winrt Part 6 - Collections"
date: 
series: "cpp/winrt"
published: false
---

## Collections ##

So far we've only put the sample `Int32` on the UI and bound it to our C++ code using `x:Bind` with a `TextBlock` and `Click=` with a `MenuBar` in our XAML code. The `Int32` was already part of our `MainWindow.idl` runtime class definition file from the WinUI template. Remember - any data that needs to be accessed from both the XAML UI and our C++ code must be defined in an `idl` file so it can be accessed in both the XAML and C++ worlds.

What if we were designing an Address Book and wanted to have something like a `std::vector<Person>` appear on the screen? 

Because we're going to access it from both the XAML UI and C++, we'll need to define it in an `idl` file. Then we'd need to define the `Person` class or struct and put it in a `std::vector` like thing in C++, and finally add it to the XAML UI by using an appropriate UI element like a `ListView`. The `INotifyPropertyChanged` requirements from [last time]({% post_url 2026-10-02-winrt-5 %}) will still be there. We'll also have to figure out how to handle any multi-threaded update conflicts between the UI and application threads. 

If you're up for the challenge, let's get started!

### IDL Files ###

Any data crossing the XAML UI and C++ ABIs must be defined in an `idl` file. So far, we've just used the `MainWindow.idl` file which defines the `MainWindow` class as being a `Microsoft.UI.Xaml.Window` object. The default WinUI template threw in an `Int32 MyProperty` member which we made *observable* by extending the runtime class with `Microsoft.UI.Xaml.Data.INotifyPropertyChanged`. 

Going forward, we'll be writing our custom runtime classes in separate `idl` files. This allows us to separate our `MainWindow` runtime class and our individual data structure runtime classes. We'll then `import` the separate runtime class definitions in any `idl` file they're needed. 

Honestly, this seems like a purely cosmetic concession as even the Microsoft documentation says having all your runtime class definitions in a single file can improve build times and may even be necessary to avoid circular includes. I'm really tempted to just write multiple runtime class definitions in the `MainWindow.idl` file, but I guess style wins over performance.

### Out with the Old ###

First, let's remove the `Int32 MyProperty` stuff (including the `INotifyPropertyChanged`) from `MainWindow.idl`.

**`MainWindow.idl`**
```idl
namespace WinUIApp
{
    [default_interface]
    runtimeclass MainWindow : Microsoft.UI.Xaml.Window
    {
        MainWindow();
    }
}
```

Next, remove all the `MyProperty` stuff and the `INotifyPropertyChanged` event handlers from `MainWindow.xaml.h`\`.cpp`. 

**`MainWindow.xaml.h`**
```c++
#pragma once

#include "MainWindow.g.h"

using namespace winrt::Microsoft::UI::Xaml; // For RoutedEventArgs
using namespace winrt::Microsoft::UI::Xaml::Data; // For PropertyChangedEventHandler

namespace winrt::WinUIApp::implementation
{
    struct MainWindow : MainWindowT<MainWindow>
    {
        MainWindow()
        {
            // Xaml objects should not call InitializeComponent during construction.
            // See https://github.com/microsoft/cppwinrt/tree/master/nuget#initializecomponent
        }

        // Click handlers
        winrt::fire_and_forget OnOpenClick(IInspectable const& sender, RoutedEventArgs const& e);
        void OnExitClick(IInspectable const& sender, RoutedEventArgs const& e);
        void OnOrientationClick(IInspectable const& sender, RoutedEventArgs const& e);
        void OnSizeClick(IInspectable const& sender, RoutedEventArgs const& e);

    private:

    };
}

namespace winrt::WinUIApp::factory_implementation
{
    struct MainWindow : MainWindowT<MainWindow, implementation::MainWindow>
    {
    };
}
```

**`MainWindow.xaml.cpp`**
```c++
#include "pch.h"
#include "MainWindow.xaml.h"
#if __has_include("MainWindow.g.cpp")
#include "MainWindow.g.cpp"
#endif


#include <winrt/Microsoft.UI.Windowing.h>   // For AppWindow
#include <winrt/Microsoft.Windows.Storage.Pickers.h>    // For FileOpenPicker

using namespace winrt::Microsoft::Windows::Storage::Pickers;    // For FileOpenPicker
using namespace winrt::Microsoft::UI::Xaml::Controls;   // For RadioMenuFlyoutItem

// To learn more about WinUI, the WinUI project structure,
// and more about our project templates, see: http://aka.ms/winui-project-info.

namespace winrt::WinUIApp::implementation
{
    winrt::fire_and_forget MainWindow::OnOpenClick(IInspectable const& sender, RoutedEventArgs const& e)
    {
        // Return the handle id for this window (necessary for non-packaged apps)
        FileOpenPicker openPicker(this->AppWindow().Id());
        
        auto file{ co_await openPicker.PickSingleFileAsync() };
    }

    void MainWindow::OnExitClick(IInspectable const& sender, RoutedEventArgs const& e)
    {
        Application::Current().Exit();
    }

    void MainWindow::OnOrientationClick(IInspectable const& sender, RoutedEventArgs const& e)
    {
        // Cast the sender parameter as a RadioMenuFlyoutItem
        auto clickedItem = sender.try_as<RadioMenuFlyoutItem>();

        // If Cast was successful, check which radio button was clicked
        if (clickedItem)
        {
            if (clickedItem.Text() == L"Portrait")
            {
            }
            else if (clickedItem.Text() == L"Landscape")
            {
            }
        }
    }

    void MainWindow::OnSizeClick(IInspectable const& sender, RoutedEventArgs const& e)
    {
    }
}
```

Finally, remove the entire `<TextBlock>` from `MainWindow.xaml` - we won't be using that `Int32` any more. 

**`MainWindow.xaml`**
```xaml
<?xml version="1.0" encoding="utf-8"?>
<Window
    ...
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>
        <MenuBar Grid.Row="0">
        ...
        </MenuBar>
    </Grid>
</Window>
```

### In with the New ###

Now we're ready to create a shiny new runtime class definition, just for our `Person` object! 

Right-click on the project in Visual Studio and select Add, New Item..., and select **View Model (C++/WinRT)**. Name it `Person` and click Add.
