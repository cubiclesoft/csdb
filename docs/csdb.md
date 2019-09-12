CSDB Class:  'support/db.php'
=============================

The CSDB class is the base class which database-specific classes (e.g. CSDB_mysql) extend.  This class provides a lot common of helper code to transform the input arrays into a final set of strings and parameters for executing parameterized queries and handling results that will be passed back to the application.

CSDB is primarily a wrapper around PDO providing additional useful functionality and generalizing SQL.

For example usage, see the [CSDB repository](https://github.com/cubiclesoft/csdb).

Protected Variables
-------------------

The following protected variables are available to derived classes:

* $numqueries - An integer containing the number of queries that have been executed.
* $totaltime - An integer containing the total amount of time spent executing queries.
* $dbobj - An object pointing to the current database object.
* $origdbobj - An object pointing to the previous database object.  Usually assigned when a master database is connected to.
* $debug - A boolean that indicates whether or not to also include debug information when encountering errors OR a resource pointing to a file to write all request information.
* $transaction - An integer indicating the number of transactions applied to the current database.  If a switch to a master database occurs in the middle of a transaction, any active transactions are committed and the same number of transactions are begun on the new connection.
* $currdb - The current database as set by a 'USE' statement.

CSDB::ConvertToDBDate($ts, $gmt = true)
---------------------------------------

Access:  public static

Parameters:

* $ts - An integer representing a UNIX timestamp.
* $gmt - A boolean specifying whether or not to return a GMT/UTC string.

Returns:  The result of `gmdate()` or `date()` depending on `$gmt`.

This static function converts a UNIX timestamp into a string suitable for a date field in a database.  Strings take the international format 'YYYY-MM-DD'.

CSDB::ConvertToDBTime($ts, $gmt = true)
---------------------------------------

Access:  public static

Parameters:

* $ts - An integer representing a UNIX timestamp.
* $gmt - A boolean specifying whether or not to return a GMT/UTC string.

Returns:  The result of `gmdate()` or `date()` depending on `$gmt`.

This static function converts a UNIX timestamp into a string suitable for a date and time field in a database.  Strings take the international format 'YYYY-MM-DD HH:MM:SS'.

CSDB::ConvertFromDBTime($field, $gmt = true)
--------------------------------------------

Access:  public static

Parameters:

* $field - A string containing a date and time in international format.
* $gmt - A boolean specifying whether or not to process `$field` as a GMT/UTC string.

Returns:  The result of `gmmktime()` or `mktime()` depending on `$gmt`.

This static function converts a date and time field string from a database in international format into an integer representing a UNIX timestamp.  If there is no time portion, '00:00:00' is assumed.

CSDB::IsAvailable()
-------------------

Access:  public

Parameters:  None.

Returns:  A boolean of false.

This function returns a boolean or a string (PDO classes only) indicating whether or not PHP supports the class.  The base class returns false.  Derived classes are expected to override this function and return a boolean of true OR a PDO driver string (e.g. 'sqlite') if the class will function properly under PHP on the host.

CSDB::GetDisplayName()
----------------------

Access:  public

Parameters:  None.

Returns:  An empty string.

This function returns a string containing the name to display to the user (e.g. for use in a dropdown to select the class).  The base class returns an empty string.  Derived classes are expected to override this function and return a displayable string (e.g. `CSDB::DB_Translate("SQLite (via PDO)")`).

CSDB::__construct($dsn = false, $username = false, $password = false, $options = array())
-----------------------------------------------------------------------------------------

Access:  public

Parameters:

* $dsn - A boolean of false or a string containing a valid DSN to connect with (Default is false).
* $username - A boolean of false or a string containing the username to connect with (Default is false).
* $password - A boolean of false or a string containing the password to connect with (Default is false).
* $options - An array containing additional connection options (Default is array()).

Returns:  Nothing.

This function initializes the class instance.  When a DSN is supplied, a `Connect()` call is executed with that DSN to save a step.

CSDB::SetDebug($debug)
----------------------

Access:  public

Parameters:

* $debug - A boolean that indicates whether or not to also include debug information when encountering errors OR a resource pointing to a file to write all request information.

Returns:  Nothing.

This function can be used to write detailed information to a file or include query details when encountering errors.  The default debug status is to disable debugging information as doing so can expose internal database structures, potentially allowing an attacker to gain access to information that they shouldn't have access to.

CSDB::AssertPDOAvailable($checkdb)
----------------------------------

Access:  protected

Parameters:

* $checkdb - A boolean that indicates whether or not the assertion should include verifing a database connection likely exists.

Returns:  Nothing.

This protected function will throw an exception if `IsAvailable()` did not return a string (i.e. not using PDO under the hood) or if `$checkdb` is true and there is no active connection to a database.

CSDB::BeginTransaction()
------------------------

Access:  public

Parameters:  None.

Returns:  Nothing.

This function begins a transaction.  If a transaction is already started, the counter is only incremented.

CSDB::NumTransactions()
-----------------------

Access:  public

Parameters:  None.

Returns:  The current transaction depth.

This function returns the value of the internal counter of started transactions.

CSDB::Commit()
--------------

Access:  public

Parameters:  None.

Returns:  Nothing.

This function commits a transaction.  The transaction counter is only decremented if it is greater than 1.

CSDB::Rollback()
----------------

Access:  public

Parameters:  None.

Returns:  Nothing.

This function rolls back a transaction.  The transaction counter is set to 0 even if multiple `BeginTransaction()` calls have been made.

CSDB::Connect($dsn, $username = false, $password = false, $options = array())
-----------------------------------------------------------------------------

Access:  public

Parameters:

* $dsn - A string containing a valid DSN to connect with.
* $username - A boolean of false or a string containing the username to connect with (Default is false).
* $password - A boolean of false or a string containing the password to connect with (Default is false).
* $options - An array containing additional connection options (Default is array()).

Returns:  A boolean of true.  Throws an exception on error.

This function connects to a database as specified by the DSN.  Note that the correct driver as specified in the DSN matches the derived class PDO driver.

CSDB::SetMaster($dsn, $username = false, $password = false, $options = array())
-------------------------------------------------------------------------------

Access:  public

Parameters:

* $dsn - A string containing a valid DSN to connect with.
* $username - A boolean of false or a string containing the username to connect with (Default is false).
* $password - A boolean of false or a string containing the password to connect with (Default is false).
* $options - An array containing additional connection options (Default is array()).

Returns:  Nothing.

This function sets a master DSN, username, password, and options for connecting to a master database in a replication-aware environment (e.g. MySQL/Maria DB database replication).  When a change query such as an INSERT is run, the connection to master database is transparently made before executing the query.  This function doesn't actually do anything but store information for later use so it isn't possible to determine that things such as credentials are correct or not.

When using replication, queries that make changes should be run as late as possible so that there are fewer SELECT queries run against the master database.  When connecting to the replication master, the same database as the slave is connected to (if the 'USE' command was used) and the same number of transactions (if any) that began on the slave are restored.

CSDB::Disconnect()
------------------

Access:  public

Parameters:  None.

Returns:  A boolean of true.

This function attempts to commit all transactions and disconnect from one or two databases.  There can be two database objects when using a replication master.  Note that an exception can still happen here at the PDO level.

Disconnecting from a database requires all references to the underlying database objects to vanish.  Disconnecting won't necessarily work as expected.  Ending the current PHP session is really the only way to guarantee that the database object is freed up.  This behavior has more to do with how PDO works than anything else.  However, most PHP scripts that use a database hurry to set up the database connection at the beginning of the script and then don't terminate the database connection until the very end, so letting PHP handle the teardown of the connection naturally at the conclusion of the script is a decent solution for most applications instead of feeling obligated to call this function.

CSDB::Query(...)
----------------

Access:  public

Parameters:  Varied.

Returns:  An instance of `CSDB_PDO_Statement`.  Throws an exception on error.

This function prepares and executes one or more parameterized queries.  This is the only function that can be used when `LargeResults()` has been called to enable large results.

For details on running queries, see the [Queries documentation](https://github.com/cubiclesoft/csdb/blob/master/docs/csdb_queries.md).

CSDB::GetRow(...)
-----------------

Access:  public

Parameters:  Varied.

Returns:  An object containing the first row of results or false if no rows were returned.  Throws an exception on error.

This function prepares and executes one or more parameterized queries.  The first row is retrieved, if any, and returned to the caller.

For details on running queries, see the [Queries documentation](https://github.com/cubiclesoft/csdb/blob/master/docs/csdb_queries.md).

CSDB::GetRowArray(...)
----------------------

Access:  public

Parameters:  Varied.

Returns:  An array containing the first row of results or false if no rows were returned.  Throws an exception on error.

This function prepares and executes one or more parameterized queries.  The first row is retrieved, if any, and returned to the caller as an array instead of an object.

For details on running queries, see the [Queries documentation](https://github.com/cubiclesoft/csdb/blob/master/docs/csdb_queries.md).

CSDB::GetCol(...)
-----------------

Access:  public

Parameters:  Varied.

Returns:  An array containing the first column of each row of results.  Throws an exception on error.

This function prepares and executes one or more parameterized queries.  The first column of results is retrieved and returned to the caller.

For details on running queries, see the [Queries documentation](https://github.com/cubiclesoft/csdb/blob/master/docs/csdb_queries.md).

CSDB::GetOne(...)
-----------------

Access:  public

Parameters:  Varied.

Returns:  The first column of the first row of results or false if no rows were returned, false otherwise.  Throws an exception on error.

This function prepares and executes one or more parameterized queries.  The first column of the first row of the results, if any, is retrieved and returned to the caller.  This function is useful for SELECT COUNT(*) style queries.

For details on running queries, see the [Queries documentation](https://github.com/cubiclesoft/csdb/blob/master/docs/csdb_queries.md).

CSDB::GetVersion()
------------------

Access:  public

Parameters:  None.

Returns:  An empty string.

This function returns a string containing the version of the connected database to display to the user (e.g. to allow the user to verify that the correct database was connected to).  The base class returns an empty string.  Derived classes are expected to override this function and execute a database query to return a displayable string.

CSDB::GetInsertID($name = null)
-------------------------------

Access:  public

Parameters:

* $name - A null value or a string containing the name of the field containing an autoincrement insert ID.

Returns:  The result of the PDO object `lastInsertId($name)` call.

This function attempts to get the result of the last INSERT query, particularly for handling AUTO INCREMENT fields.  Derived classes can override this function with database-specific queries for the last insert ID.  Support varies from database to database.

CSDB::TableExists($name)
------------------------

Access:  public

Parameters:

* $name - A string containing the name of the table to check for the existence of.

Returns:  A boolean of false.

This function detects whether or not the table name exists in the database.  The base class returns false.  Derived classes are expected to override this function and return an accurate boolean result.

CSDB::LargeResults($enable)
---------------------------

Access:  public

Parameters:

* $enable - A boolean that indicates whether or not to enable large results.

Returns:  Nothing.

This function enables/disables large results support in SOME databases.

Large results is only useful in certain circumstances.  This function, for example, will enable "unbuffered mode" for MySQL.  Under normal circumstances, PHP caches the results of a query in RAM from MySQL (aka "buffered" mode).  This is problematic for retrieving millions of rows, which can easily consume all available memory and cause PHP to unexpectedly stop working.  It is important to note that by enabling unbuffered mode, MySQL requires all returned results to be completely consumed by the application _before_ attempting to run another SQL query - otherwise, an error will occur.

CSDB::NumQueries()
------------------

Access:  public

Parameters:  None.

Returns:  An integer containing the number of queries that have been successfully executed.

This function returns the number of queries that have been executed successfully.

CSDB::ExecutionTime()
---------------------

Access:  public

Parameters:  None.

Returns:  A double containing the total amount of time, in seconds, spent executing queries.

This function returns the number of seconds spent executing queries.  The returned value has microsecond precision.

CSDB::InternalQuery($params)
----------------------------

Access:  private

Parameters:

* $params - An array as the result of calling `func_get_args()`.

Returns:  An instance of `CSDB_PDO_Statement`.  Throws an exception on error.

This internal function prepares and executes one or more parameterized queries.  This function is the main workhorse of the other query functions.

For details on running queries, see the [Queries documentation](https://github.com/cubiclesoft/csdb/blob/master/docs/csdb_queries.md).

CSDB::Quote($str, $type = PDO::PARAM_STR)
-----------------------------------------

Access:  public

Parameters:

* $str - A string to quote to prevent SQL injection attempts or a boolean of false to return a string containing "NULL".
* $type - A valid PDO PARAM type (Default is PDO::PARAM_STR).

Returns:  A string containing "NULL" if `$str` is false, a quoted string otherwise.

This function calls the PDO `quote()` function to _attempt_ to sanitize an input string for safe use in an inline SQL query.  This function should be called sparingly and strongly prefer parameterized queries over inline quoting.

CSDB::QuoteIdentifier($str)
---------------------------

Access:  public

Parameters:

* $str - A string to quote as an identifier.

Returns:  An empty string.

This function returns a string containing the quoted identifier (e.g. for use as a table name).  The base class returns an empty string.  Derived classes are expected to override this function and return an identifier suitable for the database product.

CSDB::ProcessSubqueries(&$result, &$master, $subqueries)
--------------------------------------------------------

Access:  protected

Parameters:

* $result - A string to modify, replacing `{subquerynum}` with a calculated subquery.
* $master - A boolean that specifies whether or not to use the master database to execute the final query (if any).
* $subqueries - An array containing a set of subqueries.

Returns:  A standard array of information.

This internal protected function processes subqueries for a number of other functions.

CSDB::GenerateSQL(&$master, &$sql, &$opts, $cmd, $queryinfo, $args, $subquery)
------------------------------------------------------------------------------

Access:  protected

Parameters:

* $master - A boolean that specifies whether or not to use the master database to execute the final query (if any).
* $sql - A string containing the final SQL query or an array containing the queries to run.
* $opts - An array or arrays containing the parameterized options for the SQL query or queries to run.
* $cmd - A string contianing the CSDB command to execute.
* $queryinfo - Usually an array of information containing details about the query related to the CSDB command.
* $args - An array of information containing parameters for `$queryinfo` and the final SQL query/queries.
* $subquery - A boolean indicating whether or not a subquery is being processed.

Returns:  A standard array of information.

This protected function generates one or more SQL queries in response to the input.  The base class returns a failure state.  Derived classes are expected to override this function and handle the input to generate correct SQL output for the database product.  Depending on the database product, several queries sometimes have to be run in order to accommodate the request.  The application should be completely unaware of any and all transformations involved.

CSDB::RunStatementFilter(&$stmt, &$filteropts)
----------------------------------------------

Access:  protected

Parameters:

* $stmt - A valid PDO prepare/execute result statement.
* $filteropts - An array containing information about the query.

Returns:  Nothing.

This protected function is used by derived classes to cause the driver to return the last inserted ID from an AUTO INCREMENT field for INSERT commands.  The information is saved for a later `CSDB::GetInsertID()` call.  The default function does nothing but derived classes are expected to call the parent function anyway.

CSDB::RunRowFilter(&$row, &$filteropts, &$fetchnext)
----------------------------------------------------

Access:  protected

Parameters:

* $row - An object representing the current result row.
* $filteropts - An array containing information about the query.
* $fetchnext - A boolean indicating whether or not the caller should skip to the next row.

Returns:  Nothing.

This protected function is used by derived classes to alter the row based on certain commands so that standardized output is the result.  Mostly this functionality is used for the various SHOW commands.  Derived classes are expected to call the parent function when `$fetchnext` is false.

CSDB::ProcessColumnDefinition($info)
------------------------------------

Access:  protected

Parameters:

* $info - An array containing information about a column definition.

Returns:  A standard array of information.

This protected function returns the appropriate SQL for the input column definition.  The base class returns a failure state.  Derived classes are expected to override this function and handle the input to generate correct SQL output for the database product.

CSDB::ProcessKeyDefinition($info)
---------------------------------

Access:  protected

Parameters:

* $info - An array containing information about a CREATE TABLE index key definition.

Returns:  A standard array of information.

This protected function returns the appropriate SQL for the input index key definition.  The base class returns a failure state.  Derived classes are expected to override this function and handle the input to generate correct SQL output for the database product.  Not all database products support all key types inside CREATE TABLE, so those products may implicitly run additional ADD INDEX queries after CREATE TABLE to create a similar effect.

CSDB::ProcessSELECT(&$master, &$sql, &$opts, $queryinfo, $args, $subquery, $supported)
--------------------------------------------------------------------------------------

Access:  protected

Parameters:

* $master - A boolean that specifies whether or not to use the master database to execute the final query (if any).
* $sql - A string containing the final SQL query or an array containing the queries to run.
* $opts - An array or arrays containing the parameterized options for the SQL query or queries to run.
* $queryinfo - Usually an array of information containing details about the query related to the CSDB command.
* $args - An array of information containing parameters for `$queryinfo` and the final SQL query/queries.
* $subquery - A boolean indicating whether or not a subquery is being processed.
* $supported - An array of options supported by the database.

Returns:  A standard array of information.

This protected helper function is used by derived classes to generate a SELECT SQL query.  This function is not intended to be overridden.  This function performs common handling of SELECT statements for all database classes to minimize code duplication.

CSDB::ProcessINSERT(&$master, &$sql, &$opts, $queryinfo, $args, $subquery, $supported)
--------------------------------------------------------------------------------------

Access:  protected

Parameters:

* $master - A boolean that specifies whether or not to use the master database to execute the final query (if any).
* $sql - A string containing the final SQL query or an array containing the queries to run.
* $opts - An array or arrays containing the parameterized options for the SQL query or queries to run.
* $queryinfo - Usually an array of information containing details about the query related to the CSDB command.
* $args - An array of information containing parameters for `$queryinfo` and the final SQL query/queries.
* $subquery - A boolean indicating whether or not a subquery is being processed.
* $supported - An array of options supported by the database.

Returns:  A standard array of information.

This protected helper function is used by derived classes to generate an INSERT SQL query.  This function is not intended to be overridden.  This function performs common handling of INSERT statements for all database classes to minimize code duplication.

CSDB::ProcessUPDATE(&$master, &$sql, &$opts, $queryinfo, $args, $subquery, $supported)
--------------------------------------------------------------------------------------

Access:  protected

Parameters:

* $master - A boolean that specifies whether or not to use the master database to execute the final query (if any).
* $sql - A string containing the final SQL query or an array containing the queries to run.
* $opts - An array or arrays containing the parameterized options for the SQL query or queries to run.
* $queryinfo - Usually an array of information containing details about the query related to the CSDB command.
* $args - An array of information containing parameters for `$queryinfo` and the final SQL query/queries.
* $subquery - A boolean indicating whether or not a subquery is being processed.
* $supported - An array of options supported by the database.

Returns:  A standard array of information.

This protected helper function is used by derived classes to generate an UPDATE SQL query.  This function is not intended to be overridden.  This function performs common handling of UPDATE statements for all database classes to minimize code duplication.

This function also catches certain accidental issues and will throw an exception.  For example, typing `"WHERE => id = ?"` instead of `"WHERE" => "id = ?"` will trigger the exception.  The former would unintentionally modify all rows in the table while the latter only modifies one row.

CSDB::ProcessDELETE(&$master, &$sql, &$opts, $queryinfo, $args, $subquery, $supported)
--------------------------------------------------------------------------------------

Access:  protected

Parameters:

* $master - A boolean that specifies whether or not to use the master database to execute the final query (if any).
* $sql - A string containing the final SQL query or an array containing the queries to run.
* $opts - An array or arrays containing the parameterized options for the SQL query or queries to run.
* $queryinfo - Usually an array of information containing details about the query related to the CSDB command.
* $args - An array of information containing parameters for `$queryinfo` and the final SQL query/queries.
* $subquery - A boolean indicating whether or not a subquery is being processed.
* $supported - An array of options supported by the database.

Returns:  A standard array of information.

This protected helper function is used by derived classes to generate a DELETE SQL query.  This function is not intended to be overridden.  This function performs common handling of DELETE statements for all database classes to minimize code duplication.

This function also catches certain accidental issues and will throw an exception.  For example, typing `"WHERE => id = ?"` instead of `"WHERE" => "id = ?"` will trigger the exception.  The former would unintentionally delete all rows in the table while the latter only deletes one row.

CSDB::ProcessCREATE_TABLE(&$master, &$sql, &$opts, $queryinfo, $args, $subquery, $supported)
--------------------------------------------------------------------------------------------

Access:  protected

Parameters:

* $master - A boolean that specifies whether or not to use the master database to execute the final query (if any).
* $sql - A string containing the final SQL query or an array containing the queries to run.
* $opts - An array or arrays containing the parameterized options for the SQL query or queries to run.
* $queryinfo - Usually an array of information containing details about the query related to the CSDB command.
* $args - An array of information containing parameters for `$queryinfo` and the final SQL query/queries.
* $subquery - A boolean indicating whether or not a subquery is being processed.
* $supported - An array of options supported by the database.

Returns:  A standard array of information.

This protected helper function is used by derived classes to generate a CREATE TABLE SQL query.  This function is not intended to be overridden.  This function performs common handling of CREATE TABLE statements for all database classes to minimize code duplication.

CSDB::ProcessReferenceDefinition($info)
---------------------------------------

Access:  protected

Parameters:

* $info - An array containing information about a foreign key reference.

Returns:  A valid SQL string.

This protected function returns the appropriate SQL for the input foreign key definition.  This function is not intended to be overridden.  This function performs common handling of foreign key contstraints for all database classes to minimize code duplication.

CSDB::DB_Translate($format, ...)
--------------------------------

Access:  protected static

Parameters:

* $format - A string containing valid sprintf() format specifiers.

Returns:  A string containing a translation.

This protected static function takes input strings and translates them from English to some other language if CS_TRANSLATE_FUNC is defined to be a valid PHP function name.

CSDB_PDO_Statement::__construct($db, $stmt, $filteropts)
--------------------------------------------------------

Access:  public

Parameters:

* $db - A valid CSDB object.
* $stmt - A valid PDO prepare/execute result statement.
* $filteropts - An array containing information about the query for filtering rows.

Returns:  Nothing.

This function instantiates the class for fetching row data.

CSDB_PDO_Statement::Free()
--------------------------

Access:  public

Parameters:  None.

Returns:  A boolean of true on success, false otherwise.

This function frees the statement associated with the object.  This is automatically called by the destructor, so it is rare to need to do this directly.

CSDB_PDO_Statement::NextRow($fetchtype = PDO::FETCH_OBJ)
--------------------------------------------------------

Access:  public

Parameters:

* $fetchtype - An integer containing one of the allowed PDO [fetch styles](http://php.net/manual/en/pdostatement.fetch.php) (Default is PDO::FETCH_OBJ).

Returns:  An object or array depending on `$fetchtype` if a row is retrieved, false otherwise.

This function retrieves the next row from the database.  If filteropts are associated with this request, more rows might be retrieved until the supplied `$fetchnext` remains false.  When filteropts is applied, only objects are retrieved and attempted to be translated to other PDO types once the row is allowed through.  PDO::FETCH_NUM, PDO::FETCH_ASSOC, PDO::FETCH_BOTH, and PDO::FETCH_OBJ are well-supported fetch types in CSDB.
