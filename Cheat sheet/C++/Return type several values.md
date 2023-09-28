#### Structured binding (since C++17)
https://en.cppreference.com/w/cpp/language/structured_binding

```cpp
Tuple<T1, T2> SomeFuction()
{
	T1 FirstReturnValue;
	T2 SecondReturnValue;
	...
	return Tuple<T1, T2>{ FirstReturnValue, SecondReturnValue };
}

//auto / const auto / const auto& 
auto [First, Second]{ SomeFuction() };
```