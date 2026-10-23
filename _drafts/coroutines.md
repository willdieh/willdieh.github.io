---
layout: post
title: "Coroutines"
date: 
series: 
published: false
---

```text
Task read_file()
    Task::promise_type::get_return_object()
	co_await Task::promise_type::initial_suspend()
	...
	co_await read_async{}
		read_async::await_ready()
			return false
		[generate std::coroutine_handle]
		read_async::await_suspend(handle)
			... stash handle+state ...
			return true
	Task::promise_type::return_void()
	co_await Task::promise_type::final_suspend()
	[suspend/resume]
	...
```  

**`coroutines.cpp`**
```c++
#define WIN32_LEAN_AND_MEAN
#define NOMINMAX
#include <Windows.h>

#include <coroutine>
#include <iostream>
#include <vector>
#include <string>

// Task: our coroutine return type 
// Has a 'promise_type' and a std::coroutine_handle
// [[nodiscard]] requires an object assignment so the coroutine is preserved
//				 until the object is destroyed.
struct [[nodiscard("Must assign to object to preserve coroutine during suspension!")]] 
Task
{
	struct promise_type
	{
		// override new() so we can see coroutine frame allocation in debug
		void* operator new(size_t size)
		{
			return ::operator new(size);
		}

		// override constructor so we can see coroutine instatiation in debug
		promise_type() {}

		//  creates the object returned to the caller
		Task get_return_object()
		{ 
			return Task(std::coroutine_handle<promise_type>::from_promise(*this));
		}
		
		// returns awaitable, co_awaited before body runs
		// DON'T suspend immediately on entry
		std::suspend_never initial_suspend() 
		{ 
			return {}; 
		}
		
		// returns awaitable, co_awaited after body runs [noexcept]
		// DO suspend at end (to allow processing of result and cleanup)
		std::suspend_always final_suspend() noexcept 
		{ 
			return {}; 
		}
		
		// handles exceptions thrown in the body (RETHROW)
		void unhandled_exception() 
		{ 
			throw; 
		}

		// either return_void() or return_value(v) - define co_return behavior
		void return_void() 
		{
			return;
		}
	};

	explicit Task(std::coroutine_handle<promise_type> handle) : handle(handle) {}
	Task(Task&& other) noexcept : handle(other.handle) { other.handle = {}; }
	Task(const Task&) = delete;
	Task& operator=(const Task&) = delete;
	Task& operator=(Task&&) = delete;
	~Task() { if (handle) handle.destroy(); }

	std::coroutine_handle<promise_type> handle;
};


struct pending_read
{
	OVERLAPPED* ov;
	std::coroutine_handle<> handle; 
};

std::vector<pending_read> pending;


struct read_async 
{
	HANDLE file;
	char* buffer;
	DWORD size;
	OVERLAPPED ov{};
	DWORD error = 0;

	bool await_ready() { return false; }
	bool await_suspend(std::coroutine_handle<> handle)
	{
		BOOL ok = ReadFile(file, buffer, size, 0, &ov);
		
		if (ok) return false; // finished instantly, don't suspend
		
		DWORD err = GetLastError();

		if (err != ERROR_IO_PENDING)
		{
			error = err;
			return false; // failed, don't suspend
		}

		pending.push_back(pending_read{ &ov, handle });
		
		return true;
	}
	DWORD await_resume()
	{
		if (error)
		{
			std::cout << "ReadFile failed: " << error << "\n";
			return 0;
		}
		DWORD bytes = 0;
		GetOverlappedResult(file, &ov, &bytes, FALSE);
		return bytes;
	}

};

Task read_file(const wchar_t* path)
{
	HANDLE file = CreateFileW(path, GENERIC_READ, FILE_SHARE_READ, 0, OPEN_EXISTING, FILE_FLAG_OVERLAPPED | FILE_FLAG_NO_BUFFERING, 0);

	if (file == INVALID_HANDLE_VALUE)
	{
		std::cout << "CreateFileW failed: " << GetLastError() << "\n";
		co_return;
	}

	const DWORD size = 3 * 1024 * 1024;
	char* buffer = static_cast<char*>(VirtualAlloc(0, size, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE));
	
	if (!buffer)
	{
		CloseHandle(file);
		co_return;
	}

	std::cout << "read_file: starting read\n";
	DWORD bytes = co_await read_async{ file, buffer, size };
	std::cout << "read_file read " << bytes << " bytes\n";

	VirtualFree(buffer, 0, MEM_RELEASE);
	CloseHandle(file);
}

int main(int argc, char** argv)
{
	// Get the process's directory for use in filepath
	std::wstring path(MAX_PATH, L'\0');
	DWORD len = GetModuleFileNameW(0, path.data(), MAX_PATH);
	if (!len || len == MAX_PATH)
	{
		std::cout << "Error determining working directory.\n";
		return 1;
	}
	path.resize(len);
	path.erase(path.find_last_of(L"\\/") + 1, path.length());
	path += L"test.dat";
	
	// Start our read_file() coroutine!
	// Opens file in OVERLAPPED/UNCACHED mode and allocates 3MB buffer in virtual memory.
	// As soon as co_await is called, the awaitable's await_ready() function is called
	// to see if a result is already available. It isn't so a std::coroutine_handle is 
	// generated. 
	// Then the awaitable await_suspend() function is called with the handle.
	// await_suspend()'s goal is to store the handle and some state so we can resume when
	// state is where we want it. 
	// Here, await_suspend() calls ReadFile(), passing an OVERLAPPED data struct for state.
	// After an initial error/completion check, pushes the handle+OVERLAPPED into a queue. 
	// awaitable_suspend() returns true, the coroutine suspends, main() resumes.
	// return_void() is called ?????
	Task t = read_file(path.c_str());

	
	int other_work = 0;
	while (!pending.empty())
	{
		for (size_t i = 0; i < pending.size();)
		{
			if (HasOverlappedIoCompleted(pending[i].ov))
			{
				std::coroutine_handle<> handle = pending[i].handle;
				pending.erase(pending.begin() + i);
				handle.resume();
			} 
			else
			{
				++i;
			}
		}
		++other_work;
	}

	std::cout << "scheduler did " << other_work << " units of other work during the read\n";
}
```
