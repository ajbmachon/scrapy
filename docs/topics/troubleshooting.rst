.. _topics-troubleshooting:

===============
Troubleshooting
===============

This document describes how to diagnose common Scrapy problems by understanding
where to look and what to check at each stage of the request lifecycle. Rather
than providing step-by-step debugging procedures, it focuses on developing a
mental model for mapping symptoms to their likely causes.

For detailed debugging techniques, see :ref:`topics-debug`. For understanding
the request lifecycle, see :ref:`topics-lifecycle`.

.. _troubleshooting-approach:

Diagnostic approach
===================

When Scrapy behaves unexpectedly, the first step is to identify which lifecycle
stage is affected. Scrapy processes requests through a predictable sequence:

1. **Spider yields requests** - Requests are created and yielded
2. **Scheduling** - Requests are filtered and queued
3. **Downloading** - Requests are fetched from the network
4. **Spider processing** - Responses are processed by callbacks
5. **Item processing** - Items pass through pipelines

Each stage has specific stats, signals, and log messages that indicate whether
it is functioning correctly. The diagnostic approach is to check these
indicators at each stage, starting from where you observe the problem and
working backward to find the root cause.

.. _troubleshooting-stats-overview:

Key stats for diagnosis
-----------------------

The stats collector provides counters for most operations. Access stats at the
end of a crawl in the log output (when :setting:`STATS_DUMP` is enabled), or
programmatically via ``crawler.stats.get_stats()``.

**Request flow stats:**

- ``scheduler/enqueued`` - Requests added to the scheduler queue
- ``scheduler/dequeued`` - Requests removed from queue for downloading
- ``dupefilter/filtered`` - Requests rejected as duplicates

**Response stats:**

- ``downloader/response_count`` - Responses received from the network
- ``downloader/response_status_count/{STATUS}`` - Responses by HTTP status code
- ``response_received_count`` - Responses received by the engine

**Item stats:**

- ``item_scraped_count`` - Items that passed through all pipelines
- ``item_dropped_count`` - Items dropped by pipelines
- ``item_dropped_reasons_count/{ExceptionClass}`` - Dropped items by reason

**Error stats:**

- ``downloader/exception_count`` - Download failures
- ``downloader/exception_type_count/{ExceptionClass}`` - Download failures by type
- ``spider_exceptions/count`` - Exceptions in spider callbacks
- ``spider_exceptions/{ExceptionClass}`` - Spider exceptions by type

For more on stats collection, see :ref:`topics-stats`.

.. _troubleshooting-requests-not-scheduled:

Requests not being scheduled or executed
========================================

When requests yield from a spider but nothing happens, the issue is typically
in the scheduling or downloading phase.

Symptoms
--------

- Spider runs but finishes quickly with few or no requests processed
- ``scheduler/enqueued`` stat is zero or lower than expected
- ``response_received_count`` is zero or very low
- No "Crawled" messages in the log

Where to look
-------------

**Check if requests are reaching the scheduler**

If ``scheduler/enqueued`` is zero, requests are not being scheduled. This can
occur when:

- The spider's ``start()`` or ``start_requests()`` method is not yielding requests
- Requests are being dropped by a signal handler on :signal:`request_scheduled`
- An exception is occurring before requests reach the scheduler

Check the log for "Error while reading start items" or "Error while processing
requests from start()" messages, which indicate exceptions in the spider's
start method.

**Check if requests are being filtered as duplicates**

The ``dupefilter/filtered`` stat shows how many requests were rejected by the
duplicate filter. A high value relative to ``scheduler/enqueued`` suggests:

- The spider is generating duplicate URLs
- URLs that should be unique are being fingerprinted identically
- The ``dont_filter=True`` parameter may be needed for certain requests

The default duplicate filter uses request fingerprints based on URL, method,
and body. See :setting:`DUPEFILTER_CLASS` for customization options.

**Check if requests are being dequeued**

Compare ``scheduler/enqueued`` to ``scheduler/dequeued``. If enqueued is high
but dequeued is low, requests are queued but not being processed. This can
indicate:

- Backpressure from the downloader (check :setting:`CONCURRENT_REQUESTS`)
- The scraper's response queue is full (check :setting:`SCRAPER_SLOT_MAX_ACTIVE_SIZE`)
- An early spider closure

**Check for download errors**

If ``scheduler/dequeued`` is high but ``response_received_count`` is low, check
the download error stats:

- ``downloader/exception_count`` - Total download failures
- ``downloader/exception_type_count/{ExceptionClass}`` - Failures by type
- ``retry/count`` - Number of retried requests
- ``retry/max_reached`` - Requests that exceeded retry limit

Common download failures include connection timeouts, DNS resolution failures,
and SSL errors. Check the log for "Error downloading" messages.

Relevant signals
----------------

These signals can help trace request flow:

- :signal:`request_scheduled` - Fired when a request is about to be scheduled
- :signal:`request_dropped` - Fired when the scheduler rejects a request
- :signal:`request_reached_downloader` - Fired when a request enters the downloader
- :signal:`request_left_downloader` - Fired when a request leaves the downloader

.. _troubleshooting-no-items:

Spider yielding no items
========================

When the spider processes responses but produces no items, the issue is in the
spider callback or item pipeline phase.

Symptoms
--------

- ``response_received_count`` is normal, but ``item_scraped_count`` is zero
- "Crawled" messages appear in the log, but no "Scraped" messages
- Items may be dropped or callbacks may be failing silently

Where to look
-------------

**Check if callbacks are executing**

If ``response_received_count`` is positive but no items appear, verify that
callbacks are being called. The ``spider_exceptions/count`` stat indicates
whether callbacks are raising exceptions.

Log messages at ERROR level with "Spider error processing" indicate callback
failures. Check the traceback to identify the exception source.

**Check if items are being yielded**

If callbacks run without error but yield no items, the issue is in the
callback logic. Common causes include:

- Selectors that match nothing (website structure changed)
- Conditional logic that filters out all items
- Callbacks that return ``None`` instead of yielding items

Use :ref:`topics-shell` to test selectors interactively against actual
responses.

**Check if items are being dropped**

The ``item_dropped_count`` stat shows items rejected by pipelines. If this
equals or exceeds the expected item count, check:

- Which pipeline is dropping items (``item_dropped_reasons_count/{ExceptionClass}``)
- Whether the :exc:`~scrapy.exceptions.DropItem` exception message explains why

**Check for pipeline errors**

The :signal:`item_error` signal fires when a pipeline raises an exception
other than :exc:`~scrapy.exceptions.DropItem`. Check the log for
"Error processing" messages at ERROR level.

**Check HTTP status codes**

The ``httperror/response_ignored_count`` stat shows responses ignored due to
HTTP status codes. By default, Scrapy ignores non-2xx responses. The
``httperror/response_ignored_status_count/{STATUS}`` stat breaks this down by
status code.

If the site returns 403, 404, or 500 errors, these responses will not reach
your callbacks unless you configure :setting:`HTTPERROR_ALLOWED_CODES` or
set ``handle_httpstatus_list`` on your spider.

Relevant signals
----------------

- :signal:`item_scraped` - Fired when an item passes all pipelines
- :signal:`item_dropped` - Fired when a pipeline drops an item
- :signal:`item_error` - Fired when a pipeline raises an unexpected exception
- :signal:`spider_error` - Fired when a callback raises an exception

.. _troubleshooting-middleware-pipeline:

Middleware or pipeline not running
==================================

When middleware or pipeline components appear to have no effect, the issue is
typically configuration-related.

Symptoms
--------

- Custom middleware has no visible effect on requests or responses
- Pipeline's ``process_item`` method is never called
- Expected logging from middleware or pipeline does not appear

Where to look
-------------

**Verify the component is enabled**

Middleware and pipelines must be explicitly enabled in settings. Check:

- :setting:`DOWNLOADER_MIDDLEWARES` for downloader middleware
- :setting:`SPIDER_MIDDLEWARES` for spider middleware
- :setting:`ITEM_PIPELINES` for item pipelines

Each setting maps class paths to integer priorities. A ``None`` value disables
a built-in component.

**Check priority ordering**

Middleware executes in priority order. For downloader middleware:

- ``process_request``: Lower priorities run first (toward the network)
- ``process_response``: Higher priorities run first (toward the spider)

For spider middleware:

- ``process_spider_input``: Lower priorities run first
- ``process_spider_output``: Higher priorities run first

If your middleware depends on another's output, verify the priorities allow
correct ordering.

**Check for NotConfigured exceptions**

Components can disable themselves by raising :exc:`~scrapy.exceptions.NotConfigured`
in their ``from_crawler`` class method. This typically occurs when required
settings are missing. Check the log at INFO level for "Enabled" and "Disabled"
messages during startup.

**Verify the component class path**

Settings require the full Python path to the class. Common errors include:

- Typos in the class name or module path
- Missing ``__init__.py`` files in package directories
- Import errors in the component module

Import errors will appear in the log at startup.

**Check method signatures**

Middleware methods must have correct signatures:

- Downloader middleware: ``process_request(self, request, spider)``
- Spider middleware: ``process_spider_input(self, response, spider)``
- Pipeline: ``process_item(self, item)``

Incorrect signatures will cause errors when the method is called.

.. _troubleshooting-configuration:

Common configuration mistakes
=============================

Many Scrapy issues stem from configuration errors. This section covers
frequently encountered mistakes.

Settings not taking effect
--------------------------

**Spider settings override project settings**

Settings defined in a spider's ``custom_settings`` attribute or
``update_settings`` method take precedence over project settings. If a setting
behaves differently per spider, check for spider-level overrides.

**Command-line settings have highest precedence**

Settings passed via ``-s`` on the command line override all other sources.
Check for conflicting command-line arguments.

**Settings type mismatches**

Some settings expect specific types. For example, :setting:`CONCURRENT_REQUESTS`
expects an integer. Passing a string may cause unexpected behavior.

Middleware ordering issues
--------------------------

**Built-in middleware disabled unintentionally**

Setting a custom middleware at a priority that conflicts with built-in
middleware can cause issues. To disable a built-in middleware, explicitly
set it to ``None``:

.. code-block:: python

    DOWNLOADER_MIDDLEWARES = {
        'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
        'myproject.middlewares.CustomUserAgentMiddleware': 400,
    }

**Retry middleware not handling errors**

The :class:`~scrapy.downloadermiddlewares.retry.RetryMiddleware` only retries
specific exceptions and status codes. If requests fail but are not retried,
check :setting:`RETRY_EXCEPTIONS` and :setting:`RETRY_HTTP_CODES`.

Request handling issues
-----------------------

**Requests filtered unexpectedly**

The :class:`~scrapy.downloadermiddlewares.offsite.OffsiteMiddleware` filters
requests to domains not in ``allowed_domains``. The ``offsite/filtered`` stat
shows filtered requests. Either add domains to ``allowed_domains`` or disable
the middleware.

**Depth limit reached**

The :setting:`DEPTH_LIMIT` setting restricts how many links deep the spider
will follow. The ``request_depth_count/{DEPTH}`` stats show the distribution.
Set ``DEPTH_LIMIT = 0`` to disable depth limiting.

**Request priority ignored**

Request priority affects dequeue order within the scheduler, but does not
guarantee execution order due to concurrency. High-priority requests may still
wait if the downloader is at capacity.

.. _troubleshooting-lifecycle-inspection:

Where to look at each lifecycle stage
=====================================

This section maps lifecycle stages to their diagnostic resources.

Spider start phase
------------------

**What happens**: The engine calls the spider's ``start()`` method (or
``start_requests()`` in older code patterns) to obtain initial requests.

**Log messages**:

- "Spider opened" (INFO) - Spider initialization completed
- "Error while reading start items" (ERROR) - Exception in start method
- "Error while processing requests from start()" (ERROR) - Exception processing start output

**Signals**:

- :signal:`spider_opened` - Fired when the spider is ready to crawl

Scheduling phase
----------------

**What happens**: Requests pass through duplicate filtering and enter the
scheduler queue.

**Key stats**:

- ``scheduler/enqueued``, ``scheduler/enqueued/memory``, ``scheduler/enqueued/disk``
- ``dupefilter/filtered``

**Signals**:

- :signal:`request_scheduled` - Before the request reaches the scheduler
- :signal:`request_dropped` - When the scheduler rejects a request

Download phase
--------------

**What happens**: Requests pass through downloader middleware, are fetched,
and responses pass back through middleware.

**Key stats**:

- ``downloader/request_count``, ``downloader/response_count``
- ``downloader/response_status_count/{STATUS}``
- ``downloader/exception_count``, ``downloader/exception_type_count/{TYPE}``
- ``httpcache/hit``, ``httpcache/miss`` (if caching enabled)
- ``retry/count``, ``retry/max_reached``

**Log messages**:

- "Crawled (STATUS) URL" (DEBUG) - Successful download
- "Error downloading URL" (ERROR) - Download failure
- "Retrying URL" (DEBUG) - Request being retried

**Signals**:

- :signal:`request_reached_downloader`, :signal:`request_left_downloader`
- :signal:`response_downloaded`, :signal:`response_received`
- :signal:`bytes_received`, :signal:`headers_received`

Spider callback phase
---------------------

**What happens**: Responses pass through spider middleware, callbacks execute,
and output passes back through middleware.

**Key stats**:

- ``response_received_count``
- ``spider_exceptions/count``, ``spider_exceptions/{TYPE}``
- ``httperror/response_ignored_count``

**Log messages**:

- "Spider error processing REQUEST" (ERROR) - Callback exception
- "Ignoring response STATUS URL" (INFO) - Response filtered by HTTP status

**Signals**:

- :signal:`spider_error` - Callback raised an exception

Item pipeline phase
-------------------

**What happens**: Items pass through each enabled pipeline in priority order.

**Key stats**:

- ``item_scraped_count``
- ``item_dropped_count``, ``item_dropped_reasons_count/{TYPE}``

**Log messages**:

- "Scraped from RESPONSE" (DEBUG) - Item successfully processed
- "Dropped: REASON" (WARNING or custom level) - Item dropped by pipeline
- "Error processing ITEM" (ERROR) - Pipeline exception

**Signals**:

- :signal:`item_scraped` - Item passed all pipelines
- :signal:`item_dropped` - Pipeline raised DropItem
- :signal:`item_error` - Pipeline raised unexpected exception

Spider idle and close phase
---------------------------

**What happens**: When no more work is pending, the spider enters idle state
and may close.

**Log messages**:

- "Closing spider (REASON)" (INFO) - Spider close initiated
- "Spider closed (REASON)" (INFO) - Spider close completed

**Key stats** (at close):

- ``finish_reason`` - Why the spider closed
- ``finish_time``, ``elapsed_time_seconds``

**Signals**:

- :signal:`spider_idle` - Spider has no pending work
- :signal:`spider_closed` - Spider has closed

.. _troubleshooting-engine-status:

Advanced: inspecting engine status
==================================

For deeper inspection, the engine status provides real-time information about
internal queues and component states.

The :func:`~scrapy.utils.engine.get_engine_status` function returns metrics
including:

- Active downloads count
- Scheduler queue lengths (memory and disk)
- Scraper queue length and active response count
- Whether the spider is idle
- Whether backpressure is active

To print engine status from the :ref:`Scrapy shell <topics-shell>` or within
spider code::

    from scrapy.utils.engine import print_engine_status
    print_engine_status(crawler.engine)

On Unix systems, sending SIGUSR2 to the Scrapy process triggers the
:class:`~scrapy.extensions.debug.StackTraceDump` extension, which logs engine
status along with thread stack traces. This is useful for diagnosing hangs or
investigating what the crawler is doing at a specific moment.

For memory leak investigation, the :func:`~scrapy.utils.trackref.print_live_refs`
function shows counts of tracked objects (requests, responses, items) that have
not been garbage collected.

.. seealso::

    :ref:`topics-debug`
        Techniques for debugging spider code

    :ref:`topics-leaks`
        Debugging memory leaks

    :ref:`topics-logging`
        Configuring log output

    :ref:`topics-stats`
        Working with the stats collector

    :ref:`topics-signals`
        Available signals reference

    :ref:`topics-lifecycle`
        Detailed request lifecycle documentation
