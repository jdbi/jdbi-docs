---
layout: post
title: SQL Object Argument Binding
---

Arguments passed to properly annotated methods on sql object instances will be bound to the statements being executed. There are two binding annotations included with JDBI, <code>@Bind</code> and <code>@BindBean</code>. In addition to these binding annotations, there is a facility for defining your own.

The <code>@Bind</code> annotation binds a single named argument. If no value is specified for the annotation it will bind the argument to the name <code>it</code>. For example:

{% highlight java %}
public static interface BindExamples
{
  @SqlUpdate("insert into something (id, name) values (:id, :name)")
  void insert(@Bind("id") int id, @Bind("name") String name);

  @SqlUpdate("delete from something where name = :it")
  void deleteByName(@Bind String name);
}
{% endhighlight %}

The <code>@BindBean</code> annotation binds JavaBeans&trade; properties by name. If no value is given to the annotation the bean properties will be bound directly to their property names. If a value is given, the properties will be prefixed by the value given and a period. Take the <code>Something</code> bean, which has the properties <code>id</code> and <code>name</code> in the following example:

{% highlight java %}
public static interface BindBeanExample
{
  @SqlUpdate("insert into something (id, name) values (:id, :name)")
  void insert(@BindBean Something s);

  @SqlUpdate("update something set name = :s.name where id = :s.id")
  void update(@BindBean("s") Something something);
}
{% endhighlight %}

Finally, we can define our own binding annotations. To do this we create an annotation type annotated with the <code>@BindingAnnotation</code> annotation. This annotation requires a value, which is a <code>Class</code> implementing <code>BinderFactory</code>. The <code>BinderFactory</code> is parameterized with the annotation type and the type of argument expected. When your custom binding annotation is used, a binder factory will be created and used to create <code>Binder</code> instances which actually bind the values. Let's look at a trivial example.

{% highlight java %}
// our binding annotation
@BindingAnnotation(BindSomething.SomethingBinderFactory.class)
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.PARAMETER})
public @interface BindSomething 
{ 

  public static class SomethingBinderFactory implements BinderFactory
  {
    public Binder build(Annotation annotation)
    {
      return new Binder<BindSomething, Something>()
      {
        public void bind(SQLStatement q, BindSomething bind, Something arg)
        {
          q.bind("ident", arg.getId());
          q.bind("nom", arg.getName());
        }
      };
    }
  }
}
{% endhighlight %}

This binding annotation, <code>@BindSomething</code> is used to bind the properties of a something instance to the names <code>ident</code> and <code>nom</code>. The binding annotation is declared to be a binding annotation, using the static inner class <code>SomethingBinderFactory</code>. It is also declared to only apply to methods, and that it should be retained at runtime (the <code>@Retention(RetentionPolicy.RUNTIME)</code> annotation is really important, don't forget it).

The <code>SomethingBinderFactory</code> produces a <code>Binder</code> instance which will receive the sql statement, the annotation which lead to it being used, and the argument which it is to bind. In this case it binds values statically, but you could have any values on the annotation you like to influence how it is bound.
