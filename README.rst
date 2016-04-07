=======
PyMySQL
=======

.. image:: https://travis-ci.org/PyMySQL/PyMySQL.svg?branch=master
   :target: https://travis-ci.org/PyMySQL/PyMySQL

.. image:: https://coveralls.io/repos/PyMySQL/PyMySQL/badge.svg?branch=master&service=github
   :target: https://coveralls.io/github/PyMySQL/PyMySQL?branch=master

.. contents::

This package contains a pure-Python MySQL client library. The goal of PyMySQL
is to be a drop-in replacement for MySQLdb and work on CPython, PyPy and IronPython.

Tips
-------------
如源码：
.. code:: python

self.rowcount = sum(self.execute(query, arg) for arg in args)

executemany是对execute的包装，并非multiple rows，因此在做insert操作的时候，数据量稍大些，性能就变得很差。之前一直没发现这个问题，以为executemany就是批量入库，从而浪费了不少时间与资源。
而如果利用execute并发挥MySQL的multiple rows作用，同样12万条数据入库，能从近6000秒提升至20秒。

优化前：
.. code:: python

sql = "INSERT INTO mtable(field1, field2, field3...) VALUES (%s, %s, %s...)"
for item in datas:
  batch_list.append([v1, v2, v3...])
  # 批量插入
  if len(batch_list) == common.mysql_batch_num:
      cur.executemany(sql, batch_list)
      conn.commit()
      batch_list = []
      counts += len(batch_list)
      common.print_log("inserted:" + str(counts))

优化后：
.. code:: python

sql = "INSERT INTO mtable(field1, field2, field3...) VALUES (%s, %s, %s...)"
for item in datas:
  batch_list.append(common.multipleRows([v1, v2, v3...]))
  # 批量插入
  if len(batch_list) == common.mysql_batch_num:
      sql = "INSERT INTO mtable(field1, field2, field3...) VALUES %s " % ','.join(batch_list)
      cur.execute(sql)
      conn.commit()
      batch_list = []
      counts += len(batch_list)
      common.print_log("inserted:" + str(counts))
      
.. code:: python

# 返回可用于multiple rows的sql拼装值
def multipleRows(params):
    ret = []
    # 根据不同值类型分别进行sql语法拼装
    for param in params:
        if isinstance(param, (int, long, float, bool)):
            ret.append(str(param))
        elif isinstance(param, (str, unicode)):
            ret.append('"' + param + '"')
        else:
            print_log('unsupport value: %s ' % param)
    return '(' + ','.join(ret) + ')'

Requirements
-------------

* Python -- one of the following:

  - CPython_ >= 2.6 or >= 3.3
  - PyPy_ >= 4.0
  - IronPython_ 2.7

* MySQL Server -- one of the following:

  - MySQL_ >= 4.1  (tested with only 5.5~)
  - MariaDB_ >= 5.1

.. _CPython: http://www.python.org/
.. _PyPy: http://pypy.org/
.. _IronPython: http://ironpython.net/
.. _MySQL: http://www.mysql.com/
.. _MariaDB: https://mariadb.org/


Installation
------------

The last stable release is available on PyPI and can be installed with ``pip``::

    $ pip install PyMySQL

Alternatively (e.g. if ``pip`` is not available), a tarball can be downloaded
from GitHub and installed with Setuptools::

    $ # X.X is the desired PyMySQL version (e.g. 0.5 or 0.6).
    $ curl -L https://github.com/PyMySQL/PyMySQL/tarball/pymysql-X.X | tar xz
    $ cd PyMySQL*
    $ python setup.py install
    $ # The folder PyMySQL* can be safely removed now.

Test Suite
----------

If you would like to run the test suite, create database for test like this::

    mysql -e 'create database test_pymysql  DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;'
    mysql -e 'create database test_pymysql2 DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;'

Then, copy the file ``.travis.databases.json`` to ``pymysql/tests/databases.json``
and edit the new file to match your MySQL configuration::

    $ cp .travis.databases.json pymysql/tests/databases.json
    $ $EDITOR pymysql/tests/databases.json

To run all the tests, execute the script ``runtests.py``::

    $ python runtests.py

A ``tox.ini`` file is also provided for conveniently running tests on multiple
Python versions::

    $ tox


Example
-------

The following examples make use of a simple table

.. code:: sql

   CREATE TABLE `users` (
       `id` int(11) NOT NULL AUTO_INCREMENT,
       `email` varchar(255) COLLATE utf8_bin NOT NULL,
       `password` varchar(255) COLLATE utf8_bin NOT NULL,
       PRIMARY KEY (`id`)
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
   AUTO_INCREMENT=1 ;


.. code:: python

    import pymysql.cursors

    # Connect to the database
    connection = pymysql.connect(host='localhost',
                                 user='user',
                                 password='passwd',
                                 db='db',
                                 charset='utf8mb4',
                                 cursorclass=pymysql.cursors.DictCursor)

    try:
        with connection.cursor() as cursor:
            # Create a new record
            sql = "INSERT INTO `users` (`email`, `password`) VALUES (%s, %s)"
            cursor.execute(sql, ('webmaster@python.org', 'very-secret'))

        # connection is not autocommit by default. So you must commit to save
        # your changes.
        connection.commit()

        with connection.cursor() as cursor:
            # Read a single record
            sql = "SELECT `id`, `password` FROM `users` WHERE `email`=%s"
            cursor.execute(sql, ('webmaster@python.org',))
            result = cursor.fetchone()
            print(result)
    finally:
        connection.close()

This example will print:

.. code:: python

    {'password': 'very-secret', 'id': 1}


Resources
---------

DB-API 2.0: http://www.python.org/dev/peps/pep-0249

MySQL Reference Manuals: http://dev.mysql.com/doc/

MySQL client/server protocol:
http://dev.mysql.com/doc/internals/en/client-server-protocol.html

PyMySQL mailing list: https://groups.google.com/forum/#!forum/pymysql-users

License
-------

PyMySQL is released under the MIT License. See LICENSE for more information.
