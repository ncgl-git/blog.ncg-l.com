---
title: "Various Ways to Ensure a Sequence Has One Item"
date: 2023-08-03T08:06:29-05:00
draft: false
categories: [thinking out loud]
---

Often in my job, I receive a list that is supposed to contain one item. My code makes this assumption, the business logic makes this assumption, if there is _not_ one item then it indicates a big problem. I assert this limitation in my code so we know of it, and fail on it, immediately. 

### When do I find this useful?

- asserting a query has only retrieved one item
- only run logic if multiple functions agree/return the same value, i.e.
  ```
  try:
    [agreement] = set(choice1(), choice2(), choice3())
  except ValueError:
    ...
  ```
- simply unpack things that are lists of one item (you don't always make the data model)
  - for me, `my_list[0]` is reserved for explicitly grabbing the first item of a list, and I only use it when I've sorted my list by some criteria. `my_list[0]` does _not_ serve to indicate the list contains one item. 

### Options

The common way to determine the length of an in-memory `typing.Sequence` in Python is `len(my_sequence)`. This is one of the first built-in functions you learn and it has received years of attention and optimization which makes it _much_ better than something you could roll yourself. But it has one main drawback: you can't `len(some_generator)`.

I'd like to go over some different patterns for catching this case in Python and discuss their tradeoffs.

**Option 1: Our starting point**

This option reads the entire generator into a list (storing it in memory) and then uses the already discussed built-in function to check its length. _I do not like this option._ 

```
def option_1(some_iterator):

    as_list = list(some_iterator)

    len_ = len(as_list)

    if len_ == 0:
        raise ExpectedOneItem(f"{some_iterator} contained 0 elements, expected 1!")
    elif len_ > 1:
        raise ExpectedOneItem(f"{some_iterator} contained {len_} elements, expected 1!")
    else:
        return as_list[0]
```


**Option 2a**:

This approach is my preference, as it allows you to both unpack the iterable and name the internal variable in one line. If the provided argument is a generator, it will only exhaust 2 iterables before raising the error - i.e. it doesn't dump the entire generator into memory and then raise the error.

```
def option_2a(some_iterator):
    try:
        [singular_item] = some_iterator
    except ValueError as error:
        if "too many" in str(error):
            raise ExpectedOneItem(f"{some_iterator} contained more than 1 element!")
        elif "not enough" in str(error):
            raise ExpectedOneItem(f"{some_iterator} did not contain a single element!")
        else:
            raise error
    else:
        return singular_item
```

**Option 2b:**

This approach is slightly different in that, if provided a generator, it will exhaust all iterations, and assign them to the _ variable. In some cases, this could be better as it allows you to provide more context in the error message about what was additionally returned.
```
def option_2b(some_iterator):
    try:
        singular_item, *_ = some_iterator
    except ValueError:
        raise ExpectedOneItem(f"{some_iterator} did not contain a single element!")
    else:
        if _:
            raise ExpectedOneItem(f"{some_iterator} contained {len(_) + 1} elements, expected 1!")
        else:
            return singular_item
```

---

I find that Option 2a serves most of my use-cases. It's safer and more pleasing to read than `my_list[0]`, and additionally more memory efficient than `list(my_generator)[0]`. Give it a try.