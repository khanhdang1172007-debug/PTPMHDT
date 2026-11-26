# Ch04 — Bài tập thực hành: EF Core, PostgreSQL và Full API (EduHub)

Đã hoàn thiện đầy đủ 6 nhiệm vụ và kiểm chứng:

- `dotnet build`: 0 lỗi, 0 cảnh báo.
- `dotnet test`: 24/24 pass. Test chạy bằng SQLite in-memory và Moq nên không cần PostgreSQL thật.
- Chạy thật với PostgreSQL (migration `Init`, seeding, đăng ký 201/409/400, báo cáo Dapper, ProblemDetails). Lệnh `docker compose up` cần Docker Desktop đang chạy.

## Cấu trúc

```
EduHub.Domain          Entities (Member, Club, Event, Registration) + interface repository + IUnitOfWork
EduHub.Application     Handler/Service, DTO + mapping, PagedResult/PageRequest, AppException, IEventReportQuery
EduHub.Infrastructure  EduHubDbContext (Fluent API + SeedAsync), EF repository, EfUnitOfWork,
                       DapperEventReportQuery (báo cáo SQL thuần), Migrations/Init
EduHub.API             Controllers, Requests (validation), GlobalExceptionHandler (ProblemDetails), Serilog, Dockerfile
EduHub.Tests           xUnit + Moq + SQLite in-memory
```

## Quyết định thiết kế (khi nào cần, khi nào thừa)

- Có `IEventRepository`, `IClubRepository`, `IRegistrationRepository`: mỗi cái ứng với một aggregate có truy vấn riêng,
  giúp lớp Application test bằng Moq và không phụ thuộc EF Core.
- Không có `IMemberRepository` và không có `IRepository<T>` tổng quát: Member chỉ được đọc để kiểm tra tồn tại
  (đã nằm trong `IRegistrationRepository`). Generic repository chỉ bọc lại `DbSet<T>`, không thêm giá trị.
- `IUnitOfWork` mỏng bọc `DbContext.SaveChangesAsync`: repository không tự lưu, handler gọi lưu đúng một lần
  để mọi thay đổi trong một use case nằm trong một transaction.
- Truy vấn báo cáo (JOIN, GROUP BY) viết bằng Dapper qua `IEventReportQuery`, tách khỏi repository của entity.

## Chạy bằng Docker (API và DB)

```bash
cd Bai_tap_c4
docker compose up -d --build        # DB_PORT=5433 nếu máy đã có Postgres chiếm 5432
# Swagger: http://localhost:8080/swagger
docker compose down -v
```

API tự `MigrateAsync()` và `SeedAsync()` khi khởi động (`Database__AutoMigrate=true`).
Connection string lấy từ biến môi trường `ConnectionStrings__DefaultConnection` trong compose.

## Chạy API trên máy (DB trong Docker)

```bash
docker compose up -d postgres
dotnet run --project EduHub.API --urls http://localhost:5000     # tự migrate và seed ở Development
# Swagger: http://localhost:5000/swagger
```

Cổng khác: `export ConnectionStrings__DefaultConnection="Host=localhost;Port=5433;Database=eduhub;Username=postgres;Password=secret"`

Tạo lại migration: `dotnet ef migrations add Init --project EduHub.Infrastructure --startup-project EduHub.API`

## Endpoint

| Method | Route | Kết quả |
|---|---|---|
| GET | `/api/clubs?search=&page=&pageSize=` | 200 `{items,totalCount,page,pageSize,totalPages}` |
| GET, PUT, DELETE | `/api/clubs/{id}` | 200 / 404, 204 |
| POST | `/api/clubs` | 201 / 400 |
| GET | `/api/events?clubId=&search=&page=&pageSize=` | 200 (phân trang) |
| GET | `/api/events/{id}` | 200 / 404 |
| POST | `/api/events` | 201 / 400 (Name rỗng, Capacity <= 0, StartTime <= now, club không tồn tại) |
| PUT | `/api/events/{id}` | 200 / 400 / 404 |
| DELETE | `/api/events/{id}` | 204 / 404 |
| POST | `/api/events/{eventId}/registrations` | 201 / 400 "Sự kiện đã đạt giới hạn." / 404 / 409 |
| GET | `/api/events/{eventId}/registrations` | 200 (kèm thông tin Member) / 404 |
| DELETE | `/api/events/{eventId}/registrations/{registrationId}` | 204 / 404 (hủy đăng ký) |
| GET | `/api/reports/events?minRegistered=` | 200 (Dapper: số lượt đăng ký mỗi sự kiện) |

Mọi lỗi 4xx/5xx trả `application/problem+json` (RFC 9457) có `traceId`.

## Ghi chú cho giảng viên

- Seed: 3 clubs, 2 events (Workshop .NET 10 sức chứa 100; Hackathon nhỏ sức chứa 2 để demo "đầy"), 3 members SV001–SV003.
- Demo 409/400: đăng ký SV001 hai lần (409), rồi SV002, rồi SV003 (400 đầy). Lấy `memberId` từ DB:
  `docker compose exec postgres psql -U postgres -d eduhub -c 'select "Id","StudentId" from members'`.
- Tìm kiếm dùng `ToLower().Contains(...)` để chạy được cả PostgreSQL và SQLite (khi test). Riêng PostgreSQL có thể dùng `EF.Functions.ILike`.
- Dùng `curl -G --data-urlencode "search=trí"` khi tìm tiếng Việt.
- Cảnh báo `ERR ... __EFMigrationsHistory` ở lần chạy đầu trên DB trống là bình thường (EF kiểm tra rồi mới tạo bảng).
