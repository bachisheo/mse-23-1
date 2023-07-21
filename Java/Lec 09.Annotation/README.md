# Аннотации
* Статически расширяет классы (не объекты)
* сохраняются в виде байт кода рядом с классом 
* имеют только методы мб гетеры    
почему? поля можно менять, можно было бы делать final и передавать в конструктор. в принципе без разницы, методы просто визуально красивее
* без параметров
* без исключениц 
* мб возвращать только: примитивный тип string, class, enum или массив указанных типов (?), еще аннотации    
Почему? должны быть известны на этапе компиляции, и быть представимы в виде байт кода, а объекты так не умеют (см требование 2)
```
@Target(ElementType.TYPE) - к чему может быть приписана
@Retention(RetenrionPolicy.Runtime) - в какой момент времени доступна
public @interface Mammal{
    String sound();
    int color() default 0xffffff;
}
```
```
@Mammal (color = orange, sound = "uuu")
class Giraffe{

}
```

## @Target
Указывает, какой элемент программы будет использоваться аннотауией, мб несколько вариантов - представиоом в виде массива
## @Retention
1. Source   
Существуют только в исходном коде программы и доступна компиляторы при компиляции чтбы что-то проверять и делать кодогенерацию(@Override) 
2. Class (дефолтный вариант)
в .class файле, доступны при загрузке файла, но не доступны при выполнении. Зачем? Чтобы выполнять что-то при загрузке (injection, class loader, di (dependency injection))
3. Runtime 
в .class файле, доступна при выполнении


## Пример
class User{
    ps enum Premission{USER_MNGMT, CONTENT_MNGMT}
    privete list<Pre> permissioms;
}

@Retemrion(Runtime)
@PermReq
User.Permissin() value;

@PermissionRequires(User.Permission.USER_MNGMT)

actionClass.getAnnotation(PermReq.class)
```

* А зачем так сложно? (а как бы выглядела реализация со статическим классом или рефлекшном)
быстрее ,чем статический метод. аннотация лежит в байткоде рядом с классом и быстро подгружается     
> Хочу внутри какого-то экшна и какого-то пользователя получить статический метод, который возвращает разрешения юзера. Это тяжело сделать без рефлекшна, даже если мы знаем, как он называется.
> можно хранит мапу в набор классов, которые для них можно вызвать. это плохо 0 эта инфомрация лежит отдельно от классов
>я могу это вызвать у экземпляра, но у объекта нельзя 
* похода на статические поля/методы, но доступна через другой интерфейс

* аннотация написана сверху, а статический метод где-то в глубине (это хорошо читаемо)

* ее легко менять


```Java
interface Serializer<T> {
    void toStream(T obj, OutputStream out) throws IOException;
}

class MySerializer implements Serializer<MyClass> {
    public void toStream(MyClass obj, OutputStream out) 
    throws IOException {
        throw new UnsupportedOperationException();
    }
}

@interface SerializedBy {
    Class<? extends Serializer> value();
}

@SerializedBy(MySerializer.class)
class MyClass {}

```