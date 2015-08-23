CSDB Data Access Layer
======================

A PHP library that provides a portable, cross-platform, cross-database, lightweight, debuggable, replication-aware, migration-friendly, transaction-capable, data access layer (DAL) for PHP 5.3 and later.

Features
--------

* Cross-platform.  Runs under PHP 5.3 and later for all OSes that PHP supports.
* Cross-database capable.  The same application code can generate two completely different queries for two different database products but achieve the same result.  Quite useful for writing and maintaining open source software products that talk to a database.
* Lightweight.  The base class plus one derived class instantiation plus an active database connection only uses an additional 300KB to 570KB RAM (depends on the database).
* Debuggable.  Easily see the output of each query, its parameter values, query number, how long the query took, and total time for all queries for the current connection.
* Replication-aware.  Automatically switches to a replication master when running change queries (INSERT, UPDATE, DELETE, etc).
* Migration-friendly.  The full versions of each database class allow for quickly migrating both tables and data from one database product to another.
* Nested transaction support.  Allows for nested BeginTransaction() and Commit() calls.
* Has a liberal open source license.  MIT or LGPL, your choice.
* Designed for relatively painless integration into your project.
* Sits on GitHub for all of that pull request and issue tracker goodness to easily submit changes and ideas respectively.

More Information
----------------

Documentation, examples, and official downloads of this project sit on the Barebones CMS website:

http://barebonescms.com/documentation/csdb/
