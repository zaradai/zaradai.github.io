---
layout: post
title: Testing Annotations
category: Programming
tags: [hamcrest, tdd, unit test]
---

Recently I had a few issues with code that I thought was fully covered with unit tests.  It turns out that the problem was with an incorrect value I had
assigned to a class annotation.  This led me on a search for a suitable Hamcrest annotation matcher to immediately fix up the lack of coverage.

After exhaustive searching (caveat: my searching prowess is limited to the first page returned by Google!) I was unable to find any suitable implementation so
the rest of this blog details a solution I wrote.

##What can be tested?


In order for the annotation to be visible to the matcher it must be available at runtime. Given unit tests are executed on running code this restriction will
not affect our ability to test.  An annotation under test must have a retention policy of [RUNTIME](http://docs.oracle.com/javase/7/docs/api/java/lang/annotation/RetentionPolicy.html#RUNTIME),
the definition is

> Annotations are to be recorded in the class file by the compiler and retained by the VM at run time, so they may be read reflectively.

In addition there is one other characteristic of an annotation that determines how we are able to retrieve it reflectively, [ELEMENT_TYPE](http://docs.oracle.com/javase/7/docs/api/java/lang/annotation/ElementType.html),
which is used by the TARGET meta-annotation type to specify where it is legal to use an annotation type. The types that are runtime specific and can be tested,
are,

* TYPE - Classes, Interfaces, enums
* CONSTRUCTOR - Constructor declaration
* FIELD - Field declarations including enum constants
* METHOD - Method declarations.
* PARAMETER - Parameters on methods and constructors.

##Source Code

The source code can be found in my [github](https://github.com/zaradai/matchers) repository.

or you could import it into your project using Maven

###Maven

```xml
<dependency>
  <groupId>com.zaradai</groupId>
  <artifactId>matchers</artifactId>
  <version>0.2</version>
  <scope>test</scope>
</dependency>
```

##Using the matcher to test annotations


The best way to show how to test annotations with the matcher is to run through the development of a simple domain model object using annotation from the
[javax.persistence](http://javax.persistence) package.  The domain entity i shall use to test will be a categorization look-up object requiring 3 fields, id, name, description.

###Entity Annotation

The class should be annotated as an [entity](http://docs.oracle.com/javaee/7/api/javax/persistence/Entity.html)

The annotated class looks like :-

```java
@Entity
public class Category {

}
```

and the passing test using an annotation matcher

```java
@Test
public void shouldBeAnEntity() throws Exception {
    Category category = new Category();

    assertThat(category, is(classAnnotatedWith(Entity.class)));
}
```

###Table Annotation

The [table](http://docs.oracle.com/javaee/7/api/javax/persistence/Table.html) annotation specifies the primary table for the annotated entity and we wish to
give the table a name, "categories" so the test must ensure table annotation existence and also that it has a 'name' parameter with the value 'categories'.
Note to make the example readable I have loosened on code style best practices.

The annotated class is

```java
@Entity
@Table(name="categories")
public class Category {

}
```

and the test.

```java
@Test
public void shouldHaveTableAnnotationWithNameValue() throws Exception {
    Category category = new Category();

    assertThat(category, is(classAnnotatedWithParamValue(Table.class, "name", is("categories"))));
}
```

###Identity field

Next we wish to annotate the field that will be used as a primary key for the table.  The unit test will use a field annotation matcher.

Annotated class code

```java
@Entity
@Table(name="categories")
public class Category {
    @Id
    private Integer id;
}
```

and the test.

```java
@Test
public void shouldAnnotatePrimaryKeyField() throws Exception {
    Category category = new Category();

    assertThat(category, is(fieldAnnotatedWith(Id.class, "id")));
}
```

##Summary

Annotations are code and so must be fully tested in our unit tests.  With the annotation matcher library this becomes a straightforward task and the examples
show how to test 2 types of annotations.  There are however, other types of annotations we can test. The full list is given below along with the matcher to be
used.

| Type | with | withParam | withParamValue |
|:----:|:----:|:---------:|:--------------:|
|Class|classAnnotatedWith|classAnnotatedWithParam|classAnnotatedWithParamValue|
|Constructor|constructorAnnotatedWith|constructorAnnotatedWithParam|constructorAnnotatedWithParamValue|
|Field|fieldAnnotatedWith|fieldAnnotatedWithParam|fieldAnnotatedWithParamValue|
|Method|methodAnnotatedWith|methodAnnotatedWithParam|methodAnnotatedWithParamValue|
|Constructor Parameter|constructorParameterAnnotatedWith|constructorParameterAnnotatedWithParam|constructorParameterAnnotatedWithParamValue|
|Method Parameter|methodParameterAnnotatedWith|methodParameterAnnotatedWithParam|methodParameterAnnotatedWithParamValue|


