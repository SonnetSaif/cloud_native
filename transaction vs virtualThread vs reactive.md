@Transactional
public void submitApplication(ApplicationRequestDto applicationRequestDto) {
    try {
        ExecutorService executorService = Executors.newVirtualThreadPerTaskExecutor();

        try (executorService) {
            for (int i = 0; i < 10; i++) {
                executorService.submit(() -> {
                    HealthDetailsEntity savedHealthDetails = getHealthDetailsEntity(applicationRequestDto, healthDetailsRepository, personHealthRepository);

                    PolicyInsuredInfoEntity savedPolicyInsured = getPolicyInsuredInfoEntity(applicationRequestDto, policyInsuredInfoRepository, addressRepository, savedHealthDetails);
                    PolicyOwnerInfoEntity savedPolicyOwner = getPolicyOwnerInfoEntity(applicationRequestDto, policyOwnerInfoRepository, addressRepository, savedHealthDetails);

                    List<RiderEntity> savedRiders = getRiderEntity(applicationRequestDto, riderRepository);
                    List<MedicareHospitalCareEntity> savedMedicare = getMedicareEntity(applicationRequestDto, medicareHospitalCareRepository);
                    ProductDetailsEntity savedProductDetails = getProductDetailsEntity(applicationRequestDto, productDetailsRepository, savedRiders, savedMedicare);

                    List<ExistingInsuranceEntity> savedExistingInsurances = getExistingInsuranceEntity(applicationRequestDto, existingInsuranceRepository);
                    List<BeneficiaryInformationEntity> savedBeneficiaryInformation = getBeneficiaryInformationEntity(applicationRequestDto, beneficiaryInformationRepository);

                    DeclarationAndUndertakingEntity savedDeclarationAndUndertaking = getDeclarationAndUndertakingDto(applicationRequestDto, declarationAndUndertakingRepository);

                    getDocumentDetailsEntity(applicationRequestDto, documentDetailsRepository, documentRepository);

                    KycProfileEntity savedKycProfile = getKycProfileEntity(applicationRequestDto, kycProfileRepository);

                    getApplicationRequestEntity(applicationRequestDto, savedHealthDetails, savedPolicyInsured, savedPolicyOwner,
                            savedProductDetails, savedExistingInsurances, savedBeneficiaryInformation, savedDeclarationAndUndertaking, savedKycProfile, applicationRequestRepository);
                });
            }
        }

    } catch (Exception e) {
        throw new RuntimeException(e);
    }

}

is it ok using virtual thread?


Short answer: **Not as written.** Using a per‑task virtual‑thread `ExecutorService` *inside* a `@Transactional` method like this will not do what you expect, and it can introduce correctness and stability issues.

Below is what’s going on, the risks, and safe patterns to use with Spring Data JPA + transactions + virtual threads.

***

## What’s wrong with the current approach?

### 1) The transaction context does **not** flow to the submitted tasks

*   Your method is annotated `@Transactional`, but the lambdas you submit to the executor run on **different threads**. Spring’s transaction context is **thread‑bound**.
*   That means:
    *   The work in `executorService.submit(...)` won’t run in the *same* transaction.
    *   In fact, it likely runs **without any transaction at all**, which can cause `TransactionRequiredException` on flush/commit, or inconsistent auto‑commit writes depending on your JPA provider/driver.
*   Even if some calls “seem” to succeed, you’ll get **multiple independent transactions** or **no transaction** per task, not one atomic transaction.

### 2) The JPA `EntityManager` is **not** thread-safe

*   The `EntityManager`/persistence context Spring binds to the calling thread is **not shared across threads**.
*   Inside the tasks you are creating/persisting entities on another thread where no persistence context is bound—this can lead to exceptions or detached entities, flushing issues, or subtle data anomalies.

### 3) You’re doing the whole pipeline **10 times**

```java
for (int i = 0; i < 10; i++) {
    executorService.submit(() -> {
        // full graph persisted again...
    });
}
```

*   This will persist the entire application graph **ten times**. If this was for load testing, it’s okay (in a test tool), but in production this is almost certainly a bug.
*   If your intention was to parallelize *parts* of the pipeline once, split and submit only the independent parts **once**.

### 4) Transactional semantics become unclear

*   The outer `@Transactional` will commit when the method returns, but the tasks execute **out of band** and can still be running or have already failed independently.
*   The try‑with‑resources over the `ExecutorService` will `close()` it; the JDK implementation **waits for tasks to finish** before closing, but that still has **no relation** to your Spring transaction.

### 5) Virtual threads ≠ free scalability

*   Virtual threads are great for blocking I/O, but each concurrent task still needs a DB connection when doing JDBC. Your HikariCP pool size can become the **bottleneck** fast.
*   It’s easy to accidentally create **many more concurrent DB operations** than your pool can handle, causing queueing, timeouts, or deadlocks.

***

## When (and how) to use virtual threads safely in Spring

### A) If you need the **whole write** to be atomic (one transaction)

**Don’t fork**. Do the persistence in the **same thread** covered by `@Transactional`:

```java
@Transactional
public void submitApplication(ApplicationRequestDto dto) {
    HealthDetailsEntity savedHealthDetails = getHealthDetailsEntity(dto, healthDetailsRepository, personHealthRepository);
    PolicyInsuredInfoEntity savedPolicyInsured = getPolicyInsuredInfoEntity(dto, policyInsuredInfoRepository, addressRepository, savedHealthDetails);
    PolicyOwnerInfoEntity savedPolicyOwner = getPolicyOwnerInfoEntity(dto, policyOwnerInfoRepository, addressRepository, savedHealthDetails);
    List<RiderEntity> savedRiders = getRiderEntity(dto, riderRepository);
    List<MedicareHospitalCareEntity> savedMedicare = getMedicareEntity(dto, medicareHospitalCareRepository);
    ProductDetailsEntity savedProductDetails = getProductDetailsEntity(dto, productDetailsRepository, savedRiders, savedMedicare);
    List<ExistingInsuranceEntity> savedExistingInsurances = getExistingInsuranceEntity(dto, existingInsuranceRepository);
    List<BeneficiaryInformationEntity> savedBeneficiaryInformation = getBeneficiaryInformationEntity(dto, beneficiaryInformationRepository);
    DeclarationAndUndertakingEntity savedDeclarationAndUndertaking = getDeclarationAndUndertakingDto(dto, declarationAndUndertakingRepository);
    getDocumentDetailsEntity(dto, documentDetailsRepository, documentRepository);
    KycProfileEntity savedKycProfile = getKycProfileEntity(dto, kycProfileRepository);
    getApplicationRequestEntity(dto, savedHealthDetails, savedPolicyInsured, savedPolicyOwner, savedProductDetails,
                                savedExistingInsurances, savedBeneficiaryInformation, savedDeclarationAndUndertaking,
                                savedKycProfile, applicationRequestRepository);
}
```

> This keeps a single persistence context and one commit point—**correctness first**.

### B) If you truly have **independent sub-operations** that can commit separately

Make each sub‑operation its **own transactional method** and run them in parallel **with their own transactions**.

*   Put the sub-operation into another bean or call through the proxied self (so Spring AOP applies `@Transactional` to each).
*   Use a **virtual‑thread executor bean** for simple concurrency (Spring Boot 3.2+/Java 21):

```java
// @Configuration
@Bean
Executor taskExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();
}
```

*   Annotate each subtask method with `@Transactional(propagation = Propagation.REQUIRES_NEW)` if you want **independent commits**:

```java
@Service
public class ApplicationWriter {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeHealth(ApplicationRequestDto dto) { /* persist health */ }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeInsuredAndOwner(ApplicationRequestDto dto) { /* persist insured/owner */ }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeProducts(ApplicationRequestDto dto) { /* riders/medicare/product */ }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeOther(ApplicationRequestDto dto) { /* existing, beneficiary, docs, kyc, app request */ }
}
```

*   Or use `TransactionTemplate` inside tasks:

```java
CompletableFuture<Void> f = CompletableFuture.runAsync(() ->
    transactionTemplate.executeWithoutResult(status -> writer.writeHealth(dto)), executor);
```

*   Wait for all subtasks and handle failures:

```java
CompletableFuture.allOf(f1, f2, f3, f4).join();
```

> **Note:** Once you split into multiple transactions, you must decide reconcile/compensate on partial failures.

### C) If you need **read** parallelism (e.g., upstream service calls, file I/O)

Virtual threads are excellent for this. Do the I/O concurrently, then **persist results in a single transaction** once all futures complete.

```java
var vtExecutor = Executors.newVirtualThreadPerTaskExecutor();
try (vtExecutor) {
    var ridersFuture = CompletableFuture.supplyAsync(() -> fetchRiders(dto), vtExecutor);
    var kycFuture    = CompletableFuture.supplyAsync(() -> fetchKyc(dto), vtExecutor);
    var docsFuture   = CompletableFuture.supplyAsync(() -> fetchDocs(dto), vtExecutor);

    var riders = ridersFuture.join();
    var kyc    = kycFuture.join();
    var docs   = docsFuture.join();

    // Now persist atomically
    persistAllInOneTx(dto, riders, kyc, docs); // a @Transactional method
}
```

***

## Additional guidance & guardrails

*   **Do not share entities** across threads. Create them per task and let JPA manage them per transaction.
*   **Executor lifetime:** Don’t create/shutdown executors per request unless necessary. Prefer a Spring‑managed bean. Your try‑with‑resources ensures tasks finish before close, which is good, but a shared executor is better.
*   **Connection pool sizing:** Virtual threads can enqueue many blocking JDBC calls. Tune HikariCP `maximumPoolSize` appropriately and keep concurrency reasonable.
*   **Idempotency:** If you keep the outer loop, you will create **10 copies** of the same application data. Remove the loop unless you want that on purpose.
*   **Error handling:** Decide whether a single subtask failing should fail the whole operation. With multiple transactions, you may need a cleanup/compensation step.

***

## A safer refactor using virtual threads for **I/O**, single TX for **DB**

```java
public void submitApplication(ApplicationRequestDto dto) {
    try (ExecutorService vt = Executors.newVirtualThreadPerTaskExecutor()) {
        var ridersF   = CompletableFuture.supplyAsync(() -> getRiderEntity(dto, riderRepository), vt);
        var medicareF = CompletableFuture.supplyAsync(() -> getMedicareEntity(dto, medicareHospitalCareRepository), vt);
        var docsF     = CompletableFuture.runAsync(() -> getDocumentDetailsEntity(dto, documentDetailsRepository, documentRepository), vt);
        var kycF      = CompletableFuture.supplyAsync(() -> getKycProfileEntity(dto, kycProfileRepository), vt);

        // Wait for I/O-ish parts (if those methods are DB-heavy, consider doing them in the single TX below)
        var riders   = ridersF.join();
        var medicare = medicareF.join();
        kycF.join(); // or collect entity if needed
        docsF.join();

        // Now persist the cohesive graph in ONE transaction
        persistAll(dto, riders, medicare);
    }
}

@Transactional
void persistAll(ApplicationRequestDto dto, List<RiderEntity> riders, List<MedicareHospitalCareEntity> medicare) {
    var health = getHealthDetailsEntity(dto, healthDetailsRepository, personHealthRepository);
    var insured = getPolicyInsuredInfoEntity(dto, policyInsuredInfoRepository, addressRepository, health);
    var owner = getPolicyOwnerInfoEntity(dto, policyOwnerInfoRepository, addressRepository, health);
    var product = getProductDetailsEntity(dto, productDetailsRepository, riders, medicare);
    var existing = getExistingInsuranceEntity(dto, existingInsuranceRepository);
    var bene = getBeneficiaryInformationEntity(dto, beneficiaryInformationRepository);
    var decl = getDeclarationAndUndertakingDto(dto, declarationAndUndertakingRepository);
    var kyc = getKycProfileEntity(dto, kycProfileRepository);
    getApplicationRequestEntity(dto, health, insured, owner, product, existing, bene, decl, kyc, applicationRequestRepository);
}
```

***

## Bottom line

*   Virtual threads are fine to use in Spring Boot 3.2+/Java 21, but **not** forking DB writes out of a transactional method the way you have it.
*   If you need **one atomic write**, keep everything in one transaction on a single thread.
*   If you need **parallelism**, split into **clearly independent** sub‑transactions with their own `@Transactional` boundaries, or parallelize only the **read/I/O** phase and commit once.

If you tell me what you wanted to achieve by submitting the same pipeline 10 times (e.g., load testing, or truly independent child writes), I can give you a tailored refactor with proper transaction boundaries and a virtual‑thread executor bean configuration.



How to split into sub-transactions?


Great question! When you want to **parallelize or isolate parts** of a larger write operation so that each part **commits (or rolls back) independently**, you can split the workflow into **sub‑transactions**.

Below are practical, production‑safe patterns for Spring (Boot) with JPA—covering `@Transactional` propagation, virtual threads, error handling, and orchestration.

***

## 🧭 When to split into sub-transactions

*   Parts of the workflow are **independent** (e.g., saving KYC vs. saving documents).
*   A failure in one part should **not roll back** the others.
*   You want **parallelism** (e.g., use virtual threads or a task executor).
*   You need **retry/compensation** per part.

> If you need **all-or-nothing** semantics across the whole pipeline, **do not** split—use a single transaction.

***

## 🔑 Key building blocks

1.  **`@Transactional(propagation = REQUIRES_NEW)`**  
    Starts a **new transaction** for the annotated method, suspends any existing one.

2.  **Orchestrator (no transaction or `SUPPORTS`)**  
    Submits/awaits sub-tasks and collects results.

3.  **Executor (can be virtual-thread)**  
    To run independent sub-tasks concurrently.

4.  **Avoid self-invocation**  
    Call sub-transactional methods **via a Spring proxy** (another bean or `self` injected as proxied bean), not `this.method()`.

5.  **Error strategy**  
    Decide: fail-fast or continue; log/collect errors; retry; compensate.

***

## 🛠️ Pattern A — Split into `REQUIRES_NEW` sub-transactions

### 1) Sub-transaction service (each method commits independently)

```java
@Service
public class ApplicationWriter {

    private final HealthDetailsRepository healthDetailsRepository;
    private final PersonHealthRepository personHealthRepository;
    private final PolicyInsuredInfoRepository policyInsuredInfoRepository;
    private final PolicyOwnerInfoRepository policyOwnerInfoRepository;
    private final AddressRepository addressRepository;
    private final ProductDetailsRepository productDetailsRepository;
    private final RiderRepository riderRepository;
    private final MedicareHospitalCareRepository medicareHospitalCareRepository;
    private final ExistingInsuranceRepository existingInsuranceRepository;
    private final BeneficiaryInformationRepository beneficiaryInformationRepository;
    private final DeclarationAndUndertakingRepository declarationAndUndertakingRepository;
    private final DocumentDetailsRepository documentDetailsRepository;
    private final DocumentRepository documentRepository;
    private final KycProfileRepository kycProfileRepository;
    private final ApplicationRequestRepository applicationRequestRepository;

    // constructor omitted

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public HealthDetailsEntity writeHealth(ApplicationRequestDto dto) {
        return getHealthDetailsEntity(dto, healthDetailsRepository, personHealthRepository);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public PolicyInsuredInfoEntity writePolicyInsured(ApplicationRequestDto dto, HealthDetailsEntity health) {
        return getPolicyInsuredInfoEntity(dto, policyInsuredInfoRepository, addressRepository, health);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public PolicyOwnerInfoEntity writePolicyOwner(ApplicationRequestDto dto, HealthDetailsEntity health) {
        return getPolicyOwnerInfoEntity(dto, policyOwnerInfoRepository, addressRepository, health);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public ProductDetailsEntity writeProduct(ApplicationRequestDto dto,
                                             List<RiderEntity> riders,
                                             List<MedicareHospitalCareEntity> medicare) {
        return getProductDetailsEntity(dto, productDetailsRepository, riders, medicare);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public List<RiderEntity> writeRiders(ApplicationRequestDto dto) {
        return getRiderEntity(dto, riderRepository);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public List<MedicareHospitalCareEntity> writeMedicare(ApplicationRequestDto dto) {
        return getMedicareEntity(dto, medicareHospitalCareRepository);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public List<ExistingInsuranceEntity> writeExisting(ApplicationRequestDto dto) {
        return getExistingInsuranceEntity(dto, existingInsuranceRepository);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public List<BeneficiaryInformationEntity> writeBeneficiaries(ApplicationRequestDto dto) {
        return getBeneficiaryInformationEntity(dto, beneficiaryInformationRepository);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public DeclarationAndUndertakingEntity writeDeclaration(ApplicationRequestDto dto) {
        return getDeclarationAndUndertakingDto(dto, declarationAndUndertakingRepository);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeDocuments(ApplicationRequestDto dto) {
        getDocumentDetailsEntity(dto, documentDetailsRepository, documentRepository);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public KycProfileEntity writeKyc(ApplicationRequestDto dto) {
        return getKycProfileEntity(dto, kycProfileRepository);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void linkApplication(ApplicationRequestDto dto,
                                HealthDetailsEntity health,
                                PolicyInsuredInfoEntity insured,
                                PolicyOwnerInfoEntity owner,
                                ProductDetailsEntity product,
                                List<ExistingInsuranceEntity> existing,
                                List<BeneficiaryInformationEntity> beneficiaries,
                                DeclarationAndUndertakingEntity decl,
                                KycProfileEntity kyc) {

        getApplicationRequestEntity(dto, health, insured, owner, product,
                existing, beneficiaries, decl, kyc, applicationRequestRepository);
    }
}
```

### 2) Orchestrator service (no transaction, submits tasks)

```java
@Service
public class ApplicationOrchestrator {

    private final ApplicationWriter writer;
    private final Executor executor; // define a VT executor bean

    public ApplicationOrchestrator(ApplicationWriter writer, Executor executor) {
        this.writer = writer;
        this.executor = executor;
    }

    public void submitApplication(ApplicationRequestDto dto) {
        // Parallel sub-transactions where independent
        CompletableFuture<HealthDetailsEntity> healthF = CompletableFuture.supplyAsync(() -> writer.writeHealth(dto), executor);

        // dependent on health
        CompletableFuture<PolicyInsuredInfoEntity> insuredF = healthF.thenApplyAsync(health -> writer.writePolicyInsured(dto, health), executor);
        CompletableFuture<PolicyOwnerInfoEntity>   ownerF   = healthF.thenApplyAsync(health -> writer.writePolicyOwner(dto, health),   executor);

        // independent
        CompletableFuture<List<RiderEntity>> ridersF     = CompletableFuture.supplyAsync(() -> writer.writeRiders(dto), executor);
        CompletableFuture<List<MedicareHospitalCareEntity>> medicareF = CompletableFuture.supplyAsync(() -> writer.writeMedicare(dto), executor);
        CompletableFuture<List<ExistingInsuranceEntity>> existingF   = CompletableFuture.supplyAsync(() -> writer.writeExisting(dto), executor);
        CompletableFuture<List<BeneficiaryInformationEntity>> beneF  = CompletableFuture.supplyAsync(() -> writer.writeBeneficiaries(dto), executor);
        CompletableFuture<DeclarationAndUndertakingEntity> declF     = CompletableFuture.supplyAsync(() -> writer.writeDeclaration(dto), executor);
        CompletableFuture<KycProfileEntity> kycF                      = CompletableFuture.supplyAsync(() -> writer.writeKyc(dto), executor);
        CompletableFuture<Void> docsF                                 = CompletableFuture.runAsync(() -> writer.writeDocuments(dto), executor);

        // Product depends on riders + medicare
        CompletableFuture<ProductDetailsEntity> productF = ridersF.thenCombineAsync(medicareF,
                (riders, medicare) -> writer.writeProduct(dto, riders, medicare), executor);

        // Wait for required parts to link the application
        CompletableFuture.allOf(healthF, insuredF, ownerF, productF, existingF, beneF, declF, kycF, docsF).join();

        // Get results (will throw if any failed)
        HealthDetailsEntity health     = healthF.join();
        PolicyInsuredInfoEntity insured= insuredF.join();
        PolicyOwnerInfoEntity owner    = ownerF.join();
        ProductDetailsEntity product   = productF.join();
        List<ExistingInsuranceEntity> existing = existingF.join();
        List<BeneficiaryInformationEntity> bene = beneF.join();
        DeclarationAndUndertakingEntity decl = declF.join();
        KycProfileEntity kyc = kycF.join();

        // Final link in its own sub-transaction
        writer.linkApplication(dto, health, insured, owner, product, existing, bene, decl, kyc);
    }
}
```

### 3) Virtual-thread executor bean

```java
@Configuration
public class ExecutorConfig {
    @Bean(destroyMethod = "close")
    public Executor applicationExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}
```

> This pattern gives each sub-step **its own transaction**. Failures in one do not roll back others. You can wrap each `writer.*` with retry logic if needed.

***

## 🛠️ Pattern B — Use `TransactionTemplate` inside tasks

If you prefer **programmatic** transactions:

```java
@Service
public class TxTaskRunner {
    private final TransactionTemplate txTemplate;

    public TxTaskRunner(PlatformTransactionManager txManager) {
        this.txTemplate = new TransactionTemplate(txManager);
        this.txTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    }

    public <T> T inNewTx(Supplier<T> supplier) {
        return txTemplate.execute(status -> supplier.get());
    }

    public void inNewTx(Runnable runnable) {
        txTemplate.executeWithoutResult(status -> runnable.run());
    }
}
```

Use in orchestrator:

```java
CompletableFuture<HealthDetailsEntity> healthF =
    CompletableFuture.supplyAsync(() -> txRunner.inNewTx(() -> writer.writeHealth(dto)), executor);
```

This gives you **explicit** transaction boundaries without AOP concerns.

***

## ⚠️ Important pitfalls & best practices

1.  **AOP proxy/self-invocation**  
    `@Transactional` is applied via proxies. If you call a `@Transactional` method **from the same class** (e.g., `this.method()`), the annotation **won’t apply**.  
    ✅ Put sub-transactional methods in a **separate bean** or inject `self` via interface/proxy.

2.  **DB connection pool**  
    Parallel sub-transactions need **separate JDBC connections**. Tune your Hikari pool (`maximumPoolSize`) to match concurrency, or throttle tasks.

3.  **Entity graph/data dependencies**  
    Where write order matters (e.g., `health` → `insured/owner`), enforce dependency via `thenApply`/`thenCombine` or sequence them (don’t parallelize everything blindly).

4.  **Error handling**  
    Decide:
    *   **Fail-fast** (stop on first failure) vs. **best-effort** (continue others, report partial success).
    *   **Compensation** for already-committed sub-transactions if a later step fails (e.g., delete records, mark application as “incomplete”).

5.  **Idempotency**  
    If your endpoint can be retried, make sub-steps idempotent (e.g., upsert by `applicationNo`).

6.  **Transaction semantics**
    *   `REQUIRES_NEW`: starts a brand-new tx, suspends any existing one.
    *   `NESTED`: rarely useful with JPA (needs savepoints; not always supported).
    *   Default (`REQUIRED`): joins current tx—*don’t use for sub-transactions*.

7.  **Isolation/timeout**  
    You can customize per sub-transaction:
    ```java
    @Transactional(propagation = REQUIRES_NEW, timeout = 30, isolation = Isolation.READ_COMMITTED)
    ```

***

## 🧪 Minimal example: two independent sub-transactions in parallel

```java
@Service
public class SubTxService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Long saveA(String a) { /* persist A */ return 1L; }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Long saveB(String b) { /* persist B */ return 2L; }
}

@Service
public class SplitTxOrchestrator {
    private final SubTxService svc;
    private final Executor executor;

    public SplitTxOrchestrator(SubTxService svc, Executor executor) {
        this.svc = svc;
        this.executor = executor;
    }

    public void run(String a, String b) {
        CompletableFuture<Long> aF = CompletableFuture.supplyAsync(() -> svc.saveA(a), executor);
        CompletableFuture<Long> bF = CompletableFuture.supplyAsync(() -> svc.saveB(b), executor);

        CompletableFuture.allOf(aF, bF).join();
        Long aId = aF.join();
        Long bId = bF.join();

        // Now optionally link them in its own sub-transaction
        // svc.link(aId, bId) with REQUIRES_NEW
    }
}
```

***

## ✅ When NOT to split

*   You need **strict atomicity** across the complete graph.
*   Steps **mutate the same rows** and can deadlock/update each other.
*   You can’t tolerate **partial success** without a compensation plan.

***

## Summary

*   Use **`@Transactional(REQUIRES_NEW)`** on **sub-step methods** in another bean (or via programmatic `TransactionTemplate`).
*   Orchestrate in a non-transactional service, optionally with a **virtual-thread executor** for parallelism.
*   Be explicit about **dependencies**, **error handling**, **pool sizing**, and **compensation**.

If you tell me which parts of your current `submitApplication` you want to isolate (e.g., KYC, Documents, Beneficiaries), I’ll sketch the exact split and the `CompletableFuture` graph tailored to your code.


can I introduce mono/flux in the submitApplication method?


Great question! You *can* introduce **Reactor Mono/Flux** in `submitApplication`, but whether you **should** depends on your stack and goals. Here’s a complete, practical guide tailored to your current setup (Spring MVC + JPA/Hibernate + blocking repositories).

***

## TL;DR

*   If you **keep JPA (blocking)**: you can wrap calls in `Mono/Flux` and run them on a bounded pool (or use virtual threads), but you **won’t get true non-blocking I/O**. Use this mainly for orchestration/composition (e.g., run independent steps concurrently with `Mono.zip`).
*   If you **move to R2DBC (non-blocking DB)**: you can go fully reactive (true backpressure, non-blocking) and use **`TransactionalOperator`** for reactive transactions.
*   **Do not mix** reactive chains with a standard `@Transactional` boundary that assumes thread-bound transactions—it won’t work as expected. Use reactive **`TransactionalOperator`** or keep everything imperative inside the transaction.

***

## Option 1: Keep JPA (Blocking) and Wrap with Reactor

This is the lowest-friction approach. You orchestrate steps with Reactor, but use blocking repositories on an appropriate Scheduler.

### When this makes sense

*   You want **composition/concurrency** and backpressure at the edges (e.g., your controller returns `Mono<Response>`).
*   You’re not ready to migrate to R2DBC.
*   You understand it’s still **blocking on DB** (just not blocking the Netty/event loop).

### Recommended patterns

1.  **Move orchestration to Reactor** and offload blocking to `Schedulers.boundedElastic()`:

```java
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

public Mono<String> submitApplicationReactive(ApplicationRequestDto dto) {
    // Each blocking call is wrapped and scheduled on a blocking-friendly pool
    Mono<HealthDetailsEntity> healthMono = Mono.fromCallable(() ->
            getHealthDetailsEntity(dto, healthDetailsRepository, personHealthRepository))
        .subscribeOn(Schedulers.boundedElastic());

    Mono<PolicyInsuredInfoEntity> insuredMono = healthMono.flatMap(health ->
        Mono.fromCallable(() ->
            getPolicyInsuredInfoEntity(dto, policyInsuredInfoRepository, addressRepository, health))
           .subscribeOn(Schedulers.boundedElastic())
    );

    Mono<PolicyOwnerInfoEntity> ownerMono = healthMono.flatMap(health ->
        Mono.fromCallable(() ->
            getPolicyOwnerInfoEntity(dto, policyOwnerInfoRepository, addressRepository, health))
           .subscribeOn(Schedulers.boundedElastic())
    );

    Mono<List<RiderEntity>> ridersMono = Mono.fromCallable(() ->
            getRiderEntity(dto, riderRepository))
        .subscribeOn(Schedulers.boundedElastic());

    Mono<List<MedicareHospitalCareEntity>> medicareMono = Mono.fromCallable(() ->
            getMedicareEntity(dto, medicareHospitalCareRepository))
        .subscribeOn(Schedulers.boundedElastic());

    Mono<ProductDetailsEntity> productMono = Mono.zip(ridersMono, medicareMono)
        .flatMap(tuple -> Mono.fromCallable(() ->
                getProductDetailsEntity(dto, productDetailsRepository, tuple.getT1(), tuple.getT2()))
            .subscribeOn(Schedulers.boundedElastic())
        );

    Mono<List<ExistingInsuranceEntity>> existingMono = Mono.fromCallable(() ->
            getExistingInsuranceEntity(dto, existingInsuranceRepository))
        .subscribeOn(Schedulers.boundedElastic());

    Mono<List<BeneficiaryInformationEntity>> beneMono = Mono.fromCallable(() ->
            getBeneficiaryInformationEntity(dto, beneficiaryInformationRepository))
        .subscribeOn(Schedulers.boundedElastic());

    Mono<DeclarationAndUndertakingEntity> declMono = Mono.fromCallable(() ->
            getDeclarationAndUndertakingDto(dto, declarationAndUndertakingRepository))
        .subscribeOn(Schedulers.boundedElastic());

    Mono<KycProfileEntity> kycMono = Mono.fromCallable(() ->
            getKycProfileEntity(dto, kycProfileRepository))
        .subscribeOn(Schedulers.boundedElastic());

    Mono<Void> docsMono = Mono.fromRunnable(() ->
            getDocumentDetailsEntity(dto, documentDetailsRepository, documentRepository))
        .subscribeOn(Schedulers.boundedElastic());

    // Compose and finally persist the linking entity
    return Mono.when(docsMono)
        .then(Mono.zip(healthMono, insuredMono, ownerMono, productMono, existingMono, beneMono, declMono, kycMono))
        .flatMap(tuple -> Mono.fromCallable(() -> {
            getApplicationRequestEntity(
                dto,
                tuple.getT1(), // health
                tuple.getT2(), // insured
                tuple.getT3(), // owner
                tuple.getT4(), // product
                tuple.getT5(), // existing
                tuple.getT6(), // bene
                tuple.getT7(), // decl
                tuple.getT8(), // kyc
                applicationRequestRepository
            );
            return "Success";
        }).subscribeOn(Schedulers.boundedElastic()))
        .onErrorMap(e -> new RuntimeException("Submit failed", e));
}
```

> **Transaction caution:** This composes several **independent transactions** (each repo call participates in its own transaction if configured as `@Transactional`). If you want **one atomic transaction**, don’t split like this—do it in one imperative block with a single `@Transactional` method. Reactor does not magically keep the same transaction across multiple scheduler switches.

2.  If you want **sub-transactions** (independent commits), keep the sub-methods with `@Transactional(propagation = REQUIRES_NEW)` and call them via `Mono.fromCallable(...).subscribeOn(...)`. This is similar to what we discussed earlier with `CompletableFuture`, just reactive-flavored.

***

## Option 2: Go Reactive End-to-End (R2DBC)

If you switch to **R2DBC repositories** (or `DatabaseClient`), you can go **fully non-blocking** and manage **reactive transactions**.

### Key changes

*   Replace Spring Data JPA with **Spring Data R2DBC** repositories or use `R2dbcEntityTemplate`/`DatabaseClient`.
*   Use **`TransactionalOperator`** for reactive transactional boundaries (instead of `@Transactional`).
*   All repository calls must be **non-blocking**.

### Example with `TransactionalOperator`

```java
import org.springframework.r2dbc.connection.TransactionDefinition;
import org.springframework.r2dbc.connection.TransactionSynchronizationManager;
import org.springframework.transaction.ReactiveTransactionManager;
import org.springframework.transaction.reactive.TransactionalOperator;
import reactor.core.publisher.Mono;

@Service
public class ReactiveApplicationService {

    private final ReactiveHealthRepo healthRepo;           // R2DBC repos
    private final ReactiveInsuredRepo insuredRepo;
    private final ReactiveOwnerRepo ownerRepo;
    private final ReactiveProductRepo productRepo;
    private final ReactiveExistingRepo existingRepo;
    private final ReactiveBeneficiaryRepo beneRepo;
    private final ReactiveDeclarationRepo declRepo;
    private final ReactiveKycRepo kycRepo;
    private final ReactiveAppReqRepo appReqRepo;
    private final TransactionalOperator tx;

    public ReactiveApplicationService(ReactiveTransactionManager rtm /*, repos... */) {
        this.tx = TransactionalOperator.create(rtm);
        // assign repos...
    }

    public Mono<String> submitApplication(ApplicationRequestDto dto) {
        Mono<HealthDetailsEntity> healthMono = healthRepo.saveFromDto(dto); // non-blocking

        Mono<PolicyInsuredInfoEntity> insuredMono = healthMono.flatMap(h ->
            insuredRepo.saveFromDto(dto, h));

        Mono<PolicyOwnerInfoEntity> ownerMono = healthMono.flatMap(h ->
            ownerRepo.saveFromDto(dto, h));

        Mono<List<RiderEntity>> ridersMono = productRepo.saveRidersFromDto(dto);
        Mono<List<MedicareHospitalCareEntity>> medicareMono = productRepo.saveMedicareFromDto(dto);

        Mono<ProductDetailsEntity> productMono = Mono.zip(ridersMono, medicareMono)
            .flatMap(t -> productRepo.saveProductFromDto(dto, t.getT1(), t.getT2()));

        Mono<List<ExistingInsuranceEntity>> existingMono = existingRepo.saveFromDto(dto);
        Mono<List<BeneficiaryInformationEntity>> beneMono = beneRepo.saveFromDto(dto);
        Mono<DeclarationAndUndertakingEntity> declMono = declRepo.saveFromDto(dto);
        Mono<KycProfileEntity> kycMono = kycRepo.saveFromDto(dto);

        Mono<Void> docsMono = /* write documents reactively, e.g., to S3/reactive FS */ Mono.empty();

        Mono<String> flow = Mono.when(docsMono)
            .then(Mono.zip(healthMono, insuredMono, ownerMono, productMono, existingMono, beneMono, declMono, kycMono))
            .flatMap(t -> appReqRepo.linkAll(
                dto, t.getT1(), t.getT2(), t.getT3(), t.getT4(), t.getT5(), t.getT6(), t.getT7(), t.getT8()))
            .thenReturn("Success");

        // Single reactive transaction across the chain
        return tx.transactional(flow);
    }
}
```

> This gives you **true non-blocking** and a **single transactional boundary** across the composed steps.

***

## Option 3: Reactive in the Controller Only (Edge-Compatibility)

If your app is otherwise imperative, but you want a **reactive API signature**, you can wrap your existing imperative service in a `Mono`:

```java
@PostMapping("/submit")
public Mono<Response<String>> submit(@RequestBody Mono<ApplicationRequestDto> body) {
    return body.flatMap(dto ->
        Mono.fromCallable(() -> {
            applicationService.submitApplication(dto); // existing imperative @Transactional method
            return new Response.Builder<String>().setStatus(1000).setMessage("Success").build();
        }).subscribeOn(Schedulers.boundedElastic())
         .onErrorResume(e -> Mono.just(
                new Response.Builder<String>().setStatus(1003).setMessage(e.getMessage()).build()
         ))
    );
}
```

This gives **non-blocking HTTP** handling while the internal service remains blocking. It’s a pragmatic middle ground.

***

## Virtual Threads vs Reactor

*   **Virtual threads** (Java 21) simplify concurrency with blocking code. Great for **blocking JPA + JDBC**. You can simply run your existing code in virtual threads instead of introducing Reactor. Concurrency is easy, but watch your **connection pool**.
*   **Reactor** shines with **non-blocking I/O**. If your DB moves to R2DBC and your downstream calls are reactive, Reactor provides **backpressure** and efficient resource usage.

If your main bottleneck is DB, **virtual threads** might be simpler and effective now. If you’re building a fully reactive system (WebFlux + R2DBC + reactive stores), then **Mono/Flux** is the right direction.

***

## Transactional Gotchas

*   **Don’t** wrap a reactive chain in `@Transactional` (imperative). Use **`TransactionalOperator`** for reactive transactions.
*   **Don’t** expect transaction context to survive a **scheduler hop** with JPA and `@Transactional`.
*   If you split into parallel **sub-transactions**, use `@Transactional(REQUIRES_NEW)` per sub-step (called via a Spring proxy) or **programmatic** `TransactionTemplate`.

***

## Suggested approach for your current code

Given your existing **JPA** service and the desire to parallelize/isolate:

1.  **If you need one atomic commit**: keep it **imperative in one `@Transactional`** method (no Reactor).
2.  **If you want sub-transactions or concurrency** without R2DBC:
    *   Keep sub-step methods as `@Transactional(REQUIRES_NEW)` in a separate bean.
    *   Orchestrate in **Reactor** using `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())` and `Mono.zip`/`Mono.when`.
3.  Later: consider **R2DBC** and **TransactionalOperator** for fully reactive.

***

If you tell me which steps you want to run in parallel (e.g., Riders + Medicare + Existing + Beneficiaries + Documents + KYC), I can sketch the exact `Mono.zip` graph for your entities and show where to put sub-transactions vs a single transaction.


@Service
public class DocumentService {

    private final FileStorageService fileStorageService;

    public DocumentService(FileStorageService fileStorageService) {
        this.fileStorageService = fileStorageService;
    }

    public boolean isDocumentReceived(String applicationId, String documentId, MultipartFile file) {
        saveDocument(applicationId, documentId, file);
        return true;
    }

    @Async
    public void saveDocument(String applicationId, String documentId, MultipartFile file) {
        try {
            fileStorageService.uploadFileToBlobStorage(applicationId, documentId, file.getInputStream(), file.getSize());
        } catch (Exception ex) {
            throw new FileStorageException();
        }
    }
}
Call async methods via an injected dependency instead of directly via 'this'.


You're getting the warning **“Call async methods via an injected dependency instead of directly via `this`”** because **Spring @Async only works when the method is invoked through a Spring proxy**, not direct self-invocation.

Right now your code does this:

```java
saveDocument(applicationId, documentId, file);  // ❌ direct call
```

Calling the method on `this` bypasses Spring’s AOP proxy — so **async will NOT execute asynchronously**.

***

# ✅ Correct Fix: Inject the same service into itself

Spring recommends calling `@Async` methods **through a proxy**, which can be done by injecting the same bean back into itself.

### Step 1: Add a self-reference

```java
@Service
public class DocumentService {

    private final FileStorageService fileStorageService;

    // Self-injection
    private final DocumentService self;

    public DocumentService(FileStorageService fileStorageService, DocumentService self) {
        this.fileStorageService = fileStorageService;
        this.self = self;
    }

    public boolean isDocumentReceived(String applicationId, String documentId, MultipartFile file) {
        // Call async method via proxy
        self.saveDocument(applicationId, documentId, file);
        return true;
    }

    @Async
    public void saveDocument(String applicationId, String documentId, MultipartFile file) {
        try {
            fileStorageService.uploadFileToBlobStorage(
                applicationId,
                documentId,
                file.getInputStream(),
                file.getSize()
            );
        } catch (Exception ex) {
            throw new FileStorageException();
        }
    }
}
```

***

# ⚠️ Why this works

*   `DocumentService self` is the **Spring proxy** that contains the `@Async` interceptor.
*   Calling `self.saveDocument()` goes through the proxy → async works.
*   Calling `this.saveDocument()` or just `saveDocument()` bypasses proxy → async does **NOT** work.

***

# 🔥 Alternative (cleaner) solution: Move async logic to another bean

### Async class

```java
@Service
public class AsyncDocumentSaver {

    private final FileStorageService fileStorageService;

    public AsyncDocumentSaver(FileStorageService fileStorageService) {
        this.fileStorageService = fileStorageService;
    }

    @Async
    public void saveDocument(String applicationId, String documentId, MultipartFile file) {
        try {
            fileStorageService.uploadFileToBlobStorage(
                applicationId,
                documentId,
                file.getInputStream(),
                file.getSize()
            );
        } catch (Exception ex) {
            throw new FileStorageException();
        }
    }
}
```

### Main service

```java
@Service
public class DocumentService {

    private final AsyncDocumentSaver asyncSaver;

    public DocumentService(AsyncDocumentSaver asyncSaver) {
        this.asyncSaver = asyncSaver;
    }

    public boolean isDocumentReceived(String applicationId, String documentId, MultipartFile file) {
        asyncSaver.saveDocument(applicationId, documentId, file);
        return true;
    }
}
```

This is the **cleanest**, most maintainable way.

***

# 🧠 Summary

| Approach                          | Works? | Comment                             |
| --------------------------------- | ------ | ----------------------------------- |
| Calling `this.saveDocument()`     | ❌      | Bypasses proxy, async won't trigger |
| Calling `saveDocument()` directly | ❌      | Still self-invocation               |
| Injecting `DocumentService self`  | ✅      | Async works through proxy           |
| Using a separate async bean       | ✅      | Best practice                       |

***

If you want, I can also help you add **retry logic**, **exception logging**, or **return `CompletableFuture`** for result tracking.

