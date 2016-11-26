# The Chimera called Coroutine

Seeing [Gor Nishanov's "Amazing coroutine disappearing act!"](https://youtu.be/8C8NnE1Dg4A)
, one cannot deny the potential of coroutines.

#### The Basics:
```cpp
void foo() {}
```
This is a subroutine

```cpp
coro_return_type<int> test()
{
	co_await coro_awaitable_type{};
}
```
This is a coroutine

---

#### Is this UB?
```cpp
#include <experimental\coroutine>

using namespace std::experimental;

coro_return_type<int> test()
{
	int local_val = 0;
	co_await coro_awaitable_type{};
	co_yield int(3);
	co_return int(5);
}

int main()
{
	auto handle = test();
	handle.resume();
	auto val = handle.value();
	handle.resume();
	auto val2 = handle.value();
}
```

To make it compile we *only* require:
* `coro_return_type::promise_type` specifies:  
  * `coro_return_type get_return_object()`  
  * `coro_awaitable_type initial_suspend()`  
  * `coro_awaitable_type final_suspend()`
  * `void yield_value(T val)`
  * `void return_value(T val)` and not `void return_void()`
* `T val` inside `coro_return_type::promise_type` is the same across the board
* `coro_return_type` specifies:
 * `resume()`
 * `T value()`

Whether it is UB *only* depends on:
* `bool coro_awaitable_type()::await_ready()`
* `bool coro_return_type::promise_type::initial_suspend()::await_ready()`
* `bool coro_return_type::promise_type::final_suspend()::await_ready()`

`await_ready()` return implication  
`false` suspend  
`true` do not suspend

The *obvious* implications for our example:
1. `false, false, false` val is undefined
2. `false, false, _true` val is undefined
3. `false, _true, false` **Program works as intended**
4. `false, _true, _true` val2 is undefined
5. `_true, false, false` local_val is undefined before first resume
6. `_true, false, _true` 2nd resume is UB
7. `_true, _true, false` val has the wrong value and 2nd resume is UB
8. `_true, _true, _true` 1st resume is UB

#### Possible Implementation
```cpp
struct coro_awaitable_type
{
	bool await_ready(); // return if it should suspend at all
	void await_suspend(coroutine_handle<>); // what should it do each suspend
	void await_resume(); // what should it do each resume
};
```
All 3 functions with exactly those signatures are needed

```cpp
template<typename T> class coro_return_type
{
public:
	struct promise_type
	{
		using val_type = typename T;
		T value;

		coro_return_type get_return_object()
		{
			return coro_return_type(
                coroutine_handle<promise_type>::from_promise(*this)
            );
		}

		coro_awaitable_type initial_suspend() { return{}; }
		coro_awaitable_type final_suspend() { return{}; }

		// needed if co_yield is used in a coroutine
        // with coro_return_type as return type
		void yield_value(T val) { value = val; }

		// #A
		void return_value(T val) { value = val; }

		// #B
        // void return_void() {}
	};
```
Choose one:
* #A if `co_return` is used in a coroutine with `coro_return_type` as return type or
* #B if you have no intention of returning a value out of a coroutine
```cpp
	// optional default construction
	coro_return_type() : _coroutine(nullptr) { }

	// no copy only move
	coro_return_type(const coro_return_type&) = delete;
	coro_return_type(coro_return_type&& other) :
		_coroutine(other._coroutine)
	{
		other._coroutine = nullptr;
	}

	coro_return_type& operator= (const coro_return_type&) = delete;
	coro_return_type& operator= (coro_return_type&& other)
	{
		if (this != std::addressof(other))
		{
			_coroutine = other._coroutine;
			other._coroutine = nullptr;
		}
		return *this;
	}

	// optional utility
	void resume() { _coroutine.resume(); }

	T value() { return _coroutine.promise().value; }

	void wait()
	{
		while (!_coroutine.done())
			_coroutine.resume();
	}
```


`coroutine_handle<promise_type>.done()` indicates whether
the coroutine is finished.  
However coroutines where `bool final_suspend()::await_ready()`
returns `true` will cause calling `_coroutine.done()` after it is finished,
to be **UB**.  
The final_suspend is intended to extend the coroutines lifetime, so that
information such as its completion state or a possible value are preserved.

```cpp
bool done() const { return _coroutine.done(); }

private:
	explicit coro_return_type(coroutine_handle<promise_type> coroutine) :
		_coroutine(coroutine)
	{ }

	coroutine_handle<promise_type> _coroutine;
};
```
#### Conclusion
Coroutines are great! Coroutines are horrible! On the on side they can
simplify asynchronous code greatly. On the other side its fairly easy to
forget some of the implications, and end up with cryptic compiler errors or
run time UB.
