# Tài liệu kiến trúc: Base CRUD Generic trong NestJS

> Tài liệu này giải thích chi tiết cơ chế **Base CRUD** dùng chung cho toàn bộ project (Category, Ingredients, Menu Items...), các thuật ngữ TypeScript/NestJS liên quan, và luồng xử lý request từ đầu đến cuối.

---

## 1. Vấn đề mà Base CRUD giải quyết

Nếu không có pattern này, mỗi module (category, ingredients, menu-items, orders...) sẽ phải viết lặp lại:

```typescript
// Lặp lại ở MỌI module nếu không dùng base
async findAll(query) { /* phân trang, tìm kiếm, sắp xếp... */ }
async findOne(id)    { /* tìm theo id, ném NotFoundException nếu không có */ }
async create(dto)    { /* validate unique, lưu DB, bắt lỗi duplicate key */ }
async update(id, dto){ /* tìm hiện tại, merge dữ liệu, lưu lại */ }
async delete(id)     { /* tìm, xóa */ }
```

→ **10 module = 10 lần copy-paste code gần giống hệt nhau.** Sửa 1 lỗi phải sửa 10 chỗ.

**Giải pháp:** Viết logic CRUD **một lần duy nhất** trong 2 class trừu tượng (`BaseCrudService`, `BaseCrudController`). Mỗi module cụ thể chỉ cần khai báo **cấu hình** (tên field, field nào unique, field nào tìm kiếm được...) và override phần logic riêng nếu cần.

---

## 2. Các thuật ngữ nền tảng

### 2.1. Generic (`<T>`)

**Generic** cho phép viết code dùng được với **nhiều kiểu dữ liệu khác nhau** mà vẫn giữ được type-safety (TypeScript vẫn biết chính xác kiểu dữ liệu, không phải `any`).

```typescript
export abstract class BaseCrudService<
  TDocument extends Document,   // kiểu document Mongoose, VD: CategoryDocument
  TCreateDto extends object,    // kiểu DTO khi tạo, VD: CreateCategoryDto
  TUpdateDto extends object,    // kiểu DTO khi sửa, VD: UpdateCategoryDto
> { ... }
```

Ví dụ đơn giản để hiểu generic:
```typescript
function firstItem<T>(arr: T[]): T {
  return arr[0];
}
firstItem<string>(['a', 'b']);   // trả về string
firstItem<number>([1, 2]);       // trả về number
```

Trong project: `BaseCrudService<CategoryDocument, CreateCategoryDto, UpdateCategoryDto>` nghĩa là "phiên bản CRUD Service này chuyên xử lý dữ liệu Category, không phải Ingredient hay MenuItem". TypeScript sẽ tự động check kiểu đúng ở mọi nơi bạn dùng `this.model`, `dto`, kết quả trả về...

**Ràng buộc generic (`extends`)**: `TDocument extends Document` nghĩa là generic `TDocument` **bắt buộc phải là một Mongoose Document** (hoặc kiểu con của nó), không được truyền bừa kiểu khác vào.

### 2.2. Abstract class (`abstract`)

**Abstract class** là class **không thể khởi tạo trực tiếp** (`new BaseCrudService()` sẽ báo lỗi), chỉ dùng để **class khác kế thừa (extends)**.

```typescript
export abstract class BaseCrudService<...> {
  protected constructor(...) {}   // constructor là protected → chỉ class con gọi được
  async findAll() { ... }         // method thường: dùng chung, không override
  protected prepareCreate() { ... } // method có thể override
}
```

Tại sao dùng abstract thay vì class thường?
- Ép buộc: `BaseCrudService` **chỉ có ý nghĩa khi được mở rộng** — tự nó không đại diện cho resource cụ thể nào (không có "Category chung chung").
- An toàn: tránh việc ai đó lỡ tay tạo `new BaseCrudService()` mà không gắn với Model/DTO cụ thể.

**Trong project:** `CategoryService extends BaseCrudService<...>` — Category là class con thực thi, `BaseCrudService` là khuôn mẫu (template) trừu tượng.

### 2.3. Protected vs Private vs Public

| Từ khóa | Ai gọi được |
|---|---|
| `public` (mặc định) | Gọi từ bất kỳ đâu, kể cả bên ngoài class |
| `protected` | Chỉ chính class đó và **class con (extends)** gọi được |
| `private` | Chỉ chính class đó gọi được, class con cũng không được |

Trong `BaseCrudService`:
- `findAll`, `findOne`, `create`, `update`, `delete` → **public**: Controller gọi trực tiếp.
- `prepareCreate`, `beforeCreate`, `afterCreate`... → **protected**: chỉ class con (CategoryService) override được, bên ngoài (Controller) không gọi trực tiếp được.
- `buildSearchFilter`, `resolveSort`, `ensureUnique`, `notFound`... → **private**: chi tiết triển khai nội bộ, không class con nào cần biết hay đụng vào.

→ Đây chính là nguyên tắc **encapsulation (đóng gói)**: chỉ "lộ" ra đúng những gì cần thiết.

### 2.4. Template Method Pattern (Hook Method)

Đây là **design pattern** mà `BaseCrudService` áp dụng: định nghĩa **khung xử lý cố định** (`create`, `update`...), nhưng cho phép class con "cắm" logic riêng vào từng bước thông qua các **hook method**.

```typescript
async create(dto: TCreateDto): Promise<TDocument> {
  const data = await this.prepareCreate(dto);   // [HOOK] class con có thể override
  await this.ensureUnique(data);                // logic cố định, không đổi
  await this.beforeCreate(data);                // [HOOK] class con có thể override
  const saved = await document.save();          // logic cố định
  await this.afterCreate(saved);                // [HOOK] class con có thể override
  return saved;
}
```

Nếu class con **không override** hook, hook mặc định làm không gì cả (`void data;` chỉ để tránh warning "unused parameter"):
```typescript
protected beforeCreate(data: Record<string, unknown>): void | Promise<void> {
  void data;   // không làm gì — hook rỗng mặc định
}
```

**Ứng dụng trong Category:**
```typescript
protected prepareCreate(dto: CreateCategoryDto): Record<string, unknown> {
  return {
    ...dto,
    name: dto.name.trim(),               // ghi đè: tự trim khoảng trắng
    description: dto.description.trim(),
  };
}
```
→ Category "cắm" thêm logic trim() vào đúng bước `prepareCreate`, còn toàn bộ luồng còn lại (`ensureUnique`, `save`, bắt lỗi duplicate key...) vẫn dùng nguyên bản từ base, **không cần viết lại**.

### 2.5. Interface & Type

```typescript
export interface BaseCrudOptions<TDocument extends Document> {
  resourceName: string;
  defaultSort?: { field: string; order: 'asc' | 'desc' };
  allowedSortFields?: readonly string[];
  searchFields?: readonly string[];
  uniqueFields?: readonly (keyof TDocument & string)[];
}
```

**Interface** định nghĩa "hình dạng" (shape) của một object — object phải có đúng các field này (field có `?` là optional). Không chứa logic, chỉ là hợp đồng về kiểu dữ liệu.

Điểm hay: `uniqueFields?: readonly (keyof TDocument & string)[]`
- `keyof TDocument` = union tất cả tên field có trong `TDocument` (VD: `'name' | 'description' | 'displayOrder' | ...`)
- Nghĩa là bạn **chỉ được truyền tên field thật sự tồn tại** trong schema Category, gõ sai tên field (VD: `'nam'` thay vì `'name'`) → TypeScript báo lỗi ngay lúc code, không cần chạy mới biết.

### 2.6. Factory Function (Hàm khởi tạo class động)

```typescript
export function createCrudController<TDocument, TCreateDto, TUpdateDto>(
  options: CrudControllerOptions<TCreateDto, TUpdateDto>,
): CrudControllerConstructor<TDocument, TCreateDto, TUpdateDto> {
  class CrudController extends BaseCrudController<...> {
    @Get() async findAll() {...}
    @Post() @WriteAuthorization async create() {...}
    // ...
  }
  return CrudController;   // trả về CHÍNH class (constructor), không phải instance
}
```

Đây không phải class thông thường — đây là **hàm trả về một class**. Mỗi lần gọi `createCrudController({...})`, một class Controller **mới** được sinh ra động, đã gắn sẵn 5 route REST tiêu chuẩn. `CategoryController` sau đó `extends` class được sinh ra này để "thừa hưởng" toàn bộ route mà không cần viết lại `@Get()`, `@Post()`...

### 2.7. Decorator (`@Controller`, `@Injectable`, `@Get`, `@UseGuards`...)

**Decorator** là cú pháp `@TênDecorator` gắn phía trên class/method/property để **gắn thêm metadata hoặc hành vi** mà không sửa code gốc.

| Decorator | Ý nghĩa |
|---|---|
| `@Injectable()` | Đánh dấu class này là 1 **Provider** — NestJS quản lý vòng đời, có thể inject vào class khác |
| `@Controller('path')` | Đánh dấu class xử lý HTTP route, prefix `path` |
| `@Get()`, `@Post()`, `@Patch()`, `@Delete()` | Gắn method xử lý HTTP method tương ứng |
| `@Body()`, `@Param()`, `@Query()` | Lấy dữ liệu từ request (body/route param/query string) |
| `@UseGuards(RolesGuard)` | Gắn Guard bảo vệ route/controller |
| `@InjectModel(Category.name)` | Inject Mongoose Model vào constructor |

**`applyDecorators`** (dùng trong `WriteAuthorization`):
```typescript
const WriteAuthorization = applyDecorators(
  ...(writeRoles.length > 0 ? [Roles(...writeRoles)] : []),
);
```
Gộp nhiều decorator thành 1 decorator duy nhất — nếu `writeRoles` rỗng thì không gắn `@Roles()` nào cả (route không bị giới hạn role).

### 2.8. Dependency Injection (DI)

NestJS tự động **"tiêm" (inject)** các dependency vào constructor thay vì bạn phải tự `new` ra:

```typescript
export class CategoryController extends CategoryCrudController {
  constructor(categoryService: CategoryService) {   // NestJS tự tạo & truyền vào
    super(categoryService);
  }
}
```

Bạn không bao giờ viết `new CategoryService()` — NestJS đọc `@Injectable()` trên `CategoryService`, tự tạo instance (dùng chung — singleton theo mặc định), rồi truyền vào bất kỳ đâu cần nó.

### 2.9. Pipe (`PipeTransform`)

**Pipe** biến đổi/validate dữ liệu đầu vào **trước khi** đến Controller handler.

```typescript
@Injectable()
export class DtoValidationPipe<T extends object> implements PipeTransform {
  transform(value: unknown, metadata: ArgumentMetadata): Promise<T> {
    return this.validationPipe.transform(value, {
      ...metadata,
      metatype: this.dtoType,   // gắn "cứng" kiểu DTO cần validate
    });
  }
}
```

**Vấn đề nó giải quyết:** Generic Controller (`createCrudController`) không biết trước class DTO cụ thể là gì tại thời điểm biên dịch (vì generic bị **xóa lúc runtime** — đây là giới hạn nổi tiếng của TypeScript gọi là "type erasure"). `DtoValidationPipe` nhận `dtoType` qua constructor **lúc runtime** để biết chính xác cần validate theo DTO nào (`CreateCategoryDto` hay `CreateIngredientDto`...).

`ParseObjectIdPipe` tương tự nhưng để validate `:id` trong URL có đúng định dạng MongoDB ObjectId hay không.

### 2.10. Guard (`CanActivate`)

**Guard** quyết định request có được đi tiếp hay bị chặn (`403 Forbidden`), chạy **trước** Pipe và Controller. `RolesGuard` đọc metadata gắn bởi `@Roles(...)` để kiểm tra user hiện tại (lấy từ JWT) có role phù hợp không.

---

## 3. Sơ đồ luồng file trong `common/crud`

```
common/crud/
├── crud.types.ts           → Định nghĩa Interface (BaseCrudOptions, CrudResponse...)
├── crud-query.dto.ts       → DTO cho query string (?page=&limit=&q=&sortBy=&sortOrder=)
├── dto-validation.pipe.ts  → Pipe validate Body theo DTO cụ thể (runtime)
├── base-crud.service.ts    → Class abstract xử lý logic DB (CRUD + hook)
├── base-crud.controller.ts → Class abstract + factory sinh Controller có sẵn route
└── index.ts                → Export tất cả để module khác import gọn: from '../../common/crud'
```

---

## 4. Luồng xử lý chi tiết: `POST /api/canteen/categories`

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Client gửi POST /api/canteen/categories                       │
│    Body: { name: "Đồ uống", description: "..." }                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. @UseGuards(RolesGuard) trên CategoryController                │
│    → Đọc metadata @Roles(ADMIN, MANAGER) gắn bởi @WriteAuthorization │
│    → Kiểm tra user (từ JWT) có role ADMIN/MANAGER không          │
│    → Không đúng role → 403 Forbidden, dừng luồng                 │
└─────────────────────────────────────────────────────────────────┘
                              │ (role hợp lệ)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. @Body(createPipe) dto: TCreateDto                              │
│    createPipe = new DtoValidationPipe(CreateCategoryDto)         │
│    → Validate body đúng theo class-validator rules trong          │
│      CreateCategoryDto (VD: @IsString(), @IsNotEmpty()...)        │
│    → Sai format → 400 Bad Request, dừng luồng                    │
└─────────────────────────────────────────────────────────────────┘
                              │ (dto hợp lệ)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. CrudController.create(dto)  [sinh bởi createCrudController]   │
│    return { success: true, data: await this.service.create(dto) }│
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. CategoryService.create(dto)  [kế thừa BaseCrudService.create] │
│                                                                    │
│    a) prepareCreate(dto)   → [OVERRIDE bởi Category] trim name/desc │
│    b) ensureUnique(data)   → check field 'name' đã tồn tại chưa   │
│                               (uniqueFields: ['name'])             │
│       → Trùng → 409 ConflictException, dừng luồng                │
│    c) beforeCreate(data)   → [không override] hook rỗng          │
│    d) new this.model(data) → tạo Mongoose document                │
│    e) document.save()      → ghi xuống MongoDB                    │
│       → Lỗi duplicate key (code 11000) → bắt & convert thành       │
│         409 ConflictException                                     │
│    f) afterCreate(saved)   → [không override] hook rỗng          │
│    g) return saved                                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. Trả về client:                                                 │
│    { success: true, data: { _id, name, description, ... } }      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Luồng `GET /api/canteen/categories?page=1&limit=10&q=trà&sortBy=name&sortOrder=asc`

```
1. RolesGuard → route GET không có @Roles() → cho qua (public)
2. @Query() query: CrudQueryDto → validate/transform page,limit là số nguyên hợp lệ
3. CrudController.findAll(query)
4. BaseCrudService.findAll(query)
   a) buildSearchFilter('trà')
      → fields = searchFields = ['name', 'description']
      → filter = { $or: [{ name: {$regex:'trà', $options:'i'} },
                          { description: {$regex:'trà', $options:'i'} }] }
   b) resolveSort('name', 'asc')
      → check 'name' có nằm trong allowedSortFields không → có → dùng
   c) Promise.all([model.find(filter).sort().skip().limit().exec(),
                    model.countDocuments(filter).exec()])
      → chạy song song đếm tổng + lấy dữ liệu trang hiện tại
   d) return { data, meta: { page, limit, total, totalPages } }
5. Trả về: { success: true, data: [...], meta: {...} }
```

---

## 6. Bảng ánh xạ: Thuật ngữ → Vị trí trong code → Vai trò

| Thuật ngữ | Xuất hiện ở đâu | Vai trò |
|---|---|---|
| Generic `<T>` | `BaseCrudService<TDocument, TCreateDto, TUpdateDto>` | Dùng 1 class cho nhiều "loại" resource khác nhau, vẫn giữ type-safety |
| Abstract class | `abstract class BaseCrudService`, `BaseCrudController` | Class khuôn mẫu, bắt buộc phải extends mới dùng được |
| Protected constructor | `protected constructor(model, options)` | Chỉ class con gọi `super()` được, không `new` trực tiếp từ ngoài |
| Template Method | `create()`, `update()` gọi các hook `prepareX/beforeX/afterX` | Khung xử lý cố định, cho phép "cắm" logic riêng từng bước |
| Hook method | `prepareCreate`, `beforeUpdate`, `afterDelete`... | Điểm mở rộng — override khi Category cần logic riêng |
| Interface | `BaseCrudOptions`, `CrudResponse`, `CrudListResult` | Định nghĩa hình dạng dữ liệu, không chứa logic |
| `keyof` | `uniqueFields?: readonly (keyof TDocument & string)[]` | Ràng buộc chỉ được truyền tên field thật sự có trong schema |
| Factory function | `createCrudController(options)` | Sinh động 1 class Controller có sẵn 5 route REST |
| Decorator | `@Controller`, `@Get`, `@Post`, `@UseGuards`, `@Roles` | Gắn metadata/hành vi cho class/method mà không sửa logic gốc |
| `applyDecorators` | `WriteAuthorization = applyDecorators(...)` | Gộp nhiều decorator, có điều kiện, thành 1 decorator |
| Dependency Injection | constructor nhận `Model`, `Service` | NestJS tự tạo & truyền instance, không cần `new` thủ công |
| Pipe | `DtoValidationPipe`, `ParseObjectIdPipe` | Validate/transform dữ liệu đầu vào trước khi vào Controller |
| Guard | `RolesGuard` + `@UseGuards` | Chặn/cho qua request dựa trên quyền (role) |
| Type erasure | Lý do cần `DtoValidationPipe` nhận `dtoType` qua constructor | Generic TypeScript biến mất lúc runtime, cần truyền type "thủ công" |

---

## 7. Cách tạo module mới theo pattern này (VD: Ingredient)

```typescript
// 1. Service — kế thừa BaseCrudService, khai báo config
@Injectable()
export class IngredientService extends BaseCrudService<
  IngredientDocument, CreateIngredientDto, UpdateIngredientDto
> {
  constructor(@InjectModel(Ingredient.name) model: Model<IngredientDocument>) {
    super(model, {
      resourceName: 'Nguyên liệu',
      defaultSort: { field: 'name', order: 'asc' },
      allowedSortFields: ['name', 'quantity', 'createdAt'],
      searchFields: ['name'],
      uniqueFields: ['name'],
    });
  }

  // Override hook nếu cần logic riêng, VD: không cho quantity âm
  protected prepareUpdate(dto: UpdateIngredientDto): Record<string, unknown> {
    if (dto.quantity !== undefined && dto.quantity < 0) {
      throw new BadRequestException('Số lượng không được âm');
    }
    return { ...dto };
  }
}

// 2. Controller — dùng factory, khai báo DTO + role
const IngredientCrudController = createCrudController<
  IngredientDocument, CreateIngredientDto, UpdateIngredientDto
>({
  createDto: CreateIngredientDto,
  updateDto: UpdateIngredientDto,
  writeRoles: [Role.ADMIN],
});

@Controller('api/canteen/ingredients')
@UseGuards(RolesGuard)
export class IngredientController extends IngredientCrudController {
  constructor(service: IngredientService) {
    super(service);
  }
}
```

→ Chỉ ~30 dòng code là có đủ 5 API chuẩn, phân trang, tìm kiếm, sắp xếp, validate, phân quyền, bắt lỗi trùng dữ liệu.

---

## 8. Lưu ý khi mở rộng/debug base này

- **Muốn thêm logic riêng cho 1 module** → override đúng hook (`prepareX/beforeX/afterX`) trong Service, **không sửa** `base-crud.service.ts` (sửa file base sẽ ảnh hưởng toàn bộ module khác).
- **Muốn route GET cũng cần đăng nhập** (không public) → thêm `@Roles()` hoặc guard riêng cho `findAll`/`findOne`, hiện tại 2 route này không có `@WriteAuthorization`.
- **Muốn thêm route ngoài 5 route chuẩn** (VD: `PATCH /:id/toggle-active`) → viết thêm method trong `CategoryController` (class con), decorator `@Patch(':id/toggle-active')` như bình thường, vẫn dùng chung `this.service`.
- **Debug lỗi 409 Conflict bất ngờ** → kiểm tra `uniqueFields` trong config Service, có thể field đó đang bị check trùng dữ liệu ngoài ý muốn.
- **Debug lỗi 400 khi sort** → kiểm tra `allowedSortFields`, field client gửi lên phải nằm trong danh sách này.

---

## 9. Giải thích chi tiết từng dòng — `base-crud.service.ts`

### 9.1. Khai báo class và constructor

```typescript
export abstract class BaseCrudService<
  TDocument extends Document,
  TCreateDto extends object,
  TUpdateDto extends object,
> {
  protected constructor(
    protected readonly model: Model<TDocument>,
    protected readonly options: BaseCrudOptions<TDocument>,
  ) {}
```

- `protected readonly model` — vừa là **parameter property** (cú pháp rút gọn TypeScript: khai báo tham số constructor kèm access modifier sẽ **tự động** tạo thành property của class, không cần viết `this.model = model` thủ công).
- `readonly` — chỉ gán được **một lần** (trong constructor), sau đó không thể `this.model = ...` lại nữa → tránh bug vô tình đổi model giữa chừng.
- `protected` trên `model`/`options` → class con (`CategoryService`) **đọc được** `this.model`, `this.options` nhưng code bên ngoài (Controller) **không truy cập trực tiếp được** — đúng nguyên tắc chỉ Service mới được "chạm" vào DB.
- Constructor là `protected` (không phải `public`) → **không ai `new BaseCrudService(...)` trực tiếp được**, kể cả từ ngoài package. Chỉ có class con gọi `super(model, options)` bên trong constructor của chính nó.

### 9.2. `findAll` — phân trang, tìm kiếm, sắp xếp

```typescript
async findAll(query: CrudQueryDto): Promise<CrudListResult<TDocument>> {
  const page = query.page ?? 1;
  const limit = query.limit ?? 20;
```
- `??` là **nullish coalescing operator**: chỉ dùng giá trị mặc định (`1`, `20`) khi `query.page`/`query.limit` là `null` hoặc `undefined` — khác với `||` (sẽ coi `0` là falsy và ghi đè nhầm). Ở đây `CrudQueryDto` đã có default value (`page = 1`) nên trường hợp `undefined` gần như không xảy ra, nhưng `??` vẫn là lớp bảo vệ thứ 2.

```typescript
  const filter = this.buildSearchFilter(query.q);
  const sort = this.resolveSort(query.sortBy, query.sortOrder);

  const [data, total] = await Promise.all([
    this.model.find(filter).sort({ [sort.field]: sort.order === 'asc' ? 1 : -1 })
      .skip((page - 1) * limit).limit(limit).exec(),
    this.model.countDocuments(filter).exec(),
  ]);
```
- `Promise.all([...])` — chạy **song song** 2 query MongoDB (lấy dữ liệu trang hiện tại + đếm tổng số bản ghi) thay vì tuần tự (`await` từng cái) → giảm thời gian phản hồi gần một nửa.
- `{ [sort.field]: ... }` — **computed property name**: dùng biến `sort.field` (VD: `'name'`) làm **tên key** của object động, tương đương `{ name: 1 }` hoặc `{ name: -1 }` tùy `asc`/`desc`. MongoDB dùng `1` = tăng dần, `-1` = giảm dần.
- `.skip((page - 1) * limit).limit(limit)` — công thức phân trang chuẩn: trang 1 skip 0, trang 2 skip `limit` bản ghi đầu, v.v.

```typescript
  return {
    data,
    meta: { page, limit, total, totalPages: total === 0 ? 0 : Math.ceil(total / limit) },
  };
}
```
- `totalPages` dùng `Math.ceil` (làm tròn lên) vì nếu còn dư bản ghi lẻ vẫn cần thêm 1 trang. Check `total === 0` riêng để tránh chia cho phép tính ra `0` bị hiểu nhầm — thực chất `Math.ceil(0/20) = 0` đã đúng sẵn, dòng check này chủ yếu làm rõ ý nghĩa (defensive code).

### 9.3. `findOne` — tìm theo ID, ném lỗi nếu không có

```typescript
async findOne(id: string): Promise<TDocument> {
  const document = await this.model.findById(id).exec();
  if (!document) {
    throw this.notFound(id);
  }
  return document;
}
```
- Đây là method được **tái sử dụng nhiều nơi khác** trong class: `update()` gọi `findOne()` để lấy bản ghi hiện tại trước khi sửa, `delete()` cũng vậy — tránh viết lại logic "tìm + báo lỗi 404" nhiều lần.
- `throw this.notFound(id)` — `notFound()` là **private method** chỉ tạo và trả về đối tượng `NotFoundException`, còn `throw` mới thực sự ném lỗi. Tách riêng để có thể tái sử dụng cùng 1 format message lỗi ở nhiều chỗ.

### 9.4. `create` — luồng đầy đủ + try/catch

```typescript
async create(dto: TCreateDto): Promise<TDocument> {
  const data = await this.prepareCreate(dto);
  await this.ensureUnique(data);
  await this.beforeCreate(data);

  try {
    const document = new this.model(data);
    const saved = await document.save();
    await this.afterCreate(saved);
    return saved;
  } catch (error: unknown) {
    this.rethrowPersistenceError(error);
  }
}
```
- `new this.model(data)` — `this.model` là **Mongoose Model class được truyền vào lúc runtime** (constructor injection), nên `new this.model(...)` tương đương `new CategoryModel(...)` nhưng generic, dùng chung được cho mọi resource.
- `catch (error: unknown)` — dùng `unknown` thay vì `any`: **best practice TypeScript**. `unknown` bắt buộc phải **kiểm tra kiểu (type-narrowing)** trước khi dùng (không thể gọi `error.code` trực tiếp), tránh runtime error nếu `error` không đúng cấu trúc mong đợi. So sánh:
  - `any` → tắt hoàn toàn type-check, dễ gây bug ẩn.
  - `unknown` → an toàn hơn, buộc phải "chứng minh" kiểu trước khi thao tác (xem mục 9.7).
- `rethrowPersistenceError(error)` có kiểu trả về `never` (xem 9.7) — nghĩa là hàm này **luôn luôn throw**, không bao giờ return bình thường, nên TypeScript hiểu rằng nếu code chạy tới sau dòng này thì chắc chắn đã có exception, không cần `return` thừa.

### 9.5. `update` — merge dữ liệu vào document đang có

```typescript
async update(id: string, dto: TUpdateDto): Promise<TDocument> {
  const current = await this.findOne(id);
  const data = await this.prepareUpdate(dto, current);
  if (Object.keys(data).length === 0) {
    throw new BadRequestException('Cần cung cấp ít nhất một trường để cập nhật');
  }
  await this.ensureUnique(data, id);
  await this.beforeUpdate(current, data);

  try {
    current.set(data as UpdateQuery<TDocument>);
    const saved = await current.save();
    await this.afterUpdate(saved);
    return saved;
  } catch (error: unknown) {
    this.rethrowPersistenceError(error);
  }
}
```
- `current.set(data)` — dùng method `.set()` của Mongoose document (không phải tạo document mới) để **merge từng field** đã thay đổi vào document hiện có, giữ nguyên các field không được gửi lên trong `dto` (đúng ngữ nghĩa `PATCH` — cập nhật một phần, khác `PUT` là thay thế toàn bộ).
- `data as UpdateQuery<TDocument>` — đây là **type assertion** (ép kiểu): báo cho TypeScript "tôi chắc chắn `data` tương thích với `UpdateQuery<TDocument>`", cần thiết vì `data` đang có kiểu `Record<string, unknown>` (object động, generic) còn `.set()` của Mongoose yêu cầu kiểu cụ thể hơn `UpdateQuery<TDocument>`. Đây là điểm TypeScript "chịu thua" trước tính linh hoạt của generic — buộc phải ép kiểu thủ công, đổi lại phải rất cẩn thận đảm bảo field đúng tên/đúng kiểu ở tầng DTO.
- `ensureUnique(data, id)` — truyền thêm `id` để **loại trừ chính bản ghi đang sửa** khỏi việc check trùng (xem 9.8: `$ne: excludeId`) — nếu không, sửa Category mà không đổi `name` cũng sẽ tự báo lỗi "trùng" với chính nó.

### 9.6. `delete`

```typescript
async delete(id: string): Promise<TDocument> {
  const current = await this.findOne(id);
  await this.beforeDelete(current);
  await current.deleteOne();
  await this.afterDelete(current);
  return current;
}
```
- `current.deleteOne()` — gọi trên **instance document** (Mongoose instance method), khác với `Model.deleteOne(filter)` (gọi trên Model, cần truyền filter). Cách này đảm bảo xóa **đúng chính xác** document đã tìm được ở bước `findOne`.
- Trả về `current` (dữ liệu **trước khi bị xóa**) để Controller có thể trả về cho client biết vừa xóa cái gì (hữu ích cho log, thông báo, undo...).

### 9.7. Các hook method — kiểu trả về `void | Promise<void>`

```typescript
protected beforeCreate(data: Record<string, unknown>): void | Promise<void> {
  void data;
}
```
- Kiểu trả về `void | Promise<void>` — **union type**: hook có thể là **đồng bộ** (return `void`, không trả gì) hoặc **bất đồng bộ** (`async` method, return `Promise<void>`). Ở method gọi hook, luôn dùng `await this.beforeCreate(data)` — `await` trên giá trị **không phải Promise** vẫn hoạt động bình thường (JavaScript tự "unwrap", coi như resolve ngay lập tức), nên hook dù đồng bộ hay bất đồng bộ đều gọi được thống nhất bằng `await`.
- `void data;` — đây **không phải** kiểu `void`, mà là **toán tử `void`** áp dụng lên biến `data`: cách viết "tôi cố tình không dùng biến này" để tránh cảnh báo `no-unused-vars` từ ESLint/TypeScript, vì tham số phải khai báo (để giữ đúng signature cho class con override) nhưng hook mặc định không làm gì với nó.

### 9.8. `ensureUnique` — kiểm tra trùng dữ liệu

```typescript
private async ensureUnique(data: Record<string, unknown>, excludeId?: string): Promise<void> {
  for (const field of this.options.uniqueFields ?? []) {
    if (data[field] === undefined) {
      continue;
    }
    const filter: QueryFilter<TDocument> = {
      [field]: data[field],
      ...(excludeId ? { _id: { $ne: excludeId } } : {}),
    };
    const exists = await this.model.exists(filter).exec();
    if (exists) {
      throw new ConflictException(
        `${this.options.resourceName} có ${field} '${this.formatValue(data[field])}' đã tồn tại`,
      );
    }
  }
}
```
- `this.options.uniqueFields ?? []` — nếu module không khai báo `uniqueFields` (VD: module nào đó cho phép trùng dữ liệu), vòng lặp `for` sẽ chạy trên mảng rỗng → không check gì cả, không lỗi.
- `if (data[field] === undefined) continue` — chỉ check trùng cho field **thực sự có mặt** trong `data` (VD: khi `update` chỉ gửi `{ displayOrder: 5 }`, không có `name` → bỏ qua check `name`, không tốn 1 query DB thừa).
- `...(excludeId ? { _id: { $ne: excludeId } } : {})` — **spread có điều kiện**: nếu có `excludeId`, thêm điều kiện MongoDB `$ne` (not equal) để loại trừ chính document đang sửa; nếu không có `excludeId` (trường hợp `create`), spread ra object rỗng `{}` — không thêm gì.
- `this.model.exists(filter)` — Mongoose method tối ưu hơn `findOne()` vì chỉ trả `{ _id }` hoặc `null`, không load toàn bộ document, nhanh hơn khi chỉ cần biết "có tồn tại hay không".

### 9.9. `rethrowPersistenceError` và Type Guard — kiểu trả về `never`

```typescript
private rethrowPersistenceError(error: unknown): never {
  if (this.isDuplicateKeyError(error)) {
    const field = Object.keys(error.keyPattern ?? {})[0] ?? 'dữ liệu';
    throw new ConflictException(`${this.options.resourceName} có ${field} bị trùng`);
  }
  throw error;
}

private isDuplicateKeyError(
  error: unknown,
): error is MongoDuplicateKeyError & { code: 11000 } {
  return (
    typeof error === 'object' && error !== null &&
    'code' in error && error.code === 11000
  );
}
```
- `: never` — kiểu trả về đặc biệt nghĩa là **hàm này không bao giờ trả về giá trị bình thường**, chỉ có thể kết thúc bằng `throw` (hoặc vòng lặp vô hạn). TypeScript dùng thông tin này để biết chắc rằng bất cứ dòng code nào gọi hàm `never` xong đều **không thể tiếp tục chạy tiếp** một cách hợp lệ (giúp phát hiện "dead code" nếu ai đó cố dùng giá trị trả về của nó).
- `error is MongoDuplicateKeyError & { code: 11000 }` — đây là **Type Predicate** (hay còn gọi **Type Guard function**): báo cho TypeScript biết "nếu hàm này trả về `true`, hãy tự động coi `error` có kiểu chính xác là `MongoDuplicateKeyError & { code: 11000 }` từ dòng đó trở đi" (gọi là **type narrowing**). Nhờ vậy trong `rethrowPersistenceError`, sau khi qua `if (this.isDuplicateKeyError(error))`, TypeScript **tự động cho phép** gọi `error.keyPattern` mà không cần ép kiểu (`as`) — nếu không có type guard, `error` vẫn giữ kiểu `unknown` và gọi `.keyPattern` sẽ bị báo lỗi biên dịch.
- `MongoDuplicateKeyError & { code: 11000 }` — **Intersection Type** (giao của 2 kiểu): kết hợp interface `MongoDuplicateKeyError` (có `code?: number`, `keyPattern?: ...`) với việc **thu hẹp cụ thể** `code` phải đúng bằng **literal `11000`** (không phải `number` chung chung) — đây là mã lỗi MongoDB dành riêng cho "duplicate key" (vi phạm unique index).

### 9.10. `formatValue` — an toàn khi hiển thị giá trị lỗi

```typescript
private formatValue(value: unknown): string {
  if (typeof value === 'string' || typeof value === 'number' ||
      typeof value === 'boolean' || typeof value === 'bigint') {
    return `${value}`;
  }
  return JSON.stringify(value) ?? '[giá trị]';
}
```
- Không dùng `String(value)` hay `${value}` trực tiếp cho **mọi kiểu**, vì nếu `value` là object/array, `${value}` sẽ in ra `"[object Object]"` vô nghĩa. Hàm này **narrow kiểu** trước: nếu là kiểu nguyên thủy → convert string bình thường; nếu là object → `JSON.stringify` để hiển thị dễ đọc hơn trong message lỗi.
- `?? '[giá trị]'` — phòng trường hợp hiếm `JSON.stringify` trả về `undefined` (VD: `value` là `function` hoặc `Symbol`, dù thực tế khó xảy ra ở đây vì input đến từ dữ liệu MongoDB).

---

## 10. Giải thích chi tiết — `base-crud.controller.ts` và factory

### 10.1. Vì sao cần 2 khái niệm: `BaseCrudController` (abstract class) và `createCrudController` (factory function)?

```typescript
export abstract class BaseCrudController<TDocument, TCreateDto, TUpdateDto> {
  constructor(protected readonly service: BaseCrudService<TDocument, TCreateDto, TUpdateDto>) {}
}
```
- `BaseCrudController` **chỉ giữ 1 nhiệm vụ**: định nghĩa rằng "mọi Controller CRUD đều phải nhận 1 Service qua constructor". Nó **không có route nào cả** — vì route (`@Get`, `@Post`...) cần biết **DTO cụ thể** (`CreateCategoryDto` hay `CreateIngredientDto`) để gắn Pipe đúng, mà class abstract tĩnh (static) không thể biết trước DTO nào lúc viết code chung.
- `createCrudController(options)` giải quyết đúng chỗ đó: nhận `options.createDto`, `options.updateDto` là **giá trị runtime cụ thể** (class thật, không phải generic trừu tượng), rồi **sinh động một class con** có đủ route gắn đúng Pipe cho từng DTO đó.

### 10.2. Kiểu `CrudControllerConstructor` — Constructor Type

```typescript
type CrudControllerConstructor<TDocument, TCreateDto, TUpdateDto> = new (
  service: BaseCrudService<TDocument, TCreateDto, TUpdateDto>,
) => BaseCrudController<TDocument, TCreateDto, TUpdateDto>;
```
- Đây là kiểu mô tả **"một class có thể `new` ra được"** (constructor signature), không phải mô tả 1 instance. Từ khóa `new (...) => ReturnType` là cú pháp TypeScript để khai báo kiểu của **chính class** (constructor function), chứ không phải kiểu của object được tạo ra.
- Nhờ vậy hàm `createCrudController` có thể khai báo kiểu trả về chính xác: **"trả về một class, class đó nhận `service` trong constructor, và khi `new` ra sẽ là instance của `BaseCrudController`"**. Đây chính là kiểu trả về khai báo trong factory function ở phần 2.6.

### 10.3. Vì sao Pipe/decorator được khởi tạo **ngoài** class, không phải trong mỗi method?

```typescript
const idPipe = new ParseObjectIdPipe(options.idFieldName);
const createPipe = new DtoValidationPipe(options.createDto);
const updatePipe = new DtoValidationPipe(options.updateDto);
const writeRoles = options.writeRoles ?? [];
const WriteAuthorization = applyDecorators(...(writeRoles.length > 0 ? [Roles(...writeRoles)] : []));

class CrudController extends BaseCrudController<...> {
  @Get(':id')
  async findOne(@Param('id', idPipe) id: string) { ... }
```
- Các dòng `const idPipe = ...` nằm **bên ngoài** `class CrudController` nhưng **bên trong** hàm `createCrudController` — đây là kỹ thuật tận dụng **closure** (bao đóng) của JavaScript/TypeScript: class `CrudController` được định nghĩa **bên trong** thân hàm nên các method của nó **"nhớ" được** các biến `idPipe`, `createPipe`... dù hàm `createCrudController` đã chạy xong và trả về class.
- Lợi ích: mỗi Pipe chỉ cần khởi tạo **1 lần duy nhất** khi module khởi động (khi `createCrudController()` được gọi lúc định nghĩa `CategoryController`), tái sử dụng cho **mọi request** sau đó, không tốn chi phí tạo instance mới mỗi lần có request tới.

### 10.4. Từng route giải thích

```typescript
@Get()
async findAll(@Query() query: CrudQueryDto): Promise<CrudListResponse<TDocument>> {
  const result = await this.service.findAll(query);
  return { success: true, ...result };
}
```
- `{ success: true, ...result }` — `result` có dạng `{ data, meta }`, spread ra để **gộp phẳng** thành `{ success: true, data: [...], meta: {...} }` thay vì lồng `{ success: true, result: { data, meta } }` — giữ format response nhất quán, dễ dùng ở frontend.

```typescript
@Post()
@WriteAuthorization
async create(@Body(createPipe) dto: TCreateDto): Promise<CrudResponse<TDocument>> {
  return { success: true, data: await this.service.create(dto) };
}
```
- `@Body(createPipe)` — Pipe được truyền **trực tiếp làm tham số thứ 2** của decorator `@Body()`, nghĩa là **chỉ áp dụng cho tham số này**, khác với `@UsePipes()` gắn ở method/class level (áp dụng cho toàn bộ tham số). Đây gọi là **Pipe cấp tham số (parameter-scoped pipe)**.
- `@WriteAuthorization` được đặt **phía trên** method — decorator gộp (`applyDecorators`) hoạt động y hệt việc gắn từng decorator con riêng lẻ (ở đây là `@Roles(...)`), NestJS đọc metadata này ở tầng Guard (`RolesGuard`) để biết method này yêu cầu role gì.

---

## 11. Luồng đầy đủ: `PATCH /api/canteen/categories/:id`

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. Client: PATCH /api/canteen/categories/64f...  Body: { name: "Trà" } │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. RolesGuard → đọc @Roles(ADMIN, MANAGER) từ @WriteAuthorization  │
│    → user không đúng role → 403, dừng                             │
└──────────────────────────────────────────────────────────────────┘
                              │ role hợp lệ
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. @Param('id', idPipe) → ParseObjectIdPipe kiểm tra '64f...' có   │
│    đúng định dạng MongoDB ObjectId (24 ký tự hex) không            │
│    → sai format → 400, dừng                                       │
└──────────────────────────────────────────────────────────────────┘
                              │ id hợp lệ
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 4. @Body(updatePipe) → DtoValidationPipe validate theo             │
│    UpdateCategoryDto (thường mọi field optional vì là PATCH)       │
│    → field lạ không khai báo trong DTO (forbidNonWhitelisted:true) │
│      → 400, dừng                                                  │
└──────────────────────────────────────────────────────────────────┘
                              │ dto hợp lệ
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 5. CrudController.update(id, dto)                                  │
│    → this.service.update(id, dto)                                  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 6. BaseCrudService.update(id, dto):                                │
│    a) findOne(id) → tìm document hiện tại, không có → 404, dừng    │
│    b) prepareUpdate(dto, current) → [Category override] trim name  │
│    c) check Object.keys(data).length === 0 → nếu dto rỗng → 400    │
│    d) ensureUnique(data, id) → check 'name' trùng, LOẠI TRỪ chính  │
│       document đang sửa (excludeId = id)                           │
│       → trùng với category khác → 409, dừng                        │
│    e) beforeUpdate(current, data) → hook, Category không override  │
│    f) current.set(data) → merge field mới vào document cũ          │
│    g) current.save() → ghi xuống MongoDB                           │
│       → lỗi duplicate key bất ngờ (race condition hiếm) → 409       │
│    h) afterUpdate(saved) → hook, Category không override           │
│    i) return saved                                                 │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 7. Trả về: { success: true, data: { _id, name: "Trà", ... } }      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 12. Response trả về lỗi được xử lý ở đâu? — Exception Filter

Toàn bộ các exception ném ra trong luồng trên (`NotFoundException`, `ConflictException`, `BadRequestException`, lỗi từ Guard/Pipe...) đều là **HttpException** có sẵn của NestJS. NestJS có **Exception Filter mặc định toàn cục** tự động bắt các exception này và format thành response chuẩn:

```json
{
  "statusCode": 409,
  "message": "Danh mục có name 'Đồ uống' đã tồn tại",
  "error": "Conflict"
}
```

- `Service`/`Controller` trong base CRUD này **chỉ cần `throw`**, không cần tự viết `try/catch` để format response lỗi — đây chính là điểm hay của kiến trúc NestJS: tách biệt hoàn toàn **logic nghiệp vụ** (ném lỗi đúng loại) khỏi **logic trình bày lỗi** (format JSON trả về client), tuân theo nguyên tắc **Separation of Concerns**.
- Nếu project có custom `HttpExceptionFilter` riêng (thường thấy dùng `@UseFilters()` toàn cục trong `main.ts`), nó sẽ can thiệp vào **bước cuối cùng** trước khi trả response — không ảnh hưởng đến logic nào đã trình bày ở các sơ đồ trên, chỉ đổi hình thức JSON trả về.

---

## 13. Tổng kết mối quan hệ giữa các khái niệm

```
Generic (<T>)              →  cho phép class/hàm dùng chung nhiều kiểu resource
        │
        ▼
Abstract class              →  khuôn mẫu, ép buộc phải kế thừa (BaseCrudService/Controller)
        │
        ▼
Template Method + Hook      →  khung xử lý cố định, "cắm" logic riêng qua override hook
        │
        ▼
Interface / Utility Types   →  định nghĩa hợp đồng dữ liệu (options, response...) type-safe
        │
        ▼
Factory function            →  sinh động 1 class Controller cụ thể lúc runtime (do generic
                                bị xóa lúc runtime — "type erasure")
        │
        ▼
Decorator + DI + Pipe/Guard →  NestJS gắn metadata, tự động inject, validate, phân quyền
                                dựa trên metadata đó trước khi chạy tới Controller/Service
        │
        ▼
Exception → Exception Filter →  tách biệt logic ném lỗi (Service) khỏi logic format lỗi
                                 (tầng framework), trả JSON chuẩn cho client
```

Toàn bộ base CRUD này là ví dụ thực tế kết hợp **OOP (kế thừa, đóng gói, trừu tượng hóa)** + **Generic Programming** + **Design Pattern (Template Method, Factory)** + **Framework-level features của NestJS (DI, Decorator, Pipe, Guard)** để đạt được mục tiêu: **viết 1 lần, dùng lại nhiều lần, vẫn giữ type-safety và dễ mở rộng riêng cho từng module**.
