.. _topics-extensibility-guide:

===================
Extensibility Guide
===================

Scrapy provides several extension mechanisms, each designed for specific use
cases. This guide helps you choose the correct mechanism for your needs.

For implementation details, see the reference documentation for each mechanism:

* :ref:`topics-downloader-middleware`
* :ref:`topics-spider-middleware`
* :ref:`topics-item-pipeline`
* :ref:`topics-extensions`
* :ref:`topics-signals`

.. _extensibility-overview:

Overview of extension mechanisms
================================

Scrapy's extension mechanisms operate at different points in the data flow:

**Downloader middlewares** sit between the engine and the downloader. They
intercept requests before they are sent and responses after they are received.

**Spider middlewares** sit between the engine and spiders. They process
responses before spider callbacks and process items and requests after spider
callbacks.

**Item pipelines** process scraped items sequentially. Each item passes through
the pipeline chain after being yielded by a spider.

**Extensions** are general-purpose components that connect to signals. They do
not intercept data flow directly but react to events throughout the crawl
lifecycle.

**Signals** are events fired at key points in the crawl lifecycle. Any
component can connect handlers to signals.

See :ref:`topics-architecture` for how these components fit into the overall
Scrapy data flow.

.. _extensibility-decision-guide:

Quick decision guide
====================

Use this table to identify the appropriate mechanism for common tasks:

.. list-table::
   :header-rows: 1
   :widths: 50 25 25

   * - Task
     - Mechanism
     - Key method
   * - Add headers or modify outgoing requests
     - Downloader middleware
     - ``process_request``
   * - Handle retries or HTTP errors at download level
     - Downloader middleware
     - ``process_response``, ``process_exception``
   * - Implement caching or mock responses
     - Downloader middleware
     - ``process_request`` (return Response)
   * - Set up proxy rotation
     - Downloader middleware
     - ``process_request``
   * - Filter responses before spider callback
     - Spider middleware
     - ``process_spider_input``
   * - Modify or filter spider output (items/requests)
     - Spider middleware
     - ``process_spider_output``
   * - Track crawl depth or breadth
     - Spider middleware
     - ``process_spider_output``
   * - Clean or validate item data
     - Item pipeline
     - ``process_item``
   * - Deduplicate items
     - Item pipeline
     - ``process_item``
   * - Store items to a database
     - Item pipeline
     - ``process_item``, ``open_spider``, ``close_spider``
   * - Download media files for items
     - Item pipeline (MediaPipeline)
     - See :ref:`topics-media-pipeline`
   * - Collect statistics
     - Extension
     - Signal handlers
   * - Implement crawl limits or timeouts
     - Extension
     - Signal handlers
   * - Send notifications on crawl events
     - Extension
     - Signal handlers

.. _extensibility-spider-vs-downloader:

Spider middleware vs downloader middleware
==========================================

Spider and downloader middlewares both process requests and responses, but at
different points in the data flow.

When to use downloader middleware
---------------------------------

Use downloader middleware when you need to:

* **Modify HTTP requests before they are sent** - Add authentication headers,
  set cookies, configure proxies, or modify request parameters.

* **Intercept HTTP responses before spider processing** - Implement caching,
  handle HTTP-level errors (connection failures, timeouts), or transform raw
  responses.

* **Short-circuit the download** - Return a Response directly from
  ``process_request`` to skip the actual HTTP request (useful for caching or
  mocking).

* **Retry or reschedule requests** - Return a new Request from
  ``process_response`` or ``process_exception`` to reschedule.

Downloader middleware operates at the HTTP layer. It sees every request and
response regardless of which spider callback will process them.

When to use spider middleware
-----------------------------

Use spider middleware when you need to:

* **Filter or modify spider callback input** - Process responses before they
  reach the spider callback, potentially raising exceptions to prevent
  processing.

* **Filter or modify spider callback output** - Intercept items and requests
  yielded by spider callbacks, modifying or dropping them before they reach
  pipelines or the scheduler.

* **Process start requests/seeds** - Modify the initial requests yielded by
  the spider's ``start()`` method.

* **Handle spider callback exceptions** - React to exceptions raised within
  spider callbacks via ``process_spider_exception``.

Spider middleware operates at the spider layer. It has access to the spider
instance and understands the relationship between responses and spider
callbacks.

Key differences
---------------

.. list-table::
   :header-rows: 1
   :widths: 25 37 38

   * - Aspect
     - Downloader middleware
     - Spider middleware
   * - Layer
     - HTTP/network layer
     - Spider/parsing layer
   * - Request handling
     - Before download
     - After callback yields them
   * - Response handling
     - After download, before spider
     - Before and after spider callback
   * - Item handling
     - Cannot see items
     - Can filter/modify items
   * - Exception handling
     - Download exceptions
     - Spider callback exceptions

.. _extensibility-pipeline-vs-extension:

Item pipeline vs extension
==========================

Item pipelines and extensions both process scraped data, but with different
models.

When to use item pipelines
--------------------------

Use item pipelines when you need to:

* **Process every item sequentially** - Each item passes through the pipeline
  chain in order, allowing transformation, validation, or storage.

* **Maintain per-spider state** - The ``open_spider`` and ``close_spider``
  methods provide hooks to set up and tear down resources (database
  connections, file handles) tied to the spider lifecycle.

* **Drop items** - Raise ``DropItem`` to prevent an item from continuing
  through the pipeline or being exported.

* **Download media associated with items** - The built-in ``MediaPipeline``,
  ``FilesPipeline``, and ``ImagesPipeline`` provide infrastructure for
  downloading files referenced in items.

Item pipelines have a defined interface: ``process_item`` is called for every
item, giving you a guaranteed point of interception.

When to use extensions
----------------------

Use extensions when you need to:

* **React to crawl lifecycle events** - Monitor engine start/stop, spider
  open/close, or idle states via signals.

* **Cross-cutting concerns without intercepting data** - Collect statistics,
  implement limits, or trigger actions based on events rather than individual
  items.

* **Operate independently of the data flow** - Extensions do not receive items
  or requests through hook methods. They connect to signals and respond to
  events.

Extensions are "fire and observe" components. They do not transform data
passing through the system; they react to events that occur during the crawl.

Key differences
---------------

.. list-table::
   :header-rows: 1
   :widths: 25 37 38

   * - Aspect
     - Item pipeline
     - Extension
   * - Interface
     - Defined hook methods
     - Signal handlers (no predefined hooks)
   * - Item access
     - Receives every item directly
     - Observes via ``item_scraped`` signal
   * - Can drop items
     - Yes (raise ``DropItem``)
     - No (can only observe)
   * - Can transform items
     - Yes
     - No
   * - Lifecycle hooks
     - ``open_spider``, ``close_spider``
     - Any signal (engine, spider, scheduler)

Choosing between them
---------------------

Ask yourself: "Do I need to transform or drop items, or do I just need to
observe/react?"

* **Transform or drop**: Use item pipeline.
* **Observe or react**: Use extension with signals.

If you need to both observe and transform, consider whether the pipeline's
``process_item`` can handle your observation needs. Only use an extension if
you need to react to events unrelated to item processing.

.. _extensibility-signals:

Signals: when and why to use them
=================================

Signals are Scrapy's event system. They allow components to communicate without
direct coupling.

When to use signals
-------------------

Use signals when you need to:

* **Build loosely coupled extensions** - React to events without modifying core
  components or middlewares.

* **Monitor crawl progress** - Track items scraped, requests scheduled, or
  responses received without intercepting the data flow.

* **Coordinate component lifecycle** - Know when spiders open or close, when
  the engine starts or stops, or when the scheduler becomes empty.

* **Implement cross-cutting monitoring** - Statistics collection, logging,
  notifications, and similar concerns that observe but do not modify.

Signals vs hook methods
-----------------------

Middlewares and pipelines use hook methods (like ``process_request`` or
``process_item``) that intercept and can modify data. Signals notify that
something happened but do not allow modification of the event itself.

.. list-table::
   :header-rows: 1
   :widths: 25 37 38

   * - Aspect
     - Hook methods
     - Signals
   * - Purpose
     - Intercept and modify
     - Notify and observe
   * - Can modify data
     - Yes
     - No (data is read-only)
   * - Execution
     - Sequential in chain
     - All handlers called
   * - Return value matters
     - Yes (controls flow)
     - No (except Deferred/async)

Available signals
-----------------

Scrapy defines signals for key events. See :ref:`topics-signals` for the
complete list and their parameters. Key categories include:

* **Engine signals**: ``engine_started``, ``engine_stopped``
* **Spider signals**: ``spider_opened``, ``spider_closed``, ``spider_idle``,
  ``spider_error``
* **Request signals**: ``request_scheduled``, ``request_dropped``,
  ``request_reached_downloader``, ``request_left_downloader``
* **Response signals**: ``response_received``, ``response_downloaded``,
  ``headers_received``, ``bytes_received``
* **Item signals**: ``item_scraped``, ``item_dropped``, ``item_error``
* **Feed signals**: ``feed_slot_closed``, ``feed_exporter_closed``
* **Scheduler signals**: ``scheduler_empty``

.. _extensibility-anti-patterns:

Common anti-patterns
====================

Using downloader middleware for spider-level logic
--------------------------------------------------

**Anti-pattern**: Implementing item filtering or spider-specific response
handling in downloader middleware.

**Why it is problematic**: Downloader middleware operates before the spider
sees the response. It does not have access to items and cannot easily
determine which spider callback will process a response.

**Correct approach**: Use spider middleware for response filtering before
callbacks (``process_spider_input``) or for filtering/modifying items after
callbacks (``process_spider_output``).

Using extensions to modify items
--------------------------------

**Anti-pattern**: Connecting to ``item_scraped`` signal in an extension and
attempting to modify items.

**Why it is problematic**: The ``item_scraped`` signal is fired after the item
has completed pipeline processing. Modifications at this point do not affect
the item as stored or exported.

**Correct approach**: Use an item pipeline with ``process_item`` to modify
items before they complete processing.

Using item pipelines for non-item concerns
------------------------------------------

**Anti-pattern**: Implementing crawl statistics, logging, or other non-item
functionality in an item pipeline.

**Why it is problematic**: Pipelines are designed for item processing. Using
them for other purposes couples unrelated concerns and can lead to unexpected
behavior when items are dropped or when no items are scraped.

**Correct approach**: Use an extension with appropriate signals for
cross-cutting concerns unrelated to item transformation.

Duplicating functionality across mechanisms
-------------------------------------------

**Anti-pattern**: Implementing the same logic in both downloader middleware and
spider middleware, or in both pipelines and extensions.

**Why it is problematic**: Duplicated logic leads to inconsistent behavior,
maintenance burden, and potential conflicts.

**Correct approach**: Identify the correct single point of interception based
on what data you need access to and when in the lifecycle you need to act.

Blocking in signal handlers
---------------------------

**Anti-pattern**: Performing synchronous I/O (database writes, HTTP requests)
in signal handlers without using async/await or Deferreds.

**Why it is problematic**: Scrapy is asynchronous. Blocking operations in
signal handlers block the entire crawl.

**Correct approach**: Use ``async def`` for signal handlers that perform I/O,
or use Twisted's asynchronous APIs. See :ref:`signal-deferred` for details.

.. _extensibility-component-notes:

Component instantiation and lifecycle
=====================================

All extension mechanisms (middlewares, pipelines, extensions) follow the same
instantiation pattern:

1. The component class is loaded from settings.
2. If the class has a ``from_crawler`` classmethod, it is called with the
   crawler instance.
3. Otherwise, the class is instantiated directly.
4. If the constructor or ``from_crawler`` raises ``NotConfigured``, the
   component is disabled and a warning is logged.

This pattern allows components to:

* Access crawler, settings, and signals via ``from_crawler``
* Self-disable by raising ``NotConfigured`` when required settings are missing

See :ref:`topics-components` for details on the component interface.
