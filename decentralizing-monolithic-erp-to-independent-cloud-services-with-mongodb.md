# Decentralizing Monolithic ERP to Independent Cloud Services with MongoDB

As the chief architect at [Hybrid Web Agency](https://hybridwebagency.com/), one of the most complex projects I've been involved in was the migration of an aging monolithic ERP system to a modern microservices architecture. The outdated monolith had become a costly burden to maintain and was impeding my client's ability to innovate effectively.

Fortunately, our extensive experience in delivering customized [Software Development Services in Colorado Springs](https://hybridwebagency.com/colorado-springs-co/best-software-development-company/) had equipped us for such challenges. We were intimately familiar with the challenges organizations face when dealing with rigid and hard-to-maintain systems. As a result, I'm thrilled to share the systematic approach we took to redesign the data model and decompose the monolith, both at the code and database levels.

Throughout this migration process, my team and I discovered several lesser-known best practices for structuring domain entities in a document database like MongoDB to support fully independent microservices. If you've ever wondered how to future-proof your database architecture for the cloud while preserving historical data, you're about to discover some valuable strategies.

By the time you finish reading this article, you'll have a clear roadmap for migrating your own legacy systems to a modern architecture. I'll provide practical insights to help you avoid common pitfalls and accelerate the delivery of new features to your customers. Let's dive right in!

## The Advantages of Embracing a Microservices Architecture

Opting for a microservices architecture offers numerous advantages compared to a monolithic approach. Microservices are designed to be independently deployable, allowing you to develop and release new features quickly without disrupting the entire application.

Furthermore, each microservice can be developed using different programming languages and frameworks, giving you the flexibility to choose the best tools for each specific domain. For example, you can leverage Python's machine learning libraries for a recommendation engine while using React to build the user interface.

This decoupling of functionalities empowers specialized teams to work autonomously on separate microservices. It also enables rapid idea validation through prototypes before committing to a full-scale monolithic rewrite. Onboarding new team members becomes more straightforward as they can contribute to a single microservice aligned with their expertise.

In a technology landscape marked by rapid changes, microservices mitigate the risk associated with technology shifts. When new technologies emerge, you can replace small, isolated parts of the system rather than embarking on a complete rewrite. Well-defined interfaces ensure that migrating components to new implementations is a seamless process.

#### Independent Scalability

Managing compute resources becomes more efficient when microservices can scale independently based on demand. For instance, a frontend API gateway can route traffic based on URLs and securely deploy behind a load balancer to handle traffic spikes. During peak periods, such as holidays, only the order processing microservice needs additional servers, avoiding unnecessary scaling of the entire system.

```
ordersAPI.scale(replicas: 5);
```

Horizontal scaling at the microservice level leads to significant cost savings through precise allocation of resources. Idle support microservices, such as user profiles, don't require costly overprovisioning to handle traffic that doesn't impact them.

## Analyzing the Data Model

### Understanding Entity Relationships

The initial step in migrating to a microservices architecture is analyzing how entities are interconnected within the monolithic data model. We meticulously examined each collection in the MongoDB database to identify clusters of domains and transaction boundaries.

Common entities like Users, Products, and Orders formed the core of bounded contexts. We scrutinized the relationships between these core objects, identifying candidates for service decomposition. For instance, we noticed that Orders contained foreign keys to Users for customer details and to Products to represent purchased items.

To gain a deeper understanding of these cross-dependencies, we created sample documents to visualize associated fields. This revealed that legacy code combined data that now belonged to separate business capabilities. For example, shipping addresses redundantly stored user profiles instead of utilizing lightweight references.

We employed database tools to explore these connections. In MongoDB Compass, we crafted diagrams of relations through $lookup pipelines and ran aggregate queries to tally references between entities. This process exposed crucial breakpoints for segmenting logic into coherent services.

These relationships informed the delineation of domain boundaries and ensured that services offered clean interfaces. Well-defined contracts empowered autonomous teams to develop and deploy modules incrementally, functioning as micro frontends without impeding each other.

### Identifying Transactional Boundaries

Beyond analyzing relationships, we delved into transactions within the existing codebase to comprehend the flow of business processes. This exercise helped us pinpoint where data modifications needed to be confined within individual services to uphold data consistency and integrity.

For instance, in the realm of order processing, we observed that any updates related to the order itself, associated payments, inventory levels, and shipment notifications needed to occur within a single service in a transactional manner. This insight guided the definition of our Order Management service boundary.

Our thorough analysis of both relationships and transactions provided invaluable insights for refactoring the data model and logic into independently deployable microservices with well-defined interfaces.

## Refactoring for Microservices

### Normalizing Data Schemas

To accommodate independent services that might utilize different data stores, we normalized schemas to eliminate redundancy and include only the essential data required by each service.

For example, the original Orders schema contained the entire User object. We refactored this to employ a lightweight reference:

```
// Before
Orders: {
  user: {
    name: 'John',
    address: '123 Main St...' 
  }
  //...
}

// After  
Orders: {
  userId: 1234
  //...  
}
```

Similarly, we segregated product details from Orders into their own collections, allowing these entities to evolve independently over time.

### Applying Domain-Driven Design

We embraced the principles of bounded contexts from Domain-Driven Design to logically segregate services, such as Order Fulfillment and User Profiles. Interfaces abstracted data access:

```
interface UserRepository {
  getUser(id): User;
  updateProfile(user: User): void;
}

class MongoUserRepository implements UserRepository {

  users = db.collection('users');

  async getUser(id) {
    return this.users.findOne({_id: id}); 
  }

  //...
}
```

### Evolving Data Access Patterns

Queries and commands also underwent refactoring to align with the new architecture. Previously, services accessed data directly using calls like `db.collection.find()`. We introduced abstraction through data access libraries:

```
// order.service.ts

constructor(private ordersRepo: OrderRepository) {}

getOrders() {
  return this.ordersRepo.find();
}

// mongodb.repo.ts

@Injectable()
class MongoOrderRepository implements OrderRepository {

  constructor(private mongoClient: MongoClient) {}

  find() {
   return this.mongoClient.db.collection('orders').find(); 
  }

}
```

This ensured flexibility in migrating databases without necessitating changes to consumer code.

## Deploying Microservices

### Independent Scaling

In the microservices paradigm, autoscaling is performed at the individual service level rather than at the application level. We implemented scaling logic using Docker Swarm and Kubernetes.

Deployment manifests outlined scaling policies based on CPU and memory utilization:

```yaml
# docker-compose.yml

services:
  orders:
    image: orders
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
      restart_policy: any
      placement:
        constraints: [node.role == worker]

  
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      scaler:
        min_replicas: 2    
        max_replicas: 6
```

This tiered scaling approach provided reserve capacity and adeptly managed elastic overflow. The orders service autonomously monitored its performance and spawned or terminated containers as needed:

```js
// orders.js

const cpusUsed = process.cpuUsage();

if (cpusUsed > 80) {
  swarm.scaleService('orders', 1);
}

if (cpusUsed < 20) {
  swarm.scaleService('orders', -1);  
}
```

Load balancers like Nginx and Traefik diligently routed traffic to scaled replica sets. This optimization not only enhanced resource utilization but also bolstered throughput while concurrently reducing operational costs.

### Implementing Resilience

Our arsenal of resiliency techniques included retry policies, timeouts, and circuit breakers. Rate limiting and throttling acted as safeguards against cascading failures, while the Platform service shouldered the responsibility of transient error policies for dependent services.

Homegrown and open-source solutions, including Polly, Hystrix, and Resilience4j, stood as stalwart guardians against potential mishaps. Centralized logging through Elasticsearch proved instrumental in tracing errors across the vast landscape of distributed applications.

### Ensuring Reliability

Incorporating reliability into microservices demanded the implementation of a spectrum of techniques designed to stave off single points of failure. We prioritized automated responses to transient errors and designed measures to tackle scenarios of overload.

Leveraging the Resilience4J library, we implemented circuit breakers to gracefully manage faults:

```java
// OrdersService.java

@Slf4j
@Service
public class OrdersService {

  @CircuitBreaker(name="ordersService", fallbackMethod="fallback")
  public List<Order> getOrders() {
    return orderRepository.getOrders();
  }

  public List<Order> fallback(Exception e) {
    log.error("Circuit open, returning empty orders");
    return Collections.emptyList();
  }
}
```

Rate limiting was introduced to mitigate the risk of service flooding during periods of heightened stress:

```java 
// RateLimiterConfig.java

@Configuration
public class RateLimiterConfig {

  @Bean
  public RateLimiter ordersLimiter() {
    return RateLimiter.create(maxOrdersPerSecond); 
  }

}
```

Timeouts were meticulously enforced to curtail prolonged calls:

```java
// ProductClient.java

@HystrixCommand(fallbackMethod="getProductFallback",
               commandProperties={
                 @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="1000")  
               })
public Product getProduct(Long id) {
  return productService.getProduct(id);
}
```

We introduced retry logic through policies that were finely defined at the client level:

```java
// RetryConfig.java

@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate ordersRetryTemplate() {
    SimpleRetryPolicy policy = new SimpleRetryPolicy();
    policy.setMaxAttempts(3);
    return a RetryTemplate(policy);
  }

} 
```

These techniques guaranteed uniform responses and put a halt to cascading failures across the expanse of services.

## In Conclusion

The migration of our monolithic ERP system into the realm of microservices has emerged as an enlightening odyssey. It transcends the boundaries of a mere technical transition and signifies an organizational metamorphosis that equips our client to cater to the ever-evolving needs of their customer base with unmatched agility.

Through the disintegration of tightly interconnected layers and the establishment of well-defined domain boundaries, our development team has unlocked a new realm of agility. Features can now be sculpted and deployed independently, driven solely by business priorities rather than architectural constraints. This newfound capacity for swift experimentation and refinement positions the application to remain perpetually attuned to the dynamic demands of the market.

Simultaneously, our operations team now enjoys unfettered visibility and control over each constituent of the system. Anomalous behaviors are detected promptly through enhanced monitoring of individual services. The deployment of scaling and failover mechanisms has shifted from manual endeavors to automated orchestration, fostering an elevated level of resilience that will serve as the bedrock for our client's sustained expansion.

While the merits of a microservices architecture are beyond dispute, embarking on such a migration is not without its share of challenges. Our painstaking efforts in meticulous relationship analysis, interface definition, and the introduction of abstraction, rather than resorting to a brute 'rip and replace' approach, have bestowed upon us a flexible architectural framework capable of evolving harmoniously with the evolving needs of our clientele.

Above all, I remain profoundly grateful for the opportunity this project has afforded us to collaborate closely with our client on their digital transformation journey. The act of demystifying and sharing our experiences and insights in this article is my way of paying it forward, with the hope that it empowers more businesses to embrace modernization. The rewards, both for customers and businesses alike, are unquestionably worth the endeavor.

## References 

- MongoDB Documentation - Official documentation covering data modeling, queries, deployment, and more. [https://docs.mongodb.com/](https://docs.mongodb.com/)

- Microservices Pattern - Martin Fowler's seminal overview of microservices architecture principles. [https://martinfowler.com/articles/microservices.html](https://martinfowler.com/articles/microservices.html)

- Domain-Driven Design - Eric Evans' book introducing DDD concepts for structuring services around business domains. [https://domainlanguage.com/ddd/](https://domainlanguage.com/ddd/)

- Twelve-Factor App Methodology - Best practices for building software-as-a-service apps that are readily deployable to the cloud. [https://12factor.net/](https://12factor.net/)

- Container Journal - In-depth articles on containerization, orchestration, and cloud platforms. [https://containerjournal.com/](https://containerjournal.com/)

- Docker Documentation - Comprehensive guides for building, deploying, and managing containerized apps. [https://docs.docker.com/](https://docs.docker.com/)

- Kubernetes Documentation - Kubernetes, the leading container orchestrator, and its architecture. [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/)
