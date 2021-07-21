# java8Migration

Java Guides

Java 8 Tutorial
1. Overview
In this post, we will discuss how to migrate existing code base in Java 6 or 7 to Java 8.
Keeping your code up to date with the latest versions of languages and libraries is a challenging task. Java SE 8 brings entire new concepts to the language, like lambda expressions, and adds new methods to classes that developers have been using comfortably for years. In addition, there are new ways of doing things, including the new Date and Time API, and an Optional type to help with null-safety.
In this post, we will give you some tips and tricks to migrate your code from Java 6 (or 7) to Java 8. If you are new to Java 8 then do some initial setup to compile and run your projects on JDK 8.
Initial setup
Make sure you're compiling with a Java 8 JDK.
Pick a section of the codebase to apply them to.
In the beginning, pick a small number of changes to implement.
2. Migrating Source Code to Java 8 Tips
How to use Lambda expressions?
The first step is to identify anonymous inner classes in your project like for example :
Runnable
Callable
Comparator
FileFilter
PathMatcher
EventHandler, etc
For example, you may come across a Runnable anonymous inner class:
executorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        getDs().save(new CappedPic(title));
    }
}, 0, 500, MILLISECONDS);
Replace above statement with lambda expression like
executorService.scheduleAtFixedRate(() -> getDs()
.save(new CappedPic(title)), 0, 500, MILLISECONDS);
We suggest you read Java 8 Lambda Expressions post and refer the examples and apply the same in your projects.
Understand the impact of applying lambda expressions
It is important to understand the impact while changing the code right. I found there could be two impacts
Larger anonymous inner classes may not be very readable in a lambda form.
There may be additional changes and improvements you can make.
Let's address both points with an example.
We might be using a Runnable to group a specific set of assertions in our test:
Runnable runnable = new Runnable() {
    @Override
    public void run() {
        datastoreProvider.register(database);
        Assert.assertNull(database.find(User.class, "id", 1).get());
        Assert.assertNull(database.find(User.class, "id", 3).get());

        User foundUser = database.find(User.class, "id", 2).get();
        Assert.assertNotNull(foundUser);
        Assert.assertNotNull(database.find(User.class, "id", 4).get());
        Assert.assertEquals("Should find 1 friend", 1, foundUser.friends.size());
        Assert.assertEquals("Should find the right friend", 4, foundUser.friends.get(0).id);
    }
};
Converting this to a lambda results in:
Runnable runnable = () -> {
    datastoreProvider.register(database);
    Assert.assertNull(database.find(User.class, "id", 1).get());
    Assert.assertNull(database.find(User.class, "id", 3).get());

    User foundUser = database.find(User.class, "id", 2).get();
    Assert.assertNotNull(foundUser);
    Assert.assertNotNull(database.find(User.class, "id", 4).get());
    Assert.assertEquals("Should find 1 friend", 1, foundUser.friends.size());
    Assert.assertEquals("Should find the right friend", 4, foundUser.friends.get(0).id);
};
This is not much shorter, nor does it impact readability much.
In cases like these, you may refactor the code by using an extract method technique to pull these lines into a single method instead:
Runnable runnable = () -> {
    assertUserMatchesSpecification(database, datastoreProvider);
};
You can simplify the above code, just convert to a single statement.
Runnable runnable = () -> assertUserMatchesSpecification(database, datastoreProvider);
Once you've changed your anonymous inner classes to lambdas and made any manual adjustments you might want to make, like extracting methods or reformatting the code, if you have JUnit test cases then run it and test the result.
How to use new Collection Methods?
Java 8 introduced a new way of working with collections of data, through the Streams API. For example, java.util.Iterable has a forEach method that lets you pass in a lambda that represents an operation to run on every element.
Find the for loop statements in your project, for example :
for (Person person : listOfPerson) {
      System.out.println(" Person name : " + person.getName());
}
Replace above piece of code with a lambda expression as :
listOfPerson.forEach((person) -> System.out.println(" Person name : " + person.getName()));
If you find for loop like this :
for (Class<? extends Annotation > annotation : INTERESTING_ANNOTATIONS) {
    addAnnotation(annotation);
}
Use Method Reference rather than a lambda. Method references are another new features in Java 8, which can generally be used where a lambda expression would usually call a single method.
INTERESTING_ANNOTATIONS.forEach((annotation) -> addAnnotation(annotation));
Lets see some more complex for loop statements and convert to forEach Stream API.
Stream API - forEach
The Streams API is a powerful tool for querying and manipulating data and using it could significantly change and simplify the code you write.
What does the Streams API give us that we can't simply get from using a forEach method? Let's look at an example that's slightly more complicated for loop than the previous one:
public void addAllBooksToLibrary(Set<Book> books) {
    for (Book book: books) {
        if (book.isInPrint()) {
            library.add(book);
        }
    }
}
Firstly the loop body checks some condition, then does something with the items that pass that condition. Selecting the fix Replace with forEach will use the Streams API to do the same thing:
public void addAllBooksToLibrary(Set <Book> books) {
    books.stream()
         .filter(book -> book.isInPrint())
         .forEach(library::add);
}
Replace lambda with method references
books.stream()
     .filter(Book::isInPrint)
     .forEach(library::add);
Streams API - collect
It's very common to see a for loop that iterates over some collection, performs some sort of filtering or manipulating, and outputs the results into a new collection, and that's the sort of code this inspection will identify and migrate to using the Streams API.
If you found for loop in your project like
List <ProjectEntity> listOfProjects = getProjects();
List <ProjectDTO> ProjectDTOs= new ArrayList<ProjectDTO>();
for (ProjectEntity projectEntity : listOfProjects ) {
    ProjectDTOs.add(projectEntity .getId());
}
Apply the Replace with a collect fix to turn this code into:
List<ProjectDTO> projectDTOs= listOfProjects.stream().map(PrjectDTO::getId).collect(Collectors.toList());
How to use Optional Class?
The java.util.Optional gives you a way to handle null values, and a way to specify if a method call is expected to return a null value or not.
If you see "Assignment to null" for fields, you may want to consider turning this field into an Optional. For example, in the code below, the line where an offset is assigned will be flagged:
private Integer offset;
public Builder offset(int value) {
    offset = value > 0 ? value : null;
    return this;
}

// more code...
That's because in another method, the code checks to see if this value has been set before doing something with it:
if (offset != null) {
    cursor.skip(offset);
}
Let's see how to replace the above piece of code with Optional.
private Optional<Integer> offset;
public Builder offset(int value) {
    offset = value > 0 ? Optional.of(value) : Optional.empty();
    return this;
}

// more code...
Then you can use the methods on Optional instead of performing null-checks. The simplest solution is:
if (offset.isPresent()) {
    cursor.skip(offset);
}
But it's much more elegant to use a Lambda Expression to define what to do with the value:
offset.ifPresent(() -> cursor.skip(offset));
If you have code looks like
public Customer findFirst() {
    if (customers.isEmpty()) {
        return null;
    } else {
        return customers.get(0);
    }
}
We could alter this method to return an Optional of Customer:
public Optional<Customer> findFirst() {
    if (customers.isEmpty()) {
        return Optional.empty();
    } else {
        return Optional.ofNullable(customers.get(0));
    }
}
If you found the code in your project looks like :
Customer firstCustomer = customerDao.findFirst();
if (firstCustomer == null) {
    throw new CustomerNotFoundException();
} else {
    firstCustomer.setNewOffer(offer);
}
Return the Optional and now we are returning an Optional, we can eliminate the null check:
Optional<Customer> firstCustomer = customerDao.findFirst();
firstCustomer.orElseThrow(() -> new CustomerNotFoundException())
             .setNewOffer(offer);
Simple use Cases
There are areas where Java 8 provides very nice solutions, e.g. simple mapping between DTOs and business objects without using Java 8.
final List<CommissionDto> commissionDtos = Lists.newLinkedList();
for (CassandraOfferCommission cassandraOfferCommission : cassandraOfferCommissions) {
    commissionDtos.add(buildCommissionDto(cassandraOfferCommission));
}
return commissionDtos;
Using Java 8 features the above code becomes:
return cassandraOfferCommissions.stream().map(this::buildCommissionDto).collect(toList());
We found cases where simple, textbook-like functional transformations perfectly matched our needs. Here is a piece of code that traverses a sorted list of price Ranges matching a criterion and returns the first matching one or throws an exception when there isn’t any:
return getRanges().stream()
        .filter(range -> range.containsPrice(amount))
        .findFirst()
        .orElseThrow(() -> new RangeNotFoundException(String.format("No matching range found for amount %s", amount)));
But there were places where the API felt to be missing just a little bit of something to be fully satisfying. Transforming maps is painful:
quoteDetails.entrySet().stream().collect(toMap(Entry::getKey, e -> e.getValue().getFee()))

The source code of this post is available on GitHub Repository.


