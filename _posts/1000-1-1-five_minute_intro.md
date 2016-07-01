---
layout: post
title: Five Minute Introduction
---

[JDBI](http://jdbi.org/) is a SQL convenience library for Java. It attempts to expose relational database access in idiomatic Java, using collections, beans, and so on, while maintaining the same level of detail as JDBC. It exposes two different style APIs, a fluent style and a sql object style.

# Fluent API
The fluent style looks like:

{% highlight java %}
// using in-memory H2 database
DataSource ds = JdbcConnectionPool.create("jdbc:h2:mem:test",
                                          "username",
                                          "password");
DBI dbi = new DBI(ds);
Handle h = dbi.open();
h.execute("create table something (id int primary key, name varchar(100))");

h.execute("insert into something (id, name) values (?, ?)", 1, "Brian");

String name = h.createQuery("select name from something where id = :id")
                    .bind("id", 1)
                    .map(StringColumnMapper.INSTANCE)
                    .first();
                    
assertThat(name, equalTo("Brian"));

h.close();
{% endhighlight %}

The DBI type is analogous to a JDBC DataSource, and will usually be constructed by passing in a JDBC DataSource. There are alternate constructors which take JDBC URL and credentials, and other means. From the DBI instance you obtain Handle instances. A Handle represents a single connection to the database. Handles rely on an underlying JDBC connection object.

With a handle you may create and execute statements, queries, calls, batches, or prepared batches. In the above example we execute a statement to define a table, execute another statement, this time with two positional arguments to insert a value, and finally construct a query, bind a value to a named argument in the query, map the results to a String, and take the first result which comes back.

The named argument facility on statements and queries is provided by JDBI -- it parses out the SQL and uses positional parameters when actually constructing the prepared statements. The above example uses the default colon-demarcated parser, but an alternative hash delimited parser is included as well for use with databases which use colons in their grammars, such as [PostgreSQL](http://www.postgresql.org/).

# SQL Object API

The second, SQL object, style API simplifies the common idiom of creating DAO objects where a single method maps to a single statement. A SQL object definition is an annotated interface, such as:

{% highlight java %}
public interface MyDAO
{
  @SqlUpdate("create table something (id int primary key, name varchar(100))")
  void createSomethingTable();

  @SqlUpdate("insert into something (id, name) values (:id, :name)")
  void insert(@Bind("id") int id, @Bind("name") String name);

  @SqlQuery("select name from something where id = :id")
  String findNameById(@Bind("id") int id);

  /**
   * close with no args is used to close the connection
   */
  void close();
}
{% endhighlight %}

This interface defines two updates, the first to create the same table as in the fluent api example, and the second to do the same insert, the third defines a query. In the second two cases, notice that the arguments to the statements are past to the method, and bound by name.

The final method, close(), is special. When it is invoked it will close the underlying JDBC connection. The method may be declared to raise an exception, such as the close() method does on java.io.Closeable, making it suitable for use with automatic resource management in Java 7.

To use this sql object definition, we use code like so:

{% highlight java %}
// using in-memory H2 database via a pooled DataSource
JdbcConnectionPool ds = JdbcConnectionPool.create("jdbc:h2:mem:test2",
                                                  "username",
                                                  "password");
DBI dbi = new DBI(ds);

MyDAO dao = dbi.open(MyDAO.class);

dao.createSomethingTable();

dao.insert(2, "Aaron");

String name = dao.findNameById(2);

assertThat(name, equalTo("Aaron"));

dao.close();
ds.dispose();
{% endhighlight %}

We obtain an instance of the sql object from the DBI instance, and then call methods on it. There are a couple different ways of creating sql object instances. The one one here binds the object to a specific handle, so we need to make sure to close the object when we are finished with it. 

