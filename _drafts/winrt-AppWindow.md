---
layout: series-post
title: "Working Through cpp/winrt Part 11 - AppWindow"
date: 
series: "cpp/winrt"
published: false
---

## AppWindow vs Window ##

`Window` represents the XAML presentation of a window. `AppWindow` represents the underlying `HWND` object and allows modifying window behavior.

The `Window` class is based on an `AppWindow`. To retrieve and modify the underlying `AppWindow` you call `auto var = this->AppWindow();` in the `Window` class code.

The most common reasons for working with the underlying `AppWindow` is to customize the toolbar, position, size, and presentation behavior.
