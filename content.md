class: center, middle, inverse

# Selected **C++** Idioms

## Part 1

### Sundaram Ramaswamy

---
class: center, middle, inverse

## C++ has indeed become too **‘expert friendly’**

### .right[— Bjarne Stroustrup, creator of C++]

---

# Erase–Remove

**Problem**: Remove some elements from a `std::vector`

## **Before**

``` c++
std::vector v = {1, 2, 3, 4, 5, 6, 7, 8, 9};  //  +---------------------------+
for (auto it = v.begin(); it != v.end(); ) {  //  | 1  2  3  4  5  6  7  8  9 |
  if (*it % 3 == 0)                           //  +---------------------------+
    it = v.erase(it);                         //    +------------------------+
  else                                        //    | 1  2  4  5  6  7  8  9 |
    ++it;                                     //    +------------------------+
}                                             //                ⋮
```

--

## **After**

``` c++
v.erase(std::remove_if(std::begin(v), std::end(v), [](int i){ return i % 3 == 0; }),
        std::end(v));
```

Complexity: _O(n)_ ~~_O(n²)_~~

---

## Erase–Remove

* `<algorithm>` has two variants
  - `std::remove`: remove `e` if `e == x`
  - `std::remove_if`: remove `e` if `f(e) == true`

* `std::remove` doesn’t really remove anything
  - Overwrites by moving following elements — poorly named 🤦
  - Returns collection’s new end

* Elements from first occurrence to end are moved

* Standard guarantees linear complexity

* `std::vector::erase(begin, end)` removes elements in a given range
  - Just drops last `n` elements by simply resizing

**Usage**: [`chrome/browser/ui/webui/settings/site_settings_helper.cc`][erase-remove-eg]

[erase-remove-eg]: https://source.chromium.org/chromium/chromium/src/+/master:chrome/browser/ui/webui/settings/site_settings_helper.cc;l=735?q=chrome%2Fbrowser%2Fui%2Fwebui%2Fsettings%2Fsite_settings_helper.cc&ss=chromium

---

# RAII Wrapper

**Problem**: Consistent clean-up over various code paths

## **Before**

``` c++
HANDLE f = CreateFile("path", …);
// do stuff with f -- may throw or return early
CloseHandle(f);

GtkWidget* win = gtk_window_new(…);
// do fancy stuff with window -- may throw or return early
gtk_widget_destroy(win);
```

--

## **After**

``` c++
struct MyHandleDeleter {
  void operator()(HANDLE h) { if (h != INVALID_HANDLE_VALUE) CloseHandle(h); }
};
struct GtkWidgetDeleter {
  void operator()(GtkWidget* w) { if (w) gtk_widget_destroy(w); }
};

std::unique_ptr<HANDLE, MyHandleDeleter> handle(CreateFile("path", …));
std::unique_ptr<GtkWidget, GtkWidgetDeleter> window(gtk_window_new(…));

// Look ma, no releases!
// Return anywhere; stack unwinding ensure calls to all locals' destructor
```

---

* Avoid poor variant incurring extra storage for `Deleter` function pointer

``` c++
struct FreeFunctor {
  void operator()(void* p) { if (p) free(p); }
};

std::unique_ptr<void, FreeFunctor>     mem1(malloc(4));                  // GOOD
std::unique_ptr<void, decltype(&free)> mem2(malloc(4), &free);           // BAD
std::unique_ptr<void, void(*)(void*)>  mem3(malloc(4), [](void* p){      // UGLY
                                                         if (p) free(p);
                                                       });
```
## Merits
* Automatic clean-up
* **Finalizers**: run some code at
  - Function exit (clean-ups)
  - Object destruction
* Exception safety

## Examples
- `std::lock_guard` wrapper for `std::mutex`
- `base::scoped_nsobject` for macOS `NSObject` (Objective-C++)

---

# Scope Guard

* Empty wrappers to run code on abnormality e.g. `base::ScopeExit`
* Allows disarming guard (for success path)
* Frees developer from remembering to add a bunch of code at each return
* Common return code scoped to a single location lexically

## **Before**

``` c++
bool foo() {
  int failure_value = 2;

  if (IsFailureConditionOneTrue()) {
    failure_value += 3;
    LogFailureTelemetry(failure_value);
    return false;
  }

  if (AnotherFailureConditionIsTrue())
    failure_value += 1;

  if (IsFailureConditionTwoTrue()) {
    failure_value += 5;
    LogFailureTelemetry(failure_value);
    return false;
  }

  return true;
}
```

---

## **After**

``` c++
#include "base/scope_exit.h"
bool foo() {
  int failure_value = 2;

  auto log_failure = base::MakeScopeExit([&] {
     LogFailureTelemetry(failure_value);
  });

  if (IsFailureConditionOneTrue()) {
    failure_value += 3;
    return false;
  }

  if (AnotherFailureConditionIsTrue())
    failure_value += 1;

  if (IsFailureConditionTwoTrue()) {
    failure_value += 5;
    return false;
  }

  log_failure.Dismiss(); // disable logging on success path
  return true;
}
```
* Examples (from around 70 such `base::Scope*` types)
  - [`base::ScopedObserver`](https://source.chromium.org/chromium/chromium/src/+/master:base/scoped_observer.h)
  - [`base::ScopedNativeLibrary`](https://source.chromium.org/chromium/chromium/src/+/master:base/scoped_native_library.h)
  - [`base::ScopedBlockingCall`](https://source.chromium.org/chromium/chromium/src/+/master:base/threading/scoped_blocking_call.h)

---

# Named Constructor

### **Problems**

* Missing clarity between various constructors of your class
  - All constructors of a class have the same name; semantically ambiguous
  - Only way to differentiate is number, type or order of parameters
* No control over where your class objects are instantiated

### Before

``` c++
class Point {
public:
  Point(float x, float y);     // Rectangular coordinates
  Point(float r, float a);     // Polar coordinates (radius and angle)
  // ERROR: Overload is Ambiguous: Point::Point(float,float)
};

int main()
{
  Point p1 = Point(5.7, 1.2);  // Ambiguous: Which coordinate system?

  Point *p2 = new Point(4.9, 8.3);  // Creatable on the heap too :(
}
```

---

### After

``` c++
class Point {
public:
  static Point rectangular(float x, float y);      // Rectangular coord's
  static Point polar(float radius, float angle);   // Polar coordinates

private:                       // protected works too
  Point(float x, float y);
  float x_, y_;
};

Point::Point(float x, float y) : x_(x), y_(y) { }

Point Point::rectangular(float x, float y) {
  return Point(x, y);
}

Point Point::polar(float radius, float angle) {
  return Point(radius*std::cos(angle), radius*std::sin(angle));
}

int main() {
  Point p1 = Point::rectangular(5.7, 1.2);   // Obviously rectangular
  Point p2 = Point::polar(5.7, 1.2);         // Obviously polar

  // Point *p3 = new Point();    // Error: can't access private ctor :)
}
```

* [`unique_ptr<DeclarativeConditionSet> extensions::DeclarativeConditionSet::Create`](https://source.chromium.org/chromium/chromium/src/+/master:extensions/browser/api/declarative/declarative_rule.h;l=71)
* [`scoped_refptr<ObjectManager> dbus::ObjectManager::Create`](https://source.chromium.org/chromium/chromium/src/+/master:dbus/object_manager.h;l=187)
* [`scoped_refptr<BasicShapeCircle> blink::BasicShapeCircle::Create`](https://source.chromium.org/chromium/chromium/src/+/master:third_party/blink/renderer/core/style/basic_shapes.h;l=136)

---

# Named Parameter / Fluent API

### **Problems**

* C++ supports positional parameters only
* Method takes too many parameters – ambiguity
* Numerous setters — OK, but can be more expressible/fluent

``` c++
struct File {
  File(string path,
       bool read_only,
       bool create_if_missing,
       bool append_if_writing,
       int block_size,
       bool unbuffered,
       bool exclusive_access) {
       // phew!
   }
};
```

---

## Method Chaining

``` c++
struct File(const FileOpener& params);

struct FileOpener {
  FileOpener(std::string path) : path_(path) {
  }

  // NOTE: return type & last statement
  FileOpener& create_if_missing() {        FileOpener& block_size(int new_size) {
    this->create_if_missing_ = true;         this->block_size_ = new_size;
    return *this;                            return *this;
  }                                        }

  // make it so!
  File create() {
    return File(*this);
  }

  // have meaningful defaults
  std::string path_;
  int block_size_ = 1024;
  bool create_if_missing_ = true;
  bool append_if_writing_ = false;
  // ...
};

File f = FileOpener("README.md")
           .block_size(2048)
           .read_only()
           .create_if_missing()
           .create();
```

---

* Most common usage of _method chaining_: `std::cout << a << b << c;`

* Order of the setters don’t matter, usually – ~~positional~~ named parameter

* [Fluent Interface][] — term coined by _Martin Fowler_ and _Eric Evans_

* Sometimes called the _Builder_ pattern

### Examples

- [`content::ChildThreadImpl`](https://source.chromium.org/chromium/chromium/src/+/master:content/child/child_thread_impl.h;l=298)
- [`chrome::internal::MenuItemBuilder`](https://source.chromium.org/chromium/chromium/src/+/master:chrome/browser/ui/cocoa/main_menu_builder.h;l=47)


[Fluent Interface]: https://www.martinfowler.com/bliki/FluentInterface.HTML

---

# Init Method **(Anti-Pattern)**

**Problem**: Cyclic dependency between objects


``` c++
class Notifier {
  Notifier() {  // code here may call Timer::TimeUp  }
};

base::Timer(RepeatingClosure notifier_method) {
  // notifier_method operates on a Notifier object
}
```

--

**Problem**: Need to call `virual` during construction!

``` c++
class Base {
 public:
   Base();
   ...
   virtual void foo(int n) const; // often pure virtual
   virtual double bar() const;    // often pure virtual
 };

 Base::Base() {
   foo(42); bar();  // these will NOT be dispatched dynamically!
 }
```

--

**Problem**: Construction may fail... ~~throw an~~ no, I can’t throw exceptions!

---

## Two-Phase Initialization

``` c++
Notifier::Notifier() {
  // return default constructed object; uninitialized yet!
  // simple memory allocation and zeroing out of members
}

bool Notifier::Init() {  // can indicate failure with return type
  // OK to call virtual methods; object fully constructed
}

// dependency loop broken
auto doc_notifier = Notifier();
auto timer = base::Timer(base::TimeDelta::FromMilliseconds(100),
                         base::BindRepeating(&Notifier::TimeUp,
                                             doc_notifier->GetWeakPtr()));
doc_notifier.Init();
```

--

* Every method should now check if the object is initialized — yuck! 🤮

* Every call to constructor must be paired with a call to `init` 🤦

* Don’t use it, unless you absolutely have to 👻

--
* **Alternatives**
  - Virtual Constructor
  - Builders with Fluent Interface
  - Crash and burn; if you can’t even construct an object, you’ve much bigger problems

---

# Virtual Constructor

``` c++
class Shape {
public:
  virtual ~Shape() { }                 // A virtual destructor
  virtual void draw() = 0;             // A pure virtual function
  virtual void scale() = 0;
  // ...
  virtual Shape* clone()  const = 0;   // Uses the copy constructor
  virtual Shape* create() const = 0;   // Uses the default constructor
};

class Circle : public Shape {          class Rect : public Shape {
public:                                public:
  Circle* clone()  const {               Rect* clone()  const {
    return new Circle(*this);              return new Rect(*this);
  }                                      }
  Circle* create() const {               Rect* create() const {
    return new Circle();                   return new Rect();
  }                                      }
};                                     };

void userCode(Shape& s1)
{
  Shape* s2 = s1.clone();
  Shape* s3 = s1.create();
}
```
--
[_Covariant Return Type_][covariant] of a method is one that can be replaced by a "narrower" type when the method is overridden in a subclass e.g. `Shape` and `Circle`, `Shape` and `Rect`

[covariant]: https://en.wikipedia.org/wiki/Covariant_return_type

---

# References

* ISO C++ FAQ 🌟
  - [Named Constructor](https://isocpp.org/wiki/faq/ctors#named-ctor-idiom)
  - [Method Chaining](https://isocpp.org/wiki/faq/references#method-chaining)
  - [Named Parameter](https://isocpp.org/wiki/faq/ctors#named-parameter-idiom)
  - [Virtual Constructor](https://isocpp.org/wiki/faq/virtual-functions#virtual-ctors)
* _More C++ Idioms_ — WikiBooks 🌟
  - [Erase-Remove](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Erase-Removex)
  - [Named Constructor](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Named_Constructor)
  - [Two-phase Initialization](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Calling_Virtuals_During_Initialization)
  - [Virual Constructor](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Virtual_Constructora)
* RIP Tutorial
  - [Builder / Fluent API](https://riptutorial.com/cplusplus/example/30166/builder-pattern-with-fluent-api)
* Wikipedia
  - [Method Chaining](https://en.wikipedia.org/wiki/Method_chaining)
  - [Fluent Interface](https://en.wikipedia.org/wiki/Fluent_interface)
