---
layout: post
title:  "Making a list"
date:   2012-06-27 00:00:00 +0100
categories: java
---
…and checking it twice?

Today just a quick post about making Lists in Java. It’s something I find myself doing quite often, especially when writing unit tests.

The classic way of intializing a List would be:

```java
Item item1 = new Item();
Item item2 = new Item();
Item item3 = new Item();

List<Item> items = new ArrayList<Item>();
items.add( item1 );
items.add( item2 );
items.add( item3 );
```

Now, a while ago I learned a new trick: the double-brace:

```java
List<Item> items = new ArrayList<Item>() {{
    add( item1 );
    add( item2 );
    add( item3 );
}};
```
This actually created an anonymous subclass of ArrayList, and allows you to call the methods straight on it.

But… the most elegant way I found actually requires nothing special, just the API:

```java
List<Item> items = Arrays.asList(
    item1, item2, item3 );
```
This takes advantage of the fact that Arrays.asList() takes its parameters as a generic var-args array.

Who said Java couldn’t be elegant every once in a while? :P
