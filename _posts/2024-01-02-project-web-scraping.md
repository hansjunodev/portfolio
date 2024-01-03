---
title: "Project: Web Scraping"
date: 2024-01-02
---

Project date: ~2019

One of my earliest projects involved web scraping and database maintenance.
To the best of my memory, this is how it went:


# Workflow

1. Scrape [franchise.ftc.go.kr](https://franchise.ftc.go.kr/mnu/00013/program/userRqst/list.do) for data on all franchises in the country.

![Franchise Data](/portfolio/assets/table-example.png)
^(One table out of many)

2. Transform, normalize, and store the data.

![Data Transformation](/portfolio/assets/db-rowcount.png)

3. Query and present in a CSV format.

![CSV Presentation](/portfolio/assets/csv-files.png)

4. The client, a consulting firm, curates and sells this data as part of a handbook to their customers.


# Existing Stack

The initial implementation relied on a stack comprising:

- MySQL database
- Shell scripts for database operations and CSV generation
- PHP for web scraping

![Old Tools](/portfolio/assets/old-tools.png)
^(Some of the existing tools)

```sh
# makeBrandRaw.sh
mysql -u root --password='omitted'<<!  | sed  '1s/^/\xef\xbb\xbf/' | sed 's/	//g' > $DIR_EXCEL/$WORK_YEAR/raw_$year.csv 
  use $DATABASE

  select 
      a.brandId "ID", "," ,
      a.year "Year", "," ,
      ...
  !
exit;
```
^(A lot of it was hardcoded)

Naturally, maintaining this setup posed challenges, with frequent debugging sessions due to hardcoded elements and the dynamic nature of the website structure. 



# Redesigning the Scraping Tool

So, I decided to rewrite the source of the biggest headache--the scraping tool.

After a brief attempt at it via Python Requests + BeautifulSoup4, I realized the project would quickly become very convoluted and made the switch to [Scrapy](https://scrapy.org/). It had everything out of the box: caching, multithreading, crawling, redirection, response code handling, logging, and a pipeline architecture that made debugging effortless and patches trivial. With Scrapy, the process was dead simple:

Crawl the listing > send page down the pipeline > parse, transform, then insert into DB

The crawler would scrape:

```python
# Spider.py
def parse(self, response):
    page_type = response.url.split("/")[-1]

    if page_type.startswith("list"):
        listings = response.xpath('//ul[@class="paginationList"]'
                                  '/li/a/@href')
        for a in listings:
            yield response.follow(a, callback=self.parse)
    
    # More parsing logic here
```

Then send it down a custom pipeline:

```python
# settings.py
ITEM_PIPELINES = {
    "scraper.pipelines.ClassifyTablesPipeline": 100,
    "scraper.pipelines.ParseTablesPipeline": 200,
    "scraper.pipelines.CreateEntriesPipeline": 300,
    "scraper.pipelines.IdentifyTablePipeline": 400,
    "scraper.pipelines.TranslationPipeline": 500,
    "scraper.pipelines.PopulateFieldsPipeline": 600,
    "scraper.pipelines.DatabaseFormatPipeline": 700,
    "scraper.pipelines.MySQLStorePipeline": 800,
}
```


# The Challenge

The biggest hurdle was classifying the tables. They could be roughly categorized into three types:


- **1-Dimensional Table**:

![1D Table](/portfolio/assets/1d-table.png)

  ```json
  { "name": "JEONGMI FOOD", "businessSigns": "Hoya Chicken", "representative": "Jaegu Kang", ... }
  ```

- **2-Dimensional Table**:

![2D Table](/portfolio/assets/2d-table.png)

  ```json
  [
    { "year": "2016", "newlyOpened": 22, "terminationOfAgreement": 0, "termination": 4, "changeOfName": 0 },
    { "year": "2015", "newlyOpened": 5, "terminationOfAgreement": null, "termination": 3, "changeOfName": null },
    // ... other entries ...
  ]
  ```

- **2-Dimensional Layered Table**:

![2D Layered Table](/portfolio/assets/2d-layered-table.png)

  ```json
  [
    { "year": 2016, "area": "whole", "whole": 29, "franchiseDealer": 29, "directScore": 0 },
    { "year": 2015, "area": "whole", "whole": 11, "franchiseDealer": 11, "directScore": null },
    // ... other entries ...
  ]
  ```

I don't remember my exact thought process, but the questions I had were something like:
- If there are `<th>` tags in the `<body>`, is that a 1D table? 
- If there is only 1 `<tr>` in `<thead>`, is that a 2D table? 
- If there are any `rowspan` or `colspan` attributes, is that a layered 2D table? 
- How should layered tables be flattened? 
- Should the first column be treated as another header or a data field? 

I started listing out the attributes that gave each table their identity but couldn't come up with a satisfying, all-encompassing answer. After googling around to no avail, I took the easy way out. The parsing had to be hardcoded as it changed on a case-by-case basis, so I might as well hardcode the classification along with it. 

I ended up creating a heuristic using a predefined map of table names to field names for classification.


```json
{ "table1": ["Contract Period", "first", "prolongation"], "table2": ["year", "Newly Opened", "Termination of Agreement"], ... }
```

While the solution might have been somewhat messy, it allowed for efficient handling of evolving table structures.

# Finishing Up

To wrap up the project, I added features like JSON output for debugging, command-line options with argparse, basic documentation, and a database backup script. Though I didn't write any piece of code I'm particularly proud of, this project served as a valuable learning experience. As a beginner, it provided insights into Python coding practices and familiarized me with technologies such as xpath, requests, BeautifulSoup, Scrapy, MySQL, and more.

In hindsight, if I were to start the project anew, I might explore the use of a document database like MongoDB. Additionally, creating a simple web app for querying the database could streamline the client's experience, eliminating the need to navigate through multiple CSV files.
