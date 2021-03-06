---
layout: post
title: Unit Testing Collections
category: Programming
tags: [hamcrest, junit, mockito, tdd, unit test]
---

Whenever you manipulate collection of objects, its important to test thoroughly to ensure the expectations of the code is met.

##Briefly

[Hamcrest](http://hamcrest.org/JavaHamcrest/) provide a rich set of matchers you can use on collections in combination with *assertThat* to test the outcome of
collection operations.

[Mockito](http://mockito.org/) is an excellent mocking library that can be used to verify the interaction of your code.

This post aims to provides some useful examples with a brief explanation of their usage.

##Setup

Using Maven, pull in the required dependencies within test scope.

```xml
<dependency>
    <groupid>org.hamcrest</groupid>
    <artifactid>hamcrest-all</artifactid>
    <version>1.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupid>org.mockito</groupid>
    <artifactid>mockito-all</artifactid>
    <version>1.9.5</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupid>junit</groupid>
    <artifactid>junit</artifactid>
    <version>4.11</version>
    <scope>test</scope>
</dependency>
```

**Note:** It is important to ensure that hamcrest is inserted before junit inside your dependencies list otherwise the matchers within JUnit will be used
instead and can cause unexpected *java.lang.NoSuchMethodError* exceptions to be raised.

##Looking for a match

The strictest test for inspecting the contents of a collection is via
[contains](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html#contains(E...)).  The collection is iterated and a match is made for each
item. **Order** is important!

**Failing Test:-**

```java
List<String> colours = Lists.newArrayList("blue", "red", "gold", "orange", "pink", "yellow");

assertThat(colours, contains("blue", "gold", "orange", "pink", "red", "yellow"));
```

*Expected: iterable containing ["blue", "gold", "orange", "pink", "red", "yellow"]*
*   but: item 1: was "red"*

The test failed because red in the list is at position 1 but the assert had it placed at 4. The passing test is:-

```java
List<String> colours = Lists.newArrayList("blue", "red", "gold", "orange", "pink", "yellow");

assertThat(colours, contains("blue", "red", "gold", "orange", "pink", "yellow"));
```

If order is **not** important then one can use an alternative
[containsInAnyOrder](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html#containsInAnyOrder(java.util.Collection))

```java
List<String> colours = Lists.newArrayList("blue", "red", "gold", "orange", "pink", "yellow");

assertThat(colours, containsInAnyOrder("blue", "red", "gold", "orange", "pink", "yellow"));
```

When looking for a specific item in the collection use
[hasItem](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html#hasItem(org.hamcrest.Matcher))

```java
List<String> colours = Lists.newArrayList("blue", "red", "gold", "orange", "pink", "yellow");

assertThat(colours, hasItem("blue"));
```

When looking for more than one item use
[hasItems](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html#hasItems(org.hamcrest.Matcher...))

```java
List<String> colours = Lists.newArrayList("blue", "red", "gold", "orange", "pink", "yellow");

assertThat(colours, hasItems("blue", "pink"));</pre>
```

To test for size one can use [hasSize](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html#hasSize(int))

```java
List<String> colours = Lists.newArrayList("blue", "red", "gold", "orange", "pink", "yellow");

assertThat(colours, hasSize(6));
```

To ensure the collection is empty use [empty](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html#empty()) whilst for iterables use
[emptyIterable](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html#emptyIterable())

```java
List<String> colours = Lists.newArrayList();

assertThat(colours, empty());
assertThat(colours, emptyIterable());
```

For maps there are a number of specialized *has* matchers [hasEntry](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html#hasEntry(K, V)),
 [hasKey](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html#hasKey(K)),
[hasValue](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html#hasValue(org.hamcrest.Matcher)) to deal with key/value and entries, e.g.
the following test will check to see if the map contains a specific value.

```java
Map<Integer, String> colourLikes = Maps.newHashMap();

colourLikes.put(42, "red");
colourLikes.put(24, "blue");

assertThat(colourLikes, hasValue("blue"));
```

##Matcher lookup

Quick lookup table for available collection matchers

| Test Condition | Matcher |
|:--------------:|:-------:|
|contains all items in order|contains|
|contains all items in any order|containsInAnyOrder|
|contains an item |  hasItem |
|contains multiple items |  hasItems |
|does not contain an item | not(hasItem |
|does not contain items | not(hasItems |
|empty collection |  empty |
|empty iterable | emptyIterable  |
|size of collection |  hasSize  |
|size of iterable |  iterableWithSize |
|all items match a specific condition | everyItem |
|Map contains an entry |  hasEntry |
|Map contains a key |   hasKey    |
|Map contains a value | hasValue |


##Testing Interactions

There are occasions where one needs to ensure a number of operations occur on an collection to fully cover the code.  A particular test to illustrate testing
collection interactions using the Mockito mocking library is an atomic put operation on a concurrent map.

The requirement of the code is to track the trading position for a number of accounts with the method under test returning a position object for a given
account id.  The method is thread safe and guaranteed to return a position.  The code being tested is :-

```java
public class PositionBook {
    private final Map<String, Position> positionByAccountId;

    @Inject
    PositionBook() {
        positionByAccountId = createPositionMap();
    }

    protected Map<String, Position> createPositionMap() {
        return Maps.newConcurrentMap();
    }

    public Position getPositionForAccount(String accountId) {
        ConcurrentMap<String, Position> map = (ConcurrentMap<String, Position>)getPositionByAccountMap();

        Position position = map.get(accountId);

        if (position == null) {
            position = createPosition(accountId);
            // add atomically
            Position previous = map.putIfAbsent(accountId, position);

            if (previous != null) {
                // some other thread already beat us to it so return this position.
                return previous;
            }
        }

        return position;
    }

    private Position createPosition(String accountId) {
        return new Position(accountId);
    }

    private Map<String, Position> getPositionByAccountMap() {
        return this.positionByAccountId;
    }
}
```

The most difficult part of the method is to test the state whereby another thread has inserted a position whilst the current thread is trying to insert.  The
key is [putIfAbsent](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ConcurrentMap.html#putIfAbsent(K,%20V)) which guarantees insertion atomicity
by returning an existing entry if one exists.

To test, I need to manipulate the code so that a value is returned when putIfAbsent is called.  I can then assert the returned value.  To do this the
collection is mocked using Mockito and prepared appropriately to return the value.

The unit test is then created thus:

```java
@Mock
Map<String, Position> mockedMap;

private PositionBook positionBook;

@Before
setUp() throws Exception {
    // Ensure the map gets mocked
    MockitoAnnotations.initMocks(this);
    // create the object to test
    positionBook = new PositionBook() {
        @Override
        protected Map<String, Position. createPositionMap() {
            return mockedMap;
        }
    };
}

@Test
public void shouldReturnExistingPositionIfNotAbsent() throws Exception {
    String accountId = "test";
    Position existing = mock(Position.class);
    // setup the mocked map object to return the position
    when(mockedMap.putIfAbsent(anyString(), any(Position.class))).thenReturn(existing);
    // test the method
    Position res = positionBook.getPositionForAccount(accountId);

    assertThat(res, is(existing));
}
```

1. Using @Mock annotation, build the mocked map.
2. Within setUp create an anonymous class that overrides the map creation so that our mocked map is used by the function under test.
3. Prepare the mock's behaviour to exhibit our desired test scenario.
4. Finally assert that we are returned the correct position.

