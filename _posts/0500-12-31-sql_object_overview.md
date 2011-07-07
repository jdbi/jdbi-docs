---
layout: post
title: SQL Object API Overview
---

The SQL Object API provides a declarative mechanism for a common JDBI usage -- creation of DAO type objects where one method generally equates to one SQL statement. To use the SQL Object API, create an interface annotated to declare the desired behavior, like so:

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

You then obtain an instance of the interface from a Handle or DBI, such as:

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");

MyDAO dao = dbi.open(MyDAO.class);

// do stuff with the dao

dao.close();

{% endhighlight %}

This particular method for obtaining a sql object will open a Handle bound to the instance, such that the instance needs to be closed to release the handle and the handle's bound JDBC Connection.

Rather than use the DBI#open method, we can use one of two other mechanisms to obtain a sql object instance. The following mechanism will bind to an already opened Handle, such that it will use the handle to execute all statements. This means that the handle can be closed independently of the object, causing the object to be bound to the now closed handle (and hence not very useful).

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");
Handle h = dbi.open();
MyDAO dao = h.attach(MyDAO.class);

// do stuff with the dao
        
h.close();
{% endhighlight %}

The final method for obtaining a sql object instance will obtain and release connection automatically, as it needs to. Generally this means that it will obtain a connection to execute a statement and then immediately release it, but various things such as open transactions or iterator based results will lead to the connection remaining open until either the transaction completes or the iterated result is fully traversed.

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");
MyDAO dao = dbi.onDemand(MyDAO.class);
{% endhighlight %}

In this case we do not need to (and in fact shouldn't) ever take action to close the handle the sql object uses.
