# Unit 2, Lesson 5 - The Powerful LoadDocument Server-Side Function

It's time to learn about one of the most tricky features of RavenDB: The
server-side `LoadDocument` function.

## Something about size!

In the [previous lesson](../lesson4/README.md) you wrote this code:

````csharp
static void Main(string[] args)
{
    using (var session = DocumentStoreHolder.Store.OpenSession())
    {
        var results = session
            .Query<Products_ByCategory.Result, Products_ByCategory>()
            .Include(x => x.Category)
            .ToList();

        foreach (var result in results)
        {
            var category = session.Load<Category>(result.Category);
            Console.WriteLine($"{category.Name} has {result.Count} items.");
        }
    }
}
````

As you learned in [unit 1, lesson 4](../../Unit-1/lesson4/README.md) the `Include` method
ensures that all related documents will be returned in a single response from the server, and
this is amazing.

As you probably deduced, RavenDB uses HTTP as the communication protocol. The client API uses the POST method to issue the query request.
However, GET requests are still handled by the server. Here is the URL to execute the query of the code above:

<div style="font-family: monospace; word-wrap: break-word; word-break: break-all;">http://localhost:8080/databases/Northwind/queries?query=from%20index%20%27Products/ByCategory%27%20include%20Category</div>

Click [here](http://localhost:8080/databases/Northwind/queries?query=from%20index%20%27Products/ByCategory%27%20include%20Category) and
check out this query result.

What you get when you execute this query is a big JSON object. The nice thing is that you get all the data you need with a single request. The bad thing is you receive a lot more than you need.
In the example, all related category documents will be present in the response, but you will use just one property (please, keep in mind that
Category is a small document ...).

What if you could load the documents in the server-side to produce only the information you need?

## Welcome `LoadDocument`

When writing your index definitions, you can use the `LoadDocument` function to get information from related documents.

Let's rewrite the `Products_ByCategory` index using the `LoadDocument` function:

````csharp
public class Products_ByCategory :
    AbstractIndexCreationTask<Product, Products_ByCategory.Result>
{
    public class Result
    {
        public string Category { get; set; }
        public int Count { get; set; }
    }

    public Products_ByCategory()
    {
        Map = products =>
            from product in products
            let categoryName = LoadDocument<Category>(product.Category).Name
            select new
            {
                Category = categoryName,
                Count = 1
            };

        Reduce = results =>
            from result in results
            group result by result.Category into g
            select new
            {
                Category = g.Key,
                Count = g.Sum(x => x.Count)
            };
    }
}
````

Now we are no longer storing the category `Id`, but the `Name`. Now we
can rewrite our program with no includes.

````csharp
static void Main(string[] args)
{
    using (var session = DocumentStoreHolder.Store.OpenSession())
    {
        var query = session
            .Query<Products_ByCategory.Result, Products_ByCategory>();
            //.Include(x => x.Category);

        var results = (
            from result in query
            select result
            ).ToList();

        foreach (var result in results)
        {
            //var category = session.Load<Category>(result.Category);
            //Console.WriteLine($"{category.Name} has {result.Count} items.");
            Console.WriteLine($"{result.Category} has {result.Count} items.");
        }
    }
}
````

We will still have only one request...

<div style="font-family: monospace; word-wrap: break-word; word-break: break-all;">http://localhost:8080/databases/Northwind/queries?query=from%20index%20%27Products/ByCategory%27</div>

... but now we will have a smaller response.

## It's really good, but...

The `LoadDocument` feature is a really awesome one, but it is also something that should
be used carefully.

As you know, `LoadDocument` allows you to load another document during indexing, and to use its data in your index and that is great!
The problem with `LoadDocument` is that it allows users to keep a relational
model when they work with RavenDB, and use `LoadDocument` to get away with
it when they need to do something that is hard to do with RavenDB natively.
That wouldn’t be so bad if `LoadDocument` didn’t have several important costs
associated with it. 

`LoadDocument` is an important feature and you could and should use it. But
I strongly recommend you to treat `LoadDocument` with caution.   

## Great job!

Before moving on, I would recommend you to read about the [indexing process for related documents](https://ravendb.net/docs/article-page/4.0/csharp/indexes/indexing-related-documents).

**Let's move onto [Lesson 6](../lesson6/README.md).**
