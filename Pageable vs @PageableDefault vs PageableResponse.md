Got it 👍  
You’re **already on the right track** by using `Pageable`, `@PageableDefault`, and a generic `PageableResponse`.  
I’ll now **complete the flow cleanly** in the same style you’re using — **no redesign**, just filling the missing pieces and best practices.

I’ll cover:

1.  Repository
2.  Entity → DTO mapping
3.  `PageableResponse` utility
4.  Service (completed)
5.  Controller (final version)

***

## 1️⃣ Repository (you already almost have this)

```java
@Repository
public interface SchedulerRepository
        extends JpaRepository<IwsSchedulerTiming, Long> {
}
```

✅ `findAll(Pageable pageable)` is already provided by Spring Data JPA.

***

## 2️⃣ Response Model (DTO)

This is what each row in `data[]` should look like.

```java
public class SchedulerListResponseModel {

    private Long id;
    private String month;
    private Integer year;
    private String sendingStopDateTime;
    private String sendingStartDateTime;
    private String message;

    // getters & setters
}
```

***

## 3️⃣ PageableResponse (generic wrapper)

This converts `Page<T>` → API response.

```java
public class PageableResponse<T> {

    private long count;
    private int pageNo;
    private int perPage;
    private int totalPage;
    private List<T> data;

    public static <T> PageableResponse<T> get(Page<T> page) {
        PageableResponse<T> response = new PageableResponse<>();
        response.setCount(page.getTotalElements());
        response.setPageNo(page.getNumber() + 1); // Pageable is 0‑based
        response.setPerPage(page.getSize());
        response.setTotalPage(page.getTotalPages());
        response.setData(page.getContent());
        return response;
    }

    // getters & setters
}
```

✅ Reusable for **any paginated endpoint**

***

## 4️⃣ Service (completed properly)

You must **map Entity → DTO**, not return entities directly.

```java
@Service
public class SchedulerService {

    private final SchedulerRepository schedulerRepository;

    public SchedulerService(SchedulerRepository schedulerRepository) {
        this.schedulerRepository = schedulerRepository;
    }

    public PageableResponse<SchedulerListResponseModel> getSchedules(Pageable pageable) {

        Page<IwsSchedulerTiming> page = schedulerRepository.findAll(pageable);

        Page<SchedulerListResponseModel> dtoPage =
                page.map(this::mapToResponse);

        return PageableResponse.get(dtoPage);
    }

    private SchedulerListResponseModel mapToResponse(IwsSchedulerTiming entity) {

        DateTimeFormatter formatter =
                DateTimeFormatter.ofPattern("M/d/yyyy h:mm:ss a");

        SchedulerListResponseModel dto = new SchedulerListResponseModel();
        dto.setId(entity.getId());
        dto.setMessage(entity.getMessage());

        if (entity.getSendingStopDateTime() != null) {
            dto.setMonth(entity.getSendingStopDateTime()
                    .getMonth()
                    .getDisplayName(TextStyle.FULL, Locale.ENGLISH));
            dto.setYear(entity.getSendingStopDateTime().getYear());
            dto.setSendingStopDateTime(
                    entity.getSendingStopDateTime().format(formatter));
        }

        if (entity.getSendingStartDateTime() != null) {
            dto.setSendingStartDateTime(
                    entity.getSendingStartDateTime().format(formatter));
        }

        return dto;
    }
}
```

✅ Matches response:

    3/12/2026 12:55:00 PM
    March
    2026

***

## 5️⃣ Controller (your style, finalized)

```java
@RestController
@RequestMapping("/ExternalV2/external")
public class SchedulerController {

    private final SchedulerService schedulerService;

    public SchedulerController(SchedulerService schedulerService) {
        this.schedulerService = schedulerService;
    }

    @GetMapping("/schedules")
    public PageableResponse<SchedulerListResponseModel> getSchedules(
            @ParameterObject
            @PageableDefault(
                page = 0,
                size = 10,
                sort = "id",
                direction = Sort.Direction.DESC
            )
            Pageable pageable) {

        return schedulerService.getSchedules(pageable);
    }
}
```

✅ Supports:

    GET /schedules?page=0&size=10
    GET /schedules?page=1&size=20
    GET /schedules?sort=id,desc

***

## ✅ End‑to‑End Flow (what happens)

    Controller
       ↓ Pageable
    Service
       ↓ Page<Entity>
       ↓ Page<DTO>
    PageableResponse.get()
       ↓
    JSON response

***

## ✅ Example API Response

```json
{
  "count": 12,
  "pageNo": 1,
  "perPage": 10,
  "totalPage": 2,
  "data": [
    {
      "id": 13,
      "month": "March",
      "year": 2026,
      "sendingStopDateTime": "3/12/2026 12:55:00 PM",
      "sendingStartDateTime": "3/16/2026 8:00:00 PM",
      "message": "Business Scheduler will be off till 16th March"
    }
  ]
}
```

***

## ✅ Why this approach is GOOD

✔ Uses Spring Data pagination correctly  
✔ Clean controller (no logic)  
✔ Reusable `PageableResponse`  
✔ Proper DTO mapping  
✔ Ready for Swagger / OpenAPI

***

If you want next:

*   ✅ Filtering by month/year
*   ✅ Native query for performance
*   ✅ POST‑based pagination instead of GET
*   ✅ Swagger annotations

Just tell me 👍
