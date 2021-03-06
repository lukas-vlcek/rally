Creating custom tracks
======================

.. note::
    Please see the :doc:`track reference </track>` for more information on the structure of a track.


Example track
-------------

Let's create an example track step by step. We will call this track "tutorial". The track consists of two components: the data set and the actual track specification which describes the workload that Rally should apply. We will store everything in the directory ``~/rally-tracks/tutorial`` but you can choose any other location.

First, we need some data. `Geonames <http://www.geonames.org/>`_ provides geo data under a `creative commons license <http://creativecommons.org/licenses/by/3.0/>`_. We will download `allCountries.zip <http://download.geonames.org/export/dump/allCountries.zip>`_ (around 300MB), extract it and inspect ``allCountries.txt``.

You will note that the file is tab-delimited but we need JSON to bulk-index data with Elasticsearch. So we can use a small script to do the conversion for us::

    import json

    cols = (("geonameid", "int"),
            ("name", "string"),
            ("asciiname", "string"),
            ("alternatenames", "string"),
            ("latitude", "double"),
            ("longitude", "double"),
            ("feature_class", "string"),
            ("feature_code", "string"),
            ("country_code", "string"),
            ("cc2", "string"),
            ("admin1_code", "string"),
            ("admin2_code", "string"),
            ("admin3_code", "string"),
            ("admin4_code", "string"),
            ("population", "long"),
            ("elevation", "int"),
            ("dem", "string"),
            ("timezone", "string"))


    def main():
        with open("allCountries.txt", "rt", encoding="UTF-8") as f:
            for line in f:
                tup = line.strip().split("\t")
                record = {}
                for i in range(len(cols)):
                    name, type = cols[i]
                    if tup[i] != "":
                        if type in ("int", "long"):
                            record[name] = int(tup[i])
                        elif type == "double":
                            record[name] = float(tup[i])
                        else:
                            record[name] = tup[i]
                print(json.dumps(record, ensure_ascii=False))


    if __name__ == "__main__":
        main()

Store the script as ``toJSON.py`` in our tutorial directory (``~/rally-tracks/tutorial``) and invoke the script with ``python3 toJSON.py > documents.json``.

We also need a mapping file for our documents. For details on how to write a mapping file, see `the Elasticsearch documentation on mappings <https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html>`_ and look at an `example mapping file <https://github.com/elastic/rally-tracks/blob/master/geonames/mappings.json>`_. Place the mapping file in the tutorial directory.

Finally, add a file called ``track.json`` right next to the mapping file::

    {
      "short-description": "Tutorial benchmark for Rally",
      "description": "This test indexes 8.6 million documents (POIs from Geonames) using 8 clients and 5000 docs per bulk request against Elasticsearch",
      "indices": [
        {
          "name": "geonames",
          "types": [
            {
              "name": "type",
              "mapping": "mappings.json",
              "documents": "documents.json",
              "document-count": 8647880,
              "uncompressed-bytes": 2790927196
            }
          ]
        }
      ],
      "operations": [
        {
          "name": "index",
          "operation-type": "index",
          "bulk-size": 5000
        },
        {
          "name": "force-merge",
          "operation-type": "force-merge"
        },
        {
          "name": "query-match-all",
          "operation-type": "search",
          "body": {
            "query": {
              "match_all": {}
            }
          }
        }
      ],
      "challenges": [
        {
          "name": "index-and-query",
          "description": "",
          "default": true,
          "index-settings": {
            "index.number_of_replicas": 0
          },
          "schedule": [
            {
              "operation": "index",
              "warmup-time-period": 120,
              "clients": 8
            },
            {
              "operation": "force-merge",
              "clients": 1
            },
            {
              "operation": "query-match-all",
              "clients": 8,
              "warmup-iterations": 1000,
              "iterations": 1000,
              "target-throughput": 100
            }
          ]
        }
      ]
    }


A few things to note:

* If you define multiple challenges, Rally will run the challenge where ``default`` is set to ``true``. If you want to run a different challenge, provide the command line option ``--challenge=YOUR_CHALLENGE_NAME``.
* You can add as many queries as you want. We use the `official Python Elasticsearch client <http://elasticsearch-py.readthedocs.org/>`_ to issue queries.
* The numbers below the ``types`` property are needed to verify integrity and provide progress reports.

.. note::

    You can store any supporting scripts along with your track. However, you need to place them in a directory starting with "_", e.g. "_support". Rally loads track plugins (see below) from any directory but will ignore directories starting with "_".

.. note::

    We have defined a `JSON schema for tracks <https://github.com/elastic/rally/blob/master/esrally/resources/track-schema.json>`_ which you can use to check how to define your track. You should also check the tracks provided by Rally for inspiration.

When you invoke ``esrally list tracks --track-path=~/rally-tracks/tutorial``, the new track should now appear::

    dm@io:~ $ esrally list tracks --track-path=~/rally-tracks/tutorial
    
        ____        ____
       / __ \____ _/ / /_  __
      / /_/ / __ `/ / / / / /
     / _, _/ /_/ / / / /_/ /
    /_/ |_|\__,_/_/_/\__, /
                    /____/
    Available tracks:
    
    Name        Description                   Documents    Compressed Size  Uncompressed Size  Default Challenge  All Challenges
    ----------  ----------------------------- -----------  ---------------  -----------------  -----------------  ---------------
    tutorial    Tutorial benchmark for Rally      8647880  N/A              2.6 GB             index-and-query    index-and-query

Congratulations, you have created your first track! You can test it with ``esrally --track-path=~/rally-tracks/tutorial`` and run specific challenges with ``esrally --track-path=~/rally-tracks/tutorial --challenge=index-and-query``.

.. _add_track_test_mode:

Adding support for test mode
----------------------------

When you invoke Rally with ``--test-mode``, it switches to a mode that allows you to check your track very quickly for syntax errors. To achieve that, it will postprocess its internal track representation after loading it:

* Iteration-based tasks will run at most one warmup iteration and one measurement iteration.
* Time-period-based task will run for at most 10 seconds without any warmup.

Rally will postprocess all data file names of a track. So instead of ``documents.json.bz2``, Rally will attempt to find ``documents-1k.json.bz2`` and will assume it contains 1.000 documents. However, you need to prepare these data files otherwise this test mode is not supported.

The preparation is very easy. Just pick 1.000 documents for every data file in your track. We choose the first 1.000 here but it does not matter usually which part you choose: ``head -n 1000 documents.json > documents-1k.json``.

Structuring your track
----------------------

``track.json`` is just the entry point to a track but you can split your track as you see fit. Suppose you want to add more challenges to the track above but you want to keep them in a separate files. Let's start by storing our challenge in a separate file, e.g in ``challenges/index-and-query.json``. Create the directory and store the following in ``index-and-query.json``::

    {
          "name": "index-and-query",
          "description": "",
          "default": true,
          "index-settings": {
            "index.number_of_replicas": 0
          },
          "schedule": [
            {
              "operation": "index",
              "warmup-time-period": 120,
              "clients": 8
            },
            {
              "operation": "force-merge",
              "clients": 1
            },
            {
              "operation": "query-match-all",
              "clients": 8,
              "warmup-iterations": 1000,
              "iterations": 1000,
              "target-throughput": 100
            }
          ]
        }

Now modify ``track.json`` so it knows about your new file::


    {
      "short-description": "Tutorial benchmark for Rally",
      "description": "This test indexes 8.6 million documents (POIs from Geonames) using 8 clients and 5000 docs per bulk request against Elasticsearch",
      "indices": [
        {
          "name": "geonames",
          "types": [
            {
              "name": "type",
              "mapping": "mappings.json",
              "documents": "documents.json",
              "document-count": 8647880,
              "uncompressed-bytes": 2790927196
            }
          ]
        }
      ],
      "operations": [
        {
          "name": "index",
          "operation-type": "index",
          "bulk-size": 5000
        },
        {
          "name": "force-merge",
          "operation-type": "force-merge"
        },
        {
          "name": "query-match-all",
          "operation-type": "search",
          "body": {
            "query": {
              "match_all": {}
            }
          }
        }
      ],
      "challenges": [
        {% include "challenges/index-and-query.json" %}
      ]
    }

We replaced the challenge content with  ``{% include "challenges/index-and-query.json" %}`` which tells Rally to include the challenge from the provided file. You can use ``include`` on arbitrary parts of your track.

However, if your track consists of multiple challenges it can be cumbersome to include them all explicitly. Therefore Rally brings a ``collect`` helper that collects all related files for you. Let's adapt our track to use it::

    {% import "rally.helpers" as rally %}
    {
      "short-description": "Tutorial benchmark for Rally",
      "description": "This test indexes 8.6 million documents (POIs from Geonames) using 8 clients and 5000 docs per bulk request against Elasticsearch",
      "indices": [
        {
          "name": "geonames",
          "types": [
            {
              "name": "type",
              "mapping": "mappings.json",
              "documents": "documents.json",
              "document-count": 8647880,
              "uncompressed-bytes": 2790927196
            }
          ]
        }
      ],
      "operations": [
        {
          "name": "index",
          "operation-type": "index",
          "bulk-size": 5000
        },
        {
          "name": "force-merge",
          "operation-type": "force-merge"
        },
        {
          "name": "query-match-all",
          "operation-type": "search",
          "body": {
            "query": {
              "match_all": {}
            }
          }
        }
      ],
      "challenges": [
        {{ rally.collect(parts="challenges/*.json") }}
      ]
    }

We changed two things here. First, we imported helper functions from Rally by adding ``{% import "rally.helpers" as rally %}`` in line 1. Second, we used Rally's ``collect`` helper to find and include all JSON files in the "challenges" subdirectory with the statement ``{{ rally.collect(parts="challenges/*.json") }}``. When you add new challenges in this directory, Rally will automatically pick them up.

.. note::

    If you want to check the final result, please check Rally's log file. Rally will print the full rendered track there after it has loaded it successfully.

You can even use `Jinja2 variables <http://jinja.pocoo.org/docs/2.9/templates/#assignments>`_ but you need to import the Rally helpers a bit differently then. You also need to declare all variables before the ``import`` statement::

        {% set clients = 16 %}
        {% import "rally.helpers" as rally with context %}

If you use this idiom you can then refer to variables inside your snippets with ``{{ clients }}``.

Sharing your track with others
------------------------------

At the moment your track is only available on your local machine but maybe you want to share it with other people in your team. You can share the track itself in any way you want, e.g. you can check it into version control. However, you will most likely not want to commit the potentially huge data file. Therefore, you can expose the data via http (e.g. via S3) and Rally can download it from there. To make this work, you need to add an additional property ``data-url`` at the top-level of your ``track.json`` file which contains the URL from where to download your documents. Rally expects that the URL points to the parent path and will append the document file name automatically.

It is also recommended that you compress your document corpus to save network bandwidth. We recommend to use bzip2 compression. You can create a compressed archive with the following command::

    bzip2 -9 -c documents.json > documents.json.bz2

If you want to support the test mode, don't forget to also compress your test mode corpus with::

    bzip2 -9 -c documents-1k.json > documents-1k.json.bz2

Then upload ``documents.json.bz2`` and ``documents-1k.json.bz2`` to the remote location.

Finally, specify the compressed file name in your ``track.json`` file::

    {
      "short-description": "Tutorial benchmark for Rally",
      "description": "This test indexes 8.6 million documents (POIs from Geonames) using 8 clients and 5000 docs per bulk request against Elasticsearch",
      "data-url": "http://benchmarks.elasticsearch.org.s3.amazonaws.com/corpora/geonames",
      "indices": [
        {
          "name": "geonames",
          "types": [
            {
              "name": "type",
              "mapping": "mappings.json",
              "documents": "documents.json.bz2",
              "document-count": 8647880,
              "compressed-bytes": 197857614,
              "uncompressed-bytes": 2790927196
            }
          ]
        }
      ],
      ...
    }

Specifying ``compressed-bytes`` (file size of ``documents.json.bz2``) and ``uncompressed-bytes`` (file size of ``documents.json``) is optional but helps Rally to provide progress indicators and also verify integrity.

How to contribute a track
-------------------------

First of all, please read Rally's `contributors guide <https://github.com/elastic/rally/blob/master/CONTRIBUTING.md>`_.

If you want to contribute your track, follow these steps:

1. Create a track JSON file and mapping files as described above and place them in a separate folder in the ``rally-tracks`` repository. Please also add a README file in this folder which contains licensing information (respecting the licensing terms of the source data). Note that pull requests for tracks without a license cannot be accepted.
2. Upload the associated data so they can be publicly downloaded via HTTP. The data should be compressed either as .bz2 (recommended) or as .zip. Also, don't forget to upload the "-1k" data files to support test mode properly.
3. Create a pull request in the `rally-tracks Github repo <https://github.com/elastic/rally-tracks>`_.

You've now mastered the basics of track development for Rally. It's time to pat yourself on the back before you dive into the advanced topics!

Advanced topics
---------------

Template Language
^^^^^^^^^^^^^^^^^

Rally uses `Jinja2 <http://jinja.pocoo.org/docs/dev/>`_ as template language. This allows you to use Jinja2 expressions in track files.


Extension Points
""""""""""""""""

Rally also provides a few extension points to Jinja2:

* ``now``: This is a global variable that represents the current date and time when the template is evaluated by Rally.
* ``days_ago()``: This is a `filter <http://jinja.pocoo.org/docs/dev/templates/#filters>`_ that you can use for date calculations.

You can find an example in the logging track::

    {
      "name": "range",
        "index": "logs-*",
        "type": "type",
        "body": {
          "query": {
            "range": {
              "@timestamp": {
                "gte": "now-{{'15-05-1998' | days_ago(now)}}d/d",
                "lt": "now/d"
              }
            }
          }
        }
      }
    }

The data set that is used in the logging track starts on 26-04-1998 but we want to ignore the first few days for this query, so we start on 15-05-1998. The expression ``{{'15-05-1998' | days_ago(now)}}`` yields the difference in days between now and the fixed start date and allows us to benchmark time range queries relative to now with a predetermined data set.

.. _adding_tracks_custom_param_sources:

Custom parameter sources
^^^^^^^^^^^^^^^^^^^^^^^^

.. note::

    This is a rather new feature and the API may change! However, the effort to use custom parameter sources is very low.

.. warning::

    Your parameter source is on a performance-critical code-path so please double-check with :ref:`Rally's profiling support <clr_enable_driver_profiling>` that you did not introduce any bottlenecks.


Consider the following operation definition::

    {
      "name": "term",
      "operation-type": "search",
      "body": {
        "query": {
          "term": {
            "body": "physician"
          }
        }
      }
    }

This query is defined statically in the track specification but sometimes you may want to vary parameters, e.g. search also for "mechanic" or "nurse". In this case, you can write your own "parameter source" with a little bit of Python code.

First, define the name of your parameter source in the operation definition::

    {
      "name": "term",
      "operation-type": "search",
      "param-source": "my-custom-term-param-source"
      "professions": ["mechanic", "physician", "nurse"]
    }

Rally will recognize the parameter source and looks then for a file ``track.py`` in the same directory as the corresponding JSON file. This file contains the implementation of the parameter source::

    import random


    def random_profession(indices, params):
        # you must provide all parameters that the runner expects
        return {
            "body": {
                "query": {
                    "term": {
                        "body": "%s" % random.choice(params["professions"])
                    }
                }
            },
            "index": None,
            "type": None,
            "use_request_cache": False
        }

    def register(registry):
        registry.register_param_source("my-custom-term-param-source", random_profession)

The example above shows a simple case that is sufficient if the operation to which your parameter source is applied is idempotent and it does not matter whether two clients execute the same operation.

The function ``random_profession`` is the actual parameter source. Rally will bind the name "my-custom-term-param-source" to this function by calling ``register``. ``register`` is called by Rally before the track is executed.

The parameter source function needs to declare the two parameters ``indices`` and ``params``. `indices` contains all indices of this track and ``params`` contains all parameters that have been defined in the operation definition in ``track.json``. We use it in the example to read the professions to choose.

If you need more control, you need to implement a class. The example above, implemented as a class looks as follows::

    import random


    class TermParamSource:
        def __init__(self, indices, params):
            self._indices = indices
            self._params = params

        def partition(self, partition_index, total_partitions):
            return self

        def size(self):
            return 1

        def params(self):
            # you must provide all parameters that the runner expects
            return {
                "body": {
                    "query": {
                        "term": {
                            "body": "%s" % random.choice(self._params["professions"])
                        }
                    }
                },
                "index": None,
                "type": None,
                "use_request_cache": False
            }


    def register(registry):
        registry.register_param_source("my-custom-term-param-source", TermParamSource)


Let's walk through this code step by step:

* Note the method ``register`` where you need to bind the name in the track specification to your parameter source implementation class similar to the simple example.
* The class ``TermParamSource`` is the actual parameter source and needs to fulfill a few requirements:

    * It needs to have a constructor with the signature ``__init__(self, indices, params)``. You don't need to store these parameters if you don't need them.
    * ``partition(self, partition_index, total_partitions)`` is called by Rally to "assign" the parameter source across multiple clients. Typically you can just return ``self`` but in certain cases you need to do something more sophisticated. If each clients needs to act differently then you can provide different parameter source instances here.
    * ``size(self)``: This method is needed to help Rally provide a proper progress indication to users if you use a warmup time period. For bulk indexing, this would return the number of bulks (for a given client). As searches are typically executed with a pre-determined amount of iterations, just return ``1`` in this case.
    * ``params(self)``: This method needs to return a dictionary with all parameters that the corresponding "runner" expects. For the standard case, Rally provides most of these parameters as a convenience, but here you need to define all of them yourself. This method will be invoked once for every iteration during the race. We can see that we randomly select a profession from a list which will be then be executed by the corresponding runner.

.. note::

    Be aware that ``params(self)`` is called on a performance-critical path so don't do anything in this method that takes a lot of time (avoid any I/O). For searches, you should usually throttle throughput anyway and there it does not matter that much but if the corresponding operation is run without throughput throttling, please double-check that you did not introduce a bottleneck in the load test driver with your custom parameter source.

In the implementation of custom parameter sources you can access the Python standard API. Using any additional libraries is not supported.

You can also implement your parameter sources and runners in multiple Python files but the main entry point is always ``track.py``. The root package name of your plugin is the name of your track.

.. _adding_tracks_custom_runners:

Custom runners
^^^^^^^^^^^^^^

.. warning::

    Your runner is on a performance-critical code-path so please double-check with :ref:`Rally's profiling support <clr_enable_driver_profiling>` that you did not introduce any bottlenecks.

You cannot only define custom parameter sources but also custom runners. Runners execute an operation against Elasticsearch. Out of the box, Rally supports the following operations:

* Bulk indexing
* Force merge
* Searches
* Index stats
* Nodes stats

If you want to use any other operation, you can define a custom runner. Consider, we want to use the percolate API with an older version of Elasticsearch (note that it has been replaced by the percolate query in Elasticsearch 5.0). To achieve this, we c

In track.json specify an operation with type "percolate" (you can choose this name freely)::

    {
      "name": "percolator_with_content_google",
      "operation-type": "percolate",
      "body": {
        "doc": {
          "body": "google"
        },
        "track_scores": true
      }
    }


Then create a file ``track.py`` next to ``track.json`` and implement the following two functions::

    def percolate(es, params):
        es.percolate(
            index="queries",
            doc_type="content",
            body=params["body"]
        )


    def register(registry):
        registry.register_runner("percolate", percolate)


The function ``percolate`` is the actual runner and takes the following parameters:

* ``es``, which is the Elasticsearch Python client
* ``params`` which is a dict of parameters provided by its corresponding parameter source. Treat this parameter as read only and do not attempt to write to it.

This function can return either:

* Nothing at all. Then Rally will assume that by default ``1`` and ``"ops"`` (see below)
* A tuple of ``weight`` and a ``unit``, which is usually ``1`` and ``"ops"``. If you run a bulk operation you might return the bulk size here, for example in number of documents or in MB. Then you'd return for example ``(5000, "docs")`` Rally will use these values to store throughput metrics.
* A ``dict`` with arbitrary keys. If the ``dict`` contains the key ``weight`` it is assumed to be numeric and chosen as weight as defined above. The key ``unit`` is treated similarly. All other keys are added to the ``meta`` section of the corresponding service time and latency metrics records.

Similar to a parameter source you also need to bind the name of your operation type to the function within ``register``.

If you need more control, you can also implement a runner class. The example above, implemented as a class looks as follows::

    class PercolateRunner:
        def __enter__(self):
            return self

        def __call__(self, es, params):
            es.percolate(
                index="queries",
                doc_type="content",
                body=params["body"]
            )

        def __repr__(self, *args, **kwargs):
            return "percolate"

    def register(registry):
        registry.register_runner("percolate", PercolateRunner())


The actual runner is implemented in the method ``__call__`` and the same return value conventions apply as for functions. For debugging purposes you should also implement ``__repr__`` and provide a human-readable name for your runner. Finally, you need to register your runner in the ``register`` function. Runners also support Python's `context manager <https://docs.python.org/3/library/stdtypes.html#typecontextmanager>`_ interface. Rally uses a new context for each request. Implementing the context manager interface can be handy for cleanup of resources after executing an operation. Rally uses it for example to clear open scrolls.

.. note::

    You need to implement ``register`` just once and register all parameter sources and runners there.

Custom schedulers
^^^^^^^^^^^^^^^^^

.. warning::

    Your scheduler is on a performance-critical code-path so please double-check with :ref:`Rally's profiling support <clr_enable_driver_profiling>` that you did not introduce any bottlenecks.

If you want to rate-limit execution of tasks, you can specify a ``target-throughput`` (in operations per second). For example, Rally will attempt to run this term query 20 times per second::

  {
    "operation": "term",
    "target-throughput": 20
  }

By default, Rally will use a `deterministic distribution <https://en.wikipedia.org/wiki/Degenerate_distribution>`_ to determine when to schedule the next operation. This means, that it will execute the term query at 0, 50ms, 100ms, 150ms and so on. Note that the scheduler is aware of the number of clients. Consider this example::

  {
    "operation": "term",
    "target-throughput": 20,
    "clients": 4
  }

If Rally would not take the number of clients into account and would still issue requests (from each of the four clients) at the same points in time (i.e. 0, 50ms, 100ms, 150ms, ...), it would run at a target throughput of 4 * 20 = 80 operations per second. Hence, Rally will automatically reduce the rate at which each client will execute requests. Each client will issue requests at 0, 200ms, 400ms, 600ms, 800ms, 1000ms and so on. Each client issues five requests per second but as there are four of them, we still have a target throughput of 20 operations per second. You should keep this in mind, when writing your own custom schedules.

If you want to create a custom scheduler, create a file ``track.py`` next to ``track.json`` and implement the following two functions::

    import random

    def random_schedule(current):
        return current + random.randint(10, 900) / 1000.0


    def register(registry):
        registry.register_scheduler("my_random", random_schedule)

You can then use your custom scheduler as follows::

  {
    "operation": "term",
    "schedule": "my_random"
  }

The function ``random_schedule`` returns a floating point number which represents the next point in time when Rally should execute the given operation. This point in time is measured in seconds relative to the beginning of the execution of this task. The parameter ``current`` is the last return value of your function and is 0 for the first invocation. So, for example, this scheduler could return the following series: 0, 0.119, 0.622, 1.29, 1.343, 1.984, 2.233. Note that this implementation is usually not sufficient as it does not take into account the number of clients. Therefore, you will typically want to implement a full-blown scheduler which can also take parameters. Below is an example for our random scheduler::

    import random

    class RandomScheduler:
        def __init__(self, params):
            # assume one client by default
            clients = self.params.get("clients", 1)
            # scale accordingly with the number of clients!
            self.lower_bound = clients * self.params.get("lower-bound-millis", 10)
            self.upper_bound = clients * self.params.get("upper-bound-millis", 900)

        def next(self, current):
            return current + random.randint(self.lower_bound, self.upper_bound) / 1000.0


    def register(registry):
        registry.register_scheduler("my_random", RandomScheduler)

This implementation will now achieve the same rate independent of the number of clients. Additionally, we can pass the lower and upper bound for the random function from our track::

    {
        "operation": "term",
        "schedule": "my_random",
        "clients": 4,
        "lower-bound-millis": 50,
        "upper-bound-millis": 250
    }

Running tasks in parallel
^^^^^^^^^^^^^^^^^^^^^^^^^

Rally supports running tasks in parallel with the ``parallel`` element. Below you find a few examples that show how it should be used:

In the simplest case, you let Rally decide the number of clients needed to run the parallel tasks::


        {
          "parallel": {
            "warmup-iterations": 1000,
            "iterations": 1000,
            "tasks": [
              {
                "operation": "default",
                "target-throughput": 50
              },
              {
                "operation": "term",
                "target-throughput": 200
              },
              {
                "operation": "phrase",
                "target-throughput": 200
              },
              {
                "operation": "country_agg_uncached",
                "target-throughput": 50
              }
            ]
          }
        }
      ]
    }

Rally will determine that four clients are needed to run each task in a dedicated client.

However, you can also explicitly limit the number of clients::

        {
          "parallel": {
            "clients": 2,
            "warmup-iterations": 1000,
            "iterations": 1000,
            "tasks": [
              {
                "operation": "default",
                "target-throughput": 50
              },
              {
                "operation": "term",
                "target-throughput": 200
              },
              {
                "operation": "phrase",
                "target-throughput": 200
              },
              {
                "operation": "country_agg_uncached",
                "target-throughput": 50
              }
            ]
          }
        }

This will run the four tasks with just two clients. You could also specify more clients than there are tasks but these will then just idle.

You can also specify a number of clients on sub tasks explicitly (by default, one client is assumed per subtask). This allows to define a weight for each client operation. Note that you need to define the number of clients also on the ``parallel`` parent element, otherwise Rally would determine the number of totally needed clients again on its own::

        {
          "parallel": {
            "clients": 3,
            "warmup-iterations": 1000,
            "iterations": 1000,
            "tasks": [
              {
                "operation": "default",
                "target-throughput": 50
              },
              {
                "operation": "term",
                "target-throughput": 200
              },
              {
                "operation": "phrase",
                "target-throughput": 200,
                "clients": 2
              },
              {
                "operation": "country_agg_uncached",
                "target-throughput": 50
              }
            ]
          }
        }

This will ensure that the phrase query will be executed by two clients. All other ones are executed by one client.

.. warning::
    You cannot nest parallel tasks.