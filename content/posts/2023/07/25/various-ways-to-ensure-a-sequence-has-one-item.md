---
title: "Various Ways to Ensure a Sequence Has One Item"
date: 2023-07-25T22:06:29-05:00
draft: true
categories: [thinking out loud]
---

Often in my job I receive a list that is supposed to contain one item. My code makes this assumption. The business logic makes this assumption. If there is _not_ one item then it indicates a big problem (two records have the same primary key!?) I assert this limitation in my code so we know of it, and fail on it, immediately. 

The common way to determine the length of an in-memory `typing.Sequence` in Python is `len(my_sequence)`. This is one of the first built-in functions you learn and it has received years of attention and optimization which makes it _much_ better than something you could roll yourself. But it has one main drawback: you can't `len(some_generator)`.

I'd like to go over some different patterns for catching this case in Python, and discuss their tradeoffs.

**Option 1: Our starting point**
<br>
This option reads the entire generator into a list, storing it in memory, and uses the already-discussed built-in function to check its length. 
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


**Option 2**: 
<br>
This approach is nice because it fails immediately when there are too many items. I.E. unpacking like this doesn't necessarily consume the entire generator. It will fail on the first item past 1.
```
def option_2(some_iterator):
	try:
		[singular_item] = some_iterator
	except Exception as error:
		if "too many" in error:
			raise ExpectedOneItem(f"{some_iterator} contained more than 1 element!")
		elif "not enough" in error:
			raise ExpectedOneItem(f"{some_iterator} did not contain a single element!")
		else:
			raise error
	else:
		return singular_item
```


**Option 3:**
<br>
Unlike option 2, this approach _does_ consume the entire generator. Depending on the use case, you may want this as it could provide more context in your error logs.
```
def option_2_5(some_iterator):
	try:
		singular_item, *everything_else = some_iterator
	except ValueError:
		raise ExpectedOneItem(f"{some_iterator} did not contain a single element!")
	else:
		if everything_else:
			raise ExpectedOneItem(f"{some_iterator} contained {len(everything_else) + 1} elements, expected 1!")
		else:
			return singular_item
```


**Option 4:**
<br>
```
def option_3(some_iterator):
	for i, item in enumerate(some_iterator):
		if i > 0:
			continue
		else:
			yield item
	else:
		raise ExpectedOneItem(f"{some_iterator} contained {stop + 1} items, expected 1!")
```

**Options 5:**
<br>
```
def option_4(some_iterator):
	yield next(some_iterator)

	try:
		next(some_iterator)
	except:
		pass
	else:
		i, _ for i, _ in enumerate(some_iterator)
		raise ExpectedOneItem(f"{some_iterator} contained {i + 1} additional items)
```
