---
layout: post
title: "ConceptNet: a database odyssey"
date: 2014-07-01
comments: true
categories: [conceptnet, database]
---

Ancient storytelling has given us the idea of an "odyssey": an archetypical story where the hero sets out on an adventure in a distant land, but the real story is about the long journey home. At the end, the hero may find that his home has changed considerably, or see it in a new light. The classic example would of course be Homer's Odyssey. There have been many stories of the same form, either directly or indirectly based on the ancient Greek epic. The film *O Brother, Where Art Thou* is an odyssey in every sense, of course, but so are many other stories that are less explicit about the storytelling tradition they come from.

The hero of the odyssey I'll be telling here is [ConceptNet](http://conceptnet5.media.mit.edu), an open knowledge representation project based at the MIT Media Lab. ConceptNet is a multilingual collaboration with contributors from all over the world, but I've been the primary maintainer for a long time, including for the entirety of the current major version, ConceptNet 5. 

ConceptNet has spent years wandering the seas in search of a good representation. The feedback I hear about ConceptNet 5 has been pretty consistent: it seems like it should be a great improvement over previous versions, but in practical terms, people find the data really hard to get at. They'd often rather use ConceptNet 4, which has much less data, and its representation is entangled in an old version of Django. Or sometimes they'd even use ConceptNet 2, despite its totally crazy representation, just because it still has all the Googlejuice.


## Our starting point (2008): SQLite and PostgreSQL

If you wanted to get at ConceptNet 4 programmatically, you had to do it with the object-relational mapper in Django 1.2.

When I wanted to expand ConceptNet to take in more kinds of data, in many more languages, it was clear that the Django representation was going to be pretty limiting. It was standing in the way of things we wanted to do, not to mention the fact that it forced other people who wanted to integrate with ConceptNet to use Python and Django (or rely on the web API).

By this time, we were already working with a group at NTU in Taiwan, who have done a great job collecting crowd-sourced knowledge in Traditional Chinese. But once our databases diverged, there was no good way to merge them together again. I had followed good "normalization" practices and made everything refer to everything else with sequential ID numbers, which of course won't line up once there are multiple versions of the database.

Even in English, if I noticed a blatant problem with the data, I might find myself with no efficient way to fix it. Bulk updates would wedge the PostgreSQL database, blocking many people's research. Sneaking in updates one at a time would take *weeks*. The database was too large, too complex, and had too much un-mergeable idiosyncratic data like row IDs, and I still wanted ConceptNet to be an order of magnitude larger.

The last five years have yielded many exciting new ways to represent data. Some of them sounded particularly good for ConceptNet. Thus begins the multi-year journey of trying to find a usable representation of ConceptNet 5.

I apologize if I end up being particularly negative toward various databases. I think this is a cautionary tale that needs to be told. Some of the negativity may seem unfair, as it relates to problems that have probably been fixed for years now, but the fact that these things can happen is something you should keep in mind before you run off chasing *this* year's shiny new representation.


## RDF triple-stores (2009)

The story of ConceptNet 5 had not yet begun in 2009, but I was already spending time looking for a more effective representation of it than a tangle of SQL tables.

By then, the Semantic Web was already in the realm of retro-futurology, one of those cool things we should have by now like flying cars, Mr. Fusion, and hoverboards. As it turned out, we don't really have a Web that is semantic. But it seemed possible that the representations the Semantic Web project had created were a good match for ConceptNet.

ConceptNet is built from assertions that involve a subject, a relation, and an object:

    [image goes here]

Meanwhile, RDF describes *everything* in terms of a subject, a predicate, and an object. But I could already tell that, despite sounding almost the same, this representation was not a perfect match for ConceptNet.

A ConceptNet assertion is an object that we can describe. We can, for example, add information about why we believe it's true, where it came from, how it would be translated into another language, and -- perhaps most importantly -- how strongly we should believe it. An RDF triple, on the other hand, is supposed to describe something that's incontrovertibly true, and once it's stated there's no way to refer to it.

That doesn't mean you're stuck if you want to represent ConceptNet-like data in RDF. What you have to do is *reify* each object.

If you want to assert that a dog has a tail, for example, it would be tempting to use the triple `<dog> <HasA> <tail>`, but then there's no way to refer to that information or to give it more detail. The reified version looks more like this:

    <assertion #628318> <SUBJECT> <dog>
    <assertion #628318> <RELATION> <HasA>
    <assertion #628318> <OBJECT> <tail>
    <assertion #628318> <WEIGHT> 3.0

...and so on. Then assertion #628318 is something you can refer to later.

Lots of data is represented this way, so this wasn't a reason to give up on RDF. The reason to give up on RDF was looking around at the available technologies for actually storing and querying these RDF triples. Python's `rdflib`, at the time, supported an assortment of different data backends, all of which were incomplete or unusable in one way or another except the one called `:memory:`, and you can guess how that one worked. One option in RDFlib, along with the beefier Java implementations of RDF that some people said I should be using, offered to store the triples in a SQL database, which struck me as very similar to the situation with ConceptNet 4 except even harder to update, because the data would no longer be structured appropriately and the joins would be even more horrible.

Nowadays RDFlib wants you to use BerkeleyDB, which is, coincidentally, the database that destroyed the first version of Open Mind Common Sense and silently corrupted its backup, back in 2002 or so, but that's another story.

The big question the RDF representation leaves you with is: how do you query your data once you've turned it into triples? And the common answer you'll get is "SPARQL". SPARQL is not the answer. SPARQL is not even computationally feasible. It's basically a denial of service attack against your own computer.

So that's where the Semantic Web approach would leave me: with reified data scattered to the wind, with no good way to query it, and *still* with a difficult decision about how to actually store it on a disk. After a couple of weeks of experimentation, I abandoned this idea.


## Neo4j (2011)

We had big plans for ConceptNet 5 in 2011, and graph databases were suddenly big news. As we started to design ConceptNet 5, we built our ideas around the most advanced graph database, Neo4j.

Starting this process was *fun*. We could visualize the data we were putting in with a beautiful Web interface. Neo4j's query language, Gremlin, gave us the ability to interact directly with the graph at a REPL, including some fairly complex graph queries that were relatively easy to express. There was just the slight drawback that it required interacting with a separate server running in Java, while the ConceptNet code is in Python, and Gremlin is kind of a third language on top of that... but there was an HTTP API, so that glues everything together, right?

Things started to go bad when we actually tried to import the millions of assertions in ConceptNet into Neo4j. See, Neo4j didn't have *any* kind of data importer. The recommended way to make a Neo4j graph was to copy it from another Neo4j graph you already had. There were some experimental scripts for importing from some vaguely-standardized graph languages, except they broke after a few thousand edges.

I tried sending the assertions one at a time to Neo4j's HTTP API, and gave up on that when I realized that I would need to accomplish some research in the *months* this would take to complete. I tried writing some clever Gremlin code that would try to create batches of edges out of data it read from a flat file, and found that this crashed the entire Neo4j server due to running out of "PermGen". This is a situation that if you describe it to a Java programmer, they'll know what you're talking about and nod sadly, while no other sort of programmer has ever had to worry about "PermGen".

Other problems with the server started to reveal themselves. Despite Neo4j's free academic license, we'd be missing many of the features -- such as any way to distribute the data between multiple computers -- unless we bought the expensive enterprise license. And the shiny Web interface, when running, gave the entire Internet read-write access to the graph. The Neo4j community seemed surprisingly calm about this fact. There was an option to restrict it to localhost, in theory, but it didn't work. If you tried to take matters into your own hands and start firewalling and proxying things in an attempt to enforce some security, various parts of the Javascript interface would stop working.

These bugs were presumably resolved eventually, but with a paper deadline looming, I realized I had no way to get the data in, no way to get the data out once it was in, and no way to safely run the server anyway. Neo4j was a big dangerous black box that didn't play nicely with anything else. I needed another solution, fast.


## MongoDB (2011)

MongoDB makes a great first impression. The first thing you're probably trying to do with it is to import some key-value mappings into a fresh database, and it provides a perfectly clear API with which you can do that really fast. As I was jumping ship from Neo4j, I *needed* to do that really fast, so MongoDB became my new savior. And I loved the query language that actually looked like my data.

Things become a bit less rosy when you realize you need a way to get at your data besides mapping a key to a value. Once you need an *index*, importing things into MongoDB suddenly gets a lot slower -- but this was still better than the completely infeasible situation I'd been in before. By the way, don't even think about trying to load the data first and add the index later; I had to delete a couple of perma-wedged databases after getting that bright idea.

Once the indexed data was loaded, I found MongoDB to be rather a resource hog. If you want queries to finish quickly, they say, your whole data set has to fit in RAM. ConceptNet would kind of barely fit in RAM if it was distributed across two expensive computers. And expensive they were; Amazon had roped me into AWS with a research grant, which I burned through in a couple of months. AWS was another bad idea. But I begged some more money out of a grant to pay for AWS for a while longer, ignoring the 2008 Mac Pro on my desk that had more power than the two AWS servers combined, because a 2008 Mac Pro does not "scale". I set up the two servers with MongoDB, and pressed onward.

Sharding! Distributed data! This was exciting! I successfully imported all the data, and I even put up an API so that people everywhere could query ConceptNet 5 like they were able to query ConceptNet 4.

Now that it's 2014, you're probably shaking your head sadly. You do not want to shard MongoDB. You especially did not want to shard MongoDB in 2011. It's now quite popular to make fun of the marketing statement "MongoDB is web scale", as it contrasts amusingly with the fundamental impossibility of scaling MongoDB. There were nodes that would go down and could never be convinced to come up again, there was inscrutable data loss due to the default settings being completely unsafe and the safe settings being unusably slow, and there was the occasional need to "rebalance", which would unfortunately block every node's single thread of operation for, say, 24 hours. I could not *afford* to run an independent copy of MongoDB, so whenever I did anything that required any sort of rebalancing, ConceptNet 5 would go down. For everybody. For a day or so.

But the paper was written and the code was released, so ConceptNet 5.0 consisted of a couple of MongoDBs.


## A note about MapReduce

MongoDB was also my first experience with a database that promised the siren song of "MapReduce". Many providers of NoSQL databases will, by way of apologizing that you can't "join" anymore, happily tell you that their database can do MapReduce, which is *better* than a join because it's distributed and functional and was invented by magical unicorns at Google.

Whereas previously you had to invoke a relational operation that has been well understood for decades, now you can run *arbitrary code* in your database's *only thread* that sends gigabytes of data across a network socket for *no good reason*! Excited yet?

MapReduce was created for a situation where you have more data than you can even fit on a computer. You lose orders of magnitude of efficiency, because any interesting result that you compute will eventually have to be sent over a network. You can no longer take advantage of locality of reference in your data; you have to read it all in anew at every step. But maybe you settle for this, because it gives you an obvious way to scale your code in parallel, and might even give you the ability to compute things you would not otherwise have the resources to compute.

Here are some situations where it makes absolutely no sense to use MapReduce:

1. Your data fits on a single hard disk
2. Your code runs in a single thread

These are also basically requirements for using MongoDB. MongoDB multiplies the size of your data by a huge factor and then expects the important parts of it to fit in RAM somewhere. If you hadn't put it into MongoDB, it would fit comfortably on a hard disk. A *cheap* hard disk. And MongoDB is thoroughly single-threaded, so you get no advantage from parallelism unless you buy a whole lot of computers and let 3/4 of their cores go to waste.

Now, before you think this is just the usual, fashionable Mongo-bashing, let me point out that they are not at all alone in the foolish use of MapReduce.

I have since tried Riak. Not for ConceptNet, but for things at Luminoso. To their credit, Riak has thought through some difficult issues of distributed data much more thoroughly than MongoDB has, though they fall behind on aspects like importing data quickly, or being able to provide diagnostic messages that aren't mysterious blobs of Erlang that look like `{error, <100,105,115,107,32,105,115,32,111,110,32,102,105,114,101,44,32,121,111>}`. But I digress.

For some reason, despite seeming like otherwise reasonable people who have thought about distributed data, Riak also encourages you to accomplish things with MapReduce. But only, y'know, sometime when you don't really need your database to be available for other things. Maybe you can use it once a week, to generate some kind of batch report thingy, late on Sunday night. But you should probably do a dry run on a different cluster so you can be sure it'll be done by Monday morning. There are times when nobody's using your system, right?

If you keep asking the friendly Riak mailing list, you eventually get a clearer answer about when it's a good time to run MapReduce in your database, which is never. This is not a fact about Riak, really, it's about how MapReduce and databases are fundamentally different applications.

I'm convinced that the only people who ever benefit from a database's ability to MapReduce are the salespeople for that database. Recently, the news I heard about RethinkDB made me consider giving it a try. But I notice that "MapReduce" is one of their prominently advertised features, making me think that the database is optimized for sales over efficiency like so many others.


## Solr (2012)

Let me start by being positive about Solr. Solr does a hard job pretty well, and this was enough for ConceptNet for a while.

Most databases are uncomfortable with text, especially with the idea that you'd want to search for large gobs of it. Full-text search is an under-supported addon in most systems. Solr puts full-text search front and center, and designs everything else around it.
