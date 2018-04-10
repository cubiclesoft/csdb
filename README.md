CSDB Data Access Layer
======================

A PHP library that provides a portable, cross-platform, cross-database, lightweight, debuggable, replication-aware, migration-friendly, transaction-capable, [data access layer](http://en.wikipedia.org/wiki/Data_access_layer) (DAL) for PHP.

A collection of useful, lightweight classes intended to be dropped into projects that need database access.  CSDB wraps the PHP [PDO](http://www.php.net/manual/en/book.pdo.php) layer with useful functionality for cross-database web application development.  Minimal dependencies allows for only including the files needed for a project.

Features
--------

* Cross-platform.  Runs under PHP 5.3 and later for all OSes that PHP supports.
* Cross-database capable.  The same application code can generate two completely different queries for two different database products but achieve the same result.  Quite useful for writing and maintaining open source software products that talk to a database.
* Lightweight.  The base class plus one derived class instantiation plus an active database connection only uses an additional 300KB to 570KB RAM (depends on the database).
* Debuggable.  Easily see the output of each query, its parameter values, query number, how long the query took, and total time for all queries for the current connection.
* Replication-aware.  Automatically switches to a replication master when running change queries (INSERT, UPDATE, DELETE, etc).
* Migration-friendly.  The full versions of each database class allow for quickly migrating both tables and data from one database product to another.
* Nested transaction support.  Allows for nested BeginTransaction() and Commit() calls.
* Full support for:  MySQL/Maria DB, PostgreSQL, SQLite.
* Beta support for:  Oracle (OCI).
* Has a liberal open source license.  MIT or LGPL, your choice.
* Designed for relatively painless integration into your project.
* Sits on GitHub for all of that pull request and issue tracker goodness to easily submit changes and ideas respectively.

Basic Usage
-----------

Connecting to a database:

```php
<?php
	// Replace '_mysql' references below with the correct database class and use the relevant Connect() call.
	require_once "support/db.php";
	require_once "support/db_mysql.php";

	try
	{
		$db = new CSDB_mysql();

		// Enable debug mode for testing only.
//		$db->SetDebug(fopen("out.txt", "wb"));

		// MySQL/Maria DB.
		$db->Connect("mysql:host=127.0.0.1", "username", "*******");
//		$db->SetMaster("mysql:host=remotehost", "username", "*******");

		// Postgres.
		$db->Connect("pgsql:host=127.0.0.1", "username", "*******");
//		$db->SetMaster("pgsql:host=remotehost", "username", "*******");

		// SQLite.
		$db->Connect("sqlite:/var/path/to/sqlite.db");

		// Oracle.
		$db->Connect("oci:dbname=//localhost/ORCL", "username", "*******");
//		$db->SetMaster("oci:dbname=//remotehost/ORCL", "username", "*******");

		// Assumes 'testdb' exists.
		$db->Query("USE", "testdb");
	}
	catch (Exception $e)
	{
		echo "Unable to connect to the database.  " . htmlspecialchars($e->getMessage()) . "\n";
		exit();
	}
?>
```

Standard queries:

```php
<?php
	try
	{
		// SELECT multiple rows.
		$result = $db->Query("SELECT", array(
			"*",
			"FROM" => "?",
			"WHERE" => "created >= ?",
			"ORDER BY" => "id DESC",
			"LIMIT" => 100
		), "app_users", CSDB::ConvertToDBTime(time() - 3 * 24 * 60 * 60));

		while ($row = $result->NextRow())
		{
			echo $row->id . " | " . $row->user . " | " . $row->email . " | " . date("M, j Y @ H:i", CSDB::ConvertFromDBTime($row->created)) . "\n";
		}

		// SELECT one row.
		$row = $db->GetRow("SELECT", array(
			"*",
			"FROM" => "?",
			"WHERE" => "user = ?",
		), "app_users", $_REQUEST["username"]);

		// SELECT the first column of the first row.
		$row = (int)$db->GetOne("SELECT", array(
			"COUNT(*) AS c",
			"FROM" => "?",
		), "app_users");

		// Transaction support.
		$db->BeginTransaction();

		// INSERT row.
		$db->Query("INSERT", array("app_users", array(
			"user" => $_REQUEST["username"],
			"email" => $_REQUEST["email"],
			"created" => CSDB::ConvertToDBTime(time()),
			"updated" => CSDB::ConvertToDBTime(time()),
		), "AUTO INCREMENT" => "id"));

		$id = $db->GetInsertID();

		// UPDATE row.
		$db->Query("UPDATE", array("app_users", array(
			"email" => $_REQUEST["email"],
			"updated" => CSDB::ConvertToDBTime(time()),
		), "WHERE" => "id = ?"), $id);

		// DELETE row.
		$db->Query("DELETE", array("app_users", "WHERE" => "id = ?"), $id);

		// Finalize transaction.
		$db->Commit();
	}
	catch (Exception $e)
	{
		// Rollback transaction.
		$db->Rollback();

		echo "An error occurred while running a database query.  " . htmlspecialchars($e->getMessage()) . "\n";
		exit();
	}
?>
```

Creating a database and tables (e.g. for an installer):

```php
<?php
	// Create/Use the database.
	try
	{
		$db->Query("USE", "testdb");
	}
	catch (Exception $e)
	{
		try
		{
			$db->Query("CREATE DATABASE", array("testdb", "CHARACTER SET" => "utf8", "COLLATE" => "utf8_general_ci"));
			$db->Query("USE", "testdb");
		}
		catch (Exception $e)
		{
			echo "Unable to create/use database 'testdb'.  " . htmlspecialchars($e->getMessage()) . "\n";
			exit();
		}
	}

	// Create missing database tables.
	try
	{
		$appusersfound = $db->TableExists("app_users");
	}
	catch (Exception $e)
	{
		echo "Unable to determine the existence of a database table.  " . htmlspecialchars($e->getMessage()) . "\n";
		exit();
	}

	if (!$appusersfound)
	{
		try
		{
			$db->Query("CREATE TABLE", array("app_users", array(
				"id" => array("INTEGER", 8, "UNSIGNED" => true, "NOT NULL" => true, "PRIMARY KEY" => true, "AUTO INCREMENT" => true),
				"user" => array("STRING", 1, 255, "NOT NULL" => true),
				"email" => array("STRING", 1, 255, "NOT NULL" => true),
				"created" => array("DATETIME", "NOT NULL" => true),
				"updated" => array("DATETIME", "NOT NULL" => true),
			),
			array(
				array("UNIQUE", array("user"), "NAME" => "app_users_user"),
				array("UNIQUE", array("email"), "NAME" => "app_users_email"),
				array("KEY", array("created"), "NAME" => "app_users_created"),
			)));
		}
		catch (Exception $e)
		{
			echo "Unable to create the database table 'app_users'.  " . htmlspecialchars($e->getMessage()) . "\n";
			exit();
		}
	}
?>
```

Remember that PHP function and method calls are case-insensitive.  So it's fine if you prefer `connect()`, `query()`, `nextRow()`, `getRow()`, etc.  Just be consistent.

More Information
----------------

Full documentation and examples can be found in the 'docs' directory of this repository.

* [Queries](https://github.com/cubiclesoft/csdb/blob/master/docs/csdb_queries.md) - Documentation for all query types including SELECT, INSERT, UPDATE, DELETE, CREATE TABLE, etc.
* [CSDB class](https://github.com/cubiclesoft/csdb/blob/master/docs/csdb.md) - Documentation for the CSDB base class.
