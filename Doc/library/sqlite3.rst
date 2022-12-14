:mod:`sqlite3` --- DB-API 2.0 interface for SQLite databases
============================================================

.. module:: sqlite3
   :synopsis: A DB-API 2.0 implementation using SQLite 3.x.

.. sectionauthor:: Gerhard Häring <gh@ghaering.de>

**Source code:** :source:`Lib/sqlite3/`


.. _sqlite3-intro:

Introduction
------------

SQLite is a C library that provides a lightweight disk-based database that
doesn't require a separate server process and allows accessing the database
using a nonstandard variant of the SQL query language. Some applications can use
SQLite for internal data storage.  It's also possible to prototype an
application using SQLite and then port the code to a larger database such as
PostgreSQL or Oracle.

The sqlite3 module was written by Gerhard Häring.  It provides an SQL interface
compliant with the DB-API 2.0 specification described by :pep:`249`, and
requires SQLite 3.7.15 or newer.

This document includes four main sections:

* :ref:`sqlite3-tutorial` teaches how to use the sqlite3 module.
* :ref:`sqlite3-reference` describes the classes and functions this module
  defines.
* :ref:`sqlite3-howtos` details how to handle specific tasks.
* :ref:`sqlite3-explanation` provides in-depth background on
  transaction control.


.. _sqlite3-tutorial:

Tutorial
--------

To use the module, start by creating a :class:`Connection` object that
represents the database.  Here the data will be stored in the
:file:`example.db` file::

   import sqlite3
   con = sqlite3.connect('example.db')

The special path name ``:memory:`` can be provided to create a temporary
database in RAM.

Once a :class:`Connection` has been established, create a :class:`Cursor` object
and call its :meth:`~Cursor.execute` method to perform SQL commands::

   cur = con.cursor()

   # Create table
   cur.execute('''CREATE TABLE stocks
                  (date text, trans text, symbol text, qty real, price real)''')

   # Insert a row of data
   cur.execute("INSERT INTO stocks VALUES ('2006-01-05','BUY','RHAT',100,35.14)")

   # Save (commit) the changes
   con.commit()

   # We can also close the connection if we are done with it.
   # Just be sure any changes have been committed or they will be lost.
   con.close()

The saved data is persistent: it can be reloaded in a subsequent session even
after restarting the Python interpreter::

   import sqlite3
   con = sqlite3.connect('example.db')
   cur = con.cursor()

At this point, our database only contains one row::

   >>> res = cur.execute('SELECT count(rowid) FROM stocks')
   >>> print(res.fetchone())
   (1,)

The result is a one-item :class:`tuple`:
one row, with one column.
Now, let us insert three more rows of data,
using :meth:`~Cursor.executemany`::

   >>> data = [
   ...    ('2006-03-28', 'BUY', 'IBM', 1000, 45.0),
   ...    ('2006-04-05', 'BUY', 'MSFT', 1000, 72.0),
   ...    ('2006-04-06', 'SELL', 'IBM', 500, 53.0),
   ... ]
   >>> cur.executemany('INSERT INTO stocks VALUES(?, ?, ?, ?, ?)', data)

Then, retrieve the data by iterating over the result of a ``SELECT`` statement::

   >>> for row in cur.execute('SELECT * FROM stocks ORDER BY price'):
   ...     print(row)

   ('2006-01-05', 'BUY', 'RHAT', 100, 35.14)
   ('2006-03-28', 'BUY', 'IBM', 1000, 45.0)
   ('2006-04-06', 'SELL', 'IBM', 500, 53.0)
   ('2006-04-05', 'BUY', 'MSFT', 1000, 72.0)


.. _sqlite3-placeholders:

SQL operations usually need to use values from Python variables. However,
beware of using Python's string operations to assemble queries, as they
are vulnerable to SQL injection attacks (see the `xkcd webcomic
<https://xkcd.com/327/>`_ for a humorous example of what can go wrong)::

   # Never do this -- insecure!
   symbol = 'RHAT'
   cur.execute("SELECT * FROM stocks WHERE symbol = '%s'" % symbol)

Instead, use the DB-API's parameter substitution. To insert a variable into a
query string, use a placeholder in the string, and substitute the actual values
into the query by providing them as a :class:`tuple` of values to the second
argument of the cursor's :meth:`~Cursor.execute` method. An SQL statement may
use one of two kinds of placeholders: question marks (qmark style) or named
placeholders (named style). For the qmark style, ``parameters`` must be a
:term:`sequence <sequence>`. For the named style, it can be either a
:term:`sequence <sequence>` or :class:`dict` instance. The length of the
:term:`sequence <sequence>` must match the number of placeholders, or a
:exc:`ProgrammingError` is raised. If a :class:`dict` is given, it must contain
keys for all named parameters. Any extra items are ignored. Here's an example of
both styles:

.. literalinclude:: ../includes/sqlite3/execute_1.py


.. seealso::

   https://www.sqlite.org
      The SQLite web page; the documentation describes the syntax and the
      available data types for the supported SQL dialect.

   https://www.w3schools.com/sql/
      Tutorial, reference and examples for learning SQL syntax.

   :pep:`249` - Database API Specification 2.0
      PEP written by Marc-André Lemburg.


.. _sqlite3-reference:

Reference
---------

.. _sqlite3-module-contents:

Module functions and constants
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


.. data:: apilevel

   String constant stating the supported DB-API level. Required by the DB-API.
   Hard-coded to ``"2.0"``.

.. data:: paramstyle

   String constant stating the type of parameter marker formatting expected by
   the :mod:`sqlite3` module. Required by the DB-API. Hard-coded to
   ``"qmark"``.

   .. note::

      The :mod:`sqlite3` module supports both ``qmark`` and ``numeric`` DB-API
      parameter styles, because that is what the underlying SQLite library
      supports. However, the DB-API does not allow multiple values for
      the ``paramstyle`` attribute.

.. data:: version

   Version number of this module as a :class:`string <str>`.
   This is not the version of the SQLite library.


.. data:: version_info

   Version number of this module as a :class:`tuple` of :class:`integers <int>`.
   This is not the version of the SQLite library.


.. data:: sqlite_version

   Version number of the runtime SQLite library as a :class:`string <str>`.


.. data:: sqlite_version_info

   Version number of the runtime SQLite library as a :class:`tuple` of
   :class:`integers <int>`.


.. data:: threadsafety

   Integer constant required by the DB-API, stating the level of thread safety
   the :mod:`sqlite3` module supports. Currently hard-coded to ``1``, meaning
   *"Threads may share the module, but not connections."* However, this may not
   always be true. You can check the underlying SQLite library's compile-time
   threaded mode using the following query::

     import sqlite3
     con = sqlite3.connect(":memory:")
     con.execute("""
         select * from pragma_compile_options
         where compile_options like 'THREADSAFE=%'
     """).fetchall()

   Note that the `SQLITE_THREADSAFE levels
   <https://sqlite.org/compile.html#threadsafe>`_ do not match the DB-API 2.0
   ``threadsafety`` levels.


.. data:: PARSE_DECLTYPES

   Pass this flag value to the *detect_types* parameter of
   :func:`connect` to look up a converter function using
   the declared types for each column.
   The types are declared when the database table is created.
   ``sqlite3`` will look up a converter function using the first word of the
   declared type as the converter dictionary key.
   For example:


   .. code-block:: sql

      CREATE TABLE test(
         i integer primary key,  ! will look up a converter named "integer"
         p point,                ! will look up a converter named "point"
         n number(10)            ! will look up a converter named "number"
       )

   This flag may be combined with :const:`PARSE_COLNAMES` using the ``|``
   (bitwise or) operator.


.. data:: PARSE_COLNAMES

   Pass this flag value to the *detect_types* parameter of
   :func:`connect` to look up a converter function by
   using the type name, parsed from the query column name,
   as the converter dictionary key.
   The type name must be wrapped in square brackets (``[]``).

   .. code-block:: sql

      SELECT p as "p [point]" FROM test;  ! will look up converter "point"

   This flag may be combined with :const:`PARSE_DECLTYPES` using the ``|``
   (bitwise or) operator.



.. function:: connect(database, timeout=5.0, detect_types=0, isolation_level="DEFERRED", check_same_thread=True, factory=sqlite3.Connection, cached_statements=128, uri=False)

   Open a connection to an SQLite database.

   :param database:
       The path to the database file to be opened.
       Pass ``":memory:"`` to open a connection to a database that is
       in RAM instead of on disk.
   :type database: :term:`path-like object`

   :param timeout:
       How many seconds the connection should wait before raising
       an exception, if the database is locked by another connection.
       If another connection opens a transaction to modify the database,
       it will be locked until that transaction is committed.
       Default five seconds.
   :type timeout: float

   :param detect_types:
       Control whether and how data types not
       :ref:`natively supported by SQLite <sqlite3-types>`
       are looked up to be converted to Python types,
       using the converters registered with :func:`register_converter`.
       Set it to any combination (using ``|``, bitwise or) of
       :const:`PARSE_DECLTYPES` and :const:`PARSE_COLNAMES`
       to enable this.
       Column names takes precedence over declared types if both flags are set.
       Types cannot be detected for generated fields (for example ``max(data)``),
       even when the *detect_types* parameter is set; :class:`str` will be
       returned instead.
       By default (``0``), type detection is disabled.
   :type detect_types: int

   :param isolation_level:
       The :attr:`~Connection.isolation_level` of the connection,
       controlling whether and how transactions are implicitly opened.
       Can be ``"DEFERRED"`` (default), ``"EXCLUSIVE"`` or ``"IMMEDIATE"``;
       or :const:`None` to disable opening transactions implicitly.
       See :ref:`sqlite3-controlling-transactions` for more.
   :type isolation_level: str | :const:`None`

   :param check_same_thread:
       If :const:`True` (default), only the creating thread may use the connection.
       If :const:`False`, the connection may be shared across multiple threads;
       if so, write operations should be serialized by the user to avoid data
       corruption.
   :type check_same_thread: bool

   :param factory:
       A custom subclass of :class:`Connection` to create the connection with,
       if not the default :class:`Connection` class.
   :type factory: :class:`Connection`

   :param cached_statements:
       The number of statements that ``sqlite3``
       should internally cache for this connection, to avoid parsing overhead.
       By default, 100 statements.
   :type cached_statements: int

   :param uri:
       If set to :const:`True`, *database* is interpreted as a
       :abbr:`URI (Uniform Resource Identifier)` with a file path
       and an optional query string.
       The scheme part *must* be ``"file:"``,
       and the path can be relative or absolute.
       The query string allows passing parameters to SQLite,
       enabling various :ref:`sqlite3-uri-tricks`.
   :type uri: bool

   :rtype: Connection

   .. audit-event:: sqlite3.connect database sqlite3.connect
   .. audit-event:: sqlite3.connect/handle connection_handle sqlite3.connect

   .. versionadded:: 3.4
      The *uri* parameter.

   .. versionchanged:: 3.7
      *database* can now also be a :term:`path-like object`, not only a string.

   .. versionadded:: 3.10
      The ``sqlite3.connect/handle`` auditing event.


.. function:: register_converter(typename, converter, /)

   Register the *converter* callable to convert SQLite objects of type
   *typename* into a Python object of a specific type.
   The converter is invoked for all SQLite values of type *typename*;
   it is passed a :class:`bytes` object and should return an object of the
   desired Python type.
   Consult the parameter *detect_types* of
   :func:`connect` for information regarding how type detection works.

   Note: *typename* and the name of the type in your query are matched
   case-insensitively.


.. function:: register_adapter(type, adapter, /)

   Register an *adapter* callable to adapt the Python type *type* into an
   SQLite type.
   The adapter is called with a Python object of type *type* as its sole
   argument, and must return a value of a
   :ref:`type that SQLite natively understands<sqlite3-types>`.


.. function:: complete_statement(statement)

   Returns :const:`True` if the string *statement* contains one or more complete SQL
   statements terminated by semicolons. It does not verify that the SQL is
   syntactically correct, only that there are no unclosed string literals and the
   statement is terminated by a semicolon.

   This can be used to build a shell for SQLite, as in the following example:


   .. literalinclude:: ../includes/sqlite3/complete_statement.py


.. function:: enable_callback_tracebacks(flag, /)

   Enable or disable callback tracebacks.
   By default you will not get any tracebacks in user-defined functions,
   aggregates, converters, authorizer callbacks etc. If you want to debug them,
   you can call this function with *flag* set to ``True``. Afterwards, you will
   get tracebacks from callbacks on ``sys.stderr``. Use :const:`False` to
   disable the feature again.


.. _sqlite3-connection-objects:

Connection objects
^^^^^^^^^^^^^^^^^^

.. class:: Connection

   Each open SQLite database is represented by a ``Connection`` object,
   which is created using :func:`sqlite3.connect`.
   Their main purpose is creating :class:`Cursor` objects,
   and :ref:`sqlite3-controlling-transactions`.

   .. seealso::

      * :ref:`sqlite3-connection-shortcuts`
      * :ref:`sqlite3-connection-context-manager`

   An SQLite database connection has the following attributes and methods:

   .. attribute:: isolation_level

      This attribute controls the :ref:`transaction handling
      <sqlite3-controlling-transactions>` performed by ``sqlite3``.
      If set to :const:`None`, transactions are never implicitly opened.
      If set to one of ``"DEFERRED"``, ``"IMMEDIATE"``, or ``"EXCLUSIVE"``,
      corresponding to the underlying `SQLite transaction behaviour`_,
      implicit :ref:`transaction management
      <sqlite3-controlling-transactions>` is performed.

      If not overridden by the *isolation_level* parameter of :func:`connect`,
      the default is ``""``, which is an alias for ``"DEFERRED"``.

   .. attribute:: in_transaction

      This read-only attribute corresponds to the low-level SQLite
      `autocommit mode`_.

      :const:`True` if a transaction is active (there are uncommitted changes),
      :const:`False` otherwise.

      .. versionadded:: 3.2

   .. method:: cursor(factory=Cursor)

      Create and return a :class:`Cursor` object.
      The cursor method accepts a single optional parameter *factory*. If
      supplied, this must be a callable returning an instance of :class:`Cursor`
      or its subclasses.

   .. method:: commit()

      Commit any pending transaction to the database.
      If there is no open transaction, this method is a no-op.

   .. method:: rollback()

      Roll back to the start of any pending transaction.
      If there is no open transaction, this method is a no-op.

   .. method:: close()

      Close the database connection.
      Any pending transaction is not committed implicitly;
      make sure to :meth:`commit` before closing
      to avoid losing pending changes.

   .. method:: execute(sql, parameters=(), /)

      Create a new :class:`Cursor` object and call
      :meth:`~Cursor.execute` on it with the given *sql* and *parameters*.
      Return the new cursor object.

   .. method:: executemany(sql, parameters, /)

      Create a new :class:`Cursor` object and call
      :meth:`~Cursor.executemany` on it with the given *sql* and *parameters*.
      Return the new cursor object.

   .. method:: executescript(sql_script, /)

      Create a new :class:`Cursor` object and call
      :meth:`~Cursor.executescript` on it with the given *sql_script*.
      Return the new cursor object.

   .. method:: create_function(name, narg, func, *, deterministic=False)

      Create or remove a user-defined SQL function.

      :param name:
          The name of the SQL function.
      :type name: str

      :param narg:
          The number of arguments the SQL function can accept.
          If ``-1``, it may take any number of arguments.
      :type narg: int

      :param func:
          A callable that is called when the SQL function is invoked.
          The callable must return :ref:`a type natively supported by SQLite
          <sqlite3-types>`.
          Set to :const:`None` to remove an existing SQL function.
      :type func: :term:`callback` | :const:`None`

      :param deterministic:
          If :const:`True`, the created SQL function is marked as
          `deterministic <https://sqlite.org/deterministic.html>`_,
          which allows SQLite to perform additional optimizations.
      :type deterministic: bool

      :raises NotSupportedError:
          If *deterministic* is used with SQLite versions older than 3.8.3.

      .. versionadded:: 3.8
         The *deterministic* parameter.

      Example:

      .. literalinclude:: ../includes/sqlite3/md5func.py


   .. method:: create_aggregate(name, /, n_arg, aggregate_class)

      Create or remove a user-defined SQL aggregate function.

      :param name:
          The name of the SQL aggregate function.
      :type name: str

      :param n_arg:
          The number of arguments the SQL aggregate function can accept.
          If ``-1``, it may take any number of arguments.
      :type n_arg: int

      :param aggregate_class:
          A class must implement the following methods:

          * ``step()``: Add a row to the aggregate.
          * ``finalize()``: Return the final result of the aggregate as
            :ref:`a type natively supported by SQLite <sqlite3-types>`.

          The number of arguments that the ``step()`` method must accept
          is controlled by *n_arg*.

          Set to :const:`None` to remove an existing SQL aggregate function.
      :type aggregate_class: :term:`class` | :const:`None`

      Example:

      .. literalinclude:: ../includes/sqlite3/mysumaggr.py


   .. method:: create_collation(name, callable)

      Create a collation named *name* using the collating function *callable*.
      *callable* is passed two :class:`string <str>` arguments,
      and it should return an :class:`integer <int>`:

      * ``1`` if the first is ordered higher than the second
      * ``-1`` if the first is ordered lower than the second
      * ``0`` if they are ordered equal

      The following example shows a reverse sorting collation:

      .. literalinclude:: ../includes/sqlite3/collation_reverse.py

      Remove a collation function by setting *callable* to :const:`None`.


   .. method:: interrupt()

      Call this method from a different thread to abort any queries that might
      be executing on the connection.
      Aborted queries will raise an exception.


   .. method:: set_authorizer(authorizer_callback)

      Register callable *authorizer_callback* to be invoked for each attempt to
      access a column of a table in the database. The callback should return
      :const:`SQLITE_OK` if access is allowed, :const:`SQLITE_DENY` if the entire SQL
      statement should be aborted with an error and :const:`SQLITE_IGNORE` if the
      column should be treated as a NULL value. These constants are available in the
      :mod:`sqlite3` module.

      The first argument to the callback signifies what kind of operation is to be
      authorized. The second and third argument will be arguments or :const:`None`
      depending on the first argument. The 4th argument is the name of the database
      ("main", "temp", etc.) if applicable. The 5th argument is the name of the
      inner-most trigger or view that is responsible for the access attempt or
      :const:`None` if this access attempt is directly from input SQL code.

      Please consult the SQLite documentation about the possible values for the first
      argument and the meaning of the second and third argument depending on the first
      one. All necessary constants are available in the :mod:`sqlite3` module.


   .. method:: set_progress_handler(progress_handler, n)

      Register callable *progress_handler* to be invoked for every *n*
      instructions of the SQLite virtual machine. This is useful if you want to
      get called from SQLite during long-running operations, for example to update
      a GUI.

      If you want to clear any previously installed progress handler, call the
      method with :const:`None` for *progress_handler*.

      Returning a non-zero value from the handler function will terminate the
      currently executing query and cause it to raise an :exc:`OperationalError`
      exception.


   .. method:: set_trace_callback(trace_callback)

      Register callable *trace_callback* to be invoked for each SQL statement
      that is actually executed by the SQLite backend.

      The only argument passed to the callback is the statement (as
      :class:`str`) that is being executed. The return value of the callback is
      ignored. Note that the backend does not only run statements passed to the
      :meth:`Cursor.execute` methods.  Other sources include the
      :ref:`transaction management <sqlite3-controlling-transactions>` of the
      sqlite3 module and the execution of triggers defined in the current
      database.

      Passing :const:`None` as *trace_callback* will disable the trace callback.

      .. note::
         Exceptions raised in the trace callback are not propagated. As a
         development and debugging aid, use
         :meth:`~sqlite3.enable_callback_tracebacks` to enable printing
         tracebacks from exceptions raised in the trace callback.

      .. versionadded:: 3.3


   .. method:: enable_load_extension(enabled, /)

      Enable the SQLite engine to load SQLite extensions from shared libraries
      if *enabled* is :const:`True`;
      else, disallow loading SQLite extensions.
      SQLite extensions can define new functions,
      aggregates or whole new virtual table implementations.  One well-known
      extension is the fulltext-search extension distributed with SQLite.

      .. note::

         The ``sqlite3`` module is not built with loadable extension support by
         default, because some platforms (notably macOS) have SQLite
         libraries which are compiled without this feature.
         To get loadable extension support,
         you must pass the :option:`--enable-loadable-sqlite-extensions` option
         to :program:`configure`.

      .. audit-event:: sqlite3.enable_load_extension connection,enabled sqlite3.Connection.enable_load_extension

      .. versionadded:: 3.2

      .. versionchanged:: 3.10
         Added the ``sqlite3.enable_load_extension`` auditing event.

      .. literalinclude:: ../includes/sqlite3/load_extension.py

   .. method:: load_extension(path, /)

      Load an SQLite extension from a shared library located at *path*.
      Enable extension loading with :meth:`enable_load_extension` before
      calling this method.

      .. audit-event:: sqlite3.load_extension connection,path sqlite3.Connection.load_extension

      .. versionadded:: 3.2

      .. versionchanged:: 3.10
         Added the ``sqlite3.load_extension`` auditing event.

   .. attribute:: row_factory

      A callable that accepts two arguments,
      a :class:`Cursor` object and the raw row results as a :class:`tuple`,
      and returns a custom object representing an SQLite row.

      Example:

      .. literalinclude:: ../includes/sqlite3/row_factory.py

      If returning a tuple doesn't suffice and you want name-based access to
      columns, you should consider setting :attr:`row_factory` to the
      highly optimized :class:`sqlite3.Row` type. :class:`Row` provides both
      index-based and case-insensitive name-based access to columns with almost no
      memory overhead. It will probably be better than your own custom
      dictionary-based approach or even a db_row based solution.

      .. XXX what's a db_row-based solution?


   .. attribute:: text_factory

      A callable that accepts a :class:`bytes` parameter and returns a text
      representation of it.
      The callable is invoked for SQLite values with the ``TEXT`` data type.
      By default, this attribute is set to :class:`str`.
      If you want to return ``bytes`` instead, set *text_factory* to ``bytes``.

      Example:

      .. literalinclude:: ../includes/sqlite3/text_factory.py


   .. attribute:: total_changes

      Return the total number of database rows that have been modified, inserted, or
      deleted since the database connection was opened.


   .. method:: iterdump

      Return an :term:`iterator` to dump the database as SQL source code.
      Useful when saving an in-memory database for later restoration.
      Similar to the ``.dump`` command in the :program:`sqlite3` shell.

      Example::

         # Convert file existing_db.db to SQL dump file dump.sql
         import sqlite3

         con = sqlite3.connect('existing_db.db')
         with open('dump.sql', 'w') as f:
             for line in con.iterdump():
                 f.write('%s\n' % line)
         con.close()


   .. method:: backup(target, *, pages=-1, progress=None, name="main", sleep=0.250)

      Create a backup of an SQLite database.

      Works even if the database is being accessed by other clients
      or concurrently by the same connection.

      :param target:
          The database connection to save the backup to.
      :type target: Connection

      :param pages:
          The number of pages to copy at a time.
          If equal to or less than ``0``,
          the entire database is copied in a single step.
          Defaults to ``-1``.
      :type pages: int

      :param progress:
          If set to a callable, it is invoked with three integer arguments for
          every backup iteration:
          the *status* of the last iteration,
          the *remaining* number of pages still to be copied,
          and the *total* number of pages.
          Defaults to :const:`None`.
      :type progress: :term:`callback` | :const:`None`

      :param name:
          The name of the database to back up.
          Either ``"main"`` (the default) for the main database,
          ``"temp"`` for the temporary database,
          or the name of a custom database as attached using the
          ``ATTACH DATABASE`` SQL statment.
      :type name: str

      :param sleep:
          The number of seconds to sleep between successive attempts
          to back up remaining pages.
      :type sleep: float

      Example 1, copy an existing database into another::

         import sqlite3

         def progress(status, remaining, total):
             print(f'Copied {total-remaining} of {total} pages...')

         con = sqlite3.connect('existing_db.db')
         bck = sqlite3.connect('backup.db')
         with bck:
             con.backup(bck, pages=1, progress=progress)
         bck.close()
         con.close()

      Example 2, copy an existing database into a transient copy::

         import sqlite3

         source = sqlite3.connect('existing_db.db')
         dest = sqlite3.connect(':memory:')
         source.backup(dest)

      .. versionadded:: 3.7


.. _sqlite3-cursor-objects:

Cursor objects
^^^^^^^^^^^^^^

   A ``Cursor`` object represents a `database cursor`_
   which is used to execute SQL statements,
   and manage the context of a fetch operation.
   Cursors are created using :meth:`Connection.cursor`,
   or by using any of the :ref:`connection shortcut methods
   <sqlite3-connection-shortcuts>`.

   Cursor objects are :term:`iterators <iterator>`,
   meaning that if you :meth:`~Cursor.execute` a ``SELECT`` query,
   you can simply iterate over the cursor to fetch the resulting rows::

      for row in cur.execute("select * from data"):
          print(row)

   .. _database cursor: https://en.wikipedia.org/wiki/Cursor_(databases)

.. class:: Cursor

   A :class:`Cursor` instance has the following attributes and methods.

   .. index:: single: ? (question mark); in SQL statements
   .. index:: single: : (colon); in SQL statements

   .. method:: execute(sql, parameters=(), /)

      Execute SQL statement *sql*.
      Bind values to the statement using :ref:`placeholders
      <sqlite3-placeholders>` that map to the :term:`sequence` or :class:`dict`
      *parameters*.

      :meth:`execute` will only execute a single SQL statement. If you try to execute
      more than one statement with it, it will raise a :exc:`Warning`. Use
      :meth:`executescript` if you want to execute multiple SQL statements with one
      call.

      If :attr:`~Connection.isolation_level` is not :const:`None`,
      *sql* is an ``INSERT``, ``UPDATE``, ``DELETE``, or ``REPLACE`` statement,
      and there is no open transaction,
      a transaction is implicitly opened before executing *sql*.


   .. method:: executemany(sql, parameters, /)

      Execute :ref:`parameterized <sqlite3-placeholders>` SQL statement *sql*
      against all parameter sequences or mappings found in the sequence
      *parameters*.  It is also possible to use an
      :term:`iterator` yielding parameters instead of a sequence.
      Uses the same implicit transaction handling as :meth:`~Cursor.execute`.

      Example::

          data = [
              ("row1",),
              ("row2",),
          ]
          # cur is an sqlite3.Cursor object
          cur.executemany("insert into t values(?)", data)

   .. method:: executescript(sql_script, /)

      Execute the SQL statements in *sql_script*.
      If there is a pending transaciton,
      an implicit ``COMMIT`` statement is executed first.
      No other implicit transaction control is performed;
      any transaction control must be added to *sql_script*.

      *sql_script* must be a :class:`string <str>`.

      Example::

         # cur is an sqlite3.Cursor object
         cur.executescript("""
             begin;
             create table person(firstname, lastname, age);
             create table book(title, author, published);
             create table publisher(name, address);
             commit;
         """)


   .. method:: fetchone()

      Fetch the next row of a query result set as a :class:`tuple`.
      Return :const:`None` if no more data is available.


   .. method:: fetchmany(size=cursor.arraysize)

      Fetch the next set of rows of a query result as a :class:`list`.
      Return an empty list if no more rows are available.

      The number of rows to fetch per call is specified by the *size* parameter.
      If *size* is not given, :attr:`arraysize` determines the number of rows
      to be fetched.
      If fewer than *size* rows are available,
      as many rows as are available are returned.

      Note there are performance considerations involved with the *size* parameter.
      For optimal performance, it is usually best to use the arraysize attribute.
      If the *size* parameter is used, then it is best for it to retain the same
      value from one :meth:`fetchmany` call to the next.

   .. method:: fetchall()

      Fetch all (remaining) rows of a query result as a :class:`list`.
      Return an empty list if no rows are available.
      Note that the :attr:`arraysize` attribute can affect the performance of
      this operation.

   .. method:: close()

      Close the cursor now (rather than whenever ``__del__`` is called).

      The cursor will be unusable from this point forward; a :exc:`ProgrammingError`
      exception will be raised if any operation is attempted with the cursor.

   .. method:: setinputsizes(sizes, /)

      Required by the DB-API. Does nothing in :mod:`sqlite3`.

   .. method:: setoutputsize(size, column=None, /)

      Required by the DB-API. Does nothing in :mod:`sqlite3`.

   .. attribute:: rowcount

      Read-only attribute that provides the number of modified rows for
      ``INSERT``, ``UPDATE``, ``DELETE``, and ``REPLACE`` statements;
      is ``-1`` for other statements,
      including :abbr:`CTE (Common Table Expression)` queries.
      It is only updated by the :meth:`execute` and :meth:`executemany` methods.

   .. attribute:: lastrowid

      Read-only attribute that provides the row id of the last inserted row. It
      is only updated after successful ``INSERT`` or ``REPLACE`` statements
      using the :meth:`execute` method.  For other statements, after
      :meth:`executemany` or :meth:`executescript`, or if the insertion failed,
      the value of ``lastrowid`` is left unchanged.  The initial value of
      ``lastrowid`` is :const:`None`.

      .. note::
         Inserts into ``WITHOUT ROWID`` tables are not recorded.

      .. versionchanged:: 3.6
         Added support for the ``REPLACE`` statement.

   .. attribute:: arraysize

      Read/write attribute that controls the number of rows returned by :meth:`fetchmany`.
      The default value is 1 which means a single row would be fetched per call.

   .. attribute:: description

      Read-only attribute that provides the column names of the last query. To
      remain compatible with the Python DB API, it returns a 7-tuple for each
      column where the last six items of each tuple are :const:`None`.

      It is set for ``SELECT`` statements without any matching rows as well.

   .. attribute:: connection

      Read-only attribute that provides the SQLite database :class:`Connection`
      belonging to the cursor.  A :class:`Cursor` object created by
      calling :meth:`con.cursor() <Connection.cursor>` will have a
      :attr:`connection` attribute that refers to *con*::

         >>> con = sqlite3.connect(":memory:")
         >>> cur = con.cursor()
         >>> cur.connection == con
         True

.. _sqlite3-row-objects:

Row objects
^^^^^^^^^^^

.. class:: Row

   A :class:`Row` instance serves as a highly optimized
   :attr:`~Connection.row_factory` for :class:`Connection` objects.
   It tries to mimic a :class:`tuple` in most of its features,
   and supports iteration, :func:`repr`, equality testing, :func:`len`,
   and :term:`mapping` access by column name and index.

   Two row objects compare equal if have equal columns and equal members.

   .. method:: keys

      Return a :class:`list` of column names as :class:`strings <str>`.
      Immediately after a query,
      it is the first member of each tuple in :attr:`Cursor.description`.

   .. versionchanged:: 3.5
      Added support of slicing.

Let's assume we initialize a table as in the example given above::

   con = sqlite3.connect(":memory:")
   cur = con.cursor()
   cur.execute('''create table stocks
   (date text, trans text, symbol text,
    qty real, price real)''')
   cur.execute("""insert into stocks
               values ('2006-01-05','BUY','RHAT',100,35.14)""")
   con.commit()
   cur.close()

Now we plug :class:`Row` in::

   >>> con.row_factory = sqlite3.Row
   >>> cur = con.cursor()
   >>> cur.execute('select * from stocks')
   <sqlite3.Cursor object at 0x7f4e7dd8fa80>
   >>> r = cur.fetchone()
   >>> type(r)
   <class 'sqlite3.Row'>
   >>> tuple(r)
   ('2006-01-05', 'BUY', 'RHAT', 100.0, 35.14)
   >>> len(r)
   5
   >>> r[2]
   'RHAT'
   >>> r.keys()
   ['date', 'trans', 'symbol', 'qty', 'price']
   >>> r['qty']
   100.0
   >>> for member in r:
   ...     print(member)
   ...
   2006-01-05
   BUY
   RHAT
   100.0
   35.14


PrepareProtocol objects
^^^^^^^^^^^^^^^^^^^^^^^

.. class:: PrepareProtocol

   The PrepareProtocol type's single purpose is to act as a :pep:`246` style
   adaption protocol for objects that can :ref:`adapt themselves
   <sqlite3-conform>` to :ref:`native SQLite types <sqlite3-types>`.


.. _sqlite3-exceptions:

Exceptions
^^^^^^^^^^

The exception hierarchy is defined by the DB-API 2.0 (:pep:`249`).

.. exception:: Warning

   This exception is raised by ``sqlite3`` if an SQL query is not a
   :class:`string <str>`, or if multiple statements are passed to
   :meth:`~Cursor.execute` or :meth:`~Cursor.executemany`.
   ``Warning`` is a subclass of :exc:`Exception`.

.. exception:: Error

   The base class of the other exceptions in this module.
   Use this to catch all errors with one single :keyword:`except` statement.
   ``Error`` is a subclass of :exc:`Exception`.

.. exception:: InterfaceError

   This exception is raised by ``sqlite3`` for fetch across rollback,
   or if ``sqlite3`` is unable to bind parameters.
   ``InterfaceError`` is a subclass of :exc:`Error`.

.. exception:: DatabaseError

   Exception raised for errors that are related to the database.
   This serves as the base exception for several types of database errors.
   It is only raised implicitly through the specialised subclasses.
   ``DatabaseError`` is a subclass of :exc:`Error`.

.. exception:: DataError

   Exception raised for errors caused by problems with the processed data,
   like numeric values out of range, and strings which are too long.
   ``DataError`` is a subclass of :exc:`DatabaseError`.

.. exception:: OperationalError

   Exception raised for errors that are related to the database's operation,
   and not necessarily under the control of the programmer.
   For example, the database path is not found,
   or a transaction could not be processed.
   ``OperationalError`` is a subclass of :exc:`DatabaseError`.

.. exception:: IntegrityError

   Exception raised when the relational integrity of the database is affected,
   e.g. a foreign key check fails.  It is a subclass of :exc:`DatabaseError`.

.. exception:: InternalError

   Exception raised when SQLite encounters an internal error.
   If this is raised, it may indicate that there is a problem with the runtime
   SQLite library.
   ``InternalError`` is a subclass of :exc:`DatabaseError`.

.. exception:: ProgrammingError

   Exception raised for ``sqlite3`` API programming errors,
   for example trying to operate on a closed :class:`Connection`,
   or trying to execute non-DML statements with :meth:`~Cursor.executemany`.
   ``ProgrammingError`` is a subclass of :exc:`DatabaseError`.

.. exception:: NotSupportedError

   Exception raised in case a method or database API is not supported by the
   underlying SQLite library. For example, setting *deterministic* to
   :const:`True` in :meth:`~Connection.create_function`, if the underlying SQLite library
   does not support deterministic functions.
   ``NotSupportedError`` is a subclass of :exc:`DatabaseError`.


.. _sqlite3-types:

SQLite and Python types
^^^^^^^^^^^^^^^^^^^^^^^

SQLite natively supports the following types: ``NULL``, ``INTEGER``,
``REAL``, ``TEXT``, ``BLOB``.

The following Python types can thus be sent to SQLite without any problem:

+-------------------------------+-------------+
| Python type                   | SQLite type |
+===============================+=============+
| :const:`None`                 | ``NULL``    |
+-------------------------------+-------------+
| :class:`int`                  | ``INTEGER`` |
+-------------------------------+-------------+
| :class:`float`                | ``REAL``    |
+-------------------------------+-------------+
| :class:`str`                  | ``TEXT``    |
+-------------------------------+-------------+
| :class:`bytes`                | ``BLOB``    |
+-------------------------------+-------------+


This is how SQLite types are converted to Python types by default:

+-------------+----------------------------------------------+
| SQLite type | Python type                                  |
+=============+==============================================+
| ``NULL``    | :const:`None`                                |
+-------------+----------------------------------------------+
| ``INTEGER`` | :class:`int`                                 |
+-------------+----------------------------------------------+
| ``REAL``    | :class:`float`                               |
+-------------+----------------------------------------------+
| ``TEXT``    | depends on :attr:`~Connection.text_factory`, |
|             | :class:`str` by default                      |
+-------------+----------------------------------------------+
| ``BLOB``    | :class:`bytes`                               |
+-------------+----------------------------------------------+

The type system of the :mod:`sqlite3` module is extensible in two ways: you can
store additional Python types in an SQLite database via
:ref:`object adapters <sqlite3-adapters>`,
and you can let the ``sqlite3`` module convert SQLite types to
Python types via :ref:`converters <sqlite3-converters>`.


.. _sqlite3-howtos:

How-to guides
-------------

.. _sqlite3-adapters:

Using adapters to store custom Python types in SQLite databases
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

SQLite supports only a limited set of data types natively.
To store custom Python types in SQLite databases, *adapt* them to one of the
:ref:`Python types SQLite natively understands<sqlite3-types>`.

There are two ways to adapt Python objects to SQLite types:
letting your object adapt itself, or using an *adapter callable*.
The latter will take precedence above the former.
For a library that exports a custom type,
it may make sense to enable that type to adapt itself.
As an application developer, it may make more sense to take direct control by
registering custom adapter functions.


.. _sqlite3-conform:

Letting your object adapt itself
""""""""""""""""""""""""""""""""

Suppose we have a ``Point`` class that represents a pair of coordinates,
``x`` and ``y``, in a Cartesian coordinate system.
The coordinate pair will be stored as a text string in the database,
using a semicolon to separate the coordinates.
This can be implemented by adding a ``__conform__(self, protocol)``
method which returns the adapted value.
The object passed to *protocol* will be of type :class:`PrepareProtocol`.

.. literalinclude:: ../includes/sqlite3/adapter_point_1.py


Registering an adapter callable
"""""""""""""""""""""""""""""""

The other possibility is to create a function that converts the Python object
to an SQLite-compatible type.
This function can then be registered using :func:`register_adapter`.

.. literalinclude:: ../includes/sqlite3/adapter_point_2.py


.. _sqlite3-converters:

Converting SQLite values to custom Python types
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Writing an adapter lets you convert *from* custom Python types *to* SQLite
values.
To be able to convert *from* SQLite values *to* custom Python types,
we use *converters*.

Let's go back to the :class:`Point` class. We stored the x and y coordinates
separated via semicolons as strings in SQLite.

First, we'll define a converter function that accepts the string as a parameter
and constructs a :class:`Point` object from it.

.. note::

   Converter functions are **always** passed a :class:`bytes` object,
   no matter the underlying SQLite data type.

::

   def convert_point(s):
       x, y = map(float, s.split(b";"))
       return Point(x, y)

We now need to tell ``sqlite3`` when it should convert a given SQLite value.
This is done when connecting to a database, using the *detect_types* parameter
of :func:`connect`. There are three options:

* Implicit: set *detect_types* to :const:`PARSE_DECLTYPES`
* Explicit: set *detect_types* to :const:`PARSE_COLNAMES`
* Both: set *detect_types* to
  ``sqlite3.PARSE_DECLTYPES | sqlite3.PARSE_COLNAMES``.
  Column names take precedence over declared types.

The following example illustrates the implicit and explicit approaches:

.. literalinclude:: ../includes/sqlite3/converter_point.py


Default adapters and converters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There are default adapters for the date and datetime types in the datetime
module. They will be sent as ISO dates/ISO timestamps to SQLite.

The default converters are registered under the name "date" for
:class:`datetime.date` and under the name "timestamp" for
:class:`datetime.datetime`.

This way, you can use date/timestamps from Python without any additional
fiddling in most cases. The format of the adapters is also compatible with the
experimental SQLite date/time functions.

The following example demonstrates this.

.. literalinclude:: ../includes/sqlite3/pysqlite_datetime.py

If a timestamp stored in SQLite has a fractional part longer than 6
numbers, its value will be truncated to microsecond precision by the
timestamp converter.

.. note::

   The default "timestamp" converter ignores UTC offsets in the database and
   always returns a naive :class:`datetime.datetime` object. To preserve UTC
   offsets in timestamps, either leave converters disabled, or register an
   offset-aware converter with :func:`register_converter`.


.. _sqlite3-adapter-converter-recipes:

Adapter and converter recipes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This section shows recipes for common adapters and converters.

.. code-block::

   import datetime
   import sqlite3

   def adapt_date_iso(val):
       """Adapt datetime.date to ISO 8601 date."""
       return val.isoformat()

   def adapt_datetime_iso(val):
       """Adapt datetime.datetime to timezone-naive ISO 8601 date."""
       return val.isoformat()

   def adapt_datetime_epoch(val)
       """Adapt datetime.datetime to Unix timestamp."""
       return int(val.timestamp())

   sqlite3.register_adapter(datetime.date, adapt_date_iso)
   sqlite3.register_adapter(datetime.datetime, adapt_datetime_iso)
   sqlite3.register_adapter(datetime.datetime, adapt_datetime_epoch)

   def convert_date(val):
       """Convert ISO 8601 date to datetime.date object."""
       return datetime.date.fromisoformat(val)

   def convert_datetime(val):
       """Convert ISO 8601 datetime to datetime.datetime object."""
       return datetime.datetime.fromisoformat(val)

   def convert_timestamp(val):
       """Convert Unix epoch timestamp to datetime.datetime object."""
       return datetime.datetime.fromtimestamp(val)

   sqlite3.register_converter("date", convert_date)
   sqlite3.register_converter("datetime", convert_datetime)
   sqlite3.register_converter("timestamp", convert_timestamp)


.. _sqlite3-connection-shortcuts:

Using connection shortcut methods
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Using the :meth:`~Connection.execute`,
:meth:`~Connection.executemany`, and :meth:`~Connection.executescript`
methods of the :class:`Connection` class, your code can
be written more concisely because you don't have to create the (often
superfluous) :class:`Cursor` objects explicitly. Instead, the :class:`Cursor`
objects are created implicitly and these shortcut methods return the cursor
objects. This way, you can execute a ``SELECT`` statement and iterate over it
directly using only a single call on the :class:`Connection` object.

.. literalinclude:: ../includes/sqlite3/shortcut_methods.py


.. _sqlite3-columns-by-name:

Accessing columns by name instead of by index
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

One useful feature of the :mod:`sqlite3` module is the built-in
:class:`sqlite3.Row` class designed to be used as a row factory.

Rows wrapped with this class can be accessed both by index (like tuples) and
case-insensitively by name:

.. literalinclude:: ../includes/sqlite3/rowclass.py


.. _sqlite3-connection-context-manager:

Using the connection as a context manager
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A :class:`Connection` object can be used as a context manager that
automatically commits or rolls back open transactions when leaving the body of
the context manager.
If the body of the :keyword:`with` statement finishes without exceptions,
the transaction is committed.
If this commit fails,
or if the body of the ``with`` statement raises an uncaught exception,
the transaction is rolled back.

If there is no open transaction upon leaving the body of the ``with`` statement,
the context manager is a no-op.

.. note::

   The context manager neither implicitly opens a new transaction
   nor closes the connection.

.. literalinclude:: ../includes/sqlite3/ctx_manager.py


.. _sqlite3-uri-tricks:

Working with SQLite URIs
^^^^^^^^^^^^^^^^^^^^^^^^

Some useful URI tricks include:

* Open a database in read-only mode::

    con = sqlite3.connect("file:template.db?mode=ro", uri=True)

* Do not implicitly create a new database file if it does not already exist;
  will raise :exc:`~sqlite3.OperationalError` if unable to create a new file::

    con = sqlite3.connect("file:nosuchdb.db?mode=rw", uri=True)

* Create a shared named in-memory database::

    con1 = sqlite3.connect("file:mem1?mode=memory&cache=shared", uri=True)
    con2 = sqlite3.connect("file:mem1?mode=memory&cache=shared", uri=True)
    con1.execute("create table t(t)")
    con1.execute("insert into t values(28)")
    con1.commit()
    rows = con2.execute("select * from t").fetchall()

More information about this feature, including a list of parameters,
can be found in the `SQLite URI documentation`_.

.. _SQLite URI documentation: https://www.sqlite.org/uri.html


.. _sqlite3-explanation:

Explanation
-----------

.. _sqlite3-controlling-transactions:

Transaction control
^^^^^^^^^^^^^^^^^^^

The ``sqlite3`` module does not adhere to the transaction handling recommended
by :pep:`249`.

If the connection attribute :attr:`~Connection.isolation_level`
is not :const:`None`,
new transactions are implicitly opened before
:meth:`~Cursor.execute` and :meth:`~Cursor.executemany` executes
``INSERT``, ``UPDATE``, ``DELETE``, or ``REPLACE`` statements.
Use the :meth:`~Connection.commit` and :meth:`~Connection.rollback` methods
to respectively commit and roll back pending transactions.
You can choose the underlying `SQLite transaction behaviour`_ —
that is, whether and what type of ``BEGIN`` statements ``sqlite3``
implicitly executes –
via the :attr:`~Connection.isolation_level` attribute.

If :attr:`~Connection.isolation_level` is set to :const:`None`,
no transactions are implicitly opened at all.
This leaves the underlying SQLite library in `autocommit mode`_,
but also allows the user to perform their own transaction handling
using explicit SQL statements.
The underlying SQLite library autocommit mode can be queried using the
:attr:`~Connection.in_transaction` attribute.

The :meth:`~Cursor.executescript` method implicitly commits
any pending transaction before execution of the given SQL script,
regardless of the value of :attr:`~Connection.isolation_level`.

.. versionchanged:: 3.6
   :mod:`sqlite3` used to implicitly commit an open transaction before DDL
   statements.  This is no longer the case.

.. _autocommit mode:
   https://www.sqlite.org/lang_transaction.html#implicit_versus_explicit_transactions

.. _SQLite transaction behaviour:
   https://www.sqlite.org/lang_transaction.html#deferred_immediate_and_exclusive_transactions
