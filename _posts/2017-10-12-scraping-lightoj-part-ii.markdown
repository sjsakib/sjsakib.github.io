---
title:  "Scraping Light OJ: Part II"
date:   2017-10-11 22:34:49
categories: [hacking]
tags: [scraping]

excerpt_separator: <!--more-->

showprev: yes
---

We have to write a `pipeline` to process our data. Previously we just yielded all our data to scrapy and scrapy saved it in a json file.
A pipeline is already created by scrapy when we started our project. We need to add this line to settings file to activate that.
```python
ITEM_PIPELINES = {
    'lightoj.pipelines.LightojPipeline': 300,
}
```

Now let's open the pipelines file and add our stuffs in that. First we have to implement a method `process_item`. Whenever our spider yields a item, the item will pass to this method so we can process it as we like. If it's a submission information we'll append it to the `data.json` file. And if it is a tag information we'll add it to a list property of our pipeline to use later when saving files. We'll also add a `open_spider` method to load and save tags informations when our spider starts.

```python
def open_spider(self, spider):
    self.tags = {}

def process_item(self, item, spider):
    if 'tags' in item:  # it's tag information
        self.tags[item['pid']] = item['tags']
    else:
        data = json.dumps(item)+'\n'
        with open('data.json', 'a') as f:
            f.write(data)
    return item
```

Now the last part. We'll implement a method `close_spider` in our pipeline. This method will be called when our spider closes. We'll save our codes here. Here's the code
```python
def close_spider(self, spider):
    self.all_tags = set()
    for p in self.tags:
        for tag in self.tags[p]:
            self.all_tags.add(tag)

    with open('data.json', 'r+') as f:
        if not os.path.exists('codes'):
            os.mkdir('codes')
        os.chdir('codes')
        spider.logger.info("Saving files in codes folder...")
        for line in f:
            sub = json.loads(line)
            self.save_code(sub, spider)
        f.seek(0)
        f.truncate()  # clear the data file after writing all the codes
    os.chdir('..')

def save_code(self, sub, spider):
    code = "/**\n"
    code += " * Author    : {}\n".format(settings.USER.split('@')[0])
    code += " * Lang      : {}\n".format(sub['lang'])
    code += " * Date      : {}\n".format(sub['date'])
    code += " * Problem   : {}\n".format(sub['name'])
    code += " * CPU       : {}\n".format(sub['cpu'])
    code += " * Memory    : {}\n".format(sub['mem'])
    code += "**/\n"
    code += sub['code']
    for tag in self.all_tags:
        if tag in self.tags[sub['pid']]:
            self.save_to_folder(tag, sub['lang'], sub['name'], code, sub['subid'], spider)

def save_to_folder(self, tag, lang, name, code, subid, spider):
    if not os.path.exists(tag):
        os.mkdir(tag)
    os.chdir(tag)

    if lang == 'C++':
        ext = '.cpp'
    elif lang == 'JAVA':
        ext = '.java'
    elif lang == 'C':
        ext = '.c'
    else:
        ext = 'unknown'
    filebase = 'LightOJ ' + name
    filename = filebase + ext
    v = 1
    while os.path.exists(filename):
        v += 1
        filename = filebase + ' v' + str(v) + ext
    with open(filename, 'w') as f:
        f.write(code)
        spider.done.append(subid)  # this will explained later
    os.chdir('..')

```
Now our spider should save all the codes properly. But before finishing, we'll do one more things. What happens when we run our spider again? It'll scrape all the submissions again. Instead we could make it scrape only the new submissions. To do that, we have to save which submissions have been scraped.

We'll add a `done` property in our spider when it's created. We'll add he `subid`s when we scrape a submission. And when our spider closes we'll save them to a file. We'll have to add a `__init__` method to our spider. It will be called when the instance of our spider is created by scrapy.
```python
def __init__(self, *args, **kwargs):
    super(LojSpider, self).__init__(*args, **kwargs)
    with open('done.json', 'w+') as f:
        text = f.read()
    if text:
        self.done = json.loads(text)
    else:
        self.done = []
```
Now we need to save it when the spider is done scraping. Scrapy provides a way to do things when a spider closes. We need to create method `closed`. It will be called when the spider closes.
```python
def closed(self, reason):
    with open('done.json', 'w') as f:
        json.dump(self.done, f)
```

We have to add this to `parse_all_sub` method of our spider
```python
subid = a.css('::text').extract_first().strip()
if subid in self.done:
    continue
```

Now our progress will be saved in a `done.josn` file.