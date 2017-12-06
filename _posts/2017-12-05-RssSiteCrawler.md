---
layout: post
section-type: post
title: RSS Site Crawler
category: linux
tags: [ 'RSS' ]
---

After my [unsuccessful attempts](http://www.iainbenson.com/linux/2017/11/26/RSS-Tools.html) to use existing RSS 
tools to create feeds from sites that don't have them, I decided to write my own based around [scrapy](https://scrapy.org).  It's 
called [SiteCrawler](https://github.com/0x3F3F/RssTools/tree/master/SiteCrawler).

## Setup Scrapy and Creating My First Spider

After installing scrapy using pip, I first ran a command to set-up a project, which created a directory 
structure and various files.

```bash
scrapy startproject SiteCrawler
```

Next I decided that I would attempt to develop my first spider.  I copied the html from a site
over into a web directory of the Pi using `curl`, so as not to get banned while I was developing the spider.

The first thing I did was to look at the html to see which tags I was interested in.  The site 
in question output reports every quarter, and I want a feed of these as they appear.  I used the 
`scrapy shell` to refine the tags I wanted:

```bash
scrapy shell http://127.0.0.1/test.html
>response.css('li.views-row a::attr(title)')
```

This looked for the title tags of &lt;a&gt; nested within &lt;li class="views-row"&gt; tags.  One tip here was 
that after I changed the pipeline/extractor (below) the shell stopped working -  Changinig scrapy.cfg attribute from 
`SiteCrawler.settings` to `scrapy.settings` reset to the default config and restored shell operation.  

Following a bit of investigation, I then came up with the following spider:

```python
import scrapy
import datetime, time


class PersonalAssetsSpider(scrapy.Spider):
	name = "PersonalAssets"
	#allowed_domains = ['https://patplc.co.uk']
	start_urls = ['http://127.0.0.1/test.html', ]

	# Custom settings that I can read in the pipeline & put in feed.
	custom_settings = {}
	custom_settings['RSS_TITLE'] = 'Personal Assets Trust'
	custom_settings['RSS_LINK'] = start_urls[0]
	custom_settings['RSS_OUTPUT_FILE'] = 'PersonalAssets.rss'


	def parse(self, response):
		"""Select page elements to pick for the generated xml element"""

		# Best Methed:
		# Curl page onto localhost for dev
		# run 'scrapy shell http://127.0.0.1/test.html'
		# Can then experiment: response.css('li.views-row a::attr(title)') 
		# dot to specify class.  Space separated sub elements
		for viewRow in response.css('li.views-row'):

			# Note the guid is a unique descriptor, just repeat url
			# I've set the order here in FEED_EXPORT_FIELDS cfg variable
			title = viewRow.css('a::attr("title")').extract_first()		
			link = viewRow.css('a::attr("href")').extract_first()

			yield {
			'title': title,
			'link': link,
			'guid' : link,
			'description': "Personal Assets "+title,
			}
```

The spider could then be ran manually using:

```bash
scrapy runspider PersaonalAssets_spider.py -o test.xml
```

This worked and output xml to the supplied file.  I did get one warning stating that service_identity 
cannot import opentype.  An Internet search told me this was due to pyasn1 not being installed, but I  
was confused as it was.  After much headscratching I ran a `scrapy version -v` and noted that it was 
using php3, even though default on my system was php2.  I then ran `pip3 install service_identity` which worked!

## Creating A Valid RSS Feed

The feed created by the tool appeared to be missing **rss** and **channel** tags.  So, I tried to 
write new exporter, to add them in.  Here's what I came up with:

```python
from scrapy.exporters import XmlItemExporter 

# Create subclas of XmlItemExporter Override the default XmlItemExporter and update cfg FEED_EXPORTERS to use this instead
class RssXmlItemExporter(XmlItemExporter):

	# Apparently not essential to define, as takes from parent class
	# Doing so just in case I want to add more into to it
	def __init__(self, file, **kwargs):
		XmlItemExporter.__init__(self, file, **kwargs)

	def start_exporting_rss(self, rss_title, rss_link):
		"""
		Adds opening tags, including channel and rss
		Additionall adds a title and link to the feed.
		"""
		self.xg.startDocument()

		# IRB MYsuff
		self.xg.startElement("rss", {'version':'2.0'})
		self._beautify_newline(new_item=True)
		self.xg.startElement("channel", {})
		self._beautify_newline(new_item=True)
		self._export_xml_field('title',rss_title,1)
		self._export_xml_field('link',rss_link,1)
		# End Mysuff
		
		self.xg.startElement(self.root_element, {})
		self._beautify_newline(new_item=True)



	def finish_exporting_rss(self):
		"""Closes off my custom rss and channel tags"""
		self.xg.endElement(self.root_element)

		# IRB MYsuff
		self._beautify_newline(new_item=True)
		self.xg.endElement("channel")
		self._beautify_newline(new_item=True)
		self.xg.endElement("rss")
		# End Mysuff

		self.xg.endDocument()
```

and update the config (settings.py)  to use it:

```python
FEED_EXPORTERS = {'xml' : 'SiteCrawler.feedexport.RssXmlItemExporter'}
```

## Adding a Title tag

I decide that I'd like to add in a title tag for the whole feed, which turned out to be much harder than I thought.  The 
problem was that the parameters from the spider class were not available in the exporter.

After much reading, I decided the way to do this was for me to write a pipeline which would explicitly call 
the exporter methods.  I guess scrapy uses a default one if none present.  This provides a customisable way to 
process the data that I've scraped, giving me full control.

I fetched the name/url items from the spider and then passed them into the appropriate function call.

```python
from SiteCrawler.feedexport import RssXmlItemExporter 

class SitecrawlerPipeline(object):

	def __init__(self, settings):

		# Following items were set in spider using custom_settings dict.
		self.rss_title = settings.get("RSS_TITLE")
		self.rss_link = settings.get("RSS_LINK")
		self.rss_output_file = "output_feeds/" + settings.get("RSS_OUTPUT_FILE")

		# Settings for the exporter.  These come from global settings.py
		exporterSettings = 	{}
		exporterSettings['fields_to_export'] = settings.get("FEED_EXPORT_FIELDS")
		exporterSettings['indent'] = settings.get("FEED_EXPORT_INDENT")
		exporterSettings['encoding'] = 'utf-8'
		#exporterSettings['export_empty_fields'] = False 
		#exporterSettings['item_element'] = 
		#exporterSettings['root_element'] = 
		
		# Open the rss file for writing 
		self.file = open(self.rss_output_file,"wb")	# Overwrites existing

		# Now setup the exporter to be used.  
		# Can pass params separately, but I'm using dict.
		self.exporter = RssXmlItemExporter(self.file, **exporterSettings)

		# Useful to know how to tie signals to functions.  
		# open/close_spider automatically setup
		# dispatcher.connect(self.SpiderOpenFunc, signals.spider_opened)
		# dispatcher.connect(self.SpiderCloseFunc, signals.spider_closed)


	@classmethod
	def from_crawler(cls, crawler):
		"""
		This function changes how the crawler engine calls the pipeline 
		__init__ function.  The additional parameters are added.  Therefore 
		need to update init above to include extra param(s). This allows 
		access to global settings (settings.py) and custom setting from within 
		spider
		"""
		settings = crawler.settings
		return cls(settings)
	
	def open_spider(self, spider):
		"""Use custom functions I defined that add rss and channel tags"""
		self.exporter.start_exporting_rss(self.rss_title, self.rss_link);

	def close_spider(self, spider):
		"""USe custom function that closes the new rss and channel tags"""
		self.exporter.finish_exporting_rss();
		self.file.close()

	def process_item(self, item, spider):
		"""Unchanged to what was here before"""
		self.exporter.export_item(item)
		return item
```

and update the config to use it:

```python
ITEM_PIPELINES = {
    'SiteCrawler.pipelines.SitecrawlerPipeline': 300,
}
```

## Finished RSS Feed

Running this spider then gave the following xml, which I could point my feed reader to.  The plan being to run 
the scripts daily on a CRON. My next task is to create new spiders for each site that I check.

```xml
<?xml version="1.0" encoding="utf-8"?>
<rss version="2.0">
<channel>
    <title>Personal Assets Trust</title>
    <link>http://127.0.0.1/test.html</link>
<items>
    <item>
        <title>Quarterly Report No. 85</title>
        <link>https://patplc.co.uk/sites/default/files/documents/85.pdf</link>
        <guid>https://patplc.co.uk/sites/default/files/documents/85.pdf</guid>
        <pubDate>05 December 2017 19:51</pubDate>
        <description>Personal Assets Quarterly Report No. 85</description>
    </item>
    .....
    <item>
        <title>Quarterly Report No. 83</title>
        <link>https://patplc.co.uk/sites/default/files/documents/83.pdf</link>
        <guid>https://patplc.co.uk/sites/default/files/documents/83.pdf</guid>
        <pubDate>05 December 2017 19:51</pubDate>
        <description>Personal Assets Quarterly Report No. 83</description>
    </item>
</items>
</channel>
</rss>
```


Hopefully someone will find this useful, there weren't any end to end guides which was strange as I thought exporting to 
 RSS would be one of it's main uses.  I've checked it all into [github](https://github.com/0x3F3F/RssTools/tree/master/SiteCrawler).

I've created several spiders and each one only takes around 10 mins or so, so am very happy with the rapid setup.  My next task is to expand 
some of the spiders to operate on mutiple pages on the same site.  I'm also eying scraping a phpBB site, but that will involve it logging 
in to get a session ID......

I suspect I will return to using scrapy when I get round to looking into machine learning, so 
as to  use it to scrape learning data. Hopefully I'll be more of a scraping expert by then.

Useful articles [here](https://stackoverflow.com/questions/14075941/how-to-access-scrapy-settings-from-item-pipeline), 
[here](https://doc.scrapy.org/en/latest/topics/item-pipeline.html#write-items-to-mongodb), 
[here](http://www.scrapingauthority.com/2016/09/19/scrapy-exporting-json-and-csv/) and [here](https://github.com/nblock/feeds/blob/master/feeds/pipelines.py).





 





