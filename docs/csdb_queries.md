CSDB Queries
============

The query commands that CSDB supports attempt to balance features and cross-database support.  As few commands as necessary are included in CSDB since not all databases natively support all query commands and therefore some additional work is required by the derived class to implement certain features.

When a query is run, the CSDB class automatically determines if it is a change query.  If so, the CSDB class connects to the replication master database prior to running the query.  Replication is an optional but supported feature.  When using replication, queries that make changes should be run as late as possible so that there are fewer SELECT queries run against the master database.

Depending on the database and the query, multiple SQL queries may be generated for a single command.  For example, not all databases support all index types in a CREATE TABLE statement but do support all index types via ADD INDEX.  So some CREATE TABLE commands may result in additional ADD INDEX commands to create appropriate indexes.

The goal of CSDB is nearly transparent operations from the perspective of the application.  CSDB query commands allow for a single SQL query-like set of arrays to be written and have the same operation work transparently across a wide variety of database products.

The following example is the same query run a number of different ways using a CSDB derived class in order from the least-recommended approaches to the best approach(es):

```php
<?php
	// The following is not a good approach due to:
	// - Not using parameters for the query which results in an unnecessary SQL injection risk.
	// - Since it is a raw query, it is possibly tied to a single database (e.g. MySQL).
	// - Replication won't work.  Such queries will always run against the master database.
	$row = $db->GetRow(false, "SELECT * FROM app_users WHERE id = " . (int)$id . " AND user = " . $db->Quote($_REQUEST["username"]));

	// Same as above but now all options are parameterized, so there's no risk of SQL injection.
	$row = $db->GetRow(false, "SELECT * FROM app_users WHERE id = ? AND user = ?", (int)$id, $_REQUEST["username"]);

	// Here's another way to pass parameters.  This is useful when dynamically building a query.
	$row = $db->GetRow(false, "SELECT * FROM app_users WHERE id = ? AND user = ?", array((int)$id, $_REQUEST["username"]));

	// The following query is cross-database capable.  Replication will also work as expected.
	// However, the database table reference "app_users" isn't dynamic.
	// Depending on the scenario, that might be okay but the query could be even better.
	$row = $db->GetRow("SELECT", array(
		"*",
		"FROM" => "app_users",
		"WHERE" => "id = ? AND user = ?",
	), (int)$id, $_REQUEST["username"]);

	// This is the most flexible solution since table names are now included as parameters.
	// Table names can even be variables instead of static strings.  They are also correctly quoted as identifiers.
	$db_users = "app_users";
	$row = $db->GetRow("SELECT", array(
		"*",
		"FROM" => "?",
		"WHERE" => "id = ? AND user = ?",
	), $db_users, (int)$id, $_REQUEST["username"]);
?>
```

In the last two examples, the queries get a bit more complex to write due to using PHP arrays.  However, the query is now broken up into various pieces that can be accurately analyzed and altered as necessary for a specific database.

There are "lite" versions of each class (e.g. "CSDB_mysql_lite").  These classes only support SELECT, INSERT, UPDATE, DELETE, TRUNCATE TABLE, SET, and USE commands, which results in reduced memory usage.

Transactions
------------

The CSDB class supports transactions (and nested transactions) to keep the database in proper working order.

```php
<?php
	try
	{
		$db->BeginTransaction();

		...

		$db->Commit();
	}
	catch (Exception $e)
	{
		$db->Rollback();
	}
?>
```

By default, transactions are automatically committed if the application completes successfully, but it is a good idea to just explicitly declare a commit because the application could still throw an exception.

A lot of databases have problems with consistency when it comes to transactions and schema changes.  When using transactions, stick to SELECT, INSERT, UPDATE, and DELETE query commands for maximum portability.

Subqueries
----------

Subqueries are generally SELECT commands embedded inside another command.  Some databases have really bad subquery support (e.g. poor performance).  For that reason alone, avoid subqueries and prefer joins.

Commands that accept subqueries allow for the "SUBQUERIES" option.  Strings with the format `{subquerynum}` are replaced with the associated subquery from the "SUBQUERIES" array.  An example:

```php
<?php
	$result = $db->Query("SELECT", array(
		"*",
		"FROM" => "?",
		"WHERE" => "id IN {0}",
		"ORDER BY" => "user",
		"LIMIT" => "5",
		"SUBQUERIES" => array(
			array(array(
				"id",
				"FROM" => "?",
				"WHERE" => "user = ?"
			), "app_users", "somename")
		)
	), "app_users");

	while ($row = $result->NextRow())
	{
		echo $row->id . " | " . $row->user . " | " . $row->email . " | " . date("M, j Y @ H:i", CSDB::ConvertFromDBTime($row->created)) . "\n";
	}
?>
```

In the example, the subquery `(SELECT id FROM app_users WHERE user = 'somename')` replaces the string `{0}` in the "WHERE" clause.  As can be seen above, subqueries get a bit unwieldy to work with in CSDB, which is another reason to not use them very often.

Joins, Foreign Keys, Views, Triggers, Stored Procedures
-------------------------------------------------------

When developing a cross-database friendly application, the two types of joins that are best supported by all databases are:

* Cross-join/Inner-join.
* Left outer join.

Avoid the other types of joins to maximize application portability across databases.

As far as cross-database functionality goes, CSDB currently has limited support for foreign keys.  You can create them and they will work but not all databases support them equally.  Avoid using them when writing cross-database applications.

CSDB currently has non-existent support for views, triggers, and stored procedures.  Views seem somewhat universally supported by database products in some sane format.  However, triggers and stored procedures are implemented differently by each database product and not every database supports stored procedures.  Stick to core SQL wherever possible to avoid issues with portability.

Column Definitions
------------------

The CREATE TABLE and ADD COLUMN query commands manipulate database table schemas.  One database may accept VARCHAR while another only accepts TEXT columns.  To make applications more portable, CSDB uses a limited subset of column types.

* INTEGER
* FLOAT
* DECIMAL
* STRING
* BINARY
* DATE
* TIME
* DATETIME
* BOOLEAN

The INTEGER and FLOAT types accept an optional second parameter that specifies the number of bytes of storage to use.  For example, `array("INTEGER", 2)` will result in a SMALLINT under MySQL, capable of storing 16-bit (2 byte) integers.  The INTEGER type, under some databases, also accepts the boolean option UNSIGNED.

The DECIMAL type accepts up to two extra parameters that specifies the precision and scale of a DECIMAL type.  Not all databases support this type, so the closest option is chosen.

The STRING and BINARY types accept an optional second parameter that specifies the number of bytes of storage to use.  When the value of the second parameter is 1, a required third parameter specifies the maximum number of bytes of storage for the column.  For example, `array("STRING", 1, 100)` will result in a VARCHAR(100) under MySQL.  The BINARY type is virtually identical to STRING except binary column types are used.  The STRING and BINARY types, under some databases, also accept the boolean option of FIXED, which will create a fixed-width column instead of variable-width.

The DATE, TIME, and DATETIME fields may result in an equivalent STRING type under some databases.  The CSDB date/time manipulation functions should be used, which will mitigate this issue.

The BOOLEAN type may result in an equivalent INTEGER type under many databases.

The following is a list of common options for each column:

* NOT NULL - A boolean specifying whether or not the column may not contain null values.
* DEFAULT - A boolean of false or a string containing the value to use for the default for newly inserted rows.
* PRIMARY KEY - A boolean specifying whether or not the column is the primary key.
* UNIQUE KEY - A boolean specifying whether or not the column is a unique key.
* AUTO INCREMENT - A boolean specifying whether or not the column is an auto-increment field.
* COMMENT - A string containing a comment for the column. Not all databases support this.
* REFERENCES - An array containing a foreign key reference.

If an option isn't supported, it may be emulated or may simply be ignored.

```php
<?php
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
?>
```

Index/Key Definitions
---------------------

The CREATE TABLE and ADD INDEX commands manipulate database table indexes.  If a database doesn't support a certain type of index, it will generally ignore it, which is particularly true of foreign key constraints.  To make applications more portable, CSDB uses a limited subset of index types.

* PRIMARY
* KEY
* UNIQUE
* FULLTEXT
* FOREIGN

The following is a list of common options for each index:

* NAME - A string containing a name for the index.  Should be a name that is unique across the entire database.  This option is required for some databases so it is a good idea to always include it.

Not every database supports all index types, especially FULLTEXT indexes.  For index types that aren't supported at all, the request will simply be ignored instead of throwing an error.  For index types that aren't supported in specific contexts, the class will generally attempt to emulate creating the index through other means.  For example, SQLite doesn't support KEY indexes inside of CREATE TABLE commands, but does support them via ADD INDEX, so the table is first created and then indexes are applied using multiple queries - all of which is completely transparent to the application.  However, SQLite doesn't support FULLTEXT indexes at all, so it just ignores those.

All indexes should be named with the NAME option.  Some databases require names while others don't.

Generic Database Export/Import
------------------------------

Each CSDB derived class has the the capability to export the database in a generic format for later import with any CSDB derived class.  This feature enables generic database dumps/backups as well as generic database-to-database transfers without expending a lot of effort.

Example usage for exporting a database to a file:

```php
<?php
	if (!isset($_SERVER["argc"]) || !$_SERVER["argc"])
	{
		echo "This file is intended to be run from the command-line.";

		exit();
	}

	// Temporary root.
	$rootpath = str_replace("\\", "/", dirname(__FILE__));

	require_once $rootpath . "/support/db.php";
	require_once $rootpath . "/support/db_mysql.php";

	// For databases:  dbname => true
	// For tables:  dbname.tablename => true (or false to only include the table schema)
	$exclude = array(
		"mysql" => true,
		"information_schema" => true,
		"performance_schema" => true,
		"test" => true,
	);

	try
	{
		$db = new CSDB_mysql();

		// MySQL/Maria DB.
		$db->Connect("mysql:host=127.0.0.1", "username", "*******");

		// Enable large results mode.
		$db->LargeResults(true);
	}
	catch (Exception $e)
	{
		echo "Unable to connect to the database.  " . htmlspecialchars($e->getMessage()) . "\n";
		exit();
	}

	// Get the list of databases to back up.
	$databases = array();
	$result = $db->Query("SHOW DATABASES", array());
	while ($row = $result->NextRow())
	{
		if (!isset($exclude[$row->name]))  $databases[] = $row->name;
	}

	$fp = fopen("dump.txt", "wb");

	function WriteLine($info)
	{
		global $fp;

		fwrite($fp, json_encode($info, JSON_UNESCAPED_SLASHES) . "\n");
	}

	foreach ($databases as $dbname)
	{
		// Generate a DROP DATABASE statement.
		WriteLine(array("cmd" => "DROP DATABASE", "opts" => array($dbname)));

		// Generate a CREATE DATABASE statement.
		WriteLine($db->GetRow("SHOW CREATE DATABASE", array($dbname)));

		// Switch to the database.
		WriteLine(array("cmd" => "USE", "opts" => $dbname));

		$db->Query("USE", $dbname);

		// Get the list of tables in the database to backup.
		$tables = array();
		$result = $db->Query("SHOW TABLES", array("FULL" => true));
		while ($row = $result->NextRow())
		{
			if (!isset($exclude[$dbname . "." . $row->name]) || $exclude[$dbname . "." . $row->name])  $tables[] = $row->name;
		}

		foreach ($tables as $tablename)
		{
			// Generate a DROP TABLE statement.
			WriteLine(array("cmd" => "DROP TABLE", "opts" => array($tablename)));

			// Generate a CREATE TABLE statement.
			WriteLine($db->GetRow("SHOW CREATE TABLE", array($tablename)));

			if ($exclude[$dbname . "." . $row->name])
			{
				// Dump table data.
				$result = $db->Query("SELECT", array(
					"*",
					"FROM" => "?",
					"EXPORT ROWS" => $tablename
				), $tablename);

				while ($row = $result->NextRow())
				{
					WriteLine(array("cmd" => "INSERT", "opts" => $row));
				}
			}
		}
	}

	fclose($fp);
?>
```

Example usage for importing a database from a file:

```php
<?php
	if (!isset($_SERVER["argc"]) || !$_SERVER["argc"])
	{
		echo "This file is intended to be run from the command-line.";

		exit();
	}

	// Temporary root.
	$rootpath = str_replace("\\", "/", dirname(__FILE__));

	require_once $rootpath . "/support/db.php";
	require_once $rootpath . "/support/db_sqlite.php";

	try
	{
		$db = new CSDB_sqlite();

		// SQLite.
		$db->Connect("sqlite:/var/path/to/sqlite.db");

		// Enable bulk import mode.
		$db->Query("BULK IMPORT MODE", true);
	}
	catch (Exception $e)
	{
		echo "Unable to connect to the database.  " . htmlspecialchars($e->getMessage()) . "\n";
		exit();
	}

	$fp = fopen("dump.txt", "rb");

	while (($line = fgets($fp)) !== false)
	{
		$info = @json_decode(trim($line), true);
		if (is_array($info) && isset($info["cmd"] && isset($info["opts"]))
		{
			$db->Query($info["cmd"], $info["opts"]);
		}
	}

	fclose($fp);

	// Disable bulk import mode.
	$db->Query("BULK IMPORT MODE", true);
?>
```

SELECT Command
--------------

This command selects rows from one or more database tables or the results of a database function.  Many databases perform poorly when doing joins across _databases_, so support for multi-database joins is limited and not recommended.  For improved portability, applications should also attempt to avoid using database-specific extensions and SQL functions and instead push such logic back to the application.  Keeping queries really generic goes a long way to avoiding problems later on.

Example usage:

```php
<?php
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
?>
```

The second parameter is an array containing:

* A string or an array of column names to return.
* An optional string containing the "FROM" clause to use.  Supports parameters.  Prefer to use the "FROM" option instead.
* Zero or more options.

The following options are commonly available:

* DISTINCT - A boolean indicating whether or not to run a DISTINCT SQL query.
* FROM - A string containing the FROM clause.  Supports parameters and subqueries.  Exists primarily for improving code readability.  Using parameters for table names in the string is recommended.
* WHERE - A string containing the WHERE clause.  Supports parameters and subqueries.
* GROUP BY - A string containing the GROUP BY clause.  Supports parameters.
* HAVING - A string containing the HAVING clause.  Supports parameters.
* ORDER BY - A string containing the ORDER BY clause.  Supports parameters.
* LIMIT - An array (preferred) or string or integer containing the LIMIT clause.  Supports parameters.  Not all databases support LIMIT (see notes).
* EXPORT ROWS - A string containing the table name to use.

DISTINCT is broken out specifically to accommodate SELECT prefixes that some databases support.  Using the option is more portable.

When the FROM clause contains parameters, the parameters are replaced directly with the next argument on the stack after calling the database class' `QuoteIdentifier()` function.  This has the natural side-effect of eliminating the ability to select an alternate database.  For the most portable code, use the USE command to select a database and then run queries only on tables in that database with parameters for table names.

Not all databases support the LIMIT keyword.  For those databases, LIMIT is emulated via row filtering.  LIMIT is also ignored when used inside of subqueries.

The EXPORT ROWS option returns rows in a format that are ready for use with the INSERT command.  This options allows for being able to create a general-purpose backup solution for any database product or quickly migrate data from one database to another.

INSERT Command
--------------

This command inserts a single row into a database table.  Multiple rows can be inserted with the "SELECT" option.  Inserting rows one at a time can be a lot slower over bulk inserts, but not every database supports multiple row inserts in a single command.  However, some databases optimize transactions such that a bunch of inserts in a single transaction are collected together and then run simultaneously when the transaction is committed, which results in extremely fast performance (e.g. SQLite does this and inserting a million rows takes under a minute to complete within a transaction).

Note:  The command is INSERT.  Using "INSERT INTO" will result in an exception being raised.

Example usage:

```php
<?php
	$db->Query("INSERT", array("app_users", array(
		"user" => $_REQUEST["username"],
		"email" => $_REQUEST["email"],
		"created" => CSDB::ConvertToDBTime(time()),
		"updated" => CSDB::ConvertToDBTime(time()),
	), "AUTO INCREMENT" => "id"));

	$id = $db->GetInsertID();
?>
```

When not using SELECT (a normal INSERT query), the second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* An array containing key-value pairs.  The keys (strings) will be quoted as identifiers.  The values will become parameters.
* An optional array containing key-value pairs.  The keys (strings) will be quoted as identifiers.  The values will be inlined with the query.  Not recommended for use.
* AUTO INCREMENT - An optional mapping to the column in the table that represents an auto-increment field.  Some databases require this for `GetInsertID()` to function properly (e.g. PostgreSQL).

When using SELECT, the second parameter is an array containing as follows:

* A string containing a table name. This will be quoted as an identifier.
* An optional array containing a list of keys. The keys (strings) will be quoted as identifiers.
* A SELECT option containing an array of information to pass to the subquery processor.

For an example of using SELECT with INSERT, see the DROP COLUMN command implementation in 'support/db_sqlite.php'.  SQLite doesn't support dropping columns directly, so the entire table has to be recreated twice.

When a row is inserted into the database and a primary key column is an integer with an auto-increment attribute, most databases support a method of extracting the last inserted ID.  This is exposed via the `GetInsertID()` member function.

UPDATE Command
--------------

This command updates specific rows in a database table.  Most databases implement this command consistently.

Example usage:

```php
<?php
	$db->Query("UPDATE", array("app_users", array(
		"email" => $_REQUEST["email"],
		"updated" => CSDB::ConvertToDBTime(time()),
	), "WHERE" => "id = ?"), $id);
?>
```

The second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* An array containing key-value pairs.  The keys (strings) will be quoted as identifiers.  The non-boolean values will become parameters.  Boolean values of true become DEFAULT.  Boolean values of false become NULL.
* An optional array containing key-value pairs.  The keys (strings) will be quoted as identifiers.  The values will be inlined with the query.  Not recommended for use beyond incrementing/decrementing values in a column.
* Zero or more options.

The following options are commonly available:

* WHERE - A string containing the WHERE clause.  Supports parameters and subqueries.
* ORDER BY - A string containing the ORDER BY clause.  Supports parameters.
* LIMIT - An array (preferred) or string containing the LIMIT clause.  Supports parameters.  Not all databases support LIMIT.

The LIMIT keyword is not emulated for UPDATE queries when a database doesn't support the option.  This happens because UPDATE doesn't return rows to the application.

CSDB will catch certain accidental issues and throw an exception.  For example, typing `"WHERE => id = ?"` instead of `"WHERE" => "id = ?"` will trigger the exception.  The former would unintentionally modify all rows in the table while the latter only modifies one row.

DELETE Command
--------------

This command deletes specific rows from a database table.  Most databases implement this SQL command consistently.

Example usage:

```php
<?php
	$db->Query("DELETE", array("app_users", "WHERE" => "id = ?"), $id);
?>
```

The second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* An array containing key-value pairs.  The keys (strings) will be quoted as identifiers.  The values will become parameters.
* An optional array containing key-value pairs.  The keys (strings) will be quoted as identifiers.  The values will be inlined with the query.  Not recommended for use.
* Zero or more options.

The following options are commonly available:

* WHERE - A string containing the WHERE clause.  Supports parameters and subqueries.
* ORDER BY - A string containing the ORDER BY clause.  Supports parameters.
* LIMIT - An array (preferred) or string containing the LIMIT clause.  Supports parameters.  Not all databases support LIMIT.

The LIMIT keyword is not emulated for DELETE queries when a database doesn't support the option.  This happens because DELETE doesn't return rows to the application.

CSDB will catch certain accidental issues and throw an exception.  For example, typing `"WHERE => id = ?"` instead of `"WHERE" => "id = ?"` will trigger the exception.  The former would unintentionally delete all rows in the table while the latter only deletes one row.

TRUNCATE TABLE Command
----------------------

This command deletes all rows in a database table.  In some database products, this will also reset an auto-increment field back to its original value.

Example usage:

```php
<?php
	$db->Query("TRUNCATE TABLE", array("app_users"));
?>
```

The second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.

When a database doesn't support this command natively, it is emulated using DELETE.  Some database products require extra user permissions to execute this command.

SET Command
-----------

This command sets a session or database option.  Options are usually not portable between database products and this command exists primarily for internal class use.

Example usage:

```php
<?php
	$db->Query("SET", "NAMES 'utf8'");
?>
```

The second parameter is a string that will NOT be escaped or quoted.  Strings should be hardcoded so there isn't any possibility of user input causing an application vulnerability.  This is an unusual command since most commands take an array as the second parameter.

USE Command
-----------

This command selects a database.  If the database product does not support multiple standard databases in a single session, the class will emulate this command, typically as a string prefix.

Example usage:

```php
<?php
	$db->Query("USE", "testdb");
?>
```

The second parameter is a string that will be quoted as an identifier.  This is an unusual command since most commands take an array as the second parameter.

CREATE DATABASE Command
-----------------------

This command creates a database.  If the database product does not support multiple standard databases in a single session, the class will either emulate this command or ignore the request.

Example usage:

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
?>
```

The second parameter is an array containing:

* A string containing a database name.  This will be quoted as an identifier.
* Zero or more options.

The following options are commonly available:

* CHARACTER SET - A string containing a character set to use.  Modern applications should use a UTF-8 variant and classes will automatically select this if not specified.
* COLLATE - A string containing a collation rule to use.

Note that not all databases support database-level options.  Also, specifying these options will make the application less portable.

Many database products require extra user permissions to execute this command.

DROP DATABASE Command
---------------------

This command drops a database and all tables within the database.

Example usage:

```php
<?php
	$db->Query("DROP DATABASE", array("testdb"));
?>
```

The second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.

If the database product does not support multiple standard databases in a single session, the class will emulate this command.  Note that emulation may have unexpected results.  Of course, dropping a whole database should be done carefully to begin with.

Many database products require extra user permissions to execute this command.

CREATE TABLE Command
--------------------

This command creates a new table in a database and supports normal tables, temporary tables, and CREATE TABLE AS SELECT (CTAS) tables.  Table schema declarations vary widely across database products.  CSDB normalizes schema declarations by supporting a fairly common subset of what most databases support.

Example usage:

```php
<?php
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
?>
```

When creating a normal or temporary table, the second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* An array containing column definitions.
* An optional array containing index/key definitions.
* Zero or more options.

When using CTAS, the second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* A SELECT option.
* Zero or more options.

Not every database supports CTAS.  In addition, many database products will drop indexes/keys in CTAS tables.

The following options are commonly available:

* TEMPORARY - A boolean indicating that the table to be created should be temporary.
* CHARACTER SET - A string containing a character set to use.  Modern applications should use a UTF-8 variant and classes will automatically select this if not specified.
* COLLATE - A string containing a collation rule to use.

Note that not all databases support these table-level options.  Also, specifying these options will make the application less portable.  For example, one database might accept a character set string of 'utf-8' while another accepts 'utf8'.

Many database products require extra user permissions to execute this command.

ADD COLUMN Command
------------------

This command adds a column to an existing database table.

Example usage:

```php
<?php
	$db->Query("ADD COLUMN", array("app_users", "updates", array("INTEGER", 8, "NOT NULL" => true), "AFTER" => "updated"));
?>
```

The second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* A string containing a column name.  This will be quoted as an identifier.
* An array containing the column definition.
* Zero or more options.

The following options are commonly available:

* FIRST - A boolean indicating the column should become the first column.
* AFTER - A string indicating the column before the new column.

Not all databases support all options.  If a database doesn't support an option, it is ignored.

Many database products require extra user permissions to execute this command.

DROP COLUMN Command
-------------------

This command drops/removes a column from an existing database table.  If the column to be dropped is the only column in the table, the table may be removed or an error thrown depending on the database.

Example usage:

```php
<?php
	$db->Query("DROP COLUMN", array("app_users", "updates"));
?>
```

The second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* A string containing a column name.  This will be quoted as an identifier.

If the database doesn't support removing a column, it will be emulated by recreating the table twice, which may have side-effects.  For example, SQLite doesn't have support for dropping columns, so the class emulates the process by recreating the entire table while attempting to retain column type information and indexes/keys but may not successfully preserve all information.

Many database products require extra user permissions to execute this command.

ADD INDEX Command
-----------------

This command adds an index/key to an existing database table.

Example usage:

```php
<?php
	$db->Query("ADD INDEX", array("app_users",
		array("KEY", array("updated"), "NAME" => "app_users_updated")
	));
?>
```

The second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* An array containing the index/key definition.

If a database doesn't support an index type it will either emit an error or silently ignore the request.

Many database products require extra user permissions to execute this command.

DROP INDEX Command
------------------

This command drops/removes an index/key from an existing database table.

Example usage:

```php
<?php
	$db->Query("DROP INDEX", array("app_users", "KEY", "app_users_updated"));
?>
```

The second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* A string containing the index/key type.
* A string containing the index/key name.

If a database doesn't support an index type it will either emit an error or silently ignore the request.

Many database products require extra user permissions to execute this command.

DROP TABLE Command
------------------

This command drops/removes an existing table from the database.

Example usage:

```php
<?php
	$db->Query("DROP TABLE", array("app_users"));
?>
```

The second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* Zero or more options.

The following option is commonly available:

* FROM - A string containing a database name.  This will be quoted as an identifier.

Not all databases support all options.  If a database doesn't support an option, it is ignored.

Many database products require extra user permissions to execute this command.

SHOW DATABASES Command
----------------------

This command shows all available databases that the user has access to.

Example usage:

```php
<?php
	$result = $db->Query("SHOW DATABASES", array());

	while ($row = $result->NextRow())
	{
		var_dump($row);
	}
?>
```

The second parameter is an array containing:

* An optional string containing the text to match in each database name.

When the string is specified, only database names that contain the string will be returned.

Each returned row contains a column called "name".

SHOW TABLES Command
-------------------

This command shows all available tables in a database.

Example usage:

```php
<?php
	$result = $db->Query("SHOW TABLES", array("FULL" => true));

	while ($row = $result->NextRow())
	{
		var_dump($row);
	}
?>
```

The second parameter is an array containing:

* An optional string containing the text to match in each table name.
* Zero or more options.

The following options are commonly available:

* FROM - A string containing a database name.  This will be quoted as an identifier.
* FULL - A boolean that requests additional information, if available, be returned with each row.

Not all databases support all options.  If a database doesn't support an option, it is ignored.

Each returned row contains a column called "name".

SHOW CREATE DATABASE Command
----------------------------

This command obtains a CREATE DATABASE compatible response for the specified database.

Example usage:

```php
<?php
	$row = $db->GetRow("SHOW CREATE DATABASE", array("testdb"));

	var_dump($row);
?>
```

The second parameter is an array containing:

* A string containing a database name.  This will be quoted as an identifier.

Each returned row contains a column called "cmd" containing the string "CREATE DATABASE" and a column called "opts" containing an array of values suitable for passing as the second parameter in a future CREATE DATABASE command.

SHOW CREATE TABLE Command
-------------------------

This command obtains a CREATE TABLE compatible response for the specified table.

Example usage:

```php
<?php
	$row = $db->GetRow("SHOW CREATE TABLE", array("app_users"));

	var_dump($row);
?>
```

The second parameter is an array containing:

* A string containing a table name.  This will be quoted as an identifier.
* Zero or more options.

The following option is commonly available:

* EXPORT HINTS - An array containing column definitions to use for specific columns.

Not all databases support all options.  If a database doesn't support an option, it is ignored.

Each returned row contains a column called "cmd" containing the string "CREATE TABLE" and a column called "opts" containing an array of values suitable for passing as the second parameter in a future CREATE TABLE command.

BULK IMPORT MODE Command
------------------------

This command enables or disables features for the current session for handling bulk imports of data.  This feature is primarily used as a precursor to initialize one or more offline database tables with large quantities of data.  For most databases, enabling bulk import mode results in a series of queries that disable various safety checks that are considered fairly dangerous for maintaining data integrity.  The tradeoff, however, is a very significant increase in INSERT performance on a scale of hundreds to thousands of times faster.

Example usage:

```php
<?php
	// Enable.
	$db->Query("BULK IMPORT MODE", true);

	// Perform a LOT of INSERT queries here...

	// Disable.
	$db->Query("BULK IMPORT MODE", false);
?>
```

The second parameter is a boolean that will enable or disable bulk import mode.

Note that it is very important to disable bulk import mode prior to disconnecting.  The CSDB class will not do this for you.  Disabling bulk import mode gives the database an opportunity to do cleanup work to put the database into a consistent state (e.g. refresh indexes).
