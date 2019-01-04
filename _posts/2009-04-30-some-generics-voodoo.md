---
layout: post
title:  "Some Java Generics Voodoo… no zombies involved."
date:   2009-04-30 00:00:00 +0100
categories: blog
tags: [development,generics,hibernate,java,spring]
---
In a perfect world, I’d be doing all my coding in Groovy… but since that isn’t happening any time soon, I’ve found that Generics are a good way of dealing with statically typed languages.

Specifically when implementing DAOs with Spring and Hibernate, Generics can save on a lot of duplicated code. I’d like to share a few things I’ve learned on how to use Generics…

A lot of the work I do involves the classic combo of Spring and Hibernate. Generally this means definining DAO objects for all your entities (business objects). In a pre-1.5 world you’d have to do something like this:

```java
public interface CustomerDao {
    Customer findById(Long id);
 
    Collection findAll();
 
    void save( Customer customer );
}
```

And then for another entity:

```java
public interface AccountDao {
    Account findById(Long id);
 
    Collection findAll();
 
    void save( Account account );
}
```

Anyway, you get the idea. It’s pretty obvious that this doesn’t mesh well with the DRY principle: the interfaces only differ on details.

Lucky for us, Java 1.5 brought a solution in the form of Generics:

```java
public interface Dao<T> {
 
    T findById(Long id);
 
    Collection<T> findAll();
 
    void save( T t );
}
```
There… that’s a lot cleaner already. All is well, right? Not quite… the next step of course is actually implementing the interface. Let’s start again with what it would have looked like in the 1.4 days. This implementation uses the Spring HibernateDaoSupport class:

```java
public class CustomerDaoHibernate extends HibernateDaoSupport
    implements CustomerDao {
 
    public Customer findById(Long id) {
        return (Customer) getHibernateTemplate().get(Customer.class, id);
    }
 
    public Collection findAll() {
        return getHibernateTemplate().loadAll(Customer.class );
    }
 
    public void save( Customer customer ) {
        getHibernateTemplate().save( customer );
    }
}
```
Rince and repeat for AccountDaoHibernate. Now, the Java 1.5 version:

```java
public class DaoHibernate<T> extends HibernateDaoSupport
    implements Dao<T> {
 
    public T findById(Long id) {
        return (T) getHibernateTemplate().get( //... We need a class here!
    }
 
    public Collection findAll() {
        //Same problem here!
    }
 
    public void save( T t ) {
        getHibernateTemplate().save( t );
    }
}
```
Hmmm… we have a problem here. We have a type parameter T, but there is no way to get the actual class object we need to pass to HibernateTemplate from this parameter. So, where do we get that? We add a special constructor:

```java
public class DaoHibernate<T> extends HibernateDaoSupport
    implements Dao<T> {
 
    private Class<T> clazz;
 
    public CustomerDaoHibernate( Class<T> clazz ) {
        this.clazz = clazz;
    }
 
    public T findById(Long id) {
        return (T) getHibernateTemplate().get(clazz, id);
    }
 
    public Collection<T> findAll() {
        return (Collection<T>) getHibernateTemplate().loadAll(clazz);
    }
 
    public void save( T t ) {
        getHibernateTemplate().save( t );
    }
}
```

Now, isn’t that a problem? It would seem that you can pass in any class object here, but that’s not true. Since the class Class itself is parameterized, you can only use the same class here as the one you used to parameterize the DAO. Still with me?

An example:

```java
//This is valid
Dao<Customer> customerDao = new DaoHibernate<Customer>( Customer.class );
 
//This will produce a compiler error
Dao<Customer> secondCustomerDao = new DaoHibernate<Customer>( Account.class );
```

This feels kind of clunky though… having to pass in the same class each time. Plus, if you want to be able to use these DAO objects as Spring beans, you’ll probably end up doing something like this:

```java
public class CustomerDaoHibernate extends DaoHibernate<Customer> {
 
    public CustomerDaoHibernate() {
        super( Customer.class );
    }
    ....
}
```

and:

```java
public class AccountDaoHibernate extends DaoHibernate<Account> {
 
    public AccountDaoHibernate() {
        super( Account.class );
    }
}
```

This was the solution I used for a long time. Recently I found out though that you can’t just parameterize classes, you can parameterize methods too!
Now that offers some interesting possibilities:

 
 ```java
public interface DaoFactory {
 
    <T> Dao<T> getDao( Class<T> clazz );
}
```

Now, this had me scratching my head for a bit at first. But what it means is that the T parameter from the Class object you pass in, then also becomes the parameter for the return type. So, if I call this method with Customer.class, I’m actually saying:

```java
Dao<Customer> getDao( Customer.class );
```

Now, the beauty here is that the DaoFactory class itself isn’t parameterized. It’s just a POJO with a default constructor, which can be used as a Spring bean without any problems. It also provides a nice implementation for the AbstractFactory pattern, since I could write:

```java
public class HibernateDaoFactory implements DaoFactory {
 
    private SessionFactory sessionFactory;
 
    public <T> Dao<T> getDao(Class<T> c) {
        HibernateDao<T> dao = new HibernateDao<T>(c);
        dao.setSessionFactory(this.sessionFactory);
        return dao;
    }
 
    public void setSessionFactory(SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }
}
```

This provides an AbstractFactory for Hibernate DAO implementations. But, I could just as easily write a factory that produces MemoryDao objects (which use a HashSet for ‘persistency’).

Now, to finish it all up: I started out by saying that one of the reasons for using this pattern is the ability to use DAOs from a Spring context. There are 2 ways in which this can be done:

  * Inject the factory itself as a dependency, and let it generate DAOs on the fly when needed
  * Use the Spring factory-method setting

The first option should be pretty obvious, so I’ll only illustrate the second one here:
 
```xml
<!-- the factory bean -->
<bean id="myFactoryBean" class="net.nightwhistler.example.HibernateDaoFactory" />
 
<!-- our DAO, to be created via the factory bean -->
<bean id="customerDao"
      factory-bean="myFactoryBean"
      factory-method="getDao"/>
 
    <constructor-arg value="net.nightwhistler.domain.Customer" />
</bean>
```

This allows you to wire fully-parameterized beans, without coding 😃 Isn’t life grand?
