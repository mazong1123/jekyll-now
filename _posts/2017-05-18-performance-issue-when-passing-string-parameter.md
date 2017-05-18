---
layout: post
title: Performance issue when passing string parameter
date: 2017-05-18 10:58
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

# Pass std::string("01") or "01" to a function?

What's the different of option 1 and option 2?

```cpp
map<string, string> m;

// Option 1
m.insert(std::make_pair(std::string("01"), std::string("NI")));

// Option 2
m.insert(std::make_pair("01", "NI"));

```

Let's write a microbenchmark:

```cpp
#include <string>
#include <map>
#include <climits>
#include <ctime>
#include <iostream>

using std::string;
using std::map;
using std::cout;
using std::endl;

int main()
{
        map<string, string> m;

        size_t count = 10000000;

        const clock_t beginTime = clock();
        for (size_t i = 0; i <= count; ++i)
        {
                m.insert(std::make_pair(std::string("01"), std::string("NI")));
                //m.insert(std::make_pair("01", "NI"));
        }

        cout << float(clock() - beginTime) / CLOCKS_PER_SEC << endl;

        return 0;
}
```

Run the program we got following result (g++ 5.4.0 without specifying -O):

std::string("01"): 1.8 sec
"01": 1.34 sec

So the std::string version is 40% slower than passing string directly.

If we add -O3, the differences are more significant:

std::string("01"): 0.34 sec
"01": 0.14 sec

# What happened?

Let's run g++ with `-fdump-tree-optimized`. We found std::string("01") version has following two additional lines:

```cpp
 std::allocator<char>::allocator (&D.24921); <-- Bad ass!
 std::__cxx11::basic_string<char>::basic_string (&D.24922, "NI", &D.24921); <-- Bad ass!
```

std::string("01"):

```cpp
<bb 2>:
  std::map<std::__cxx11::basic_string<char>, std::__cxx11::basic_string<char> >::map (&m);
  count_11 = 10000000;
  beginTime_13 = clock ();
  i_14 = 0;

  <bb 3>:
  # i_1 = PHI <i_14(2), i_34(13)>
  if (i_1 > count_11)
    goto <bb 14>;
  else
    goto <bb 4>;

  <bb 4>:
  std::allocator<char>::allocator (&D.24921); <-- Bad ass!
  std::__cxx11::basic_string<char>::basic_string (&D.24922, "NI", &D.24921); <-- Bad ass!

  <bb 5>:
  std::allocator<char>::allocator (&D.24918);
  std::__cxx11::basic_string<char>::basic_string (&D.24919, "01", &D.24918);

  <bb 6>:
  D.24980 = std::make_pair<std::__cxx11::basic_string<char>, std::__cxx11::basic_string<char> > (&D.24919, &D.24922); [return slot optimization]

  <bb 7>:
  std::pair<const std::__cxx11::basic_string<char>, std::__cxx11::basic_string<char> >::pair<std::__cxx11::basic_string<char>, std::__cxx11::basic_string<char> > (&D.25068, &D.24980);
```

"01":

```cpp
std::map<std::__cxx11::basic_string<char>, std::__cxx11::basic_string<char> >::map (&m);
  count_8 = 10000000;
  beginTime_10 = clock ();
  i_11 = 0;

  <bb 3>:
  # i_1 = PHI <i_11(2), i_18(8)>
  if (i_1 > count_8)
    goto <bb 9>;
  else
    goto <bb 4>;

  <bb 4>:
  D.24974 = std::make_pair<const char*, const char*> ("01", "NI");

  <bb 5>:
  std::pair<const std::__cxx11::basic_string<char>, std::__cxx11::basic_string<char> >::pair<const char*, const char*> (&D.25046, &D.24974);

  <bb 6>:
  D.25162 = std::map<std::__cxx11::basic_string<char>, std::__cxx11::basic_string<char> >::insert (&m, &D.25046);

  <bb 7>:
  std::pair<const std::__cxx11::basic_string<char>, std::__cxx11::basic_string<char> >::~pair (&D.25046);

```

#Conclusion

Don't use `m.insert(std::make_pair(std::string("01"), std::string("NI")));`. Use `m.insert(std::make_pair("01", "NI"));` instead.
