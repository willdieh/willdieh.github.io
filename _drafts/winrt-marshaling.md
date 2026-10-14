---
layout: series-post
title: "Working Through cpp/winrt Part 7 - Marshaling"
date: 2026-10-13
series: "cpp/winrt"
published: false
---

## Marshaling ##

[Last time]({% post_url 2026-10-03-winrt-6 %}) we touched briefly on threading models in cpp/winrt. I don't claim to be any kind of expert on this topic but I'd like to work through an example that shows how things must generally be done on the main UI thread when it comes to updating UI elements to keep bad things from happening.

### Background ###

Because winrt runs through COM, behind the scenes it follows the COM workflow. Typically, when starting up COM in an old Win32 Windows application, you would initialize COM by first asking for a specific type of *apartment*. That apartment, in which all the COM activity happens, can be either single-threaded (default) or multi-threaded. You can work with multiple single-threaded apartments but can only ever have at most *one* multi-threaded apartment.

When working with objects *within* an apartment, objects are modified *directly* by the apartment's thread(s). 

When attempting to work with an object in *another* apartment, the object must be *marshalled* between the two apartments. This is done by calling special COM functions to create a proxied reference to the object and passing the reference to the external apartment.

External access to COM objects inside a single-threaded apartment is provided via the Windows message queue. By forcing all access through a single queue, data contention issues are avoided within that single-threaded apartment. 

External access to COM objects inside a multi-threaded apartment is *not* synchronized using the Windows message queue - it's not synchronized at all! Multiple external requests can be processed simultaneously by a multi-threaded apartment, each on a different apartment thread, so it is necessary to protect objects with proper synchronization primitives like mutexes. 

### WinUI and C++/Winrt ###

WinUI applications are built with XAML and default to a single-threaded apartment. If you look in the `Generated Files` folder of your project (after hitting Build Solution at least once to fire off the XAML compiler), you'll see an auto-generated `App.xaml.g.hpp` file that contains the default `wWinMain` function. In there you'll notice a call to `winrt::init_apartment(winrt::apartment_type::single_threaded);` whereby our single-threaded COM apartment is created. All XAML UI operations must happen on this single thread. XAML objects check the thread requesting access and throw `RPC_E_WRONG_THREAD` if it's not the main UI thread. 

Custom runtime classes, like our `Person` class, are by default a special kind of COM object known as an [*agile* object](https://learn.microsoft.com/en-us/windows/apps/develop/cpp-winrt/agile-objects). Agile objects implement the `IAgileObject` COM interface. These agile objects are declaring that they do not participate in the standard marshalling process. Instead, they are stating that they can be accessed by any apartment directly by pointer, without marshalling. Agile objects run on whatever thread attempts to access them. If multiple threads access an agile object, it must be properly thread synchronized in code.

It's possible to opt out of the default agile implementation by adding `winrt::non_agile` to the inherited types of a C++ runtime class implementation like `WinUIApp::implementation::Person : PersonT<Person, winrt::non_agile, ...>`. This would allow the object to provide the marshalling interfaces and be marshalled by reference using the standard COM functions as described above instead of as raw pointers.

If your agile runtime class needs to access and update a COM object that isn't agile (most *are* agile), you can wrap that non-agile object in an agile reference like `winrt::agile_ref<[non_agile_typename]> agileThing {nonAgileThing};` or `auto agileThing = winrt::make_agile(nonAgileThing);`. This would allow you to access that non-agile object from whatever thread your agile object is in using `agileThing.get()`, automagically performing marshalling into that non-agile object's apartment. The only catch is you have to create the reference on the same thread as the non-agile object (or properly marshall it onto your current thread). Sigh. Hope we don't need to do this.

I expect we will just be running on the main UI thread. For those situations where we need to do work on a background thread, it's probably easiest to just stay in the agile world where threads don't matter but always switch back to our main UI thread before updating the UI with our results. We can do this by capturing the main UI thread before switching to a background thread then switching back to the UI thread when we're done.

## Open File ##

Let's put all this into practice by working with the `FilePicker` to open a CSV file, process it on a background thread to create `Person` objects, then update our UI.

Our first run-through will be blissfully ignorant of threads and all that nonsense! Let's find out what happens!

### CSV Format ###

Create a new file in the project folder called `people.csv` and add a couple people, one line at a time like this:

**`people.csv`**
```csv
Alice,Johnson,1995-06-15,555-0101,alice@example.com
Bob,Martinez,1992-12-01,555-0102,bob@example.com
Carol,Nguyen,1975-04-05,555-0103,carol@example.com
Dan,Okafor,1990-02-20,555-0104,dan@example.com
Eve,Kowalski,1988-09-30,555-0105,eve@example.com
```

### `FilePicker` ###



