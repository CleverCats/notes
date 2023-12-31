C++11有一些这样的改善，这种改善保证写出的代码比以往任何时候的执行效率都要好。这种改善之一就是生成常量表达式，允许程序利用编译时的计算能力。假如你熟悉模板元编程，你将发现constexpr使这一切变得更加简单。假如你不知道模板元编程，也没什么。constexpr使我们很容易利用上编译时编程的优势。

常量表达式主要是允许一些计算发生在编译时，即发生在代码编译而不是运行的时候。这是很大的优化：假如有些事情可以在编译时做，它将只做一次，而不是每次程序运行时。需要计算一个编译时已知的常量，比如特定值的sine或cosin？确实你亦可以使用库函数sin或cos，但那样你必须花费运行时的开销。使用constexpr，你可以创建一个编译时的函数，它将为你计算出你需要的数值。用户的电脑将不需要做这些工作。


/*假如你将一个成员函数标记为constexpr，则顺带也将它标记为了const。如果你将一个变量标记为constexpr，则同样它是const的。但相反并不成立，一个const的变量或函数，并不是constexpr的。*/



constexpr初探
为了使函数获取编译时计算的能力，你必须指定constexpr关键字到这个函数。

constexpr int multiply (int x, int y)
{
    return x * y;
}

// 将在编译时计算
const int val = multiply( 10, 10 );
除了编译时计算的性能优化，constexpr的另外一个优势是，它允许函数被应用在以前调用宏的所有场合。例如，你想要一个计算数组size的函数，size是10的倍数。如果不用constexpr，你需要创建一个宏或者使用模板，因为你不能用函数的返回值去声明数组的大小。但是用constexpr，你就可以调用一个constexpr函数去声明一个数组。

constexpr int getDefaultArraySize (int multiplier)
{
    return 10 * multiplier;
}

int my_array[ getDefaultArraySize( 3 ) ];
constexpr函数的限制
一个constexpr有一些必须遵循的严格要求：

函数中只能有一个return语句（有极少特例）
只能调用其它constexpr函数
只能使用全局constexpr变量
注意递归并不受限制。但只允许一个返回语句，那如何实现递归呢？可以使用三元运算符（?:)。例如，计算n的阶乘：

constexpr int factorial (int n)
{
    return n > 0 ? n * factorial( n - 1 ) : 1;
}
现在你可以使用factorial(2)，编译器将在编译时计算这个值，这种方式运行更巧妙的计算，与内联截然不同。你无法内联一个递归函数。

constexpr函数还有那些特点？

一个constexpr函数，只允许包含一行可执行代码。但允许包含typedefs、 using declaration && directives、静态断言等。

constexpr和运行时
一个声明为constexpr的函数同样可以在运行时被调用，当这个函数的参数是非常量的：

int n;
cin >> n;
factorial( n );
这意味着你不需要分别写运行时和编译时的函数。

编译时使用对象
假如你有一个Circle类：

class Circle
{
    public:
    Circle (int x, int y, int radius) : _x( x ), _y( y ), _radius( radius ) {}
    double getArea () const
    {
        return _radius * _radius * 3.1415926;
    }
    private:
        int _x;
        int _y;
        int _radius;
};
你希望在编译期构造一个Circle接着算出他的面积。

constexpr Circle c( 0, 0, 10 );
constexpr double area = c.getArea();
事实证明你可以给Circle类做一些小的修改以完成这件事。首先，我们需要将构造函数声明为constexpr，接着我们需要将getarea函数声明为constexpr。将构造函数声明为constexpr则运行构造函数在编译期运行，只要这个构造函数的参数为常量，且构造函数仅仅包含成员变量的constexpr构造（所以默认构造可以看成constexpr，只要成员变量都有constexpr构造）。

class Circle
{
    public:
    constexpr Circle (int x, int y, int radius) : _x( x ), _y( y ), _radius( radius ) {}
    constexpr double getArea ()
    {
        return _radius * _radius * 3.1415926;
    }
    private:
        int _x;
        int _y;
        int _radius;
};
constexpr vs const
假如你将一个成员函数标记为constexpr，则顺带也将它标记为了const。如果你将一个变量标记为constexpr，则同样它是const的。但相反并不成立，一个const的变量或函数，并不是constexpr的。

constexpr和浮点数
到这里我们讲到的constexpr功能都可以通过模板元编程实现。但constexpr支持的一项能力是可以计算浮点型的数据。因为double和float不是有效的模板参数，你不可以轻易的通过模板编译期计算浮点数的值。而constexpr允许编译期计算浮点型数据。

权衡constexpr
C++开发者早就深受修改一个头文件则引发重新编译导致编译缓慢的困扰。而constexpr可能引入增加编译时间的风险，但也有一些技术去降低这种风险。首先，因为constexpr函数相同的参数会输出相同的结果，所以它们可以被memoized，事实上GCC已经支持memoization。

因为可以对constexpr函数memoize,所以用constexpr函数替换模板函数的地方，(编译）性能不会变得更坏，但代码会变得清晰。事实上，替换掉一部分模板实例，编译会显著加快。

最后，标准允许编译器去限制递归函数的级数。这样可以限制深度递归的编译性能损耗。