# The Infrastructure Engineer
*Master fault-tolerant messaging and observability*

**Duration**: 3-4 weeks | **Level**: Intermediate | **Prerequisites**: Basic Docker/Linux knowledge

---

## Your Journey Overview

Build the invisible backbone that keeps applications running reliably at scale. This path focuses on **fault tolerance**, **observability**, and **graceful degradation**—the essential skills for systems that never sleep.

### What You'll Build
- **Message Streaming Platforms**: Kafka clusters that handle millions of events
- **Observability Stacks**: Complete monitoring with metrics, logs, and traces
- **Fault-Tolerant Systems**: Infrastructure that recovers from failures automatically
- **Production Dashboards**: Real-time visibility into system health and performance

---

## Learning Philosophy: The Reliability Mindset

Infrastructure engineering is about **expecting failure and designing for resilience**. Every component will fail eventually—your job is to ensure the system as a whole keeps running. Think like a reliability engineer: measure everything, automate recovery, and eliminate single points of failure.

### Core Mental Models

**🌊 At-Least-Once Delivery**  
In distributed systems, it's better to process a message twice than to lose it forever. Design your systems to be **idempotent**—they should handle duplicate work gracefully.

**📊 Observability as Debugging at Scale**  
You can't SSH into every server when things go wrong. Instead, you need **telemetry**—metrics, logs, and traces that tell the story of what happened when you weren't looking.

**🔄 Circuit Breakers and Graceful Degradation**  
When a dependency fails, fail fast and provide a degraded experience rather than cascading failures. Better a slow response than no response.

---

## Phase 1: Fault-Tolerant Messaging (Week 1-2)
*Build systems that never lose messages*

### 🎯 **Learning Goals**
- [ ] Understand Kafka's architecture and guarantees
- [ ] Master consumer groups and partition rebalancing
- [ ] Implement idempotent message processing patterns
- [ ] Design dead letter queues and error handling strategies

### 📚 **Study Guide**
**Primary**: [Technical Deep Dive: Kafka Consumer Fault Tolerance and Rebalancing](../technical_guides/kafka/kafka-consumer-fault-tolerance-and-rebalancing.md)

### 🛠️ **Hands-On Project**: E-commerce Event Streaming
Build a complete event-driven system with order processing and inventory management:
```java
// Your goal: understand at-least-once delivery and idempotency
@Service
public class OrderProcessingConsumer {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private DeadLetterProducer deadLetterProducer;
    
    @KafkaListener(
        topics = "order-events",
        groupId = "order-processing-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void processOrderEvent(
        @Payload OrderEvent event,
        @Header KafkaHeaders headers,
        Acknowledgment ack
    ) {
        try {
            // Idempotent processing - check if already processed
            if (orderService.isAlreadyProcessed(event.getOrderId(), event.getEventId())) {
                log.info("Event already processed, skipping: {}", event.getEventId());
                ack.acknowledge();
                return;
            }
            
            // Process the order event
            switch (event.getType()) {
                case ORDER_CREATED:
                    processOrderCreated(event);
                    break;
                case ORDER_PAID:
                    processOrderPaid(event);
                    break;
                case ORDER_CANCELLED:
                    processOrderCancelled(event);
                    break;
                default:
                    throw new UnknownEventTypeException("Unknown event type: " + event.getType());
            }
            
            // Mark as successfully processed
            orderService.markEventProcessed(event.getOrderId(), event.getEventId());
            ack.acknowledge();
            
        } catch (RetryableException e) {
            // Let Kafka retry by not acknowledging
            log.warn("Retryable error processing event: {}", e.getMessage());
            throw e;
            
        } catch (PoisonPillException e) {
            // Send to dead letter queue and acknowledge to prevent blocking
            log.error("Poison pill detected, sending to DLQ: {}", e.getMessage());
            deadLetterProducer.send(event, e.getMessage());
            ack.acknowledge();
        }
    }
    
    private void processOrderCreated(OrderEvent event) {
        // Idempotent order creation logic
        orderService.createOrUpdateOrder(event.getOrderData());
        
        // Emit downstream events
        inventoryService.reserveItems(event.getOrderData().getItems());
        paymentService.initializePayment(event.getOrderData());
    }
}
```

### ✅ **Checkpoint Questions**
1. How does `FOR UPDATE SKIP LOCKED` prevent duplicate message processing?
2. What's the difference between partition rebalancing and consumer failure?
3. When should you use a dead letter queue vs. retrying indefinitely?

---

## Phase 2: Production-Grade Observability (Week 2-3)
*Build systems you can debug at 3 AM*

### 🎯 **Learning Goals**
- [ ] Design comprehensive monitoring with the PLG stack (Prometheus, Loki, Grafana)
- [ ] Implement distributed tracing for request lifecycle visibility
- [ ] Build alerting systems that focus on symptoms, not causes
- [ ] Create runbooks and incident response procedures

### 📚 **Study Guide**
**Primary**: [A Comprehensive Guide to Production-Grade Monitoring with Grafana, Prometheus, and Loki](../technical_guides/observability/a-comprehensive-guide-to-production-grade-monitoring-with-grafana-prometheus-and-loki.md)

### 🛠️ **Hands-On Project**: Complete Observability Stack
Build a monitoring system for your Kafka-based order processing:
```yaml
# Your goal: complete observability with correlated data
version: '3.8'

services:
  # Metrics Collection
  prometheus:
    image: prom/prometheus:v2.45.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'

  # Log Collection
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

  # Visualization & Alerting
  grafana:
    image: grafana/grafana:10.0.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-storage:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources

  # Distributed Tracing
  jaeger:
    image: jaegertracing/all-in-one:1.47
    ports:
      - "16686:16686"
      - "14268:14268"
    environment:
      - COLLECTOR_OTLP_ENABLED=true

volumes:
  grafana-storage:
```

```java
// Application instrumentation with proper correlation
@RestController
public class OrderController {
    
    private static final Counter ORDER_REQUESTS = Counter.build()
        .name("order_requests_total")
        .help("Total order requests")
        .labelNames("status", "method")
        .register();
    
    private static final Histogram ORDER_DURATION = Histogram.build()
        .name("order_processing_duration_seconds")
        .help("Order processing duration")
        .register();
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(
        @RequestBody CreateOrderRequest request,
        HttpServletRequest httpRequest
    ) {
        // Start distributed trace
        Span span = tracer.nextSpan()
            .name("create-order")
            .tag("order.user_id", request.getUserId())
            .tag("order.item_count", String.valueOf(request.getItems().size()))
            .start();
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            // Structured logging with correlation
            log.info("Processing order creation", 
                kv("user_id", request.getUserId()),
                kv("trace_id", span.context().traceId()),
                kv("span_id", span.context().spanId()),
                kv("item_count", request.getItems().size())
            );
            
            // Business logic with child spans
            Order order = orderService.createOrder(request);
            
            // Success metrics
            ORDER_REQUESTS.labels("success", "POST").inc();
            sample.stop(Timer.builder("order.creation.duration")
                .tag("status", "success")
                .register(meterRegistry));
            
            log.info("Order created successfully",
                kv("order_id", order.getId()),
                kv("trace_id", span.context().traceId())
            );
            
            return ResponseEntity.ok(order);
            
        } catch (ValidationException e) {
            ORDER_REQUESTS.labels("validation_error", "POST").inc();
            span.tag("error", true).tag("error.message", e.getMessage());
            
            log.warn("Order validation failed",
                kv("error", e.getMessage()),
                kv("trace_id", span.context().traceId())
            );
            
            throw e;
            
        } catch (Exception e) {
            ORDER_REQUESTS.labels("error", "POST").inc();
            span.tag("error", true).tag("error.message", e.getMessage());
            
            log.error("Order creation failed",
                kv("error", e.getMessage()),
                kv("trace_id", span.context().traceId()),
                e
            );
            
            throw e;
            
        } finally {
            span.end();
        }
    }
}
```

### ✅ **Checkpoint Questions**
1. How do you correlate metrics, logs, and traces using shared labels?
2. What's the difference between RED and USE methodologies for monitoring?
3. Why should alerts focus on symptoms rather than causes?

---

## Phase 3: Resilience Patterns & Automation (Week 3-4)
*Build systems that heal themselves*

### 🎯 **Learning Goals**
- [ ] Implement circuit breakers and bulkhead patterns
- [ ] Design auto-scaling and load balancing strategies
- [ ] Build health check and service discovery systems
- [ ] Create chaos engineering practices for testing resilience

### 🛠️ **Hands-On Project**: Self-Healing Infrastructure
Build a system that automatically responds to failures:
```java
// Your goal: resilience patterns in practice
@Component
public class ResilientOrderService {
    
    @Autowired
    private PaymentServiceClient paymentService;
    
    @Autowired
    private InventoryServiceClient inventoryService;
    
    // Circuit breaker for external service calls
    @CircuitBreaker(name = "payment-service", fallbackMethod = "fallbackPayment")
    @Retry(name = "payment-service")
    @TimeLimiter(name = "payment-service")
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // This call is protected by circuit breaker
            return paymentService.processPayment(request);
        });
    }
    
    // Graceful degradation when payment service is down
    public CompletableFuture<PaymentResult> fallbackPayment(PaymentRequest request, Exception ex) {
        log.warn("Payment service unavailable, using fallback", 
            kv("payment_id", request.getPaymentId()),
            kv("error", ex.getMessage())
        );
        
        // Store for later processing when service recovers
        paymentQueue.queueForLaterProcessing(request);
        
        return CompletableFuture.completedFuture(
            PaymentResult.deferred(request.getPaymentId(), "Payment queued for processing")
        );
    }
    
    // Bulkhead pattern - separate thread pools for different operations
    @Async("orderProcessingExecutor")
    public CompletableFuture<Void> processOrderAsync(Order order) {
        // Isolated execution for order processing
        return CompletableFuture.runAsync(() -> {
            processOrderInternal(order);
        });
    }
    
    @Async("notificationExecutor")
    public CompletableFuture<Void> sendNotificationAsync(Order order) {
        // Separate thread pool for notifications - failures don't impact orders
        return CompletableFuture.runAsync(() -> {
            notificationService.sendOrderConfirmation(order);
        });
    }
}
```

```yaml
# Kubernetes deployment with auto-scaling and health checks
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: order-service:latest
        ports:
        - containerPort: 8080
        
        # Health checks for automatic recovery
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
          
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        
        # Resource limits and requests
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
            
        # Environment configuration
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "kafka:9092"

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### ✅ **Checkpoint Questions**
1. How does a circuit breaker prevent cascading failures?
2. What's the difference between liveness and readiness probes?
3. When should you use horizontal vs. vertical scaling?

---

## Mastery Assessment

### Portfolio Projects
By completion, you'll have built:
1. **Event Streaming System** - Kafka-based order processing with fault tolerance
2. **Observability Stack** - Complete PLG monitoring with correlated data
3. **Self-Healing Infrastructure** - Auto-scaling deployment with resilience patterns
4. **Chaos Engineering Lab** - Controlled failure injection and recovery testing

### Skills Validation
- [ ] **Message Reliability**: Design systems that never lose important data
- [ ] **Observability**: Build comprehensive monitoring and alerting
- [ ] **Fault Tolerance**: Create systems that gracefully handle component failures
- [ ] **Automation**: Implement self-healing and auto-scaling infrastructure
- [ ] **Incident Response**: Debug and resolve production issues quickly

---

## Real-World Scenarios

### When Systems Fail
Learn to handle production incidents:
- **Kafka Cluster Down**: Implement producer buffering and consumer lag recovery
- **Database Overload**: Circuit breakers and connection pool management
- **Memory Leaks**: Automatic pod recycling and heap dump analysis  
- **Network Partitions**: Split-brain prevention and consensus algorithms
- **Cascading Failures**: Bulkhead patterns and graceful degradation

### Performance at Scale
Master techniques for high-throughput systems:
- **Load Balancing**: Distribute load across multiple instances effectively
- **Caching Strategies**: Multi-level caches with proper invalidation
- **Database Sharding**: Horizontal partitioning for massive datasets
- **CDN Integration**: Global content delivery and edge computing
- **Async Processing**: Non-blocking I/O and event-driven architectures

---

## What's Next?

### Specialization Paths
- **🦀 High-Performance Systems**: Combine with [Rust Systems Engineer](./rust-systems-engineer.md) for optimized infrastructure
- **🗄️ Data Infrastructure**: Add [Database Architect](./database-architect.md) for data-intensive systems
- **🎯 Distributed Systems**: Extend with [System Design Expert](./system-design-expert.md) for architecture leadership

### Advanced Topics
- **Service Mesh**: Istio/Linkerd for microservices communication
- **GitOps**: Infrastructure as code with ArgoCD and Flux
- **Security**: Zero-trust networking and vulnerability management
- **Cost Optimization**: FinOps practices and resource efficiency
- **Edge Computing**: CDN and distributed processing at the edge

### Platform Engineering
- **Developer Experience**: Internal platforms and self-service infrastructure
- **CI/CD Pipelines**: Automated testing and deployment systems
- **Multi-Cloud**: Vendor-agnostic infrastructure and disaster recovery
- **Compliance**: SOC2, PCI-DSS, and other regulatory requirements
- **Team Leadership**: Technical mentoring and architecture decisions

---

*Ready to build infrastructure that never sleeps? Start with Phase 1 and create systems that scale reliably.*