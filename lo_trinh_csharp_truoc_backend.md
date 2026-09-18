# Lộ trình học C# từ nền tảng đến trước Backend

> Mục tiêu: học **chậm, chắc, hiểu bản chất**, không học thuộc cú pháp.
>
> Nguyên tắc: **chỉ chuyển sang phần tiếp theo khi tự giải thích được phần hiện tại bằng lời của mình và tự code lại mà không nhìn mẫu**.

---

## Cách học mỗi kiến thức

Với **mỗi chủ đề**, học theo đúng 5 bước:

1. **Thuật ngữ là gì?**
2. **Nó giải quyết vấn đề gì?**
3. **Cú pháp viết thế nào?**
4. **Tự code ví dụ nhỏ.**
5. **Làm bài tập không nhìn đáp án.**

Không cần học nhanh. Một phần mất 1–3 ngày cũng hoàn toàn bình thường.

---

# Giai đoạn 0 — Làm quen môi trường C#

## 0.1. Project C# là gì?

Cần hiểu:

- `.NET` là gì.
- `.NET SDK` là gì.
- C# là ngôn ngữ gì.
- `dotnet new console` làm gì.
- `dotnet run` làm gì.
- File `.csproj` là gì.
- File `Program.cs` là gì.
- Thư mục `bin/` và `obj/` là gì.

### Phải tự làm được

```bash
dotnet new console -n Hello
cd Hello
dotnet run
```

### Qua phần này khi

Bạn giải thích được:

> Tôi viết code C# ở đâu, ai compile nó, và `dotnet run` đang làm gì?

---

# Giai đoạn 1 — C# cơ bản

Nếu bạn đã có base lập trình thì phần này học nhanh, nhưng vẫn nên kiểm tra lại.

## 1.1. Biến

Thuật ngữ:

- **Variable** = biến, nơi giữ một giá trị.
- **Data type** = kiểu dữ liệu.

Học:

```csharp
int age = 20;
double score = 8.5;
string name = "An";
bool isStudent = true;
char grade = 'A';
```

Cần hiểu:

- khai báo biến;
- gán giá trị;
- thay đổi giá trị;
- khác nhau giữa `int`, `double`, `string`, `bool`, `char`;
- `var` là gì.

---

## 1.2. Toán tử

Học:

```text
+  -  *  /  %
== != > < >= <=
&& || !
++ --
```

Phải hiểu khác nhau giữa:

```csharp
x = 10;
x == 10;
```

---

## 1.3. Điều kiện

Học:

```csharp
if
else if
else
switch
```

Ví dụ:

```csharp
if (age >= 18)
{
    Console.WriteLine("Đủ tuổi");
}
else
{
    Console.WriteLine("Chưa đủ tuổi");
}
```

---

## 1.4. Vòng lặp

Học:

```csharp
for
while
do while
foreach
```

Sau đó học:

```csharp
break;
continue;
```

Phải hiểu vòng lặp nào phù hợp khi nào.

---

## 1.5. Hàm / Method

Thuật ngữ:

- **Method** = một khối code có tên, thực hiện một công việc.
- **Parameter** = biến nhận dữ liệu đầu vào của hàm.
- **Argument** = giá trị thật truyền vào khi gọi hàm.
- **Return value** = giá trị hàm trả về.

Ví dụ:

```csharp
int Add(int a, int b)
{
    return a + b;
}

int result = Add(10, 20);
```

Phải phân biệt được:

```csharp
void
```

với:

```csharp
int
string
bool
```

làm kiểu trả về.

---

# Giai đoạn 2 — Kiểu dữ liệu và bộ nhớ

Phần này rất đáng học kỹ vì nó giúp hiểu C# sâu hơn.

## 2.1. Value type

Ví dụ:

```csharp
int
double
bool
char
struct
```

Hiểu khái niệm:

> Biến chứa trực tiếp giá trị.

---

## 2.2. Reference type

Ví dụ:

```csharp
string
class
array
List<T>
```

Hiểu khái niệm:

> Biến thường giữ tham chiếu tới object.

Ví dụ nên thử:

```csharp
Person a = new Person();
Person b = a;
```

Sau đó sửa `b` và quan sát `a`.

---

## 2.3. `null`

Học:

```csharp
string? name = null;
```

Thuật ngữ:

- **null** = không trỏ tới / không có giá trị object.
- **nullable** = kiểu cho phép nhận `null`.

Học tiếp:

```csharp
?. 
??
!
```

Nhưng chỉ cần hiểu từ từ, không học thuộc.

---

# Giai đoạn 3 — Class và Object

Đây là cửa vào OOP.

## 3.1. Class

Thuật ngữ:

- **Class** = bản thiết kế cho object.
- **Object** = một thực thể được tạo ra từ class.
- **Instance** = một object cụ thể của class.

Ví dụ:

```csharp
class Car
{
    public string Name { get; set; } = "";
    public int Speed { get; set; }
}
```

---

## 3.2. Tạo object

```csharp
Car car = new Car();
```

Phải hiểu từng phần:

```text
Car       car       =       new Car();
kiểu      biến              tạo object
```

---

## 3.3. Field và Property

### Field

```csharp
public string name = "";
```

### Property

```csharp
public string Name { get; set; } = "";
```

Thuật ngữ:

- **Field** = biến nằm trong class.
- **Property** = cách C# cung cấp quyền đọc/ghi dữ liệu của object qua `get` và `set`.

Chưa cần học property nâng cao ngay.

---

# Giai đoạn 4 — Constructor

Thuật ngữ:

- **Constructor** = hàm đặc biệt tự chạy khi object được tạo bằng `new`.

Ví dụ:

```csharp
class Car
{
    public string Name { get; set; }
    public int Speed { get; set; }

    public Car(string name, int speed)
    {
        Name = name;
        Speed = speed;
    }
}
```

Dùng:

```csharp
Car car = new Car("BMW", 200);
```

Phải hiểu:

- constructor chạy lúc nào;
- tại sao constructor không có kiểu trả về;
- constructor mặc định;
- constructor có tham số;
- constructor overloading;
- `this` là gì.

### Qua phần này khi

Bạn tự giải thích được vì sao:

```csharp
new Car();
```

có thể lỗi sau khi bạn tự tạo:

```csharp
public Car(string name)
```

---

# Giai đoạn 5 — Encapsulation

Thuật ngữ:

- **Encapsulation** = tính đóng gói.
- Ý tưởng chính: object tự kiểm soát dữ liệu của chính nó, không để code bên ngoài sửa tùy tiện.

Học:

```csharp
public
private
protected
```

Ví dụ:

```csharp
class BankAccount
{
    private double balance;

    public void Deposit(double amount)
    {
        if (amount > 0)
        {
            balance += amount;
        }
    }

    public double GetBalance()
    {
        return balance;
    }
}
```

Phải hiểu tại sao không nên:

```csharp
public double Balance;
```

rồi để nơi khác viết:

```csharp
account.Balance = -999999;
```

---

# Giai đoạn 6 — Property nâng cao

Học kỹ:

```csharp
get;
set;
```

Sau đó:

```csharp
private set;
```

Ví dụ:

```csharp
public double Balance { get; private set; }
```

Rồi học:

```csharp
init;
```

và khi nào dùng.

Đây là phần nối trực tiếp với Encapsulation.

---

# Giai đoạn 7 — Inheritance

Thuật ngữ:

- **Inheritance** = kế thừa.
- **Base class / Parent class** = class cha.
- **Derived class / Child class** = class con.

Ví dụ:

```csharp
class Animal
{
    public void Eat()
    {
        Console.WriteLine("Eating");
    }
}

class Dog : Animal
{
}
```

Dùng:

```csharp
Dog dog = new Dog();
dog.Eat();
```

Cần học:

- class con nhận được gì từ class cha;
- constructor cha/con;
- từ khóa `base`;
- `protected`.

Quan trọng:

> Kế thừa không phải để "tái sử dụng code bằng mọi giá". Phải có quan hệ hợp lý kiểu **Dog is an Animal**.

---

# Giai đoạn 8 — Polymorphism

Thuật ngữ:

- **Polymorphism** = đa hình.
- Một kiểu chung có thể đại diện cho nhiều implementation khác nhau.

Học:

```csharp
virtual
override
```

Ví dụ:

```csharp
class Animal
{
    public virtual void Speak()
    {
        Console.WriteLine("Animal sound");
    }
}

class Dog : Animal
{
    public override void Speak()
    {
        Console.WriteLine("Woof");
    }
}
```

Sau đó:

```csharp
Animal animal = new Dog();
animal.Speak();
```

Phải hiểu tại sao kết quả là:

```text
Woof
```

Đây là nền tảng cực kỳ quan trọng để hiểu `abstract`, `interface`, DI sau này.

---

# Giai đoạn 9 — Abstract class

Thuật ngữ:

- **abstract** = trừu tượng.
- Abstract class dùng làm một khuôn chung và có thể yêu cầu class con phải triển khai một số hành vi.

Ví dụ:

```csharp
abstract class Animal
{
    public abstract void Speak();
}
```

Class con:

```csharp
class Dog : Animal
{
    public override void Speak()
    {
        Console.WriteLine("Woof");
    }
}
```

Phải hiểu:

- tại sao không `new Animal()`;
- abstract method là gì;
- abstract class vẫn có thể chứa code bình thường;
- khi nào abstract hợp lý.

Không học thuộc câu "abstract là ẩn chi tiết". Phải hiểu bằng ví dụ thực tế.

---

# Giai đoạn 10 — Interface

Thuật ngữ:

- **Interface** = hợp đồng hành vi.
- Nó mô tả một object **có thể làm gì**, mà code sử dụng không cần biết object thực hiện bên trong thế nào.

Ví dụ:

```csharp
interface ILogger
{
    void Log(string message);
}
```

Implementation:

```csharp
class ConsoleLogger : ILogger
{
    public void Log(string message)
    {
        Console.WriteLine(message);
    }
}
```

Học kỹ:

- interface khác class thế nào;
- interface khác abstract class thế nào;
- một class có thể implement nhiều interface;
- vì sao interface giúp giảm phụ thuộc giữa các class.

Đây là phần phải hiểu thật chắc trước DI.

---

# Giai đoạn 11 — Composition

Thuật ngữ:

- **Composition** = một object chứa / sử dụng object khác để hoàn thành công việc.

Ví dụ:

```csharp
class Engine
{
    public void Start()
    {
        Console.WriteLine("Engine started");
    }
}

class Car
{
    private Engine engine = new Engine();

    public void Start()
    {
        engine.Start();
    }
}
```

Quan hệ:

```text
Car HAS AN Engine
```

Khác với inheritance:

```text
Dog IS AN Animal
```

Phải hiểu:

```text
IS-A  -> thường liên quan inheritance
HAS-A -> thường liên quan composition
```

Composition cực kỳ quan trọng cho thiết kế backend.

---

# Giai đoạn 12 — Dependency

Thuật ngữ:

- **Dependency** = một object/class mà class khác cần sử dụng để làm việc.

Ví dụ:

```csharp
class UserService
{
    private EmailService emailService = new EmailService();
}
```

Ở đây:

```text
EmailService là dependency của UserService.
```

Chưa nói DI vội.

Trước tiên phải hiểu chính xác **dependency là gì**.

---

# Giai đoạn 13 — Dependency Injection (DI)

Chỉ học sau khi đã hiểu:

- class/object;
- constructor;
- interface;
- polymorphism;
- composition;
- dependency.

Ví dụ ban đầu:

```csharp
class UserService
{
    private EmailService emailService = new EmailService();
}
```

`UserService` tự tạo dependency.

Chuyển sang:

```csharp
class UserService
{
    private readonly EmailService emailService;

    public UserService(EmailService emailService)
    {
        this.emailService = emailService;
    }
}
```

Dependency được truyền từ ngoài vào.

Đó là ý tưởng gốc của:

> **Dependency Injection = truyền dependency từ bên ngoài vào thay vì class tự tạo dependency.**

Sau đó mới học:

```csharp
ILogger
IUserRepository
IEmailService
```

và inject interface.

Cuối cùng mới tìm hiểu DI Container của ASP.NET Core.

---

# Giai đoạn 14 — Collections

Backend xử lý dữ liệu liên tục nên phần này rất quan trọng.

## 14.1. Array

```csharp
int[] numbers = { 1, 2, 3 };
```

## 14.2. List

```csharp
List<string> names = new List<string>();
```

hoặc cú pháp mới hơn:

```csharp
List<string> names = [];
```

Học:

```csharp
Add
Remove
Contains
Count
```

## 14.3. Dictionary

```csharp
Dictionary<string, int>
```

Hiểu:

```text
key -> value
```

## 14.4. HashSet

Biết nó dùng cho tập hợp không muốn phần tử trùng nhau.

Không cần thuộc toàn bộ API.

---

# Giai đoạn 15 — Generic

Thuật ngữ:

- **Generic** = viết code làm việc với nhiều kiểu dữ liệu mà vẫn giữ kiểm tra kiểu của compiler.

Ví dụ:

```csharp
List<int>
List<string>
```

`List<T>`:

```text
T = kiểu dữ liệu được truyền vào.
```

Tự viết ví dụ:

```csharp
class Box<T>
{
    public T Value { get; set; }
}
```

Dùng:

```csharp
Box<int> box1 = new Box<int>();
Box<string> box2 = new Box<string>();
```

Generic xuất hiện khắp .NET backend.

---

# Giai đoạn 16 — Exception Handling

Thuật ngữ:

- **Exception** = lỗi xảy ra trong lúc chương trình đang chạy.
- **Exception handling** = xử lý exception.

Học:

```csharp
try
catch
finally
throw
```

Ví dụ:

```csharp
try
{
    int number = int.Parse("abc");
}
catch (FormatException ex)
{
    Console.WriteLine(ex.Message);
}
```

Phải hiểu:

- lỗi compile khác exception runtime thế nào;
- không được `catch` mọi thứ rồi bỏ qua;
- khi nào nên `throw`.

---

# Giai đoạn 17 — Enum

Thuật ngữ:

- **enum** = tập hợp các giá trị có tên cố định.

Ví dụ:

```csharp
enum UserRole
{
    User,
    Admin,
    Moderator
}
```

Dùng:

```csharp
UserRole role = UserRole.Admin;
```

Rất hữu ích với trạng thái, role, loại dữ liệu cố định.

---

# Giai đoạn 18 — Record

Sau khi class đã chắc, học `record`.

Ví dụ:

```csharp
public record UserDto(string Name, string Email);
```

Thuật ngữ:

- **record** = kiểu dữ liệu C# rất phù hợp cho dữ liệu thiên về giá trị, đặc biệt DTO.

Không cần học record quá sớm.

---

# Giai đoạn 19 — LINQ

Đây là kiến thức C# cực kỳ quan trọng trước backend.

Thuật ngữ:

- **LINQ** = Language Integrated Query.
- Cho phép truy vấn / biến đổi collection bằng C#.

Bắt đầu với:

```csharp
Where
Select
First
FirstOrDefault
Any
All
OrderBy
OrderByDescending
Count
```

Ví dụ:

```csharp
List<int> numbers = [1, 2, 3, 4, 5];

var result = numbers
    .Where(x => x > 2)
    .ToList();
```

Chưa hiểu `x => x > 2` thì chuyển sang phần Lambda ngay bên dưới.

---

# Giai đoạn 20 — Lambda Expression

Thuật ngữ:

- **Lambda expression** = cách viết một hàm nhỏ, ngắn gọn, thường truyền vào hàm khác.

Ví dụ:

```csharp
x => x > 10
```

Có thể hiểu gần giống:

```csharp
bool IsGreaterThanTen(int x)
{
    return x > 10;
}
```

Lambda và LINQ phải luyện nhiều bằng collection nhỏ.

---

# Giai đoạn 21 — Delegate cơ bản

Thuật ngữ:

- **Delegate** = một kiểu có thể giữ tham chiếu tới method.

Không cần đào quá sâu trước backend, nhưng phải biết nó liên quan tới:

- lambda;
- callback;
- event;
- `Func<>`;
- `Action<>`.

Ví dụ:

```csharp
Func<int, int, int> add = (a, b) => a + b;
```

---

# Giai đoạn 22 — Async / Await

Cực kỳ quan trọng trước backend.

Thuật ngữ:

- **Synchronous** = chạy đồng bộ.
- **Asynchronous** = bất đồng bộ.
- **Task** = đại diện cho một công việc có thể hoàn thành trong tương lai.
- `async` = method có logic bất đồng bộ.
- `await` = chờ Task mà không block luồng theo cách đồng bộ thông thường.

Bắt đầu:

```csharp
async Task<string> GetDataAsync()
{
    await Task.Delay(1000);
    return "Done";
}
```

Học thật kỹ:

```csharp
Task
Task<T>
async
await
```

Không cần đi sâu Thread ngay lúc đầu.

Phải hiểu vì sao backend dùng async rất nhiều:

```text
database
HTTP request
file I/O
Redis
message queue
```

đều có thời gian chờ I/O.

---

# Giai đoạn 23 — File và JSON

Backend làm việc với dữ liệu nên cần biết cơ bản.

## File

```csharp
File.ReadAllText(...)
File.WriteAllText(...)
```

## JSON

Học:

```csharp
System.Text.Json
```

Ví dụ:

```csharp
string json = JsonSerializer.Serialize(user);
```

và deserialize:

```csharp
User? user = JsonSerializer.Deserialize<User>(json);
```

Thuật ngữ:

- **Serialization** = biến object thành dạng dữ liệu truyền/lưu được.
- **Deserialization** = biến dữ liệu đó trở lại object.

---

# Giai đoạn 24 — Namespace

Thuật ngữ:

- **Namespace** = cách tổ chức và phân nhóm class để tránh trùng tên.

Ví dụ:

```csharp
namespace MyApp.Services;
```

Học:

```csharp
using MyApp.Services;
```

Đây là lúc bắt đầu chia code thành nhiều file.

---

# Giai đoạn 25 — Access Modifier

Ôn và mở rộng:

```csharp
public
private
protected
internal
```

Không học thuộc định nghĩa.

Hãy tạo project nhiều class và thử xem code nào truy cập được code nào.

---

# Giai đoạn 26 — `static`

Thuật ngữ:

- **Instance member** = thuộc về từng object.
- **Static member** = thuộc về class, không thuộc riêng object nào.

Ví dụ:

```csharp
class MathHelper
{
    public static int Add(int a, int b)
    {
        return a + b;
    }
}
```

Dùng:

```csharp
MathHelper.Add(1, 2);
```

không cần:

```csharp
new MathHelper();
```

Phải hiểu static khác object bình thường ở đâu.

---

# Giai đoạn 27 — `const` và `readonly`

Học khác nhau giữa:

```csharp
const
readonly
```

Đặc biệt:

```csharp
private readonly ILogger logger;
```

sẽ xuất hiện rất nhiều trong backend và DI.

---

# Giai đoạn 28 — SOLID ở mức cơ bản

Chỉ học khi OOP + interface + DI đã tương đối chắc.

Không học SOLID bằng định nghĩa suông.

Học từng nguyên tắc qua code xấu → refactor code tốt hơn.

## S — Single Responsibility Principle

Một class nên có một trách nhiệm chính rõ ràng.

## O — Open/Closed Principle

Thiết kế để có thể mở rộng hành vi mà hạn chế sửa code ổn định hiện có.

## L — Liskov Substitution Principle

Object class con phải dùng được ở nơi mong đợi class cha mà không phá ý nghĩa hợp đồng.

## I — Interface Segregation Principle

Không ép class phụ thuộc vào những method nó không cần.

## D — Dependency Inversion Principle

Code cấp cao nên phụ thuộc vào abstraction thay vì phụ thuộc cứng vào implementation cụ thể.

Đây là chỗ DI và Interface bắt đầu nối lại với nhau.

---

# Giai đoạn 29 — Testing cơ bản

Trước backend nên biết test là gì.

Thuật ngữ:

- **Unit test** = test một đơn vị logic nhỏ.
- **Arrange** = chuẩn bị.
- **Act** = thực hiện.
- **Assert** = kiểm tra kết quả.

Có thể học `xUnit`.

Ví dụ tư duy:

```text
Arrange:
tạo Calculator

Act:
gọi Add(2, 3)

Assert:
kết quả phải bằng 5
```

Không cần test nâng cao ngay.

---

# Giai đoạn 30 — Git cơ bản

Trước khi làm backend project nên dùng được:

```bash
git init
git status
git add
git commit
git branch
git switch
git merge
git pull
git push
```

Hiểu:

- repository;
- commit;
- branch;
- merge;
- remote.

---

# Giai đoạn 31 — Kiến thức Web trước Backend

Đến đây **chưa vào ASP.NET Core ngay**.

Hãy hiểu Web trước.

## 31.1. Client và Server

Hiểu:

```text
Client -> Request -> Server
Client <- Response <- Server
```

## 31.2. HTTP

Học:

```text
GET
POST
PUT
PATCH
DELETE
```

Học:

- URL;
- route/path;
- query parameter;
- header;
- body;
- status code.

Các status code cơ bản:

```text
200
201
204
400
401
403
404
409
500
```

## 31.3. REST API

Hiểu resource.

Ví dụ:

```text
GET    /users
GET    /users/10
POST   /users
PUT    /users/10
DELETE /users/10
```

## 31.4. JSON

Ví dụ:

```json
{
  "id": 1,
  "name": "An",
  "email": "an@gmail.com"
}
```

Phải hiểu request body và response body.

---

# Giai đoạn 32 — Database cơ bản trước Backend

Không cần thành DBA nhưng phải biết:

- database là gì;
- table;
- row;
- column;
- primary key;
- foreign key;
- relation.

Học SQL cơ bản:

```sql
SELECT
INSERT
UPDATE
DELETE
WHERE
ORDER BY
JOIN
GROUP BY
```

Sau đó hiểu:

```text
1 - 1
1 - N
N - N
```

Không nên vào Entity Framework mà chưa hiểu SQL cơ bản.

---

# Sau khi hoàn thành: bắt đầu Backend ASP.NET Core

Khi phần trên đã chắc, mới đi tiếp:

```text
ASP.NET Core
    ↓
Web API
    ↓
Controller / Minimal API
    ↓
Routing
    ↓
DTO
    ↓
Service
    ↓
Dependency Injection Container
    ↓
Entity Framework Core
    ↓
Database
    ↓
Validation
    ↓
Authentication
    ↓
Authorization
    ↓
JWT
    ↓
Logging
    ↓
Exception handling
    ↓
Caching / Redis
    ↓
Testing
    ↓
Docker
    ↓
Deploy
```

Lúc này các khái niệm như:

```csharp
public UserService(IUserRepository repository)
{
    _repository = repository;
}
```

sẽ không còn là code "ma thuật", vì bạn đã hiểu:

- constructor;
- dependency;
- interface;
- abstraction;
- polymorphism;
- composition;
- DI.

---

# Thứ tự học rút gọn

Bạn có thể đánh dấu từng mục:

- [ ] 0. Môi trường .NET / project C#
- [ ] 1. Biến, điều kiện, vòng lặp, method
- [ ] 2. Value type / Reference type / null
- [ ] 3. Class / Object
- [ ] 4. Constructor / `this`
- [ ] 5. Encapsulation
- [ ] 6. Property
- [ ] 7. Inheritance
- [ ] 8. Polymorphism
- [ ] 9. Abstract class
- [ ] 10. Interface
- [ ] 11. Composition
- [ ] 12. Dependency
- [ ] 13. Dependency Injection
- [ ] 14. Collections
- [ ] 15. Generic
- [ ] 16. Exception
- [ ] 17. Enum
- [ ] 18. Record
- [ ] 19. LINQ
- [ ] 20. Lambda
- [ ] 21. Delegate cơ bản
- [ ] 22. Async / Await
- [ ] 23. File / JSON
- [ ] 24. Namespace
- [ ] 25. Access Modifier
- [ ] 26. Static
- [ ] 27. Const / Readonly
- [ ] 28. SOLID cơ bản
- [ ] 29. Unit Test
- [ ] 30. Git
- [ ] 31. HTTP / REST
- [ ] 32. SQL / Database
- [ ] Bắt đầu ASP.NET Core Backend

---

# Quy tắc để biết mình "đã hiểu"

Không đánh dấu một phần chỉ vì:

> "Đọc code thấy hiểu."

Chỉ đánh dấu khi bạn làm được cả 4:

1. Giải thích được thuật ngữ bằng lời đơn giản.
2. Tự viết ví dụ mà không nhìn tài liệu.
3. Giải thích được **tại sao cần nó**.
4. Phân biệt được nó với khái niệm gần giống.

Ví dụ với Constructor, phải trả lời được:

- Constructor là gì?
- Nó chạy khi nào?
- Tại sao dùng constructor thay vì gán property sau?
- Constructor mặc định là gì?
- Vì sao viết constructor có tham số xong thì `new Car()` có thể lỗi?
- `this` là gì?

Nếu trả lời được, mới chuyển bài.

---

# Cách chúng ta có thể học theo roadmap này

Mỗi buổi chỉ lấy **1 mục nhỏ**.

Ví dụ:

```text
Bài 1: Class và Object
Bài 2: Field và Property
Bài 3: Constructor
Bài 4: this
Bài 5: Encapsulation
...
```

Mỗi bài nên theo format:

```text
1. Thuật ngữ
2. Vấn đề thực tế
3. Code đơn giản
4. Mổ từng dòng code
5. Ví dụ sai
6. Ví dụ đúng
7. Bài tập
8. Câu hỏi kiểm tra hiểu
```

Mục tiêu không phải "học hết C#".

Mục tiêu là:

> **Khi vào ASP.NET Core, nhìn code backend và hiểu vì sao nó được thiết kế như vậy.**
