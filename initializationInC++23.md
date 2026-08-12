# تهيئة المتغيرات في C++23

> محاضرة تقنية احترافية ومناسبة لجميع المستويات، من أول برنامج C++ إلى سياسات التهيئة في المشاريع الإنتاجية.

![C++](https://img.shields.io/badge/C%2B%2B-23-00599C?style=flat-square&logo=c%2B%2B)
![Level](https://img.shields.io/badge/Level-Beginner%20to%20Advanced-2ea44f?style=flat-square)
![Topic](https://img.shields.io/badge/Topic-Initialization-f05032?style=flat-square)

---

## أهداف المحاضرة

بعد دراسة هذه المحاضرة ستكون قادرًا على:

- التفريق بين **التصريح** و**التعريف** و**التهيئة** و**الإسناد**.
- فهم صيغ التهيئة الأساسية في C++23 وما الذي يفعله المترجم في كل صيغة.
- منع القيم غير المحددة والتحويلات المضيقة وMost Vexing Parse.
- اختيار الصيغة المناسبة للأعداد والكائنات والحاويات و`auto` وبيانات الأصناف.
- كتابة سياسة تهيئة متسقة لفريق برمجي احترافي.
- مراجعة الكود واكتشاف الأخطاء المتعلقة بعمر المتغير وحالته الابتدائية.

---

## الفكرة المركزية

التهيئة ليست تفصيلًا شكليًا. إنها اللحظة التي يبدأ فيها عمر الكائن ويحصل فيها على حالته الأولى. كلما كانت هذه الحالة صحيحة وواضحة، قلت الحالات غير الصالحة التي يجب على بقية البرنامج التعامل معها.

> **القاعدة الأهم:** لا تنشئ متغيرًا قبل أن تملك قيمة صحيحة لتهيئته، وهيئ كل كائن عند إنشائه.

---

## 1. المصطلحات الأساسية

### التصريح Declaration

يُدخل اسمًا ونوعًا في البرنامج، وقد لا ينشئ كائنًا في بعض السياقات:

```cpp
extern int global_count; // تصريح، والتعريف موجود في مكان آخر.
```

### التعريف Definition

ينشئ الكيان أو يوفر تعريفه الكامل:

```cpp
int global_count{0}; // تعريف وتهيئة.
```

### التهيئة Initialization

تعطي الكائن حالته الأولى في لحظة إنشائه:

```cpp
int score{100};
std::string name{"Ada"};
```

### الإسناد Assignment

يغيّر قيمة كائن موجود مسبقًا:

```cpp
int score{100}; // تهيئة
score = 120;    // إسناد
```

هذا الفرق مهم جدًا مع الأصناف. التهيئة تستدعي constructor مناسبًا، بينما الإسناد يستدعي assignment operator على كائن حي بالفعل.

### مثال على العمل المكرر

```cpp
std::string message;          // إنشاء string فارغ.
message = "Build completed"; // إسناد لاحق.

// الأفضل عندما تكون القيمة معروفة:
std::string better{"Build completed"};
```

---

## 2. لماذا نهيئ كل متغير؟

بعض الكائنات المحلية من الأنواع الأساسية قد لا تحصل على قيمة مفيدة إذا تُركت دون initializer:

```cpp
void bad_example()
{
    int count;            // لا تفترض أن قيمته صفر.
    double average;       // لا تستخدمه قبل إعطائه قيمة.
    int* pointer;         // لا تفترض أنه nullptr.
}
```

الصيغة الآمنة:

```cpp
void good_example()
{
    int count{0};
    double average{0.0};
    int* pointer{nullptr};
}
```

لكن الصفر ليس دائمًا قيمة مجال صحيحة. الأفضل أحيانًا تأخير التصريح حتى تُحسب القيمة:

```cpp
const int count = read_count();
const double average = calculate_average();
```

لا تستخدم قيمة وهمية ثم تستبدلها لاحقًا إذا كان بالإمكان البناء مباشرة بالقيمة الصحيحة.

---

## 3. خريطة صيغ التهيئة

```cpp
T a;          // default-initialization
T b{};        // value/list initialization بحسب T والسياق
T c(value);   // direct-initialization
T d{value};   // direct-list-initialization
T e = value;  // copy-initialization
T f = {value}; // copy-list-initialization
```

لا تعني كلمة `copy` بالضرورة حدوث نسخ فعلي في الكود المولد. هي اسم لمجموعة قواعد لغوية واختيار constructors والتحويلات.

---

## 4. Default Initialization

```cpp
int number;            // قيمة غير صالحة للاستخدام قبل الكتابة إليها.
std::string text;      // يستدعي default constructor، فتكون string فارغة.
Widget widget;         // يستدعي default constructor إن كان متاحًا.
```

السلوك يعتمد على النوع والسياق. لذلك لا تطبق قاعدة ذهنية واحدة على `int` و`std::string`.

### سياسة المشروع

- لا تترك scalar محليًا دون قيمة.
- اسمح بـ default construction لكائن class عندما تكون حالته الافتراضية معرفة وصحيحة دلاليًا.
- إذا لم توجد حالة افتراضية صحيحة، فلا توفر default constructor.

```cpp
class Port
{
public:
    explicit Port(unsigned short value) : value_{value} {}

private:
    unsigned short value_;
};

// Port port; // غير مسموح، وهذا جيد إذا لم يوجد منفذ افتراضي صحيح.
Port port{8080};
```

---

## 5. Value Initialization وEmpty Braces

```cpp
int count{};          // صفر
bool enabled{};       // false
double ratio{};       // 0.0
int* pointer{};       // nullptr
std::string text{};   // string فارغة
```

تعد `{}` وسيلة مفيدة للحصول على حالة ابتدائية معروفة، لكنها لا تعني دائمًا "ضع أصفارًا في الذاكرة". القواعد تعتمد على نوع الكائن وconstructors الخاصة به.

```cpp
struct Metrics
{
    int requests;
    double latency;
};

Metrics metrics{}; // تهيئة عناصر الـ aggregate وفق قواعد التهيئة الفارغة.
```

### متى تكون `{}` مناسبة؟

- عندما تكون الحالة الصفرية أو الافتراضية صحيحة دلاليًا.
- عند تهيئة scalar إلى قيمة معروفة.
- عند إنشاء container فارغ.
- عند تهيئة aggregate بالكامل إلى defaults مناسبة.

### متى لا تكون كافية؟

إذا كانت الصفر قيمة غير صالحة أو تخفي عدم وجود البيانات:

```cpp
// غامض: هل 0 قيمة حقيقية أم "غير معروف"؟
int user_age{};

// أوضح إذا كان الغياب مسموحًا:
std::optional<int> user_age;
```

---

## 6. Direct Initialization بالأقواس الدائرية

```cpp
std::string stars(5, '*'); // "*****"
Widget widget(argument);
```

هذه الصيغة مهمة خصوصًا عندما تملك الأقواس المعقوفة معنى مختلفًا بسبب `std::initializer_list`.

### Most Vexing Parse

```cpp
Widget widget(); // تصريح عن دالة اسمها widget، وليس كائنًا.
```

الصحيح:

```cpp
Widget widget{};
```

---

## 7. List Initialization بالأقواس المعقوفة

```cpp
int count{42};
std::string name{"Grace"};
Widget widget{argument};
```

من أهم فوائدها أنها ترفض عددًا من التحويلات المضيقة:

```cpp
double value{3.8};

// int count{value}; // خطأ: narrowing conversion.
int count = static_cast<int>(value); // التحويل المقصود أصبح صريحًا.
```

### أمثلة Narrowing

```cpp
// int a{3.14};             // double إلى int
// unsigned char b{1000};   // القيمة لا تتسع
// float c{1.234567890123}; // قد يرفض بسبب التضييق
```

### الاستثناء المهم: `initializer_list`

الأقواس المعقوفة قد تفضل constructor يستقبل `std::initializer_list`:

```cpp
std::vector<int> first(3, 9); // ثلاثة عناصر: 9, 9, 9
std::vector<int> second{3, 9}; // عنصران: 3, 9
```

إذن عبارة "استخدم الأقواس المعقوفة دائمًا" ليست سياسة كافية. في الحاويات، اختر الصيغة التي تعبّر عن المعنى المطلوب.

---

## 8. Copy Initialization

```cpp
int count = 42;
std::string name = "Bjarne";
```

الصيغة مألوفة وواضحة، وتعمل جيدًا في حالات كثيرة. لكنها لا تأخذ constructors الموسومة `explicit` ضمن بعض سياقات copy-initialization:

```cpp
class Distance
{
public:
    explicit Distance(double meters) : meters_{meters} {}

private:
    double meters_;
};

Distance first{12.5};   // صحيح
Distance second(12.5);  // صحيح
// Distance third = 12.5; // غير صحيح لأن constructor explicit.
```

### لماذا `explicit`؟

لمنع التحويلات الضمنية التي قد تغير معنى الاستدعاء دون أن يلاحظ القارئ:

```cpp
void travel(Distance distance);

// travel(10.0);       // مرفوض
travel(Distance{10.0}); // النية واضحة
```

---

## 9. Copy-List Initialization

```cpp
int count = {42};
Widget widget = {argument};
```

تحصل على فحص narrowing الخاص بالقوائم، لكن إذا كان constructor المختار `explicit` فقد تُرفض الصيغة في copy-list-initialization. غالبًا تكون `T object{args};` أبسط وأوضح للكائنات.

---

## 10. Aggregate Initialization

الـ aggregate نوع بسيط يحقق شروطًا لغوية محددة، ويمكن تهيئة عناصره مباشرة:

```cpp
struct Point
{
    double x;
    double y;
};

Point origin{0.0, 0.0};
Point target{4.5, 8.0};
```

إذا نقصت العناصر، تُطبق قواعد التهيئة على البقية، لكن لا تعتمد على ذلك إذا كان إخفاء القيمة يضعف الوضوح:

```cpp
Point point{2.0}; // y تصبح 0.0، لكن Point{2.0, 0.0} أوضح غالبًا.
```

### Designated Initializers منذ C++20

```cpp
struct ServerConfig
{
    std::string host{"localhost"};
    int port{8080};
    bool tls{false};
};

ServerConfig config{
    .host = "api.example.com",
    .port = 443,
    .tls = true
};
```

في C++ يجب أن تتبع designators ترتيب التصريح، ولا تملك جميع مرونة الصيغة المقابلة في C. كما أنها مخصصة للـ aggregates.

### متى نستخدم Aggregate؟

- DTOs وبنى بيانات بسيطة بلا invariants معقدة.
- configuration واضح.
- قيم رياضية بسيطة.

إذا كان النوع يحتاج تحققًا أو يحافظ على invariant، استخدم constructor أو factory بدل كشف كل البيانات:

```cpp
class Percentage
{
public:
    [[nodiscard]] static std::optional<Percentage> create(int value)
    {
        if (value < 0 || value > 100) {
            return std::nullopt;
        }
        return Percentage{value};
    }

    [[nodiscard]] int value() const noexcept { return value_; }

private:
    explicit Percentage(int value) : value_{value} {}
    int value_;
};
```

---

## 11. تهيئة المصفوفات والحاويات

### المصفوفات القياسية

```cpp
int raw_values[4]{}; // جميع العناصر صفر.
std::array<int, 4> values{1, 2, 3, 4};
```

فضّل `std::array` للحجم الثابت و`std::vector` للحجم الديناميكي في معظم كود التطبيقات.

### `std::vector`

```cpp
std::vector<int> empty;
std::vector<int> three_zeros(3);
std::vector<int> four_sevens(4, 7);
std::vector<int> explicit_values{4, 7};
```

راجع دائمًا هل تريد **عدد عناصر** أم **قائمة قيم**.

### String

```cpp
std::string empty;
std::string repeated(5, 'x');
std::string text{"hello"};
```

---

## 12. `auto` والاستدلال النوعي

`auto` لا يعني أن C++ أصبحت dynamically typed. النوع يُستدل وقت الترجمة ولا يتغير بعد ذلك.

```cpp
auto count = 42;             // int
auto ratio = 0.5;            // double
auto name = std::string{"Ada"};
auto iterator = values.begin();
```

### أفضل استخدامات `auto`

- عندما يكون النوع ظاهرًا في الطرف الأيمن.
- مع iterators والأنواع الطويلة.
- لمنع تكرار النوع.
- عند ضرورة الاحتفاظ بنوع التعبير الدقيق.

```cpp
auto order = std::make_unique<Order>();
const auto result = calculate_result();
```

### متى نكتب النوع صراحة؟

- عندما يضيف النوع معنى مهمًا للقارئ.
- عندما نريد تحويلًا أو حدًا نوعيًا مقصودًا.
- عندما قد يؤدي الاستدلال إلى نوع مفاجئ.

```cpp
std::uint64_t file_size = read_size();
Seconds timeout{30};
```

### فخ `auto` مع الأقواس

```cpp
auto a = 5;   // int
auto b{5};    // int
```

لكن القواعد تختلف مع `auto x = { ... }`، التي قد تستدل `std::initializer_list`:

```cpp
auto values = {1, 2, 3}; // std::initializer_list<int>
```

لا تستخدم هذه الصيغة إذا لم يكن `initializer_list` هو المقصود.

### `decltype(auto)`

يحافظ على قواعد `decltype` بما فيها المراجع، ولذلك هو أداة متقدمة لا مرادفًا تجميليًا لـ `auto`:

```cpp
decltype(auto) access(std::vector<int>& values, std::size_t index)
{
    return (values[index]); // يعيد int& بسبب الأقواس وقواعد decltype.
}
```

استخدمه فقط عندما تريد الحفاظ الدقيق على value category والنوع المرجعي.

---

## 13. الثوابت: `const`, `constexpr`, و`constinit`

### `const`

يعني أن الكائن لا يتغير من خلال هذا الاسم بعد التهيئة:

```cpp
const int retry_count = read_retry_count();
```

قد تُحسب القيمة وقت التشغيل.

### `constexpr`

يطلب قابلية الاستخدام في constant expressions عندما تتحقق الشروط:

```cpp
constexpr int max_connections{128};
constexpr auto buffer_size = 4uz * 1024uz; // suffix لـ size_t في C++23.
```

### `constinit`

يُستخدم لمتغيرات ذات static أو thread storage duration لضمان static initialization، ولا يجعل المتغير ثابتًا:

```cpp
constinit int startup_state{0};
startup_state = 1; // مسموح، فهو ليس const.
```

### سياسة المشروع

- استخدم `constexpr` للقيم الثابتة وقت الترجمة.
- استخدم `const` عندما لا تتغير القيمة بعد بنائها، حتى لو حُسبت وقت التشغيل.
- استخدم `constinit` عند الحاجة إلى ضمان تهيئة ثابتة لمتغير global/static قابل للتغيير.
- تجنب mutable global state قدر الإمكان.

---

## 14. تهيئة أعضاء الأصناف

### In-Class Member Initializers

```cpp
class ConnectionOptions
{
public:
    ConnectionOptions() = default;

private:
    int timeout_seconds_{30};
    bool keep_alive_{true};
    std::string host_{"localhost"};
};
```

تعطي default موحدًا لكل constructors ما لم يستبدله constructor معين.

### Member Initializer List

```cpp
class User
{
public:
    User(std::string name, int age)
        : name_{std::move(name)}, age_{age}
    {
    }

private:
    std::string name_;
    int age_;
};
```

لا تنشئ العضو ثم تسند إليه داخل جسم constructor:

```cpp
class BadUser
{
public:
    BadUser(std::string name)
    {
        name_ = std::move(name); // name_ بُني أولًا ثم أُسند إليه.
    }

private:
    std::string name_;
};
```

### ترتيب التهيئة

الأعضاء تُهيأ حسب ترتيب تصريحها داخل class، لا حسب ترتيبها في member initializer list:

```cpp
class Range
{
public:
    Range(int first, int last)
        : first_{first}, last_{last}
    {
    }

private:
    int first_;
    int last_;
};
```

اجعل ترتيب القائمة مطابقًا لترتيب التصريح، حتى لا تضلل القارئ وتتجنب أخطار الاعتماد بين الأعضاء.

### Delegating Constructors

```cpp
class Buffer
{
public:
    Buffer() : Buffer{1024} {}

    explicit Buffer(std::size_t size)
        : data_(size)
    {
    }

private:
    std::vector<std::byte> data_;
};
```

استخدم constructor مركزيًا لتأسيس invariant بدل تكرار منطق التهيئة.

---

## 15. المراجع والمؤشرات و`nullptr`

### المراجع يجب أن تُهيأ

```cpp
int value{42};
int& reference{value};
const int& read_only{value};
```

المرجع لا يمكن أن يكون "فارغًا" في C++ الصحيح ولا يمكن إعادة ربطه بعد التهيئة.

### استخدم `nullptr`

```cpp
Widget* optional_widget{nullptr};
```

لا تستخدم `0` أو `NULL` في Modern C++ لأن `nullptr` له نوع مخصص ويعمل بصورة أفضل مع overload resolution.

### لا تستخدم raw owning pointer

```cpp
auto widget = std::make_unique<Widget>();
```

استخدم المؤشر الخام غالبًا للمراقبة غير المالكة، واستخدم RAII والـ smart pointers للملكية الديناميكية.

---

## 16. التهيئة من نتائج قد تفشل

لا تستخدم sentinel غامضًا إذا كان الفشل جزءًا طبيعيًا من العقد.

### قيمة اختيارية

```cpp
[[nodiscard]] std::optional<int> find_port(const Config& config);

if (const auto port = find_port(config)) {
    connect(*port);
}
```

### نتيجة أو خطأ في C++23

```cpp
#include <expected>
#include <string>

[[nodiscard]] std::expected<int, std::string>
parse_port(std::string_view text);

const auto result = parse_port("8080");
if (!result) {
    report_error(result.error());
    return;
}

const int port = *result;
```

بهذا لا يوجد متغير في حالة "نأمل أن تتم تهيئته لاحقًا".

---

## 17. Init Statements وتقليل النطاق

يمكن تهيئة متغير داخل `if` أو `switch` ليبقى نطاقه محدودًا:

```cpp
if (const auto user = repository.find(id); user.has_value()) {
    display(*user);
}
```

```cpp
switch (const auto status = run_job(); status) {
case Status::success:
    break;
case Status::failed:
    report_failure();
    break;
}
```

تقليل النطاق يمنع الاستخدام العرضي، ويجعل مكان التهيئة قريبًا من الاستخدام.

---

## 18. التهيئة في الحلقات

### Range-for

```cpp
for (const auto& item : items) {
    inspect(item);
}
```

اختر نوع متغير الحلقة وفق النية:

- `const auto&` للقراءة دون نسخ.
- `auto&` للتعديل.
- `auto` عندما تريد نسخة فعلية أو يكون العنصر صغيرًا وقابلًا للنسخ بكلفة بسيطة.
- `auto&&` في generic code أو عند التعامل الواعي مع proxy/reference categories.

### متغير العداد

```cpp
for (std::size_t index{0}; index < values.size(); ++index) {
    process(values[index]);
}
```

أو فضّل range-for عندما لا تحتاج إلى الفهرس.

---

## 19. الأنواع الصحيحة والـ Literal Suffixes

```cpp
const std::int64_t distance{1'000'000};
const auto timeout = 500ms; // بعد using namespace std::chrono_literals;
const std::size_t count{42};
```

في C++23 يوجد suffix للحرفيات من نوع `std::size_t`:

```cpp
constexpr auto page_count = 64uz;
```

استخدم digit separators لتحسين القراءة:

```cpp
constexpr int one_million{1'000'000};
constexpr std::uint32_t mask{0xFF00'00FFu};
```

لا تضف suffix عشوائيًا. اجعل النوع متوافقًا مع المجال والعمليات التي ستجرى عليه.

---

## 20. دقة الأعداد العشرية

```cpp
float a{0.5F};
double b{0.5};
long double c{0.5L};
```

لا تستخدم floating-point للأموال إذا كان التقريب الثنائي غير مقبول. مثّل أصغر وحدة صحيحة أو استخدم نوع decimal/money مصممًا للمجال:

```cpp
struct Money
{
    std::int64_t cents{};
};

const Money price{1'999}; // 19.99 بوحدة العملة المفترضة.
```

اكتب الوحدة في النوع أو الاسم، ولا تعتمد على تعليق قابل للتقادم.

---

## 21. Move Initialization

```cpp
std::string source{"large payload"};
std::string destination{std::move(source)};
```

بعد النقل يبقى `source` صالحًا لكن حالته غير محددة القيمة ضمن الضمانات الخاصة بنوعه، ويمكن عادة تدميره أو إسناد قيمة جديدة إليه. لا تفترض أنه فارغ إلا إذا ضمن النوع ذلك.

### تمرير بالقيمة ثم النقل إلى عضو

```cpp
class Message
{
public:
    explicit Message(std::string text)
        : text_{std::move(text)}
    {
    }

private:
    std::string text_;
};
```

هذا نمط واضح عندما تريد امتلاك نسخة من القيمة وتقبل النسخ من lvalue والنقل من rvalue. قِس الأداء في الأنواع والمسارات الحساسة بدل تعميمه بلا تحليل.

---

## 22. Structured Bindings

```cpp
const auto [iterator, inserted] = cache.try_emplace(key, value);

if (inserted) {
    notify_new_entry(iterator->first);
}
```

انتبه إلى النسخ مقابل المراجع:

```cpp
auto [key, value] = pair;        // نسخ أو نقل بحسب السياق.
auto& [key_ref, value_ref] = pair;       // مراجع قابلة للتعديل.
const auto& [key_view, value_view] = pair; // مراجع للقراءة.
```

---

## 23. Static Initialization Order

المتغيرات global ذات dynamic initialization في translation units مختلفة قد تعتمد على ترتيب غير مضمون، وهو ما يعرف بمشكلة Static Initialization Order Fiasco.

بدل globals المترابطة، استخدم function-local static عند الحاجة:

```cpp
Logger& logger()
{
    static Logger instance{/* configuration */};
    return instance;
}
```

تهيئة local static آمنة من سباق التهيئة منذ C++11، لكن الكائن المشترك نفسه يحتاج مزامنة إذا كانت عملياته غير آمنة خيطيًا.

الأفضل معماريًا في كثير من الحالات هو dependency injection والملكية الواضحة بدل الوصول العالمي.

---

## 24. أخطاء شائعة في المشاريع

### الخطأ 1: التصريح المبكر

```cpp
Result result;
// عشرات الأسطر...
result = calculate();
```

الأفضل:

```cpp
const Result result = calculate();
```

### الخطأ 2: صفر وهمي

```cpp
int selected_id{0}; // هل 0 صالح أم يعني لا يوجد؟
```

الأفضل:

```cpp
std::optional<UserId> selected_id;
```

### الخطأ 3: Uniform Initialization بلا فهم

```cpp
std::vector<int> values{10, 5}; // عنصران، وليس عشرة عناصر قيمتها 5.
```

### الخطأ 4: `auto` يخفي تحويلًا مطلوبًا

```cpp
auto total = unsigned_count - signed_offset; // راجع قواعد التحويل بعناية.
```

استخدم أنواع المجال أو تحويلًا صريحًا بعد التحقق من الحدود.

### الخطأ 5: عضو غير مهيأ

```cpp
class Job
{
private:
    int priority_; // قد يبقى بلا قيمة إذا لم تهيئه كل constructors.
};
```

الأفضل:

```cpp
class Job
{
private:
    int priority_{0};
};
```

أو constructor يطلب القيمة إذا لم يكن الصفر صالحًا.

### الخطأ 6: ترتيب قائمة الأعضاء مضلل

اكتب initializer list بنفس ترتيب تصريح الأعضاء.

### الخطأ 7: التحويلات C-style

```cpp
int count = (int)value;
```

الأفضل:

```cpp
const int count = static_cast<int>(value);
```

لكن افحص المجال أولًا إذا كان فقدان البيانات غير مقبول.

### الخطأ 8: Initializer List Lifetime

لا تحتفظ بمؤشر إلى عناصر قائمة مؤقتة بعد انتهاء عمر المصفوفة الكامنة. كن حذرًا عندما يخزن نوع ما `std::initializer_list` بدل نسخ عناصرها.

### الخطأ 9: الاعتماد على Default Constructor لحالة غير صالحة

إذا كان الكائن لا يستطيع أداء وعوده بعد default construction، اطلب البيانات في constructor أو استخدم factory يعيد `expected`.

### الخطأ 10: استخدام `memset` لتهيئة كائن C++

```cpp
// لا تفعل هذا لكائن غير trivial.
// std::memset(&object, 0, sizeof object);
```

استخدم constructors وmember initializers. البتات الصفرية ليست بالضرورة تمثيل قيمة صحيحة لكل نوع، و`memset` قد يخرّب invariants وبيانات داخلية.

---

## 25. سياسة احترافية مقترحة للمشروع

### القاعدة 1: هيئ كل كائن

```cpp
int retries{0};
Widget* observer{nullptr};
```

أو أجّل تعريفه حتى تتوفر القيمة الصحيحة.

### القاعدة 2: اجعل النطاق أصغر ما يمكن

عرّف المتغير قرب أول استخدام، داخل `if`, `for`, أو block مناسب.

### القاعدة 3: اجعل عدم التغيير هو الافتراضي

```cpp
const auto config = load_config();
```

أزل `const` فقط عندما توجد حاجة فعلية للتغيير.

### القاعدة 4: استخدم `constexpr` للقيم الثابتة وقت الترجمة

```cpp
constexpr std::size_t cache_capacity{256};
```

### القاعدة 5: استخدم `{}` افتراضيًا للقيم والكائنات البسيطة، مع استثناء واعٍ

```cpp
Point point{1.0, 2.0};
```

استخدم `()` عندما تكون semantics المطلوبة هي constructor عددي أو عندما يتعارض `{}` مع `initializer_list`:

```cpp
std::vector<int> scores(100, 0);
```

### القاعدة 6: امنع Narrowing

اجعل التحويل المقصود صريحًا، مع فحص الحدود عند الحاجة.

### القاعدة 7: لا توفر حالة افتراضية غير صالحة

صمم النوع بحيث يخرج من constructor صالحًا، أو يفشل الإنشاء بآلية واضحة.

### القاعدة 8: استخدم In-Class Initializers للـ defaults الثابتة

واستخدم member initializer list للقيم المعتمدة على معاملات constructor.

### القاعدة 9: استخدم `auto` لإزالة التكرار لا لإخفاء المعنى

```cpp
auto handle = std::make_unique<FileHandle>();
Meters distance{calculate_distance()};
```

### القاعدة 10: مثّل الغياب والفشل بأنواع صريحة

استخدم `optional`, `expected`, أو variant مناسبًا بدل sentinel values.

### القاعدة 11: فعّل التحذيرات ولا تتجاهلها

```bash
g++ -std=c++23 -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
    -Wshadow -Wuninitialized -Werror=return-type main.cpp
```

لا تجعل `-Werror` سياسة عمياء لكل dependency وكل compiler، بل طبّقه بطريقة مضبوطة على كود المشروع.

### القاعدة 12: ثبّت السياسة آليًا

استخدم:

- `clang-tidy` لفحوص التهيئة والتحويلات وModern C++.
- `clang-format` لاتساق الصياغة.
- Sanitizers في CI.
- Code review checklist.
- اختبارات تبني الأنواع في الحالات الحدية.

---

## 26. دليل قرار سريع

```text
هل القيمة معروفة الآن؟
|
+-- نعم -> عرّف المتغير وهيئه مباشرة.
|
+-- لا
    |
    +-- هل يمكن تأخير التعريف؟ -> نعم: أخّر التعريف.
    |
    +-- هل الغياب حالة حقيقية؟ -> استخدم optional أو expected.
    |
    +-- هل للكائن default صالح؟ -> استخدم default construction الواعي.

هل تريد قائمة قيم أم عددًا وحشوًا؟
|
+-- قائمة قيم -> braces: vector{1, 2, 3}
+-- عدد/حشو -> parentheses: vector(10, 5)

هل التحويل قد يفقد بيانات؟
|
+-- نعم -> افحص المجال واستخدم تحويلًا صريحًا.
+-- لا  -> اختر صيغة واضحة واترك النوع يحمي العقد.
```

---

## 27. مثال قبل وبعد

### قبل

```cpp
class Report
{
public:
    Report()
    {
        title_ = "Untitled";
        page_count_ = 0;
    }

    void generate(const Data& data)
    {
        double average;
        int count = data.size();

        if (count > 0) {
            average = data.sum() / count;
        } else {
            average = 0;
        }

        render(average);
    }

private:
    std::string title_;
    int page_count_;
};
```

### بعد

```cpp
class Report
{
public:
    Report() = default;

    void generate(const Data& data)
    {
        const auto count = data.size();
        const double average = count == 0
            ? 0.0
            : data.sum() / static_cast<double>(count);

        render(average);
    }

private:
    std::string title_{"Untitled"};
    std::size_t page_count_{0};
};
```

### لماذا النسخة الثانية أفضل؟

- كل عضو يملك default واضحًا قرب تصريحه.
- لا يوجد متغير محلي غير مهيأ.
- `count` و`average` لا يتغيران بعد التهيئة.
- نوع عدد الصفحات يعبّر عن عدم السلبية.
- التحويل إلى `double` ظاهر في موضع القسمة.
- النطاق صغير، وconstructor غير ضروري أصبح `= default`.

---

## 28. مراجعة حسب المستوى

### للمبتدئ

احفظ أربع قواعد:

1. لا تكتب `int x;` ثم تستخدمه.
2. استخدم `nullptr` للمؤشرات الفارغة.
3. عرّف المتغير عند توفر قيمته.
4. انتبه إلى الفرق بين `vector(10, 5)` و`vector{10, 5}`.

### للمستوى المتوسط

أضف:

- `const` افتراضيًا.
- منع narrowing باستخدام braces.
- member initializer lists.
- `optional` بدل sentinel.
- `auto` عندما لا يخفي معنى النوع.

### للمستوى المتقدم

راجع:

- overload resolution مع `initializer_list`.
- قواعد aggregate وdesignated initialization.
- `constexpr` و`constinit` وstatic initialization.
- lifetime extension والمراجع.
- forwarding, `decltype(auto)`, وvalue categories.
- سياسات التحذيرات والتحليل الساكن في CI.

---

## 29. تمارين

### تمرين 1

صحح الكود:

```cpp
int calculate(const std::vector<int>& values)
{
    int result;
    if (!values.empty()) {
        result = values.front();
    }
    return result;
}
```

ناقش هل القيمة الفارغة يجب أن تعطي صفرًا، أم `optional<int>`، أم خطأ.

### تمرين 2

اشرح الفرق في الناتج:

```cpp
std::vector<int> a(4, 2);
std::vector<int> b{4, 2};
```

### تمرين 3

أعد تصميم الصنف بحيث لا توجد حالة غير صالحة:

```cpp
class Email
{
public:
    Email() = default;
    void set_value(std::string value);

private:
    std::string value_;
};
```

### تمرين 4

استبدل القيم الوهمية بنوع مناسب:

```cpp
int selected_user_id{-1};
std::string error_message{""};
```

### تمرين 5

حدد أي المتغيرات يجب أن تكون `const`, `constexpr`, أو `constinit`:

```cpp
int max_attempts = 3;
int attempts = read_attempts();
static int startup_flag = 0;
```

### تمرين 6

اكتب `ServerConfig` كـ aggregate مع defaults وdesignated initializer، ثم أعد تصميمه كـ class إذا أصبح المنفذ مطلوبًا أن يكون ضمن مجال محدد.

---

## 30. أسئلة مراجعة

1. ما الفرق بين التهيئة والإسناد؟
2. لماذا لا تجعل `{}` كل صيغ التهيئة الأخرى غير ضرورية؟
3. ما الفرق بين `vector<int>(5, 1)` و`vector<int>{5, 1}`؟
4. متى يكون default constructor تصميمًا سيئًا؟
5. لماذا يجب أن يطابق ترتيب member initializer list ترتيب التصريح؟
6. كيف يساعد `const` على فهم الكود؟
7. متى يكون `optional<T>` أفضل من `T{}`؟
8. ما الفرق بين `const`, `constexpr`, و`constinit`؟
9. ما الخطر في `auto x = {1, 2, 3}`؟
10. كيف تمنع Static Initialization Order Fiasco؟

---

## 31. قائمة مراجعة Pull Request

- [ ] لا يوجد scalar محلي يُقرأ قبل تهيئته.
- [ ] كل عضو class يحصل على قيمة في كل constructor.
- [ ] لا توجد قيمة sentinel غامضة يمكن تمثيلها بنوع أوضح.
- [ ] المتغيرات معرفة قرب أول استخدام.
- [ ] استُخدم `const` أو `constexpr` عندما لا يلزم التغيير.
- [ ] لا توجد narrowing conversions ضمنية خطرة.
- [ ] الفرق بين `()` و`{}` مقصود، خصوصًا في الحاويات.
- [ ] لا توجد raw owning pointers.
- [ ] initializer lists مرتبة مثل تصريح الأعضاء.
- [ ] defaults الخاصة بالأعضاء موجودة في class عندما تكون مشتركة.
- [ ] النوع يحافظ على invariants بعد الإنشاء.
- [ ] التحذيرات والتحليل الساكن والاختبارات تعمل في CI.

---

## الخلاصة

القاعدة الاحترافية ليست "استخدم الأقواس المعقوفة في كل مكان"، بل:

> **أنشئ كل كائن في أقرب موضع إلى استخدامه، بقيمة صحيحة تعبّر عن المجال، وبصيغة تجعل التحويلات والملكية والنية واضحة.**

استخدم `{}` كخيار قوي للقيم والكائنات البسيطة ومنع narrowing، واستخدم `()` عندما تكون semantics الـ constructor هي المقصودة، خصوصًا مع الحاويات و`initializer_list`. فضّل الأنواع التي تمنع الحالات غير الصالحة، واستخدم `const`, `constexpr`, `optional`, و`expected` حتى تصبح النية جزءًا من النوع نفسه.

---

## المراجع

- [cppreference: Initialization](https://en.cppreference.com/w/cpp/language/initialization)
- [cppreference: List Initialization](https://en.cppreference.com/w/cpp/language/list_initialization)
- [cppreference: Aggregate Initialization](https://en.cppreference.com/w/cpp/language/aggregate_initialization)
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
- [cppreference: C++23](https://en.cppreference.com/w/cpp/23)

### قواعد ذات صلة من C++ Core Guidelines

- ES.20: Always initialize an object.
- ES.21: Do not introduce a variable before you need to use it.
- ES.22: Do not declare a variable until you have a value to initialize it with.
- ES.23: Prefer the `{}` initializer syntax.
- ES.25: Declare an object `const` or `constexpr` unless you want to modify its value later.

---
