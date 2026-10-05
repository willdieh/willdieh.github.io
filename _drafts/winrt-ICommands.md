---
layout: series-post
title: "Working Through cpp/winrt Part 6 - Commands"
date: 
series: "cpp/winrt"
published: false
---

If you decide to write a Part 6 on this, you'll find that C++/WinRT handles commands primarily through Windows::UI::Xaml::Input::XamlUICommand (or custom implementations of the underlying ICommand interface).

A great outline for that post could look like this:

    The Problem with Code-Behind Click Handlers: A quick recap of why cluttering your window class with OnOpenClick / OnExit logic gets messy as an app scales.

    Introducing Commands: Explaining what Execute and CanExecute actually do under the hood in WinUI 3.

    Wiring It Up in C++: Creating a command object, binding it in XAML via {x:Bind MyCommand}, and handling parameters.

    Automatic State Management: Showing how CanExecute can automatically disable or enable menu items without manual UI tinkering.

It will make a fantastic capstone or next chapter to your series, especially since you've already conquered the mountain of setting up INotifyPropertyChanged!
