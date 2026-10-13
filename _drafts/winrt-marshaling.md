I’d suggest Part 7: Loading Data from a File.

It ties up loose ends from earlier posts:

The OnOpenClick handler from Part 3 picks a file but does nothing with it.
Part 6 explained marshaling between threads but never demonstrated it.

The post could cover reading a simple CSV or JSON file with Windows.Storage APIs, parsing it on a background thread with co_await winrt::resume_background(), switching back with co_await winrt::resume_foreground(DispatcherQueue()), and replacing the sample data in People. It also lets you show what happens when you skip the switch back: the cross-thread exception you described.
