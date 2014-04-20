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

在我们的 [sample with transportation data](https://github.com/objcio/issue-4-importing-and-fetching) 中，我们有 12,800 个车站，其中几乎 3,000,000 个停止时间相互关联。对接近北纬 52° 29' 57.30"，东经 +13° 25' 5.40" 的车站，如果我们想要通过开始时间介于 8：00 和 8：30 之间的对象来查找停止时间，我们不会想要在这个 context 中加载所有的 12,800 个 `车站` 对象和 3,000,000 个 `停止时间` 对象，然后再对它们进行循环访问。如果我们这样做，将不得不花费大量时间以及相当大的存储空间以将所有的对象加载到存储器中。取而代之，我们想要的是使用 SQLite 来缩减进入储存器的对象的集合。


### Geo-Location Predicate
### 地理定位谓词


Let's start out small and create a fetch request for stops close to 52° 29' 57.30" North, +13° 25' 5.40" East. First we create the fetch request:

让我们从小处开始，为位置接近北纬 52° 29' 57.30" 东经 +13° 25' 5.40" 的车站创建一个 fetch 请求。首先我们创建这个 fetch 请求：

    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:[Stop entityName]]

We're using the `+entityName` method that we mention in [Florian's data model article][250]. Next, we need to limit the results to just those close to our point.
我们使用[Florian's data model article][250]中提到的 `+entityName` 方法。然后，我们需要将结果限定为那些接近我们关注点的。

We'll simply use a (not quite) square region around our point of interest. The actual math is [a bit complex](https://en.wikipedia.org/wiki/Geographical_distance), because the Earth happens to be somewhat similar to an ellipsoid. If we cheat a bit and assume the earth is spherical, we get away with this formula:
我们可以简单的用一个（不完全）正方形区域围绕我们的兴趣点。实际在数学上这[有些复杂] (https://en.wikipedia.org/wiki/Geographical_distance)，因为地球恰好有点类似于一个椭球。如果我们假设地球是球体，则可以得到这个公式：

    D = R * sqrt( (deltaLatitude * deltaLatitude) +
	              (cos(meanLatitidue) * deltaLongitude) * (cos(meanLatitidue) * deltaLongitude))

We end up with something like this (all approximate):
我们用以下内容作为结束（均为近似值）：

    static double const R = 6371009000; // Earth readius in meters
    double deltaLatitude = D / R * 180 / M_PI;
    double deltaLongitude = D / (R * cos(meanLatitidue)) * 180 / M_PI;

Our point of interest is:
我们的兴趣点是：

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
（当我们接近 180° 经线的时候，这个公式不成立。由于我们的交通数据源于离 180° 经线很远很远的柏林，所以我们忽略这个问题。）

    request.result = [NSPredicate predicateWithFormat:
                      @"(%@ <= longitude) AND (longitude <= %@)"
                      @"AND (%@ <= latitude) AND (latitude <= %@)",
                      @(minLongitude), @(maxLongitude), @(minLatitude), @(maxLatitude)];

There's no point in specifying a sort descriptor. Since we're going to be doing a second in-memory pass over all objects, we will, however, ask Core Data to fill in all values for all returned objects:
指定一种排序描述符毫无意义。因为我们会用另一个内存来传递所有对象，不过我们将用 Core Data 填入所有返回对象的所有的值。

    request.returnsObjectsAsFaults = NO;

Without this, Core Data will fetch all values into the persistent store coordinator's row cache, but it will not populate the actual objects. Often that makes sense, but since we'll immediately be accessing all of the objects, we don't want that behavior.
否则，core data将进入持久化存储协调器的row cache读取所有的值，不过它不会填充实际对象。这往往是可行的，不过由于我们将立刻访问所有对象，我们并不希望出现这种行为。

As a safe-guard, it's good to add:
为安全防范考虑，最好加上：

    request.fetchLimit = 200;

We execute this fetch request:
执行这条 fetch 请求

    NSError *error = nil;
    NSArray *stops = [moc executeFetchRequest:request error:&error];
    NSAssert(stops != nil, @"Failed to execute %@: %@", request, error);

The only (likely) reasons the fetch would fail is if the store went corrupt (file was deleted, etc.) or if there's a syntax error in the fetch request. So it's safe to use `NSAssert()` here.
读取失败唯一（可能）的原因是储存器损坏（文件被删除等等），或者 fetch 请求中出现语法错误。所以在这里使用 `NSAssert()` 是安全的。

We'll now do the second pass over the in-memory data using Core Locations advance distance math:
我们现在使用 Core Locations 推进距离数学，为内存中的数据做第二个通行证。

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
至此我们完成了全部设置。

#### Geo-Location Performance
#### 地理定位性能


These fetches take around 360µs on average on a recent MacBook Pro with SSD. That is, you can do approximately 2,800 of these requests per second. On an iPhone 5 we'd be getting around 1.67ms on average, or some 600 requests per second.
使用装载了 SSD 硬盘的新一代 MacBook Pro 读取这些数据平均约需要 360µs，也就是说，你每秒可以做大约 2800 次请求。iPhone 5 平均约需要 1.67ms，每秒 600 次请求。

If we add `-com.apple.CoreData.SQLDebug 1` as launch arguments to the app, we get this output:
如果加上 `-com.apple.CoreData.SQLDebug1` 作为启动参数传递给应用程序，我们将得到如下输出：

    sql: SELECT 0, t0.Z_PK, t0.Z_OPT, t0.ZIDENTIFIER, t0.ZLATITUDE, t0.ZLONGITUDE, t0.ZNAME FROM ZSTOP t0 WHERE (? <=  t0.ZLONGITUDE AND  t0.ZLONGITUDE <= ? AND ? <=  t0.ZLATITUDE AND  t0.ZLATITUDE <= ?)  LIMIT 100
    annotation: sql connection fetch time: 0.0008s
    annotation: total fetch execution time: 0.0013s for 15 rows.

In addition to some statistics (for the store itself), this shows us the generated SQL for these fetches:
除了一些统计信息（对于存储本身），还向我们展示了读取到的数据生成的SQL：

    SELECT 0, t0.Z_PK, t0.Z_OPT, t0.ZIDENTIFIER, t0.ZLATITUDE, t0.ZLONGITUDE, t0.ZNAME FROM ZSTOP t0
	WHERE (? <=  t0.ZLONGITUDE AND  t0.ZLONGITUDE <= ? AND ? <=  t0.ZLATITUDE AND  t0.ZLATITUDE <= ?)
	LIMIT 200

which is what we'd expect. If we'd want to investigate the performance, we can use the SQL [`EXPLAIN` command](https://www.sqlite.org/eqp.html). For this, we'd open the database with the command line `sqlite3` command like this:
这正是我们所期望的。如果我们想要对这个性能进行调查研究，我们可以使用 SQL [`EXPLAIN` command](https://www.sqlite.org/eqp.html)。为此，我们可以像下面这样使用命令行 `sqlite3` 来打开数据库：

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
这告诉我们 SQLite 为 `(ZLONGITUDE>? AND ZLONGITUDE<?)` 条件使用了 `ZSTOP_ZLONGITUDE_INDEX`。我们像 [model article][260] 中描述的那样使用 *compound index* 则会做的更好。由于我们总是为了效率而同时搜索经纬度，而且我们可以去除经度和纬度各自的指数。

This would make the output look like this:
这将使输出像下面这样：

    0|0|0|SEARCH TABLE ZSTOP AS t0 USING INDEX ZSTOP_ZLONGITUDE_ZLATITUDE (ZLONGITUDE>? AND ZLONGITUDE<?) (~6944 rows)

In our simple case, adding a compound index hardly affects performance.
在我们的简单案例中加上复合索引几乎不影响性能。

As explained in the [SQLite Documentation](https://www.sqlite.org/eqp.html), the warning sign is a `SCAN TABLE` in the output. That basically means that SQLite needs to go through *all* entries to see which ones are matching. Unless you store just a few objects, you'd probably want an index.

就像在 [SQLite 文档](https://www.sqlite.org/eqp.html) 中的说明一样，警报信号在输出中是一个 `SCAN TABLE`。这基本上意味着 SQLite 需要遍历 *所有的* 记录来看看那些是相匹配的。

### Subqueries
### 子查询


Let's say we only want those stops near us that are serviced within the next twenty minutes.
假设我们只想要那些接近我们的且在接下来20分钟之内提供服务的停止对象。

We can create a predicate for the *StopTimes* entity like this:
我们可以像这样为 *停止时间* 实体创建一个谓词：

    NSPredicate *timePredicate = [NSPredicate predicateWithFormat:@"(%@ <= departureTime) && (departureTime <= %@)",
                                  startDate, endDate];

But what if what we want is a predicate that we can use to filter *Stop* objects based on the relationship to *StopTime* objects, not *StopTime* objects themselves? We can do that with a `SUBQUERY` like this:
但是如果我们想要的谓词是可以用来过滤哪些是基于与 *停止时间* 对象的关系之上的 *停止* 对象，而不是 *停止时间* 对象本身，我们可以使用一个这样的 `子查询` ：

    NSPredicate *predicate = [NSPredicate predicateWithFormat:
                              @"(SUBQUERY(stopTimes, $x, (%@ <= $x.departureTime) && ($x.departureTime <= %@)).@count != 0)",
                              startDate, endDate];

Note that this logic is slightly flawed if we're close to midnight, since we ought to wrap by splitting the predicate up into two. But it'll work for this example.
请注意，如果接近午夜，这个逻辑是稍有瑕疵的，因为我们应当将谓词一分为二。不过这个逻辑在这个例子中是可行的。

Subqueries are very powerful for limiting data across relationship. The Xcode documentation for [`-[NSExpression expressionForSubquery:usingIteratorVariable:predicate:]`](https://developer.apple.com/library/ios/documentation/cocoa/reference/foundation/Classes/NSExpression_Class/Reference/NSExpression.html#//apple_ref/occ/clm/NSExpression/expressionForSubquery:usingIteratorVariable:predicate:) has more info. 

对于限制数据在关系之上的，子查询非常有用。在 Xcode 文档[`-[NSExpression expressionForSubquery:usingIteratorVariable:predicate:]`](https://developer.apple.com/library/ios/documentation/cocoa/reference/foundation/Classes/NSExpression_Class/Reference/NSExpression.html#//apple_ref/occ/clm/NSExpression/expressionForSubquery:usingIteratorVariable:predicate:) 中有更多信息。

We can combine two predicates simply using `AND` or `&&`, i.e.
我们可以简单的使用 `and` 或者 `&&` 来组合两个谓词，例如：


    [NSPredicate predicateWithFormat:@"(%@ <= departureTime) && (SUBQUERY(stopTimes ....

or in code using [`+[NSCompoundPredicate andPredicateWithSubpredicates:]`](https://developer.apple.com/library/ios/DOCUMENTATION/Cocoa/Reference/Foundation/Classes/NSCompoundPredicate_Class/Reference/Reference.html#//apple_ref/occ/clm/NSCompoundPredicate/andPredicateWithSubpredicates:).

或者在代码中使用 [`+[NSCompoundPredicate andPredicateWithSubpredicates:]`](https://developer.apple.com/library/ios/DOCUMENTATION/Cocoa/Reference/Foundation/Classes/NSCompoundPredicate_Class/Reference/Reference.html#//apple_ref/occ/clm/NSCompoundPredicate/andPredicateWithSubpredicates:)。

We end up with a predicate that looks like this:
我们用一个像这样的谓词来作为结束：

    (lldb) po predicate
    (13.39657778010461 <= longitude AND longitude <= 13.42266155792719
		AND 52.63249629924865 <= latitude AND latitude <= 52.64832433715332)
		AND SUBQUERY(
            stopTimes, $x, CAST(-978250148.000000, "NSDate") <= $x.departureTime 
            AND $x.departureTime <= CAST(-978306000.000000, "NSDate")
        ).@count != 0

#### Subquery Performance
#### 子查询性能


If we look at the generated SQL it looks like this:
如果我们看一下生成的 SQL ，就像下面这样：

    sql: SELECT 0, t0.Z_PK, t0.Z_OPT, t0.ZIDENTIFIER, t0.ZLATITUDE, t0.ZLONGITUDE, t0.ZNAME FROM ZSTOP t0
	     WHERE ((? <=  t0.ZLONGITUDE AND  t0.ZLONGITUDE <= ? AND ? <=  t0.ZLATITUDE AND  t0.ZLATITUDE <= ?)
		        AND (SELECT COUNT(t1.Z_PK) FROM ZSTOPTIME t1 WHERE (t0.Z_PK = t1.ZSTOP AND ((? <=  t1.ZDEPARTURETIME AND  t1.ZDEPARTURETIME <= ?))) ) <> ?)
		 LIMIT 200

This fetch request now takes around 12.3 ms to run on a recent MacBook Pro. On an iPhone 5, it'll take about 110 ms. Note that we have three million stop times and almost 13,000 stops.
这个 fetch 请求在新一代 MacBook Pro 上运行大约需要 12.3 ms。在 iPhone 5 上，大约需要 110 ms。请注意，我们有 3,000,000 个停止时间 和将近 13,000 个车站。

The query plan explanation looks like this:
这个查询计划的解释如下：
    sqlite> EXPLAIN QUERY PLAN SELECT 0, t0.Z_PK, t0.Z_OPT, t0.ZIDENTIFIER, t0.ZLATITUDE, t0.ZLONGITUDE, t0.ZNAME FROM ZSTOP t0
       ...> WHERE ((13.37190946378911 <=  t0.ZLONGITUDE AND  t0.ZLONGITUDE <= 13.3978625285315 AND 52.41186440524024 <=  t0.ZLATITUDE AND  t0.ZLATITUDE <= 52.42769244314491) AND
       ...> (SELECT COUNT(t1.Z_PK) FROM ZSTOPTIME t1 WHERE (t0.Z_PK = t1.ZSTOP AND ((-978291733.000000 <=  t1.ZDEPARTURETIME AND  t1.ZDEPARTURETIME <= -978290533.000000))) ) <> ?)
       ...> LIMIT 200;
    0|0|0|SEARCH TABLE ZSTOP AS t0 USING INDEX ZSTOP_ZLONGITUDE_ZLATITUDE (ZLONGITUDE>? AND ZLONGITUDE<?) (~3472 rows)
    0|0|0|EXECUTE CORRELATED SCALAR SUBQUERY 1
    1|0|0|SEARCH TABLE ZSTOPTIME AS t1 USING INDEX ZSTOPTIME_ZSTOP_INDEX (ZSTOP=?) (~2 rows)

Note that it is important how we order the predicate. We want to put the longitude and latitude stuff first, since it's cheap, and the subquery last, since it's expensive.
请注意，我们如何队谓词排序非常重要。我们希望把经纬度排列在首位，因为代价低，而子查询由于代价高将其排列在最后。

### Text Search
### 文本搜索


A common scenario is searching for text. In our case, let's look at searching for *Stop* entities by their name.
搜索文本是一种常见的情况。在我们的例子中，来看看使用名称来搜索 *车站* 实体。

Berlin has a stop called "U Görlitzer Bahnhof (Berlin)". A naïve way to search for that would be:
柏林有个被称为 "U Görlitzer Bahnhof (Berlin)" 的车站。一种天真的搜索该站的方法如下：

    NSString *searchString = @"U Görli";
    predicate = [NSPredicate predicateWithFormat:@"name BEGINSWITH %@", searchString];

Things get even worse if you want to be able to do:
如果你想按照如下所示做的话，事情会变得更糟：

    name BEGINSWITH[cd] 'u gorli'
 
i.e. do a case and / or diacritic insensitive lookup.
例如：进行一项大小写和 / 或音调不敏感的查询。

Things are not that simple, though. Unicode is very complicated and there are quite a few gotchas. First and foremost ís that many characters can be represented in multiple ways. Both [U+00F6](http://unicode.org/charts/PDF/U0080.pdf) and [U+006F](http://www.unicode.org/charts/PDF/U0000.pdf) [U+0308](http://unicode.org/charts/PDF/U0300.pdf) represent "ö." And concepts such as uppercase / lowercase are very complicated once you're outside the ASCII code points.

尽管如此，事情并不是那么简单。Unicode 非常复杂，并且有很多陷阱。首要的是很多字符可以通过多种方式来表示。
[U+00F6](http://unicode.org/charts/PDF/U0080.pdf) 和 [U+006F](http://www.unicode.org/charts/PDF/U0000.pdf)都代表 [U+0308](http://unicode.org/charts/PDF/U0300.pdf)都可以表示 "ö."。如果你不使用 ASCII 码，像大写 / 小写这样的概念就会非常复杂。

SQLite will do the heavy lifting for you, but it comes at a price. It may seem straightforward, but it's really not. What we want to do for string searches is to have a *normalized* version of the field that you can search on. We'll remove diacritics and make the string lowercase and then put that into a`normalizedName` field. We'll then do the same to our search string. SQLite then won't have to consider diacritics and case, and the search will effectively still be case and diacritics insensitive. But we have to do the heavy lifting up front.

SQLite 会为你减轻负担，但它是要付出代价的。虽然它看起来简单，但事实并非如此。对于字符串搜索，我们想做的是在我们搜索的领域内有一个 *规范化* 的版本。我们将消除音调符号，让字符串成为小写字母，然后将其列入一个 *规范名称* 领域。然后我们将对搜索字符串做同样的事情。然后 SQLite 就不必考虑音调和大小写，在大小写和音调不敏感的情况下，搜索仍然有效。但是我们必须先完成繁重的任务。

Searching with `BEGINSWITH[cd]` takes around 7.6 ms on a recent MacBook Pro with the sample strings in our sample code (130 searches / second). On an iPhone 5 those numbers are 47 ms per search and 21 searches per second.

在最新一代 MacBook Pro 上，使用示例代码中的示例字符串（130次搜索 / 秒）搜索 `BEGINSWITH[cd]` 需要 7.6ms，在 iPhone 5 上这个数字是每次搜索 47ms，每秒进行 21 次搜索。

To make a string lowercase and remove diacritics, we can use `CFStringTransform()`:
为了将字符串转换为小写和移除其音调，我们可以使用 `CFStringTransform()`：

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
我们将更新 `车站` 类来自动更新 `规范名称`：

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
有了下面的代码，我们可以用 `BEGINSWITH` 代替 `BEGINSWITH[cd]` 来搜索：

    predicate = [NSPredicate predicateWithFormat:@"normalizedName BEGINSWITH %@", [searchString normalizedSearchString]];

Searching with `BEGINSWITH` takes around 6.2 ms on a recent MacBook Pro with the sample strings in our sample code (160 searches / second). On an iPhone 5 it takes 40ms corresponding to 25 searches / second.

在最新一代 MacBook Pro 上，使用示例代码中的示例字符串（160 次搜索 / 秒）搜索 `BEGINSWITH`
需要6.2ms，在 iPhone 5 大约上需要 40ms，25 次搜索 / 秒。

#### Free Text Search
#### 自由文本搜索

Our search still only works if the beginning of the string matches the search string. The way to fix that is to create another *Entity* that we search on. Let's call this Entity `SearchTerm`, and give it a single attribute `normalizedWord` and a relationship to a `Stop`. For each `Stop` we would then normalize the name and split it into words, e.g.:

我们的搜索还只能在字符串的开头和搜索字符串相匹配的情况下有效。要解决这个问题就要创建另一个用来搜索的 *实体*。我们称这个实体为 `搜索关键词`，给其一个唯一的 `规范词` 属性，以及一个和 `车站` 的关系。对于每个`车站` 我们将规范它们的名称，以及将其拆分成一个个词。例如：

        "Gedenkstätte Dt. Widerstand (Berlin)"
	->  "gedenkstatte dt. widerstand (berlin)"
	->  "gedenkstatte", "dt", "widerstand", "berlin"

For each word, we create a `SearchTerm` and a relationship from the `Stop` to all its `SearchTerm` objects. When the user enters a string, we search on the `SearchTerm` objects' `normalizedWord` with:

对于每个词。我们创建一个 `搜索关键词` 和一个从 `车站` 到它的所有 `搜索关键词` 对象的关系。当用户输入一个字符串，我们用以下代码在 `搜索关键词` 对象的 `规范词` 上搜索：

	predicate = [NSPredicate predicateWithFormat:@"normalizedWord BEGINSWITH %@", [searchString normalizedSearchString]]

This can also be done in a subquery directly on the `Stop` objects.
这也可以在 `车站`对象中直接用子查询完成。

## Fetching All Objects
## 读取所有对象

If we don't set a predicate on our fetch request, we'll retrieve all objects for the given *Entity*. If we did that for the `StopTimes` entity, we'll be pulling in three million objects. That would be slow, and use up a lot of memory. Sometimes, however, we need to get all objects. The common example is that we want to show all objects inside a table view.

如果我们的 fetch 请求中没有设置谓词，我们将为给定 *实体* 检索所有对象。如果我们对 `停止时间` 实体这样做的话，我们将会牵涉 300 万个对象。这将会变得缓慢，以及占用大量内存。然而有时候，我们需要获取所有对象。常见的例子是我们想要在一个 table view 中显示所有对象。

What we would do in this case, is to set a batch size:
在这种情况中，我们要做的是设置批处理量：
  
    request.fetchBatchSize = 50;

When we run `-[NSManagedObjectContext executeFetchRequest:error:]` with a batch size set, we still get an array back. We can ask it for its count (which will be close to three million for the `StopTime` entity), but Core Data will only populate it with objects as we iterate through the array. And Core Data will get rid of objects again, as they're no longer accessed. Simply put, the array has batches of size 50 (in this case). Core Data will pull in 50 objects at a time. Once more than a certain number of these batches are around, Core Data will release the oldest batch. That way you can loop through all objects in such an array, without having to have all three million objects in memory at the same time.

当我们设置了批处理量运行 `-[NSManagedObjectContext executeFetchRequest:error:]` 的时候，我们仍然会得到一个返回的数组。我们可以查询它的大小（对于 `停止时间` 实体而言，这将接近 300 万），不过 Core Data 将只会随着我们对数组的循环访问将对象填充进去。如果这些对象不再被访问，Core Data 则会再次清理对象。简单来说，数组的批处理量为 50（在这个例子中）。Core Data 将一次获取50个对象。一旦有超过一定数量的批量对象，Core Data 将释放最旧一批对象。于是，你就可以在这样的数组中循环访问所有对象，而无需在存储器中同时存所有 300 万个对象。

On iOS, when you use an `NSFetchedResultsController` and you have a lot of objects, make sure that the `fetchBatchSize` is set on your fetch request. You'll have to experiment with what size works well for you. Twice the amount of objects that you'll be displaying at any given point in time is a good starting point.

在 iOS 中，如果你使用 `NSFetchedResultsController` 且有很多对象，请确保你的 fetch 请求中设置了 `fetchBatchSize`。你不得不测试多少处理量更适合你。在一开始最好让对象数目翻倍，这样你就可以在任何指定的时间点及时显示对象。

译文 [Fetch 请求](https://github.com/answer-huang/objcio_cn/blob/master/%E7%BF%BB%E8%AF%91%E5%AE%8C%E6%88%90/Fetch%20Request)
