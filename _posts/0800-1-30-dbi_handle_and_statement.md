---
layout: post
title: DBI, Handles, and SQL Statements
---

# DBI

When starting with JDBI, the first thing you need to do is construct a [DBI](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/DBI.html) instance. The DBI instance provides connections to the database via [Handle](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/Handle.html) instances. DBIs can be constructed three primary ways.

The first is to pass a JDBC [DataSource](http://download.oracle.com/javase/6/docs/api/javax/sql/DataSource.html) instance to the constructor. In this case connections will be obtained from the datasource. This is generally the best option for cases where you want connection pooling.

The second method is to pass in a combination of a JDBC url, properties, and/or a username and password. See the constructors on [DBI](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/DBI.html) for the exact combinations. All these forms pass through to the JDBC [DriverManager](http://download.oracle.com/javase/6/docs/api/java/sql/DriverManager.html) against its matching static methods.

The third form uses a JDBI specific interface, [ConnectionFactory](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/tweak/ConnectionFactory.html) which allows for unusual ways of obtaining connections, such as for interfacing with thread-bound connection instances in Spring, or other exotic techniques.

## DBI Options

Once a DBI is obtained there are a number of configurable options on it. These include specifying the transaction handler, how externalized SQL statements are looked up, how statements are translated, configuring logging, timing collection, and registering top level customizers. All of these things will be examined in more detail in section XXXX.

# Handles

Handles can be obtained from a DBI by opening it as such:

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");
Handle handle = dbi.open();

// make sure to close it!
handle.close();
{% endhighlight %}

This requires explicitly closing the handle when you are through with it. The alternative is to pass in a callback which will receive an open handle, and will ensure it is closed when the callback completes, as follows:

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");
dbi.withHandle(new HandleCallback<Void>()
{
  public Void withHandle(Handle handle) throws Exception
  {
    handle.execute("create table silly (id int)");
    return null;
  }
});
{% endhighlight %}

Handles are used to create and execute statements, queries, calls, batches, and prepared batches. Additionally, statement customizations, result set mappers, and so on can be registered on the Handle. Anything set on the handle will override the settings inherited from the DBI, but only for that Handle and statements created from it.

# Direct Statements

The simplest way to execute statements on a handle is direct execution via the Handle#execute and Handle#query:

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");
Handle h = dbi.open();

h.execute("create table something (id int primary key, name varchar(100))");
h.execute("insert into something (id, name) values (?, ?)", 3, "Patrick");

List<Map<String, Object>> rs = h.select("select id, name from something");
assertThat(rs.size(), equalTo(1));

Map<String, Object> row = rs.get(0);
assertThat((Integer)row.get("id"), equalTo(3));
assertThat((String)row.get("name"), equalTo("Patrick"));
{% endhighlight %}

Direct statements work fine for simple DML and when SQL is not being [externalized](/externalizing_sql/), but getting back untyped maps for queries is generally annoying, particularly as different JDBC drivers will interpret ResultSet#getObject() differently, particularly for numeric types.

# Creating SQLStatements

A more sophisticated way to execute sql is to create [SQLStatement](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/SQLStatement.html) instances. Variants exist for [calls](/fluent_calls), updates, [queries](/fluent_queries/), and [prepared batches](/fluent_batches/). The general form for updates looks like:

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");
Handle h = dbi.open();
h.execute("create table something (id int primary key, name varchar(100))");

h.createStatement("insert into something(id, name) values (:id, :name)")
    .bind("id", 4)
    .bind("name", "Martin")
    .execute();

h.close();
{% endhighlight %}        

The call to Handle#createStatement creates an [Update](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/Update.html) instance, which we then bind two [named arguments](/named_parameters/) to, and finally execute. In addition to binding parameters, we can set properties on the generated statement, such as the query timeout. SQLStatement instances generally return the same instance from each call on the statement, until the statement is executed. This allows for method chaining, as above. This method chaining is by no means required, however. The above example works the same way when written as:

{% highlight java %}
Update s = h.createStatement("insert into something(id, name) values (:id, :name)");
s.bind("id", 4);
s.bind("name", "Martin");
s.execute();
{% endhighlight %}

Which you use is a matter of taste.

Queries are also SQLStatement instances, but they generally get more complicated so [have their own section](/fluent_queries/).
