title: Java Mocking入门——使用Mockito
date: 2015-05-23 19:23:17
tags: 
- Java
- Mockito
comments: false
---

考虑下面这个示例模型类：

```java
package com.example.mock;

import java.util.List;

/**
 * Model class for the book details.
 */
public class Book {

    private String isbn;
    private String title;
    private List<String> authors;
    private String publication;
    private Integer yearOfPublication;
    private Integer numberOfPages;
    private String image;

    public Book(String isbn, String title, List<String> authors, String publication, Integer yearOfPublication, Integer numberOfPages, String image) {
        super();
        this.isbn = isbn;
        this.title = title;
        this.authors = authors;
        this.publication = publication;
        this.yearOfPublication = yearOfPublication;
        this.numberOfPages = numberOfPages;
        this.image = image;
    }

    public String getIsbn() {
        return isbn;
    }

    public void setIsbn(String isbn) {
        this.isbn = isbn;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public List<String> getAuthors() {
        return authors;
    }

    public void setAuthors(List<String> authors) {
        this.authors = authors;
    }

    public String getPublication() {
        return publication;
    }

    public void setPublication(String publication) {
        this.publication = publication;
    }

    public Integer getYearOfPublication() {
        return yearOfPublication;
    }

    public void setYearOfPublication(Integer yearOfPublication) {
        this.yearOfPublication = yearOfPublication;
    }

    public Integer getNumberOfPages() {
        return numberOfPages;
    }

    public void setNumberOfPages(Integer numberOfPages) {
        this.numberOfPages = numberOfPages;
    }

    public String getImage() {
        return image;
    }

    public void setImage(String image) {
        this.image = image;
    }

}

```

操作Book模型的DAL类：

```java
package com.example.mock;

import java.util.Collections;
import java.util.List;

/**
 * API layer for persisting and retrieving the Book objects.
 */
public class BookDAL {

    private static BookDAL bookDAL = new BookDAL();

    public static BookDAL getInstance() {
        return bookDAL;
    }

    public List<Book> getAllBooks() {
        return Collections.emptyList();
    }

    public Book getBook(String isbn) {
        return null;
    }

    public String addBook(Book book) {
        return book.getIsbn();
    }

    public String updateBook(Book book) {
        return book.getIsbn();
    }
}

```

当前DAL层以上没有任何功能，我们将对这段代码[TDD](http://blog.sanaulla.info/2012/04/28/test-driven-development-my-thoughts/)进行单元测试，DAL层可能与ORM映射API或者数据库API通信，而我们不关心这些API是如何设计的。

### 测试驱动DAL层

对单元测试和Java Mock有很多框架，对这个例子，我将选择JUnit做单元测试，[Mockito](https://code.google.com/p/mockito/)作为Java mock。我们会在Maven的pom.xml文件中更新依赖属性。

```xml
<project
    xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example.mock</groupId>
    <artifactId>MockSimple</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>MockSimple</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-all</artifactId>
            <version>1.9.5</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>

```

现在进行BookDAL类的单元测试。单元测试中，我们将注入mock数据到`BookDAL`类，因此我们可以不依赖数据源就可以完成API的测试。

最初我们有一个空的测试类：

```java
package com.example.mock;

public class BookDALTest {

    public static void setUp() throws Exception {
    }

    public void testGetAllBook() throws Exception {
    }

    public void testGetBook() throws Exception {
    }

    public void testAddBook() throws Exception {
    }

    public void testUpdateBook() throws Exception {
    }
}

```

我们将使用下面的`setUp()`函数把`BookDAL`类注入mock对象，并设置mock对象数据：

```java
package com.example.mock;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNotNull;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import java.util.Arrays;
import java.util.List;

import org.junit.BeforeClass;
import org.junit.Test;

public class BookDALTest {

    private static BookDAL mockedBookDAL;
    private static Book book1;
    private static Book book2;

    @BeforeClass
    public static void setUp() throws Exception {
        // Create mock object of BookDAL
        mockedBookDAL = mock(BookDAL.class);
        
        // Create few instances of Book class.
        book1 = new Book("1561451365165", "Java开发", Arrays.asList("大神1", "大神2"), "清华大学出版社", 2015, 500, "Book Image");
        book2 = new Book("1561451365166", "Android开发", Arrays.asList("大神3", "大神4"), "电子工业出版社", 2016, 700, "Book Image");

        // Stubbing the methods of mocked BookDAL with mocked data. 
        when(mockedBookDAL.getAllBooks()).thenReturn(Arrays.asList(book1, book2));
        when(mockedBookDAL.getBook("1561451365166")).thenReturn(book2);
        when(mockedBookDAL.addBook(book1)).thenReturn(book1.getIsbn());
        when(mockedBookDAL.updateBook(book2)).thenReturn(book2.getIsbn());
    }

    public void testGetAllBook() throws Exception {
    }

    public void testGetBook() throws Exception {
    }

    public void testAddBook() throws Exception {
    }

    public void testUpdateBook() throws Exception {
    }
}

```

在上面的setUp()方法中，我做了：

1. 创建一个BookDAL的mock对象

        mockedBookDAL = mock(BookDAL.class);

2. 存根带mock数据的BookDAL对象的API，这样无论何时调用API都可以返回mock数据。

    ```java
    // When getAllBooks() is invoked then return the given data and so on for the other methods.
    when(mockedBookDAL.getAllBooks()).thenReturn(Arrays.asList(book1, book2));
    when(mockedBookDAL.getBook("1561451365166")).thenReturn(book2);
    when(mockedBookDAL.addBook(book1)).thenReturn(book1.getIsbn());
    when(mockedBookDAL.updateBook(book2)).thenReturn(book2.getIsbn());
    ```

填充剩余的测试方法：

```java
package com.example.mock;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNotNull;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import java.util.Arrays;
import java.util.List;

import org.junit.BeforeClass;
import org.junit.Test;

public class BookDALTest {

    private static BookDAL mockedBookDAL;
    private static Book book1;
    private static Book book2;

    @BeforeClass
    public static void setUp() throws Exception {
        mockedBookDAL = mock(BookDAL.class);
        book1 = new Book("1561451365165", "Java开发", Arrays.asList("大神1", "大神2"), "清华大学出版社", 2015, 500, "Book Image");
        book2 = new Book("1561451365166", "Android开发", Arrays.asList("大神3", "大神4"), "电子工业出版社", 2016, 700, "Book Image");

        when(mockedBookDAL.getAllBooks()).thenReturn(Arrays.asList(book1, book2));
        when(mockedBookDAL.getBook("1561451365166")).thenReturn(book2);
        when(mockedBookDAL.addBook(book1)).thenReturn(book1.getIsbn());
        when(mockedBookDAL.updateBook(book2)).thenReturn(book2.getIsbn());
    }

    @Test
    public void testGetAllBook() throws Exception {
        List<Book> allBooks = mockedBookDAL.getAllBooks();
        assertEquals(2, allBooks.size());

        Book book = allBooks.get(0);
        assertEquals("1561451365165", book.getIsbn());
        assertEquals("Java开发", book.getTitle());
        assertEquals(2, book.getAuthors().size());
        assertEquals("清华大学出版社", book.getPublication());
        assertEquals(2015, book.getYearOfPublication().intValue());
        assertEquals(500, book.getNumberOfPages().intValue());
        assertEquals("Book Image", book.getImage());
    }

    @Test
    public void testGetBook() throws Exception {
        String isbn = "1561451365166";
        Book book = mockedBookDAL.getBook(isbn);

        assertNotNull(book);
        assertEquals("1561451365166", book.getIsbn());
        assertEquals("Android开发", book.getTitle());
        assertEquals(2, book.getAuthors().size());
        assertEquals("电子工业出版社", book.getPublication());
        assertEquals(2016, book.getYearOfPublication().intValue());
        assertEquals(700, book.getNumberOfPages().intValue());
        assertEquals("Book Image", book.getImage());
    }

    @Test
    public void testAddBook() throws Exception {
        String isbn = mockedBookDAL.addBook(book1);
        assertNotNull(isbn);
        assertEquals(book1.getIsbn(), isbn);
    }

    @Test
    public void testUpdateBook() throws Exception {
        String isbn = mockedBookDAL.updateBook(book2);
        assertNotNull(isbn);
        assertEquals(book2.getIsbn(), isbn);
    }
}

```

使用maven命令：`mvn test`进行测试。输出如下：

```
-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running com.example.mock.BookDALTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.194 sec

Results :

Tests run: 4, Failures: 0, Errors: 0, Skipped: 0

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 2.589 s
[INFO] Finished at: 2016-05-23T19:20:01+08:00
[INFO] Final Memory: 13M/154M
[INFO] ------------------------------------------------------------------------
```

至此，通过使用mock框架，在不用实际配置数据源的情况下，就可以测试DAL类。

*参考*：
> 原文 https://dzone.com/articles/getting-started-mocking-java