# Generics
Позволяет единообразным образом работать с разными типами
не было до java 5, все было object

* работают не как шаблоны в плюсах: мы не можем перекомпилировать байткод. Вместо этого при компиляции из generic стирается тип, оставляя снаиболее общий (по умолчанию Object)

* что меняется ? инстанцирование ArrayList просто создается, но в точках входа (методы, принимали на вход тип Т) и выхода (возвращали тип Т) насильно кастуется к указанному типу. 
* * на входе - это ка проверка типов на этапе компиляции
* * на выходе - возможность получить результаты нужного типа

* поэтому для примитивных типов generic нет. поэтому их можно боксить/анбоксить в классы-обертки

* поэтому внутри листа нельзя использовать никакие методы, кроме object? 

* Зачем integer как обертки до 5 java? кроме этого 

## Как работает?
## Проблемы?
1. Generic-типы не совместимы по присваиванию 
list<Object> = list<Integer> - это ошибка компиляции, т.к. в object можно класть что угодно, а у меня в исходный - инт   
> вместо этого можно использовать copy - он способен вывести более общий тип
2. обратная совместимость с 5 java (аккуратнее писать и стирать <..>)
3. в дженерик классах нельзя пистаь простой тип new T()... 
* * мы стираем тип
* * у класса может не быть дефолтного конструктора
* * нельзя создавать массивы T[12] - по сути это массив object'ов, нужно указывать явно
```T[] = new Object[]```
4. попвтка передать листь к наиболее общему типу - ошибка копмиляции (дженерик типы не совместимы по присваиванию). чтобы сделать функцию, которая будет принимать любой тип - используем void foo(Collections<?> c). Все, что мы достанем из него - приведется к object и все будет норм.
* * точки выхода от него кастуются к Object, но обратно положить туда object нельзя
* * точки входа от collection<?> кастуются к nullType - это тип, который населен одним жителем (null) и не наследуется от object
```c.add(c.get()) - compilation error``` 

# Часть 2
При компиляции происходит проверка типов и  их стирание до наиболее общего (по умолчанию object). При указании generic типа нельзя писать слово super (T super S). Почему нет? Когда стираем, T стирается до object. Почему хотелось бы? <T, S super T>. А зачем, если можно extends?

А потому что 
```Java
R foldl(List<D>, Function<A, B, C>, E init )
//C -> A
//C -> R
//E -> A
//
```
Е - не дженерик тип, поэтому туда можно передать любой тип или его наследника. Поэтому можно поменять его на А (и любой его неаследник подойдет)

Так же и R можно каставать к любому надтипу С

Можно выкинуть D оставив & extends B

Или B на ? super D

Это мы не рассматриваем, что список может быть нулевой длины. Если может, то   
E наследует R и А   
С наследует R и А   
Разрешит это можно, но только с дополнительными допущениями


1. Generics не совместимы по присваиванию (только если у них совсем один тип)
> т.к. они будут ссылаться на одну область в памяти, и через одну ссылку мы кладем крокодилов, а через другой - достаем интенджеры
2. В методах с дженериками нельзя создавать конструтор по умолчанию (его может не быть) или массив (почему*)
3. Нельзя передавать в параметры generic от object (т.к. присваивание). Решение: collection<?>
Имеет итератор по ?, который кастует объекты к обекту. Точки выхода кастует к Object, а точки входа - nulltype ( т.е. добавить новый элемент в коллекцию <?> мы не можем)
4.  А если хотим положить ограничение на аргументы? (например, принимать только геометрические фигуры с реализацией методов Draw).   
**Решение 1** Можем навесить extends Shape (Shape или его наследников, точки выхода - shape. Вход - тоже nulltype. Shape нельзя добавить в список кружочков)
```Java
```
5. Хотим все элементы массива добавить в коллекцию. Но в методе add упадет, т.к. все точки входа кастуются к nulltype. Решение 1: тольько один дженерик-параметр    
О МАССИВАХ 
```Java      
String[] x;
Object [] y = x;
```
Массив хранит не объекты, а ссылки. Могу ли я смотреть на ссылки на стринги, как на object? Да, но ведь туда можно писать крокодилов (будет каст экцепшн). Тогда почему в коллекциях нельзя? 
a. Массивы уже были, туда тяжело добавить новую логику
b. На дженериках такая проверка откидывает кучу проблем    
Тогда какой тип выбирать? 
a. Если есть дженерик-коллекция - выбираем ее тип
b. А если две коллекции типа Т? Мы берем из одной коллекции и добавляем в другую. Выбираем первую коллекцию, если вторая имеет другой тип - падаем.
6. bounded type argiment 
```

```
Плохо   
выводит тип дважды   
постоянно провверяем зависимости между типами

7. Bounded Wildcard
то же,что 6. Но типа один, а второй - вопросик с его наследником.   
**Плохо**:  что мы вытащили из коллекции мы не можем положить обратно (т.к. не имеем явного имени типа-наследника)   
**Хорошо**: все любят, хорошо читаемо
8. Хотим посчитать максимум на коллекции. Как сравнивать?     
```java
iterface Complalable<t>{
    int compareTo(T t);
}

class A implements Compalable<A>{

}
//maximum
<T extends Comparable <T>>
T max(Collection<T> c){
    ...
}
class T implements Comparable<Object>{...}
List<Test> tl; 
// Ошибка : не можем присвоить к <object> к <t> - не присваиваются по типам
Test t = max(tl) 
```    
Но если крокодил сравнивается с животными и он его наследник -то и с животными можно сравнить и с другими крокодилами. Отметить это явно.
```Java 
List<?> out -> object, in -> nullType
List<? extends> out -> Apple, in -> nullType (как я запишу животного в крокодила?производители производят конкретные сорта яблок. к нему нельзя добавить яблоко другого типа, но все, что он мне дает - я считаю яблоком)
List<? super> out -> object, in -> Apple
``` 