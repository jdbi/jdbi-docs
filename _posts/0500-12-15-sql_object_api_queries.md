---
layout: post
title: SQL Object Queries
---

Queries are denoted on sql object interfaces via the <code>@SqlQuery</code> annotation on the query methods. The return type of the method indicates what to do with the result set. Take the following queries:

{% highlight java %}
public interface SomeQueries
{
  @SqlQuery("select name from something where id = :id")
  String findName(@Bind("id") int id);

  @SqlQuery("select name from something where id > :from and id < :to")
  List<String> findNamesBetween(@Bind("from") int from, @Bind("to") int to);

  @SqlQuery("select name from something")
  Iterator<String> findAllNames();
}
{% endhighlight %}

The first method, <code>findName</code> infers that you want only the first result, and it will be mapped into a <code>String</code>. It will take the first element in the first row of the result set to return.

The second method, <code>findNamesBetween</code> will again infer that we are looking for a String, and pull the first string from each row of the result set. Because it returns a <code>java.util.List</code> it will eagerly map each row to a String and return the full result set.

The third method, <code>findAllNames</code> does the same String extraction from each row, but because the method returns a <code>java.util.Iterator</code> it loads results lazily, only traversing the result set as <code>Iterator#next</code> or <code>Iterator#hasNext</code> is called. The iterator returned is actually a [ResultIterator](/maven_site/apidocs/org/skife/jdbi/v2/ResultIterator.html). The underlying result set will be closed when the <code>ResultIterator#close</code> method is invoked, or when the end of the result set is reached.

As with String, mappings for singular primitive types are provided out of the box by JDBI. You can plug in support for additional singular (read: single-column) data types by registering [ResultColumnMapper](/maven_site/apidocs/org/skife/jdbi/v2/tweak/ResultColumnMapper.html) or [ResultColumnMapperFactory](/maven_site/apidocs/org/skife/jdbi/v2/ResultColumnMapperFactory.html) instances with either the DBI, the Handle, or on the sql object or individual method.

{% highlight java %}
public class ValueType
{
  public static ValueType valueOf(String value)
  {
    return value == null ? null : new ValueType(value);
  }

  private final String value;

  private ValueType(String value)
  {
    this.value = value;
  }

  public String getValue()
  {
    return value;
  }
}

public class ValueTypeMapper implements ResultColumnMapper<ValueType>
{
  public ValueType mapColumn(ResultSet r, int columnNumber, StatementContext ctx) throws SQLException
  {
    return ValueType.valueOf(r.getString(columnNumber)); 
  }

  public ValueType mapColumn(ResultSet r, String columnLabel, StatementContext ctx) throws SQLException
  {
    return ValueType.valueOf(r.getString(columnLabel));
  }
}
{% endhighlight %}

You can plug in support for the new mapper by registering it on the object mapper:

{% highlight java %}
@RegisterColumnMapper(ValueTypeMapper.class)
public interface ValueTypeQuery
{
  @SqlQuery("select value from thing where id = :id")
  ValueType findById(@Bind("id") int id);
}
{% endhighlight %}

Alternately, we can specify a column mapper to use for a specific method:

{% highlight java %}
public interface ValueTypeQuery
{
  @SqlQuery("select value from thing where id = :id")
  @RegisterColumnMapper(ValueTypeMapper.class)
  ValueType findById(@Bind("id") int id);
}
{% endhighlight %}

We can also register column mappers on the <code>DBI</code> or <code>Handle</code> instance the sql object is attached to. The choice of where to register a mapper depends on how widely you want it to be scoped. For example, registering a mapper on the <code>DBI</code> instance effectively makes it global, whereas registering it on a SQL object method limits the scope of the mapper to just that method.

For more sophisticated mappings that span multiple columns, you can register [ResultSetMapper](/maven_site/apidocs/org/skife/jdbi/v2/tweak/ResultSetMapper.html) or [ResultSetMapperFactory](/maven_site/apidocs/org/skife/jdbi/v2/ResultSetMapperFactory.html) instances with either the DBI, the Handle, or on the sql object or individual method. Take for example the following result set mapper and class it maps to:

{% highlight java %}
public class Something
{
  private int id;
  private String name;
  private ValueType value;
  
  public Something() { }

  public Something(int id, String name, ValueType value)
  {
    this.id = id;
    this.name = name;
    this.value = value;
  }

  public int getId()
  {
    return id;
  }

  public void setId(int id)
  {
    this.id = id;
  }

  public String getName()
  {
    return name;
  }

  public void setName(String name)
  {
    this.name = name;
  }
  
  public ValueType getValue()
  {
    return value;
  }
  
  public void setValue(ValueType value)
  {
    this.value = value;
  }
}

public class SomethingMapper implements ResultSetMapper<Something>
{
  public Something map(int index, ResultSet r, StatementContext ctx) throws SQLException
  {
    return new Something(r.getInt("id"), r.getString("name"), ValueType.valueOf(r.getString("value"));
  }
}
{% endhighlight %}

We can now make use of the <code>SomethingMapper</code> in a couple ways. The first is to register it on the sql object itself:

{% highlight java %}
@RegisterMapper(SomethingMapper.class)
public interface AnotherQuery
{
  @SqlQuery("select id, name, value from something where id = :id")
  Something findById(@Bind("id") int id);
}
{% endhighlight %}

In this case whenever a query needs to map to <code>Something</code> it will find the registered mapper and try to use that.

Alternately, we can specify a result mapper to use for a specific method:

{% highlight java %}
public interface YetAnotherQuery
{
  @SqlQuery("select id, name, value from something where id = :id")
  @RegisterMapper(SomethingMapper.class)
  Something findById(@Bind("id") int id);
}
{% endhighlight %}

You can also register result set mappers on the <code>DBI</code> or <code>Handle</code> instance the sql object is attached to.

jDBI provides a bean mapper out of the box, which maps column names to bean property names. The bean mapper delegates mapping of individual columns to the column mapper registry, so make sure to register an appropriate column mapper for any custom/non-standard data types.

{% highlight java %}
public interface OneMoreQuery
{
  @SqlQuery("select id, name, value from something where id = :id")
  @RegisterColumnMapper(ValueTypeMapper.class)
  @MapResultAsBean
  Something findById(@Bind("id") int id);
}
{% endhighlight %}

Note that the bean mapper skip properties that were not selected in the SQL query, and thus are an effective tool to retrieve a limited view of your data.
