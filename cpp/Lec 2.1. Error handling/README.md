 # Error handling. Exceptions. Exception safety.
 # Errors
 Что считать ошибкой во время выполнения некоторой функции:
* Нарушение постусловий функции
* Нарушения предусловий других вызываемых функций
* Неудачное восстановление инварианта класса (для неприватных методов)

У функций есть контракт:
* узкий - пишем в комментариях, доках. Например, квадратные скобки у вектора (при обращении не проверяется выход за границы). 
* широкий - проверяется программой. Предполагается, что программа отрабатывает корректно (в том числе сообщает об ошибке) на любых входах. Например, функция at у вектора

Инвариант - фиксированный набор свойств.

Примеры
* узкий контракт
```c++
// pre-condition: h >= 0, w >= 0
// post-condition: area >= 0
int area(int h, int w) {
    return h * w;
}

int use_area() {
    int h = get_h();
    int w = get_w();
    return area(h, w);
}
```
* Нарушение инварианта - нет проверок на границы 
```c++
class GeoPoint {
void move(double lat_direction, double lon_direction
{
    lat_ += lat_direction;
    lon_ += lon_direction;
}
private:
double lat_; // [-90; 90]
double lon_; // (-180; 180]
};
```
 # Handling errors
 ## Способы обработки ошибок
* использование глобальной переменной
* возврат кода ошибки (out аргументы)
* завершение программы (terminate, exit. лучше логировать)
* исключения
* использование std::optional
 ## Проверка условий в коде
* static_assert — проверка во время компиляции
* assert — проверка во время исполнения (в дебажной сборке)

логику в assert вносить не нужно - он сохраняется только в дебаге и вырезается из релиза
 ## Useful things
* Логирование (помогает, когда нельзя подлезть дебагером) можно делать boost, cout
* Debugger (плох в релизной сборке, т.к. код не совпадает)
* Crash reporting: stacktraces (стек функций при падении программы), memory dump
 # Exceptions
* \+ Компактный код
* \+ Механизм распростренения поддержан рантаймом языка
* \- Non-zero overhead: 
1. исполняется много дополинтельного кода (очистка стека и тд),
2. дополнительный код, который занимает дополнительную память  
но если не бросаем исключение, то ничего страшного)  
ПОЭТОМУ ЕГО НЕ НУЖНО ВСТРАИВАТЬ В ЛОГИКУ
* \- Проблемы с ABI (binary interface)   
Пример ABI: если файлы из одного проекта собраны разными компилаторами -> линкер может не собрать, т.к. стандарт  не фиксирован и реализация может быть разная 
 
 Выброс исключения - синтаксическиф сахар над вызовом последовательности функций из некоторой библиотеки. Они могут быть различные у компиляторов.
 ## Throwing exceptions
 ```c++
throw expression
throw //просто пробросить дальше по стеку текущее исключение
 ```
* можно выбросить объект любого типа
* пользовательский класс исключения наследуется от
std::exception (idiomatic way)

* исключения из стандартной библиотеки: <stdexcept> (можно вместо создания своего взять оттуда)
 ## Stack unwinding разомотка стека
 Если брошено исключение - проходимся вверх по стеку вызовов. Если там нет ни одного обработчика - сразу std::terminate
 Если есть перехватчик catch
 ```c++
struct T {
    int i;
    ~T() { std::cout << "~T" << i << "\n"; }
};
void f() {
    T t{1}; 
    throw std::runtime_error("error");
}
void g() {
 try {
    T t{2};
    f();
 } catch (...) { // значит перехватывает любые исклюения
 throw;
 }
}
int main() {
    T t{3};
    g();
}
>> 
t(3)
t(2)
t(1)
error

out:: 1 (проверить)
```
размотка стека - объекты ниже обработчика удаляются
 ## Catching exceptions syntax
### 1. try-block
* можно ловить как по ссылке, так и по значению
* точно использует RTTI
* не забываем перехватывать (приимать в catch) по константной сслке!
```c++
try {
f();
} catch (const std::overflow_error& e) {
// this executes if f() throws std::overflow_error (
} catch (const std::runtime_error& e) {
// this executes if f() throws std::underflow_error
} catch (const std::exception& e) {
// this executes if f() throws std::logic_error (bas
} catch (...) {
// this executes if f() throws std::string or
// int or any other unrelated type
}
```
 ### 2. function-try-block
 * можно обернуть список инициализации
 * можно обернуть функцию снаружи
 ```c++
struct S {
std::string m;
S(const std::string& arg) try : m(arg, 100) {
    std::cout << "constructed, mn = " << m << '\n';
} catch(const std::exception& e) {
    std::cerr << "arg=" << arg
<< " failed: " << e.what() << '\n';
} // implicit throw; here
};
```
 # noexcept operator & noexcept specifier

 ## noexcept specifier
спецификатор - пишем в конце метода/функции
* точно не бросает, а если бросит - сразу terminate без размотки стека
* оптимизации компилятора 
* ускорение алгоритмов
* помощь в обеспечении гарантий безопасности исключений
* можно писать у лямбды тоже
```c++
template <class T>
struct UniquePtr {
UniquePtr(UniquePtr&& other) noexcept
: resource_(other.release())
{}
T* release() { /* implementation */ }
private:
T* resource_;
};
void nothrow_func() noexcept { /*body*/ }
int main() {
auto f = [](int i) noexcept { return i; };
f(1);
}
```
Пример: reserve  у std::vector (задаем capacity). Если исключение происходит:
* если при выделении - ничего страшного, объект в валидном состоянии
* при перемещении - конструктор перемещения noexcept, все ок. move без noexcept вообще нельзя: один смували нормально, а второй с ошибкой - испортили состояние исходного вектора и при ошибке не сможем откатить назад. Поэтому все копируется (что долго, но сохраняет состояние)
* если создавать свой тип, то все с мув семантикой нужно делать noecept (тогда веткор будет гораздо быстрее работать)
 ## const_expression
 const_expression - Значение известно на этапе компиляции. можно указывать, noexcept ли метод в зависимотсти от этого флага
```c++
 void f() noexcept(true) { }
void g() noexcept(false) { }
...
#include <type_traits>
template <class T, class U>
void assign(T& dest, const U& src)
noexcept(std::is_nothrow_assignable_v<T, U>)
{
dest = src;
}
```
## noexcept operator
Во время компиляции проверяет может ли expression (выражение, мб не функция) выбросить
исключение. Возвращает bool. Вызова нет (т.к. этап компиляции)
```c++
void f() noexcept(true) {}
void g() noexcept(false) {}
int main() {
static_assert(noexcept(f()) == true);
static_assert(noexcept(g()) == false);
}
```
### А если есть параметры?
```c++
#include <type_traits>
template <class T, class U>
void assign(T& dest, const U& src)
noexcept(std::is_nothrow_assignable_v<T, U>)
{
dest = src;
}
template <class T, class U>
void assign2(T& dest, const U& src)
noexcept(noexcept(std::declval<T>() = std::declval<U>()))
{
dest = src;
}
```
внешний - спецификатор, а внутренний - оператор (вернет)
 ## non-noexcept & stl
 * Для пользоватлеьских типов в конструкторах перемещения следует писать noexcept!
 ```c++
 size_t cost = 0;
struct T {
 T() = default;
 T(const T&) { cost += 100; }
 T(T&&) noexcept { cost += 2; }
 T& operator=(const T&) { cost += 150; return *this; }
 T& operator=(T&&) noexcept { cost += 3; return *this; }
};
int main() {
 std::vector<T> ts(32); // 32 items
 assert(ts.size() == 32);
 cost = 0;
 ts.push_back(T());
 std::cout << cost; // good
}
 ```
 # Exception-safety (гарантии безопасности исключений)
* Nothrow exception guarantee
* Strong exception guarantee - вызываем для калааса функцию с ичключением - состояние класса дб в том же ввиде, что до исключения
* Basic exception guarantee - после ислкючения класс будет в каком-то валидном состоянии (мб не в том, в каком был)
* No exception guarantee - утечки
 ## Nothrow exception guarantee
* Функции всегда выполняются успешно (не выбрасывают исключения)
* акая гарантия ожидается от всех функций, вызывающихся при
"размотке" стека. Деструкторы в том числе (по умолчанию. можно вручную написать, но тогда будет вызов noexcept, при размотке стека будет вызове деструктора и еще одна исключение... ).
* Обычно такая гарантия ожидается у move-конструкторов/
операторов, swap-функций.
## Strong exception guarantee
* Исключение в функции приведет программу в состояние до
вызова этой функции.
* Выполнение функции можно рассматривать как транзакцию.
## Basic exception guarantee
Выброс исключение оставляет программу в валидном
состоянии:
* инварианты сохранены
* утечки отсутствуют
* STL: Все контейнеры реализуют по крайней мере эту гарантию
## No exception guarantee
Все плохо:
* утечки ресурсов
* нарушены инварианты
* Дальшейшее выполнение программы неопределено.
 # exception_ptr 
умный указатель для исключений

при выбросе исключения оно аллоцируется где-то в тред локал сторадже

```c++
void handle_eptr(std::exception_ptr eptr) // passing by value is ok
{
 try {
 if (eptr) { //??
    std::rethrow_exception(eptr);
 }
 } catch(const std::exception& e) {
    std::cout << "Caught exception \"" << e.what() << "\"\n";
 }
}
int main()
{
 std::exception_ptr eptr;
 try {
    std::string().at(1); // this generates an std::out_of_range
 } catch(...) {
    eptr = std::current_exception(); // capture
 }
 //из catch  уже вышли, а исключение еще держим в указателе и можем бросить позже (даже в другой поток)
 handle_eptr(eptr);
} // destructor for std::out_of_range called here, when the eptr is destructed
```
 ##
 ##
 ##

