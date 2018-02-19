# Prerequistes

"Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a **strong negative impact when debugging and maintenance are considered**. We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%." - Donald Knuth

When profiling, a number of programs were used.

- tcpdump
useful for monitoring load testers. The command

```bash
$ tcpdump -l -s 1024 -A -i lo0 'tcp port 3000' | grep -A 6 "HTTP"
```

was especially useful for monitoring HTTP traffic.

- siege
command line tool for load testing. The advantage of siege is that curl requests are easily translatable

https://groups.google.com/forum/#!topic/siege-users/sO-H1myTSMg

example command line usage

```bash
$ siege -b -c 10 -r 10 -H 'authorization: bearer 74ccfd9f15da412190de0de2f1c08e27' -H 'content-type: application/json' 'http://localhost:3000/graphql POST {"query":"query {\n  game(name: \"Greedy Pigeon\"){\n    key\n  }\n}","variables":{"url":"https://github.com/jborden"}}'
```

Note that the '<hostname> <method> <body>' are contained in single quotes, this differs from curl

- JMeter
The GUI is used to create tests and debug them. When running actual load tests, it should be run from
the command line, for example:

```bash
$ jmeter -n -t game_test.jmx -l testresult.jtl
```

jmeter resources:

http://blog.e-zest.com/how-to-run-jmeter-in-non-gui-mode/

https://varunver.wordpress.com/2014/05/07/jmeter-posting-json-to-api/

Some notes:
- make sure the protocol http: is not included in host portion
- make sure you are using POST if you are sending JSON in body
- the GUI quickly becomes overloaded when doing load tests, MUST use from the command line
- JMeter is VERY fast, especially in comparison to siege. siege has an issue when too many threads are used, it seemingly has to stop and restart (CPU usage is almost max, goes down to threshold levels, and then peaks again)
- a comparison of load testers at https://www.blazemeter.com/blog/open-source-load-testing-tools-which-one-should-you-use shows that JMeter is on top

Profiling:

visualVM is used.

When starting a server from 'lein ring server', the second Clojure PID is the actual server.
- Initial instrumentation takes some time. Be sure that it is complete and then run a few simple
curl requests to make sure everything is running at speed. There might be an initial delay in response from the server before things settle down.

- Sampler and Profiler yield very similar results. However, Sampler is much more responsive.
http://stackoverflow.com/questions/12130107/difference-between-sampling-and-profiling-in-jvisualvm

- Be sure to note the 'Setting' checkbox towards the top right. This enables additional filters on packages. Helpful for narrowing down performance issues.

- By far the biggest bottleneck are calls to org.eclipse.jetty.util.blockingarrayqueue.poll(). Though this issue should be explored more, excluding the org.eclipse.jetty.util.* package can be helpful in determining which bottleneck you can control

sampler:

running 'lein ring server', game query, preoptimizations

```bash
$ jmeter -n -t game_query_test.jmx -l testresult.jtl
Writing log file to: /Users/james/jborden/leaderboard-api/test/jmeter/jmeter.log
Creating summariser <summary>
Created the tree successfully using game_query_test.jmx
Starting the test @ Fri May 19 12:14:57 CDT 2017 (1495214097746)
Waiting for possible Shutdown/StopTestNow/Heapdump message on port 4445
summary +    484 in 00:00:02 =  222.0/s Avg:   302 Min:    71 Max:   564 Err:     0 (0.00%) Active: 100 Started: 100 Finished: 0
summary +   9516 in 00:00:28 =  341.2/s Avg:   291 Min:    16 Max:   527 Err:     0 (0.00%) Active: 0 Started: 100 Finished: 100
summary =  10000 in 00:00:30 =  332.5/s Avg:   291 Min:    16 Max:   564 Err:     0 (0.00%)
Tidying up ...    @ Fri May 19 12:15:27 CDT 2017 (1495214127892)
... end of run
```bash

![jetty game query preoptimization](/assets/images/jetty_game_query_preopt.png)

The calls to org.eclipse.jetty.util.BlockingArrayQueue.poll() are taking up a lot of resources. How does this compare with running the server using Tomcat 7?

Steps for installing Tomcat locally:

1. download latest release from https://tomcat.apache.org/download-70.cgi
2. extract dir
3. rm -rf * in webapps/ dir
4. leaderboard-api$ lein ring uberwar
5. leaderboard-api$ mv target/leaderboard-api-0.1.0-SNAPSHOT-standalone.war ~/src/apache-tomcat-7.0.78/webapps/ROOT.war
6. apache-tomcat-7.0.78$ chmod +x bin/startup.sh bin/shutdown.sh bin/catalina.sh
7. apache-tomcat-7.0.78$ rm -rf webapps/*
8. apache-tomcat-7.0.78$ bin/startup.sh
9. setup bin/startup.sh
   example file:
   export OPENSHIFT_PG_HOST=localhost
   export OPENSHIFT_PG_PORT=5432
   export OPENSHIFT_PG_DATABASE=leaderboard
   export OPENSHIFT_PG_USERNAME=leaderboard
   export OPENSHIFT_PG_PASSWORD=leaderboardbackerboard
10. create the war, put it into the dir
	leaderboard-api$ lein ring uberwar ; rm -rf ~/src/apache-tomcat-7.0.78/webapps/* ; cp target/leaderboard-api-0.1.0-SNAPSHOT-standalone.war ~/src/apache-tomcat-7.0.78/webapps/ROOT.war
11. visit http://localhost:8080

# Tomcat

org.apache.tomcat.util.threads.TaskQueue.poll() now becomes the dominant resource user. 

![tomcat preopt](/assets/images/tomcat_game_query_preopt.png)

However, there also calls to org.postgresql.core.VisibleBufferedInputStream.readMore()

# Cache system
