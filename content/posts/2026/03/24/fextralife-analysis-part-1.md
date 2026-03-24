---
title: "Fextralife Analysis Part 1 - Writing a Crawler"
date: 2026-03-24T09:38:45-04:00
draft: false
---

I've had this idea in my head a while, of how cool it would be to visualize the growth and evolution of Elden Ring knowledge, as determined by the content in the popular Fextralife wiki. Theres something optimistic about gamers coming together to discover, document and discuss really elaborate or hidden content in videogames, especially one as massive as Elden Ring. Oddly, it gives me hope that we would do the same for any other problem of science or engineering.

Anyways, I knew I needed to write a crawler for this.

My first predisposition is that this code would have some plain flow control. Something broadly like:

```python
saved = {}

def crawl(url: str):
	resp = requests.get(url)

	saved[url] = resp.text

	child_urls = get_child_urls(resp.text)

	for child_url in child_urls:
		if child_url not in SEEN_URLS:
			SEEN_URLS.add(child_url)
			crawl(child_url)
```
I knew I wanted a function like this to exist, that would be easy to understand the general looping functionality.

Eventually, after many iterations, I settled on this hierarchy: Crawler -> Page -> Parser.

The crawler orchestrates method calls on the Page in the easy-to-understand flow control above.
The page is responsible for the various network calls - current site, revisions, each revision id (using the backend classes to hit caches when possible) and encapsulates the children urls. The parser is responsible for returning certain data structures off of the html. I think it settled nicely.

So the function above ended up looking like this:

```python

    async def _crawl_url(self, async_client: httpx.AsyncClient, url: str) -> None:
        logger_ = logging.LoggerAdapter(logging.getLogger(), {"url": url})

        current = FextraPage(
            async_client=async_client,
            base_url=self.base_url,
            url=url,
            user_exclusion_filter=self.user_exclusion_filter,
            delay_min=self.delay_min,
            delay_max=self.delay_max,
            cache_backend_url=self.cache_backend_url,
            cache_backend_revision=self.cache_backend_revision,
        )

        try:
            await current.get_current()
        except Exception:
            logger_.warning("Failed retrieving current. Continuing on.")
            return
        else:
            if self.include_revisions:
                if await current.page_uuid is None:
                    logger_.warning("Page ID not found, unable to retrieve revisions. Continuing on.")
                    return
                await current.get_revisions()

            for filtered_child_url in await current.child_urls:
                if filtered_child_url not in self.seen_urls:
                    self.seen_urls.add(filtered_child_url)
                    logger_.debug(f"Adding url '{filtered_child_url}' to queue...")
                    await self.queue.put(filtered_child_url)

            logger_.info(f"Completed.")
```
Still perfectly readable in my mind.

I knew I wanted the user to bring their own backend. I mainly work with AWS, so I started with DynamoDB. But I can realistically see that someone may want Redis or Elasticsearch instead. Or you could go straight into a relational database to save yourself the export that I'm going to have to do later for analysis.

You can see above that these backend classes are passed into the page, where they are delegated to. More about that in a moment.

### Some dead ends on this project:

- Early versions of this used class methods and class attributes frequently. I.E. it feels slightly wrong to be passing delay_min and delay_max into every instance of FextraPage, instead of just one `FextraPage.set_delay(delay_min, delay_max)`. But if this library ever does see use, the user probably would expect to be able to run more than 1 crawler with varying delays per API. Seperately, there was a period of time, where these values were actually passed into the cache backends, since they only invoke during cache misses...which the cache backends know about. But then its up to every implementer of those classes to implement the delay...Lastly, I could say the same for the backend caches. Feels wrong to pass them in each init. 

- Early versions used a direct `DdbCacheUrl.memoize` decator on the FextraPage instance. Allowing the user to provide their backend meant I needed to decorate at runtime. I think that memoize function is probably the weirdest thing in this repo.

- Early versions of this used playwright. Once I worked out the exact call patterns I realized user auth was not needed and could remove a lot of the browser and content setup methods. 

- Early versions of this used a config singleton that other classes pulled from during instantiation. This was an attempt at avoiding passing in values to each instance that I thought should either be built in (like delay_min and delay_max), or more global. One example: 
Instead of the parser having to know about the base_url or exclusion patterns (which is passed in at the crawler) we pass them into a method instead. Now the parser __init__ can be pretty simple, and we have nice handoffs like this below to apply the custom logic.
```python

        async def get_current(self) -> None:
            self._fp_current = await self._memoized_fetch(self, self._url)
            filtered_children = await self._fp_current.filter_child_urls(self._base_url, self._user_exclusion_filter)
            await self._record_child_urls(filtered_children)

```
### Some things I didn't know (and Gemini taught me)

- I didn't know about producer/consumer models for async pipelines.
- I didn't know the name for the concept of coarse grain parallelism.
- I didn't know that Brotli was intended for html, and how much better it was for this task than say gzip.
- I didn't know you could scan a DDB table in parallel.

Gemini was very useful (and encouraging) to write this. I can't say I'd have stuck through it without it. But there was one bug it couldn't see:

```python
    
class DdbRevisionCache:
    async def abstract_scan(self, transform_lambda: Callable[[dict, dict], None], total_segments: int = 10) -> Any:
        local_cache = {}
        map_lock = asyncio.Lock()

        table = await self.get_table()

        async def scan_segment(segment_id: int):
            logger.info(f"{self.__class__.__name__}: Scanning with segment id {segment_id}.")
            exclusive_start_key = None

            while True:
                scan_kwargs = {
                    "TotalSegments": total_segments,
                    "Segment": segment_id,
                }
                if exclusive_start_key:
                    scan_kwargs["ExclusiveStartKey"] = exclusive_start_key

                response = await table.scan(**scan_kwargs)

                batch_dict = transform_lambda(response)

                async with map_lock:
                    local_cache.update(batch_dict)

                exclusive_start_key = response.get("LastEvaluatedKey")
                if not exclusive_start_key:
                    break

        tasks = [scan_segment(i) for i in range(total_segments)]

        await asyncio.gather(*tasks)

        return local_cache

    async def download_cache(self, total_segments: int = 10) -> CachedUrls_T:

        def transform_lambda(response: Dict) -> Dict:
        	batch_dict = default_dict(dict)
            for item in response.get("Items", []):
                batch_dict[item["url"]][item["revision_id"]] = item["children_urls"]
            return batch_dict

        return await self.abstract_scan(transform_lambda, total_segments)
```
This is the code for the DdbRevisionCache to pull the records into local memory. This structure is slightly different: its `{url: {revision_id: [child_urls]}}`.

The bug here is that the local_dict.update(batch_dict) doesnt append the new url revisions to the existing ones, it just updates the key entirely.

I had to track this down when I started to notice that I kept pulling revisions for urls I knew I had completed. Gemini could not find this bug after repeated prompting.

### Conclusion:

All-in-all I have no interest in outsourcing my thinking to LLMs. But the way this project worked out was very nice. My taste and experience dictated the interface of what was built. I worked with Gemini to tackle the specifics and learned a bit along the way.

https://github.com/ncgl-git/fextra-crawler/blob/main/README.md

