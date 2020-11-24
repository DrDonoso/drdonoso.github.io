---
title: "Autoscale for Cosmos DB"
date: 2019-07-31T15:58:26+08:00
lastmod: 2019-07-31T15:58:26+08:00
draft: true
# author: "David"
authorLink: ""
description: ""
license: ""

tags: [cosmosdb]
categories: [data]
hiddenFromHomePage: false

featuredImage: ""
featuredImagePreview: ""

toc: false
autoCollapseToc: true
math: true
lightgallery: true
linkToMarkdown: true
share:
  enable: true
comment: true
---
If you have ever used Cosmos DB in your project, maybe you realized that one of the main concerns of using Cosmos DB is that it doesn't have autoscale. Microsoft started to work on this two years ago.

![](https://thepracticaldev.s3.amazonaws.com/i/0e2pef8gci80rkjz64ri.png)

One problem is when you are having lots of requests in a range of hours or maybe because something triggers that lots of users use the application at the same time.


# How to do
First of all we will need the method to modify the offer. For this I added some methods to get the offers and scale it:

```c#
private async Task ScaleAsync(string database, string collection, int scaleBatch, int minThroughput, int maxThroughput)
{
	var offer = await GetOfferAsync(database, collection);
	await ScaleAsync(offer, scaleBatch, minThroughput, maxThroughput);
}

private async Task ScaleAsync(OfferV2 offer, int scaleBatch, int minThroughput, int maxThroughput)
{
	var currentThroughput = offer.Content.OfferThroughput;

	var newThroughput = currentThroughput + scaleBatch;
	if (newThroughput < minThroughput)
	{
	    newThroughput = minThroughput;
	}

	if (newThroughput > maxThroughput)
	{
	    newThroughput = maxThroughput;
	}
	var updatedOffer = new OfferV2(offer, newThroughput);
	await documentClient.ReplaceOfferAsync(updatedOffer);
}

private async Task<OfferV2> GetOfferAsync(string database, string collection)
{
	return await GetCollectionOfferAsync(database, collection) ?? await GetDatabaseOfferAsync(database);
}

private async Task<OfferV2> GetCollectionOfferAsync(string database, string collection)
{
	var collectionUri = UriFactory.CreateDocumentCollectionUri(databaseName, collectionName);
    var collection = (await _documentClient.ReadDocumentCollectionAsync(collectionUri)).Resource;

	return await GetOfferAsync(collection.SelfLink);
}

private async Task<OfferV2> GetDatabaseOfferAsync(string database)
{
	var databaseUri = UriFactory.CreateDatabaseUri(databaseName);
    var database = (await _documentClient.ReadDatabaseAsync(databaseUri)).Resource;

	return await GetOfferAsync(database.SelfLink);
}

private async Task<OfferV2> GetOfferAsync(string selfLink)
{
	return (await _documentClient.CreateOfferQuery()
                .Where(o => o.ResourceLink == selfLink)
                .AsDocumentQuery()
                .ExecuteNextAsync<OfferV2>()).FirstOrDefault();
}
```

## Scale up

When there is a request that is exceeding the provisioned throughput that you have for the database or collection, Cosmos is throwing a ``DocumentClientException`` with the status code 429.
So, what we should do is to catch this exception and scale up this database or collection and retry the execution:

```c#
public async Task<T> ExecuteAsync<T>(Func<IDocumentClient, Task<T>> func, string database, string collection, int scaleUpBatch, int minThroughput, int maxThroughput, int retries = 0)
{
    try
    {
        return await func(_documentClient);
    }
    catch (DocumentClientException exception) when ((int)exception.StatusCode == 429)
    {
        retries++;
        await ScaleAsync(database, collection, scaleUpBatch, minThroughput, maxThroughput);
        if (retries < _config.MaxRetries) return await ExecuteAsync(func, database, collection, scaleUpBatch, minThroughput, maxThroughput, retries);
        throw;
    }
}
```


## Scale down
To scale down we should use a cronjob, I was using Azure Functions to scale down all the collections.

```c#
public async Task ScaleDownAll(int scaleDownBatch, int minThroughput, int maxThroughput)
{
    var scaleBatch = Math.Abs(scaleDownBatch) * -1;
    var databases = _documentClient.CreateDatabaseQuery().AsEnumerable().ToList();

    foreach (var database in databases)
    {
        var offer = await GetOfferAsync(database.SelfLink);

        if (offer != null)
        {
            await ScaleAsync(offer, scaleBatch, minThroughput, maxThroughput);
        }
        else
        {
            var collections = _documentClient.CreateDocumentCollectionQuery(database.SelfLink).ToList();
            foreach (var collection in collections)
            {
                offer = await GetOfferAsync(collection.SelfLink);
                await ScaleAsync(offer, scaleBatch, minThroughput, maxThroughput);
            }
        }
    }
}
```

If you want to see more about this, I started a repo in Github:

https://github.com/DrDonoso/Cosmos-Autoscale


