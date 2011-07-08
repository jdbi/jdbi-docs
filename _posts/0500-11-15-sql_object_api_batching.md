---
layout: post
title: SQL Object Batching
---

The SQL Object API supports prepared batch operations via the <code>@SqlBatch</code> annotation. Batch methods must return either <code>void</code> or <code>int[]</code>. In the case of an integer array, the values represent the number of rows modified for that element in the batch.

The contents of each binding in the prepared batch are determined by finding the bound arguments which are either arrays, <code>java.util.Iterable</code>, or <code>java.util.Iterator</code> instances, henceforth called iterable arguments. The various iterable arguments will be zipped together, stopping at the end of the shortest. Any non-iterable bound arguments will be considered constant values.

Take the following method as an example:
{% highlight java %}
public interface BatchExample
{
  @SqlBatch("insert into something (id, name) values (:id, :first || ' ' || :last)")
  void insertFamily(@Bind("id") List<Integer> ids,
                    @Bind("first") Iterator<String> firstNames,
                    @Bind("last") String lastName);

  @SqlUpdate("create table something(id int primary key, name varchar(32))")
  void createSomethingTable();

  @SqlQuery("select name from something where id = :it")
  String findNameById(@Bind int id);
}
{% endhighlight %}

The first method, <code>insertFamily</code>, is the batch method. It takes three arguments: the first is a list of integers which will be bound to <code>id</code>; the second is an iterator of first names, which will be bound to <code>first</code>; the final value is a constant argument which will be bound to <code>last</code>.

If we then exercise it as follows

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");
Handle h = dbi.open();

BatchExample b = h.attach(BatchExample.class);
b.createSomethingTable();

List<Integer> ids = asList(1, 2, 3, 4, 5);
Iterator<String> first_names = asList("Tip", "Jane", "Brian", "Keith", "Eric").iterator();

b.insertFamily(ids, first_names, "McCallister");

assertThat(b.findNameById(1), equalTo("Tip McCallister"));
assertThat(b.findNameById(2), equalTo("Jane McCallister"));
assertThat(b.findNameById(3), equalTo("Brian McCallister"));
assertThat(b.findNameById(4), equalTo("Keith McCallister"));
assertThat(b.findNameById(5), equalTo("Eric McCallister"));

h.close();
{% endhighlight %}

The values which will be bound for each binding into the the prepared batch are therefore

<pre>
id | first | last
------------------------
1  | Tip   | McCallister
2  | Jane  | McCallister
3  | Brian | McCallister
4  | Keith | McCallister
5  | Eric  | McCallister 
</pre>

A common case for batching is to apply bulk updates in incremental batches, say of a thousand or so rows at a time, in order to be gentle on the transaction log. The <code>@BatchChunkSize</code> annotation causes the batch to be processed in chunks of the specified size. Take the following batch method:

{% highlight java %}
public interface ChunkedBatchExample
{
  @SqlBatch("insert into something (id, name) values (:id, :name)")
  @BatchChunkSize(1000)
  void insertAll(@BindBean Iterator<Something> somethings);
}                          
{% endhighlight %}

If we a two-hundred thousand line CSV file of ids and names to insert, we could define an iterator across the rows in the file and pass that iterator to this method. The sql object would then create and execute prepared batches of 1000 rows at a time, 200 times.
