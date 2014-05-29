About
-----

Ferrit is an API driven web crawler service written in Scala using [Akka](http://akka.io), [Spray.IO](http://spray.io) and [Cassandra](http://cassandra.apache.org).

I created it to help me learn more about small service design using Akka and the *Functional Reactive* programming style.


Features
--------

Ferrit is a [focused web crawler](http://en.wikipedia.org/wiki/Focused_crawler) with separate crawl configurations and jobs per website. For a whole-internet style of crawler there is Nutch.

* The service is managed via a REST/JSON
* Data is stored in Cassandra - such as crawler configurations and job data
* A novel feature: configuring regex URI filters can be tricky. Ferrit lets you verify your filter rules ahead of time with tests to ensure that resources will either be fetched or not fetched as expected.


What else is there?

* *Politeness* is important. Ferrit supports the [Robots Exclusion Standard](http://www.robotstxt.org) (robots.txt). Although a custom fetch delay can be set per crawler configuration the final decision about the crawl delay is with the site webmaster as defined by the crawl-delay directive in robots.txt. Crawlers require a [User-agent](http://en.wikipedia.org/wiki/User_agent#Format_for_automated_agents_.28bots.29) property be set so that webmasters can contact the owner if necessary.
* *Crawl depth* can be set to limit fetches to the top N pages of a given website
* *URI filtering* is used to decide which resources should be fetched, e.g. fetch only HTML pages (uses accept/reject rules with regular expressions).
* *URI normalization* - helps to reduce fetching of duplicate resources.
* *Spider trap* prevention: in some cases jobs can run on indefinitely. Crawlers can be configured with basic safeguards such as a *crawl timeout* and *max fetches*.
* *Concurrent job* support - this is pretty basic so far, needs more exploration.


Requirements
------------

* Java 7/8
* Scala 2.10
* SBT 0.13.2
* Cassandra 2.0.3

Runs on Linux and Windows.


Build and Run
-------------

There is no binary file with this project so just checkout and build with sbt. It should only take a few minutes.

> B.t.w. before starting Ferrit do make sure Cassandra is already running. The first time you run the project you'll need to create a new keyspace and tables in Cassandra. Just open this [Cassandra schema file](https://raw.githubusercontent.com/reggoodwin/ferrit/master/src/main/resources/cassandra-schema.sql), open the CQL console and paste the contents of the file into it.

You can build/run Ferrit one of two ways:

(1) Run from within sbt (uses the excellent Spray [sbt-revolver](https://github.com/spray/sbt-revolver) plugin):

    cd <root project directory>
    sbt
    re-start

    // then make API calls to configure crawler
    // run crawls ...
    // when finished stop with:

    re-stop


(2) Assemble and run the executable Jar (uses [sbt-assembly](https://github.com/sbt/sbt-assembly) plugin):

    // Build Ferrit
    // This takes a while, is resource intensive, builds a Jar and places in /bin
    cd <this project directory>
    sbt assembly
  
    // To start Ferrit:
    cd bin
    ferrit

    // To shutdown (in new console window)
    curl -X POST localhost:6464/shutdown

A succesful startup will display:

    Ferrit -                                  _ _
    Ferrit -               ____ __  _ _  _ _ (_| )_
    Ferrit - -------------| __// _)| '_)| '_)| | |_--------------
    Ferrit - -------------| _| \__)|_|--|_|--|_|\__)-------------
    Ferrit - =============|_|====================================
    Ferrit -
    Ferrit - ------------ THE  W E B  C R A W L E R -------------
    Ferrit -
    Cluster - Starting new cluster with contact points [/127.0.0.1]
    ControlConnection - [Control connection] Refreshing node list and token map
    ControlConnection - [Control connection] Refreshing schema
    ControlConnection - [Control connection] Successfully connected to /127.0.0.1
    Session - Adding /127.0.0.1 to list of queried hosts
    Ferrit - Server started on http://localhost:6464



The host and port settings can be changed in: src/main/resources/application.conf.

    app {
      server {
        host = "127.0.0.1"
        port = 6464
      }
      ...
    }


> Check that the service started correctly by visiting http://localhost:6464



How to Run a Crawl Job
------------------------

Here's the 5 minute guide to using the API to create a crawler configuration and start a crawl job.
(There is also a user interface project on the way which is preferable to operating with curl).

(1) First create a new crawler configuration.

This example creates a sample configuration of a crawler that will fetch content from the W3C website. 
Before issuing the POST, insert a user agent string value where you see the "userAgent" property of the JSON sample below. Example:

    Your Name (contact email)

After you POST this configuration, copy the crawlerId property returned in the JSON response.

    curl -XPOST "localhost:6464/crawlers" --header "Content-Type: application/json" -d '{
        "id": "new",
        "crawlerName": "The W3C",
        "seeds": [
            "http://www.w3.org/"
        ],
        "uriFilter": {
            "filterClass": "org.ferrit.core.filter.PriorityRejectUriFilter",
            "rules": [
                "accept: http://www.w3.org/",
                "reject: (?i).*(\\.(jpe?g|png|gif|bmp))$"
            ]
        },
        "tests": [
            "should accept: http://www.w3.org/standards/webdesign/",
            "should reject: http://www.w3.org/2012/10/wplogo_transparent.png"
        ],
        "userAgent": #CHANGE ME#,
        "maxDepth": 2,
        "maxFetches": 10,
        "maxQueueSize": 1000,
        "maxRequestFails": 0.2,
        "crawlDelayMillis": 1000,
        "crawlTimeoutMillis": 100000
    }'

(2) Run a new crawl job using this new crawler configuration:

    curl -XPOST "localhost:6464/job_processes" --header "Content-Type: application/json" -d '{
        "id": "#ID-OF-NEW-CRAWLER#"
    }'


The job should start and finish after about a minute because only 10 resources are fetched.
Check the Ferrit console log for progress ...


> Important -
web crawlers get a bad reputation because they often crawl too aggressively so please use Ferrit politely. For the example above the maximum number of fetches is set to an artificially low 10 pages. Please don't increase this unless coincidentally you really intend to be crawling the W3C website! I test crawler functionality against a small website running on Apache locally before scaling up to real websites.

API Documentation
-----------------

## Crawlers

#### GET /crawlers

Returns details of stored crawler configurations. Todo: add search options and paging.

#### GET /crawlers/{crawlerId}

Returns details of a stored crawler configuration.

#### GET /crawlers/{crawlerId}/jobs

Returns an array of recent jobs that run or are running for a crawler.

#### GET /crawlers/{crawlerId}/jobs/{jobId}

Returns details about a job.


#### GET /crawlers/{crawlerId}/jobs/{jobId}/assets

Retrieve asset metadata collected by a crawler during the given job.


#### GET /crawlers/{crawlerId}/jobs/{jobId}/fetches

Retrieve details about fetches for a particular job.


#### POST /crawlers

Store a new crawler configuration, the request body should be a JSON crawl config. See example above ...

Properties:

| Property | What it Does |
| -------- | ------------ |
| crawlerName | the display name for the crawler as would be shown in a UI. |
| seeds | a string array, each being a starting URI to hint where the crawl should start. |
| userAgent | a string that is included in the User-agent header sent with each fetch request. Is a mandatory field and required for crawl politeness. |
| maxDepth | set a limit on the depth of the website crawled. |
| maxFetches | set a limit on the total number of fetches during a crawl, helps to prevent unbounded crawls due to configuration error. |
| crawlDelayMillis | this controls the delay between each fetch, e.g. 100. A crawl-delay directive found in robots.txt will take priority over this value. |
| crawlTimeoutMillis | set a time limit on the crawl, helps to prevent unbounded crawls due to configuration error. |
| maxRequestFails | set a limit on how fetches can fail before the crawler gives up. A value of 0.2 means stop if 20% of fetches have failed. |
| maxQueueSize | set a limit on the size of the internal Frontier (aka fetch queue). Not especially useful, more to catch unusual crawl situations. |
| uriFilter | controls which links are followed, can be a PriorityRejectUriFilter or FirstMatchUriFilter. |
| tests | allows you to provide example URIs that should match or not match your URI filter rules. Means that you can change your URI filter rules without breaking the expected behaviour. |


#### PUT /crawlers/{crawlerId}

Updates an existing crawler configuration, the request entity should be a JSON crawl configuration.


#### DELETE /crawlers/{crawlerId}

Deletes an existing crawler configuration.
Fetches and job history associated with the crawler will be removed automatically by Cassandra after the time to live has expired (set in application.conf).


#### POST /crawlers/config_test

Tests a crawler configuration and one or more additional URIs. In practice this is just testing the UriFilter assigned to the crawler configuration.

    {
        config: {
            "id": "new",
            "crawlerName": "Name of the Crawler", 
            "seeds": [
                "http://site.net"
            ],
            ... etc ...
            },
        uris: [uri1, uri2 ...]
    }


## Job Processes

#### GET /job_processes

Returns an array of all the running jobs known to the crawl manager.


#### POST /job_processes

Starts a new crawl, the request body should be a JSON object with a crawler ID value. See example above ...


#### DELETE /job_processes/{jobId}

Sends a stop request for the given running job.

#### DELETE /job_processes
    
Sends a stop request for all running crawl jobs.


## Crawl Jobs


#### GET /jobs

Returns an array of crawl jobs. Defaults to returning jobs that ran today. Jobs are ordered most recent first.

To fetch jobs for other days include the date query string parameter with the given date format, e.g.

    /jobs?date=YYYY-MM-DD


More About Persistence
----------------------

Each crawl job generates a completely separate document set. That means URI uniqueness is scoped at the job level. This design choice means that writes can be super fast because there is no need for a *read-before-write* to see if a resource should be updated rather than inserted (read-before-write is an anti-pattern in Cassandra anyway). Fetched documents have a time-to-live setting so that old content is removed automatically over time without needing a manual clean up process.


What's Missing?
---------------

* Usability! Yes, running a crawler with curl is no fun and rather error prone. Fortunately there is a user interface project in the works that will be sure to work wonders ...
* No [web scraping](http://en.wikipedia.org/wiki/Web_scraper) functionality added (yet), apart from automatic link extraction. So in a nutshell you can't actually do anything with this crawler except crawl !!
* No content deduplication support
* Redirect responses from servers are not (yet) handled properly
* Job clustering is not supported
* No backpressure support in the Journal (not an issue at present because Cassandra writes are so fast)
* JavaScript is not evaluated so links in dynamically generated DOM content are not discovered.


Where's the User Interface?
---------------------------

I have a web interface for this service in another Play Framework / AngularJS project.
The code for that other project needs to be cleaned up before publishing.


Influences
----------

Guess you already know about those other great open source web crawler projects out there which have influenced this design. First there is the granddaddy of the Java pack [Apache Nutch](https://nutch.apache.org). Then also there is the mature [Scrapy](http://scrapy.org) project which makes me yearn to be able to program Python! I also found that [Crawler4J](https://code.google.com/p/crawler4j) is good to study when first starting out because there is not too much code to review.


Known Issues
------------

* Most of the websites I have crawled are fine except for a few where the crawl job just aborts. I need to to fix that.
* Not all links extracted from pages can be parsed. When unparseable links are found they are logged but not added to the Frontier (aka queue) for fetching.
