#include<iostream>
using namespace std;
class Base    //在类中生成一个虚函数表(可以看作静态函数指针数组,存储虚函数指针),还会与一个虚函数指针变量(在构造函数中被编译器插入代码初始化)
{
public:
	//这个虚函数表相当于一个数组,下标0,1,2分别代表这个三个虚函数的首地址
	//即父类虚函数表中包含基于父类的f(),g(),h();
	virtual void f(){ cout << "Base f() Called" << endl; }
	virtual void g(){ cout << "Base g() Called" << endl; }
	virtual void h(){ cout << "Base h() Called" << endl; }
};
class Derive:public Base //在类中生成一个虚函数表(类似静态函数数组)与一个虚函数指针(Vptr)变量
{						 //编译器会自动的在构造函数中插入设置Vptr的指向
						 /*A).首先，在基类构造函数中，Vptr被设置为指向基类的VTABLE
  							  B).之后，在子类构造函数中，Vptr被设置为指向子类的VTABLE
   							 注意，这里只存在一个Vptr，在子对象的构建过程中被重写（即重新指定指向）。
  						     而对于多重继承，有类似的过程，不过子类对象中存在不止一个Vptr。*/
public:
	//重写父类的虚函数,使得继承下来虚函数中,该虚函数被子类覆盖(也就是该虚函数地址被子类覆盖)
	//此时子类的生成的虚函数表包含父类继承下来的f(),h(),与子类的g()
	//注意:在单继承下,只存在一个Vptr(由父类继承下来),只是在子类的构造函数中,Vptr在构造函数中被指向的子类的虚函数表,在父类构造函数中则指向父类虚函数表
	virtual void  g(){ cout <<"Derive g() Called （重写以覆盖父类虚函数g()）" << endl; }
	//这个virtual无论加不加都为虚函数,因为在父类中它是虚函数
};
int main()
{
	Derive dir;
	//将派生类的this指针(对象首地址)(这里Base与Derive的this地址相同)提取出来这里转换为占4字节的整形指针,使*Vtable正好取出4字节的完整的虚函数表指针
	int*Derive_this = reinterpret_cast<int*>(&dir);
	//(*Vtable)取出该类对象中占4字节的Vptr(虚函数指针),将去转化为4字节的整形指针,此时Dive_vptr即为指向本类中虚函数表首地址的指针变量
	int*Div_vptr = (int*)(*Derive_this);
	cout << "-----------------Begin Show Derive::Vtable[index] Contain------------------" << endl;
	for (int i = 0; i < 3; ++i)
	{
		printf("Derive Vptr[%d] = %p\n",i,Div_vptr[i]);
	}
	cout << "------------------------------------End------------------------------------" << endl;
	typedef void(*Func)(void);
	Func m_f = (Func)Div_vptr[0];
	Func m_g = (Func)Div_vptr[1];
	Func m_h = (Func)Div_vptr[2];
	m_f();  m_g();  m_h();
	cout << "-------------------------------------------------------------------------------" << endl;
	//构造Base对象,编译器在构造函数中插入代码使Vptr指向父类的虚函数表首地址
	Base bas;
	int*Base_this = reinterpret_cast<int*>(&bas);
	//(*Vtable)取出该类对象中占4字节的Vptr(虚函数指针),将去转化为4字节的整形指针,此时Dive_vptr即为指向本类中虚函数表首地址的指针变量
	int*Base_vptr = (int*)(*Base_this);
	cout << "-----------------Begin Show Derive::Vtable[index] Contain------------------" << endl;
	for (int i = 0; i < 3; ++i)
	{
		printf("Base Vptr[%d] = %p\n", i,Base_vptr[i]);
	}
	cout << "------------------------------------End------------------------------------" << endl;
	cout << "-----------------Begin Show Base::Vtable[index] Contain------------------" << endl;
	typedef void(*Func)(void);
	Func m_f_2 = (Func)Base_vptr[0];
	Func m_g_2 = (Func)Base_vptr[1];
	Func m_h_2 = (Func)Base_vptr[2];
	//调用父类虚函数表中的 m_f,m_g,m_h
	m_f_2();  m_g_2();  m_h_2();
	cout << "------------------------------------End------------------------------------" << endl;
	system("PAUSE");
	return 0;
}