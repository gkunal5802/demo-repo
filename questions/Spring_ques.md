## Production and Scenarios based questions

1. What actually breaks when two threads write to the same HashMap?
- HashMap is not thread-safe. Concurrent writes can cause lost updates, inconsistent internal state, and problems during resizing because multiple threads can modify the bucket array without synchronization.

2. Your counter is volatile and still gives the wrong total. Why?
- volatile buys visibility, not atomicity. count++ is read, modify, write. Three steps, no lock. Fix is AtomicInteger, Synchronization or LongAdder under heavy contention.

3. You put Transactional on a method and called it from another method in the same class. What happens?
- Spring's @Transactional is implemented through AOP proxies. If a transactional method is called from another method in the same class using this.method(), the call bypasses the Spring proxy, so the transactional advice on the called method is not applied. If the outer method itself is transactional, however, the inner method executes within that existing transaction.

4. Your service runs 200 threads on a pool of 10 connections. Traffic triples. CPU sits at 20%. What is going on?
- Your threads are parked waiting on HikariCP, not working. The bottleneck is the pool, not the CPU. Add more servers and it gets worse.

5. Why is double checked locking broken without volatile?
- Another thread can see a non null reference to a half constructed object. The write gets reordered. Rare, real, and brutal to debug.

6. Your query has an index and the database still does a full scan. Why?
- A function wrapped around the indexed column. Wrong leading column of a composite index. Implicit type conversion. All three kill it.

7. Where does a memory leak hide in a language that has garbage collection?
- Static collections that only grow. ThreadLocals never removed inside a thread pool. Listeners never unregistered. GC cannot free what you are still holding.

8. What happens to your in flight requests when Kubernetes sends SIGTERM?
- Without graceful shutdown they die mid write. Half the answer is the shutdown hook. The other half is a preStop delay so traffic stops reaching you first.

9. Your Kafka consumer commits the offset before processing the message. What did you just choose?
- At most once. Crash after the commit and that message is gone forever. Commit after processing and you chose at least once, which means you now need idempotency.

10. You mutate a field on an object already used as a HashMap key. What happens to it?
- The hashCode changes. The entry stays in the old bucket. containsKey returns false for the exact object you put in. It is in the map and unreachable

11. What are indexes in sql?
- An index is a separate data structure maintained by the database to speed up data retrieval. Instead of scanning every row of a table, the database can use the index to quickly locate matching rows. Indexes are commonly used on columns involved in WHERE, JOIN, ORDER BY, and sometimes GROUP BY operations. However, indexes consume storage and add overhead to INSERT, UPDATE, and DELETE operations, so we shouldn't create unnecessary indexes.

## Core Spring Boot questions

1. How does Spring Boot determine which auto-configuration to apply?
2. What happens internally when you add spring-boot-starter-web?
3. Why does Spring Boot follow Convention over Configuration?
4. How does Spring Boot load application.properties internally?
5. What is the exact startup flow of a Spring Boot application?
6. What is the difference between @ComponentScan and @SpringBootApplication?
7. How does Spring Boot automatically detect embedded Tomcat?
8. What happens when two beans of the same type exist without @Qualifier?
9. How does Spring Boot load profile-specific configurations?
10. What is the role of SpringFactoriesLoader?
11. How did Spring Boot eliminate the need for XML configuration in most cases?
12. What is the internal difference between @RestController and @Controller?
13. How does Spring Boot manage dependency versions automatically?
14. What is the complete lifecycle of a Spring Bean?
15. How does Spring Boot handle externalized configuration?
16. What happens when both application.yml and application.properties exist?
17. How does Spring Boot integrate with Actuator internally?
18. What is the difference between a @Configuration class and a normal class?
19. How does Spring Boot automatically create a DataSource?
20. What is the practical use of CommandLineRunner?
21. How does Spring Boot handle exception translation?
22. What is the difference between @EnableAutoConfiguration and @Import?
23. What happens when you exclude an auto-configuration class?
24. Why is Spring Boot well-suited for microservices?
25. What is the difference between a fat JAR and a normal JAR?
26. How does Spring Boot handle logging by default?
27. How does Spring Boot determine server port priority?
28. What happens internally when you hit a REST endpoint?
29. Why is Spring Boot preferred for cloud-native applications?
30. What are the most common Spring Boot performance mistakes?
    
## ➤ Spring Boot & Microservices

1. How do you implement JWT authentication?
2. OAuth2 vs JWT?
3. RestTemplate vs WebClient?
4. How do microservices communicate?
5. What is API Gateway and why is it required?
6. How to implement Global Exception Handling?
7. How does @Transactional work internally?

## ➤ Database & JPA

1. First-level vs Second-level cache?
- The first-level cache is associated with a Hibernate Session / JPA EntityManager.
- The second-level cache is associated with the Hibernate SessionFactory / JPA persistence unit rather than a single session.
2. What is N+1 problem?
3. How to optimize slow queries?
4. How do you handle concurrent updates? - Optimistic and pessimistic locking. Atomic DB locking and distributed locking
5. IoC vs DI? 
- IoC is a broader design principle where control over object creation and dependency management is transferred from the application code to an external container or framework. Dependency Injection is one of the primary techniques used to achieve IoC, where dependencies are supplied to an object rather than the object creating them itself.

## 𝗖𝗼𝗻𝗰𝘂𝗿𝗿𝗲𝗻𝗰𝘆 & 𝗠𝘂𝗹𝘁𝗶𝘁𝗵𝗿𝗲𝗮𝗱𝗶𝗻𝗴
🔹 10 threads execute counter++ on a shared volatile int. Why can the final value be incorrect?
🔹 You replace volatile int with AtomicInteger. What changes internally?
🔹 When can CAS-based concurrency perform worse than locking?
🔹 A ThreadPoolExecutor has corePoolSize = 10, maxPoolSize = 50 and an unbounded queue. Why might it never create more than 10 threads?
🔹 A production service has 500 threads but CPU utilization is only 20%. What could those threads be doing?
🔹 Two services acquire locks A and B in opposite order. How would you prove a deadlock exists and eliminate it?
🔹 Why doesn't making every method synchronized automatically make a class thread-safe?

## 𝗦𝗽𝗿𝗶𝗻𝗴 & 𝗗𝗶𝘀𝘁𝗿𝗶𝗯𝘂𝘁𝗲𝗱 𝗦𝘆𝘀𝘁𝗲𝗺𝘀
🔹 Why might @Transactional have no effect when one method calls another method in the same class?
🔹 A transaction updates the database and then calls a payment service. Payment succeeds but the database transaction rolls back. How would you design around this?
🔹 Your API has a 2-second timeout but the downstream service takes 10 seconds. Where should the timeout be enforced?
🔹 A Kafka consumer crashes after updating the database but before committing the offset. What happens next?
🔹 How would you make Kafka message processing idempotent?

## 𝗣𝗿𝗼𝗱𝘂𝗰𝘁𝗶𝗼𝗻-𝗟𝗲𝘃𝗲𝗹 𝗣𝗿𝗼𝗯𝗹𝗲𝗺 𝗦𝗼𝗹𝘃𝗶𝗻𝗴
🔹 API latency increases from 200ms to 5 seconds, while CPU and heap usage remain normal. What do you investigate first?
🔹 Database CPU is only 40%, but the application is timing out while waiting for connections. What could cause this?
🔹 GC pauses are low and heap usage is stable, but the JVM keeps creating more threads. What could be happening?
🔹 A cache improves response time by 10× but occasionally returns stale financial data. How would you handle consistency?