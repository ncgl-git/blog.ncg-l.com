---
title: "Bad Abstractions Part 1: Freight Trains vs Motorbikes"
date: 2023-11-12T11:42:52-06:00
draft: false
categories: [thinking out loud]
---

When used correctly, abstractions can make working within a codebase efficient and pleasant. They surface only the considerations that matter, while handling the shared logic and relationships internally. They are additionally straight-forward to test. But if you abstract too far, there will undoubtedly come a day where you must sacrifice the integrity of the abstraction for the sake of familiarity and standardization. It's important to understand where the abstractions should exist, because as I'll show, the tradeoffs can be drastic. 

### Higher-level API Clients

I once worked in a codebase that used this function everywhere:

```python

def call(**kwargs):

	resp = requests.request(
		url=kwargs['url'],
		method=kwargs['method'],
		json=kwargs['payload'],
		headers=kwargs.get('headers', {}),
		...
	)

	try:
		resp.raise_for_status()

	except Exception as error:
		# ... some convoluted retry logic

		return {
			"error": resp.text,
			"status": resp.status_code,
			...
		}
```

Everywhere an API call was made, our code opted for this function. At first glance we relate to the developer - why reuse `requests.request` in my API calls, when I could abstract it? Furthermore, this even plans ahead for the (unlikely) chance that we change REST libraries entirely. They are reducing code, and by standardizing on this function they easily allow future changes to be applied everywhere

 However, on further examination we see why this function falls apart and created many, many issues for our team. 

1. You don't control the API.
	- Some API's rate limit by the second, others by the hour. Some want exponential backoff on their retries, while others ask for an additional API call when they return a 307, and even others still use a response header. Even within the _endpoint_ you may see varying behaviour.

This abstraction has been tasked with handling _all_ of these considerations. When we encountered a new API or endpoint with unhandled behaviour, we had to update this function, which risked all other implementations. 

2. Shared benefits, shared costs.
	- The most damaging choice this function made, by far, was to return errors as dictionaries. And that is a reality of abstractions - you are stuck with the good decisions _and_ the bad. 

It is likely your abstractions will live in a shared repository. This further introduces a deploy and upversion cycle to enact the changes. So on top of muddied responsibilities, and a massive blast radius, there is now a 10 minute deploy cycle to get these changes into your repositories. 

Let's weigh these tradeoffs against the earlier-listed benefits - standardization and reusability:

1. Asking function to handle all endpoint or API-specific behaviour
2. Additional deploy + upversion process to propagate changes
3. Risk all other implementations when modifying, which as 1. illustrates, will be often

This abstraction is too broad. I don't think it should exist. The better choice would have been to use bare `requests` for each endpoint, and let them define their own backoff and retry logic. This would isolate where the changes take place, and give the developers freedom to handle whichever API behaviour they encountered. 

So instead of:

```python

def call(**kwargs):
	
	resp = requests.request(
		url=kwargs['url'],
		method=kwargs['method'],
		json=kwargs['payload'],
		headers=kwargs.get('headers', {}),
		...
	)

	try:
		resp.raise_for_status()

	except Exception as error:

		if resp.status_code == 429 and 'my api' in url and method == 'GET':
			time.sleep(30)
			return call(**kwargs)

		if resp.status_code == 429 and 'my api' in url and method == 'POST':
			time.sleep(20)
			return call(**kwargs)

		# ... some convoluted retry logic for other APIs

		return {
			"error": resp.text,
			"status": resp.status_code,
			...
		}

...

class Api1:
	
	def __init__(self):
		self.headers = {...}

	def post_data(self):
		url = 'http://...'
		method = 'POST'

		# internal retry logic, and result is a truthy dictionary, regardless of request success
		return call(
			url=url,
			method=method,
			headers=self.headers
		)

	def get_data(self):
		url = 'http://...'
		method = 'GET'

		# internal retry logic, and result is a truthy dictionary, regardless of request success
		return call(
			url=url,
			method=method,
			headers=self.headers
		)

```

it would be:

```python

class Api1:

	def __init__(self):
		self.headers = {...}

	def post_data(self):
		url = 'http://...'
		method = 'POST'

		# free to implement endpoint-specific retry logic and return is a requests.Response object
		resp = requests.request(url=url, method=method)

		if resp.status_code == 429:
			time.sleep(20)
			return self.get_data()

		return resp.json()

	def get_data(self):
		url = "http://"
		method =' GET'

		# free to implement endpoint-specific retry logic and return is a requests.Response object
		resp = requests.request(url=url, method=method)

		if resp.status_code == 429:
			time.sleep(30)
			return self.get_data()

		return resp.json()

```

The latter is flatter, surfaces the right concerns to the developer, and uses a familiar abstraction that most Python developers immediately understand. Changes would impact only the method modified. It additionally avoids the deploy and upversion cycle that a shared abstraction introduces. This is non-trivial.

Abstractions can give you better testability, and under predictable constraints, breadth. You can implement a change across hundreds of implementations and dozens of repos with a single pull request. But, they do not give you nimbleness, and in unpredictable constraints, they slow you down. Their benefits can be hard-hitting, their costs are catastrophic. 
