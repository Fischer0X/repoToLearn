# repoToLearn
C++ Programming for Hackers
# Smart Pointers in C++23

> **محاضرة تقنية احترافية** حول إدارة الملكية وعمر الكائنات في Modern C++، مع أفضل الممارسات، الأنماط التصميمية، الأخطاء الشائعة، وميزات C++23.

![C++](https://img.shields.io/badge/C%2B%2B-23-00599C?style=flat-square&logo=c%2B%2B)
![Topic](https://img.shields.io/badge/Topic-Smart%20Pointers-f05032?style=flat-square)
![Level](https://img.shields.io/badge/Level-Intermediate%20to%20Advanced-2ea44f?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)

---

## فهرس المحتويات

1. [أهداف المحاضرة](#أهداف-المحاضرة)
2. [المشكلة: من يملك الكائن؟](#المشكلة-من-يملك-الكائن)
3. [RAII ونموذج الملكية](#raii-ونموذج-الملكية)
4. [`std::unique_ptr`](#stdunique_ptr)
5. [`std::shared_ptr`](#stdshared_ptr)
6. [`std::weak_ptr`](#stdweak_ptr)
7. [تمرير Smart Pointers إلى الدوال](#تمرير-smart-pointers-إلى-الدوال)
8. [تعدد الأشكال ووراثة الأصناف](#تعدد-الأشكال-ووراثة-الأصناف)
9. [Custom Deleters وإدارة موارد غير الذاكرة](#custom-deleters-وإدارة-موارد-غير-الذاكرة)
10. [ميزات مرتبطة في C++23](#ميزات-مرتبطة-في-c23)
11. [التزامن والسلامة الخيطية](#التزامن-والسلامة-الخيطية)
12. [الأداء ونموذج الذاكرة](#الأداء-ونموذج-الذاكرة)
13. [الأخطاء الشائعة](#الأخطاء-الشائعة)
14. [شجرة اتخاذ القرار](#شجرة-اتخاذ-القرار)
15. [مثال معماري متكامل](#مثال-معماري-متكامل)
16. [الاختبار وأدوات كشف الأخطاء](#الاختبار-وأدوات-كشف-الأخطاء)
17. [تمارين](#تمارين)
18. [قائمة مراجعة](#قائمة-مراجعة)
19. [المراجع](#المراجع)

---

## أهداف المحاضرة

بنهاية هذه المحاضرة يُفترض أن تكون قادرًا على:

- تفسير الفرق بين **الوصول إلى كائن** و**امتلاك عمره**.
- اختيار `std::unique_ptr` أو `std::shared_ptr` أو `std::weak_ptr` بناءً على نموذج الملكية، لا بناءً على سهولة النسخ.
- تصميم واجهات دوال تعبّر عن نقل الملكية أو مشاركتها أو الاقتراض منها.
- منع التسريبات، والحذف المزدوج، والمؤشرات المتدلية، والدورات المرجعية.
- استخدام `std::out_ptr` و`std::inout_ptr` في C++23 عند التكامل مع واجهات C.
- تحليل كلفة كل اختيار من حيث الذاكرة، والـ allocations، والعمليات الذرية، ووضوح التصميم.

---

## المشكلة: من يملك الكائن؟

المؤشر الخام `T*` يخبرنا بعنوان كائن، لكنه لا يجيب وحده عن الأسئلة التالية:

- من المسؤول عن تدمير الكائن؟
- هل يجوز نسخ المؤشر؟
- هل يبقى الكائن حيًا أثناء الاستدعاء؟
- هل يشير المؤشر إلى كائن واحد أم إلى أول عنصر في مصفوفة؟
- هل استُخدم `new` أم API مخصص يتطلب دالة تحرير مختلفة؟

### مثال هش

```cpp
#include <stdexcept>

class Connection {};

void process()
{
    Connection* connection = new Connection{};

    // إذا رُمي استثناء هنا فلن يصل التنفيذ إلى delete.
    throw std::runtime_error{"processing failed"};

    delete connection;
}
```

المشكلة ليست في `new` فقط، بل في كون مسؤولية التحرير منفصلة زمنيًا ومنطقيًا عن اكتساب المورد.

### القاعدة الحديثة

> اجعل الملكية ممثلة بنوع RAII. استخدم `T*` و`T&` غالبًا للوصول غير المالك، لا للتعبير عن الملكية.

وتوصي C++ Core Guidelines باستخدام `unique_ptr` أو `shared_ptr` لتمثيل الملكية، وتفضيل `unique_ptr` ما لم تكن المشاركة الحقيقية مطلوبة.[^core-guidelines]

---

## RAII ونموذج الملكية

**RAII** اختصار لـ **Resource Acquisition Is Initialization**. الفكرة أن يُربط المورد بعمر كائن تلقائي:

1. يكتسب الكائن المورد في أثناء الإنشاء.
2. يحافظ على invariant صالح.
3. يحرر المورد في destructor.
4. يُستدعى destructor عند الخروج الطبيعي أو أثناء فك المكدس بسبب استثناء.

```cpp
#include <memory>

class Connection {};

void process()
{
    auto connection = std::make_unique<Connection>();

    // لا حاجة إلى delete.
    // يُدمّر Connection تلقائيًا حتى عند رمي استثناء.
}
```

### ثلاث دلالات يجب الفصل بينها

- **Exclusive ownership:** مالك واحد فقط، ويمثله `std::unique_ptr<T>`.
- **Shared ownership:** عدة مالكين يشتركون في إبقاء الكائن حيًا، ويمثله `std::shared_ptr<T>`.
- **Non-owning observation:** وصول لا يُبقي الكائن حيًا، ويمثله عادة `T*` أو `T&`، أو `std::weak_ptr<T>` إذا كان الكائن مملوكًا عبر `shared_ptr` ويجب اكتشاف انتهاء عمره.

### لا تستخدم الـ heap بلا حاجة

```cpp
// الأفضل إذا لم تكن هناك حاجة لعمر ديناميكي أو polymorphism مالك.
Widget widget;

// استخدم heap فقط عندما يفرضه العمر أو الحجم أو polymorphism أو بنية البيانات.
auto dynamic_widget = std::make_unique<Widget>();
```

الكائن ذو التخزين التلقائي أبسط غالبًا من أي smart pointer.

---

## `std::unique_ptr`

يمثل `std::unique_ptr<T>` **ملكية حصرية**. هو قابل للنقل Moveable، وغير قابل للنسخ Copyable. عندما يُدمّر، يستدعي الـ deleter المرتبط به. وتوفر المكتبة تخصصًا للمصفوفات `unique_ptr<T[]>`.[^unique-ptr]

### الإنشاء المفضل

```cpp
#include <memory>
#include <string>

struct User
{
    explicit User(std::string name) : name{std::move(name)} {}
    std::string name;
};

int main()
{
    auto user = std::make_unique<User>("Ada");
}
```

فضّل `std::make_unique` على كتابة `new` مباشرة لأنه:

- يقلل تكرار النوع.
- يجعل الملكية فورية وواضحة.
- يحافظ على سلامة الاستثناءات في التعبيرات المركبة.
- يمنع أخطاء مزاوجة `new` مع نوع smart pointer غير صحيح.

### نقل الملكية

```cpp
#include <memory>

struct Task {};

void submit(std::unique_ptr<Task> task)
{
    // الدالة أصبحت مالكة لـ Task.
}

int main()
{
    auto task = std::make_unique<Task>();
    submit(std::move(task));

    // task أصبح فارغًا بعد النقل.
    if (!task) {
        // متوقع.
    }
}
```

`std::move` لا ينقل بذاته، بل يحول التعبير إلى rvalue يسمح باستدعاء move constructor. بعد النقل يبقى `unique_ptr` المصدر صالحًا لكنه يكون فارغًا في هذا السيناريو.

### إعادة `unique_ptr` من Factory

```cpp
#include <memory>

class Parser
{
public:
    virtual ~Parser() = default;
    virtual void parse() = 0;
};

class JsonParser final : public Parser
{
public:
    void parse() override {}
};

[[nodiscard]] std::unique_ptr<Parser> make_parser()
{
    return std::make_unique<JsonParser>();
}
```

لا تكتب `return std::move(pointer);` على متغير محلي في الحالات العادية، فقد يمنع بعض تحسينات الإرجاع ولا حاجة إليه.

### التخزين في الحاويات

```cpp
#include <memory>
#include <vector>

struct Shape
{
    virtual ~Shape() = default;
    virtual double area() const = 0;
};

std::vector<std::unique_ptr<Shape>> shapes;
```

هذا اختيار طبيعي عندما تملك الحاوية كائنات polymorphic ولا تريد slicing.

### `get`, `release`, `reset`

```cpp
auto pointer = std::make_unique<int>(42);

int* observer = pointer.get(); // اقتراض فقط، لا تحذف observer.

pointer.reset(new int{7});     // يحذف 42 ثم يمتلك 7.
int* raw = pointer.release();  // يتخلى عن الملكية دون حذف.
delete raw;                    // أصبحت مسؤولًا عن التحرير.
```

#### قاعدة مهمة

- استخدم `get()` للتوافق المؤقت مع API لا يتملك المؤشر.
- استخدم `release()` فقط عند نقل الملكية إلى API موثق بوضوح، أو عند بناء adapter منخفض المستوى.
- لا تستدعِ `delete` على ناتج `get()`.

### المصفوفات

```cpp
#include <memory>

std::size_t count = 100;
auto values = std::make_unique<int[]>(count);
values[0] = 42;
```

لكن فضّل `std::vector<T>` للمصفوفات الديناميكية المعتادة، لأنه يحمل الحجم ويدعم الخوارزميات، والنسخ والنقل بصورة أوضح.

### PImpl والأنواع غير المكتملة

يمكن التصريح عن `unique_ptr` إلى نوع غير مكتمل، لكن عند استخدام `default_delete` يجب أن يكون النوع مكتملًا في الموضع الذي يُستدعى فيه الحذف. لذلك عرّف destructor الخاص بالصنف المالك في ملف `.cpp` بعد اكتمال النوع.[^unique-ptr]

```cpp
// service.hpp
#pragma once
#include <memory>

class Service
{
public:
    Service();
    ~Service();

    Service(Service&&) noexcept;
    Service& operator=(Service&&) noexcept;

    Service(const Service&) = delete;
    Service& operator=(const Service&) = delete;

private:
    class Impl;
    std::unique_ptr<Impl> impl_;
};
```

```cpp
// service.cpp
#include "service.hpp"

class Service::Impl
{
    // تفاصيل خاصة.
};

Service::Service() : impl_{std::make_unique<Impl>()} {}
Service::~Service() = default;
Service::Service(Service&&) noexcept = default;
Service& Service::operator=(Service&&) noexcept = default;
```

---

## `std::shared_ptr`

يمثل `std::shared_ptr<T>` **ملكية مشتركة**. يبقى الكائن حيًا حتى يختفي آخر `shared_ptr` مشارك في ملكيته. عادة توجد كتلة تحكم **control block** تحتوي عدّادات الملكية والمراقبة والـ deleter ومعلومات أخرى خاصة بالتنفيذ.

### متى يكون مناسبًا؟

استخدمه عندما يكون نموذج المجال نفسه يتطلب عدة مالكين مستقلين، مثل:

- مهمة غير متزامنة يجب أن تبقي حالة مشتركة حية.
- كائنات تشترك في مورد لا يوجد مالك طبيعي واحد لها.
- API يعيد handles قابلة للاحتفاظ المستقل.

لا تستخدمه لمجرد أنك لا تعرف من يجب أن يملك الكائن. عدم وضوح الملكية مشكلة تصميمية، و`shared_ptr` قد يخفيها بدل حلها.

### الإنشاء

```cpp
#include <memory>
#include <string>

struct Session
{
    std::string id;
};

int main()
{
    auto first = std::make_shared<Session>(Session{"S-101"});
    auto second = first;

    // كلاهما يشارك في إبقاء Session حيًا.
}
```

فضّل `std::make_shared<T>` عادة لأنه ينشئ الكائن وكتلة التحكم غالبًا في allocation واحدة، ما يحسن locality ويقلل كلفة التخصيص.[^core-guidelines]

### ملاحظة عمر الذاكرة مع `make_shared`

إذا بقيت `weak_ptr` بعد اختفاء آخر مالك، يُدمّر `T`، لكن الذاكرة المشتركة التي تضم كتلة التحكم والكائن قد لا تتحرر بالكامل حتى تختفي المراجع الضعيفة. إذا كان `T` ضخمًا وكان هذا الفرق مهمًا ومقاسًا فعليًا، فقد يكون إنشاء `shared_ptr` من allocation منفصل قرارًا مقصودًا. لا تجعل هذا الاستثناء هو القاعدة.

### لا تنشئ أكثر من Control Block للمؤشر نفسه

```cpp
Widget* raw = new Widget{};

std::shared_ptr<Widget> first{raw};
std::shared_ptr<Widget> second{raw}; // خطأ كارثي: كتلتان وتحذفان الكائن مرتين.
```

الصحيح:

```cpp
auto first = std::make_shared<Widget>();
auto second = first;
```

### `std::enable_shared_from_this`

إذا احتاج الكائن إلى إنشاء `shared_ptr` يشترك في كتلة التحكم الموجودة، فلا تستخدم `shared_ptr<T>{this}`.

```cpp
#include <memory>

class Connection : public std::enable_shared_from_this<Connection>
{
public:
    [[nodiscard]] std::shared_ptr<Connection> self()
    {
        return shared_from_this();
    }
};

int main()
{
    auto connection = std::make_shared<Connection>();
    auto same_owner = connection->self();
}
```

استدعاء `shared_from_this()` قبل أن يصبح الكائن مملوكًا من `shared_ptr` صالح يؤدي إلى `std::bad_weak_ptr`. اجعل الإنشاء عبر factory إذا أردت فرض invariant.

### Aliasing constructor

يمكن لـ `shared_ptr<U>` مشاركة ملكية كائن، مع الإشارة إلى subobject مختلف:

```cpp
#include <memory>
#include <string>

struct Profile
{
    std::string display_name;
};

int main()
{
    auto profile = std::make_shared<Profile>(Profile{"Ada"});

    std::shared_ptr<std::string> name{
        profile,
        &profile->display_name
    };

    // name يشير إلى display_name لكنه يشارك ملكية Profile كله.
}
```

هذا متقدم ومفيد، لكنه قد يجعل علاقة العنوان بالملكية غير بديهية. وثّق استخدامه وتجنب تمرير مؤشر خام مأخوذ من aliasing pointer إلى عمليات قد تفقد المالك.[^core-guidelines]

---

## `std::weak_ptr`

`std::weak_ptr<T>` مراقب غير مالك لكائن تديره مجموعة `shared_ptr`. لا يزيد strong reference count، لذا لا يمنع تدمير الكائن.

### لماذا لا نستخدمه مباشرة؟

قد ينتهي عمر الكائن بين فحصه واستخدامه. لذلك استخدم `lock()` الذي يعيد `shared_ptr` صالحًا أو فارغًا بطريقة ذرية بالنسبة إلى انتهاء الملكية:

```cpp
#include <memory>

void use(std::weak_ptr<Session> weak_session)
{
    if (auto session = weak_session.lock()) {
        // session يبقي الكائن حيًا خلال هذا النطاق.
    } else {
        // انتهى عمر Session.
    }
}
```

لا تعتمد على النمط التالي:

```cpp
if (!weak_session.expired()) {
    // قد ينتهي العمر هنا قبل الاستخدام اللاحق.
}
```

استخدم `expired()` للاستعلام أو التشخيص، و`lock()` للاستخدام الآمن.

### كسر الدورة المرجعية

```cpp
#include <memory>
#include <string>
#include <vector>

struct Node
{
    explicit Node(std::string name) : name{std::move(name)} {}

    std::string name;
    std::vector<std::shared_ptr<Node>> children; // Ownership downward.
    std::weak_ptr<Node> parent;                  // Observation upward.
};
```

لو كان `parent` من نوع `shared_ptr` فقد تتكون دورة تمنع وصول العداد القوي إلى الصفر. توصي Core Guidelines باستخدام `weak_ptr` لكسر دورات `shared_ptr`.[^core-guidelines]

### حالات استخدام أخرى

- cache لا تريد أن يبقي العناصر حية.
- observer يحتاج اكتشاف انتهاء المصدر.
- callback غير مالك في نظام asynchronous.

ملاحظة: `weak_ptr` ليس بديلًا عامًا للمؤشر الخام. استخدمه تحديدًا عندما تحتاج مراقبة lifetime تديره `shared_ptr`.

---

## تمرير Smart Pointers إلى الدوال

نوع المعامل جزء من عقد الملكية. لا تمرر smart pointer فقط لأن المتصل يخزن الكائن بواحد. توصي Core Guidelines بأخذ smart pointer كمعامل فقط عندما تريد التعبير عن دلالة lifetime.[^core-guidelines] كما توصي أدوات Microsoft التحليلية باستخدام `T*` أو `T&` عندما تكون الدالة لا تدير العمر بل تصل إلى الكائن فقط.[^ms-c26415]

### 1. الدالة تستخدم كائنًا مطلوبًا ولا تتملكه

```cpp
void render(const Widget& widget);
```

استخدم reference عندما لا يُسمح بـ `null`.

### 2. الدالة تستخدم كائنًا اختياريًا ولا تتملكه

```cpp
void set_parent(const Widget* parent);
```

استخدم pointer عندما يكون `nullptr` ذا معنى.

### 3. الدالة تستحوذ على الملكية الحصرية

```cpp
void install(std::unique_ptr<Plugin> plugin);

install(std::move(plugin));
```

التمرير بالقيمة يجعل نقل الملكية صريحًا.

### 4. الدالة قد تستبدل `unique_ptr` لدى المستدعي

```cpp
void reload(std::unique_ptr<Plugin>& plugin)
{
    plugin = std::make_unique<Plugin>();
}
```

هذا عقد reseat. استخدمه فقط عندما تكون تلك هي النية.

### 5. الدالة تشارك الملكية

```cpp
void schedule(std::shared_ptr<Job> job)
{
    queue.push_back(std::move(job));
}
```

التمرير بالقيمة مناسب عندما ستحتفظ الدالة بنسخة.

### 6. الدالة تقترض فقط من كائن مملوك بـ `shared_ptr`

```cpp
void inspect(const Job& job);

inspect(*job_pointer);
```

لا تستخدم `const std::shared_ptr<Job>&` إذا لم تكن دلالة المشاركة مطلوبة.

### 7. callback لا ينبغي أن يطيل العمر

```cpp
void register_observer(std::weak_ptr<Observer> observer);
```

### ملخص العقود

| التوقيع | الدلالة |
|---|---|
| `void f(T&)` | اقتراض غير قابل للـ null |
| `void f(const T&)` | قراءة غير مالكة وغير قابلة للـ null |
| `void f(T*)` | اقتراض اختياري |
| `void f(std::unique_ptr<T>)` | استحواذ على ملكية حصرية |
| `void f(std::unique_ptr<T>&)` | قد يعيد تعيين المالك الحصري |
| `void f(std::shared_ptr<T>)` | مشاركة أو احتفاظ بالملكية |
| `void f(std::shared_ptr<T>&)` | قد يعيد تعيين المالك المشترك |
| `void f(std::weak_ptr<T>)` | مراقبة lifetime مشتركة دون إبقاء الكائن حيًا |

---

## تعدد الأشكال ووراثة الأصناف

### destructor افتراضي للقاعدة

إذا سيُحذف كائن مشتق من خلال مؤشر إلى القاعدة، اجعل destructor القاعدة افتراضيًا:

```cpp
struct Interface
{
    virtual ~Interface() = default;
    virtual void execute() = 0;
};
```

بدون ذلك، يؤدي الحذف عبر مؤشر القاعدة إلى undefined behavior.

### تحويل `unique_ptr<Derived>` إلى `unique_ptr<Base>`

```cpp
std::unique_ptr<Interface> make_command()
{
    return std::make_unique<ConcreteCommand>();
}
```

ينجح التحويل عندما تكون علاقة المؤشرات والـ deleter متوافقة.

### تجنب slicing

```cpp
std::vector<Interface> objects; // غير مناسب لواجهة abstract، ويسبب slicing في حالات أخرى.
std::vector<std::unique_ptr<Interface>> objects; // ملكية polymorphic صحيحة.
```

### Clone افتراضي آمن

```cpp
struct Shape
{
    virtual ~Shape() = default;
    [[nodiscard]] virtual std::unique_ptr<Shape> clone() const = 0;
};

struct Circle final : Shape
{
    [[nodiscard]] std::unique_ptr<Shape> clone() const override
    {
        return std::make_unique<Circle>(*this);
    }
};
```

---

## Custom Deleters وإدارة موارد غير الذاكرة

المؤشرات الذكية ليست لإدارة الذاكرة فقط. يمكنها إدارة أي handle يملك قاعدة تحرير مناسبة، لكن أحيانًا يكون wrapper مخصص أكثر وضوحًا.

### إدارة `FILE*`

```cpp
#include <cstdio>
#include <memory>

struct FileCloser
{
    void operator()(std::FILE* file) const noexcept
    {
        if (file != nullptr) {
            std::fclose(file);
        }
    }
};

using FileHandle = std::unique_ptr<std::FILE, FileCloser>;

[[nodiscard]] FileHandle open_file(const char* path)
{
    return FileHandle{std::fopen(path, "rb")};
}
```

### تأثير الـ deleter على الحجم

الـ deleter جزء من نوع `unique_ptr`. deleter عديم الحالة قد يستفيد من Empty Base Optimization أو `[[no_unique_address]]` داخليًا حسب التنفيذ. أما function pointer أو deleter ذو حالة فقد يزيد حجم المؤشر.

```cpp
using FunctionDeleter = void (*)(Resource*);
using ResourcePtr = std::unique_ptr<Resource, FunctionDeleter>;
```

فضّل functor عديم الحالة عندما يكون مناسبًا.

### لا تخلط آليات التخصيص والتحرير

- `new` يقابله `delete`.
- `new[]` يقابله `delete[]`.
- `malloc` يقابله `free`.
- `fopen` يقابله `fclose`.
- API خارجي يجب أن يستخدم دالة التحرير التي يوثقها.

---

## ميزات مرتبطة في C++23

الأنواع الأساسية `unique_ptr`, `shared_ptr`, و`weak_ptr` أقدم من C++23. الإضافة المهمة في C++23 للتكامل مع C APIs هي `std::out_ptr` و`std::inout_ptr` في `<memory>`.[^cpp23] [^inout-ptr]

> قد يتفاوت دعم ميزات C++23 بين إصدارات المكتبات القياسية. افحص توثيق compiler والـ standard library المستخدمة، ويمكن فحص feature-test macro باسم `__cpp_lib_out_ptr`.

### المشكلة مع C APIs

واجهات C قد تستخدم معاملًا مثل `T**` كي تُرجع موردًا للمتصل:

```cpp
extern "C" int create_resource(Resource** out);
extern "C" int replace_resource(Resource** inout);
extern "C" void destroy_resource(Resource* resource);
```

قبل C++23 قد نضطر إلى إدارة raw pointer مؤقتًا ثم `reset` أو `release` بحذر.

### `std::out_ptr`

استخدمه عندما ستكتب الدالة الخارجية مؤشرًا جديدًا إلى out-parameter، ويجب أن يتبناه smart pointer:

```cpp
#include <memory>

struct ResourceDeleter
{
    void operator()(Resource* resource) const noexcept
    {
        destroy_resource(resource);
    }
};

using ResourcePtr = std::unique_ptr<Resource, ResourceDeleter>;

[[nodiscard]] ResourcePtr make_resource()
{
    ResourcePtr resource;

    if (create_resource(std::out_ptr(resource)) != 0) {
        throw std::runtime_error{"create_resource failed"};
    }

    return resource;
}
```

### `std::inout_ptr`

استخدمه عندما تستقبل C API مؤشرًا موجودًا، وقد تحرره أو تستبدله. `inout_ptr` يكيّف smart pointer لهذا العقد، ويجري عملية release اللازمة قبل الاستدعاء ثم يعيد ربط الناتج وفق semantics الـ adapter.[^inout-ptr]

```cpp
ResourcePtr resource;

// بعد تهيئة resource بالطريقة الملائمة:
if (replace_resource(std::inout_ptr(resource)) != 0) {
    throw std::runtime_error{"replace_resource failed"};
}
```

### تحذيرات مهمة

- افهم عقد C API بدقة: هل يحرر القيمة القديمة؟ ماذا يحدث عند الفشل؟
- اجعل object الناتج من `out_ptr` أو `inout_ptr` temporary داخل full-expression؛ فهو يلتقط معاملات reset بالمراجع.[^inout-ptr]
- لا تستخدم `inout_ptr` تلقائيًا لكل `T**`. الاختيار يعتمد على ملكية القيمة الداخلة وسلوك API.
- لا تفترض توفر الميزة لمجرد تمرير `-std=c++23`. تحقق من نسخة standard library.

---

## التزامن والسلامة الخيطية

هناك فرق بين:

1. سلامة **كتلة التحكم** الخاصة بـ `shared_ptr`.
2. سلامة **الكائن المشار إليه**.
3. سلامة **كائن shared_ptr نفسه** عند تعديله.

يمكن لنسخ مختلفة من `shared_ptr` تشترك في كتلة تحكم واحدة أن تُنسخ أو تُدمر عبر threads وفق ضمانات المكتبة، لكن هذا لا يجعل `T` نفسه thread-safe. كما لا يجوز تعديل كائن `shared_ptr` نفسه من threads متعددة دون مزامنة مناسبة.

```cpp
struct Counter
{
    void increment(); // يجب أن يطبق تزامنه الخاص إذا استُخدم بالتوازي.
};

auto counter = std::make_shared<Counter>();
```

وجود `shared_ptr` يمنع تدمير `Counter` مبكرًا، لكنه لا يمنع data races داخل `Counter`.

للوصول الذري إلى shared ownership استخدم واجهات `std::atomic<std::shared_ptr<T>>` المناسبة، لا قفلًا مفترضًا داخل `shared_ptr`.

### نمط callback غير متزامن

```cpp
class Worker : public std::enable_shared_from_this<Worker>
{
public:
    void start()
    {
        std::weak_ptr<Worker> weak_self = weak_from_this();

        executor_.post([weak_self] {
            if (auto self = weak_self.lock()) {
                self->run_one_step();
            }
        });
    }

private:
    void run_one_step() {}
    Executor executor_;
};
```

استخدم `shared_ptr` في الـ lambda إذا كان مطلوبًا أن تطيل المهمة عمر الكائن حتمًا. استخدم `weak_ptr` إذا كان الإلغاء الصامت بعد تدمير المالك هو السلوك المرغوب. هذا قرار semantics لا مجرد منع crash.

---

## الأداء ونموذج الذاكرة

### `unique_ptr`

- غالبًا بحجم مؤشر خام عندما يكون الـ deleter عديم الحالة.
- لا يوجد reference counting.
- النقل عادة نقل للمؤشر والـ deleter.
- الاختيار الافتراضي للملكية الديناميكية الحصرية.

### `shared_ptr`

- يحمل عادة مؤشرًا إلى العنصر ومعلومة مرتبطة بكتلة التحكم حسب التنفيذ.
- النسخ والتدمير يعدلان strong count، وغالبًا يتطلبان عمليات ذرية.
- يحتاج control block.
- `make_shared` يقلل allocations غالبًا إلى allocation واحدة.
- قد يزيد contention عندما تُنسخ الملكية بكثافة عبر threads.

### `weak_ptr`

- يحتاج control block مثل `shared_ptr` لكنه لا يزيد strong count.
- `lock()` يختبر الحياة ويحاول إنشاء مالك مشترك.

### لا تحسّن بلا قياس

الأولوية الأولى هي صحة نموذج الملكية. بعد ذلك استخدم benchmark وprofiler. لا تستبدل `shared_ptr` بمؤشرات خام في hot path من دون إثبات lifetime واضح، ولا تمرر `shared_ptr` بالقيمة إن لم تكن بحاجة إلى المشاركة.

---

## الأخطاء الشائعة

### 1. استخدام `shared_ptr` كخيار افتراضي

**المشكلة:** يخفي المالك الحقيقي، ويضيف كلفة، وقد ينشئ دورات.

**الأفضل:** ابدأ بكائن مباشر، ثم `unique_ptr`، وانتقل إلى `shared_ptr` فقط عند وجود ملكية مشتركة حقيقية.

### 2. إنشاء `shared_ptr` من `this`

```cpp
std::shared_ptr<Widget> bad()
{
    return std::shared_ptr<Widget>{this};
}
```

ينشئ control block جديدة وقد ينتهي بحذف مزدوج. استخدم `enable_shared_from_this` مع إنشاء أصلي عبر `shared_ptr`.

### 3. حذف ناتج `get()`

```cpp
auto owner = std::make_unique<Widget>();
delete owner.get(); // خطأ: سيحاول owner الحذف مرة أخرى.
```

### 4. امتلاك المؤشر الخام نفسه مرتين

```cpp
Widget* raw = new Widget{};
std::unique_ptr<Widget> a{raw};
std::unique_ptr<Widget> b{raw}; // حذف مزدوج.
```

### 5. `release()` دون مالك جديد

```cpp
pointer.release(); // تسريب إذا لم يُلتقط الناتج ولم تنتقل الملكية.
```

### 6. دورة `shared_ptr`

اجعل اتجاه الملكية قويًا، والروابط العكسية أو المراقبة ضعيفة.

### 7. تمرير smart pointer عندما تحتاج إلى الكائن فقط

```cpp
// غير ضروري إذا كانت الدالة تقرأ Widget فقط.
void print(const std::shared_ptr<Widget>& widget);

// أوضح.
void print(const Widget& widget);
```

### 8. التقاط `shared_ptr` في callback طويل العمر بلا قصد

```cpp
auto self = shared_from_this();
callback_ = [self] { self->run(); };
```

إذا كان `callback_` عضوًا في `self` فقد تنشأ دورة. استخدم `weak_ptr` إن كانت semantics تسمح.

### 9. استخدام `use_count()` لاتخاذ قرار مزامنة

قيمة `use_count()` قد تتغير فورًا، ولا تثبت انفرادًا آمنًا في نظام متزامن. استخدمها للتشخيص بحذر، لا كآلية synchronization أو correctness.

### 10. `unique_ptr<Base>` بقاعدة ذات destructor غير افتراضي

عند الحذف polymorphically، يجب أن يكون destructor القاعدة افتراضيًا أو يجب استعمال deleter صحيح يضمن التدمير الكامل.

### 11. استخدام `unique_ptr<T[]>` بدل container بلا سبب

استخدم `std::vector<T>` افتراضيًا، أو `std::array<T, N>` للحجم الثابت.

### 12. افتراض أن smart pointer يمنع كل dangling references

قد تأخذ `T*` من `get()` أو `T&` من `*pointer` ثم تحتفظ به بعد انتهاء المالك. smart pointer يحمي الملكية التي يديرها، لا كل الاقتراضات التي تنشئها.

---

## شجرة اتخاذ القرار

```text
هل تحتاج أصلًا إلى تخصيص ديناميكي؟
|
+-- لا  -> استخدم كائنًا مباشرًا T أو container قياسيًا.
|
+-- نعم
    |
    +-- هل يوجد مالك واحد واضح؟
    |   |
    |   +-- نعم -> std::unique_ptr<T>
    |   |
    |   +-- لا
    |       |
    |       +-- هل عدة أطراف يجب أن تُبقي الكائن حيًا استقلاليًا؟
    |           |
    |           +-- نعم -> std::shared_ptr<T>
    |           |          والروابط غير المالكة -> std::weak_ptr<T>
    |           |
    |           +-- لا -> أعد تصميم الملكية، واستخدم T& أو T* للاقتـراض.
    |
    +-- هل المورد من C API ويخرج عبر T**؟
        |
        +-- قيمة جديدة فقط -> std::out_ptr في C++23
        +-- قيمة موجودة قد تُستبدل -> std::inout_ptr في C++23
```

### ترتيب تفضيل عملي

1. كائن مباشر ذو automatic storage.
2. container قياسي مثل `std::vector` أو `std::string`.
3. `std::unique_ptr` للملكية الديناميكية الحصرية.
4. `std::shared_ptr` عند إثبات المشاركة.
5. `std::weak_ptr` لمراقبة ملكية مشتركة دون تمديد العمر.
6. مؤشرات ومراجع خام للاقتـراض، ضمن lifetime واضح.

---

## مثال معماري متكامل

المثال التالي يصمم نظام أوامر:

- `CommandBus` يملك الأوامر حصريًا أثناء الانتظار.
- `ExecutionContext` حالة مشتركة بين مهام غير متزامنة.
- `Observer` يُراقب دون أن يفرض بقاءه حيًا.

```cpp
#include <concepts>
#include <deque>
#include <memory>
#include <string>
#include <utility>
#include <vector>

class Observer
{
public:
    virtual ~Observer() = default;
    virtual void on_completed(std::string_view command_name) = 0;
};

class ExecutionContext
{
public:
    void write_log(std::string message)
    {
        logs_.push_back(std::move(message));
    }

private:
    std::vector<std::string> logs_;
};

class Command
{
public:
    virtual ~Command() = default;
    [[nodiscard]] virtual std::string_view name() const noexcept = 0;
    virtual void execute(ExecutionContext& context) = 0;
};

class BuildCommand final : public Command
{
public:
    [[nodiscard]] std::string_view name() const noexcept override
    {
        return "build";
    }

    void execute(ExecutionContext& context) override
    {
        context.write_log("project built");
    }
};

class CommandBus
{
public:
    explicit CommandBus(std::shared_ptr<ExecutionContext> context)
        : context_{std::move(context)}
    {
    }

    void subscribe(std::weak_ptr<Observer> observer)
    {
        observers_.push_back(std::move(observer));
    }

    void submit(std::unique_ptr<Command> command)
    {
        queue_.push_back(std::move(command));
    }

    void run_next()
    {
        if (queue_.empty()) {
            return;
        }

        auto command = std::move(queue_.front());
        queue_.pop_front();

        command->execute(*context_);
        notify(command->name());
    }

private:
    void notify(std::string_view command_name)
    {
        auto iterator = observers_.begin();

        while (iterator != observers_.end()) {
            if (auto observer = iterator->lock()) {
                observer->on_completed(command_name);
                ++iterator;
            } else {
                iterator = observers_.erase(iterator);
            }
        }
    }

    std::shared_ptr<ExecutionContext> context_;
    std::deque<std::unique_ptr<Command>> queue_;
    std::vector<std::weak_ptr<Observer>> observers_;
};

int main()
{
    auto context = std::make_shared<ExecutionContext>();
    CommandBus bus{context};

    bus.submit(std::make_unique<BuildCommand>());
    bus.run_next();
}
```

### تحليل الملكية

- `CommandBus::submit(unique_ptr<Command>)` تعني أن الناقل يستحوذ على الأمر.
- `deque<unique_ptr<Command>>` تمنع slicing وتضمن مالكًا واحدًا لكل أمر.
- `shared_ptr<ExecutionContext>` مناسب فقط إذا كانت الحالة فعلًا مشتركة مع مكونات أخرى مستقلة العمر.
- `weak_ptr<Observer>` يمنع نظام الأحداث من إبقاء المراقبين أحياء قسرًا.
- `execute(ExecutionContext&)` اقتراض غير مالك لأن الأمر لا يحتفظ بالسياق.

إذا كان `CommandBus` هو المالك الوحيد للسياق، فالأفضل تحويله إلى قيمة مباشرة أو `unique_ptr`. التصميم يُحكم عليه وفق العلاقات الفعلية، لا وفق المثال وحده.

---

## الاختبار وأدوات كشف الأخطاء

### البناء بتحذيرات قوية

#### GCC أو Clang

```bash
g++ -std=c++23 -Wall -Wextra -Wpedantic -Wconversion -Wshadow \
    -fsanitize=address,undefined -fno-omit-frame-pointer \
    main.cpp -o smart-pointers-demo
```

#### تشغيل

```bash
./smart-pointers-demo
```

### أدوات مفيدة

- **AddressSanitizer:** يكشف use-after-free، الحذف المزدوج، وتجاوزات الذاكرة.
- **UndefinedBehaviorSanitizer:** يكشف فئات من undefined behavior.
- **LeakSanitizer:** يتوفر غالبًا ضمن بيئة sanitizer على أنظمة مدعومة.
- **Valgrind:** مفيد على Linux عندما يناسب المنصة والبناء.
- **clang-tidy:** طبّق فحوص Modern C++ وCore Guidelines المناسبة للمشروع.
- **Static Analysis:** مثل MSVC Code Analysis وتحذيرات C26415 ذات الصلة بعقود smart pointers.[^ms-c26415]

### اختبارات عمر الكائن

يمكن استخدام عدّاد تشخيصي داخل test فقط:

```cpp
#include <cassert>
#include <memory>

struct Tracked
{
    Tracked() { ++alive; }
    ~Tracked() { --alive; }

    inline static int alive = 0;
};

void lifetime_test()
{
    assert(Tracked::alive == 0);

    {
        auto value = std::make_unique<Tracked>();
        assert(Tracked::alive == 1);
    }

    assert(Tracked::alive == 0);
}
```

لا تعتمد على هذا بدل sanitizers، بل كاختبار semantics محدد.

---

## تمارين

### تمرين 1: نقل الملكية

صمّم `Repository` يملك `DatabaseConnection` حصريًا. امنع النسخ واسمح بالنقل. أضف factory تعيد `unique_ptr<Repository>`.

### تمرين 2: اكتشاف دورة

أنشئ صنفين `Person` و`Team` يرتبطان بـ `shared_ptr` في الاتجاهين، ثم أثبت عبر destructors أن الكائنات لا تُدمّر. أصلح التصميم بجعل اتجاه واحد `weak_ptr`، واشرح لماذا هو الاتجاه غير المالك.

### تمرين 3: واجهات الدوال

أعد تصميم التواقيع التالية كي تعبر عن النية:

```cpp
void inspect(std::shared_ptr<Document> document);
void save(std::unique_ptr<Document>& document);
void register_cache(Document* document);
```

حدد أولًا: هل الدالة تقترض، تستحوذ، تشارك، تعيد التعيين، أم تراقب؟ لا توجد إجابة صحيحة دون عقد وظيفي.

### تمرين 4: Custom Deleter

أنشئ RAII wrapper باستخدام `unique_ptr` لمورد C يفتح بـ `library_open()` ويغلق بـ `library_close()`.

### تمرين 5: C++23 Interop

اكتب adapter لواجهة C تعيد `Image**`. استخدم `std::out_ptr` مع deleter يستدعي `image_destroy()`. عالج حالة الفشل دون تسريب.

### تمرين 6: تصميم Event Bus

صمم Event Bus لا يبقي الـ subscribers أحياء. استخدم `weak_ptr`, ونظف المراجع المنتهية أثناء النشر. ناقش أثر التزامن.

---

## قائمة مراجعة

قبل دمج الكود، اسأل:

- [ ] هل أحتاج heap allocation أصلًا؟
- [ ] هل يستطيع القارئ تحديد المالك من النوع وحده؟
- [ ] هل بدأت بـ `unique_ptr` بدل `shared_ptr`؟
- [ ] هل المشاركة حقيقية أم تعويض عن تصميم غير واضح؟
- [ ] هل الروابط العكسية أو callbacks قد تنشئ دورة؟
- [ ] هل معاملات الدوال تعبّر عن lifetime semantics؟
- [ ] هل استخدمت `T&` أو `T*` عندما كان المطلوب اقتراضًا فقط؟
- [ ] هل destructor القاعدة افتراضي عند الحذف polymorphically؟
- [ ] هل تجنبت `shared_ptr<T>{this}` وامتلاك raw pointer أكثر من مرة؟
- [ ] هل كل custom deleter يطابق API الذي أنشأ المورد؟
- [ ] هل فحصت callbacks غير المتزامنة ودورات الالتقاط؟
- [ ] هل شغلت sanitizers والتحليل الساكن؟
- [ ] هل قست الأداء قبل إجراء تحسينات منخفضة المستوى؟
- [ ] هل تحققت من دعم `out_ptr` و`inout_ptr` في standard library المستخدمة؟

---

## الخلاصة

Smart pointers ليست بديلًا شكليًا عن `new` و`delete`. هي **لغة لتوصيف الملكية**:

- `unique_ptr` يقول: **أنا المالك الوحيد**.
- `shared_ptr` يقول: **أنا أحد المالكين الذين يبقون الكائن حيًا**.
- `weak_ptr` يقول: **أراقب كائنًا ذا ملكية مشتركة دون تمديد عمره**.
- `T&` و`T*` يقولان غالبًا: **أقترض الوصول ولا أملك**.

التصميم الأفضل هو الذي يجعل العمر واضحًا من التواقيع وبنية الكائنات، ويمنع الحالات غير الصحيحة قبل التشغيل. ابدأ دائمًا بأبسط نموذج ملكية، واجعل المشاركة قرارًا معماريًا مثبتًا.

---

## المراجع

[^core-guidelines]: [C++ Core Guidelines, Resource Management and Smart Pointer Rules](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-resource)
[^unique-ptr]: [cppreference: `std::unique_ptr`](https://en.cppreference.com/w/cpp/memory/unique_ptr)
[^cpp23]: [cppreference: C++23 Library Features](https://en.cppreference.com/w/cpp/23)
[^inout-ptr]: [cppreference: `std::inout_ptr`](https://en.cppreference.com/w/cpp/memory/inout_ptr_t/inout_ptr)
[^ms-c26415]: [Microsoft Learn: C26415, smart pointer parameter used only for access](https://learn.microsoft.com/en-us/cpp/code-quality/c26415?view=msvc-170)

### مراجع إضافية

- [cppreference: `std::shared_ptr`](https://en.cppreference.com/w/cpp/memory/shared_ptr)
- [cppreference: `std::weak_ptr`](https://en.cppreference.com/w/cpp/memory/weak_ptr)
- [cppreference: `std::out_ptr`](https://en.cppreference.com/w/cpp/memory/out_ptr_t/out_ptr)
- [Microsoft Learn: Smart pointers in modern C++](https://learn.microsoft.com/en-us/cpp/cpp/smart-pointers-modern-cpp?view=msvc-170)

---



