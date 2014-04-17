---
layout: post
title: Fetch Requests
category: "4"
date: "2013-09-09 06:00:00"
author: "<a href=\"http://twitter.com/danielboedewadt\">Daniel Eggert</a>"
tags: article
---

{% include links-4.md %}

A way to get objects out of the store is to use an `NSFetchRequest`. Note, though, that one of the most common mistakes is to fetch data when you don't need to. Make sure you read and understand [Getting to Objects][320]. Most of the time, traversing relationships is more efficient, and using an `NSFetchRequest` is often expensive.

调用对象的方法之一是使用 `NSFetchRequest`。但是请注意，尽管如此，有一个最常见的错误是在你不需要的时候读取数据。你要确保你已经阅读并理解 [Getting to Objects][320]。大多数时候，遍历关系更加有效，而使用 `NSFetchRequest` 成本更高。

There are usually two reasons to perform a fetch with an `NSFetchRequest`: (1) You need to search your entire object graph for objects that match specific predicates. Or (2), you want to display all your objects, e.g. in a table view. There's a third, less-common scenario, where you're traversing relationships but want to pre-fetch more efficiently. We'll briefly dive into that, too. But let us first look at the main two reasons, which are more common and each have their own set of complexities.
通常有两个原因使用 `NSFetchRequest` 来执行数据读取：（1）你需要为匹配特定谓词的对象搜索整个对象图；或者（2）你想要显示所有的对象，例如使用表视图。第三，是一个较不常见的情况，你遍历关系的同时却想要更高效地预先提取。我们也将简单深入这个问题。不过我们先来看看两个主要原因，它们更加常见并且每个都具有自己的复杂性。

## The Basics
## 基本原理

We won't cover the basics here, since the Xcode Documentation on Core Data called [Fetching Managed Objects](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreData/Articles/cdFetching.html) covers a lot of ground already. We'll dive right into some more specialized aspects.

在这里我们不会涉及基本原理，因为一个关于 Core Data 的名为 [Fetching Managed Objects](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreData/Articles/cdFetching.html) 的 Xcode 文档已经涵盖了大量基本原理。我们将深入到一些更专业的方面。



## Searching the Object Graph
## 搜索对象图

In our [sample with transportation data](https://github.com/objcio/issue-4-importing-and-fetching), we have 12,800 stops and almost 3,000,000 stop times that are interrelated. If we want to find stop times with a departure time between 8:00 and 8:30 for stops close to 52° 29' 57.30" North, +13° 25' 5.40" East, we don't want to load all 12,800 *stop* objects and all three million *stop time* objects into the context and then loop through them. If we did, we'd have to spend a huge amount of time to simply load all objects into memory and then a fairly large amount of memory to hold all of these in memory. Instead what we want to do is have SQLite narrow down the set of objects that we're pulling into memory.

在我们的 [sample with transportation data](https://github.com/objcio/issue-4-importing-and-fetching) 中，我们停止了12,800次，其中几乎有3,000,000个停止时间相互关联。如果我们由于停止接近北52° 29' 57.30"，东13° 25' 5.40"，而想要使用介于八点到八点半之间的开始时间来查找停止时间，我们不会想要在这个背景中加载所有的12800个 *停止* 对象和30万个 *停止时间* 对象，然后再对它们进行循环访问。如果我们这么做了，我们将不得不花费大量时间向储存器中加载所有的对象，然后再花大量时间储存它们。我们应该使用SQLite来缩减进入储存器的对象的集合。


### Geo-Location Predicate
### 地理定位谓词

Let's start out small and create a fetch request for stops close to 52° 29' 57.30" North, +13° 25' 5.40" East. First we create the fetch request:
让我们从小处开始，为接近北52° 29' 57.30"东13° 25' 5.40"的停止创建一个读取请求。首先我们创建这个读取请求：

    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:[Stop entityName]]

We're using the `+entityName` method that we mention in [Florian's data model article][250]. Next, we need to limit the results to just those close to our point.
我们使用[Florian's data model article][250]中提到的`+entityName`法。然后我们需要限制结果，使其接近我们的关注点。

We'll simply use a (not quite) square region around our point of interest. The actual math is [a bit complex](https://en.wikipedia.org/wiki/Geographical_distance), because the Earth happens to be somewhat similar to an ellipsoid. If we cheat a bit and assume the earth is spherical, we get away with this formula:

我们仅仅在我们的关注点周围使用一个正方形区域。现实中的数学公式[有些复杂] (https://en.wikipedia.org/wiki/Geographical_distance)，这是由于地球从某种程度上来说接近椭圆。如果我们假设地球是正圆形，我们可以使用以下方法来解决问题：

    D = R * sqrt( (deltaLatitude * deltaLatitude) +
	              (cos(meanLatitidue) * deltaLongitude) * (cos(meanLatitidue) * deltaLongitude))

We end up with something like this (all approximate):
以以下内容来结束（均为近似值）
    static double const R = 6371009000; // Earth readius in meters
    double deltaLatitude = D / R * 180 / M_PI;
    double deltaLongitude = D / (R * cos(meanLatitidue)) * 180 / M_PI;

Our point of interest is:
我们的关注点是：
	CLLocation *pointOfInterest = [[CLLocation alloc] initWithLatitude:52.4992490 
                                                             longitude:13.4181670];

We want to search within ±263 feet (80 meters):
我们想在±263英尺（80米）内进行搜索：	
	static double const D = 80. * 1.1;
    double const R = 6371009.; // Earth readius in meters
	double meanLatitidue = pointOfInterest.latitude * M_PI / 180.;
	double deltaLatitude = D / R * 180. / M_PI;
	double deltaLongitude = D / (R * cos(meanLatitidue)) * 180. / M_PI;
	double minLatitude = pointOfInterest.latitude - deltaLatitude;
	double maxLatitude = pointOfInterest.latitude + deltaLatitude;
	double minLongitude = pointOfInterest.longitude - deltaLongitude;
	double maxLongitude = pointOfInterest.longitude + deltaLongitude;

(This math is broken when we're close to the 180° meridian. We'll ignore that since our traffic data is for Berlin which is far, far away.)
（当我们接近经线180°的时候，这个公式无法成立。由于我们使用的是离经线180°很远很远的柏林交通数据，所以我们选择忽视这个问题。）

    request.result = [NSPredicate predicateWithFormat:
                      @"(%@ <= longitude) AND (longitude <= %@)"
                      @"AND (%@ <= latitude) AND (latitude <= %@)",
                      @(minLongitude), @(maxLongitude), @(minLatitude), @(maxLatitude)];

There's no point in specifying a sort descriptor. Since we're going to be doing a second in-memory pass over all objects, we will, however, ask Core Data to fill in all values for all returned objects:
指定一种描述符毫无意义。由于我们会对所有的对象做另外一个存储通行证，不过我们将用core data填入所有返回对象的所有的值。

    request.returnsObjectsAsFaults = NO;

Without this, Core Data will fetch all values into the persistent store coordinator's row cache, but it will not populate the actual objects. Often that makes sense, but since we'll immediately be accessing all of the objects, we don't want that behavior.
否则，core data将进入持久化存储协调器的row cache读取所有的值，不过它不会填充实际对象。这往往是可行的，不过由于我们可以立刻访问所有对象，我们不会作出如上行为。

As a safe-guard, it's good to add:
为安全防范考虑，最好加上如下代码：
    request.fetchLimit = 200;

We execute this fetch request:
执行这条读取请求
    NSError *error = nil;
    NSArray *stops = [moc executeFetchRequest:request error:&error];
    NSAssert(stops != nil, @"Failed to execute %@: %@", request, error);

The only (likely) reasons the fetch would fail is if the store went corrupt (file was deleted, etc.) or if there's a syntax error in the fetch request. So it's safe to use `NSAssert()` here.
本次读取失败唯一可能的理原因是储存器损坏（文件被删除等待），或者读取请求中出现句法错误。所以说在这里使用`NSAssert()`是安全的。

We'll now do the second pass over the in-memory data using Core Locations advance distance math:
我们现在使用Core Locations推进距离数学，为储存器中的数据做第二个通行证。

    NSPredicate *exactPredicate = [self exactLatitudeAndLongitudePredicateForCoordinate:self.location.coordinate];
    stops = [stops filteredArrayUsingPredicate:exactPredicate];

and:

    - (NSPredicate *)exactLatitudeAndLongitudePredicateForCoordinate:(CLLocationCoordinate2D)pointOfInterest;
    {
        return [NSPredicate predicateWithBlock:^BOOL(Stop *evaluatedStop, NSDictionary *bindings) {
            CLLocation *evaluatedLocation = [[CLLocation alloc] initWithLatitude:evaluatedStop.latitude longitude:evaluatedStop.longitude];
            CLLocationDistance distance = [self.location distanceFromLocation:evaluatedLocation];
            return (distance < self.distance);
        }];
    }

And we're all set.
至此我们全部设置好了
#### Geo-Location Performance
#### 地理定位性能

These fetches take around 360µs on average on a recent MacBook Pro with SSD. That is, you can do approximately 2,800 of these requests per second. On an iPhone 5 we'd be getting around 1.67ms on average, or some 600 requests per second.
最近使用SSD硬盘的MacBook Pro读取数据平均约需要360µs，也就是说每秒约做2800次请求。iPhone 5平均需要1.67ms，每秒600次请求。

If we add `-com.apple.CoreData.SQLDebug 1` as launch arguments to the app, we get this output:
如果我们给app加上启动参数`-com.apple.CoreData.SQLDebug 1`，我们得到如下输出结果：

    sql: SELECT 0, t0.Z_PK, t0.Z_OPT, t0.ZIDENTIFIER, t0.ZLATITUDE, t0.ZLONGITUDE, t0.ZNAME FROM ZSTOP t0 WHERE (? <=  t0.ZLONGITUDE AND  t0.ZLONGITUDE <= ? AND ? <=  t0.ZLATITUDE AND  t0.ZLATITUDE <= ?)  LIMIT 100
    annotation: sql connection fetch time: 0.0008s
    annotation: total fetch execution time: 0.0013s for 15 rows.

In addition to some statistics (for the store itself), this shows us the generated SQL for these fetches:
另外，对某些对储存器本身的统计而言，这向我们展示了这些请求的生成SQL

    SELECT 0, t0.Z_PK, t0.Z_OPT, t0.ZIDENTIFIER, t0.ZLATITUDE, t0.ZLONGITUDE, t0.ZNAME FROM ZSTOP t0
	WHERE (? <=  t0.ZLONGITUDE AND  t0.ZLONGITUDE <= ? AND ? <=  t0.ZLATITUDE AND  t0.ZLATITUDE <= ?)
	LIMIT 200

which is what we'd expect. If we'd want to investigate the performance, we can use the SQL [`EXPLAIN` command](https://www.sqlite.org/eqp.html). For this, we'd open the database with the command line `sqlite3` command like this:
这是我们所希望得到的。如果我们想要对这个性能进行调查研究，我们可以使用SQL[`EXPLAIN` command](https://www.sqlite.org/eqp.html)。我们可以像这样使用命令行`sqlite3`来打开数据库：

    % cd TrafficSearch
	% sqlite3 transit-data.sqlite
	SQLite version 3.7.13 2012-07-17 17:46:21
	Enter ".help" for instructions
	Enter SQL statements terminated with a ";"
	sqlite> EXPLAIN QUERY PLAN SELECT 0, t0.Z_PK, t0.Z_OPT, t0.ZIDENTIFIER, t0.ZLATITUDE, t0.ZLONGITUDE, t0.ZNAME FROM ZSTOP t0
	   ...> WHERE (13.30845219672199 <=  t0.ZLONGITUDE AND  t0.ZLONGITUDE <= 13.33441458422844 AND 52.42769566863058 <=  t0.ZLATITUDE AND  t0.ZLATITUDE <= 52.44352370653525)
	   ...> LIMIT 100;
	0|0|0|SEARCH TABLE ZSTOP AS t0 USING INDEX ZSTOP_ZLONGITUDE_INDEX (ZLONGITUDE>? AND ZLONGITUDE<?) (~6944 rows)

This tell us that SQLite was using the `ZSTOP_ZLONGITUDE_INDEX` for the `(ZLONGITUDE>? AND ZLONGITUDE<?)` condition. We could do better by using a *compound index* as described in the [model article][260]. Since we'd always search for a combination of longitude and latitude that is more efficient, and we can remove the individual indexes on longitude and latitude.
这告诉我们SQLite使用`ZSTOP_ZLONGITUDE_INDEX` for the `(ZLONGITUDE>? AND ZLONGITUDE<?)`条件。我们可以像[model article][260]中描述的那样使用*compound index*，则会做的更好。由于我们总是为了效率考虑而同时搜索经纬度，我们可以去除经度和纬度各自的指数。

This would make the output look like this:
输出如下：
    0|0|0|SEARCH TABLE ZSTOP AS t0 USING INDEX ZSTOP_ZLONGITUDE_ZLATITUDE (ZLONGITUDE>? AND ZLONGITUDE<?) (~6944 rows)

In our simple case, adding a compound index hardly affects performance.
在我们简单的案例中加上复合索引几乎不会对性能造成影响。
As explained in the [SQLite Documentation](https://www.sqlite.org/eqp.html), the warning sign is a `SCAN TABLE` in the output. That basically means that SQLite needs to go through *all* entries to see which ones are matching. Unless you store just a few objects, you'd probably want an index.
就像在[SQLite 文档](https://www.sqlite.org/eqp.html)中所作出的解释一样，警报信号是输出一个SCAN TABLE`。这基本上表示SQLite需要检查*所有*数据表，来寻找相匹配的的数据表。

### Subqueries

Let's say we only want those stops near us that are serviced within the next twenty minutes.

We can create a predicate for the *StopTimes* entity like this:

    NSPredicate *timePredicate = [NSPredicate predicateWithFormat:@"(%@ <= departureTime) && (departureTime <= %@)",
                                  startDate, endDate];

But what if what we want is a predicate that we can use to filter *Stop* objects based on the relationship to *StopTime* objects, not *StopTime* objects themselves? We can do that with a `SUBQUERY` like this:

    NSPredicate *predicate = [NSPredicate predicateWithFormat:
                              @"(SUBQUERY(stopTimes, $x, (%@ <= $x.departureTime) && ($x.departureTime <= %@)).@count != 0)",
                              startDate, endDate];

Note that this logic is slightly flawed if we're close to midnight, since we ought to wrap by splitting the predicate up into two. But it'll work for this example.

Subqueries are very powerful for limiting data across relationship. The Xcode documentation for [`-[NSExpression expressionForSubquery:usingIteratorVariable:predicate:]`](https://developer.apple.com/library/ios/documentation/cocoa/reference/foundation/Classes/NSExpression_Class/Reference/NSExpression.html#//apple_ref/occ/clm/NSExpression/expressionForSubquery:usingIteratorVariable:predicate:) has more info. 

We can combine two predicates simply using `AND` or `&&`, i.e.

    [NSPredicate predicateWithFormat:@"(%@ <= departureTime) && (SUBQUERY(stopTimes ....

or in code using [`+[NSCompoundPredicate andPredicateWithSubpredicates:]`](https://developer.apple.com/library/ios/DOCUMENTATION/Cocoa/Reference/Foundation/Classes/NSCompoundPredicate_Class/Reference/Reference.html#//apple_ref/occ/clm/NSCompoundPredicate/andPredicateWithSubpredicates:).

We end up with a predicate that looks like this:

    (lldb) po predicate
    (13.39657778010461 <= longitude AND longitude <= 13.42266155792719
		AND 52.63249629924865 <= latitude AND latitude <= 52.64832433715332)
		AND SUBQUERY(
            stopTimes, $x, CAST(-978250148.000000, "NSDate") <= $x.departureTime 
            AND $x.departureTime <= CAST(-978306000.000000, "NSDate")
        ).@count != 0

#### Subquery Performance

If we look at the generated SQL it looks like this:

    sql: SELECT 0, t0.Z_PK, t0.Z_OPT, t0.ZIDENTIFIER, t0.ZLATITUDE, t0.ZLONGITUDE, t0.ZNAME FROM ZSTOP t0
	     WHERE ((? <=  t0.ZLONGITUDE AND  t0.ZLONGITUDE <= ? AND ? <=  t0.ZLATITUDE AND  t0.ZLATITUDE <= ?)
		        AND (SELECT COUNT(t1.Z_PK) FROM ZSTOPTIME t1 WHERE (t0.Z_PK = t1.ZSTOP AND ((? <=  t1.ZDEPARTURETIME AND  t1.ZDEPARTURETIME <= ?))) ) <> ?)
		 LIMIT 200

This fetch request now takes around 12.3 ms to run on a recent MacBook Pro. On an iPhone 5, it'll take about 110 ms. Note that we have three million stop times and almost 13,000 stops.

The query plan explanation looks like this:

    sqlite> EXPLAIN QUERY PLAN SELECT 0, t0.Z_PK, t0.Z_OPT, t0.ZIDENTIFIER, t0.ZLATITUDE, t0.ZLONGITUDE, t0.ZNAME FROM ZSTOP t0
       ...> WHERE ((13.37190946378911 <=  t0.ZLONGITUDE AND  t0.ZLONGITUDE <= 13.3978625285315 AND 52.41186440524024 <=  t0.ZLATITUDE AND  t0.ZLATITUDE <= 52.42769244314491) AND
       ...> (SELECT COUNT(t1.Z_PK) FROM ZSTOPTIME t1 WHERE (t0.Z_PK = t1.ZSTOP AND ((-978291733.000000 <=  t1.ZDEPARTURETIME AND  t1.ZDEPARTURETIME <= -978290533.000000))) ) <> ?)
       ...> LIMIT 200;
    0|0|0|SEARCH TABLE ZSTOP AS t0 USING INDEX ZSTOP_ZLONGITUDE_ZLATITUDE (ZLONGITUDE>? AND ZLONGITUDE<?) (~3472 rows)
    0|0|0|EXECUTE CORRELATED SCALAR SUBQUERY 1
    1|0|0|SEARCH TABLE ZSTOPTIME AS t1 USING INDEX ZSTOPTIME_ZSTOP_INDEX (ZSTOP=?) (~2 rows)

Note that it is important how we order the predicate. We want to put the longitude and latitude stuff first, since it's cheap, and the subquery last, since it's expensive.


### Text Search

A common scenario is searching for text. In our case, let's look at searching for *Stop* entities by their name.

Berlin has a stop called "U Görlitzer Bahnhof (Berlin)". A naïve way to search for that would be:

    NSString *searchString = @"U Görli";
    predicate = [NSPredicate predicateWithFormat:@"name BEGINSWITH %@", searchString];

Things get even worse if you want to be able to do:

    name BEGINSWITH[cd] 'u gorli'

i.e. do a case and / or diacritic insensitive lookup.

Things are not that simple, though. Unicode is very complicated and there are quite a few gotchas. First and foremost ís that many characters can be represented in multiple ways. Both [U+00F6](http://unicode.org/charts/PDF/U0080.pdf) and [U+006F](http://www.unicode.org/charts/PDF/U0000.pdf) [U+0308](http://unicode.org/charts/PDF/U0300.pdf) represent "ö." And concepts such as uppercase / lowercase are very complicated once you're outside the ASCII code points.

SQLite will do the heavy lifting for you, but it comes at a price. It may seem straightforward, but it's really not. What we want to do for string searches is to have a *normalized* version of the field that you can search on. We'll remove diacritics and make the string lowercase and then put that into a`normalizedName` field. We'll then do the same to our search string. SQLite then won't have to consider diacritics and case, and the search will effectively still be case and diacritics insensitive. But we have to do the heavy lifting up front.

Searching with `BEGINSWITH[cd]` takes around 7.6 ms on a recent MacBook Pro with the sample strings in our sample code (130 searches / second). On an iPhone 5 those numbers are 47 ms per search and 21 searches per second.

To make a string lowercase and remove diacritics, we can use `CFStringTransform()`:

    @implementation NSString (SearchNormalization)
    
    - (NSString *)normalizedSearchString;
    {
        // C.f. <http://userguide.icu-project.org/transforms>
        NSString *mutableName = [self mutableCopy];
        CFStringTransform((__bridge CFMutableStringRef) mutableName, NULL, 
                          (__bridge CFStringRef)@"NFD; [:Nonspacing Mark:] Remove; Lower(); NFC", NO);
        return mutableName;
    }
    
    @end

We'll update the `Stop` class to automatically update the `normalizedName`:

    @interface Stop (CoreDataForward)
    
    @property (nonatomic, strong) NSString *primitiveName;
    @property (nonatomic, strong) NSString *primitiveNormalizedName;
    
    @end
    
    
    @implementation Stop
    
    @dynamic name;
    - (void)setName:(NSString *)name;
    {
        [self willAccessValueForKey:@"name"];
        [self willAccessValueForKey:@"normalizedName"];
        self.primitiveName = name;
        self.primitiveNormalizedName = [name normalizedSearchString];
        [self didAccessValueForKey:@"normalizedName"];
        [self didAccessValueForKey:@"name"];
    }
    
	// ...
	
    @end


With this, we can search with `BEGINSWITH` instead of `BEGINSWITH[cd]`:

    predicate = [NSPredicate predicateWithFormat:@"normalizedName BEGINSWITH %@", [searchString normalizedSearchString]];

Searching with `BEGINSWITH` takes around 6.2 ms on a recent MacBook Pro with the sample strings in our sample code (160 searches / second). On an iPhone 5 it takes 40ms corresponding to 25 searches / second.


#### Free Text Search

Our search still only works if the beginning of the string matches the search string. The way to fix that is to create another *Entity* that we search on. Let's call this Entity `SearchTerm`, and give it a single attribute `normalizedWord` and a relationship to a `Stop`. For each `Stop` we would then normalize the name and split it into words, e.g.:

        "Gedenkstätte Dt. Widerstand (Berlin)"
	->  "gedenkstatte dt. widerstand (berlin)"
	->  "gedenkstatte", "dt", "widerstand", "berlin"

For each word, we create a `SearchTerm` and a relationship from the `Stop` to all its `SearchTerm` objects. When the user enters a string, we search on the `SearchTerm` objects' `normalizedWord` with:

	predicate = [NSPredicate predicateWithFormat:@"normalizedWord BEGINSWITH %@", [searchString normalizedSearchString]]

This can also be done in a subquery directly on the `Stop` objects.

## Fetching All Objects

If we don't set a predicate on our fetch request, we'll retrieve all objects for the given *Entity*. If we did that for the `StopTimes` entity, we'll be pulling in three million objects. That would be slow, and use up a lot of memory. Sometimes, however, we need to get all objects. The common example is that we want to show all objects inside a table view.

What we would do in this case, is to set a batch size:
  
    request.fetchBatchSize = 50;

When we run `-[NSManagedObjectContext executeFetchRequest:error:]` with a batch size set, we still get an array back. We can ask it for its count (which will be close to three million for the `StopTime` entity), but Core Data will only populate it with objects as we iterate through the array. And Core Data will get rid of objects again, as they're no longer accessed. Simply put, the array has batches of size 50 (in this case). Core Data will pull in 50 objects at a time. Once more than a certain number of these batches are around, Core Data will release the oldest batch. That way you can loop through all objects in such an array, without having to have all three million objects in memory at the same time.

On iOS, when you use an `NSFetchedResultsController` and you have a lot of objects, make sure that the `fetchBatchSize` is set on your fetch request. You'll have to experiment with what size works well for you. Twice the amount of objects that you'll be displaying at any given point in time is a good starting point.
