---
title: Setting up SQLite as cache (and why it was a bad idea)
author: apoplexi24
date: 2025-03-21 10:00:00 +0800
categories: [Development, Database]
tags: [sqlite, caching, python, fastapi, optimization]
pin: false
math: true
mermaid: false
image:
  path: https://cdn.jsdelivr.net/gh/apoplexi24/blog-assets@main/sqlite-cache/img/sqlite-fragmentation-2.jpg
  alt: SQLite meme showing its limitations
---

Cache is every backend developer's wet dream. Having your variables loaded onto your RAM is very convenient indeed, but sometimes the RAM does become a bottleneck when you become over ambitious and try to cache everything. It's like that one time you tried to memorize the entire dictionary – noble, but ultimately leading to a headache and a newfound appreciation for Ctrl+F. I, too, fell victim to the siren song of "instantaneous data access," and decided that SQLite, the plucky little database that could, would be my caching savior.

Spoiler alert: it wasn't *all* sunshine and rainbows. In fact, it was more like a partly cloudy day with a high chance of existential dread and a few "why did I do this to myself?" contemplation. This note chronicles my journey into the heart of caching darkness, armed with nothing but good intentions, questionable assumptions, and a database engine that's probably still judging me.

<img src="https://cdn.jsdelivr.net/gh/apoplexi24/blog-assets@main/sqlite-cache/img/sqlite-gaussianbell.webp" alt="SQLite Meme" />
_When SQLite seems like the perfect solution... at first_

## The Beginning of the End

It all starts with non-technical stakeholders (it always does). Our accountants, who are well versed in Excel, wanted to open a .dbf file that comes from a vendor. 

> A .dbf file is a database file, commonly associated with the dBASE database management system. It's a file format that stores structured data in a table format, with rows and columns, similar to a spreadsheet or a database table.
{: .prompt-info }

Under normal circumstances, our good old reliable MS Excel can open the damned .dbf file. But then again, every good thing has a 'but' attached to it. The 'but' in MS Excel's case is the memory and computation required to open big files. Oh, and how can I forget, Excel cannot handle files with more than 1,048,576 rows[^1], making it near impossible to work with big data on Excel (who would have known that Excel is not a database **/s**).

So my manager aptly suggested we use a database connector to MS Access and we could pull the relevant data on MS Excel when needed. But greed gets to the best of us all, as the stakeholder wanted to load the entire file with all rows in memory for aggregation to see all of it at once in one place.

## The Project (In Brief, Hopefully)

I'll dive straight to the data aspect of the project and save the nuances for another post. 
This was the tech stack I used to deploy the endpoint:
* FastAPI for backend 
* Nginx for reverse proxy
* HTMX for frontend

### Reasons to Cache
* Improve loading time of files rather than reading it every time
* Storing the files uploaded on disk and loading the current file onto the cache
* Eliminate reuploading of files as it takes the most time in the process

One of the features the stakeholders wanted was fast load times. The issue would be loading the file onto a pandas dataframe each time the data is uploaded would take a lot of time. To overcome that, one could just use a global variable that would be loaded once per session and then copies could be made in each function so that the dataframe loaded as global variable would be untouched.

```python
##### rest of the code here ########
# yeah I like type hinting the obvious, bite me
currently_loaded_dataframe: pd.DataFrame = None
filtered_dataframe: pd.DataFrame = None

@app.get('/apply_filter_on_table')
async def post_endpoint(req: Request):
    global currently_loaded_dataframe, filtered_dataframe
    req_as_dict: dict = await req.json() 
    copy_df: pd.DataFrame()  = currently_loaded_dataframe.copy()
    ####### filter operations on copy_df based on req params #######
    filtered_df = copy_df
    # defining api response here
    return response
```

This seemed a perfect enough implementation for one user on one browser at a time. Good enough for POC as we don't want to do premature optimizations and over engineer it. If I wanted to scale it up I would use a FIFO Ordered Dictionary with fixed number of keys:

```python
from collections import OrderedDict

class CustomFifoCache(OrderedDict):
    def __init__(self, capacity):
        super().__init__()
        self._capacity = capacity

    def __setitem__(self, key, value):
        super().__setitem__(key, value)
        if len(self) > self._capacity:
            self.popitem(last=False)  # yeet the oldest item (FIFO)

loaded_dataframe_dict: CustomFifoCache = CustomFifoCache(capacity=3)

@app.post('/upload_file')
async def upload_large_file(file_id: str = Form(...),
                            file: UploadFile = File(None)):
    global loaded_dataframe_dict
    destination = os.path.join(BASE_FILES_DIR, file_id + '.dbf')
    async with aiofiles.open(destination, "wb") as out_file:
        while content := await file.read(1024 * 1024):  # Read in 1MB chunks.
            await out_file.write(content)
    try: 
        df = await load_dbf_to_dataframe(destination)
        loaded_dataframe_dict[file_id] = df     
    except Exception as e:
        print("error while loading dbf to df: ", e)
        
    url = f"/dashboard?file_id={file_id}"
    response = RedirectResponse(url=url, status_code=302)
    response.headers["HX-Redirect"] = url
    
    return response
```

## Gunicorn - The Winged Horse of Asynchronous Agony

Wrong. I forgot the very fact that the backend is managed by Gunicorn. 

<img src="https://cdn.jsdelivr.net/gh/apoplexi24/blog-assets@main/sqlite-cache/img/error2black.png" alt="Error Message" />
_The moment of realization_

> Gunicorn 'Green Unicorn' is a Python WSGI HTTP Server for UNIX. It's a pre-fork worker model. The Gunicorn server is broadly compatible with various web frameworks, simply implemented, light on server resources, and fairly speedy.
{: .prompt-info }

Gunicorn spun up the default count of 25 (ie 2 * $num_cores+1$) uvicorn workers to do its bidding. That means each request is getting routed to one of those 25 uvicorn workers based on request traffic, while they all have their own version of the global variable loaded. The file I was uploading was getting updated only in the uvicorn worker to which the request was routed and not propagated to the other workers.

<img src="https://cdn.jsdelivr.net/gh/apoplexi24/blog-assets@main/sqlite-cache/img/gunicorn-flowchart.png" alt="Gunicorn Workers" />
_Multiple workers, multiple problems_

## SQLite - A Batman or Bane?

SQLite prides itself on being faster than a filesystem by a factor of 35%[^2], which means it **should** be blazing fast. But wait, what about the space-time trade-off in data storage (or any algorithm in the world). We saved some time so there must be a space tradeoff somewhere which we cannot afford, the runtime by itself is consuming RAM like there is no tomorrow.

## Major Issues with using SQLite as Cache

A good SQLite database architecture makes use of indices for speed in querying. If the filter is applied on a particular column that is not indexed, the entire table is loaded onto the memory rather than using an offset[^3]. This means anytime the filter is not on the columns I intend the users to query on, then the ETA converts from "3 minutes" to "1000 hours" like I am downloading a badly seeded file using P2P torrent.

<img src="https://cdn.jsdelivr.net/gh/apoplexi24/blog-assets@main/sqlite-cache/img/sqlite-fragmentation.jpg" alt="Download Time" />
_Actual footage of what goes on in data science teams_

Let's delve into the real reasons why SQLite as a cache turned my coding utopia into a debugging dystopia:

1. **The Indexing Illusion:** Each unindexed filter query became a full table scan, turning my "blazing fast" cache into a digital molasses swamp. It was like inviting Usain Bolt to a race and then making him run through quicksand.

2. **Memory Hogging (Again):** SQLite decided to helpfully load entire tables into memory during those unindexed queries. My server started sweating more than I do during a production deployment.

3. **Concurrency Conundrums:** SQLite is surprisingly good with concurrent _reads_. But throw in a few _writes_, and things get dicey.

4. **The "It's a Feature, Not a Bug" Fallacy:** The persistence of SQLite gave me a false sense of security, and made me less inclined to implement proper cache invalidation.

5. **Overhead of SQL operations:** Even with the indices, there is an overhead of converting the dataframe to SQL operations.

## Conclusion 

> Maybe a "perfect caching system" was all the friends we made along the way.
{: .prompt-tip }

Even though the process was harrowing, I managed to make it work. 

<img src="https://cdn.jsdelivr.net/gh/apoplexi24/blog-assets@main/sqlite-cache/img/prod-abomination.png" alt="Prod Abomination" />

_Abomination of a code if I must_

> Ugly Working Software >>> Fancy Software that doesn't work
{: .prompt-warning }

The management was pleased by this maneuver. I had reduced a poor person's daily work of 6 hours of splitting the file for operations on it on a slow windows server. Everyone's happy, except the FastAPI service running on the VM with the mammoth responsibility of handling huge data.

Ultimately there's nothing wrong in using a file, global variable, Redis or SQLite as cache, it all depends on the use case, hardware restrictions, application architecture and most importantly, the mental sanity of the developer. A developer should ideally choose the most simplest approach and if it works, it works. Optimize only when necessary or else you will never finish the project, ever.

## References

[^1]: [MS Excel Specifications and Limits](https://support.microsoft.com/en-us/office/excel-specifications-and-limits-1672b34d-7043-467e-8e27-269d656771c3)
[^2]: [SQLite Faster than Filesystem](https://www.sqlite.org/fasterthanfs.html)
[^3]: [Why You Shouldn't Use SQLite](https://www.hendrik-erz.de/post/why-you-shouldnt-use-sqlite)
