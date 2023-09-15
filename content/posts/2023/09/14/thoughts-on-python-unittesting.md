---
title: "Thoughts on Python Unittesting"
date: 2023-09-14T11:40:43-05:00
draft: true
---

Unit tests need to balance 3 things:

- confidence
- immunity to arbitrary refactors
- readability

## Confidence
Are we using the right tools, the right way, to test the right things?

`unittest.mock.MagicMock` will let you access any attribute, any method, without complaining (assertion variants aside). This opens the floodgates for typos, and if your tests are bad, you can easily write a test that returns a false positive. Here's an example:

```python
    @mock.patch.object(my_api, "requests")
    def test_example(self, requests):
        self.MyAPI.get_from_api()
        self.assertTrue(requests.requests.call_coun)
```

A junior dev might write this test and think they're asserting the request function was called. This is what's actually happening:

1. the requests argument is populated with a `MagicMock`
2. there is a reasonable typo on the `MagicMock` attribute "call_count"
3. the `MagicMock` class allows for access of any attribute or method, so the incorrectly-spelled "call_coun" is returning itself as a `MagicMock` (thats how they work), an inherently truthy value
4. the junior dev is using a less-than-ideal `TestCase` method to assert that requests was invoked, so `self.assertTrue(requests.requests.call_coun)` evaluates to `True` regardless if it was called 0 ,1, 2... times.

#### how to improve (pt. 1) - use autospec

```python
    @mock.patch.object(my_api, "requests", autospec=True)
    def test_example(self, requests):
        self.MyAPI.get_from_api()
        self.assertTrue(requests.requests.call_coun)
```
This above example would immediately fail when trying to access the attribute that didn't exist, `.call_coun`. 

Every exception made in tests, which set or use attributes or methods not available in prod, is an exception that weakens the test. I cannot think of a realworld scenario when you would not want to use `autospec=True`.

### how to improve (pt. 2) - io is king

`.call_count` is a shallow assertion. It doesn't detect changes in payloads or function signatures. Hands down, `.assert_has_calls` is the better assertion. To be honest, nobody should care about call counts alone. 

`.assert_called_with` only asserts that a certain call was made, it doesn't detect how many invocations were made. Again, `.assert_has_calls` is the better assertion. 

My personal belief is that the only unittests that matter are IO tests: given a sanctioned (approved by SME's and PMs), assert a sanctioned (approved by SME's and PMs) output. For my job with very nuanced and particular JSON this is a hard requirement.

Additionally, by standardizing on this approach you conform your tests. There is no longer a smattering of `call_count`, `assert_called_with`, etc on your mocks. There is one thing the dev needs to be concerned with: given this input, we get this output, whether its JSON to a file object, or a specific request body, or a payload to a boto3 client - this is all that matters. 

Given these last 2 points, lets see what a better test would look like:

```python
    @mock.patch.object(my_api, "requests", autospec=True)
    def test_example(self, requests):
        self.MyAPI.get_from_api()

        requests.request.assert_has_calls(
        	calls=[
        		mock.call(method='POST', headers={}, payload={}),
        		mock.call(method='GET', headers={}, payload={})
        	]
        )
```

This will detect unintended changes to headers, payloads, urls, methods, call_counts, etc.

# Immune to Refactoring
One of the biggest complaints I hear about unittests (and have experienced myself), is that as a system scales out, or certain patterns are identified, existing unittests pressure against refactoring. It's actually an enormous burden on the developer.

Only testing IO allows for making these intermediary changes freely. There aren't tests that a random class is invoked, or an obscure method is called. IO tests guarantee the core functionality works (and because of `.assert_has_calls`, works exactly like we expect). I'm talking things like db writes with `boto3` or http calls with `requests` - modules unlikely to change from a refactor. As long these go unmodified, you can moves things around. (*The exception may be the module locations of your mocks*)

## Test public first
From my experience, private methods only serve to logically group the behaviour of your code. They do not represent a piece of functionality. To this extent they should not be your priority, because the underlying logic may change, or another dev may visualize the process differently. Let me contrive an example:


```python
# my_module/important_logic.py

__all__ = ["normalize_name"]

def _clean(name: str) -> str:
    return name.strip()

def _format(name: str) -> str:
    return name.upper()

def normalize_name(name: str) -> str:
    name = _clean(name)
    name = _format(name)
    return name
```

From this example it is clear the developer broke the steps of normalization into their own grouping. No one is intended to call `_clean` or `_format`. If we only test (and to be sure, I am saying *vigorously* test), `normalize_name` then we allow for future devs to restructure this code without breaking the core functionality. All that matters is a sanctioned input and a sanctioned output to the public function.

If you scale this out to classes that perhaps contain only one public method, but many private ones, you can see what you get from this approach:

1. freedom to format the internal workings of functions and classes
2. freedom to move methods over to base classes
3. test trust: the test that matters is the test that breaks, as opposed to multiple tests breaking that all assert the same behaviour and would all demand the refactorers examination
4. an incentive to collate *sanctioned* test data
   1. it is _so_ easy to bloat a test directory with garbage test data that is forgotten after the PR

> ### *So, understanding that the IO & the public-facing functionality is what matters, we realize that what we're talking about is testing _interfaces_.*

This is very important to distinguish because, if you weren't thinking about how interfaces exist in your codebase, you are now.

# Readability

When a refactor breaks a test this is usually how it goes:
1. Dev makes an improvement to the codebase by doing something
2. 1 or more unittests break, some with blames from over 1 year ago. The originator is probably long gone.
3. Dev is immediately yanked into context they do not grasp: 
   1. What service is this testing? Who uses it?
   2. What were the requirements for this service?
   3. Who was the PM on this service?
   4. How valid is the data its using?
4. The dev is forced to find an SME for this code and bring themselves up to speed on what could otherwise have been a painless refactor, OR they choose to either modify the test **(which is guaranteed a lossy action - test quality in these cases does not improve)**, or they just delete the test.

Leveraging basic programming features here, I feel, can help.

1. If theres _any_ additional context to your test, add a docstring. Who you talked to, the team requesting the service etc.
2. Any non-obvious assertion, add a comment.

These two things will help guide the developer into the realm of consideration that was present when the code was initially written. It's not perfect, but it's better than nothing.

Leaving comments helps to communicate what you want to test. Consider this example:

```python
    @mock.patch.object(my_api, "boto3", autospec=True)
    def test_database_call(self, mock_boto3):
        self.MyClass.invoke()

        mock_boto3.assert_called_with(
            calls=[
                mock.call(data={'my-data': 1})
            ]
           )

        # assert we write to the db
        self.assertEqual(1, True)
```

Well hell! This developer said they were testing a database write! That last line isn't doing anything!

If this comment wasn't here, you'd be sweating at the thought of deleting this assertion - what boulders would that dislodge, to barrel into your team months later? The comment gives you the courage theres no footgun here. It's a line of code that was missed in a PR review.
