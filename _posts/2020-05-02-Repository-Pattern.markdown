---
layout: post
title:  Repository Pattern
date:   2020-05-14 21:30:20 -0400
description: Describing Repository Pattern in C#
img: repository-pattern.png
tags: [Blog, Repository Pattern]
author: Zyad Alyashae
published: true
---
Why is the repository pattern still being talked about in 2020? Surely there are better alternatives out there using ORMs like EF (Entity Framework) & Dapper. You have to think really hard to find answers and with most answers, you'd be walking on thin ice. In most situations, you'd be looking towards just using an ORM. There are times where both EF and a repository are utilized, although this is seen as an anti-pattern that causes more confusion than intended - why build an extra abstraction on top of EF that which already handles it? One of the only logical answers to using a repository is for testing purposes. By breaking the design into multiple repositories, you can further mock and unit test business logic without relying on integration tests. Anyways, the point of this article isn't to fuel a war between one or the other, but rather to show the use of the pattern and allow you to decide whether it's useful for the project at hand.

I'll be going through the below design diagram, from top to bottom, to create an inventory REST API in order to understand the use behind the repository pattern.
It is highly recommended to utilize IoC through dependency injection while using repository pattern. I'll go through this step too!

![Design Diagram]({{site.baseurl}}/assets/img/repository-pattern.png)

# Controller (presentation layer)
This is the entry point to the API. Here is where all the routes lead to ;) (sorry, the dad joke was oozing to come out). I won't go into detail regarding this section as it does not really change whether you are using EF or the repository pattern.

{% highlight csharp %}
// Dependency Injection
private IInventoryRepository inventoryRepository;

public ItemController(IInventoryRepository inventoryRepository)
{
    this.inventoryRepository = inventoryRepository;
}

[HttpPost]
[Route("")]
public HttpResponseMessage AddItem(Item item)
{
    if (item == null)
    {
        return new HttpResponseMessage(HttpStatusCode.BadRequest);
    }

    ItemDTO itemDTO = itemRepository.AddItem(item.ToCommand());

    if (itemDTO == null)
    {
        return new HttpResponseMessage(HttpStatusCode.BadRequest);
    }
    else
    {
        return Request.CreateResponse(HttpStatusCode.OK, itemDTO);
    }
}
{% endhighlight %}

> Using AutoMapper, found in NuGet, we can map the ItemDTO to the Item Model. This allows for breakage between the business and application logic.

As you can see, I've kept the controller fairly easy to use and understand. It simply fails fast if needed before calling further layers, utilizes the service layer to prepare the model then calls the repository layer for application level work.

# Service Layer

This layer is meant to be very simple, in our case it will merely translate business logic to application models. I used AutoMapper, which you can install from NuGet, to do the mapping. I have two models, **Item** which holds business layered logic & **ItemDTO** which we use in the application layer.
Another reason is because we may use properties in the DTO for data processing that should not be upstreamed to the presentation domain and vice-versa.

{% highlight csharp %}
public class Item
{
    // Calling this function will map the matching properties between Item & ItemDTO 
    public ItemDTO ToCommand()
    {
        return Mapper.Map<ItemDTO>(this);
    }

    [JsonProperty]
    public string Name { get; set; }
    [JsonProperty]
    public string Count { get; set; }
    [JsonProperty]
    public Boolean Category { get; set; }
}
{% endhighlight %}

Below is our application layer model. You may notice that I have three extra properties (ItemId, Location, LastChecked). The main reason is so the extra properties will not be exposed to the presentation layer during a request. We only want to show users the ItemId, Location, and LastChecked properties on a response. On that note, the extra properties may be used for other various reasons such as sorting, querying, filtering...etc.

{% highlight csharp %}
public class ItemDTO
{
    public long ItemId { get; set; }
    public long Name { get; set; }
    public string Count { get; set; }
    public bool Category { get; set; }
    public string Location { get; set; }
    public DateTime? LastChecked { get; set; }
}
{% endhighlight %}

> C# makes creating models very easy with inline encapsulation.

# Repository Layer

It's important to keep in mind that the repository layer is meant to only utilize basic CRUD calls. The repository is only meant to extend your data access layer. Keeping it simple is the trick. First, create an interface - we can call this IInventoryRepository. For simplicity sake, we can create one function within the interface that adds an item. Below is the implementation of the interface.

{% highlight csharp %}
public class InventoryRepository : IInventoryRepository
{
    // Dependency Injection
    private IItemDataProvider itemDataProvider;

    public InventoryRepository(IItemDataProvider itemDataProvider)
    {
        this.itemDataProvider = itemDataProvider;
    }

    public ItemDTO GetItem(long itemId)
    {
        return itemDataProvider.GetItem(itemId);
    }
}
{% endhighlight %}

On another note, one can create a generic repository to avoid adding dozens of functions. Also, if multiple operations are called from a single function then it would be ideal to utilize the **Unit of Work** approach. The Unit of Work approach dictates that the operations be run through a transaction, so if one fails, the others do not run.

# Data Access Layer

This layer is meant to communicate with the database, be it through stored procedures, LINQ to SQL or SQL queries. This layer is usually only access by the repository layer. The same interface rules from the repository apply here.

{% highlight csharp %}
public class InventoryDataProvider : IInventoryDataProvider
{
    public long AddItem(ItemDTO itemDTO)
    {
        // Access SQL database and add data
        // return newly added itemId
    }
}
{% endhighlight %}

In the case above, I am only returning a number, but for a **GET REST** request you may grab the data returned from SQL and build ItemDTO and return that instead.

# Dependency Injection (DI)

You may have caught on already, in the ItemController & InventoryRepository, I injected the IInventoryRepository & IInventoryDataProvider through the constructors. Using the **Unity Framework**, I've setup my UnityConfig.cs to map IInventoryRepository to InventoryRepository & IInventoryDataProvider to InventoryDataProvider. Using DI, we can eliminate the use of the keyword **new** everytime we need to access the repository or data provider, which conforms with the SOLID principles as well. Below is the snippet from the UnitConfig class located in the App_Start folder.

{% highlight csharp %}
public static void RegisterTypes(IUnityContainer container)
{
    container.RegisterType<IInventoryRepository, InventoryRepository>();
    container.RegisterType<IInventoryDataProvider, InventoryDataProvider>();
}
{% endhighlight %}

# Conclusion

Following this pattern allows for thorough unit tests as well as increased code coverage since you can test each layer fairly effectively. C# is designed very well (for most things) to allow you to create a Web Api utilizing this pattern in a matter of hours. 