# About

This project demonstrates the RSS increase in Netty with Java 11 graalvm native image.

## Issue

In native-image, when netty receives http requests, the amount of RSS increase is much higher in Java 11 than Java 8.

## Testing env

The http server code, `HttpHelloWorldServer`, is copied from [the netty example](https://github.com/netty/netty/tree/4.1/example/src/main/java/io/netty/example/http/helloworld).

GraalVM CE 21.2.0 in container.

Netty 4.1.68.Final.

## Observation

| Num of req | RSS in Java 8 | RSS in Java 11 | Remarks
|------------|---------------|----------------|----------
| 0          | 17540         | 17132          | At start, it is almost same RSS
| 1          | 19364         | 35216          |
| 2          | 19892         | 52108          |
| 3          | 20416         | 69000          |
| 4          | 20940         | 85892          |
| 5          | 21464         | 102784         |
| 6          | 21988         | 119676         |
| 7          | 22512         | 136568         |
| 8          | 23036         | 153460         |
| 9          | 23560         | 170352         |
| 10         | 24084         | 187244         |
| 11         | 24608         | 204140         |
| 12         | 25136         | 221032         |
| 13         | 25660         | 237924         |
| 14         | 26184         | 254816         |
| 15         | 26708         | 271708         |
| 16         | 27232         | 288600         | RSS becomes stable
| 17         | 27232         | 288600         |
| 18         | 27232         | 288600         |
| 19         | 27232         | 288600         |
| 20         | 27232         | 288600         |

Initially, both Java 8 and 11 have similar size of RSS.
When the netty http server receives a http request, RSS constantly increases.
The RSS increase in Java 8 is about 500KB per request, whereas 16MB in Java 11.

Once the number of received requests reached 16, the value of RSS becomes stable.
After the RSS becomes stable, it sometimes fluctuates but doesn't change much.


The `PooledByteBufAllocator.DEFAULT.metric()` showed as followings:
```
...
req=13: PooledByteBufAllocatorMetric(usedHeapMemory: 0; usedDirectMemory: 218103808; numHeapArenas: 16; numDirectArenas: 16; smallCacheSize: 256; normalCacheSize: 64; numThreadLocalCaches: 13; chunkSize: 16777216)
req=14: PooledByteBufAllocatorMetric(usedHeapMemory: 0; usedDirectMemory: 234881024; numHeapArenas: 16; numDirectArenas: 16; smallCacheSize: 256; normalCacheSize: 64; numThreadLocalCaches: 14; chunkSize: 16777216)
req=15: PooledByteBufAllocatorMetric(usedHeapMemory: 0; usedDirectMemory: 251658240; numHeapArenas: 16; numDirectArenas: 16; smallCacheSize: 256; normalCacheSize: 64; numThreadLocalCaches: 15; chunkSize: 16777216)
req=16: PooledByteBufAllocatorMetric(usedHeapMemory: 0; usedDirectMemory: 268435456; numHeapArenas: 16; numDirectArenas: 16; smallCacheSize: 256; normalCacheSize: 64; numThreadLocalCaches: 16; chunkSize: 16777216)
req=17: PooledByteBufAllocatorMetric(usedHeapMemory: 0; usedDirectMemory: 268435456; numHeapArenas: 16; numDirectArenas: 16; smallCacheSize: 256; normalCacheSize: 64; numThreadLocalCaches: 16; chunkSize: 16777216)
req=18: PooledByteBufAllocatorMetric(usedHeapMemory: 0; usedDirectMemory: 268435456; numHeapArenas: 16; numDirectArenas: 16; smallCacheSize: 256; normalCacheSize: 64; numThreadLocalCaches: 16; chunkSize: 16777216)
...
```

The reason RSS became stable is most likely the cache size(`numThreadLocalCaches`) is set to 16. 

The `numThreadLocalCaches` metric is associated to the `PoolThreadCache` class.
This class handles the memory assignment with jemalloc algorithm according to its javadoc.
My guess is that the implementation of this memory allocation may be causing different memory handling behavior in jdk.


## Setup

Steps:
- Create docker images with graalvm java 8 and 11.
- Run the image mounting the project dir to `/opt/project`.
- Build a native image app
- Run the app
- Make http requests and check RSS

Create docker images
```
# for java8
docker build -f Dockerfile-java8 . --tag=graalvm-ce:21.2.0-native-java8
docker run --rm -it -v <path to native-netty-sample>:/opt/project graalvm-ce:21.2.0-native-java8 bash

# for java11
docker build -f Dockerfile-java11 . --tag=graalvm-ce:21.2.0-native-java11
docker run --rm -it -v <path to native-netty-sample>:/opt/project graalvm-ce:21.2.0-native-java11 bash
```

Build project in docker
```
cd /opt/project
./mvnw -Pnative package
```

Run and check RSS
```
./target/example &

# initial RSS
APP_PID=`ps -ef | grep example | grep -v grep | awk '{print $2}'`
ps -o rss $APP_PID

# make request and get RSS
for i in `seq 20`; do curl -s -o /dev/null localhost:8080; echo "req=$i RSS=`ps -o rss= $APP_PID`"; done
```

