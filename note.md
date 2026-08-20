# Ghi chú NestJS – DI, JWT, Pipeline & OOP trong Base CRUD

## 📑 Mục lục

1. [Dependency Injection (DI)](#1-dependency-injection-di)
   - [1.1. Vấn đề khi KHÔNG có DI](#11-vấn-đề-khi-không-có-di)
   - [1.2. Giải pháp khi CÓ DI](#12-giải-pháp-khi-có-di)
   - [1.3. Định nghĩa DI](#13-định-nghĩa-di)
   - [1.4. DI trong NestJS (IoC Container)](#14-di-trong-nestjs-ioc-container)
   - [1.5. Câu trả lời phỏng vấn (elevator pitch)](#15-câu-trả-lời-phỏng-vấn-elevator-pitch)
   - [1.6. DI có phải là đa hình runtime không?](#16-di-có-phải-là-đa-hình-runtime-không)
2. [JWT & Authorization](#2-jwt--authorization)
   - [2.1. jwt.strategy.ts](#21-jwtstrategyts)
   - [2.2. jwt.guard.ts](#22-jwtguardts)
   - [2.3. role.guard.ts](#23-roleguardts)
3. [Pipeline xử lý Request trong NestJS (8 tầng)](#3-pipeline-xử-lý-request-trong-nestjs-8-tầng)
4. [Base CRUD Service – 4 tính chất OOP](#4-base-crud-service--4-tính-chất-oop)
   - [4.1. Tính trừu tượng (Abstraction)](#41-tính-trừu-tượng-abstraction)
   - [4.2. Tính kế thừa (Inheritance)](#42-tính-kế-thừa-inheritance)
   - [4.3. Tính đóng gói (Encapsulation)](#43-tính-đóng-gói-encapsulation)
   - [4.4. Tính đa hình (Polymorphism)](#44-tính-đa-hình-polymorphism)
   - [4.5. Bản trả lời phỏng vấn hoàn chỉnh](#45-bản-trả-lời-phỏng-vấn-hoàn-chỉnh)
5. [So sánh Trừu tượng vs Đóng gói](#5-so-sánh-trừu-tượng-vs-đóng-gói)
6. [Lý thuyết OOP tổng hợp (trả lời phỏng vấn)](#6-lý-thuyết-oop-tổng-hợp-trả-lời-phỏng-vấn)
7. [Override & Runtime Polymorphism](#7-override--runtime-polymorphism)

---

## 1. Dependency Injection (DI)

### 1.1. Vấn đề khi KHÔNG có DI

Mỗi file/service muốn dùng chung 5 dependency (`RedisService`, `RabbitMQService`, `JwtService`, `ConfigService`, `CredentialModel`) thì đều phải **copy-paste y hệt** đoạn khởi tạo:

```typescript
// File: auth.service.ts
class AuthService {
    private redisService: RedisService;
    private rabbitMQService: RabbitMQService;
    private jwtService: JwtService;
    private configService: ConfigService;
    private credentialModel: CredentialModel;

    constructor() {
        this.configService = new ConfigService();
        this.redisService = new RedisService(process.env.REDIS_URL);
        this.rabbitMQService = new RabbitMQService(process.env.RABBIT_HOST, process.env.RABBIT_USER, process.env.RABBIT_PASS);
        this.jwtService = new JwtService(process.env.JWT_SECRET);
        this.credentialModel = new CredentialModel(process.env.MONGO_URI);
    }
}

// File: test.service.ts — phải LẶP LẠI y hệt đoạn trên
class TestService {
    private redisService: RedisService;
    private rabbitMQService: RabbitMQService;
    private jwtService: JwtService;
    private configService: ConfigService;
    private credentialModel: CredentialModel;

    constructor() {
        this.configService = new ConfigService();
        this.redisService = new RedisService(process.env.REDIS_URL);
        this.rabbitMQService = new RabbitMQService(process.env.RABBIT_HOST, process.env.RABBIT_USER, process.env.RABBIT_PASS);
        this.jwtService = new JwtService(process.env.JWT_SECRET);
        this.credentialModel = new CredentialModel(process.env.MONGO_URI);
    }
}

// File: user.service.ts — lại LẶP LẠI lần nữa...
// File: notification.service.ts — lại LẶP LẠI lần nữa...
```

→ Mỗi service cần dùng 5 thứ đó thì phải copy nguyên đoạn khởi tạo này, biết chính xác `RedisService` cần url, `RabbitMQService` cần host/user/pass, biến môi trường tên gì...

**Các vấn đề phát sinh:**

- ❌ Vi phạm nguyên tắc **DRY** (Don't Repeat Yourself) — code trùng lặp khắp nơi.
- ❌ Mỗi service phải "biết" chi tiết cách khởi tạo của 5 dependency kia (cần tham số gì, lấy từ đâu) — trong khi đáng lẽ `TestService` chỉ cần biết "tôi có `redisService` để dùng", không cần quan tâm nó được tạo ra thế nào.
- ❌ **Đổi 1 chỗ, sửa khắp nơi** — ví dụ sau này `RedisService` đổi thêm tham số `RESP: 2`, phải lục tìm và sửa ở mọi file đã copy đoạn constructor này.
- ❌ **Tạo dư thừa kết nối** — mỗi lần `new` là 1 kết nối Redis/RabbitMQ/MongoDB mới, dù thực chất chỉ cần dùng chung 1.

### 1.2. Giải pháp khi CÓ DI

Chỉ viết code khởi tạo đúng **1 lần**, ở đúng **1 nơi**:

```typescript
// Chỉ viết logic connect trong RedisService — viết 1 lần duy nhất
@Injectable()
class RedisService implements OnModuleInit {
    async onModuleInit() {
        this.client = createClient({ url: this.configService.get('REDIS_URL') });
        await this.client.connect();
    }
}
```

Mọi service khác chỉ cần khai báo "tôi cần dùng" — không phải viết lại logic khởi tạo:

```typescript
class AuthService {
    constructor(private redisService: RedisService) {} // 1 dòng, xong
}

class TestService {
    constructor(private redisService: RedisService) {} // 1 dòng, xong — không copy-paste gì cả
}
```

### 1.3. Định nghĩa DI

**Dependency Injection (DI)** là một nguyên lý thiết kế phần mềm, trong đó một class không tự tạo (`new`) ra các đối tượng mà nó phụ thuộc, mà các đối tượng đó được tạo sẵn và "tiêm" từ bên ngoài vào — thường là qua constructor. Class chỉ cần khai báo "tôi cần cái gì", còn việc tạo ra nó và quản lý vòng đời của nó do framework (**IoC Container**) lo.

### 1.4. DI trong NestJS (IoC Container)

Trong NestJS, DI được triển khai qua **IoC Container** (Inversion of Control). Khi một class được đánh dấu `@Injectable()` và khai báo trong `providers` của Module, NestJS sẽ tự tạo ra đúng 1 instance duy nhất (mặc định là **singleton**), rồi tiêm chính instance đó vào bất kỳ class nào khai báo cần nó trong constructor.

**Vấn đề DI giải quyết:**

Nếu không có DI, mỗi service muốn dùng chung 1 dependency (ví dụ `RedisService`, `RabbitMQService`) đều phải tự viết lại đoạn code khởi tạo/kết nối (`new RedisService(url)`, `new RabbitMQService(host, user, pass)`...) — dẫn tới 3 vấn đề:

1. Code trùng lặp ở nhiều nơi, vi phạm nguyên tắc DRY.
2. Tạo dư thừa kết nối — mỗi lần `new` là 1 kết nối riêng (Redis, MongoDB, RabbitMQ...), trong khi đáng lẽ chỉ cần dùng chung 1.
3. Khó sửa và khó test — đổi cách khởi tạo phải sửa ở mọi nơi; muốn viết unit test (thay Redis thật bằng bản giả/mock) cũng khó vì dependency bị "hard-code" cứng trong class.

DI giải quyết bằng cách **tách việc "tạo object" ra khỏi việc "dùng object"**: class chỉ khai báo phụ thuộc, framework lo phần khởi tạo và dùng chung (singleton) — giúp code **loose coupling** (ít phụ thuộc chặt), dễ maintain, và dễ test.

### 1.5. Câu trả lời phỏng vấn (elevator pitch)

> DI là kỹ thuật để 1 class không tự tạo những thứ nó cần dùng, mà nhận (được tiêm) từ bên ngoài — thường qua constructor. Trong NestJS, framework tự tạo và quản lý các object này (mặc định singleton — chỉ 1 instance dùng chung toàn app), giúp code giảm phụ thuộc chặt, tránh lặp code khởi tạo, và dễ viết unit test hơn vì có thể dễ dàng thay bằng mock.

**💡 Mẹo khi phỏng vấn:** nếu được hỏi thêm "cho ví dụ thực tế", có thể kể lại đúng ví dụ `RedisService`/`RabbitMQService` ở trên — vì đã tự suy luận ra được cả 2 vấn đề cốt lõi (dư thừa kết nối + code trùng lặp), đó là hiểu bản chất thật sự, không phải học thuộc lòng.

### 1.6. DI có phải là đa hình runtime không?

**Có liên quan, nhưng không phải cùng một khái niệm.** Cảm giác chúng giống nhau là hợp lý vì cả hai đều có tinh thần: code sử dụng chỉ gọi qua một tên/kiểu chung, còn đối tượng hoặc hành vi thực tế phía sau có thể được quyết định khi chương trình chạy.

- **Đa hình qua override:** cùng lời gọi `this.prepareCreate(dto)`, nhưng chạy phiên bản của `CategoryService` hay `IngredientService` phụ thuộc vào object thật mà `this` đang trỏ tới lúc runtime.
- **DI:** `AuthService` chỉ khai báo rằng nó cần một dependency; object cụ thể được đưa vào là gì (Redis thật hay mock khi test) phụ thuộc vào cấu hình provider của DI container.

**Điểm khác nhau về bản chất:**

| Tiêu chí | Đa hình qua override | Dependency Injection |
|---|---|---|
| Giải quyết vấn đề gì? | **Hành vi nào** được thực thi khi gọi method | **Ai tạo và cung cấp object** cho class sử dụng |
| Cơ chế chính | Kế thừa, override và dynamic dispatch | Nhận dependency từ bên ngoài, thường qua constructor, thay vì tự `new` |
| Thuộc phạm trù | Cơ chế OOP về hành vi | Kỹ thuật thiết kế để thực hiện IoC và quản lý dependency |
| Có bắt buộc cần kế thừa/interface không? | Override cần quan hệ class cha–class con | Không; inject trực tiếp một class cụ thể vẫn là DI |

Ví dụ dưới đây **đã là DI**, dù `AuthService` phụ thuộc trực tiếp vào class cụ thể và chưa dùng đa hình:

```typescript
@Injectable()
class AuthService {
  constructor(private readonly cache: RedisService) {}
}
```

#### Chỗ DI và đa hình thật sự kết hợp

DI phát huy tính linh hoạt mạnh nhất khi class sử dụng phụ thuộc vào một **abstraction** thay vì implementation cụ thể. Khi đó cùng một code của `AuthService` có thể hoạt động với nhiều implementation khác nhau:

```typescript
interface CacheService {
  get(key: string): Promise<string | null>;
}

const CACHE_SERVICE = Symbol('CACHE_SERVICE');

@Injectable()
class AuthService {
  constructor(
    @Inject(CACHE_SERVICE)
    private readonly cache: CacheService,
  ) {}
}
```

Binding dùng Redis trong ứng dụng thật:

```typescript
@Module({
  providers: [
    AuthService,
    RedisService,
    { provide: CACHE_SERVICE, useExisting: RedisService },
  ],
})
class AuthModule {}
```

Khi unit test, có thể thay provider bằng mock mà không sửa code của `AuthService`:

```typescript
const moduleRef = await Test.createTestingModule({
  providers: [
    AuthService,
    {
      provide: CACHE_SERVICE,
      useValue: { get: jest.fn() },
    },
  ],
}).compile();
```

> **Lưu ý TypeScript/NestJS:** không thể chỉ viết `constructor(cache: CacheService)` rồi mong Nest tự inject theo interface, vì interface TypeScript bị xóa sau khi compile và không tồn tại ở runtime. Cần một runtime token như `Symbol`, string hoặc abstract class, rồi dùng `@Inject(token)`.

Trường hợp trên có cả hai cơ chế:

1. **DI** chịu trách nhiệm tạo/chọn object và đưa nó vào constructor.
2. **Đa hình** cho phép `AuthService` gọi cùng method `cache.get()` nhưng nhận hành vi khác nhau từ `RedisService` hoặc mock.

Điều này cũng phù hợp với **Dependency Inversion Principle (DIP)** — chữ **D** trong SOLID: module cấp cao nên phụ thuộc vào abstraction, không phụ thuộc chặt vào implementation cụ thể. Tuy nhiên cần nhớ: **DI và DIP liên quan nhưng không đồng nghĩa**; DI là một kỹ thuật thường được dùng để hiện thực hóa DIP.

**Câu trả lời gọn khi phỏng vấn:**

> "DI và đa hình có liên quan nhưng khác tầng. Đa hình quyết định hành vi nào chạy cho cùng một lời gọi dựa trên object thực tế. DI tách việc tạo và cung cấp object ra khỏi class sử dụng. DI không bắt buộc phải có đa hình, nhưng khi inject theo abstraction và đổi được implementation thật/mock, DI đang tận dụng đa hình để code linh hoạt và dễ test hơn."

[⬆ Về mục lục](#-mục-lục)

---

## 2. JWT & Authorization

### 2.1. jwt.strategy.ts
📁 `jwt/jwt.strategy.ts`

**Nhiệm vụ:** Kiểm tra token có hợp lệ không, và token đó là của ai.

- Lấy token từ header `Authorization`.
- Gọi sang Auth Service (qua HTTP) để hỏi "token này còn dùng được không, user tương ứng là ai".
- Nếu hợp lệ → trả về thông tin user (`_id`, `email`, `role`...) để gắn vào `request.user`.

### 2.2. jwt.guard.ts
📁 `jwt/jwt.guard.ts`

**Nhiệm vụ:** Quyết định route hiện tại có cần đăng nhập hay không, và kích hoạt việc kiểm tra token.

- Nếu route được đánh dấu `@Public()` → bỏ qua, cho vào thẳng, không cần token.
- Nếu không → gọi `JwtStrategy` để xác thực token; token sai/hết hạn → chặn lại, trả lỗi **401 Unauthorized**.

### 2.3. role.guard.ts
📁 `role/role.guard.ts`

**Nhiệm vụ:** Kiểm tra user đã đăng nhập có đủ quyền (role) để truy cập route này không.

- Chạy **sau** `JwtAuthGuard` (vì cần biết `request.user.role` trước, mà thông tin này do `JwtStrategy` gắn vào).
- So sánh role của user (ví dụ `admin`, `manager`, `user`) với danh sách role được phép truy cập route đó (thường khai báo qua decorator kiểu `@Roles('admin')`).
- Nếu role không khớp → chặn, trả lỗi **403 Forbidden** (khác với 401 — đây là "biết bạn là ai rồi, nhưng bạn không đủ quyền").

[⬆ Về mục lục](#-mục-lục)

---

## 3. Pipeline xử lý Request trong NestJS (8 tầng)

Luồng xử lý request trong NestJS đi qua nhiều tầng theo thứ tự cố định. Đây là toàn bộ pipeline từ lúc client gửi request đến khi trả response:

```
Client Request
     │
     ▼
1. Middleware
     │
     ▼
2. Guards
     │
     ▼
3. Interceptors (trước khi vào handler)
     │
     ▼
4. Pipes
     │
     ▼
5. Controller (Route Handler)
     │
     ▼
6. Service (business logic)
     │
     ▼
7. Interceptors (sau khi handler trả về - xử lý response)
     │
     ▼
8. Exception Filters (nếu có lỗi ném ra ở bất kỳ bước nào)
     │
     ▼
Client Response
```

**Giải thích từng tầng:**

1. **Middleware** — chạy đầu tiên, giống Express middleware. Dùng để log request, parse cookie, kiểm tra header thô... Không biết route handler nào sẽ được gọi.
2. **Guards** — quyết định request có được phép đi tiếp hay không (`return true/false`). Thường dùng cho Authentication & Authorization (ví dụ `JwtAuthGuard`, `RolesGuard`). Nếu return `false` → NestJS tự động trả **403 Forbidden**, không đi tiếp xuống dưới.
3. **Interceptors (pre-controller)** — chạy trước khi vào handler, có thể transform request, đo thời gian, thêm logic trước khi gọi controller (dùng `intercept()` với `next.handle()`).
4. **Pipes** — validate và transform dữ liệu đầu vào (`@Body()`, `@Param()`, `@Query()`). Ví dụ `ValidationPipe` kiểm tra DTO có đúng schema không, `ParseIntPipe` convert string → number. Nếu invalid → ném lỗi **400 Bad Request**.
5. **Controller** — nhận request đã được pipe xử lý, gọi xuống Service để thực thi logic nghiệp vụ.
6. **Service** — chứa business logic thật sự (query DB, gọi API khác...), trả kết quả về Controller.
7. **Interceptors (post-controller)** — sau khi Controller trả về, Interceptor có thể transform response (ví dụ wrap response theo format chuẩn `{ data, statusCode }`, cache response...).
8. **Exception Filters** — nếu bất kỳ bước nào ném Exception, filter sẽ bắt và format lỗi trả về cho client (ví dụ `HttpExceptionFilter`).

[⬆ Về mục lục](#-mục-lục)

---

## 4. Base CRUD Service – 4 tính chất OOP

> ⚠️ Đúng hướng nhưng nếu chỉ dừng ở mức "định nghĩa suông" thì chưa đủ — cần chỉ ra **bằng chứng cụ thể trong code**, nếu không sẽ nghe giống học thuộc lòng. Interviewer giỏi sẽ hỏi tiếp "cho ví dụ trong code em vừa nói" — cần trả lời ngay không ấp úng.

### 4.1. Tính trừu tượng (Abstraction)

Không chỉ là "khai báo abstract" — mà là: **ẩn đi chi tiết triển khai phức tạp, chỉ lộ ra interface cần thiết**.

**Bằng chứng:** `BaseCrudService` ẩn hoàn toàn logic phân trang, check trùng field, bắt lỗi MongoDB duplicate key (code `11000`)... Class con (`CategoryService`) chỉ cần biết "tôi cần gọi `create(dto)`", không cần biết bên trong nó có `ensureUnique`, `rethrowPersistenceError` hay không. Đó là trừu tượng hóa nghiệp vụ CRUD thành một khái niệm chung, không quan tâm resource cụ thể là gì.

> **Trả lời phỏng vấn:** "Em trừu tượng hóa toàn bộ logic CRUD chung (phân trang, check unique, bắt lỗi trùng key) vào 1 class abstract, các class nghiệp vụ chỉ cần quan tâm 'làm gì' chứ không cần biết 'làm như thế nào'."

### 4.2. Tính kế thừa (Inheritance)

Không phải kế thừa cho có, mà để **tái sử dụng code + mở rộng có kiểm soát**.

**Bằng chứng:** `CategoryService extends BaseCrudService<...>` — kế thừa toàn bộ `findAll`/`findOne`/`create`/`update`/`delete` mà không viết lại 1 dòng nào, đồng thời override đúng 2 method (`prepareCreate`, `prepareUpdate`) để thêm logic riêng (`trim()`). Đây là **kế thừa có chọn lọc** — không phải kế thừa để copy y nguyên.

### 4.3. Tính đóng gói (Encapsulation)

Đóng gói không chỉ là gắn `private`/`protected`, mà là **kiểm soát ai được đụng vào cái gì**.

> Ví dụ `model` (kết nối MongoDB) để `protected` — chỉ class con thấy được, Controller không thấy. Controller muốn tạo/sửa/xóa dữ liệu thì phải gọi qua hàm `create`, `update` public, mà bên trong mấy hàm đó đã validate, check trùng sẵn rồi. Nên Controller không thể bỏ qua bước kiểm tra đó được.
>
> Còn mấy hàm nhỏ như `ensureUnique` hay `notFound` để `private`, vì đó chỉ là cách làm bên trong thôi. Sau này đổi cách check trùng khác đi, bên ngoài cũng không bị ảnh hưởng gì, vì họ đâu có gọi trực tiếp vào đó.

**Bằng chứng cụ thể (2 chỗ trong code, nhìn cả `base-crud.controller.ts` và `base-crud.service.ts` cùng lúc mới thấy rõ):**

**a) Controller không hề có `model` — chỉ có `service`:**

```typescript
// base-crud.controller.ts
export abstract class BaseCrudController<TDocument, TCreateDto, TUpdateDto> {
  constructor(
    protected readonly service: BaseCrudService<TDocument, TCreateDto, TUpdateDto>,
    //         ↑ Controller CHỈ cầm được "service", không hề có "model" ở đây
  ) {}
}
```

`BaseCrudController` không có property `model` nào cả — nó chỉ giữ 1 tham chiếu tới `service`. Vì `model` nằm trong `BaseCrudService` và được khai báo `protected`, và `BaseCrudController` không `extends` `BaseCrudService` mà chỉ cầm 1 instance của nó (composition, không phải kế thừa), nên `model` hoàn toàn vô hình với Controller — không có cách nào gõ `this.service.model` được, TypeScript sẽ báo lỗi ngay.

**b) Controller bắt buộc gọi qua method public của service, và method đó đã validate sẵn:**

```typescript
// base-crud.controller.ts — bên trong createCrudController()
@Post()
@WriteAuthorization
async create(@Body(createPipe) dto: TCreateDto): Promise<CrudResponse<TDocument>> {
  return { success: true, data: await this.service.create(dto) };
  //                              ↑ chỉ gọi được service.create(), không có cách nào khác
}

@Patch(':id')
@WriteAuthorization
async update(...): Promise<CrudResponse<TDocument>> {
  return { success: true, data: await this.service.update(id, dto) };
  //                              ↑ tương tự
}
```

Controller chỉ có thể gọi `this.service.create(dto)` hoặc `this.service.update(id, dto)` — 2 method này public trong `BaseCrudService`, nên gọi được. Nhưng bên trong 2 method này đã có sẵn validate/check trùng rồi:

```typescript
// base-crud.service.ts
async create(dto: TCreateDto): Promise<TDocument> {
  const data = await this.prepareCreate(dto);
  await this.ensureUnique(data);      // ← check trùng, Controller KHÔNG thể bỏ qua bước này
  await this.beforeCreate(data);
  try {
    const document = new this.model(data);   // ← chỉ có method NÀY mới đụng được this.model
    const saved = await document.save();
    ...
```

**Tóm lại — chuỗi logic đầy đủ:**

| Muốn làm | Controller có làm được không? | Vì sao |
|---|---|---|
| `this.service.model.deleteMany({})` (né hết validate) | ❌ Không | Vì `model` là `protected` trong `BaseCrudService`, và `BaseCrudController` không extends `BaseCrudService` nên không thấy được |
| `this.service.create(dto)` | ✅ Có | Vì `create` là public, nhưng bên trong nó đã tự động chạy `ensureUnique` — Controller không có cách nào gọi `create` mà "nhảy cóc" qua bước check trùng này |

→ 2 đoạn code này (constructor của `BaseCrudController` chỉ nhận `service`, và method `create`/`update` public trong `BaseCrudService` có validate bên trong) chính là bằng chứng cụ thể cho câu trả lời phỏng vấn — không phải suy luận chung chung, mà chỉ thẳng được vào 2 dòng code này khi bị hỏi "ví dụ đâu".

### 4.4. Tính đa hình (Polymorphism)

Đây là tính dễ bị bỏ sót nhất vì trong base CRUD nó không "hiển nhiên" như 3 tính kia, nhưng thực ra có **2 dạng đa hình** rất rõ:

**a) Đa hình qua override (runtime polymorphism / method overriding)**

Method `create()` trong `BaseCrudService` gọi `this.prepareCreate(dto)` — nhưng tùy class con nào đang chạy, `this.prepareCreate` sẽ thực thi phiên bản khác nhau:

- `CategoryService.prepareCreate` → trim `name`, `description`
- Nếu có `IngredientService` override khác → chạy logic khác hoàn toàn

Cùng một dòng code `await this.prepareCreate(dto)` trong `base-crud.service.ts`, nhưng hành vi thực thi phụ thuộc vào object nào gọi nó lúc runtime — đây chính là định nghĩa sách giáo khoa của đa hình: **"cùng 1 lời gọi, nhiều hành vi khác nhau tùy đối tượng"**.

**b) Đa hình qua Generic (parametric polymorphism)**

`BaseCrudService<TDocument, TCreateDto, TUpdateDto>` là 1 đoạn code duy nhất nhưng hoạt động đúng đắn với nhiều kiểu dữ liệu khác nhau (`Category`, `Ingredient`, `MenuItem`...) mà vẫn giữ type-safety — không cần viết lại, không cần ép `any`. Đây là dạng đa hình phổ biến trong TypeScript/Java/C# (generic), khác với đa hình override ở chỗ nó xảy ra ở **compile-time** (TypeScript biết chính xác kiểu tại mỗi lần dùng), không phải runtime.

### 4.5. Bản trả lời phỏng vấn hoàn chỉnh

> "Em có 1 ví dụ thực tế project em làm — base CRUD Service dùng chung cho nhiều module.
>
> - **Trừu tượng:** em gom toàn bộ logic CRUD chung (phân trang, check trùng, bắt lỗi DB) vào 1 abstract class, class con không cần biết chi tiết bên trong.
> - **Kế thừa:** `CategoryService extends BaseCrudService`, tái sử dụng gần như toàn bộ logic, chỉ override đúng phần cần custom.
> - **Đóng gói:** Model DB là `protected`, Controller không đụng trực tiếp được, bắt buộc đi qua method public đã có validate.
> - **Đa hình:** có 2 dạng — một là override method (`prepareCreate`) chạy hành vi khác nhau tùy class con lúc runtime; hai là dùng Generic để 1 class dùng chung được cho nhiều kiểu dữ liệu (`Category`, `Ingredient`...) mà vẫn type-safe lúc compile-time."

[⬆ Về mục lục](#-mục-lục)

---

## 5. So sánh Trừu tượng vs Đóng gói

Trừu tượng và đóng gói đều có yếu tố "ẩn", nhưng **mục đích khác nhau**.

- **Trừu tượng (Abstraction)** là ẩn đi sự phức tạp để người dùng chỉ cần biết hệ thống làm được gì, không cần biết bên trong làm như thế nào. Ví dụ trong project, `BaseCrudService` gom toàn bộ logic như phân trang, check unique, xử lý duplicate key, save database... vào các hàm chung như `create()`, `update()`, `findAll()`. `CategoryService` chỉ cần gọi hoặc kế thừa các hàm đó mà không cần quan tâm chi tiết triển khai bên trong.

- **Đóng gói (Encapsulation)** là kiểm soát quyền truy cập vào các thành phần bên trong, tức là quy định cái gì bên ngoài được phép dùng và cái gì không được phép đụng tới. Ví dụ `model` được khai báo `protected`, `ensureUnique` được để `private`, nên Controller không thể truy cập trực tiếp vào `model` hoặc gọi các helper nội bộ. Controller chỉ được sử dụng các method public như `create()`, `update()`, `findAll()`.

**Nói ngắn gọn:**

- Trừu tượng là: **"Không cần biết bên trong làm như thế nào."**
- Đóng gói là: **"Không được phép đụng trực tiếp vào bên trong."**

**Ví dụ với `create()`:**

- Khi Controller gọi `service.create(dto)`, đó là **trừu tượng** vì Controller chỉ biết rằng `create` dùng để tạo dữ liệu, không cần biết bên trong phải prepare data, check unique, save MongoDB hay xử lý lỗi.
- Nhưng Controller không thể gọi `service.model` hoặc `service.ensureUnique()` vì những thành phần đó bị giới hạn bằng `protected`/`private`. Đó là **đóng gói**.

[⬆ Về mục lục](#-mục-lục)

---

## 6. Lý thuyết OOP tổng hợp (trả lời phỏng vấn)

**1. Tính trừu tượng – Abstraction**
Trừu tượng là ẩn đi sự phức tạp để người dùng chỉ cần biết hệ thống làm được gì, không cần biết bên trong làm như thế nào.

**2. Tính đóng gói – Encapsulation**
Đóng gói là kiểm soát quyền truy cập vào các thành phần bên trong, tức là quy định cái gì bên ngoài được phép dùng và cái gì không được phép đụng tới.

**3. Tính kế thừa – Inheritance**
Là cho phép một class con kế thừa thuộc tính và phương thức của class cha. Mục đích là tái sử dụng code và cho phép class con mở rộng hoặc thay đổi hành vi của class cha.

**4. Tính đa hình – Polymorphism**
Là cùng một interface hoặc cùng một lời gọi phương thức nhưng có thể có nhiều cách thực thi khác nhau tùy đối tượng cụ thể. Mục đích là giúp chương trình linh hoạt và dễ mở rộng.

[⬆ Về mục lục](#-mục-lục)

---

## 7. Override & Runtime Polymorphism

### Override là gì

**Override (ghi đè)** là khi một class con định nghĩa lại một method đã có sẵn trong class cha, dùng đúng tên và chữ ký gần giống, để thay thế hành vi mặc định bằng hành vi riêng của mình.

```typescript
// Class cha (BaseCrudService) — có sẵn hành vi mặc định
protected prepareCreate(dto: TCreateDto): Record<string, unknown> {
  return { ...(dto as Record<string, unknown>) };   // mặc định: giữ nguyên dto
}

// Class con (CategoryService) — override lại, thay hành vi khác
protected prepareCreate(dto: CreateCategoryDto): Record<string, unknown> {
  return { ...dto, name: dto.name.trim(), description: dto.description.trim() };
}
```

`CategoryService` không viết lại toàn bộ `create()`, chỉ ghi đè đúng 1 method con (`prepareCreate`) để chèn logic `trim()` vào.

### Runtime là gì, và tại sao override gắn liền với runtime

**Runtime** = thời điểm chương trình đang chạy thực tế, khác với **compile-time** = thời điểm code được biên dịch/kiểm tra kiểu trước khi chạy.

Override được gọi là "runtime polymorphism" vì: khi TypeScript biên dịch dòng code

```typescript
await this.prepareCreate(dto);
```

nằm trong `base-crud.service.ts`, trình biên dịch không biết trước `this` sẽ là instance của class nào — `CategoryService`, `IngredientService`, hay class nào khác kế thừa `BaseCrudService`. Nó chỉ biết `this` có kiểu `BaseCrudService`.

Chỉ đến khi chương trình thực sự chạy (runtime) — tức là lúc NestJS tạo ra một instance thật, ví dụ `new CategoryService(...)`, và instance đó gọi `create()` — thì hệ thống mới tra cứu xem `this` đang là object nào, rồi quyết định chạy đúng bản `prepareCreate` của `CategoryService`. Cơ chế tra cứu này gọi là **dynamic dispatch** (điều phối động).

**Ví dụ minh họa luồng runtime thật:**

```typescript
const service = new CategoryService(model);   // this ở đây LÀ CategoryService

await service.create(dto);  
// bên trong create() có dòng: await this.prepareCreate(dto)
// → lúc CODE ĐANG CHẠY, JS/TS mới tra: "this" hiện tại là CategoryService
// → chạy đúng bản trim() của CategoryService, KHÔNG chạy bản mặc định của base
```

Nếu ngày mai có thêm `IngredientService extends BaseCrudService` với `prepareCreate` khác, thì không cần sửa 1 dòng nào trong `base-crud.service.ts` — chỉ cần gọi `ingredientService.create()`, và cùng dòng `this.prepareCreate(dto)` đó tự động chạy ra hành vi khác. Đây chính là ý nghĩa "linh hoạt, dễ mở rộng" trong định nghĩa đa hình.

**So sánh nhanh để nhớ:**

| | Compile-time | Runtime |
|---|---|---|
| Khi nào quyết định | Lúc biên dịch/viết code | Lúc chương trình chạy |
| Biết trước method nào chạy? | Không (chỉ biết kiểu chung) | Có — dựa vào object thật |
| Ví dụ trong code | Generic `<TDocument>` | Override `prepareCreate` |

**Câu trả lời gọn cho interviewer:**

> "Override là runtime polymorphism vì compiler không biết trước `this` sẽ là instance nào lúc viết code trong base class — chỉ đến khi chương trình chạy thật, gọi `service.create()` với `service` là instance cụ thể, hệ thống mới tra và chạy đúng bản `prepareCreate` của class đó. Đây gọi là dynamic dispatch."

[⬆ Về mục lục](#-mục-lục)
