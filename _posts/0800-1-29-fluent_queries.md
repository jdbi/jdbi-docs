---
layout: post
title: Fluent Queries
---

[Query](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/Query.html) instances are [SQLStatement](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/SQLStatement.html) instances specialized for SQL queries. They are created from [Handle](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/Handle.html) instances, and support fluent-style method chaining.

Queries are parameterized with the type each row in the result set will be mapped to, for example:

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");
Handle h = dbi.open();
h.execute("create table something (id int primary key, name varchar(100))");
h.execute("insert into something (id, name) values (1, 'Brian')");
h.execute("insert into something (id, name) values (2, 'Keith')");


Query<Map<String, Object>> q =
    h.createQuery("select name from something order by id");
Query<String> q2 = q.map(StringMapper.FIRST);
List<String> rs = q2.list();

assertThat(rs, equalTo(asList("Brian", "Keith")));

h.close();
{% endhighlight %}

The defauly representation of a result row is a Map&lt;String, Object&gt;, as can be seen when the Query is first created. To map the rows to something else we use a [ResultSetMapper](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/tweak/ResultSetMapper.html). A number of mappers are included, such as the [StringMapper](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/util/StringMapper.html) static instance we use here to extract a String from position 1 (the first thing) in each row.

Finally, we execute the query via one of several methods, in this case list(), which eagerly maps all rows and stores them in a List.

This example uses intermediate assignment to make clear what is happening at each step, more a more idiommatic way of writing the above would be to chain the calls to the initial Query:

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");
Handle h = dbi.open();
h.execute("create table something (id int primary key, name varchar(100))");
h.execute("insert into something (id, name) values (1, 'Brian')");
h.execute("insert into something (id, name) values (2, 'Keith')");


List<String> rs = h.createQuery("select name from something order by id")
    .map(StringMapper.FIRST)
    .list();

assertThat(rs, equalTo(asList("Brian", "Keith")));

h.close();
{% endhighlight %}

There are several other methods for executing the query, depending on how you want to retrieve the results:

{% highlight java %}
String rs = h.createQuery("select name from something order by id")
    .map(StringMapper.FIRST)
    .first();

assertThat(rs, equalTo("Brian"));
{% endhighlight %}

The first() method sets the max rows to 1, then extracts the value from just that first row of the result set.

The iterator() method works a bit differently then first() and list(). It returns an Interator which lazily traverses the result set, leaving them open until either the iterator is closed, or the end is reached.

{% highlight java %}
ResultIterator<String> rs = h.createQuery("select name from something order by id")
    .map(StringMapper.FIRST)
    .iterator();

assertThat(rs.next(), equalTo("Brian"));
assertThat(rs.next(), equalTo("Keith"));
assertThat(rs.hasNext(), equalTo(false));

rs.close();
{% endhighlight %}

Here we can see we actually get back a sub-interface of Iterator called [ResultIterator](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/ResultIterator.html) which defines a close() method. We can call the close() to close the result set and prepared statement underlying the query.

If we know we will traverse to the end of the result set, we don't need to call close() explicitely. The call to Iterator#hasNext which returns false will automatically close the results, so we can write the above as

{% highlight java %}
Iterator<String> rs = h.createQuery("select name from something order by id")
    .map(StringMapper.FIRST)
    .iterator();

assertThat(rs.next(), equalTo("Brian"));
assertThat(rs.next(), equalTo("Keith"));
assertThat(rs.hasNext(), equalTo(false));
{% endhighlight %}

Additionally, Query implements [Iterable](http://download.oracle.com/javase/6/docs/api/java/lang/Iterable.html) so we can do things like

{% highlight java %}
for (String name : h.createQuery("select name from something order by id").map(StringMapper.FIRST))
{
    assertThat(name, equalsOneOf("Brian", "Keith"));
}
{% endhighlight %}

The final means of traversing a result set is to [fold across it](http://en.wikipedia.org/wiki/Fold_(higher-order_function))

{% highlight java %}
StringBuilder rs = h.createQuery("select name from something order by id")
                    .map(StringMapper.FIRST)
                    .fold(new StringBuilder(), new Folder2<StringBuilder>()
                    {
                        public StringBuilder fold(StringBuilder acc, ResultSet rs, StatementContext ctx) throws SQLException
                        {
                            acc.append(rs.getString(1)).append(", ");
                            return acc;
                        }
                    });

rs.delete(rs.length() - 2, rs.length()); // trim the extra ", "
assertThat(rs.toString(), equalTo("Mark, Tatu"));
{% endhighlight %}

We supply an implementation of [Folder2](http://jdbi.org/maven_site/apidocs/org/skife/jdbi/v2/Folder2.html) which receives the first argument and first row to the fold() call to the initial invocation, and returns a value to be passed to the next invocation, along with the next row, and so on until the last row in the result set, the value returned from the last invocation will be returned from the fold() call.

