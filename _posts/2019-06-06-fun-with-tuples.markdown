---
layout: post
title:  "All combinations of types in a tuple in C++"
date:   2019-06-06 15:00 +0100
categories: jekyll update
---

## What?
We have a tuple of types
```cpp
#include <tuple>

template<typename ...  Ts>
using t=std::tuple<Ts...>;

struct a{};
struct b{};
struct c{};

using my_tuple = t<a,b,c>;
```
and we would like to get all possible type combinations of length n, taken from this tuple.  
That corresponds the nth power of cartesian products of the tuple.
So, my result should look like this:

```cpp
combinations<my_tuple, 2> // returns t<t<t<a,a>, t<a,b>, t<a,c>>,
                          //           t<t<b,a>, t<b,b>, t<b,c>>,
                          //           t<t<c,a>, t<c,b>, t<c,c>>>
```

## Combinatorics in python
I like to prototype algorithms in python first and then translate.. Less fiddling with details  
One possible solution to do combinatorics looks like that:
```python
def combinations(arr, n, res=[]): 
    if n == 0: 
        return res 
    return [combinations(arr, n-1, res+[i]) for i in arr]
```
It will recursively call combinations, "keeping track" of the current indices by appending them to the result and then returning when we have reached the desired dimension.

## Now with C++ types

### Check all the types
There is a neat trick for checking which type you are currently fiddling with.  
Declare some type that holds your type of interest and do not define it,  
then gcc and clang give you a nice error if you try to instantiate one of these bad boys,  
displaying your type nicely:
```cpp
template<typename ... Ts>
struct type_printer;

int main () {
  type_printer<my_tuple>{};
}
```
in gcc-9.1 gives
```
<source>: In function 'int main()':
<source>:47:37: error: invalid use of incomplete type 'struct type_printer<std::tuple<a, b, c> >'
   47 |     type_printer<std::tuple<a,b,c>>{};
      |                                     ^
<source>:5:8: note: declaration of 'struct type_printer<std::tuple<a, b, c> >'
    5 | struct type_printer;
      |        ^~~~~~~~~~~~
```

### Recurse in the type system
Recursing works fairly straight forward in the C++ type system.  
You can see that in many parts of the STL and everywhere on StackOverflow.  
Remember, we need something that refers to itself and some stopping condition.  
A small example of recursion is something along the lines of `std::make_index_sequence`:
```cpp

template<std::size_t ... Is>
struct index_sequence{};

//result... carries the ascending pack of integers
template<std::size_t n, std::size_t ... result>
struct make_index_sequence {
    //every time we iterate, we append n-1 to the result.
    using type = typename make_index_sequence<n-1, n-1, result...>::type;

};

//stopping condition: we will not continue if we reached 0
template<std::size_t ... result>
struct make_index_sequence<0, result...> {
    using type = index_sequence<result...>;
};
```

### Some helpers
To concatenate and append to tuples types, we use these little helpers, making use of `std::tuple_cat` to determine the type:
```cpp
template <typename... tups>
using tuple_cat_t = decltype(std::tuple_cat(std::declval<tups>()...));

template <typename tup, typename item>
using append = tuple_cat_t<tup, std::tuple<item>>;
```

We will also need to "iterate over tuples", which is normally done via index sequences, therefore we define an index sequence with the length of a tuple:
```cpp
template <typename tup>
using index_sequence_for_tuple =
    std::make_index_sequence<std::tuple_size_v<tup>>;
```

### Element-wise tuple transformations
Now, we need a helper to execute one operation on each entry of a tuple and "return" a transformed tuple, very similar to `boost::hana::transform`
```cpp
template <typename tup,
          template <typename> typename op,
          std::size_t... Is>
auto operate_t_impl(std::index_sequence<Is...>)
    -> std::tuple<op<std::tuple_element_t<Is, tup>>...>;

template <typename tup, template <typename> typename op>
using operate_t = decltype(
    operate_t_impl<tup, op>(std::declval<index_sequence_for_tuple<tup>>()));
```

The usual way get a parameter pack of the types from a tuple is
```cpp
std::tuple_element_t<Is>(my_tup)...
```
`Is` is a parameter pack of the indices you want to gather the types from, so in our case all of them `0,1,2,3,4...`.  
That is the reason for the existence of the helper `index_sequence_for_tuple`.  
Since the `index_sequence` is no parameter pack as we need it for the tuple iteration,  
we use a common trick involving function template argument deduction in `operate_t_impl`.  
To get the `std::size_t ... Is` from our `index_sequence`, we give (a `std::declval` of) the sequence as function argument 
and let the argument deduction deduce `std::size_t ... Is` for us.

Ok, so now we can invoke "unary operations" (type-transformations) with a signature `template <typename> typename op` on all elements of the tuple,
and "return" a result tuple. So to say, we just implemented poor mans `hana::transform`.

### Bring stuff together
Now we can perform elementwise transformations on a tuple and recurse, lets bring it together to perform our task:
```cpp
template <typename tup, std::size_t n, typename result = std::tuple<>>
struct combinations {
  //this "operation" is conceptually similar to a unary lambda given in std::transform
  //its python equivalent: combinations(arr, n-1, res+[i])
  template <typename item>
  using operation = typename ::combs<tup, n - 1, append<result, item>>::type;

  //this "loops" over the tuple, each time invoking operation, which takes care of the recursion
  //its python equivalent: [operation for i in arr]
  using type = operate_t<tup, operation>;
};
```
and the partial template specialization corresponding to the stopping condition:
```cpp
//its python equivalent: 
//    if n == 0: 
//        return res 
template <typename tup, typename result>
struct combinations<tup, 0, result> { 
  using type = result;
};
```
Thats it, much less code that i would have expected when starting this exercise.. :D
