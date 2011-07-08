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
  
  @SqlQuery("select name from something where id = :id")
  Query<String> queryById(@Bind("id") int id);

}
{% endhighlight %}

The first method, <code>findName</code> infers that you want only the first result, and it will be mapped into a <code>String</code>. It will take the first element in the first row of the result set to return.

The second method, <code>findNamesBetween</code> will again infer that we are looking for a String, and pull the first string from each row of the result set. Because it returns a <code>java.util.List</code> it will eagerly map each row to a String and return the full result set.

The third method, <code>findAllNames</code> does the same String extraction from each row, but because the method returns a <code>java.util.Iterator</code> it loads results lazily, only traversing the result set as <code>Iterator#next</code> or <code>Iterator#hasNext</code> is called. The iterator returned is actually a [ResultIterator](/maven_site/apidocs/org/skife/jdbi/v2/ResultIterator.html). The underlying result set will be closed when the <code>ResultIterator#close</code> method is invoked, or when the end of the result set is reached.

The final method, <code>queryById</code> returns a [Query](/maven_site/apidocs/org/skife/jdbi/v2/Query.html) instance mapped to a string. This form binds what it knows about, applies customizers, and returns the [fluent-api query](/fluent_queries/) instance which has not yet been executed.

As with String, mappings for singular primitive types in the first position of the result set are provided out of the box by JDBI. For more sophisticated mappings you can register [ResultSetMapper](/maven_site/apidocs/org/skife/jdbi/v2/tweak/ResultSetMapper.html) or [ResultSetMapperFactory](/maven_site/apidocs/org/skife/jdbi/v2/ResultSetMapperFactory.html) instances with either the DBI, the Handle, or on the sql object or individual method. Take for example the following result set mapper and class it maps to:

{% highlight java %}
public class Something
{
  private int id;
  private String name;
  
  public Something() { }

  public Something(int id, String name)
  {
    this.id = id;
    this.name = name;
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
}

public class SomethingMapper implements ResultSetMapper<Something>
{
  public Something map(int index, ResultSet r, StatementContext ctx) throws SQLException
  {
    return new Something(r.getInt("id"), r.getString("name"));
  }
}
{% endhighlight %}

We can now make use of the <code>SomethingMapper</code> in a couple ways. The first is to register it on the sql object itself:

{% highlight java %}
@RegisterMapper(SomethingMapper.class)
public interface AnotherQuery
{
  @SqlQuery("select id, name from something where id = :id")
  Something findById(@Bind("id") int id);
}
{% endhighlight %}

In this case whenever a query needs to map to <code>Something</code> it will find the registered mapper and try to use that.

Alternately, we can specify a result mapper to use for a specific method:

{% highlight java %}
public interface YetAnotherQuery
{
  @SqlQuery("select id, name from something where id = :id")
  @Mapper(SomethingMapper.class)
  Something findById(@Bind("id") int id);
}
{% endhighlight %}

You can also register result set mappers on the <code>DBI</code> or <code>Handle</code> instance the sql object is attached to.

