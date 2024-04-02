- [Scalability](#scalability)
  - [How to scale read](#how-to-scale-read)
  - [How to scale write](#how-to-scale-write)


# Scalability

## How to scale read

**Cache**

* Memcache or Rediss usually are cluster, and usually one operation takes 0.1ms or less.
  * 1 CPU cores =&gt; 5000-10000 requests
  * 20-40 CPU cores =&gt; 200K requests \(one machine\)

```text
string originalUrl = getUrlFromMemcache(tinyUrl)
if (originalUrl == null)
{
    originalUrl = readFromDatabase(tinyUrl);
    putUrlIntoCache(tinyUrl, originalUrl);
}
return originalUrl;
```

**One master multiple slaves**

* Write to master, streamly replicated to slaves, usually less than one second Read from slaves.
* Pros
  * Increase availability to handle single point of failure
  * Reduce read pressure on a single node
* Cons
  * Replication lag
    * Solution1: We can write to memcache when creating a new tiny url. Service can get tiny url from memcache instead of database. 
    * Solution2: We can implement Raft protocol against relational database or use opensource or commercial realted database. 

**Optimize based on geographical info**

* Web server
  * Different web servers deployed in different geographical locations
  * Use DNS to parse different web servers to different geographical locations
* Database 
  * Centralized MySQL + Distributed memcached server
  * Cache server deployed in different geographical locations

## How to scale write

**Sharding**

**Problematic scenarios**

* Too many write operations
  * The limit of 62^6 is approaching

**How to locate physical machine**

* Problem: For "31bJF4", how do we find which physical machine the code locate at? 
* Our query is 
  * Select original\_url from tiny\_url where short\_url = XXXXX;
* The simple one
  * We query all nodes. But there will be n queries
  * This will be pretty expensive, even if we fan out database calls. 
  * But the overall QPS to database will be n times QPS 
* Solution
  * Add virtual node number to prefix "31bJF4" =&gt; "131bJF4"
  * When querying, we can find virtual nodes from prefix \(the tinyUrl length is 6\). Then we can directly query node1.

```text
// Assign virtual nodes to physical nodes
// Store within database
{
    0: "db0.tiny_url.com",
    1: "db1.tiny_url.com",
    2: "db2.tiny_url.com",
    3: "db3.tiny_url.com"
}
```

**Sharding with multiple MySQL instances**

* Vertical sharing
  * Only one table
  * Even with Custom URL, two tables in total
* Horizontal sharding: Choose sharding key?
  * Use Long Url as sharding key
    * Short to long operation will require lots of cross-shard joins
  * Use ID as sharding key
    * Short to long url: First convert shourt url to ID; Find database according to ID; Find long url in the corresponding database
    * Long to short url: Broadcast to N databases to see whether the link exist before. If not, get the next ID and insert into database. 
  * Combine short Url and long Url together
    * Hash\(longUrl\)%62 + shortkey
    * Given shortURL, we can get the sharding machine by the first bit of shortened url.
    * Given longURL, get the sharding machine according to Hash\(longURL\) % 62. Then take the first bit.
  * Sharding according to the geographical info. 
    * First know which websites are more popular in which region. Put all websites popular in US in US DB.

**How to get global unique ID?**

* Zookeeper
* Use a specialized database for managing IDs