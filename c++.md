```c++
namespace std {
  ///
}

std::move()
A&&
using size_t = unsigned int;

template<typename C>
using Element_type = typename C::value_type;


template<typename T,typename... Tail>
void(T head,Tail... tail){
  g(head);
  f(tail...)
}

void f() {}


template<typename T>
class Less_than{
  const T val;
public:
  Less_than(const T& v):val(v) {}
  bool operator()(cont T& x) const  {return x < val;}
};

=default
=delete

// move

Vector::Vector(Vector&& a)
:elem{a.elem},
sz(a.sz)
{
  a.elem = nullptr;
  a.sz = 0;
}
Vector& operator=(Vector&& a);


// copy
Vector(const Vector& a);
Vector& operator=(const Vector& a);


unique_ptr
shared_ptr


```
