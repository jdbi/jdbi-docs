---
layout: post
title: SQL Object Data Manipulation
---

Update, insert, and data definition statements are indicated in the SQL Object API via the <code>@SqlUpdate</code> annotation. The methods for these statements must have either <code>void</code> or <code>int</code> return types. If the return type is <code>int</code>, then the value will be the number of rows changed. Alternatively, if the method is annotated with <code>@GetGeneratedKeys</code>, then the return value with be the auto-generated keys.

Update methods look like

{% highlight java %}
public static interface Update
{
  @SqlUpdate("create table something (id integer primary key, name varchar(32))")
  void createSomethingTable();

  @SqlUpdate("insert into something (id, name) values (:id, :name)")
  int insert(@Bind("id") int id, @Bind("name") String name);

  @SqlUpdate("update something set name = :name where id = :id")
  int update(@BindBean Something s);
}
{% endhighlight %}

As with [queries](/sql_object_api_queries/), arguments to the statements are [bound from arguments to the method](/sql_object_api_argument_binding). 

We can exercise this code by creating and instance and just calling the methods:

{% highlight java %}
DBI dbi = new DBI("jdbc:h2:mem:test");
Handle h = dbi.open();
        
Update u = h.attach(Update.class);
u.createSomethingTable();
u.insert(17, "David");
u.update(new Something(17, "David P."));

String name = h.createQuery("select name from something where id = 17")
    .map(StringMapper.FIRST)
    .first();
assertThat(name, equalTo("David P."));

h.close();
{% endhighlight %}
