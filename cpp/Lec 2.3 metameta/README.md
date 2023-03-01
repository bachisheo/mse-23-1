# Metaprogramming.SFINAE
requires

# Метапрограммирование
Суть: программа, порождающая программу, совершающая
полезные действия во время компиляции

## Что дают
* переиспользование кода
* расширение синтаксиса
* эффективные реализации/предподсчеты
* проверки времени компиляции — static_assert, concepts
* * if_constexpr - ветка выбирается на этапе компиляции
staticassert в ветке, которая false - падает
* * concepts описываем свойтсва типов и ожидаем их выполнения в шаблоне
> \<class T> -> \<Container T>(должны быть begin, end, size...) это можно проверить SFINAE, но концептами красивее

## Метафункции
Функции **времени компиляции** над метаданными программы
(над сведениями об ее элементах). принимает и возвращает метаданные (типы/интегральные константы)

* sizeof - размер класса
* typeid (для неполиморфного класса, т.е. без виртуальных таблиц. Для полиморфных нужен RTTI)
* decltype 
* * для entity - тип переменной, который ей задан
* * expression - еще и тип категории значений (rvalue, lvalue)

## Шаблоны как метафункции
у реализации шаблона на входе — аргументы
на выходе — результат instantiation
мемберы — значения времени компиляции
результат работы часто значение или тип (один) — где-то
внутри
template <size_t N>
struct fib {
static const size_t value = ...
};
int main() {
// входной параметр + результат
cout << fib<10>::value;
}
### Шаблонный хелпер
template <size_t N>
constexpr (или const? проверить) size_t fib_v = fib<N>::value;

# Приемы метапрограммирования
## Рекурсия по множеству аргументов из variadic templates
Кортеж — хранение всех элементов из списка в одной
структуре
Обобщение std::pair, где достаточно T1 first; T2 second;
```c++
template <typename...>
struct tuple_element;

template <typename Head, typename... Tail>
struct tuple_element <Head, Tail ...> {
Head  data;
tuple_element<Tail...> rest;

template size_t<N>
//we don't know type 
//есть два подхода
//1. auto
// 2. сделать метафункцию, которая ее выберет
//select_nth_type_t<N, Head, Tail...>
auto get(){
    if const_expr( N == 0){
        return data;
    } else {
        //return rest.get<N-1>(); - так он не понимает, что get это шаблон
        return rest.template get<N-1>();
    }
}
}
//база рекурсии
template <>
struct tuple_element<> {};
//обращение
t.get<1>()
```

## Ветвление
* частичные специализации шаблона класса
* перегрузки шаблона функции по обычным правилам
* \+ правило SFINAE [*] 
* if constexpr ( /* ...​/ ) { / ...​*/ } else { ...​}

# SFINAE
* SFINAE - Substitution Failure Is Not An Error (неудача подстановки - это не ошибка)
* Помогает реализовывать ветвление на этапе компиляции базируясь на перегрузках функции: какая-то из функций не будет релаизована
*  способы реализациии: функции, enable_if, ...
* При перегрузки функции ошибочное инстанцирование
шаблонов — не ошибка компиляции
* * функция просто выбрасывается из списка кандидатов на
наиболее подходящую перегрузку (и не инстанцируется)
* * ⇒ нужна "работающая" альтернатива
* SFINAE только про заголовок функции: ошибки в теле будут
пропущены

```c++
struct HasF{
    void f(){}
};
struct HasNoF{
};
//это - метафункция
//1. По дефолту всегда выбирается true...

template <class T>
struct HasMethodF<HasF>{
    //нужно придумать, как бы обратиться к функции

    //выбирается, когда метод есть
    template<class C>
    static std::true_type select(void(C::*)() = &C::f){
        //void(C::*)() указатель на метод класса C, который возвращает void
        // и не принимает аргументов
    }    
    //выбирается, когда метода нет
    std::false_type select(...){
        //эта функция по приоритету вторая, т.к. она вариадик
        
    }

    static const bool value = decltype<select<T>(nullptr)::value;
}
//2. аргементов нет - выбиратьт нечего

template <class T>
struct HasMethodF<HasF>{
    //нужно придумать, как бы обратиться к функции

    //выбирается, когда метод есть
    template<class C, void(C::*)() = &C::f>
    //является ли &C::f compile time?
    static std::true_type select(){
        //void(C::*)() указатель на метод класса C, который возвращает void
        // и не принимает аргументов
    }    
    //выбирается, когда метода нет
    std::false_type select(...){
        //эта функция по приоритету вторая, т.к. она вариадик
    }
    static const bool value = decltype<select<T>()::value;
}
//3. 
template <class T>
struct HasMethodF<HasF>{
    //нужно придумать, как бы обратиться к функции

    //выбирается, когда метод есть
    template<class C, void(C::*)() = &C::f>
    //является ли &C::f compile time?
    static std::true_type select(void*){
        //void(C::*)() указатель на метод класса C, который возвращает void
        // и не принимает аргументов
    }    
    //выбирается, когда метода нет
    std::false_type select(...){
        //эта функция по приоритету вторая, т.к. она элипсис
         //(не вариадик, вариадик - это в template)
    }
    static const bool value = decltype<select<T>(0)::value;
}
//Как будет работать для нестатического метода? 
//...

// Как будет работать для потомка?

struct ChildOfHasF : HasF{
    //не работает из-за C::*, он имеет тип C::C1::*
    //даже с этим
    using  HasF::f;
    //cant convert from void (HasF::*)() to (ChildOfHasF::*)()
    //т.е. для ребенка все равно думает, что в C = HasF
}
// 4 
template <class T>
struct HasMethodF<HasF>{
    template<class C, void(C::*)() = &C::f>
    static std::true_type select(void*){}    
    std::false_type select(...){}
    static const bool value = decltype<select<T>(0)::value;
}

// // 5 declval 
template <class T>
struct HasMethodF<HasF>{
    template<class C, class U = decltype(std::declval<C>().f())>
    //declval на самом деле ничего не создает, он делает выражение какого-то типа
    //эти штуки работают только к компайл тайм
    //проблема в рантайм: не можем создать этот объект, т.к. не знаем конструкторы
    //можно так
    //    template<class C, class U = decltype(((С*)nullptr).f())>

    static std::true_type select(void*){}    
    std::false_type select(...){}
    static const bool value = decltype<select<T>(0)::value;
}
int main(){
    static_assert(HasMethodF<HasF>::value = true);
    static_assert(HasNoMethodF<HasF>::value = false);
    static_assert(HasNoMethodF<int>::value = false);
    void (HAsF::*ptr)() = &ChildOfHasF::f;
    void (ChildOfHasF::*ptr2)() = &ChildOfHasF::f;
    static_assert(HasMethodF<HasF>::value = true);

}

```
template <class T>
std::enable_if_t<condition, int> f(){
    return 3;
    //в свинае эту штуку нужно тащит в шаблон, чтобы функция вообще не создавалась
}

Проверка наличия метода у класса
```c++
template <class T>
struct is_f_with_strict_signature_defined {
// might be substitution failure
template <class Z, void (Z::*)() = &Z::f>
struct wrapper {};
template <class C>
static std::true_type check(wrapper <C> * p);
template <class C>
static std::false_type check (...);
static const bool value = decltype(check<T>(0))::value;
};
```

# Возможности STL
* типы-значения: std::integral_constant
> std::true_type и std::false_type
* SFINAE-селекторы: std::void_t, std::enable_if_t
<type_traits>
создаем класс, который по умолчанию фалзе, создаем второй класс с конструтором типа мембера (которыйпри этом зануляется void_t)


изменения типов: std::add_pointer_t<T> и др.
проверки типов: std::is_pointer_v<T>, std::is_same_v<T1,
T2> ...​
категории итераторов в <iterator>

# AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```c++
template <typename T, typename = void>
struct iterator_trait
: std::iterator_traits<T> {};
 
template <typename T>
struct iterator_trait<T, std::void_t<typename T::container_type>>
: std::iterator_traits<typename T::container_type::iterator> {};
```
Зафиксируем вопросы: 
1. является ли &C::f compile time? (да, но почему?)
2. что делать со static?
3. наследование работает с auto, но будут ли ложные срабатывания на поля?
4. как еще можно починить наследование?

# ПРАКТИКА

```c++
template <class T>
class MyTemp{
    static const size_t value = 0;
    using type = int; 
}
MyTemp<T>::type
// мы не знаем, что такое, пока не инстанцируем класс (мы не знаем тип)
// а еще не знаем, тип это или валью. по умолчанию думает, что (статик?)валью
//нужно подсказать ему так
typename MyTemp<T>::type

MyTemp<T>::value 
// работает нормально, вычисление типа откладывает на "попозже"

```

Почему с шаблонами долго компилируется? при изменении все файлы изменяются, модулю пока не помогают

FACTORIAL в разных техниках
## constexpr
```c++
#include <type_traits>
template<int>struct F;

template<>
struct F<0>{
    static constexpr int value = 1;
}
template <int N>
struct F{
    static //существует без экземпляра класса
    constexpr //(считай в compiletime)
    int value = N * F<N-1>::value;
}
```
## using
```c++
template<class T> 
class PointerToObject {};

template<class T, size_t N> 
class PointerToArray {};  // T[N]

template<class T> 
class PointerToUnboundArray {};  // T[]

//1.object, by default
template<class T>
struct PointerHelper{
    using type = PointerToObject<T>;
}

//2. to array. STATIC ARRAY! type T[x]
//int a[3];
// std::string str[42];
template<class T, size_t N>
struct PointerHelper<T[N]>{
    using type = PointerToArray<T, N>;
}

//3. to unbounded array! type T[]
//чем T[] лучше *T?  знаем, что удалять через delete[]
template<class T>
struct PointerHelper<T[]>{
    using type = PointerToUnboundArray<T, N>;
}

template<class T> 
using Pointer = PointerHelper<T>::type;  // выберет одну из реализаций
static_assert(std::is_same_v<Pointer<int[]>>,PointerToUnboundArray<int>)
////// штука ниже не имеет смысла
template<class T, class N> struct PointerHelper<T[N]>
```
Среди перегрузок функций выбирают более конкретную (где нужно меньше кастов типов)

Среди перегрузок паттернов тоже

## std::conditional_t, ветвление
Напишите функцию, находящую тип с максимальным размером
```c++
template<class A, class B> 
struct Maximal { 
    using type = typename std::conditional_t<(sizeof(A) > sizeof(B)), A, B>;
    };
    static_assert(std::is_same_v<Maximal<int, char>::type, int>);
    static_assert(std::is_same_v<Maximal<int, std::vector<int>>::type, std::vector<int>>);

```
## манипуляции над списком типов
Напишите функцию, возвращающую std::tuple<T…​> (для удобства) с типами внутри в обратном порядке

```c++
template<class Head, class ...Tail> struct RevertedTypesTuple {
    using type = ???
};

template<class ...Tail> struct RevertedTypesTuple {
    using type = ???
};

static_assert(
    std::is_same_v<
        typename RevertedTypesTuple<int, char>::type,
        std::tuple<char, int>
    >
)
```
техники:

variadic templates + рекурсия по множеству элементов

[*] decltype, declval

⇒ преобразование типов

⇒ аналог объекта-списка_типов + revert над ним (см задачу 1)

## enable_if
bool, type

хранит memeber_type, если true

если не true, то не хранит (вообще ничего)

реализация: 
```c++
```

идея: написать несколько вариантов, каждый из которых будет проверять свой набор случаев
```c++
template<typename T, std::is_pointer>
int foo(T) {return std::is_pointer_v<T>;}
//1. не сработает для int
int foo(T t) {*t; return std::is_pointer_v<T>;}

//2. 
template <typename T, bool Condition>
struct FooImpl;

template <typename T>
struct FooImpl<T, true>;

template <typename T>
struct FooImpl<T, false>;

teplate<class T>
using Foo = FooImpl<T, std::is_pointer_v<T>>::value;
//тут велью, оно должно быть статик, и известно только в компайл тайм ((

//3
template <
class T, 
class = std::enable_if_t<std::is_pointer_v<T>, bool> = true>
int foo(T t){t*; return std::is_pointer_v<T>;}
//тут нужно явно писать вторую вариацию с отрицанием, т.к. иначе 
//реализации считаются одинаковыми (там же по умолчанию задан class)
//это плохо тем, что мы туда можем что-то явно подставить (какой-нибудь char)
template <
class T, 
class = std::enable_if_t<std::is_pointer_v<T>, bool> = false>
int foo(T t){t*; return std::is_pointer_v<T>;}


int main(){
    std::cout << foo(1);  
    //cout:: 0
    int * p;
    std::cout << foo(p);
    //cout:: 1

}
```