Title: Fetch and Save First, Then Process.
Date: 2014-10-01 10:20
Category: Data Munging
Tags: scraping
Slug: when-scraping-fetch-and-save-first-then-process
Summary: A common pattern for scraping websites.

### TL;DR Version:

    ::Makefile
    # Makefile
    FETCH = data/raw/FETCHED
    PROCESS = data/clean/CLEANED

    all: $(FETCH) $(PROCESS)

    $(FETCH): scripts/fetch.py
        @mkdir -p data/raw
        python scripts/fetch.py
        @touch $(FETCH)

    $(PROCESS): $(FETCH) scripts/process.py
        @mkdir -p data/clean
        python scripts/process.py
        @touch $(PROCESS)

Then,

    ::Bash
    $ make
- - - 

### Verbose Version

On your first web scraping project, you are quickly exposed to ideas concerning politeness. You learn that you should cache aggressively; abide by [robots.txt](http://en.wikipedia.org/wiki/Robots_exclusion_standard); and, [crawl at a rate](http://en.wikipedia.org/wiki/Robots_exclusion_standard#Crawl-delay_directive) that does not overwhelm the target. Then, you learn to use a specific scraping tool. At this point, for most people, learning is over. Data munging is schlep work. Once you have something that works, it's natural to stop looking for improvements.

Here, I'd like to offer one improvement that makes life easier. It's nothing original, but I wanted to reify it as a **P**attern. (And, I needed a first blog post -- preferably one that didn't have the title, "Hello, World!") **When scraping a website, fetch and save the pages first; then, process them locally**. 

If your lucky, the site will publish a [sitemap](http://en.wikipedia.org/wiki/Sitemaps), providing you with machine readable means of identifying pages to fetch. If the website doesn't have a sitemap or it is incomplete, you will probably have to do *some* online processing to figure out which pages need to be collected. In this case, I suggest only doing enough processing to collect the interesting URLs. Don't try to extract anything else if it can be done later. Defer.

*Why is this useful?*

It can be a form of aggressive caching. Prior to fetching a page, you can consult your local file system (or, whatever you are using for data storage). If the file exists and it's not too old, you can skip it. This is an elaboration on the common method of simply consulting a transient run-time dictionary of already fetched pages. This is polite, and benefits the owner of the target site.

Additionally, it is unlikely that you coded your scraper perfectly the first time. This is especially true if the website uses varied portrayals, frustrating attempts at a single set of CSS selectors or XPath expressions. If you are performing a live scrape, you may have to refetch everything by the time you first notice your error. This is very impolite.

Finally -- and, for me, critically -- you may not know exactly what information you want to extract when writing the scraper. The fetched page may have many potential variables. You think you only need A and B, but in the future you might want C - Z. When faced with this problem, your first instinct may be to write the code to scrape everything. But, if you have a saved snapshot, you can defer writing an exhaustive extractor until if and when it is ever needed. You get to be intelligently lazy.



