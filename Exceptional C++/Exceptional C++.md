[toc]
# 001 Variable Initialization 
**Difficulty: 4 / 10**
How many ways are there to initialize variables? Don't forget to watch out for bugs that look like variable initialization, but aren't.

----

**Problem**

What is the difference, if any, between the following?
``` cpp
SomeType t = u;
SomeType t(u);
SomeType t();
SomeType t;
```

----

**Solution**

Taking them in reverse order:
``` cpp
SomeType t;
```
The variable `t` is initialised using the default ctor `SomeType::SomeType()`.
``` cpp
SomeType t();
```
This was a trick; it might look like a variable declaration, but it's a function declaration for a function `t` that takes no parameters and returns a `SomeType`.
``` cpp
SomeType t(u);
```
This is direct initialization. The variable `t` is initialised using `SomeType::SomeType(u)`.
``` cpp
SomeType t = u;
```
This is copy initialization, and the variable t is always initialised using `SomeType`'s copy ctor. (Even though there's an "`=`" there, that's just a syntax holdover from C... this is always initialization, never assignment, and so `operator=` is never called.)

Semantics: If u also has type `SomeType`, this is the same as "`SomeType t(u)`" and just calls `SomeType`'s copy ctor. If u is of some other type, then this is the same as "`SomeType t(SomeType(u))`"... that is, u is converted to a temporary `SomeType` object, and t is copy-constructed from that.

**Note:** The compiler is actually allowed (but not required) to optimize away the copy construction in this kind of situation. If it does optimize it, the copy ctor must still be accessible.
**[Guideline]** Prefer using the form "`SomeType t(u)`". It always works wherever "`SomeType t = u`" works, and has other advantages (for instance, it can take multiple parameters).



# 002 Temporary Objects
**Difficulty: 5 / 10**
Unnecessary temporaries are frequent culprits that can throw all your hard work \- and your program's performance \- right out the window.

**Problem**
You are doing a code review. A programmer has written the following function, which uses unnecessary temporary objects in at least three places.

How many can you identify, and how should the programmer fix them?

``` cpp
string FindAddr(list<Employee> l, string name) {
    for(list<Employee>::iterator i = l.begin(); i != l.end(); i++) {
        if(*i == name) {
            return (*i).addr;
        }
    }
    return "";
}
```

**Solution**
Believe it or not, these few lines have three obvious cases of unnecessary temporaries, two subtler ones, and a red herring.

``` cpp
    string FindAddr(list<Employee> l, string name)
                    ^^^^^^^1^^^^^^^^  ^^^^^2^^^^^
```

**1 & 2.** The parameters should be const references. Pass-by-value copies both the list and the string, which can be expensive.

   **[Rule]** Prefer passing const& instead of copied values.

``` cpp
    for(list<Employee>::iterator i = l.begin(); i != l.end(); i++)
                                                              ^3^
```

**3.** This one was more subtle. Preincrement is more efficient than postincrement, because for postincrement the object must increment itself and then return a temporary containing its old value. Note that this is true even for builtins like int!


   **[Guideline]** Prefer preincrement, avoid postincrement.

``` cpp
    if(*i == name)
          ^4
```

**4.** The Employee class isn't shown, but for this to work it must either have a conversion to string or a conversion ctor taking a string. Both cases create a temporary object, invoking either operator= for strings or for Employees. (The only way there wouldn't be a temporary is if there was an operator= taking one of each.)    


   **[Guideline]** Watch out for hidden temporaries created by parameter conversions. One good way to avoid this is to make ctors explicit when possible.

``` cpp
    return "";
          ^5
```

**5.** This creates a temporary (empty) string object.

More subtly, it's better to declare a local string object to hold the return value and have a single return statement that returns that string. This lets the compiler use the return value optimisation to omit the local object in some cases, e.g., when the client code writes something like:

``` cpp
    string a = FindAddr(l, "Harold");
```

   **[Rule]** Follow the single-entry/single-exit rule. Never write multiple return statements in the same function.

   **[Note: After more performance testing, I no longer agree with the above advice. It has been revised in Exceptional C++.]**

``` cpp
    string FindAddr(list<Employee> l, string name)
    ^^^*^^
```

\*. This was a red herring. It may seem like you could avoid a temporary in all return cases simply by declaring the return type to be string& instead of string. Wrong (generally see note below)! If you're lucky, your program will crash as soon as the calling code tries to use the reference, since the (local!) object it refers to no longer exists. If you're unlucky, your code will appear to work and fail intermittently, causing long nights of toiling away in the debugger.

   **[Rule]** Never, ever, EVER return references to local objects.

(Note: Some posters correctly pointed out that you could make this a reference return without changing the function's semantics by declaring a static object that is returned on failure. This illustrates that you do have to be aware of object lifetimes when returning references.)

There are other optimisation opportunities, such as avoiding the redundant calls to `end()`. The programmer could/should also have used a `const_iterator`. Ignoring these for now, a corrected version follows:
``` cpp
string FindAddr(const list<Employee>& l, const string& name) {
    string addr;
    for(list<Employee>::const_iterator i = l.begin(); i != l.end(); ++i) {
        if((*i).name == name) {
            addr = (*i).addr;
            break;
        }
    }
    return addr;
}
```



# 003 Using the Standard Library (or, Temporaries Revisited)
**Difficulty: 3 / 10**
You're much better off using standard library algorithms than handcrafting your own. Here we reconsider the example in the last GotW to demonstrate how many of the problems would have been avoided by simply reusing what's already available in the standard library.

----

**Problem**
How many of the pitfalls in GotW #2 could have been avoided in the first place if the programmer had just used a standard-library algorithm instead of handcrafting his own loop? (Note: as before, don't change the semantics of the function even though they could be improved.)

**Recap**
Original flawed version:

``` cpp
string FindAddr(list<Employee> l, string name) {
    for(list<Employee>::iterator i = l.begin(); i != l.end(); i++) {
        if(*i == name) {
            return (*i).addr;
        }
    }
    return "";
}
```

Mostly fixed version, `l.end()` still being called redundantly:

``` cpp
string FindAddr(const list<Employee>& l, const string& name) {
    string addr;
    for(list<Employee>::const_iterator i = l.begin(); i != l.end(); ++i) {
        if((*i).name == name) {
            addr = (*i).addr;
            break;
        }
    }
    return addr;
}
```

----

**Solution**
With no other changes, simply using `find()` would have avoided two temporaries and almost all of the `l.end()` inefficiency in the original:

``` cpp
string FindAddr(list<Employee> l, string name) {
    list<Employee>::iterator i = find(l.begin(), l.end(), name);
    
    if(i != l.end()) {
        return (*i).addr;
    }
    return "";
}
```

Combining this with the other fixes:

``` cpp
string FindAddr(const list<Employee>& l, const string& name) {
    string addr;
    list<Employee>::const_iterator i = find(l.begin(), l.end(), name);
    
    if(i != l.end()) {
        addr = (*i).addr;
    }
    return addr;
}
```

   **[Guideline]** Reuse standard library algorithms instead of handcrafting your own. It's faster, easier, AND safer!



# 004 Class Mechanics
**Difficulty: 7.5 / 10**
How good are you at the details of writing classes? This GotW focuses not only on blatant errors, but even more so on professional style.

----

**Problem**
You are doing a code review. A programmer has written the following class, which shows some poor style and has some real errors. How many can you find, and how would you fix them?
``` cpp
class Complex {
public:
    Complex(double real, double imaginary = 0) 
        : _real(real), _imaginary(imaginary) {};

    void operator+ (Complex other) {
        _real = _real + other._real;
        _imaginary = _imaginary + other._imaginary;
    }

    void operator<<(ostream os) {
        os << "(" << _real << "," << _imaginary << ")";
    }

    Complex operator++() {
        ++_real;
        return *this;
    }

    Complex operator++(int) {
        Complex temp = *this;
        ++_real;
        return temp;
    }

private:
    double _real, _imaginary;
};
```

----

**Solution**
**Preface**
There is a lot more wrong with this class than will be shown here. The point of this puzzle was primarily to highlight class mechanics (e.g., "what is the canonical form of `operator<<`?", "should operator+ be a member?") rather than point out where the interface is just plain badly designed. However, I will start off with one very useful comment, #0...

**0.** Why write a Complex class when one already exists in the standard library? (And, incidentally, one that doesn't have any of the following problems and has been crafted based on years of practice by the best people in our industry? Humble thyself and reuse!)

   **[Guideline]** Reuse standard library algorithms instead of handcrafting your own. It's faster, easier, AND safer!

``` cpp
class Complex {
public:
    Complex(double real, double imaginary = 0)
        : _real(real), _imaginary(imaginary) {};
```

**1.** Style: This can be used as a single-parameter constructor, hence as an implicit conversion. This may not always be intended!

   **[Guideline]** Watch out for silent conversions. One good way to avoid them is to make ctors explicit when possible.

``` cpp
    void operator+ (Complex other) {
        _real = _real + other._real;
        _imaginary = _imaginary + other._imaginary;
    }
```

**2.** Style: For efficiency, the parameter should be a `const&` and "`a=a+b`" should be rewritten "`a+=b`".

   **[Rule]** Prefer passing const& instead of copied values.

   **[Guideline]** Prefer using "`a op= b`" instead of "`a = a op b`" for arithmetic operations (where appropriate; some classes \-\- not any that you wrote, right? \-\- might not preserve the natural relationship between their op and op=).

**3.** Style: `operator+` should not be a member function. If it's a member like this, you can write "`a=b+1`" but not "`a=1+b`". For efficiency, you may want to provide an `operator+(Complex,int)` and `operator+(int,Complex)` too.)

   **[Rule]** Prefer these guidelines for making an operator a member vs. nonmember function: (Lakos96: 143-144; 591-595; Murray93: 47-49)
   - unary operators are members
   - `=` `()` `[]` and `->` must be members
   - `+=` `-=` `/=` `*=` (etc.) are members
   - all other binary operators are nonmembers

**4.** ERROR: operator+ should not modify this object's value. It should return a temporary object containing the sum. Note that this return type should be "`const Complex`" (not just "`Complex`") in order to prevent usage like "`a+b=c`".

(Actually, the original code is much closer to what operator+= should be than it is to operator+.)

**5.** Style: You should normally define `op=` if you define op. Here, you should define `operator+=`, since you defined `operator+`. In this case, the above function should be `operator+=` in any case (with a tweak for the proper return value, see below).

``` cpp
    void operator<<(ostream os) {
        os << "(" << _real << "," << _imaginary << ")";
    }
```

(Note: For a real `operator<<`, you should also do things like check the stream's current format flags to conform to common usage. Check your favourite STL book for details... recommended (if dated) are Steve Teale's "C++ IOStreams Handbook", Glass and Schuchert's "The STL <Primer>", and Plauger's "The (Draft) Standard C++ Library".)

**6.** ERROR: `operator<<` should not be a member function (see above), and the parameters should be "`(ostream&, const Complex&)`". Note that, as James Kanze pointed out, it's preferable not to make it a friend either! Rather, it should call a "print" public member function that does the work.

**7.** ERROR: The function should have return type "`ostream&`", and should end with "`return os;`" to permit chaining (so you can write "`cout << a << b;`").

   **[Rule]** Always return stream references from `operator<<` and `operator>>`.

``` cpp
    Complex operator++() {
        ++_real;
        return *this;
    }
```

**8.** Style: Preincrement should return `Complex&` to let client code operate more intuitively.

``` cpp
    Complex operator++(int) {
        Complex temp = *this;
        ++_real;
        return temp;
    }
```

**9.** Style: Postincrement should return `"const Complex"`. This prevents things like `"a++++"`, which doesn't do what a naive coder might think.

**10.** Style: It would be better to implement postincrement in terms of preincrement.

   **[Guideline]** Prefer to implement postincrement in terms of preincrement.

``` cpp
    private:
        double _real, _imaginary;
    };
```

**11.** Style: Try to avoid names with leading underscores. Yes, I've habitually used them, and yes, popular books like "Design Patterns" (Gamma et al) do use it... but the standard reserves some leading-underscore identifiers for the implementation and the rules are hard enough to remember (for you and for compiler writers!) that you might as well avoid this in new code. (Since I'm no longer allowed to use leading underscores as my "member variable" tag, I'll now use trailing underscores!)

That's it. Here's a corrected version of the program, ignoring design and style issues not explicitly noted above:

``` cpp
class Complex {
public:
    explicit Complex(double real, double imaginary = 0)
        : real_(real), imaginary_(imaginary) {}

    Complex& operator+=(const Complex& other) {
        real_ += other.real_;
        imaginary_ += other.imaginary_;
        return *this;
    }

    Complex& operator++() {
        ++real_;
        return *this;
    }

    const Complex operator++(int) {
        Complex temp = *this;
        ++(*this);
        return temp;
    }

    ostream& print(ostream& os) const {
        return os << "(" << real_ << "," << imaginary_ << ")";
    }

private:
    double real_, imaginary_;
    friend ostream& 
    operator<<(ostream& os, const Complex& c);
};

const Complex operator+(const Complex& lhs, const Complex& rhs) {
    Complex ret(lhs);
    ret += rhs;
    return ret;
}

ostream& operator<<(ostream& os, const Complex& c) {
    return c.print(os);
}
```



# 005 Overriding Virtual Functions
**Difficulty: 6 / 10**
Virtual functions are a pretty basic feature, right? If you can answer questions like this one, then you know them cold.

----

**Problem**
In your travels through the dusty corners of your company's code archives, you come across the following program fragment written by an unknown programmer. The programmer seems to have been experimenting to see how some C++ features worked. What did the programmer probably expect the program to print, but what is the actual result?

``` cpp
#include <iostream>
#include <complex>
using namespace std;

class Base {
public:
    virtual void f(int) {
        cout << "Base::f(int)" << endl;
    }

    virtual void f(double) {
        cout << "Base::f(double)" << endl;
    }

    virtual void g(int i = 10) {
        cout << i << endl;
    }
};

class Derived: public Base {
public:
    void f(complex<double>) {
        cout << "Derived::f(complex)" << endl;
    }

    void g(int i = 20) {
        cout << "Derived::g() " << i << endl;
    }
};

void main() {
    Base    b;
    Derived d;
    Base*   pb = new Derived;

    b.f(1.0);
    d.f(1.0);
    pb->f(1.0);

    b.g();
    d.g();
    pb->g();

    delete pb;
}
```

----

**Solution
First, some style issues:

**1.** `void main()`

This is not one of the legal declarations of main, although many compilers will allow it. Use either "`int main()`" or "`int main(int argc, char* argv[])`".

However, note that you still don't need a return statement (though it's good style to report errors to outside callers!)... if main has no return statement, the effect is that of executing "`return 0;`".

**2.** `delete pb;`

This looks innocuous, and it would be if the writer of Base had supplied a virtual destructor. As it is, deleting via a pointer-to-base without a virtual destructor is evil, pure and simple, and corruption is the best thing you can hope for.

   **[RULE]** Make base class destructors virtual.

**3.** `Derived::f(complex<double>)`

Derived does not overload `Base::f`... it hides them. This distinction is very important, because it means that `Base::f(int)` and Base::f(double) are not visible in Derived! (Note that certain popular compilers do not even emit a warning for this.)

   **[RULE]** When providing a function with the same name as an inherited function, be sure to bring the inherited functions into scope with a `"using"` declaration if you don't want to hide them.

**4.** `Derived::g(int i = 10)`

Unless you're really out to confuse people, don't change the default parameters of the inherited functions you override. (In general, it's not a bad idea to prefer overloading to parameter defaulting anyway, but that's a subject in itself.) Yes, this is legal C++, and yes, the result is well-defined, and no, don't do it. See below for how this can really confuse people.

   **[RULE]** Never change the default parameters of overridden inherited functions.

Now that we have the major style issues out of the way, let's look at the mainline and see whether it does that the programmer intended:

``` cpp
void main() {
    Base    b;
    Derived d;
    Base*   pb = new Derived;

    b.f(1.0);
```

No problem. Calls `Base::f(double)`.

```
    d.f(1.0);
```

This calls `Derived::f(complex<double>)`. Why? Well, remember that Derived doesn't declare "`using Base:f;`", and so clearly `Base::f(int)` and `Base::f(double)` can't be called.

The programmer may have expected it to call the latter, but in this case won't even get a compile error since fortunately(?) `complex<double>` has an implicit(`*`) conversion from double, and so the compiler sees this as `Derived::f(complex<double>(1.0))`.

(\*) as of the current draft, the conversion ctor is not explicit

``` cpp
    pb->f(1.0);
```

Interestingly, even though the `Base* pb` is pointing to a Derived object, this calls `Base::f(double)` because overload resolution is done on the static type (here Base), not the dynamic type (here Derived).

``` cpp
    b.g();
```

This prints `"10"`, since it simply invokes `Base::g(int)` whose parameter defaults to the value \*10\*. No sweat.

``` cpp
    d.g();
```

This prints "`Derived::g() 20`", since it simply invokes `Derived::g(int)` whose parameter defaults to the value \*20\*. Also no sweat.

``` cpp
    pb->g();
```

This prints "`Derived::g() 10`"... which may temporarily lock your mental brakes and bring you to a screeching halt until you realise that what the compiler has done is quite proper(`**`). The thing to remember is that, like overloads, default parameters are taken from the static type (here Base) of the object, hence the default value of 10 is taken. However, the function happens to be virtual, and so the function actually called is based on the dynamic type (here Derived) of the object.

(\*\*) although of course the progammer ought to be shot

If you understand the last few paragraphs (since "Oops!"), then you understand this stuff cold. Congratulations!

``` cpp
        delete pb;
    }
```

And the delete, of course, will corrupt your memory anyway and leave things partially destroyed... see the part about virtual dtors above.



# 006 Const-Correctness
**Difficulty: 6 / 10**
Always use const as much as possible, but no more. Here are some obvious and not-so-obvious places where const should be used \-\- or shouldn't.

----

**Problem**
Const is a powerful tool for writing safer code, and it can help compiler optimizations. You should use it as much as possible... but what does "as much as possible" really mean?

Don't comment on or change the structure of this program; it's contrived and condensed for illustration only. Just add or remove "`const`" (including minor variants and related keywords) wherever appropriate. Bonus Question: In what places are the program's results undefined/uncompilable due to const errors?

``` cpp
class Polygon {
public:
    Polygon() : area_(-1) {}
  
    void AddPoint(const Point pt) {
        InvalidateArea();
        points_.push_back(pt);
    }
  
    Point GetPoint(const int i) {
        return points_[i];
    }
  
    int GetNumPoints() {
        return points_.size();
    }
  
    double GetArea() {
        if(area_ < 0) // if not yet calculated and cached
            CalcArea();     // calculate now
        return area_;
    }
  
private:
    void InvalidateArea() { area_ = -1; }
    
    void CalcArea() {
        area_ = 0;
        vector<Point>::iterator i;
        for(i = points_.begin(); i != points_.end(); ++i)
            area_ += /* some work */;
    }
    
    vector<Point> points_;
    double        area_;
};

Polygon operator+(Polygon& lhs, Polygon& rhs) {
    Polygon ret = lhs;
    int last = rhs.GetNumPoints();
    for(int i = 0; i < last; ++i) // concatenate
        ret.AddPoint(rhs.GetPoint(i));
    return ret;
}

void f(const Polygon& poly) {
    const_cast<Polygon&>(poly).AddPoint(Point(0,0));
}

void g(Polygon& const rPoly) {
    rPoly.AddPoint(Point(1,1));
}

void h(Polygon* const pPoly) {
    pPoly->AddPoint(Point(2,2));
}

int main() {
    Polygon poly;
    const Polygon cpoly;
    f(poly);
    f(cpoly);
    g(poly);
    h(&poly);
}
```

----

**Solution**

``` cpp
class Polygon {
public:
    Polygon() : area_(-1) {}
    
    void AddPoint(const Point pt) {
        InvalidateArea();
        points_.push_back(pt);
    }
```

**1.** Since the point object is passed by value, there is little or no benefit to declaring it const.

``` cpp
    Point GetPoint(const int i) {
        return points_[i];
    }
```

**2.** Same comment about the parameter as above. Normally const pass-by-value is unuseful and misleading at best.

**3.** This should be a const member function, since it doesn't change the state of the object.

**4.** (Arguable.) Return-by-value should normally be const for non-builtin return types. This assists client code by making sure the compiler emits an error if the caller tries to modify the temporary (for example, "`poly.GetPoint(i) = Point(2,2);`"... after all, if this was intended to work, `GetPoint()` should have used return-by-reference and not return-by-value in the first place, and as we will see later it makes sense for `GetPoint()` to return a const value or a const reference since it should be usable on const Polygon objects in `operator+()`).

Note: Lakos (pg. 618) argues against returning const value, and notes that it is redundant for builtins anyway (for example, returning "`const int`"), which he notes may interfere with template instantiation.

   **[Guideline]**: When using return-by-value for non-builtin return types, prefer returning a const value.

``` cpp
    int GetNumPoints() {
        return points_.size();
    }
```

**5.** Again, the function should be const.

(It should not return '`const int`' in this case, though, since the int is already an rvalue and to put in '`const`' can interfere with template instantiation and is confusing, misleading, and probably fattening.)

``` cpp
    double GetArea() {
        if(area_ < 0) // if not yet calculated and cached
            CalcArea();     // calculate now
        return area_;
    }
```

**6.** Even though it modifies the object's internal state, this function should be const because the object's observable state is unchanged (we are doing some caching, but that's an implementation detail and the object is logically const). This means that area_ should be declared mutable. If your compiler doesn't support mutable yet, kludge this with a `const_cast` of `area_` (and a comment to remove the cast when mutable is available!), but do make the function const.

``` cpp
private:
    void InvalidateArea() { area_ = -1; }
```

**7.** While this one is debatable, I'd still recommend that this function ought also to be const, even if for no other reason than consistency. (Granted, semantically it will only be called from non-const functions, since its purpose is to invalidate the cached `area_` when the object's state changes.)

``` cpp
    void CalcArea() {
        area_ = 0;
        vector<Point>::iterator i;
        for(i = points_.begin(); i != points_.end(); ++i)
            area_ += /* some work */;
    }
```

**8.** This member function definitely should be const. After all, it will be called from another const member function, namely `GetArea()`.

**9.** Since the iterator should not change the state of the `points_` collection, it ought to be a `const_iterator`.

``` cpp
    vector<Point> points_;
    double        area_;
};

    Polygon operator+(Polygon& lhs, Polygon& rhs) {
```

**10.** Pass by const references, of course.

**11.** Again, return-by-value should be const.

``` cpp
    Polygon ret = lhs;
    int last = rhs.GetNumPoints();
```

**12.** Since '`last`' should never change, say so with "`const int`".

``` cpp
    for(int i = 0; i < last; ++i) // concatenate
        ret.AddPoint(rhs.GetPoint(i));
```

(Another reason why `GetPoint()` should be a const member function, returning either const value or const reference.)

``` cpp
        return ret;
    }

    void f(const Polygon& poly) {
        const_cast<Polygon&>(poly).AddPoint(Point(0,0));
```

Bonus: This result is undefined if the referenced object is declared as const (which it is in the case of `f(cpoly)` below). The parameter isn't really const, so don't declare it as const!

``` cpp
    }

    void g(Polygon& const rPoly) {
        rPoly.AddPoint(Point(1,1));
    }
```

**13.** This '`const`' is useless, since references cannot be changed to refer to a different object anyway.

``` cpp
    void h(Polygon* const pPoly) {
        pPoly->AddPoint(Point(2,2));
    }
```

**14.** This '`const`' is equally useless, but for a different reason: since you're passing the pointer by value, this makes as little sense as passing a parameter '`const int`' above.

(If your answer to the bonus part said something about these functions being uncompilable... sorry, they're quite legal C++. You were probably thinking of putting the '`const`' to the left of the `&` or `*`, which would have made the function body illegal.)

``` cpp
int main() {
    Polygon poly;
    const Polygon cpoly;
    f(poly);
```

This is fine.

``` cpp
    f(cpoly);
```

This causes undefined results when `f()` tries to cast away the constness of and then modify its parameter.

``` cpp
    g(poly);
```

This is fine.

``` cpp
    h(&poly);
```

This is fine.

``` cpp
}
```

That's it. Here's a corrected version (remember, correcting const only, not other poor style):

``` cpp
class Polygon {
public:
    Polygon() : area_(-1) {}

    void  AddPoint(Point pt) {
        InvalidateArea();
        points_.push_back(pt);
    }
    
    const Point GetPoint(int i) const  { return points_[i]; }
    
    int GetNumPoints() const { return points_.size(); }

    double GetArea() const {
        if(area_ < 0) // if not yet calculated and cached
            CalcArea();     // calculate now
        return area_;
    }

private:
    void InvalidateArea() const { area_ = -1; }

    void CalcArea() const {
        area_ = 0;
        vector<Point>::const_iterator i;
        for(i = points_.begin(); i != points_.end(); ++i)
            area_ += /* some work */;
    }

    vector<Point>  points_;
    mutable double area_;
};

const Polygon operator+(const Polygon& lhs, const Polygon& rhs) {
    Polygon ret = lhs;
    const int last = rhs.GetNumPoints();
    for(int i = 0; i < last; ++i) // concatenate
        ret.AddPoint(rhs.GetPoint(i));
    return ret;
}

void f(Polygon& poly) {
    poly.AddPoint(Point(0,0));
}

void g(Polygon& rPoly) {
   rPoly.AddPoint(Point(1,1)); 
}

void h(Polygon* pPoly) {
   pPoly->AddPoint(Point(2,2)); 
}

int main() {
    Polygon poly;
    f(poly);
    g(poly);
    h(&poly);
}
```


# 007 Compile-Time Dependencies
**Difficulty: 7 / 10**
Most programmers #include many more headers than necessary. Do you? To find out, consider this issue's problem.

----

**Problem**
\[Warning: It's tougher than it looks. The comments are important.]

Most programmers #include much more than necessary. This can seriously degrade build times, especially when a popular header file includes too many other headers.

In the following header file, what `#include` directives could be immediately removed without ill effect? Second, what further #includes could be removed given suitable changes, and how? (You may not change the public interfaces of classes X and Y; that is, any changes you make to this header must not affect current client code).

``` cpp
// gotw007.h (implementation file is gotw007.cpp)

#include "a.h"  // class A
#include "b.h"  // class B
#include "c.h"  // class C
#include "d.h"  // class D
                // (note: only A and C have virtual functions)
#include <iostream>
#include <ostream>
#include <sstream>
#include <list>
#include <string>

class X : public A {
public:
    X(const C&);
    D    Function1(int, char*);
    D    Function1(int, C);
    B&   Function2(B);
    void Function3(std::wostringstream&);
    std::ostream& print(std::ostream&) const;
private:
    std::string  name_;
    std::list<C> clist_;
    D            d_;
};

std::ostream& operator<<(std::ostream& os, const X& x) {
    return x.print(os);
}

class Y : private B {
public:
    C  Function4(A);
private:
    std::list<std::wostringstream*> alist_;
};
```

----

**Solution**
First, consider the #includes that can be immediately removed. Here is the original header again as a reminder:

``` cpp
// gotw007.h (implementation file is gotw007.cpp)
//
#include "a.h"  // class A
#include "b.h"  // class B
#include "c.h"  // class C
#include "d.h"  // class D
                // (note: only A and C have virtual functions)
#include <iostream>
#include <ostream>
#include <sstream>
#include <list>
#include <string>

class X : public A {
public:
    X(const C&);
    D    Function1(int, char*);
    D    Function1(int, C);
    B&   Function2(B);
    void Function3(std::wostringstream&);
    std::ostream& print(std::ostream&) const;
private:
    std::string  name_;
    std::list<C> clist_;
    D            d_;
};
std::ostream& operator<<(std::ostream& os, const X& x) {
    return x.print(os);
}

class Y : private B {
public:
    C  Function4(A);
private:
    std::list<std::wostringstream*> alist_;
};
```

**1.** We can immediately remove:

   - iostream, because although streams are being used nothing in iostream specifically is being used
   
   - ostream and sstream, because parameter and return types need only be forward-declared and hence only the iosfwd header is needed (note that there are no comparable '`stringfwd`' or '`listfwd`' standard headers; iosfwd was introduced for backwards compatibility so as not to break code written for the old non-templated versions of the streams subsystem)

We cannot immediately remove:

   - a.h, because `A` is a base class of `X`
   
   - b.h, because `B` is a base class of `Y`
   
   - c.h, because many current compilers require `list<C>` be able to see the definition of `C` (this should be fixed on future versions of those compilers)
   
   - d.h, list and string, because `X` needs to know the size of `D` and string, and both `X` and `Y` need to know the size of list

Second, consider the #includes that can be removed by hiding the implementation details of `X` and `Y`:

**2.** We can remove `d.h`, list and string by letting `X` and `Y` use `pimpl_`'s (that is, the private parts are replaced by pointers to a forward- declared type of implementation object), because then `X` and `Y` do not need to know the sizes of `D` or list or string. This also lets us get rid of `c.h`, since besides in `X::clist_` `C` objects only appear as parameters and return values.

Important Note: The inlined free `operator<<` may still be inlined and use its ostream parameter, even though ostream has not been defined! This is because you only need the definition if you are going to call member functions, not if you are only going to accept an object and do nothing with it except use it as parameters to other function calls.

Finally, consider what we can fix by making other small changes:

**3.** We can remove `b.h` by noticing that it is a private base class of `Y` but that B has no virtual functions. The only major reason one would choose private inheritance over composition/containment is to override virtual functions. Hence, instead of inheriting from `B`, `Y` should have a member of type `B`. To remove the `b.h` header, this member should be in `Y`'s hidden `pimpl_` portion.

**[Guideline]**: Prefer using `pimpl_`'s (pointers to implementations) to insulate client code from implementation details.

Excerpted from the GotW coding standards:

   - encapsulation and insulation:
   
      - avoid showing private members of a class in its declaration:
     
         - use an opaque pointer declared as "`struct XxxxImpl* pimpl_`" to store private members (incl. both state variables and member functions), e.g., `class Map { private: struct MapImpl* pimpl_; };` (Lakos96: 398-405; Meyers92: 111-116; Murray93: 72-74)

**4.** We still can't do anything about `a.h` since `A` is used as a public base class, and the `IS-A` relationship is probably needed and used by client code since `A` has virtual functions. However, we could at least mitigate this by noticing that `X` and `Y` are fundamentally unrelated, and splitting the definitions of classes `X` and `Y` into two separate headers (providing the current header as a stub which includes both `x.h` and `y.h`, so as not to break existing code). This way, at least `y.h` does not need to include `a.h` since it only uses `A` as a function parameter type, which does not require a definition.

Putting it all together, we get much cleaner headers:

``` cpp
//---------------------------------------------------------------
// new file x.h: only TWO includes!
//
#include "a.h"  // class A
#include <iosfwd>

class C;
class D;

class X : public A {
public:
    X(const C&);
    D    Function1(int, char*);
    D    Function1(int, C);
    B&   Function2(B);
    void Function3(std::wostringstream&);
    std::ostream& print( std::ostream&) const;
private:
    class XImpl* pimpl_;
};

inline std::ostream& operator<<(std::ostream& os, const X& x){
    return x.print(os);
}
// NOTE: this does NOT require ostream's definition!


//---------------------------------------------------------------
// new file y.h: ZERO includes!
//
class A;
class C;

class Y {
public:
    C  Function4(A);
private:
    class YImpl* pimpl_;
};


//---------------------------------------------------------------
// gotw007.h is now just a compatibility stub with two lines, and
// pulls in only TWO extra secondary includes (through x.h)
//
#include "x.h"
#include "y.h"


//---------------------------------------------------------------
// new structures in gotw007.cpp... note that the impl objects
// will be new'd by the X/Y ctors and delete'd by the X/Y dtors
// and X/Y member functions will access the data through their
// pimpl_ pointers
//
struct XImpl    // yes, this can be called "struct" even
{               // though the forward-decl says "class"
    std::string  name_;
    std::list<C> clist_;
    D            d_;
}

struct YImpl
{
    std::list<std::wostringstream*> alist_;
    B b_;
}
```

Bottom Line: Clients of `X` need only pay for the #includes of `a.h` and `iosfwd`. Current clients of `Y` need only pay for the #includes of `a.h` and `iosfwd`, and if they are later updated to use `y.h` instead of gotw007.h they need pay for no secondary #includes at all. What an improvement over the original!



# 008 CHALLENGE EDITION: Exception Safety
**Difficulty: 9 / 10**
Exceptions can be an elegant solution to some problems, but they introduce many hidden control flows and can be difficult to use correctly. Try your hand at implementing a very simple container (a stack that users can push and pop) and see the issues involved with making it exception-safe and exception-neutral.

----

**Problem**
**1.** Implement the following container to be exception-neutral. Stack objects should always be in a consistent state and destructible even if some internal operations throw, and should allow any T exceptions to propagate through to the caller.

``` cpp
template <class T>    // T must have default ctor and copy assignment
class Stack
{
public:
    Stack();
    ~Stack();
    Stack(const Stack&);
    Stack& operator=(const Stack&);

    unsigned Count();   // returns # of T's in the stack
    void     Push(const T&);
    T        Pop();     // if empty, returns default-
                        // constructed T

private:
    T*       v_;        // pointer to a memory area big
                        //  enough for 'vsize_' T objects
    unsigned vsize_;    // the size of the 'v_' area
    unsigned vused_;    // the number of T's actually
                        //  used in the 'v_' area
};
```

**BONUS QUESTIONS**
**2.** Are the standard library containers exception-safe or exception-neutral, according to the current draft?

**3.** Should containers be exception-neutral? Why or why not? What are the tradeoffs?

**4.** Should containers use exception specifications? For example, should we declare "`Stack::Stack() throw(bad_alloc);`"?

**CHALLENGE**
With many current compilers, using "`try`" and "`catch`" often adds unnecessary overhead to your programs, which would be nice to avoid in this kind of low-level reusable container. Can you implement all Stack member functions as required without ever using "`try`" or "`catch`"?

----

Here are two example functions (but which do not necessarily meet the requirements) to get you started:

``` cpp
template<class T>
Stack<T>::Stack()
  : v_(0), vsize_(10), vused_(0)
{
    v_ = new T[vsize_]; // initial allocation
}

template<class T>
T Stack<T>::Pop()
{
    T result; // if empty, return default-constructed T
    if(vused_ > 0)
    {
        result = v_[--vused_];
    }
    return result;
}
```

----

**Solution**
[The solution did turn out to be entirely correct. For an updated and greatly enhanced version of this material, see my [September and November/December 1997 C++ Report articles](http://www.awlonline.com/cseng/meyerscddemo/DEMO/MAGAZINE/SU_FRAME.HTM), and further final updates in the book Exceptional C++.]

IMPORTANT NOTE: I do not claim that this solution actually meets my original requirements. In fact, I don't have a compiler than can compile it! While I've addressed as many of the interactions as I can think of, a major point of this exercise was to demonstrate that writing exception-safe code requires care.

See also Tom Cargill's excellent article, ["Exception Handling: A False Sense of Security"](http://www.informit.com/content/downloads/aw/meyerscddemo/DEMO/MAGAZINE/CA_FRAME.HTM) (C++ Report, vol. 9 no. 6, Nov-Dec 1994). He shows that EH is tricky, but please note that his article was probably not meant to dismiss EH in general and people shouldn't get too superstitious about using exceptions. Just be aware, and drive with care.

Last note: To keep this solution simpler, I've decided not to demonstrate the base class technique for exception-safe resource ownership. I invite Dave Abrahams (or others) to follow up to this solution and demonstrate this very effective technique.

To recap, here's the required interface:

``` cpp
template <class T>
    // T must have default ctor and copy assignment
class Stack
{
public:
    Stack();
    ~Stack();
    Stack(const Stack&);
    Stack& operator=(const Stack&);

    unsigned Count();   // returns # of T's in the stack
    void     Push(const T&);
    T        Pop();     // if empty, returns default-
                        // constructed T

private:
    T*       v_;        // pointer to a memory area big
                        //  enough for 'vsize_' T objects
    unsigned vsize_;    // the size of the 'v_' area
    unsigned vused_;    // the number of T's actually
                        //  used in the 'v_' area
};
```

Now for the implementations. One requirement we will place on T is T dtors must not throw. If dtors may throw, many operations are difficult or impossible to implement safely.

``` cpp
//----- DEFAULT CTOR ----------------------------------------------
template<class T>
Stack<T>::Stack()
  : v_(new T[10]),  // default allocation
    vsize_(10),
    vused_(0)       // nothing used yet
{
    // if we got here, the construction was okay
}

//----- COPY CTOR -------------------------------------------------
template<class T>
Stack<T>::Stack(const Stack<T>& other)
  : v_(0),      // nothing allocated or used yet
    vsize_(other.vsize_),
    vused_(other.vused_)
{
    v_ = NewCopy(other.v_, other.vsize_, other.vsize_);
    // if we got here, the copy construction was okay
}

//----- COPY ASSIGNMENT -------------------------------------------
template<class T>
Stack<T>& Stack<T>::operator=(const Stack<T>& other) 
{
    if(this != &other) 
	{
        T* v_new = NewCopy(other.v_, other.vsize_, other.vsize_);
        // if we got here, the allocation and copy were okay

        delete[] v_;
        // note that this cannot throw, since T dtors cannot
        // throw and ::operator delete[] is declared as throw()
    
        v_ = v_new;
        vsize_ = other.vsize_;
        vused_ = other.vused_;
    }
    
    return *this;   // safe, no copy involved
}

//----- DTOR ------------------------------------------------------
template<class T>
Stack<T>::~Stack() 
{
    delete[] v_;    // again, this can't throw
}

//----- COUNT -----------------------------------------------------
template<class T>
unsigned Stack<T>::Count() 
{
    return vused_;  // it's just a builtin, nothing can go wrong
}

//----- PUSH ------------------------------------------------------
template<class T>
void Stack<T>::Push(const T& t) {
    if(vused_ == vsize_)  // grow if necessary
    {
        unsigned vsize_new = (vsize_+1)*2; // grow factor
        T* v_new = NewCopy(v_, vsize_, vsize_new);
        // if we got here, the allocation and copy were okay

        delete[] v_;    // again, this can't throw
        v_ = v_new;
        vsize_ = vsize_new;
    }
    
    v_[vused_] = t; // if this copy throws, the increment
    ++vused_;       //  isn't done and the state is unchanged
}

//----- POP -------------------------------------------------------
template<class T>
T Stack<T>::Pop()
{
    T result;
    if(vused_ > 0)
    {
        result = v_[vused_-1];  // if this copy throws, the
        --vused_;               //  decrement isn't done and
    }                           //  the state is unchanged
    return result;
}

//
// NOTE: As alert reader Wil Evers was the first to point out,
//  "As defined in the challenge, Pop() forces its users to write
//  exception-unsafe code, which first causes a side effect
//  (popping an element off the stack) and then lets an exception
//  escape (copying the return value into [the caller's
//  destination object])."
//
// This points out that one reason writing exception-safe
// code is so hard is that it affects not only your
// implementations, but also your interfaces!  Some
// interfaces, like this one, can't be implemented in a
// fully exception-safe manner.
//
// One way to correct this one is to respecify the function as
// "void Stack<T>::Pop( T& result)".  This way we can know
// the copy to the result has actually succeeded before we
// change the state of the stack.  For example, here's an
// alternative version of Pop() that's more exception-safe:
//
template<class T>
void Stack<T>::Pop(T& result)
{
    if(vused_ > 0)
    {
        result = v_[vused_-1];  // if this copy throws, the
        --vused_;               //  decrement isn't done and
    }                           //  the state is unchanged
}
//
// Alternatively, we may let Pop() return void and provide a
// Front() member function to access the topmost object.
//


//----- HELPER FUNCTION -------------------------------------------
// When we want to make a (possibly larger) copy of a T buffer,
//  this function will allocate the new buffer and copy over the
//  original elements.  If any exceptions are encountered, the
//  function releases all temporary resources and propagates
//  the exception so that nothing is leaked.
//
template<class T>
T* NewCopy(const T* src, unsigned srcsize, unsigned destsize)
{
    destsize = max(srcsize, destsize); // basic parm check
    T* dest = new T[destsize];
    // if we got here, the allocation/ctors were okay

    try
    {
        copy(src, src+srcsize, dest);
    }
    catch(...)
    {
        delete[] dest;
        throw;  // rethrow the original exception
    }
    // if we got here, the copy was okay
    
    return dest;
}
```

**BONUS QUESTIONS**
**2.** Are the standard library containers exception-safe or exception-neutral, according to the current draft?

This is currently unspecified. Lately there has been some discussion within the committee about whether to provide either weak ("container is always destructible") or strong ("all container operations have commit-or-rollback semantics") exception safety guarantees. As Dave Abrahams has pointed out in that discussion, and since then in email, often the strong guarantee "comes along for free" if you implement the weak guarantee. This is the case with several operations above.

**3.** Should containers be exception-neutral? Why or why not? What are the tradeoffs?

Some containers have operations with unavoidable space tradeoffs if they are to be made exception-neutral. Exception-neutrality is a Good Thing in itself, but may not be practical when implementing the strong guarantee requires much more space or time than does implementing the weak guarantee. Often a good compromise is to document which T operations are expected not to throw and then guarantee exception-neutrality based on conformance to those assumptions.

**4.** Should containers use exception specifications? For example, should we declare "`Stack::Stack() throw(bad_alloc);`"?

No, since it is unknown in advance which T operations might throw or what they might throw.

Notice that some container operations (e.g., `Count()`) simply return a scalar and are known not to throw. While it's possible to declare these with "`throw()`", there are two possible reasons why you wouldn't: first, it limits you in the future in case you want to change the underlying implementation to a form which could throw; and second, exception specifications incur a performance overhead whether an exception is thrown or not. For widely-used operations, it may be better not to use exception specifications to avoid this overhead.

**CHALLENGE**
With many current compilers, using "`try`" and "`catch`" often adds unnecessary overhead to your programs, which would be nice to avoid in this kind of low-level reusable container. Can you implement all Stack member functions as required without ever using "`try`" or "`catch`"?

Yes, because we only care about catching "`...`". In general, code of the form

``` cpp
    try 
	{ 
	    TryCode(); 
	} 
	catch(...) 
	{ 
	    CatchCode(parms); throw; 
	}
```

can be rewritten as

``` cpp
    struct Janitor {
        Janitor(Parms p) : pa(p) {}
        ~Janitor() { if uncaught_exception() CatchCode(pa); }
        Parms pa;
    };
    
    {
        Janitor j(parms); // j is destroyed both if TryCode()
                          // succeeds and if it throws
        TryCode();
    }
```

Our solution uses `try/catch` only in the NewCopy function, so let's illustrate this technique by rewriting NewCopy:

``` cpp
template<class T>
T* NewCopy(const T* src, unsigned srcsize, unsigned destsize)
{
    destsize = max(srcsize, destsize); // basic parm check

    struct Janitor {
        Janitor(T* p) : pa(p) {}
        ~Janitor() { if(uncaught_exception()) delete[] pa; }
        T* pa;
    };
    
    T* dest = new T[destsize];
    // if we got here, the allocation/ctors were okay
    
    Janitor j(dest);
    copy(src, src+srcsize, dest);
    // if we got here, the copy was okay... otherwise, j
    // was destroyed during stack unwinding and will handle
	
    // the cleanup of dest to avoid leaking memory
    
    return dest;
}
```

Having said that, I've now talked to several people who've done empirical speed tests. In the case when no exception occurs, `try/catch` is usually much faster, and can be expected to continue to be faster. However, it is still important to know about this kind of technique because it often yields more elegant and maintainable code, and because some current compilers still produce extremely inefficient code for `try/catch` in both the exceptional and exception-free code paths.


# 009 Memory Management - Part I
**Difficulty: 3 / 10**
This GotW covers basics about C++'s main distinct memory stores. The following problem attacks some deeper memory-management questions in more detail.

----

**Problem**
C++ has several distinct memory areas where objects and non-object values may be stored, and each area has different characteristics.

Name as many of the distinct memory areas as you can. For each, summarize its performance characteristics and describe the lifetime of objects stored there.

Example: The stack stores automatic variables, including both builtins and objects of class type.

----

**Solution**
The following summarizes a C++ program's major distinct memory areas. Note that some of the names (e.g., "heap") do not appear as such in the draft [standard].

| Memory Area | Characteristics and Object Lifetimes |
| ----------- | ------------------------------------ |
|Const Data | The const data area stores string literals and other data whose values are known at compile time.  No objects of class type can exist in this area.  All data in this area is available during the entire lifetime of the program. Further, all of this data is read-only, and the results of trying to modify it are undefined. This is in part because even the underlying storage format is subject to arbitrary optimization by the implementation.  For example, a particular compiler may store string literals in overlapping objects if it wants to.|
|Stack| The stack stores automatic variables. Typically allocation is much faster than for dynamic storage (heap or free store) because a memory allocation involves only pointer increment rather than more complex management.  Objects are constructed immediately after memory is allocated and destroyed immediately before memory is deallocated, so there is no opportunity for programmers to directly manipulate allocated but uninitialized stack space (barring willful tampering using explicit dtors and placement new).|
|Free Store | The free store is one of the two dynamic memory areas, allocated/freed by new/delete.  Object lifetime can be less than the time the storage is allocated; that is, free store objects can have memory allocated without being immediately initialized, and can be destroyed without the memory being immediately deallocated.  During the period when the storage is allocated but outside the object's lifetime, the storage may be accessed and manipulated through a `void*` but none of the proto-object's nonstatic members or member functions may be accessed, have their addresses taken, or be otherwise manipulated.|
|Heap | The heap is the other dynamic memory area, `allocated/freed` by `malloc/free` and their variants.  Note that while the default global new and delete might be implemented in terms of malloc and free by a particular compiler, the heap is not the same as free store and memory allocated in one area cannot be safely deallocated in the other. Memory allocated from the heap can be used for objects of class type by `placement-new` construction and explicit destruction.  If so used, the notes about free store object lifetime apply similarly here.|
|Global/Static|   Global or static variables and objects have their storage allocated at program startup, but may not be initialized until after the program has begun executing.  For instance, a static variable in a function is initialized only the first time program execution passes through its definition.  The order of initialization of global variables across translation units is not defined, and special care is needed to manage dependencies between global objects (including class statics).  As always, uninitialized proto-objects' storage may be accessed and manipulated through a `void*` but no nonstatic members or member functions may be used or referenced outside the object's actual lifetime.|
Note about Heap vs. Free Store: We distinguish between "`heap`" and "`free store`" because the draft deliberately leaves unspecified the question of whether these two areas are related. For example, when memory is deallocated via operator delete, 18.4.1.1 states:

   *It is unspecified under what conditions part or all of such reclaimed storage is allocated by a subsequent call to `operator new` or any of `calloc`, `malloc`, or `realloc`, declared in `<cstdlib>`.*

Similarly, it is unspecified whether new/delete is implemented in terms of `malloc/free` or vice versa in a given implementation. Effectively, the two areas behave differently and are accessed differently \-\- so be sure to use them differently!



# 010 Memory Management - Part II
**Difficulty: 6 / 10**
Are you thinking about doing your own class-specific memory management, or even replacing C++'s global new and delete? First try this problem on for size.

----

**Problem**
Here are excerpts from a program with classes that perform their own memory management. Point out as many memory- related errors as possible, and answer the additional questions.

``` cpp
//  Why do B's operators delete have a second parameter,
//  whereas D's do not?
class B {
public:
    virtual ~B();
    void operator delete  (void*, size_t) throw();
    void operator delete[](void*, size_t) throw();
    void f(void*, size_t) throw();
};

class D : public B {
public:
    void operator delete  (void*) throw();
    void operator delete[](void*) throw();
};

void f() {
    //  Which operator delete is called for each of the
    //  following?  Why, and with what parameters?
    D* pd1 = new D;
    delete pd1;

    B* pb1 = new D;
    delete pb1;

    D* pd2 = new D[10];
    delete[] pd2;

    B* pb2 = new D[10];
    delete[] pb2;

    //  Are the following two assignments legal?
    B b;
    typedef void (B::*PMF)(void*, size_t);
    PMF p1 = &B::f;
    PMF p2 = &B::operator delete;
}

class X {
public:
    void* operator new(size_t s, int) throw(bad_alloc) {
        return ::operator new(s);
    }
};

class SharedMemory {
public:
    static void* Allocate(size_t s) {
        return OsSpecificSharedMemAllocation(s);
    }
    static void  Deallocate(void* p, int i) {
        OsSpecificSharedMemDeallocation(p, i);
    }
};

class Y {
public:
    void* operator new(size_t s, SharedMemory& m) throw(bad_alloc) {
        return m.Allocate(s);
    }

    void  operator delete(void* p, SharedMemory& m, int i) throw() {
        m.Deallocate(p, i);
    }
};

void operator delete(void* p) throw() {
    SharedMemory::Deallocate(p);
}

void operator delete(void* p, std::nothrow_t&) throw() {
    SharedMemory::Deallocate(p);
}
```

----

**Solution**

``` cpp
//  Why do B's operators delete have a second parameter,
//  whereas D's do not?
class B {
public:
    virtual ~B();
    void operator delete  (void*, size_t) throw();
    void operator delete[](void*, size_t) throw();
    void f(void*, size_t) throw();
};

class D : public B {
public:
    void operator delete  (void*) throw();
    void operator delete[](void*) throw();
};
```

Answer: Preference. Both are usual deallocation functions, not placement deletes. (See 3.7.3.2/2)

However, both classes provide operators `delete` and `delete[]` without providing corresponding operators `new` and `new[]`. This is extremely dangerous. For example, consider what happens if a further-derived class provides its own operator `new` or `new[]`.

``` cpp
void f(){
    //  Which operator delete is called for each of the
    //  following?  Why, and with what parameters?
    //
    D* pd1 = new D;
    delete pd1;
```
Calls `D::operator delete(void*)`.
``` cpp
    B* pb1 = new D;
    delete pb1;
```

Calls `D::operator delete(void*)`. Since B's dtor is virtual, of course D's dtor is properly called, but the fact that B's dtor is virtual also implicitly means that `D::operator delete()` must be called, even though `B::operator delete()` is not (in fact, cannot) be virtual.

``` cpp
    D* pd2 = new D[10];
    delete[] pd2;
```

Calls `D::operator delete[](void*)`.

``` cpp
    B* pb2 = new D[10];
    delete[] pb2;
```

Undefined behaviour. The language requires that the static type of the pointer that is passed to operator delete must be the same as its dynamic type. For more information on this topic, see also Scott Meyers' section on "Never Treat Arrays Polymorphically" in Effective C++ or More Effective C++.

``` cpp
    //  Are the following two assignments legal?
    //
    B b;
    typedef void (B::*PMF)(void*, size_t);
    PMF p1 = &B::f;
    PMF p2 = &B::operator delete;
}
```

The first assignment is fine, but the second assignment is illegal because "`void operator delete(void*, size_t) throw()`" is NOT a member function of B even though as written above it may look like one. The trick here is to remember that operators new and delete are always static, even if not explicitly declared static. It's a good habit to always declare them static, just to make sure that the fact is obvious to all programmers reading through your code.

``` cpp
class X {
public:
    void* operator new(size_t s, int) throw(bad_alloc) {
        return ::operator new(s);
    }
};
```

This invites a memory leak since no corresponding placement delete exists. Similarly below:

``` cpp
class SharedMemory {
public:
    static void* Allocate(size_t s) {
        return OsSpecificSharedMemAllocation(s);
    }
    
    static void  Deallocate(void* p, int i) {
        OsSpecificSharedMemDeallocation(p, i);
    }
};

class Y {
public:
    void* operator new(size_t s, SharedMemory& m) throw(bad_alloc) {
        return m.Allocate(s);
    }
```

This invites a memory leak, since no operator delete matches this signature. If an exception is thrown during construction of an object to be located in memory allocated by this function, the memory will not be properly freed. For example:

``` cpp
    SharedMemory shared;
    ...
    new (shared) T; // if T::T() throws, memory is leaked
```

Further, the memory cannot safely be deleted since the class does not provide a usual operator delete. This means that a base or derived class's operator delete, or the global one, will have to try to deal with this deallocation (almost certainly unsuccessfully, unless you also replace all such surrounding operators delete, which would be onerous and evil).

``` cpp
    void  operator delete(void* p, SharedMemory& m, int i) throw() {
        m.Deallocate(p, i);
    }
};
```

This operator delete is useless since it can never be called.

``` cpp
void operator delete(void* p) throw() 
{
    SharedMemory::Deallocate(p);
}
```

A serious error, since the replacement global operator delete is going to delete memory allocated by the default `::operator new`, not by `SharedMemory::Allocate()`. The best you can hope for is a quick core dump. Evil.

``` cpp
void operator delete(void* p, std::nothrow_t&) throw() {
    SharedMemory::Deallocate(p);
}
```

Same here, but slightly more subtle. This replacement operator delete will only be called if a "`new (nothrow) T`" fails because T's ctor exits with an exception, and will try to deallocate memory not allocated by `SharedMemory::Allocate()`. Evil and insidious.

If you got all of these answers, then you're definitely on your way to becoming an expert in memory-management mechanics.




# 011 Object Identity
**Difficulty: 5 / 10**
"Who am I, really?" This problem addresses how to decide whether two pointers really refer to the same object.

----

**Problem**
The "`this != &other`" test (illustrated below) is a common coding practice intended to prevent self-assignment. Is the condition necessary and/or sufficient to accomplish this? Why or why not? If not, how would you fix it? Remember to distinguish between "protecting against Murphy vs. protecting against Machiavelli."

``` cpp
T& T::operator=(const T& other){
    if(this != &other) {  // the test in question
        // ...
    }
    return *this;
}
```

----

**Solution**
Short answer: Technically, it's neither necessary nor sufficient. In practice it's probably fine and may even be fixed in the standard.

**Issue: Exception Safety (Murphy)**
If `operator=()` is exception-safe, you don't need to test for self-assignment. There are two efficiency downsides, however: 
   a) if you can test for self-assignment then you can completely optimize away the assignment; and 
   b) often making code exception-safe also makes it less efficient (a.k.a. the "paranoia has a price principle").

**Nonissue: Multiple Inheritance**
The problem has nothing to do with multiple inheritance, though some have suggested this in the past. The problem is a technical question of how the draft lets you compare pointers. Namely:

**Issue: Operator Overloading (Machiavelli)**
Since classes may provide their own `operator&()`, the test in question may do something completely different than intended. This comes under the heading of "protecting against Machiavelli" because presumably the writer of `operator=()` knows whether or not his class also overloads `operator&()`.

Note that while a class may also provide a `T::operator!=()`, it's irrelevant since it can't interfere with this test. The reason is that you can't write an `operator!=()` that takes two `T*` parameters since at least one parameter to an overloaded operator must be of class type.

**Postscript #1**
Here's a "code joke". Believe it or not, it's been tried by well-meaning but clearly misguided coders:

``` cpp
T::T(const T& other) {
    if(this != &other) {
        // ...
    }
}
```

Did you get the point on first reading?

**Postscript #2**
Note that there are other cases where pointer comparison is not what most people would consider intuitive. For instance:

   1. As James Kanze points out, comparing pointers into string literals is undefined. The reason (which I didn't see stated) is that the draft explicitly allows compilers to store string literals in overlapping areas of memory as a space optimization.

   2. In general you cannot compare arbitrary bald pointers using builtin operators `<`, `<=`, `>`, and `>=` with well-defined results, although the results are defined in specific situations (e.g., pointers to objects in the same array). The standard library works around this limitation by saying that the library functions `less<>` et al. must give an ordering of pointers, so that you can create, say, a map with keys of pointer type, e.g., `map< T*, U, less<T*> >`.




# 012 Control Flow
**Difficulty: 6 / 10**
How well do you really know the order in which C++ code is executed? Test your knowledge against this problem.

----

**Problem**
"The devil is in the details." Point out as many problems as possible in the following (somewhat contrived) code, focusing on those related to control flow.

``` cpp
#include <cassert>
#include <iostream>
#include <typeinfo>
#include <string>
using namespace std;

//  The following lines come from other header files.

char* itoa(int value, char* workArea, int radix);
extern int fileIdCounter;

//  Helpers to automate class invariant checking.

template<class T>
inline void AAssert(T& p) {
    static int localFileId = ++fileIdCounter;
    if(!p.Invariant()) {
        cerr << "Invariant failed: file " << localFileId << ", " 
             << typeid(p).name() << " at " << static_cast<void*>(&p) << endl;
        assert(false);
    }
}

template<class T>
class AInvariant {
public:
    AInvariant(T&) : p_(p) { AAssert(p_); }
    ~AInvariant()            { AAssert(p_); }
private:
    T& p_;
};

#define AINVARIANT_GUARD AInvariant<AIType> invariantChecker(*this)

//-------------------------------------------------
class Array : private ArrayBase, public Container {
    typedef Array AIType;
public:
    Array(size_t startingSize = 10)
      : Container(startingSize),
        ArrayBase(Container::GetType()),
        used_(0),
        size_(startingSize),
        buffer_(new char[size_])
    {
        AINVARIANT_GUARD;
    }

    void Resize(size_t newSize) {
        AINVARIANT_GUARD;
        char* oldBuffer = buffer_;
        buffer_ = new char[newSize];
        memset(buffer_, ' ', newSize);
        copy(oldBuffer, oldBuffer+min(size_,newSize), buffer_);
        delete[] oldBuffer;
        size_ = newSize;
    }

    string PrintSizes() {
        AINVARIANT_GUARD;
        char buf[30];
        return string("size = ") + itoa(size_,buf,10) +
               ", used = " + itoa(used_,buf,10);
    }

    bool Invariant() {
        if(used_ > 0.9*size_) 
            Resize(2*size_);
        return used_ <= size_;
    }
private:
    char*  buffer_;
    size_t used_, size_;
};

int f(int& x, int y = x) { return x += y; }
int g(int& x)            { return x /= 2; }

int main(int, char*[]) {
    int i = 42;
    cout << "f(" << i << ") = " << f(i) << ", "
         << "g(" << i << ") = " << g(i) << endl;
    Array a(20);
    cout << a.PrintSizes() << endl;
}
```


----

**Solution**
   "Lions and tigers and bears, oh my!" 
      \-\- Dorothy

``` cpp
#include <cassert>
#include <iostream>
#include <typeinfo>
#include <string>
using namespace std;

//  The following lines come from other header files.
//
char* itoa(int value, char* workArea, int radix);
extern int fileIdCounter;
```

The presence of a global variable should already put us on the lookout for clients that might try to use it before it has been initialized. The order of initialization for global variables (including class statics) between translation units is undefined.

``` cpp
//  Helpers to automate class invariant checking.
//
template<class T>
inline void AAssert(T& p) {
    static int localFileId = ++fileIdCounter;
```

Aha, here we have a case in point. If the compiler happens to initialize fileIdCounter before it initializes any `AAssert<T>::localFileId`, well and good. Otherwise, the value set will be based on whatever was in `fileIDCounter`'s raw memory before it was initialized.

``` cpp
    if(!p.Invariant()) {
        cerr << "Invariant failed: file " << localFileId
             << ", " << typeid(p).name()
             << " at " << static_cast<void*>(&p) << endl;
        assert(false);
    }
}

template<class T>
class AInvariant {
public:
    AInvariant(T& p) : p_(p) { AAssert(p_); }
    ~AInvariant()            { AAssert(p_); }
private:
    T& p_;
};

#define AINVARIANT_GUARD AInvariant<AIType> \
                         invariantChecker(*this)
```

These helpers are an interesting idea, where any client class that would like to automatically check its class invariants before and after function calls simply writes a typedef of `AIType` to itself, then writes "`AINVARIANT_GUARD;`" as the first line of member functions. Not entirely bad, in itself.

In the client code below, these ideas go unfortunately astray. The main reason for this is that AInvariant hides calls to `assert()`, which will be automatically removed by the compiler when the program is built in non-debug mode. As written, the following client code was likely written by a programmer who wasn't aware of this build dependency and the resulting change in side effects.

``` cpp
//-------------------------------------------------
class Array : private ArrayBase, public Container {
    typedef Array AIType;
public:
    Array(size_t startingSize = 10) 
	    :Container(startingSize),
        ArrayBase(Container::GetType()),
```

This constructor's initializer list has two potential errors. This first one is NOT necessarily an error, but was left in as a bit of a red herring:

1. If `GetType()` is a static member function, or a member function that does not use its '`this`' pointer (i.e., uses no member data) and does not rely on any side effects of construction (e.g., static usage counts), then this is merely poor style but will run correctly.

2. Otherwise (mainly, if `GetType()` is a normal nonstatic member function), then we have a problem. Nonvirtual base classes are initialized in `left-to-right` order as they are declared, and so `ArrayBase` is initialized before Container. Unfortunately, that means we're trying to use a member of the not-yet-initialized Container subobject.

``` cpp
        used_(0),
        size_(startingSize),
        buffer_(new char[size_])
```

This second is indeed an error, since the variables will be actually initialized in the order in which they appear later in the class definition:

``` cpp
        buffer_(new char[size_])
        used_(0),
        size_(startingSize),
```

Writing it this way makes the error obvious. The call to `new[]` will make buffer an unpredictable size \-\- typically zero or something extremely large, depending on whether the compiler happens to initialize object memory to nulls before invoking constructors. At any rate, the initial allocation is unlikely to end up actually being for 'startingSize' bytes.

``` cpp
    {
        AINVARIANT_GUARD;
    }
```

Minor efficiency issue: Here the `Invariant()` function will be needlessly called twice, once during construction and again during destruction of the hidden temporary. This is a nit, though, and is unlikely to be a real issue.

``` cpp
    void Resize(size_t newSize) {
        AINVARIANT_GUARD;
        char* oldBuffer = buffer_;
        buffer_ = new char[newSize];
        memset(buffer_, ' ', newSize);
        copy(oldBuffer, oldBuffer+min(size_,newSize), buffer_);
        delete[] oldBuffer;
        size_ = newSize;
    }
```

There is a serious control flow problem here. I didn't see anyone point it out (sorry if I missed anyone), which is good since I deliberately put it in to see if anyone would comment.

Before reading on, examine the function again to see if you can spot a (hint: pretty obvious) control flow problem.

----

Answer: It's not exception-safe. If the call to `new[]` throws a bad_alloc exception, not only is the current object left in an invalid state, but the original buffer is leaked since all pointers to it are lost and so it can never be deleted.

The point of this function was to show how, so far, it seems that few if any programmers yet write exception-safe code as a matter of habit \-\- even after we recently had an extensively discussed GotW on exception safety!

``` cpp
    string PrintSizes() {
        AINVARIANT_GUARD;
        char buf[30];
        return string("size = ") + itoa(size_,buf,10) +
               ", used = " + itoa(used_,buf,10);
    }
```

The prototyped `itoa()` function uses the passed buffer as a scratch area. However, there's a control flow problem because there's no way to predict the order in which the expressions in the last line are evaluated, because the order in which function parameters are ordered is undefined and implementation-dependent. (Note that this would not be true for the BUILTIN `operator+`, but as soon as you provide your own the rules change to the rules for function calls.)

What the last line really amounts to is something like this, since `+` calls are still performed `left-to-right`:

``` cpp
        return
            operator+(
                operator+(
                    operator+(string("size = "),
                              itoa(size_,buf,10)) ,
                    ", y = ") ,
                itoa(used_,buf,10));
```

Say that `size_` is `10` and `used_` is `5`. Then if the outer `operator+()`'s first parameter is evaluated first, the output will be the correct "`size = 10, used = 5`" since the results of the first `itoa()` is used and stored in a temporary string before the second `itoa()` reuses the same buffer. If the outer `operator+()`'s second parameter is evaluated first (as it is, for example, under MSVC 4.x), the output will be the incorrect "`size = 10, used = 10`" since the outer `itoa()` is executed first and then the inner `itoa()` will clobber the results of the outer `itoa()` before either value is used.

``` cpp
    bool Invariant() {
        if(used_ > 0.9*size_)
            Resize(2*size_);
        return used_ <= size_;
```

The call to `Resize()` has two problems.

1. In this case, the program wouldn't work at all anyway, since if the condition is true then `Resize()` will be called, only to immediately call `Invariant()` again, which will find the condition still true and will call `Resize()` again, which... you get the idea.

2. What if, for efficiency, the writer of `AAssert()` decided to remove the error reporting and simply wrote "`assert(p->Invariant());`"? Then this client code becomes deplorable style, because it puts code with side effects inside an `assert()` call. This means the program's behaviour will be different when compiled in debug mode than it is when compiled in release mode. Even without the first problem above, this would be bad because it means that Array objects will resize at different times depending on the build mode, which will make the testers' lives a living hell as they try to reproduce customer problems on a debug build that ends up having a different runtime memory image characteristics.

Bottom line: Never write code with side effects inside a call to `assert()` (or something that might be one), and always make sure your recursions really terminate.

``` cpp
    }
private:
    char*  buffer_;
    size_t used_, size_;
};

int f(int& x, int y = x) { return x += y; }
```

The second parameter default isn't legal C++ at any rate, so this shouldn't compile under a conforming compiler (though some systems will take this). One reason this is bad is again that the compiler is free to choose the order in which it wants to evaluate the function parameters, so if this were allowed then y might be initialized before x.

``` cpp
int g(int& x)            { return x /= 2; }

int main(int, char*[]) {
    int i = 42;
    cout << "f(" << i << ") = " << f(i) << ", "
         << "g(" << i << ") = " << g(i) << endl;
```

Here we run into parameter evaluation ordering again. Since there's no telling the order in which `f(i)` or `g(i)` will be executed (or, for that matter, the ordering of the two bald evaluations of '`i`' itself), the printed results can be quite incorrect. One example result is MSVC's "`f(22) = 22, g(21) = 21`", which means the compiler is likely evaluating all function arguments in order from right to left.

But isn't the result wrong? No, the compiler is right... and another compiler could print out something else and still be right, too, because the programmer is relying on something that is undefined in C++.

``` cpp
    Array a(20);
    cout << a.PrintSizes() << endl;
}
```

Perhaps Dorothy wasn't quite right... the following might be closer:

   "Parameters and globals and exceptions, oh my!" 
      \-\- Dorothy, after an intermediate C++ course



# 013 OOP
**Difficulty: 4 / 10**
Is C++ an object-oriented language? It both is and is not, contrary to popular opinion.

----

**Problem**
"C++ is a powerful language that provides many advanced object-oriented constructs, including encapsulation, exception handling, inheritance, templates, polymorphism, strong typing, and a complete module system."

Discuss.

----

**Solution**
The purpose of this GotW was to provoke discussion about major (and even missing) features in C++, to provide a healthy dose of reality. Specifically, I hoped that the resulting discussion would illustrate three things, and I wasn't disappointed:

**1.** Not everyone agrees on what what "`OO`" means. Most would agree that inheritance and polymorphism are "`OO`" concepts; some would include encapsulation; a few might include exception handling; perhaps no one would include templates. Still, there are differing opinions.

**2.** C++ is a multiparadigm language, not an `OO` language. It supports many `OO` features, but it doesn't force programmers to use them. You can write completely `non-OO` programs in C++, and many people do.

**3.** No language is the be-all and end-all. Today, I'm using C++ as my primary programming language; tomorrow, I'll use whatever best suits what I'm doing then. C++ does not have a module system (complete or otherwise), it lacks other major features like garbage collection, and it has static typing but not necessarily "strong" typing. All languages have advantages and drawbacks. Just pick the right tool for the job, and avoid the temptation to become a nearsighted language zealot. :-)



# 014 Class Relationships - Part I
**Difficulty: 5 / 10**
How are your OO design skills? This GotW illustrates a common class design mistake that still catches many programmers.

----

**Problem**
A networking application has two different kinds of communications sessions, each with its own message protocol. The two protocols have similarities (some computations and even some messages are the same), and so the programmer has come up with the following design to encapsulate the common computations and messages in a BasicProtocol class:

```
class BasicProtocol /* : possible base classes */ {
public:
    BasicProtocol();
    virtual ~BasicProtocol();
    bool BasicMsgA(/*...*/);
    bool BasicMsgB(/*...*/);
    bool BasicMsgC(/*...*/);
};

class Protocol1 : public BasicProtocol {
public:
    Protocol1();
    ~Protocol1();
    bool DoMsg1(/*...*/);
    bool DoMsg2(/*...*/);
    bool DoMsg3(/*...*/);
    bool DoMsg4(/*...*/);
};

class Protocol2 : public BasicProtocol {
public:
    Protocol2();
    ~Protocol2();
    bool DoMsg1(/*...*/);
    bool DoMsg2(/*...*/);
    bool DoMsg3(/*...*/);
    bool DoMsg4(/*...*/);
    bool DoMsg5(/*...*/);
};
```

The member functions of each derived classes call the base functions as needed to perform the common work, but perform the actual transmissions themselves. Each class may have additional members, but you can assume that all significant members are shown.

Comment on this design. Is there anything you would change? If so, why?

----

**Solution**
This GotW illustrates a very common pitfall in OO class relationship design. To recap, classes Protocol1 and Protocol2 are publicly derived from a common base class BasicProtocol which performs some common work.

A key to identifying the problem is the following sentence:

   The member functions of each derived classes call the base functions as needed to perform the common work, but perform the actual transmissions themselves.

Here we have it: This is clearly describing an "is implemented in terms of" relationship, which in C++ is spelled either "private inheritance" or "membership." Unfortunately, many people still frequently misspell it as "public inheritance," thereby confusing implementation inheritance with interface inheritance. The two are not the same thing, and that confusion is at the root of the problem here.[1]

In a little more detail, here are several clues that help indicate this problem:

  1. BasicProtocol provides no virtual functions (other than the destructor, which we'll get to in a minute).[2] This means that it is not intended to be used polymorphically, which is a strong hint against public inheritance.
  
  2. BasicProtocol has no protected functions or members. This means that there is no "derivation interface," which is a strong hint against any inheritance at all, either public or private.
  
  3. BasicProtocol encapsulates common work, but as described it does not seem to actually perform its own transmissions as the derived classes do. This means that a BasicProtocol object does not WORK-LIKE-A derived protocol object, nor is it USABLE-AS-A derived protocol object. Public inheritance should be used to model one thing and one thing only: a true interface IS-A relationship that obeys the Liskov substitution principle. For greater clarity, I usually call this WORKS-LIKE-A or USABLE-AS-A.[3]
  
  4. The derived classes use BasicProtocol's public interface only. This means that they do not benefit from being derived classes, and could as easily perform their work using a separate helper object of type BasicProtocol.

This means we have a few cleanup issues: First, since BasicProtocol is clearly not designed to be derived from, its virtual destructor is unnecessary and should be eliminated. Second, BasicProtocol should probably be renamed to something less misleading, such as MessageCreator.

Once we've made those changes, which option should be used to model this "is implemented in terms of" relationship: private inheritance, or membership? The answer is pretty easy to remember:

**[Guideline]:** When modelling "is implemented in terms of," always prefer membership. Use private inheritance only when inheritance is absolutely necessary, that is, when you need access to protected members or you need to override a virtual function. Never use public inheritance for code reuse.

Using membership forces a better separation of concerns since the using class is a normal client with access to only the used class' public interface. Prefer it, and you'll find that your code is cleaner, easier to read, and easier to maintain. In short, your code will cost less!

**Notes**
1. Incidentally, programmers in the habit of making this mistake (using public inheritance for implementation) usually end up creating deep inheritance hierarchies. This greatly increases the maintenance burden by adding unnecessary complexity, forcing users to learn the interfaces of many classes even when all they want to do is use a specific derived class. It can also impact memory use and program performance by adding unnecessary vtables and indirections to classes which do not really need them. If you find yourself frequently creating deep inheritance hierarchies, you should review your design style to see if you've picked up this bad habit. Deep hierarchies are rarely needed and almost never good... and if you don't believe that, thinking that OO just isn't OO without lots of inheritance, a good example to look at is the standard library.

2. Even if BasicProtocol were itself derived from another class, we come to the same conclusion because it still does not provide any new virtual functions. If some base class does provide virtual functions, then it's that remote base class that's intended to be used polymorphically, not BasicProtocol itself, and so if anything we should be deriving from that remote base instead.

3. Yes, sometimes when you inherit publicly to get an interface, some implementation can come along too if the base class has both an interface that you want and some implementation of its own. This is nearly always possible to design away (see the next GotW), but it's not always necessary to take the true purist approach of "one responsibility per class."



# 015 Class Relationships - Part II
**Difficulty: 6 / 10**
Design patterns are an important tool in writing reusable code. Do you recognize the patterns used in this GotW? If so, can you improve them?

----

**Problem**
A database manipulation program often needs to do some work on every record (or selected records) in a given table, by first performing a read-only pass through the table to cache information about which records need to be processed and then performing a second pass to actually make the changes.

Instead of rewriting much of the common logic each time, a programmer has tried to provide a generic reusable framework in the following abstract class. The intent is that the abstract class should encapsulate the repetitive work by: first, compiling a list of table rows on which work needs to be done; and second, performing the work on each affected row. Derived classes are responsible for providing the details of their specific operations.

``` cpp
//---------------------------------------------------
// File gta.h
//---------------------------------------------------
class GenericTableAlgorithm {
public:
    GenericTableAlgorithm(const string& table);
    virtual ~GenericTableAlgorithm();
    
    // Process() returns true iff successful.
    // It does all the work: a) physically reads
    // the table's records, calling Filter() on each
    // to determine whether it should be included
    // in the rows to be processed; and b) when the
    // list of rows to operate upon is complete, calls
    // ProcessRow() for each such row.

    bool Process();

private:
    // Filter() returns true iff the row should be
    // included in the ones to be processed.  The
    // default action is to include every row.
    //
    virtual bool Filter(const Record&) {
      return true;
    }
    
    // ProcessRow() is called once per record that
    // was included for processing.  This is where
    // the concrete class does its specialized work.
    // (Note: This means every row to be processed
    // will be read twice, but assume that that is
    // necessary and not an efficiency consideration.)

    virtual bool ProcessRow(const PrimaryKey&) =0;
    
    class GenericTableAlgorithmImpl* pimpl_; // MYOB
};
```

For example, the client code to derive a concrete worker class and use it in a mainline looks something like this:

``` cpp
class MyAlgorithm : public GenericTableAlgorithm {
    // ... override Filter() and ProcessRow() do
    //     implement a specific operation ...
};

int main(int, char*[]) {
    MyAlgorithm a("Customer");
    a.Process();
}
```

The questions:

**1.** This is a good design and implements a well-known design pattern. Which pattern is this? Why is it useful here?

**2.** Without changing the fundamental design, critique the way this design was executed. What might you have done differently? What is the purpose of the "pimpl_" member?

**3.** This design can, in fact, be substantially improved. What are GenericTableAlgorithm's responsibilities? If more than one, how could they be better encapsulated? Explain how your answer affects the class' reusability, especially its extensibility.

----

**Solution**
**1.** This is a good design and implements a well-known design pattern. Which pattern is this? Why is it useful here?

This is the Template Method pattern (not to be confused with C++ templates).[1] It's useful because we can generalize a common way of doing something that always follows the same steps, and only the details may differ and can be supplied by a concrete derived class.

(Note: The pimpl_ idiom is superficially similar to Bridge[1], but here it's only intended to hide this particular class' own implementation as a compilation dependency firewall, not to act as a true extensible bridge.)

**2.** Without changing the fundamental design, critique the way this design was executed. What might you have done differently? What is the purpose of the "pimpl_" member?

This design uses bools as return codes, with apparently no other way (status codes or exceptions) of reporting failures. Depending on the requirements, this may be fine, but it's something to note.

The (intentionally pronounceable) pimpl_ member nicely hides the implementation behind an opaque pointer. The struct that pimpl_ points to will contain the private member functions and member variables, so that any change to them will not require client code to recompile. This is an important technique documented by Lakos[2] and others, because while it's a little annoying to code it does help compensate for C++'s lack of a module system.

**3.** This design can, in fact, be substantially improved. What are GenericTableAlgorithm's responsibilities? If more than one, how could they be better encapsulated? Explain how your answer affects the class' reusability, especially its extensibility.

GenericTableAlgorithm can be substantially improved because it currently holds two jobs. Just as humans get stressed when they have to hold two jobs \-\- because that means they're loaded up with extra and competing responsibilities \-\- so too this class could benefit from adjusting its focus.

In the original version, GenericTableAlgorithm is burdened with two different and unrelated responsibilities which can be effectively separated, since the two responsibilities are to support entirely different audiences. In short, they are:

   1. Client code USES the (suitably specialized) generic algorithm.
   
   2. GenericTableAlgorithm USES the specialized concrete "details" class to specialize its operation for a specific case or usage.

That said, let's look at some improved code:

``` cpp
//---------------------------------------------------
// File gta.h
//---------------------------------------------------

// Responsibility #1: Providing a public interface that encapsulates 
// common functionality as a template method.  This has nothing to do
// with inheritance relationships, and can be nicely isolated to stand
// on its own in a better-focused class.  
// The target audience is external users of GenericTableAlgorithm.

class GTAClient;

class GenericTableAlgorithm {
public:
    // Ctor now takes a concrete implementation object.

    GenericTableAlgorithm(const string& table, GTAClient& worker);
    
    // Since we've separated away the inheritance relationships, the dtor
    // doesn't need to be virtual. In fact, we may not need it at all.

    ~GenericTableAlgorithm();
    
    bool Process(); // unchanged

private:
    class GenericTableAlgorithmImpl* pimpl_; // MYOB
};
```

``` cpp
//---------------------------------------------------
// File gtaclient.h
//---------------------------------------------------

// Responsibility #2: Providing an abstract interface for extensibility.  
// This is an implementation detail of GenericTableAlgorithm that has nothing
// to do with its external clients, and can be nicely separated out into a 
// better-focused abstract protocol class.  The target audience is writers of 
// concrete "implementation detail" classes which work with (and extend) 
// GenericTableAlgorithm.

class GTAClient {
public:
    virtual ~GTAClient() =0;
    
    virtual bool Filter(const Record&) {
        return true;
    }
    
    virtual bool ProcessRow(const PrimaryKey&) =0;
};
```

As shown, these two classes should appear in separate header files. With these changes, how does this now look to the client code? The answer is, pretty much the same:

``` cpp
class MyWorker : public GTAClient {
    // ... override Filter() and ProcessRow() do
    //     implement a specific operation ...
};
```

``` cpp
int main(int, char*[]) {
    GenericTableAlgorithm a("Customer", MyWorker());
    a.Process();
}
```

While this may look pretty much the same, consider three important effects:

   1. What if `GenericTableAlgorithm`'s common public interface changes (e.g., a new public member is added)? In the original version, all concrete worker classes would have to be recompiled because they are derived from `GenericTableAlgorithm`. In this version, any change to `GenericTableAlgorithm`'s public interface is nicely isolated, and does not affect the concrete worker classes at all.
   
   2. What if `GenericTableAlgorithm`'s extensible protocol changes (e.g., if additional defaulted arguments were added to `Filter()` or `ProcessRow()`)? In the original version, all external clients of `GenericTableAlgorithm` would have to be recompiled even though the public interface is unchanged, because a derivation interface is visible in the class definition. In this version, any changes to `GenericTableAlgorithm`'s extension protocol interface is nicely isolated, and does not affect external users at all.
   
   3. Any concrete worker classes can now be used within any other algorithm which can operate using the `Filter()/ProcessRow()` interface, not just `GenericTableAlgorithm`.

In fact, what we've ended up with is very similar to the Strategy pattern.[1]

Remember the computer science motto: Most any problem can be solved by adding a level of indirection. Of course, it's wise to temper this with Occam's Razor: Don't multiply entities more than necessary. A proper balance between the two in this case delivers much better reusability and maintainability, at little or no cost... a good deal by all accounts!

You may have noticed that GenericTableAlgorithm could actually be a function instead of a class (in fact, some people might be tempted to rename `Process()` as `operator()()` since now the class apparently really is just a functor). The reason it could be replaced with a function is that the description doesn't say that it needs to keep state across calls to `Process()`. For example, if it does not need to keep state across invocations, we could replace it with:

```  cpp
bool GenericTableAlgorithm(const string& table, GTAClient& method) {
    // ... original contents of Process() go here ...
}

int main(int, char*[]) {
    GenericTableAlgorithm("Customer", MyWorker());
}
```

What we've really got here is a generic function, which can be given "specialized" behaviour as needed. If you know that 'method' objects never need to store state (that is, all instances are functionally equivalent and provide only the virtual functions), you can get fancy and make 'method' a non-class template parameter instead:

``` cpp
template<typename GTACworker>
bool GenericTableAlgorithm(const string& table) {
    // ... original contents of Process() go here ...
}

int main(int, char*[]) {
    GenericTableAlgorithm<MyWorker>("Customer");
}
```

I don't think that this buys you much here besides getting rid of a comma in the client code, so the first function is better. It's always good to resist the temptation to write cute code for its own sake.

At any rate, whether to use a function or a class in a given situation can depend on what you're trying to achieve, but in this case writing a generic function may be a better solution.

 

**Notes**
1. E. Gamma et al., **Design Patterns: Elements of Reusable Object-Oriented Software** (Addison-Wesley, 1995).

2. J. Lakos. **Large-Scale C++ Software Design** (Addison-Wesley, 1996).



# 016 Maximally Reusable Generic Containers
**Difficulty: 8 / 10**
How flexible can you make this simple container class? Hint: You'll learn more than a little about member templates along the way.

----

**Problem**
Implement copy construction and copy assignment for the following fixed-length vector class to provide maximum usability. Hint: Think about the kinds of things that client code might want to do.

```  cpp
template<typename T, size_t size>
class fixed_vector {
public:
    typedef T*       iterator;
    typedef const T* const_iterator;
    
    iterator       begin()       { return v_; }
    iterator       end()         { return v_+size; }
    const_iterator begin() const { return v_; }
    const_iterator end()   const { return v_+size; }

private:
    T v_[size];
};
```

**Notes**
   - Don't fix other things. This container is not intended to be fully STL-compliant, and has at least one subtle problem. It's only meant to illustrate some important issues in a simplified setting.

   - This example is adapted from one presented by Kevlin Henney and later analyzed by Jon Jagger in Issues 12 and 20 of the British C++ user magazine Overload. (British readers beware: The answer to this GotW goes well beyond that presented in Overload #20. In fact, the efficiency optimization presented there won't work in the solution that I'm going to post.)

----

**Solution**
**\[Note: This original solution contains some bugs since fixed in [Exceptional C++](http://www.gotw.ca/publications/xc++.htm) and the [Errata list](http://www.gotw.ca/publications/xc++-errata.htm).]**

Implement copy construction and copy assignment for the following fixed-length vector class to provide maximum usability. Hint: Think about the kinds of things that client code might want to do.

For this GotW solution, we'll do something a little different: I'll present the solution code, and your mission is to supply the explanation.

Q: What is the following solution doing, and why? Explain each constructor and operator.

```  cpp
template<typename T, size_t size>
class fixed_vector {
public:
    typedef T*       iterator;
    typedef const T* const_iterator;
    
    fixed_vector() { }
    
    template<typename O, size_t osize>
    fixed_vector(const fixed_vector<O,osize>& other) {
        copy(other.begin(), other.begin()+min(size,osize), begin());
    }
    
    template<typename O, size_t osize>
    fixed_vector<T,size>&
    operator=(const fixed_vector<O,osize>& other) {
        copy(other.begin(), other.begin()+min(size,osize), begin());
        return *this;
    }
    
    iterator       begin()       { return v_; }
    iterator       end()         { return v_+size; }
    const_iterator begin() const { return v_; }
    const_iterator end()   const { return v_+size; }

private:
    T v_[size];
};
```

Let's analyze this and see how well it measures up to what the question asked.

**Copy Construction and Copy Assignment**
First, note that the question as stated is a bit of a red herring: the original code already had a copy constructor and a copy assignment operator that worked fine. Our solution proposes to add a templated constructor and a templated assignment operator to make construction and assignment more flexible.

Congratulations to Valentin Bonnard and others who were quick to point out that the proposed copy constructor is not a copy constructor at all! In fact, we can go further: the proposed copy assignment operator is not a copy assignment operator at all, either.

Here's why: A copy constructor or copy assignment operator specifically `constructs/assigns` from another object of exactly the same type... including the same template arguments, if the class is templated. For example:

``` cpp
struct X {
    template<typename T>
    X(const T&);    // NOT copy ctor, T can't be X
    
    template<typename T>
    operator=(const T&);   // NOT copy ass't, T can't be X
};
```

"But," you say, "those two templated member functions could exactly match the signatures of copy construction and copy assignment!" Well, actually, no... they couldn't, because in both cases T may not be X. To quote from CD2 \[note: also appears in the later official standard of 1998; "CD2" was "Comittee Draft 2" as of 1995]:

\[12.8/2 note 4]

Because a template constructor is never a copy constructor, the presence of such a template does not suppress the implicit declaration of a copy constructor.

There's similar wording in \[12.8/9 note 7] for copy assignment. So the proposed solution in fact still has the same copy constructor and copy assignment operator as the original code did, because the compiler still generates the implicit versions. What we've done is extended the construction and assignment flexibility, not replaced the old versions. For example, consider the following program:

``` cpp
    fixed_vector<char,4> v;
    fixed_vector<int,4>  w;

    fixed_vector<int,4>  w2(w);    // calls implicit copy ctor
    fixed_vector<int,4>  w3(v);    // calls templated conversion ctor

    w = w2; // calls implicit assignment operator
    w = v;  // calls templated assignment operator
```

So what the question was really looking for was for us to provide flexible "construction and assignment from other fixed_vectors," not specifically flexible "copy construction and copy assignment" which already existed.

**Usability Issues for Construction and Assignment**
There are two major usability considerations:

1. Support varying types (including inheritance).

While fixed_vector definitely is and should remain a homogeneous container, sometimes it makes sense to construct or assign from another fixed_vector which actually contains different objects. As long as the source objects are assignable to our type of object, this should be allowed. For example, clients may want to write something like this:

``` cpp
    fixed_vector<char,4> v;
    fixed_vector<int,4>  w(v);  // copy
    w = v;                      // assignment

    class B { /*...*/ };
    class D : public B { /*...*/ };

    fixed_vector<D,4> x;
    fixed_vector<B,4> y(x);     // copy
    y = x;                      // assignment
```

2. Support varying sizes.

Similarly, clients may want to construct or assign from fixed_vectors with different sizes. Again, it makes sense to support this feature. For example:

``` cpp
    fixed_vector<char,6> v;
    fixed_vector<int,4>  w(v);  // copy 4 objects
    w = v;                      // assign 4 objects

    class B { /*...*/ };
    class D : public B { /*...*/ };

    fixed_vector<D,16> x;
    fixed_vector<B,42> y(x);    // copy 16 objects
    y = x;                      // assign 16 objects
```

**Alternative: The Standard Library Approach**
I happen to like the syntax and usability of the above functions, but there are still some nifty things they won't let you do. Consider another approach that follows the style of the standard library:

1. Copying.

``` cpp
template<Iter>
fixed_vector(Iter first, Iter last) {
  copy(first, first+min(size,last-first), begin());
}
```

Now when copying, instead of writing:

``` cpp
    fixed_vector<char,6> v;
    fixed_vector<int,4>  w(v);  // copy 4 objects
```

we need to write:

``` cpp
    fixed_vector<char,6> v;
    fixed_vector<int,4>  w(v.begin(), v.end());    // copy 4 objects
```

For construction, which style is better: the style of our proposed solution, or this standard library-like style? Here the former is somewhat easier to use and the latter is much more flexible (e.g., it allows users to choose subranges and copy from other kinds of containers), so you can take your pick or simply supply both flavours.

2. Assignment.

Note that we can't templatize assignment to take an iterator range, since `operator=()` may take only one parameter. Instead, we can provide a named function:

``` cpp
template<Iter>
fixed_vector<T,size>& assign(Iter first, Iter last) {
    copy(first, first+min(size,last-first), begin());
    return *this;
}
```

Now when assigning, instead of writing:

``` cpp
    w = v;                      // assign 4 objects
```

we need to write:

``` cpp
    w.assign(v.begin(), v.end());   // assign 4 objects
```

Technically, assign() isn't even necessary since we could still get the same flexibility without it, but that would be uglier and less efficient:

``` cpp
    w = fixed_vector<int,4>(v.begin(), v.end());   // assign 4 objects
```

For assignment, which style is better: the style of our proposed solution, or this standard library-like style? This time the flexibility argument doesn't hold water because the user can just as easily (and even more flexibly) write the copy himself. Instead of writing:

``` cpp
    w.assign(v.begin(), v.end());
```

the user just writes:

``` cpp
    copy v.begin(), v.end(), w.begin());
```

There's little reason to write `assign()` in this case, so for assignment it's probably best to use the technique from the proposed solution and let clients use `copy()` directly whenever subrange assignment is desired.

**Why Write the Default Constructor?**
Finally, why does the proposed solution also write an empty default constructor, which merely does the same thing as the compiler-generated default constructor? This is necessary because as soon as you define a constructor of any kind the compiler will not generate the default one for you, and clearly client code like the above requires it.

**Summary: What About Member Function Templates?**
Hopefully this GotW has convinced you that member function templates are definitely handy. I hope that it's also helped to show why they're widely used in the standard library. If you're not familiar with them already, don't despair... not all compilers support member templates today, but it's in the standard and so soon all compilers will. (As of this writing, Microsoft Visual C++ 5.0 can compile the solution, but it can't deduce the osize parameters in some of the example client code.)

Use member templates to good effect when creating your own classes and you'll likely not just have happy users, but have more of them, as they flock to reusing the code that's best-designed for reuse.



# 017 Casts
**Difficulty: 6 / 10**
How well do you know C++'s casts? Using them well can greatly improve the reliability of your code.

----

**Problem**
The new-style casts in (Draft) Standard C++ offer more power and safety than the old-style C casts. How well do you know them? The rest of this problem uses the following classes and global variables:

``` cpp
class  A             { /*...*/ };
class  B : virtual A { /*...*/ };
struct C : A         { /*...*/ };
struct D : B, C      { /*...*/ };

A a1; B b1; C c1; D d1;
const A a2;
const A& ra1 = a1;
const A& ra2 = a2;
char c;
```

**1.** Which of the following new-style casts are NOT equivalent to a C cast?

``` cpp
const_cast
dynamic_cast
reinterpret_cast
static_cast
```

**2.** For each of the following C casts, write the equivalent new-style cast. Which are incorrect if not written as a new-style cast?

``` cpp
void f() {
    A* pa; B* pb; C* pc;
    
    pa = (A*)&ra1;
    pa = (A*)&a2;
    pb = (B*)&c1;
    pc = (C*)&d1;
}
```

**3.** Critique each of the following C++ casts for style and correctness.

``` cpp
void g() {
    unsigned char* puc = static_cast<unsigned char*>(&c);
    signed char* psc = static_cast<signed char*>(&c);

    void* pv = static_cast<void*>(&b1);
    B* pb1 = static_cast<B*>(pv);

    B* pb2 = static_cast<B*>(&b1);

    A* pa1 = const_cast<A*>(&ra1);
    A* pa2 = const_cast<A*>(&ra2);

    B* pb3 = dynamic_cast<B*>(&c1);

    A* pa3 = dynamic_cast<A*>(&b1);

    B* pb4 = static_cast<B*>(&d1);

    D* pd = static_cast<D*>(pb4);

    pa1 = dynamic_cast<A*>(pb2);
    pa1 = dynamic_cast<A*>(pb4);

    C* pc1 = dynamic_cast<C*>(pb4);
    C& rc1 = dynamic_cast<C&>(*pb2);
}
```

----

**Solution**
The new-style casts in (Draft) Standard C++ offer more power and safety than the old-style C casts. How well do you know them? The rest of this problem uses the following classes and global variables:

``` cpp
class  A             { /*...*/ };
class  B : virtual A { /*...*/ };
struct C : A         { /*...*/ };
struct D : B, C      { /*...*/ };

A a1; B b1; C c1; D d1;
const A a2;
const A& ra1 = a1;
const A& ra2 = a2;
char c;
```

**1.** Which of the following new-style casts are NOT equivalent to a C cast?

Only dynamic_cast is not equivalent to some C cast. All other new-style casts have old-style equivalents.

**2.** For each of the following C casts, write the equivalent new-style cast. Which are incorrect if not written as a new-style cast?

``` cpp
void f() {
    A* pa; B* pb; C* pc;

    pa = (A*)&ra1;
```

Use const_cast: `pa = const_cast<A*>(&ra1);`

``` cpp
    pa = (A*)&a2;
```

This cannot be expressed as a new-style cast. The closest candidate is `const_cast`, but since a2 is a const object the results are undefined.

``` cpp
    pb = (B*)&c1;
```

Use reinterpret_cast: `pb = reinterpret_cast<B*>(&c1);`

``` cpp
    pc = (C*)&d1;
}
```

The above cast is wrong in C. In C++, no cast is required: `pc = &d1;`

**3.** Critique each of the following C++ casts for style and correctness.

First, a general note: We don't know whether any of these classes have virtual functions, and all of the following dynamic_casts are errors if the classes involved do not have virtual functions. For the rest of this discussion, we will assume that the classes do all have virtual functions, making the `dynamic_casts` legal.

``` cpp
void g() {
    unsigned char* puc = static_cast<unsigned char*>(&c);
    signed char* psc = static_cast<signed char*>(&c);
```

Error: we must use `reinterpret_cast` for both. This might surprise you at first, but the reason is that char, signed char, and unsigned char are three distinct types. Even though there are implicit conversions between them, they are unrelated, and so pointers to them are unrelated.

``` cpp
    void* pv = static_cast<void*>(&b1);
    B* pb1 = static_cast<B*>(pv);
```

These are both fine, but the first is unnecessary since there is already an implicit conversion from an object pointer to a `void*`.

``` cpp
    B* pb2 = static_cast<B*>(&b1);
```

This is fine, but unnecessary since the argument is already a `B*`.

``` cpp
    A* pa1 = const_cast<A*>(&ra1);
```

This is legal, but casting away const is usually indicative of poor style. Most of the cases where you legitimately would want to remove the const-ness of a pointer or reference are related to class members and covered by the '`mutable`' keyword. See GotW #6 for more discussion about const-correctness.

``` cpp
    A* pa2 = const_cast<A*>(&ra2);
```

Error: this will produce undefined behaviour if the pointer is used to write on the object, since `a2` really is a const object. To see why, consider that a compiler is allowed to see that `a2` is created as a const object and use that information to store it in read-only memory as an optimization. Casting away const on such an object is obviously dangerous.

Note: I showed no examples of using `const_cast` to convert a non-const pointer to a const pointer. The reason is that it's redundant; it is already legal to assign a non-const pointer to a const pointer. We only need `const_cast` to do the reverse.

``` cpp
    B* pb3 = dynamic_cast<B*>(&c1);
```

Error (if you try to use pb3): since c1 IS-NOT-A B (because C is not publicly derived from B, in fact it is not derived from B at all), this will set pb3 to null. The only legal cast would be a reinterpret_cast, and using that is almost always evil.

``` cpp
    A* pa3 = dynamic_cast<A*>(&b1);
```

Error: since b1 IS-NOT-A A (because B is not publicly derived from A, but its derivation is private), this is illegal.

``` cpp
    B* pb4 = static_cast<B*>(&d1);
```

This is fine, but not necessary since derived-to-base pointer conversions can be done implicitly.

``` cpp
    D* pd = static_cast<D*>(pb4);
```

This is fine, which may surprise you if you expected this to require a `dynamic_cast`. The reason is that downcasts can be static when the target is known, but beware: you are telling the compiler that you know for a fact that what is being pointed to really is of that type. If you are wrong, then the cast cannot inform you of the problem (as could `dynamic_cast`, which would return a null pointer if the cast failed) and at best you will get spurious runtime errors and/or program crashes.

``` cpp
    pa1 = dynamic_cast<A*>(pb2);
    pa1 = dynamic_cast<A*>(pb4);
```

These two look very similar. Both attempt to use `dynamic_cast` to convert a `B*` into an `A*`. However, the first is an error while the second is not.

Here's the reason: as noted above, you cannot use `dynamic_cast` to cast a pointer to what really is a B object (and here pb2 points to the object b1) into an A object, since B inherits privately, not publicly, from A. However, the second cast succeeds because pb4 points to the object d1, and D does have A as an indirect public base class (through C), and `dynamic_cast` is able to cast across the inheritance hierarchy using the path `B*` -> `D*` -> `C*` -> `A*`.

``` cpp
    C* pc1 = dynamic_cast<C*>(pb4);
```

This too is fine, for the same reason as the last: `dynamic_cast` can navigate the inheritance hierarchy and perform cross-casts, and so this is legal and will succeed.

``` cpp
    C& rc1 = dynamic_cast<C&>(*pb2);
}
```

Finally, this isn't fine... since `*pb2` isn't really a C, `dynamic_cast` will throw a bad_cast exception to signal failure. Why? Well, `dynamic_cast` can and does return null if a pointer cast fails, but since there's no such thing as a null reference it can't return a null reference if a reference cast fails. There's no way to signal such a failure to the client code besides throwing an exception, so that's what the standard `bad_cast` exception class is for.



# 018 Iterators
**Difficulty: 7 / 10**
Every programmer who uses the standard library has to be aware of these common and not-so-common iterator mistakes. How many of them can you find?

----

**Problem**
The following program has at least four iterator-related problems. How many can you find?

``` cpp
int main(int, char*[]) {
    vector<Date> e;
    copy(istream_iterator<Date>(cin), istream_iterator<Date>(), back_inserter(e));
    
    vector<Date>::iterator first = find(e.begin(), e.end(), "01/01/95");
    vector<Date>::iterator last = find(e.begin(), e.end(), "12/31/95");
    
    *last = "12/30/95";
    
    copy(first, last, ostream_iterator<Date>(cout, "\n"));
    
    e.insert(--e.end(), TodaysDate());
    
    copy(first, last, ostream_iterator<Date>(cout, "\n"));
}
```

----

**Solution**
The following program has at least four iterator-related problems. How many can you find?

``` cpp
int main(int, char*[]) {
    vector<Date> e;
    copy(istream_iterator<Date>(cin), 
         istream_iterator<Date>(), 
         back_inserter(e));
```
This is fine so far. The Date class writer provided an extractor function with the signature `operator>>(istream&, Date&)`, which is what `istream_iterator<Date>` uses to read the Dates from the cin stream. The copy algorithm just stuffs the Dates into the vector.

``` cpp
    vector<Date>::iterator first = find(e.begin(), e.end(), "01/01/95");
    vector<Date>::iterator last = find(e.begin(), e.end(), "12/31/95");

    *last = "12/30/95";
```

Error: This may be illegal, because '`last`' may be `e.end()` and therefore not a dereferenceable iterator.

The find algorithm retuns its second argument (the end iterator of the range) if the value is not found. In this case, if "12/31/95" is not in e, then '`last`' is equal to `e.end()` which points to one-past-the-end of the container and is not a valid iterator.

``` cpp
    copy( first, last, ostream_iterator<Date>(cout, "\n"));
```

Error: This may be illegal, because '`first`' may actually be after '`last`'.

If "01/01/95" is not found in e but "12/31/95" is, then the iterator '`last`' will point to something earlier in the collection (the Date object equal to "12/31/95") than does the iterator '`first`' (one past the end). However, copy requires that '`first`' must point to an earlier place in the same collection as '`last`'... that is, `\[first, last)` must be a valid range.

Unless you're using a checked version of the standard library, the likely symptom if this happens will be a difficult-to-diagnose core dump during or sometime after the copy.

``` cpp
    e.insert(--e.end(), TodaysDate()); 
```

Error: The expression "`--e.end()`" is illegal.

The reason is simple, if a little obscure: `vector<Date>::iterator` is simply a `Date*`, and you're not allowed to modify temporaries of builtin type. For example, the following plain-jane code is also illegal:

``` cpp
    Date* f();    // function that returns a Date*
    p = --f();    // error, but could be "f() - 1"
```

Fortunately, there's no loss of efficiency in writing this (more) correctly as:

``` cpp
    e.insert(e.end() - 1, TodaysDate()); 
```

Error: Now you still have the other error... if e is empty, `e.end() - 1` will not be a valid iterator.

``` cpp
    copy(first, last, ostream_iterator<Date>(cout, "\n")); 
}
```

Error: '`first`' and '`last`' may not be valid iterators any more.

A vector grows in "`chunks`" so that it won't have to reallocate its buffer every time you insert something into it. However, sometimes the vector will be full, and adding something will trigger a reallocation. Here, as a result of the insert operation, the vector may or may not grow. If it does, it will invalidate our existing iterators and the buggy copy will again generally manifest as a difficult-to-diagnose core dump.

Summary
When using iterators, be aware of four main issues:
   
   1. Valid values: Is the iterator dereferenceable? For example, writing "`*e.end()`" is always a logical error.
   
   2. Valid lifetimes: Is the iterator still valid when it's being used? Or has it been invalidated by some operation since we obtained it?
   
   3. Valid ranges: Is a pair of iterators a valid range? Is '`first`' really before (or equal to) '`last`'? Do both really point into the same container?
   
   4. Illegal builtin manipulation. For example, modifying a temporary of builtin type as in "`--e.end()`" above (fortunately, the compiler will catch this for you, and for iterators of class type the library author will often choose to allow this sort of thing for syntactic convenience).



# 019 Automatic Conversions
**Difficulty: 4 / 10**
Automatic conversions from one type to another can be extremely convenient. This GotW covers a typical example to illustrate why they're also extremely dangerous.

----

**Problem**
The standard string has no automatic conversion to a `const char*`. Should it?

----

Background: It is often useful to be able to access a string as a C-style const `char*`. Indeed, string has a member function `c_str()` to do just that, which returns the `const char*`. Here's the difference in client code:

``` cpp
    string s1("hello"), s2("world");

    strcmp(s1, s2);                // 1 (error)
    strcmp(s1.c_str(), s2.c_str()) // 2 (ok)
```

It would certainly be nice to do #1, but #1 is an error because strcmp requires two pointers and there's no automatic conversion from `string` to `const char*`. #2 is okay, but longer to write because we have to call `c_str()` explicitly. Wouldn't it be better if we could just write #1?

----

Solution
The standard string has no automatic conversion to a `const char*`. Should it?

No, with good reason. It's almost always a good idea to avoid writing automatic conversions, either as conversion operators or as single-argument non-explicit constructors.[1]

The two main reasons that implicit conversions are unsafe in general is that:
   
   a) they can interfere with overload resolution; and
   
   b) they can silently let "wrong" code compile cleanly.

If a string had an automatic conversion to `const char*`, that conversion could be implicitly called anywhere the compiler felt it was necessary. What this means is that you would get all sorts of subtle conversion problems \-\- the same ones you get into when you have non-explicit conversion constructors. It becomes far too easy to write code that looks right, is in fact not right and should fail, but by sheer coincidence will compile by doing something completely different from what was intended.

There are many good examples. Here's a simple one:

``` cpp
    string s1, s2, s3;

    s1 = s2 - s3;   // oops, probably meant "+"
```

The subtraction is meaningless and should be wrong. If string had an implicit conversion to `const char*`, however, this code would compile cleanly because the compiler would silently convert both `string`s to `const char*`'s and then subtract those pointers.

In summary, from the GotW coding standards:

   - avoid writing conversion operators (Meyers96: 24-31; Murray93: 38, 41-43; Lakos96: 646-650)
   

**Notes**
1. I've focused on the usual problems of implicit conversions, but there are other reasons why a string class should not have a conversion to const `char*`. Here are a few citations to further discussions:

   Koenig97: 290-292
   Stroustrup94 (D&E): 83

**Selected References**
|  | |
|---|---|
| Koenig97 | Andrew Koenig.<br/>"Ruminations on C++"<br/>Addison-Wesley, 1997 |
|Lakos96|         John Lakos.<br/>"Large-Scale C++ Software Design"<br/>Addison-Wesley, 1996|
|Meyers96|        Scott Meyers.<br/>"More Effective C++"<br/>Addison-Wesley, 1996|
|Murray93|        Robert Murray.<br/>"C++ Strategies and Tactics"<br/>Addison-Wesley, 1993|
|Stroustrup94<br/>(or D&E) |    Bjarne Stroustrup.<br/>"The Design and Evolution of C++"<br/>Addison-Wesley, 1994|



# 020 Code Complexity - Part I
**Difficulty: 9 / 10**
This problem presents an interesting challenge: How many execution paths can there be in a simple three-line function? The answer will almost certainly surprise you.

----

**Problem**
Without being given any additional information, how many execution paths could there be in the following code?

``` cpp
String EvaluateSalaryAndReturnName(Employee e) {
    if(e.Title() == "CEO" || e.Salary() > 100000)     {
        cout << e.First() << " " << e.Last() << " is overpaid" << endl;
    }
    return e.First() + " " + e.Last();
}
```

----

**Solution**
Assumptions:

**a)** Different orders of evaluating function parameters and exceptions thrown by destructors are ignored.[1]

Followup question for the intrepid:
How many more execution paths are there if destructors are allowed to throw?

**b)** Called functions are considered atomic. For example, the call "`e.Title()`" could throw for several reasons (e.g., it could throw an exception itself, it could fail to catch an exception thrown by another function it has called, or it could return by value and the temporary object's constructor could throw). All that matters to the function is whether performing the operation `e.Title()` results in an exception being thrown or not.

Answer: **23** (in just four lines of code!)

        If you found:       Rate yourself:
            3                   Average
            4-14                Exception-Aware
            15-23               Guru Material
    
        The 23 are made up of:
            - 3 non-exceptional code paths
            - 20 invisible exceptional code paths
For the non-exceptional execution paths, the trick was to know C/C++'s short-circuit evaluation rule:

**1.** If `e.Title() == "CEO"` then the second part of the condition doesn't need to be be evaluated (e.g., `e.Salary()` will never get called), but the cout will be performed.[2]

**2.** `If e.Title() != "CEO"` but `e.Salary() > 100000`, both parts of the condition will be evaluated and the cout will be performed.

**3.** `If e.Title() != "CEO"` and `e.Salary() <= 100000`, the cout will not be performed.

This leaves the exceptional execution paths:

``` cpp
String EvaluateSalaryAndReturnName(Employee e)
  ^*^                                       ^4^
```

**4.** The argument is passed by value, which invokes the Employee copy constructor. This copy operation might throw.

\*. String's copy constructor might throw while copying the temporary return value into the caller's area. We'll ignore this one, however, since it happens outside this function (and it turns out that we have enough execution paths of our own to keep us busy anyway!).

``` cpp
if(e.Title() == "CEO" || e.Salary() > 100000)
      ^5^   ^7^  ^6^ ^11^   ^8^    ^10^  ^9^
```

**5.** The `Title()` member function might itself throw, or it might return an object of class type by value, and that copy operation might throw.

**6.** To match a valid `operator==`, the string literal may need to be converted to a temporary object of class type (probably the same as `e.Title()`'s return type), and that construction of the temporary might throw.

**7.** If `operator==` is a programmer-supplied function, it might throw.

**8.** Similarly to #5, `Salary()` might itself throw, or it might return a temporary object and this construction operation might throw.

**9.** Similarly to #6, a temporary object may need to be constructed and this construction might throw.

**10.** Similarly to #7, this might be a programmer-provided function and therefore might throw.

**11.** Similarly to #7 and #10, this might be a programmer-provided function and therefore might throw (see note [2] again).

``` cpp
cout << e.First() << " " << e.Last()
         ^17^                ^18^
     << " is overpaid" << endl;
```

**12-16.** As documented in the draft standard, any of the five calls to `operator<<` might throw.

**17-18.** Similarly to #5, `First()` and/or `Last()` might throw, or each might return a temporary object and those construction operations might throw.

``` cpp
return e.First()  +  " "   +   e.Last();
         ^19^   ^22^ ^21^ ^23^  ^20^
```

**19-20.** Similarly to #5, `First()` and/or `Last()` might throw, or each might return a temporary object and those construction operations might throw.

**21.** Similarly to #6, a temporary object may need to be constructed and this construction might throw.

**22-23.** Similarly to #7, this might be a programmer-provided function and therefore might throw.

One purpose of this GotW was to demonstrate just how many invisible execution paths can exist in simple code in a language that allows exceptions. Does this invisible complexity affect the function's reliability and testability? See the following GotW problem for the answer.


**Notes**
   1. Never allow an exception to propagate from a destructor. Code that does this can't be made to work well. See [my article in the Nov/Dec 1997 issue of C++ Report](http://www.awlonline.com/cseng/meyerscddemo/DEMO/MAGAZINE/SU_FRAME.HTM) for more about Destructors That Throw and Why They're Evil.
   
   2. With suitable overloads for `==`, `||`, and`/`or `>` in the if's condition, the `||` could actually turn out to be a function call. If it is a function call, the short-circuit evaluation rule would be suppressed and both parts of the condition would be evaluated all the time.



# 021 Code Complexity - Part II
**Difficulty: 7 / 10**
The challenge: Take the three-line function from GotW #20 and make it strongly exception-safe. This exercise illustrates some important lessons about exception safety.

----

**Problem**
Consider again the function from GotW #20. Is it exception-safe (works properly in the presence of exceptions) and exception-neutral (propagates all exceptions to the caller)?

``` cpp
String EvaluateSalaryAndReturnName(Employee e) {
    if(e.Title() == "CEO" || e.Salary() > 100000) {
        cout << e.First() << " " << e.Last() << " is overpaid" << endl;
    }
    return e.First() + " " + e.Last();
}
```

Explain your answer. If it is exception-safe, does it support the basic guarantee or the strong guarantee? If not, how must it be changed to support either guarantee?

Assume that all called functions are exception-safe (might throw but do not have side effects if they do throw), and that any objects being used, including temporaries, are exception-safe (clean up their resources when destroyed).

**Background: The Basic and Strong Guarantees**
For details on the basic and strong guarantees, see [my articles in the September and November/December 1997 issues of C++ Report](http://www.awlonline.com/cseng/meyerscddemo/DEMO/MAGAZINE/SU_FRAME.HTM). In brief, the first guarantees destructibility and no leaks, while the second additionally guarantees full commit-or-rollback semantics.

----

**Solution**
Consider again the function from GotW #20. Is it exception-safe (works properly in the presence of exceptions) and exception-neutral (propagates all exceptions to the caller)?

``` cpp
String EvaluateSalaryAndReturnName(Employee e) {
    if(e.Title() == "CEO" || e.Salary() > 100000) {
        cout << e.First() << " " << e.Last() << " is overpaid" << endl;
    }
    return e.First() + " " + e.Last();
}
```

**A Word About Assumptions**
As the problem stated, we will assume that all called functions \-\- including the stream functions \-\- are exception-safe (might throw but do not have side effects), and that any objects being used, including temporaries, are exception-safe (clean up their resources when destroyed).

Streams happen to throw a monkey wrench into this because of their possible "un-rollbackable" side effects. For example, operator<< might throw after emitting part of a string which can't be "un-emitted"; also, the stream's error state may be set. We will ignore those issues for the most part; the point of this discussion is to examine how to make a function exception-safe when the function is specified to have two distinct side effects.

**Basic vs. Strong Guarantees**
As written, this function satisfies the basic guarantee: in the presence of exceptions, the function does not leak resources.

This function does not satisfy the strong guarantee. The strong guarantee is that, if the function fails due to an exception, program state must not be changed. This function, however, has two distinct side effects (as hinted at in the function's name):
   
   1. An "...overpaid..." message is emitted to cout.
   
   2. A name string is returned.

As far as the latter is concerned, the function already meets the strong guarantee, since if an exception occurs the value will never be returned. As far as the former is concerned, the function is not exception-safe for two reasons:
   
   1. If an exception is thrown after the first part of the message has been emitted to cout but before the message has been completed (e.g., if the fourth << throws), then a partial message was emitted to cout.[1]
   
   2. If the message was emitted successfully but an exception occurs later in the function (e.g., during the assembly of the return value), then a message was emitted to cout even though the function failed because of an exception.

Instead, to meet the strong guarantee, the behaviour should be that either both effects are completed, or an exception is thrown and neither effect is performed.

Can we accomplish this? Here's one way we might try it (call this Attempt #1):

``` cpp
String EvaluateSalaryAndReturnName(Employee e) {
    String result = e.First() + " " + e.Last();
    
    if(e.Title() == "CEO" || e.Salary() > 100000)
    {
        String message = e.First() + " " + e.Last() + " is overpaid\n";
        cout << message;
    }
    return result;
}
```

This isn't bad. Note that we've replaced the endl with a newline character (which isn't exactly equivalent) in order to get the entire string into one << call. (Of course, this doesn't guarantee that the underlying stream system won't itself fail partway through writing the message, resulting in incomplete output, but we've done the best we can at this high level.)

**A Little Bothersome Issue**
We still have one minor quibble, however, as illustrated by the following client code:

``` cpp
    String theName;
    theName = EvaluateSalaryAndReturnName(bob);
```

The String copy constructor is invoked because the result is returned by value, and the copy assignment operator is invoked to copy the result into theName. If either copy fails, then the function has completed its side effects (since the message was completely emitted and the return value was completely constructed) but the result has been irretrievably lost (oops).

Can we do better, and perhaps avoid the problem by avoiding the copy? For example, we could let the function take a non-const String reference parameter and place the return value in that:

``` cpp
void EvaluateSalaryAndReturnName(Employee e, String& r) {
    String result = e.First() + " " + e.Last();
    
    if(e.Title() == "CEO" || e.Salary() > 100000) {
        String message = e.First() + " " + e.Last() + " is overpaid\n";
        cout << message;
    }
    r = result;
}
```

Unfortunately, the assignment to r might still fail, which leaves us with one side effect complete and the other incomplete. Bottom line, this attempt doesn't really buy us much.

We might try returning the result in an auto_ptr (call this Attempt #3):

``` cpp
auto_ptr<String> EvaluateSalaryAndReturnName(Employee e) {
    auto_ptr<String> result = new String(e.First() + " " + e.Last());
    
    if(e.Title() == "CEO" || e.Salary() > 100000) {
        String message = e.First() + " " + e.Last() + " is overpaid\n";
        cout << message;
    }
    return result;  // rely on transfer of ownership
}
```

This does the trick, since we have effectively hidden all of the work to construct the second side effect (the return value) while ensuring that it can be safely returned to the caller using only nonthrowing operations after the first side effect has completed (the printing of the message). The price? As often happens when implementing strong exception safety, the strong safety comes at the cost of efficiency \-\- here, the extra dynamic memory allocation.

**Exception Safety and Multiple Side Effects**
In this case, it turned out to be possible in Attempt #3 to perform both side effects with essentially commit-or-rollback semantics (except for the stream issues). The reason it was possible is that there turned out to be a technique by which the two effects could be performed atomically... that is, all of the "real" preparatory work for both could be completed in such a way that actually performing the visible side effects could be done using only nonthrowing operations.

Even though we were lucky this time, it's not always that simple: It's impossible to write strongly exception-safe functions that have two or more unrelated side effects that cannot be performed atomically (for example, what if the two side effects had been to emit one message to cout and another to cerr?), since the strong guarantee is that in the presence of exceptions "program state will remain unchanged"... in other words, if there's an exception, there must be no side effects. When you come across a case where the two side effects cannot be made to work atomically, usually the only way to get strong exception safety is to split the one function into two others that can be performed atomically.

This GotW should illustrate three important things:
   
   1. Providing the strong exception safety guarantee often (but not always) requires you to trade off performance.
   
   2. If a function has multiple unrelated side effects, it cannot always be made strongly exception-safe. If not, it can only be done by splitting the function it into several functions each of whose side effects can be performed atomically.
   
   3. Not all functions need to be strongly exception-safe. Both the original code and Attempt #1 satisfy the basic guarantee. For many clients, Attempt #1 is sufficient and minimizes the opportunity for side effects to occur in the exceptional situations, without requiring the performance tradeoffs of Attempt #3.

**Postscript: Streams and Side Effects**
As it turns out, our assumption that no called function has side effects is not entirely true. In particular, there is no way to guarantee that the stream operations will not fail after partly emitting a result. This means that we can't get true commit-or-rollback fidelity from any function that performs stream output, at least not on these standard streams. Another issue is that, if the stream output fails, the stream state will have changed. We currently do not check for that or recover from it, but it is possible to further refine the function to catch stream exceptions and reset cout's error flags before rethrowing the exception to the caller.

 
**Notes**
1. If you're thinking that it's a little pedantic to worry about whether a message is completely cout'ed or not, you're partly right. In this case, maybe no one would care. However, the same principle applies to any function that attempts to perform two side effects, and that's why the following discussion is useful.



# 022 Object Lifetimes - Part I
**Difficulty: 5 / 10**
"To be, or not to be..." When does an object actually exist? This problem considers when an object is safe to use.

----

**Problem**
Critique the following code fragment. Is the code in #2 safe and/or legal? Explain.
``` cpp
void f() {
    T t(1);
    T& rt = t;
    // #1: do something with t or rt
    t.~T();
    new (&t) T(2);
    // #2: do something with t or rt
}   // t is destroyed again
```

----

**Solution**
Yes, #2 is safe and legal (if you get to it), but:
   
   a) the function as a whole is not safe; and
   
   b) it's a bad habit to get into.

**Why #2 Is Safe (If You Get To It)**
The draft explicitly allows this code. The reference rt is not invalidated by the in-place destruction and reconstruction. (Of course, you can't use t or rt between the `t.~T()` and the placement new, since during that time no object exists. We're also assuming that `T::operator&()` hasn't been overloaded to do something other than return the object's address.)

The reason we say #2 is safe "if you get to it" is that `f()` as a whole may not be exception-safe:

**Why the Function Is Not Safe**
If T's constructor may throw in the `T(2)` call, then `f()` is not exception-safe. Consider why: If the `T(2)` call throws, then no new object has been reconstructed in the '`t`' memory area, yet at the end of the function `T::~T()` is naturally called (since t is an automatic variable) and "t is destroyed again" like the comment says. That is, '`t`' will be constructed once but destroyed twice (oops). This is likely to create unpredictable side effects, such as core dumps.

**Why This Is a Bad Habit**
Ignoring the exception safety issues, the code happens to work in this setting because the programmer knows the complete type of the object being constructed and destroyed. That is, it was a `T` and is being destroyed and reconstructed as a `T`.

This technique is rarely if ever necessary in real code, and is a very bad habit to get into because it's fraught with (sometimes subtle) dangers if it appears in a member function:
``` cpp
void T::f(int i) {
    this->~T();
    new (this) T(i);
}
```
Now is this technique safe? In general, No. Consider the following code:
``` cpp
class U : /*...*/ public T { /* ... */ };

void f() {
    /*AAA*/ t(1);
    /*BBB*/& rt = t;
    // #1: do something with t or rt
    t.f(2);
    // #2: do something with t or rt
}   // t is destroyed again
```
If "`/*AAA*/`" is "`T`", the code in #2 will still work, even if "`/*BBB*/`" is not "`T`" (it could be a base class of `T`).

If "`/*AAA*/`" is "`U`", all bets are off no matter what "`/*BBB*/`" is. Probably the best you can hope for is an immediate core dump, because the call `t.f()` "slices" the object. Here, the slicing issue is that `t.f()` replaces the original object with another object of a different type \-\- that is, a `T` instead of a `U`. Even if you're willing to write nonportable code, there's no way of knowing whether the object layout of a `T` is even usable as a `U` when superimposed in memory where a `U` used to be. Chances are good that it's not. Don't go there... this is never a good practice.

This GotW has covered some basic safety and slicing issues of in-place destruction and reconstruction. This sets the stage for the followup question in [GotW #23: Object Lifetimes - Part II](http://www.gotw.ca/gotw/023.htm).



# 023 Object Lifetimes - Part II
Difficulty: 6 / 10
Following up from #22, this issue considers a C++ idiom that's frequently recommended... but often dangerously wrong.



Problem
Critique the following idiom (shown as commonly presented):
``` cpp
    T& T::operator=(const T& other) {
        if(this != &other) {
            this->~T();
            new (this) T(other);
        }
        return *this;
    }
```
1. What legitimate goal does it try to achieve? Correct any coding flaws in the version above.

2. Even with any flaws corrected, is this idiom safe? Explain. If not, how else should the programmer achieve the intended results?

(See also GotW #22, and the October 1997 C++ Report.)



Solution
Critique the following idiom (shown as commonly presented):
``` cpp
    T& T::operator=(const T& other) {
        if(this != &other) {
            this->~T();
            new (this) T(other);
        }
        return *this;
    }
```
Summary[1]
This idiom is frequently recommended, and it appears as an example in the draft standard.[2] It is also poor form and, if anything, exactly backwards. Don't do it.

1. What legitimate goal does it try to achieve?

This idiom expresses copy assignment in terms of copy construction. That is, it's trying to make sure that T's copy assignment and its copy constructor do the same thing, which keeps the programmer from having to needlessly repeat the same code in two places.

This is a noble goal. After all, it makes programming easier when you don't have to write the same thing twice, and if T changes (e.g., gets a new member variable) you can't forget to update one of the functions when you update the other.

This idiom could be particularly useful when there are virtual base classes that have data members, which would otherwise be assigned incorrectly at worst or multiple times at best. While this sounds good, it's not really much of a benefit in reality because virtual base classes shouldn't have data members anyway.[3] Also, if there are virtual base classes it means that this class is designed for inheritance, which (as we're about to see) means we can't use this idiom because it is too dangerous.

Correct any coding flaws in the version above.

The code above has one flaw that can be corrected, and several others that cannot.

Problem #1: It Can Slice Objects
The line "this->~T();" does the wrong thing if T is a base class with a virtual destructor. When called on an object of a derived class, it will destroy the derived object and replace it with a T object. This will almost certainly break any subsequent code that tries to use the object. (See GotW #22 for more discussion about slicing.)

In particular, this makes life a living hell for authors of derived classes (and there are other potential traps for derived classes, see below). Recall that derived assignment operators are normally written in terms of the base's assignment as follows:
``` cpp
    Derived&
    Derived::operator=(const Derived& other) {
        Base::operator=(other);
        // ...now assign Derived members here...
        return *this;
    }
```
In this case, we get:
``` cpp
    class U : /* ... */ T { /* ... */ };

    U& U::operator=(const U& other) {
        T::operator=(other);
        // ...now assign U members here... oops
        return *this;                   // oops
    }
```
As written, the call to T::operator=() silently breaks all of the code after it (both the U member assignments and the return statement). This will often manifest as a mysterious and hard-to-debug runtime error if the U destructor doesn't reset its data members to invalid values.

To correct this problem, the code could call "this->T::~T();" instead, which ensures that for a derived object only the T subobject will be replaced (rather than the whole derived object be sliced and wrongly transformed into a T). This replaces an obvious danger with a subtler one that can still affect authors of derived classes (see below).

2. Even with any flaws corrected, is this idiom safe? Explain.

No. Note that none of the following problems can be fixed without giving up on the entire idiom:

Problem #2: It's Not Exception-Safe
The 'new' statement will invoke the T copy constructor. If that constructor can throw (and many/most classes do report constructor errors by throwing an exception), then the function is not exception-safe because it will end up destroying the old object without replacing it with anything.

Like slicing, this flaw will break any subsequent code that tries to use the object. Worse, it will probably cause the program to attempt to destroy the same object twice since the outside code has no way of knowing that the destructor for this object has already been run. (See GotW #22 for more discussion about double destruction.)

Problem #3: It's Inefficient for Assignment
This idiom is inefficient because construction almost always involves more work than resetting values during assignment. Destruction and reconstruction done together involve even more work.

Problem #4: It Changes Normal Object Lifetimes
This idiom breaks any code that relies on normal object lifetimes. In particular, it breaks or interferes with all classes that use the common "initialization is resource acquisition" idiom. In general, it breaks or interferes with any class whose constructor or destructor has side effects.

For example, what if T (or any base class of T) acquires a mutex lock or starts a database transaction in its constructor and frees the lock or transaction in its destructor? Then the lock/transaction will be incorrectly released and reacquired during an assignment \-\- typically breaking both client code and the class itself. Besides T and T's base classes, this can also break T's derived classes if they rely on T's normal lifetime semantics.

Some will say, "But of course I'd never do this in a class that acquires/releases a mutex in its ctor/dtor!" The short answer is, "Really? And how do you know that none of your (direct or indirect) base classes does so?" Frankly, you often have no way of knowing this, and you should never rely on your base classes' working properly in the face of playing unusual games with object lifetimes.

The fundamental problem is that this idiom subverts the meaning of construction and destruction. Construction and destruction correspond exactly to the beginning/end of an object's lifetime, at which times the object typically acquires/releases resources. Construction and destruction are not meant to be used to change an object's value (and in fact they do not; they actually destroy the old object and replace it with a lookalike that happens to carry the new value, which is not the same thing at all).

Problem #5: It Can Still Break Derived Classes
With Problem #1 solved by calling "this->T::~T();" instead, this idiom only replaces the "T part" (or "T subobject") within a derived object. Many derived classes can react well to having their objects' base parts swapped out and in like this, but some may not.

In particular, any derived class that takes responsibility for its base class' state could fail if its base parts are modified without its knowledge (and invisibly destroying and reconstructing an object certainly counts as modification). This danger can be mitigated as long as the assignment doesn't do anything extraordinary or unexpected from what a "normally written" assignment operator would do.

Problem #6: It Relies on Unreliable Pointer Comparisons
The idiom relies completely on the "this != &other" test. (If you doubt that, consider what happens in the case of self-assignment.)

The problem is that that test is not guaranteed to do what you might think: While the standard guarantees that pointers to the same object must compare equal, it doesn't guarantee that pointers to different objects must compare unequal. If this happens, the assignment won't be done when it should. (See GotW #11 for more about the "this != &other" test.)

For those who think that this is pedantic, a brief thought from GotW #11: Any copy assignment that "must" check for self-assignment is not exception-safe.[4] [Note: See Exceptional C++ and the Errata page for updated information.]

There are other potential hazards that can affect client code and/or derived classes (such as behaviour in the presence of virtual assignment operators, which are always a bit tricky at the best of times), but this should be enough to demonstrate that the idiom has some serious problems.

So What Should We Do Instead?
If not, how else should the programmer achieve the intended results?

The idea of having one member function do the work of both kinds of copying (copy construction and copy assignment) is good: It means that the code only needs to be written and maintained in one place. This idiom just chose the wrong common function, that's all.

If anything, the idiom is exactly backwards: copy construction should be implemented in terms of copy assignment, rather than the reverse. For example:
``` cpp
    T::T( const T& other ) {
      /* T:: */ operator=( other );
    }

    T& T::operator=( const T& other ) {
      // the real work goes here
      // (presumably done exception-safely, but now it
      // can throw whereas throwing broke us before)
      return *this;
    }
```
This has all the benefits of the original idiom, and none of the problems.[5] For prettiness, you might write a common private helper function that does the real work, but it's pretty much the same:
``` cpp
    T::T( const T& other ) {
      do_copy( other );
    }

    T& T::operator=( const T& other ) {
      do_copy( other );
      return *this;
    }

    T& T::do_copy( const T& other ) {
      // the real work goes here
      // (presumably done exception-safely, but now it
      // can throw whereas throwing broke us before)
    }
```
Conclusion
The original idiom is full of pitfalls, it's often wrong, and it makes life a living hell for the authors of derived classes. I'm sometimes tempted to post the above code in the office kitchen with the caption: "Here be dragons."

From the GotW coding standards:

- prefer writing a common private function to share code between copying and copy assignment, if necessary; never use the trick of implementing copy assignment in terms of copy construction by using an explicit destructor followed by placement new, even though this trick crops up every three months on the newsgroups (i.e., never write:
``` cpp
        T& T::operator=( const T& other )
        {
            if( this != &other)
            {
                this->~T();             // evil
                new (this) T( other );  // evil
            }
            return *this;
        }
```

Notes
1. I'm ignoring pathological cases (e.g., overloading T::operator&() to do something other than return this). GotW #11 mentioned a few.

2. In the draft standard, this example was intended to demonstrate the object lifetime rules, not to recommend a good practice (it isn't!). For those interested, here it is (slightly edited for space) from 3.8/7:
``` cpp
[Example:

    struct C {
        int i;
        void f();
        const C& operator=( const C& );
    };
    const C& C::operator=( const C& other)
    {
        if ( this != &other )
        {
            this->~C();     // lifetime of '*this' ends
            new (this) C(other);
                            // new object of type C created
            f();            // well-defined
        }
        return *this;
    }
    C c1;
    C c2;
    c1 = c2; // well-defined
    c1.f();  // well-defined; c1 refers to
             //  a new object of type C
--end example]
```
As further proof that this is not intended to recommend good practice, note that here C::operator=() returns a const C& rather than a plain C&, which needlessly prevents portable use of these objects in standard library containers.

From the GotW coding standards:

    - declare copy assignment as "T& T::operator=(const T&)"
    
    - don't return const T&; although this would be nice since it prevents usage like "(a=b)=c", it would mean you couldn't portably put T objects into standard library containers, since these require that assignment returns a plain T& (Cline95: 212; Murray93: 32-33)

3. See also Scott Meyers' "Effective C++".

4. While you can't rely on the "this != &other" test, there's nothing wrong with using it as an attempt to optimize away known self-assignments. If it works, you've saved yourself an assignment. If it doesn't, of course, your assignment operator should still be written in such a way that it's safe for self-assignment. There are arguments both for and against using this test as an optimization, but that's beyond the scope of this GotW.

5. True, it still requires a default constructor and it may still not be optimally efficient, but you can only get optimal efficiency by using initializer lists (initializing the member variables during construction as one step, rather than constructing them and then assigning them as two steps). Of course, doing that would sacrifice the code commonality, and arguing those tradeoffs is again beyond the scope of this GotW.



# 024 Compilation Firewalls
Difficulty: 6 / 10
Using the Pimpl Idiom can dramatically reduce code interdependencies and build times. But what should go into a pimpl_ object, and what is the safest way to use it?



Problem
In C++, when anything in a class definition changes (even private members) all users of that class must be recompiled. To reduce these dependencies, a common technique is to use an opaque pointer to hide some of the implementation details:
``` cpp
    class X {
    public:
      /* ... public members ... */
    protected:
      /* ... protected members? ... */
    private:
      /* ... private members? ... */
      class XImpl* pimpl_;  // opaque pointer to
                            // forward-declared class
    };
```
Questions
1. What should go into XImpl? There are four common disciplines, including:

- put all private data (but not functions) into XImpl;

- put all private members into XImpl;

- put all private and protected members into XImpl;

- make XImpl entirely the class that X would have been, and write X as only the public interface made up entirely of simple forwarding functions (a handle/body variant).

What are the advantages/drawbacks? How would you choose among them?

2. Does XImpl require a "back pointer" to the X object?



Solution
First, two definitions:

visible class : the class the client code sees and manipulates (here X)

pimpl : the implementation class (here XImpl) hidden behind an opaque pointer (the eponymous pimpl_) in the visible class

In C++, when anything in a class definition changes (even private members) all users of that class must be recompiled. To reduce these dependencies, a common technique is to use an opaque pointer to hide some of the implementation details:

This is a variant of the handle/body idiom. As documented by Coplien,[1] it was described as being primarily useful for reference counting of a shared implementation.

As it turns out, handle/body (in the form of what I call the "pimpl idiom" because of the intentionally pronounceable "pimpl_" pointer)[2] is also useful for breaking compile-time dependencies, as pointed out by Lakos.[3] The rest of this solution concentrates on that usage, and some of the following is not true for handle/body in general.

The major costs of this idiom are in performance:

1. Each construction must allocate memory. This can be mitigated using a custom allocator, but that's more work.

2. Each access of a hidden member requires at least one extra indirection. (If the hidden member being accessed itself uses a back pointer to call a function in the visible class, there will be multiple indirections.)

1. What should go into XImpl? There are four common disciplines, including:

- put all private data (but not functions) into XImpl;

This is a good start, because now we can forward-declare any class which only appears as a data member (rather than #include the class' actual declaration, which would make client code depend on that too). Still, we can usually do better.

- put all private members into XImpl;

This is (almost) my usual practice these days. After all, in C++, the phrase "client code shouldn't and doesn't care about these parts" is spelled "private," and privates are best hidden (except in some Scandinavian countries with more liberal laws).

There are two caveats, the first of which is the reason for my "almost" above:

1. You can't hide virtual member functions in the pimpl class, even if the virtual functions are private. If the virtual function overrides one inherited from a base class, then it must appear in the actual derived class. If the virtual function is not inherited, then it must still appear in the visible class in order to be available for overriding by further derived classes.

2. Functions in the pimpl may require a "back pointer" to the visible object if they need to use other functions, which adds another level of indirection. By convention this back pointer is usually named self_ where I've worked.

- put all private and protected members into XImpl;

Taking this extra step is actually wrong. Protected members should never go into a pimpl, since putting them there just emasculates them. After all, protected members exist specifically to be seen and used by derived classes, and so aren't nearly as useful if derived classes can't see or use them.

- make XImpl entirely the class that X would have been, and write X as only the public interface made up entirely of simple forwarding functions (a handle/body variant).

This is useful in a few restricted cases, and has the benefit of avoiding a back pointer since all services are available in the pimpl class. The chief drawback is that it normally makes the visible class useless for any inheritance, as either a base or a derived class.

2. Does XImpl require a "back pointer" to the X object?

Often, unhappily, yes. After all, what we're doing is splitting each object into two halves for the purposes of hiding one part.

Whenever a function in the visible class is called, usually some function or data in the hidden half is needed to complete the request. That's fine and reasonable. What's perhaps not as obvious at first is that often a function in the pimpl must call a function in the visible class, usually because the called function is public or virtual.

 

Notes
1. James O. Coplien. Advanced C++ Programming Styles and Idioms (Addison-Wesley, 1992).

2. I always used to write impl_. The eponymous pimpl_ was actually coined by friend and colleague Jeff Sumner, who shares my penchant for Hungarian-style "p" prefixes for pointer variables and who has an occasional taste for horrid puns.

3. J. Lakos. Large-Scale C++ Software Design (Addison-Wesley, 1996).

# 025 SPECIAL EDITION: auto_ptr
Difficulty: 8 / 10
This GotW covers basics about how you can use the standard auto_ptr safely and effectively. (This GotW Special Edition was written in honor of the voting out of the Final Draft International Standard for Programming Language C++, which included a last-minute auto_ptr change.)



Problem
Comment on the following code: What's good, what's safe, what's legal, and what's not?
``` cpp
    auto_ptr<T> source() { return new T(1); }
    void sink( auto_ptr<T> pt ) { }

    void f() {
        auto_ptr<T> a( source() );
        sink( source() );
        sink( auto_ptr<T>( new T(1) ) );

        vector< auto_ptr<T> > v;
        v.push_back( new T(3) );
        v.push_back( new T(4) );
        v.push_back( new T(1) );
        v.push_back( a );
        v.push_back( new T(2) );
        sort( v.begin(), v.end() );

        cout << a->Value();
    }

    class C {
    public:    /*...*/
    protected: /*...*/
    private:
        auto_ptr<CImpl> pimpl_;
    };
```

Solution
Comment on the following code: What's good, what's safe, what's legal, and what's not?

STANDARDS UPDATE: This week [the week this GotW was posted], at the WG21/J16 meeting in Morristown NJ USA, the Final Draft International Standard (FDIS) for Programming Language C++ was voted out for balloting by national bodies. We expect to know by the next meeting (Nice, March 1998) whether it has passed and will become an official ISO Standard.

This GotW was posted knowing that auto_ptr was going to be refined at the New Jersey meeting in order to satisfy national body comments. This Special Edition of GotW covers the final auto_ptr, how and why been made safer and easier to use, and how to use it best.

In summary:

1. All legitimate uses of auto_ptr work as before, except that you can't use (i.e., dereference) a non-owning auto_ptr.

2. The dangerous abuses of auto_ptr have been made illegal.

SOME WELL-DESERVED ACKNOWLEDGMENTS: Many thanks from all of us to Bill Gibbons, Greg Colvin, Steve Rumsby, and others who worked hard on the final refinement of auto_ptr. Greg in particular has laboured over auto_ptr and related classes for many years to satisfy various committee concerns and requirements, and deserves public recognition for that work.

Background
The original motivation for auto_ptr was to make code like the following safer:
``` cpp
    void f() {
        T* pt( new T );
        /*...more code...*/
        delete pt;
    }
```

If f() never executes the delete statement (either because of an early return or by an exception thrown in the function body), the allocated object is not deleted and we have a classic memory leak.

A simple way to make this safe is to wrap the pointer in a "smarter" pointer-like object which owns the pointer and which, when destroyed, deletes the pointer automatically:
``` cpp
    void f() {
        auto_ptr<T> pt( new T );
        /*...more code...*/
    } // cool: pt's dtor is called as it goes out of
      // scope, and the allocated object is deleted
```
Now the code will not leak the T object, no matter whether the function exits normally or by means of an exception, because pt's destructor will always be called during stack unwinding. Similarly, auto_ptr can be used to safely wrap pointer data members [note: there are important safety details not mentioned in this GotW; see later GotW issues including GotW #62, and the book Exceptional C++]:
``` cpp
    // file c.h
    class C {
    public:
        C();
        /*...*/
    private:
        auto_ptr<CImpl> pimpl_;
    };

    // file c.cpp
    C::C() : pimpl_( new CImpl ) { }
```
Now the destructor need not delete the pimpl_ pointer, since the auto_ptr will handle it automatically. We'll revisit this example again at the end.

Sources and Sinks
This is cool stuff all by itself, but it gets better. Based on Greg Colvin's work and experience at Taligent, people noticed that if you defined copying for auto_ptrs then it would be very useful to pass them to and from functions, as function parameters and return values.

This is in fact the way auto_ptr worked in the second committee draft (Dec 1996), with the semantics that the act of copying an auto_ptr transfers ownership from the source to the target. After the copy, only the target auto_ptr "owned" the pointer and would delete it in due time, while the source also still contained the same pointer but did not "own" it and therefore would not delete it (else we'd have a double delete). You could still use the pointer through either an owning or a non-owning auto_ptr object.

For example:
``` cpp
    void f() {
        auto_ptr<T> pt1( new T );
        auto_ptr<T> pt2;

        pt2 = pt1;  // now pt2 owns the pointer, and
                    // pt1 does not

        pt1->DoSomething(); // ok (before last week)
        pt2->DoSomething(); // ok

    } // as we go out of scope, pt2's dtor deletes the
      // pointer, but pt1's does nothing
```
This gets us to the first part of the GotW code:[1]
``` cpp
    auto_ptr<T> source() { return new T(1); }
    void sink( auto_ptr<T> pt ) { }
```
  SUMMARY
  |         Before NJ   After NJ
  | Legal?     Yes         Yes
  | Safe?      Yes         Yes
This demonstrates exactly what the people at Taligent had in mind:

1. source() allocates a new object and returns it to the caller in a completely safe way, by letting the caller assume ownership of the pointer. Even if the caller ignores the return value (of course, you would never write code that ignores return values, right?), the allocated object will always be safely deleted.

See also GotW #21, which demonstrates why this is an important idiom, since returning a result by wrapping it in an auto_ptr is sometimes the only way to make a function strongly exception-safe.

2. sink() takes an auto_ptr by value and therefore assumes ownership of it. When sink() is done, the deletion is done (as long as sink() itself hasn't handed off ownership to someone else). Since the sink() function as written above doesn't do anything with the body, calling "sink( a );" is a fancy way of writing "a.release();".

The next piece of code shows source() and sink() in action:
``` cpp
    void f() {
        auto_ptr<T> a( source() );
```
  SUMMARY
  |         Before NJ   After NJ
  | Legal?     Yes         Yes
  | Safe?      Yes         Yes
Here f() takes ownership of the pointer received from source(), and (ignoring some problems later in f()) it will delete it automatically when the automatic variable a goes out of scope. This is fine, and it's exactly how passing back an auto_ptr by value is meant to work.
``` cpp
        sink( source() );
```
  SUMMARY
  |         Before NJ   After NJ
  | Legal?     Yes         Yes
  | Safe?      Yes         Yes
Given the trivial (i.e., empty) definitions of source() and sink() here, this is just a fancy way of writing "delete new T(1);". So is it really useful? Well, if you imagine source() as a nontrivial factory function and sink() as a nontrivial consumer, then yes, it makes a lot of sense and crops up regularly in real-world programming.
``` cpp
        sink( auto_ptr<T>( new T(1) ) );
```
  SUMMARY
  |         Before NJ   After NJ
  | Legal?     Yes         Yes
  | Safe?      Yes         Yes
Again, a fancy way of writing "delete new T(1);", and a useful idiom when sink() is a nontrivial consumer function that takes ownership of the pointed-to object.

Things Not To Do, and Why Not To Do Them
"So," you say, "that's cool, and obviously supporting auto_ptr copying is a Good Thing." Well, yes, it is, but it turns out that it can also get you into hot water where you least expect it, and that's why the national body comments objected to leaving auto_ptr in the CD2 form. Here's the fundamental issue, and I'll highlight it to make sure it stands out:

For auto_ptr, copies are NOT equivalent.

It turns out that this has important effects when you try to use auto_ptr with generic code that does make copies and isn't necessarily aware that copies aren't equivalent (after all, usually copies are!). Consider:
``` cpp
        vector< auto_ptr<T> > v;
```
  SUMMARY
  |         Before NJ   After NJ
  | Legal?     Yes          No
  | Safe?       No          No
This is the first indication of trouble, and one of the things the national body comments wanted fixed. In short, even though a compiler wouldn't burp a single warning here, auto_ptrs are NOT safe to put in containers. This is because we have no way of warning the container that copying auto_ptrs has unusual semantics (transferring ownership, changing the right-hand side's state). True, today most implementations I know about will let you get away with this, and code nearly identical to this even appears as a "good example" in the documentation of certain popular compilers. Nevertheless, it was actually unsafe (and is now illegal).

The problem is that auto_ptr does not quite meet the requirements of a type you can put into containers, for copies of auto_ptrs are not equivalent. For one thing, there's nothing that says a vector can't just decide to up and make an "extra" internal copy of some object it contains. Sure, normally you can expect vector not to do this (simply because making extra copies happens to be unnecessary and inefficient, and for competitive reasons a vendor is unlikely to ship a library that's needlessly inefficient), but it's not guaranteed and so you can't rely on it.

But hold on, because it's about to get worse:
``` cpp
        v.push_back( new T(3) );
        v.push_back( new T(4) );
        v.push_back( new T(1) );
        v.push_back( a );
```
(Aside: Note that copying a into the vector means that the 'a' object no longer owns the pointer it's carrying. More on that in a moment.)
``` cpp
        v.push_back( new T(2) );
        sort( v.begin(), v.end() );
```
  SUMMARY
  |         Before NJ   After NJ
  | Legal?     Yes          No
  | Safe?       No          No
Here's the real devil, and another reason why the national body comment was more that just a suggestion (the body in question actually voted No on CD2 largely because of this problem). When you call generic functions that will copy elements, like sort() does, the functions have to be able to assume that copies are going to be equivalent. For example, at least one popular sort internally takes a copy of a "pivot" element, and if you try to make it work on auto_ptrs it will merrily take a copy of the pivot auto_ptr object (thereby taking ownership and putting it in a temporary auto_ptr on the side), do the rest of their work on the sequence (including taking further copies of the now-non-owning auto_ptr that was picked as a pivot value), and when the sort is over the pivot is destroyed and you have a problem: at least one auto_ptr in the sequence (the one that was a copy of the pivot value) no longer owns the pointer it holds, and in fact the pointer it holds has already been deleted!

The problem with the auto_ptr in CD2 is that it gave you no protection \-\- no warning, nothing \-\- against innocently writing code like this. The national body comment required that auto_ptr be refined to either get rid of the unusual copy semantics or else make such dangerous code uncompilable, so that the compiler itself could stop you from doing the dangerous things, like making a vector of auto_ptrs or trying to sort it.

The Scoop on Non-Owning auto_ptrs
``` cpp
        // (after having copied a to another auto_ptr)
        cout << a->Value();
    }
```
  SUMMARY
  |         Before NJ   After NJ
  | Legal?     Yes          No
  | Safe?     (Yes)         No
(We'll assume that a was copied, but that its pointer wasn't deleted by the vector or the sort.) Under CD2 this was fine, since even though a no longer owns the pointer, it would still contain a copy of it; a just wouldn't call delete on its pointer when a itself goes out of scope, that's all, because it would know that it doesn't own the pointer.

Now, however, copying an auto_ptr not only transfers ownership but resets the source auto_ptr to null. This is done specifically to avoid letting anyone do anything through a non-owning auto_ptr. Under the final rules, then, using a non-owning auto_ptr like this is not legal and will result in undefined behaviour (typically a core dump on most systems).

In short:
``` cpp
    void f() {
        auto_ptr<T> pt1( new T );
        auto_ptr<T> pt2( pt1 );
        pt1->Value(); // using a non-owning auto_ptr...
                      //  this used to be legal, but is
                      //  now an error
        pt2->Value(); // ok
    }
```
This brings us to the last common usage of auto_ptr:

Wrapping Pointer Members
``` cpp
    class C {
    public:    /*...*/
    protected: /*...*/
    private:
        auto_ptr<CImpl> pimpl_;
    };
```
[Note: there are important safety details not mentioned in this GotW; see later GotW issues including GotW #62, and the book Exceptional C++.]

  SUMMARY
  |         Before NJ   After NJ
  | Legal?     Yes         Yes
  | Safe?      Yes         Yes
auto_ptrs always were and still are useful for encapsulating pointing member variables. This works very much like our motivating example in the "Background" section at the beginning, except that instead of saving us the trouble of doing cleanup at the end of a function, it now saves us the trouble of doing cleanup in C's destructor.

There is still a caveat, of course... just like if you were using a bald pointer data member instead of an auto_ptr member, you MUST supply your own copy constructor and copy assignment operator for the class (even if you disable them by making them private and undefined), because the default ones will do the wrong thing.

News Flash: The "const auto_ptr" Idiom
Now that we've waded through the deeper stuff, here's a technique you'll find interesting. Among its other benefits, the refinement to auto_ptr also makes copying const auto_ptrs illegal. That is:
``` cpp
    const auto_ptr<T> pt1( new T );
        // making pt1 const guarantees that pt1 can
        // never be copied to another auto_ptr, and
        // so is guaranteed to never lose ownership

    auto_ptr<T> pt2( pt1 ); // illegal
    auto_ptr<T> pt3;
    pt3 = pt1;              // illegal
```
This "const auto_ptr" idiom is one of those things that's likely to become a commonly used technique, and now you can say that you knew about it since the beginning.

I hope you enjoyed this Special Edition of GotW, posted in honour of the voting out of ISO Final Draft International Standard C++ [in November 1997].

 

Notes
1. In the original question, I forgot that there is no conversion from T* to auto_ptr<T> because the constructor is "explicit". The quoted code below is fixed. (That's what I get for dashing this off near midnight on Friday before rushing to New Jersey!)

# 026 Bool
Difficulty: 7 / 10
Do we really need a builtin bool type? Why not just emulate it in the existing language? This GotW shows the answer.

----

Problem
Besides wchar_t (which was a typedef in C), bool is the only builtin type to be added to C++ since the ARM.[1] Could bool's effect have been duplicated without adding a builtin type? If yes, show an equivalent implementation. If no, show why possible implementations do not behave the same as the builtin bool.

----

Solution
Besides wchar_t (which was a typedef in C), bool is the only builtin type to be added to C++ since the ARM.[1] Could bool's effect have been duplicated without adding a builtin type? If yes, show an equivalent implementation.

The answer is: No. The bool builtin type (and the reserved keywords true and false) were added to C++ precisely because they couldn't be duplicated completely using the existing language.

If no, show why possible implementations do not behave the same as the builtin bool.

There are four major implementations:

Option 1: Typedef (score: 8.5 / 10)
This option means to "typedef <something> bool;", typically:
``` cpp
    typedef int bool;
    const bool true  = 1;
    const bool false = 0;
```
This solution isn't bad, but it doesn't allow overloading on bool. For example:
``` cpp
    // file f.h
    void f( int  ); // ok
    void f( bool ); // ok, redeclares the same function

    // file f.cpp
    void f( int  ) { /*...*/ }   // ok
    void f( bool ) { /*...*/ }   // error, redefinition
```
Another problem is that it can allow code like this:
``` cpp
    void f( bool b ) {
        assert( b != true && b != false );
    }
```
So Option 1 isn't good enough.

Option 2: #define (score: 0 / 10)
This option means to "#define bool <something>", typically:
``` cpp
    #define bool  int
    #define true  1
    #define false 0
```

This is, of course, purely evil. It not only has all of the same problems as Option 1 above, but it also wreaks the usual havoc of #defines. For example, pity the poor customer who tries to use this library and already has a variable named 'false'; now this definitely behaves differently from a builtin type.

Trying to use the preprocessor to simulate a type is just a bad idea.

Option 3: Enum (score: 9 / 10)
This option means to make an "enum bool", typically:
``` cpp
    enum bool { false, true };
```
This is somewhat better than Option 1, in my opinion. It allows overloading (the main problem with #1), but doesn't allow automatic conversions from a conditional expression (which would have been possible with #1), to wit:
``` cpp
    bool b;
    b = ( i == j );
```
This doesn't work because ints cannot be implicitly converted to enums.

Option 4: Class (score: 9 / 10)
Heck, this is an object-oriented language, right? So why not write an class, typically:
``` cpp
    class bool {
    public:
        bool();

        bool( int );      // to enable conversions from
        operator=( int ); //  conditional expressions

        //operator int();   // questionable!
        //operator void*(); // questionable!

    private:
        unsigned char b_;
    };

    const bool true ( 1 );
    const bool false( 0 );
```
This works except for the conversion operators marked "questionable". They're questionable because:

1. WITH an automatic conversion, bools will interfere with overload resolution, just as do all classes having non-explicit (conversion) constructors and/or automatic conversions (especially conversions from/to common types).

2. WITHOUT a conversion to something like int or void*, bool objects can't be tested "naturally" in conditions. For example:
``` cpp
    bool b;
    /*...*/
    if( b ) // error without an automatic conversion to
    {       // something like int or void*
        /*...*/
    }
```
It's a classic Catch-22 situation: We must choose one or the other, but neither option lets us duplicate the effect of having a builtin bool type.

Summary
A typedef ... bool wouldn't allow overloading on bool.

A #define bool wouldn't allow overloading either and would wreak the usual havoc of #defines.

An enum bool would allow overloading but couldn't be automatically converted from a conditional expression (as in "b = (i == j);").

A class bool would allow overloading but wouldn't let a bool object be tested in conditions (as in "if( b )") unless it provided an automatic conversion to something like int or void*, which would wreak the usual havoc of automatic conversions.

Yes, we really did need a builtin bool! And, finally, there's one more thing (related to overloading) that we couldn't have done otherwise, either, except perhaps with Option 4: specify that conditional expressions have type bool.

 

Notes
1. M. Ellis M and B. Stroustrup. The Annotated C++ Reference Manual (Addison-Wesley, 1990).

# 027 Forwarding Functions
Difficulty: 3 / 10
What's the best way to write a forwarding function? The basic answer is easy, but we'll also learn about a recent and subtle change to the language.



Problem
Forwarding functions are useful tools for handing off work to another function or object, especially when the handoff is done efficiently.

Critique the following forwarding function. Would you change it? If so, how?
``` cpp
    // file f.cpp

    #include "f.h"
    /*...*/
    bool f( X x ) {
        return g( x );
    }
```
(Warning: One purpose of this GotW is to illustrate the consequences of a subtle language refinement made this July [1997] in London.)



Solution
Forwarding functions are useful tools for handing off work to another function or object, especially when the handoff is done efficiently.

This introduction gets to the heart of the matter: efficiency.

Critique the following forwarding function. Would you change it? If so, how?
``` cpp
    // file f.cpp

    #include "f.h"
    /*...*/
    bool f( X x ) {
        return g( x );
    }
```
There are two main enhancements that would make this function more efficient. The first should always be done, and the second is a matter of judgment.

1. Pass the parameter by const& instead of by value
"Isn't that blindingly obvious?" you might ask. No, it isn't, not in this particular case. Until recently, the language said that, since a compiler can prove that the parameter x will never be used for any other purpose than passing it in turn to g(), the compiler may decide to elide x completely. For example, if the client code looks something like this:
``` cpp
    X my_x;
    f( my_x );
```
then the compiler might either:

a) create a copy of my_x for f()'s use (this is the parameter named x in f()'s scope) and pass that to g(), or

b) pass my_x directly to g() without creating a copy at all since it notices that the extra copy will never be otherwise used except as a parameter to g().

The latter is nicely efficient, isn't it? That's what optimizing compilers are for, aren't they?

Yes, and yes, until the London meeting in July 1997. At that meeting, the draft was amended to place more restrictions on the situations where the compiler is allowed to elide "extra" copies like this.[1] The only places where a compiler may still elide copy constructors is for the return value optimization (see your favourite textbook for details) and for temporary objects.

This means that, for forwarding functions like f, a compiler is now required to perform two copies. Since we (as the authors of f) know in this case that the extra copy isn't necessary, we should fall back on our general rule and declare x as a const X& parameter.

(Note: If we'd been following this general rule all along instead of trying to take advantage of detailed knowledge about what the compiler is allowed to do, the change in the rules wouldn't have affected us. This is a stellar example of why simpler is better \-\- avoid the dusty corners of the language as much as you can, and strive to never rely on cute subtleties.)

2. Inline the function
This one is a matter of judgment. In short, prefer to write all functions out-of-line by default, and then selectively inline individual functions as necessary only after you know that the performance gain from inlining is actually needed.

If you inline the function, the positive side is that you avoid the overhead of the extra function call to f.

The negative side is that inlining f exposes f's implementation and make client code depend on it, so that if f changes all client code must recompile. Worse, client code now also needs at least the prototype for function g(), which is a bit of a shame since client code never actually calls g directly and probably never needed g's prototype before (at least, not as far as we can tell from our example). And if g() itself were changed to take other parameters of still other types, client code would now depend on those classes' declarations, too.

Both inlining or not inlining can be valid choices. It's a judgment call where the benefits and drawbacks depend on what you know about how (and how widely) f is used today, and how (and how often) it's likely to change in the future.

From the GotW coding standards:

- prefer passing parameters of class type by const& rather than passing them by value

- avoid inlining functions until profiling tells you it's justified (programmers are notoriously bad at guessing which parts of their code are performance bottlenecks)

 

Notes
1. This change was necessary to avoid the problems that can come up when compilers are permitted to wantonly elide copy construction, especially when copy construction has side effects. There are cases where reasonable code may rely on the number of copies actually made of an object.

# 028 The Fast Pimpl Idiom
Difficulty: 6 / 10
It's sometimes tempting to cut corners in the name of "reducing dependencies" or in the name of "efficiency," but it may not always be a good idea. Here's an excellent idiom to accomplish both objectives simultaneously and safely.



Problem
Standard malloc() and new() calls are very expensive. In the code below, the programmer originally has a data member of type X in class Y:
``` cpp
    // file y.h
    #include "x.h"

    class Y {
        /*...*/
        X x_;
    };

    // file y.cpp
    Y::Y() {}
```
This declaration of class Y requires the declaration of class X to be visible (from x.h). To avoid this, the programmer first tries to write:
``` cpp
    // file y.h
    class X;

    class Y {
        /*...*/
        X* px_;
    };

    // file y.cpp
    #include "x.h"

    Y::Y() : px_( new X ) {}
    Y::~Y() { delete px_; px_ = 0; }
```
This nicely hides X, but it turns out that Y is used very widely and the dynamic allocation overhead is degrading performance. Finally, our fearless programmer hits on the "perfect" solution that requires neither including x.h in y.h nor the inefficiency of dynamic allocation (and not even a forward declaration!):
``` cpp
    // file y.h
    class Y {
        /*...*/
        static const size_t sizeofx = /*some value*/;
        char x_[sizeofx];
    };

    // file y.cpp
    #include "x.h"

    Y::Y() {
        assert( sizeofx >= sizeof(X) );
        new (&x_[0]) X;
    }

    Y::~Y() {
        (reinterpret_cast<X*>(&x_[0]))->~X();
    }
```
Discuss.



Solution
The Short Answer
Don't do this. Bottom line, C++ doesn't support opaque types directly, and this is a brittle attempt (some people would even say "hack"[1]) to work around that limitation.

What the programmer almost certainly wants is something else, namely the "Fast Pimpl" idiom, which I'll present right after the "Why Attempt #3 is Deplorable" section below.

Why Attempt #3 is Deplorable
First, let's consider a few reasons why Attempt #3 above truly is deplorable form:

1. Alignment. Unlike dynamic memory acquired by ::operator new(), the x_ character buffer isn't guaranteed to be properly aligned for objects of type X. To make this work more reliably, the programmer could use a "max_align" union, which is usually implemented as something like:
``` cpp
    union max_align {
      short       dummy0;
      long        dummy1;
      double      dummy2;
      long double dummy3;
      void*       dummy4;
      /*...and pointers to functions, pointers to
           member functions, pointers to member data,
           pointers to classes, eye of newt, ...*/
    };
```
It would be used like this:
``` cpp
    union {
      max_align m;
      char x_[sizeofx];
    };
```
This isn't guaranteed to be fully portable, but in practice it's close enough because there are few or no systems on which this won't work as expected.

I know that some experts will accept and even recommend this technique. In good conscience I still have to call it a hack and strongly discourage it.

2. Brittleness. The author of Y has to be inordinately careful with otherwise-ordinary Y functions. For example, Y must not use the default assignment operator, but must either suppress assignment or supply its own.

Writing a safe Y::operator=() isn't hard, but I'll leave it as an exercise for the reader. Remember to account for exception safety in that and in Y::~Y(). Once you're finished, I think you'll agree that this is a lot more trouble than it's worth.

3. Maintenance Cost. When sizeof(X) grows beyond sizeofx, the programmer must bump up sizeofx. This can be an unattractive maintenance burden. Choosing a larger value for sizeofx mitigates this, but at the expense of trading off efficiency (see #4).

4. Inefficiency. Whenever sizeofx > sizeof(X), space is being wasted. This can be minimized, but at the expense of maintenance effort (see #3).

5. Just Plain Wrongheadedness. I include this one last, but not least: In short, it's obvious that the programmer is trying to do "something unusual." Frankly, in my experience, "unusual" is just about always a synonym for "hack." Whenever you see this kind of subversion \-\- whether it's allocating objects inside character arrays like this programmer is doing, or implementing assignment using explicit destruction and placement new as discussed in GotW #23 \-\- you should Just Say No.

I really mean that. Make it a reflex.

A Better Solution: The "Fast Pimpl"
The motivation for hiding X is to avoid forcing clients of Y to know about (and therefore depend upon) X. The usual C++ workaround for eliminating this kind of implementation dependency is to use the pimpl idiom,[2] which is what our fearless programmer first tried to do.

The only issue in this case was the pimpl method's performance because of allocating space for X objects in the free store. The usual way to address allocation performance for a specific class is to simply overload operator new for that specific class, because fixed-size allocators can be made much more efficient than general-purpose allocators.

Unfortunately, this assumes that the author of Y is also the author of X. That's not likely to be true in general. The real solution is to use an Efficient Pimpl, which is a true pimpl class with its own class-specific operator new:
``` cpp
    // file y.h
    class YImpl;

    class Y {
      /*...*/
      YImpl* pimpl_;
    };

    // file y.cpp
    #include "x.h"

    struct YImpl {  // yes, 'struct' is allowed :-)
      /*...private stuff here...*/

      void* operator new( size_t )   { /*...*/ }
      void  operator delete( void* ) { /*...*/ }
    };

    Y::Y() : pimpl_( new YImpl ) {}
    Y::~Y() { delete pimpl_; pimpl_ = 0; }
```
"Aha!" you say. "We've found the holy grail \-\- the Fast Pimpl!" you say. Well, yes, but hold on a minute and think about how this will work and what it will cost you.

Your favourite advanced C++ or general-purpose programming textbook has the details about how to write efficient fixed-size [de]allocation functions, so I won't cover that again here. I will talk about usability: One technique is to put them in a generic fixed-size allocator template, perhaps something like:
``` cpp
    template<size_t S>
    class FixedAllocator {
    public:
      void* Allocate( /*requested size is always S*/ );
      void  Deallocate( void* );
    private:
      /*...implemented using statics?...*/
    };
```
Because the private details are likely to use statics, however, there could be problems if Deallocate() is ever called from a static object's dtor. Probably safer is a singleton that manages a separate free list for each request size (or, as an efficiency tradeoff, a separate free list for each request size "bucket"; e.g., one list for blocks of size 0-8, another for blocks of size 9-16, etc.):
``` cpp
    class FixedAllocator {
    public:
      static FixedAllocator* Instance();

      void* Allocate( size_t );
      void  Deallocate( void* );
    private:
        /*...singleton implementation, typically
             with easier-to-manage statics than
             the templated alternative above...*/
    };
```
Let's throw in a helper base class to encapsulate the calls:
``` cpp
    struct FastPimpl {
      void* operator new( size_t s ) {
        return FixedAllocator::Instance()->Allocate(s);
      }
      void operator delete( void* p ) {
        FixedAllocator::Instance()->Deallocate(p);
      }
    };
```
Now, you can easily write as many Fast Pimpls as you like:
``` cpp
    //  Want this one to be a Fast Pimpl?
    //  Easy, then just inherit...
    struct YImpl : FastPimpl {
      /*...private stuff here...*/
    };
```
But Beware!
This is nice and all, but don't just use the Fast Pimpl willy-nilly. You're getting extra allocation speed, but as usual you should never forget the cost: Managing separate free lists for objects of specific sizes usually means incurring a space efficiency penalty because any free space is fragmented (more than usual) across several lists.

As with any other optimization, use this one only after profiling and experience prove that the extra performance boost is really needed in your situation.

 

Notes
1. I'm one of them. :-)

2. See GotW #24.






# 029 Strings
Difficulty: 7 / 10
So you want a case-insensitive string class? Your mission, should you choose to accept it, is to write one.



Problem
Write a ci_string class which is identical to the standard 'string' class, but is case-insensitive in the same way as the (nonstandard but widely implemented) C function stricmp():
``` cpp
    ci_string s( "AbCdE" );

    // case insensitive
    assert( s == "abcde" );
    assert( s == "ABCDE" );

    // still case-preserving, of course
    assert( strcmp( s.c_str(), "AbCdE" ) == 0 );
    assert( strcmp( s.c_str(), "abcde" ) != 0 );
```

Solution
Write a ci_string class which is identical to the standard 'string' class, but is case-insensitive in the same way as the (nonstandard but widely implemented) C function stricmp():

The "how can I make a case-insensitive string?" question is so common that it probably deserves its own FAQ \-\- hence this issue of GotW.

Note 1: The stricmp() case-insensitive string comparison function is not part of the C standard, but it is a common extension on many C compilers.

Note 2: What "case insensitive" actually means depends entirely on your application and language. For example, many languages do not have "cases" at all; for those that do, you still have to decide whether you want accented characters to compare equal to unaccented characters, and so on. This GotW provides guidance on how to implement case-insensitivity for standard strings in whatever sense applies to your particular situation.

Here's what we want to achieve:
``` cpp
    ci_string s( "AbCdE" );

    // case insensitive
    assert( s == "abcde" );
    assert( s == "ABCDE" );

    // still case-preserving, of course
    assert( strcmp( s.c_str(), "AbCdE" ) == 0 );
    assert( strcmp( s.c_str(), "abcde" ) != 0 );
```
The key here is to understand what a "string" actually is in standard C++. If you look in your trusty string header, you'll see something like this:
``` cpp
  typedef basic_string<char> string;
```
So string isn't really a class... it's a typedef of a template. In turn, the basic_string<> template is declared as follows, in all its glory:
``` cpp
  template<class charT,
           class traits = char_traits<charT>,
           class Allocator = allocator<charT> >
      class basic_string;
```
So "string" really means "basic_string<char, char_traits<char>, allocator<char> >". We don't need to worry about the allocator part, but the key here is the char_traits part because char_traits defines how characters interact and compare(!).

basic_string supplies useful comparison functions that let you compare whether a string is equal to another, less than another, and so on. These string comparison functions are built on top of character comparison functions supplied in the char_traits template. In particular, the char_traits template supplies character comparison functions named eq(), ne(), and lt() for equality, inequality, and less-than comparisons, and compare() and find() functions to compare and search sequences of characters.

If we want these to behave differently, all we have to do is provide a different char_traits template! Here's the easiest way:
``` cpp
  struct ci_char_traits : public char_traits<char>
                // just inherit all the other functions
                //  that we don't need to override
  {
    static bool eq( char c1, char c2 )
      { return toupper(c1) == toupper(c2); }

    static bool ne( char c1, char c2 )
      { return toupper(c1) != toupper(c2); }

    static bool lt( char c1, char c2 )
      { return toupper(c1) <  toupper(c2); }

    static int compare( const char* s1,
                        const char* s2,
                        size_t n ) {
      return memicmp( s1, s2, n );
             // if available on your compiler,
             //  otherwise you can roll your own
    }

    static const char*
    find( const char* s, int n, char a ) {
      while( n-- > 0 && toupper(*s) != toupper(a) ) {
          ++s;
      }
      return s;
    }
  };
```
And finally, the key that brings it all together:

  typedef basic_string<char, ci_char_traits> ci_string;
All we've done is created a typedef named "ci_string" which operates exactly like the standard "string", except that it uses ci_char_traits instead of char_traits<char> to get its character comparison rules. Since we've handily made the ci_char_traits rules case-insensitive, we've made ci_string itself case-insensitive without any further surgery \-\- that is, we have a case-insensitive string without having touched basic_string at all!

This GotW should give you a flavour for how the basic_string template works and how flexible it is in practice. If you want different comparisons than the ones memicmp() and toupper() give you, just replace the five functions shown above with your own code that performs character comparisons the way that's appropriate in your particular application.

Exercises for the Reader
1. Is it safe to inherit ci_char_traits from char_traits<char> this way? Why or why not?

2. Why does the following code fail to compile?
``` cpp
    ci_string s = "abc";
    cout << s << endl;
```
HINT 1: See GotW #19.

HINT 2: From 21.3.7.9 [lib.string.io], the declaration of operator<< for basic_string is specified as:
``` cpp
    template<class charT, class traits, class Allocator>
    basic_ostream<charT, traits>&
    operator<<(basic_ostream<charT, traits>& os,
               const basic_string<charT,traits,Allocator>& str);
```
ANSWER: Notice first that cout is actually a basic_ostream<char, char_traits<char> >. Then we see the problem: operator<< for basic_string is templated and all, but it's only specified for insertion into a basic_ostream with the same 'char type' and 'traits type' as the string. That is, the standard operator<< will let you output a ci_string to a basic_ostream<char, ci_char_traits>, which isn't what cout is even though ci_char_traits inherits from char_traits<char> in the above solution.

There are two ways to resolve this: define insertion and extraction for ci_strings yourself, or tack on ".c_str()":
``` cpp
        cout << s.c_str() << endl;
```
3. What about using other operators (e.g., +, +=, =) and mixing strings and ci_strings as arguments? For example:
``` cpp
        string    a = "aaa";
        ci_string b = "bbb";
        string    c = a + b;
```
ANSWER: Again, define your own operators, or tack on ".c_str()":
``` cpp
        string    c = a + b.c_str();
```




# 030 Name Lookup
Difficulty: 9.5 / 10
When you call a function, which function do you call? The answer is determined by name lookup, but you're almost certain to find some of the details surprising.

----

Problem
In the following code, which functions are called? Why? Analyze the implications.
``` cpp
namespace A {
  struct X;
  struct Y;
  void f( int );
  void g( X );
}

namespace B {
    void f( int i ) {
        f( i );   // which f()?
    }
    void g( A::X x ) {
        g( x );   // which g()?
    }
    void h( A::Y y ) {
        h( y );   // which h()?
    }
}
```

Solution
In the following code, which functions are called? Why?

Two of the three cases are (fairly) obvious, but the third requires a good knowledge of C++'s name lookup rules \-\- in particular, Koenig lookup.
``` cpp
namespace A {
    struct X;
    struct Y;
    void f( int );
    void g( X );
}

namespace B {
    void f( int i ) {
        f( i );   // which f()?
    }
```
This f() calls itself, with infinite recursion.

There is another function with signature f(int), namely the one in namespace A. If B had written "using namespace A;" or "using A::f;", then A::f(int) would have been visible as a candidate when looking up f(int), and the "f(i);" call would have been ambiguous between A::f(int) and B::f(int). Since B did not bring A::f(int) into scope, however, only B::f(int) is considered and so the call is unambiguous.
``` cpp
    void g( A::X x ) {
        g( x );   // which g()?
    }
```
This call is ambiguous between A::g(X) and B::g(X). The programmer must explicitly qualify the call with the appropriate namespace name to get the g() he wants.

You may at first wonder why this should be so. After all, as with f(), B hasn't written "using" anywhere to bring A::g(X) into its scope, and so you'd think that only B::g(X) would be visible, right? Well, this would be true but for an extra rule that C++ uses when looking up names:

    **Koenig Lookup (simplified)**
    If you supply a function argument of class type (here x, of type A::X), then to find the function name the compiler considers matching names in the namespace (here A) containing the argument's type.

There's a little more to it, but that's essentially it. Here's an example, right out of the standard:
``` cpp
namespace NS {
    class T { };
    void f(T);
}

NS::T parm;
int main() {
    f(parm);    // OK, calls NS::f
}
```
I won't delve here into the reasons why Koenig lookup is a Good Thing (if you want to stretch your imagination, replace "NS" with "std", "T" with "string", and "f" with "operator<<" and consider the ramifications). See the "further reading" at the end for much more on Koenig lookup and its implications for namespace isolation and analyzing class dependencies. Suffice it to say that Koenig lookup is indeed a Good Thing, and that you should be aware of how it works because it can sometimes affect the code you write.
``` cpp
    void h( A::Y y ) {
        h( y );   // which h()?
    }
}
```
There is no other h( A::Y ), and so this h() calls itself with infinite recursion.

Although B::h()'s signature mentions a type found in namespace A, this doesn't affect name lookup because there are no functions in A matching the name h(A::Y).

Analyze the implications.

In short, the meaning of code in namespace B is being affected by a function declared in the completely separate namespace A, even though B has done nothing but simply mention a type found in A and there's nary a "using" in sight!

What this means is that namespaces aren't quite as independent as people originally thought. Don't start decrying namespaces just yet, though; namespaces are still pretty independent and they fit their intended uses to a T. The purpose of this GotW is just to point out one of the (rare) cases where namespaces aren't quite hermetically sealed... and, in fact, where they should _not_ be hermetically sealed, as the "further reading" shows.

**Further Reading**
The implications of namespaces and Koenig lookup go much further than I have covered here. For more information on why Koenig lookup is not a "hole" in namespaces, and how it affects the way we analyze class dependencies, see my column in the March 1998 issue of C++ Report.



# 031 (Im)pure Virtual Functions
**Difficulty: 7 / 10**
Does it ever make sense to make a function pure virtual, but still provide a body?

----

**Problem**
**JG Question**
**1.** What is a pure virtual function? Give an example.

**Guru Question**
**2.** Why might you declare a pure virtual function and also write a definition (body)? Give as many reasons or situations as you can.

----

**Solution**
**1.** What is a pure virtual function? Give an example.

A pure virtual function is a virtual function that you want to force derived classes to override. If a class has any unoverridden pure virtuals, it is an "abstract class" and you can't create objects of that type.

``` cpp
class AbstractClass {
public:
    // declare a pure virtual function:
    // this class is now abstract
    virtual void f(int) = 0;
};

class StillAbstract : public AbstractClass {
    // does not override f(int),
    // so this class is still abstract
};

class Concrete : public StillAbstract {
public:
    // finally overrides f(int),
    // so this class is concrete
    void f(int) { /*...*/ }
};

AbstractClass a;    // error, abstract class
StillAbstract b;    // error, abstract class
Concrete      c;    // ok, concrete class
```

**2.** Why might you declare a pure virtual function and also write a definition (body)? Give as many reasons or situations as you can.

There are three main reasons you might do this. #1 is commonplace, #2 is pretty rare, and #3 is a workaround used occasionally by advanced programmers working with weaker compilers.

Most programmers should only ever use #1.

*1. Pure Virtual Destructor*
All base classes should have a virtual destructor (see your favourite C++ book for the reasons). If the class should be abstract (you want to prevent instantiating it) but it doesn't happen to have any other pure virtual functions, a common technique to make the destructor pure virtual:

``` cpp
// file b.h
class B {
public: /*...other stuff...*/
    virtual ~B() = 0; // pure virtual dtor
};
```

Of course, any derived class' destructor must call the base class' destructor, and so the destructor must still be defined (even if it's empty):

``` cpp
// file b.cpp
B::~B() { /* possibly empty */ }
```

If this definition were not supplied, you could still derive classes from B but they could never be instantiated, which isn't particularly useful.

*2. Force Conscious Acceptance of Default Behaviour*
If a derived class doesn't choose to override a normal virtual, it just inherits the base version's behaviour by default. If you want to provide a default behaviour but not let derived classes just inherit it "silently" like this, you can make it pure virtual but still provide a default that the derived class author has to call deliberately if he wants it:

``` cpp
class B {
public:
    virtual bool f() = 0;
};

bool B::f() {
    return true;  // this is a good default, but
}                 // shouldn't be used blindly

class D : public B {
public:
    bool f() {
        return B::f(); // if D wants the default
    }             // behaviour, it has to say so
};
```

*3. Workaround Poor Compiler Diagnostics*
There are situations where you could accidentally end up calling a pure virtual function (indirectly from a base constructor or destructor; see your favourite advanced C++ book for examples). Of course, well-written code normally won't get into these problems, but no one's perfect and once in a while it happens.

Unfortunately, not all compilers[1] will actually tell you when this is the problem. Those that don't can give you spurious unrelated errors that take forever to track down. "Argh," you scream, when you do finally diagnose the problem yourself some hours later, "why didn't the compiler just tell me that's what I did?!"

One way to protect yourself against this wasted debugging time is to provide definitions for the pure virtual functions that should never be called, and put some really evil code into those definitions that lets you know right away if you do call them accidentally. For example:

``` cpp
class B {
public:
    virtual bool f() = 0;
};
};

bool B::f() {   // this should NEVER be called
    if( PromptUser( "pure virtual B::f called -- "
                    "abort or ignore?" ) == Abort )
        DieDieDie();
}
```

In the common DieDieDie() function, do whatever is necessary on your system to get into the debugger or dump a stack trace or otherwise get diagnostic information. Here are some common methods that will get you into the debugger on most systems. Pick the one that you like best.

``` cpp
void DieDieDie() {  // scribble through null ptr
    memset( 0, 1, 1 );
}

void DieDieDie() {  // another C-style method
    assert( false );
}

void DieDieDie() {  // back to last "catch(...)"
    class LocalClass {};
    throw LocalClass();
}

void DieDieDie() {  // for standardphiles
    throw logic_error();
}
```

You get the idea. Be creative. :-)

 

**Notes**
1. Well, technically it's the runtime environment that catches this sort of thing. I'll say "compiler" anyway because it's generally the compiler that ought to slip in the code that checks this for you at runtime.



# 032 Preprocessor Macros
**Difficulty: 4 / 10**
With all the type-safe features in C++, why would you ever use #define?

----

**Problem**
**1.** With flexible features like overloading and type-safe templates available, why would a C++ programmer ever write "#define"?

----

**Solution**
**1.** With flexible features like overloading and type-safe templates available, why would a C++ programmer ever write "#define"?

C++ features often, but not always, cancel out the need for using #define. For example, "const int c = 42;" is superior to "#define c 42" because it provides type safety, avoids accidental preprocessor edits, and other reasons. There are, however, still a few good reasons to write #define:

*1. Header Guards*
This is the usual trick to prevent multiple header inclusions:

``` cpp
#ifndef MYPROG_X_H
#define MYPROG_X_H

// ... the rest of the header file x.h goes here...

#endif
```

*2. Accessing Preprocessor Features*
It's often nice to insert things like line numbers and build times in diagnostic code. An easy way to do this is to use predefined macros like __FILE__, __LINE__, __DATE__ and __TIME__. For the same and other reasons, it's often useful to use the stringizing and token-pasting preprocessor operators (# and ##).

*3. Selecting Code at Compile Time (or, Build-Specific Code)*
This is the richest, and most important, category of uses for the preprocessor. Although I am anything but a fan of preprocessor magic, there are things you just can't do as well, or at all, in any other way.

*a) Debug Code*

Sometimes you want to build your system with certain "extra" pieces of code (typically debugging information) and sometimes you don't:

``` cpp
void f()
{
#ifdef MY_DEBUG
    cerr << "some trace logging" << endl;
#endif

    // ... the rest of f() goes here...
}
```

It is possible to do this at run time, of course. By making the decision at compile time, you avoid both the overhead and the flexibility of deferring the decision until run time.

*b) Platform-Specific Code*

Usually it's best to deal with platform-specific code in a factory for better code organization and runtime flexibility. Sometimes, however, there are just too few differences to justify a factory, and the preprocessor can be a useful way to switch optional code.

*c) Variant Data Representations*

A common example is that a module may define a list of error codes, which outside users should see as a simple enum with comments, but which inside the module should be stored in a map for easy lookup. That is:

``` cpp
// For outsiders:
enum Errors {
  ERR_OK = 0,           // No error
  ERR_INVALID_PARAM = 1 // <description>
  ...
}

// For the module's internal use:
map<Error,const char*> lookup;
lookup.insert( make_pair( Error(0), "No error" ) );
lookup.insert( make_pair( Error(1), "<description>" ) );
...
```

We'd like to have both representations without defining the actual information (code/msg pairs) twice. With macro magic, we can simply write a list of errors as follows, creating the appropriate structure at compile time:

``` cpp
DERR_ENTRY( ERR_OK,            0, "No error" ),
DERR_ENTRY( ERR_INVALID_PARAM, 1, "<description>" ),
//...
```

The implementations of DERR_ENTRY and related macros is left to the reader.

These are three common examples; there are many more.


# 033 Inline
**Difficulty: 4 / 10**
Contrary to popular opinion, the keyword inline is not some sort of magic bullet. It is, however, a useful tool when employed properly. The question is, When should you use it?

----

**Problem**
**1.** Does inlining a function increase efficiency?

**2.** When and how should you decide to inline a function?

----

**Solution**
**Write What You Know, and Know What You Write**
**1.** Does inlining a function increase efficiency?

Not necessarily.

First off, if you tried to answer this question without first asking what you want to optimize, you fell into a classic trap. The first question has to be: "What do you mean by efficiency?" Does the above question mean program size? memory footprint? execution time? development speed? build time? or something else?

Second, contrary to popular opinion, inlining can improve OR worsen any of these:

    a) Program Size. Many programmers assume that inlining increases program size, because instead of having one copy of a function's code the compiler creates a copy in every place that function is used. This is often true, but not always: If the function size is smaller than the code the compiler has to generate to perform the function call, then inlining will reduce program size.
    
    b) Memory Footprint. Inlining usually has little or no effect on a program's memory usage, apart from the basic program size (above).
    
    c) Execution Time. Many programmers assume that inlining a function will improve execution time, because it avoids the function call overhead, and because "seeing through the veil" of the function call gives the compiler's optimizer more opportunities to work its craft. This can be true, but often isn't: If the function is not called extremely frequently, there will usually be no visible improvement in overall program execution time. In fact, just the opposite can happen: If inlining increases a calling function's size it will reduce that caller's locality of reference, which means that overall program speed can actually worsen if the caller's inner loop no longer fits in the processor's cache.
    
    To put this point in perspective, don't forget that most programs are not CPU-bound. Probably the most common bottleneck is to be I/O-bound, which can include anything from network bandwidth or latency to file or database access.
    
    d) Development Speed, Build Time. To be most useful, inlined code has to be visible to the caller, which means that the caller has to depend on the internals of the inlined code. Depending on another module's internal implementation details increases the practical coupling of modules (it does not, however, increase their theoretical coupling, because the caller doesn't actually use any of the callee's internals). Usually when functions change, callers do not need to be recompiled (only relinked, and often not even that). When inlined functions change, callers are forced to recompile.

Finally, if you're looking to improve efficiency in some way, always look to your algorithms and data structures first... they will give you the order-of-magnitude overall improvements, whereas process optimizations like inlining generally (note, "generally") yield less dramatic results.

**Just Say 'No For Now'**
**2.** When and how should you decide to inline a function?

Just like any other optimization: After a profiler tells you to, and not a minute sooner. The only time you'd inline right away is when it's an empty function that's likely to stay empty, or you're absolutely forced to, i.e., when writing a non-exported template.

Bottom line, inlining always costs something, if only increased coupling, and you should never pay for something until you know you're going to turn a profit \-\- that is, get something better in return.

"But I can always tell where the bottlenecks are," you may think? Don't worry, you're not alone, most programmers think this at one time or another, but you're still wrong. Dead wrong. Programmers are notoriously poor guessers about where their code's true bottlenecks lie.

Usually only experimental evidence (a.k.a. profiling output) helps to tell you where the true hot spots are. Nine times out of ten, a programmer cannot identify the number-one hot-spot bottleneck in his code without some sort of profiling tool. After more than a decade in this business, I have yet to see a consistent exception in any programmer I've ever worked with or heard about... even though everyone and their kid brother may claim until they're blue in the face that this doesn't apply to them. :-)

[Note another practical reason for this: Profilers aren't as good at identifying which inlined functions should NOT be inlined.]

**What About Computation-Intensive Tasks (e.g., Numeric Libraries)?**
Some people write small, tight library code, such as advanced scientific and engineering numerical libraries, and can sometimes do reasonably well with seat-of-the-pants inlining. Even those developers, however, tend to inline judiciously and tend to tune later rather than earlier. Note that writing a module and then comparing performance with "inlining on" and "inlining off" is generally an unsound idea, because "all on" or "all off" is a coarse measure that tells you only about the average case... it doesn't tell you WHICH functions benefited (and how much each one did). Even in these cases, usually you're better off to use a profiler, and optimize based on its advice.

**What About Accessors?**
There are people who will argue that one-line accessor functions (like "X& Y::f() { return myX_; }") are a reasonable exception, and could/should be automatically inlined. I understand the reasoning, but be careful: At the very least, all inlined code increases coupling, so unless you're certain in advance that inlining will help there's no harm deferring that decision to profiling time. Later, when the code is stable, a profiler might point out that inlining will help, and at that point: a) you know that what you're doing is worth doing; and b) you'll have avoided all the coupling and possible build overhead until the end of the project development cycle. Not a bad deal, that.

**In Summary**
From the GotW coding standards:

    - avoid inlining or detailed tuning until performance profiles prove the need (Cline95: 139-140, 348-351; Meyers92: 107-110; Murray93: 234-235; 242-244)
    
    - corollary: in general, avoid inlining (Lakos96: 631-632; Murray93: 242-244)
    
    [Note that this deliberately says "avoid" (not "never") and that these same coding standards do encourage prior inlining in one or two very restricted situations.]
    
    Cline95: Marshall Cline and Greg Lomow. "C++ FAQs" Addison-Wesley, 1995
    
    Lakos96: John Lakos. "Large-Scale C++ Software Design" Addison-Wesley, 1996
    
    Meyers92: Scott Meyers. "Effective C++" Addison-Wesley, 1992
    
    Murray93: Robert Murray. "C++ Strategies and Tactics" Addison-Wesley, 1993



# 034 Forward Declarations
**Difficulty: 8 / 10**
Forward declarations are a great way to eliminate needless compile-time dependencies. But here's an example of a forward-declaration snare... how would you avoid it?

----

**Problem**
**JG Question**
**1.** Forward declarations are very useful tools. In this case, they don't work as the programmer expected. Why are the marked lines errors?

``` cpp
// file f.h
#ifndef XXX_F_H_
#define XXX_F_H_

class ostream;  // error
class string;   // error

string f( const ostream& );

#endif
```

**Guru Question**
**2.** Without including any other files, can you write the correct forward declarations for ostream and string above?

----

**Solution**
**1.** Forward declarations are very useful tools. In this case, they don't work as the programmer expected. Why are the marked lines errors?

``` cpp
// file f.h
#ifndef XXX_F_H_
#define XXX_F_H_

class ostream;  // error
class string;   // error

string f( const ostream& );

#endif
```

Alas, you cannot forward-declare ostream and string this way because they are not classes... both are typedefs of templates.

(True, you used to be able to forward-declare ostream and string this way, but that was many years ago and is no longer possible in Standard C++.)

**2.** Without including any other files, can you write the correct forward declarations for ostream and string above?

Unfortunately, the answer is that there is no standard and portable way to do this. The standard says:

    It is undefined for a C++ program to add declarations or definitions to namespace std or namespaces within namespace std unless otherwise specified.

Among other things, this allows vendors to provide implementations of the standard library that have more template parameters for library templates than the standard requires (suitably defaulted, of course, to remain compatible).

The best you can do (which is not a solution to the problem "without including any other files") is this:

``` cpp
#include <iosfwd>
#include <string>
```

The iosfwd header does contain bona fide forward declarations. The string header does not. This is all that you can do and still be portable. Fortunately, forward-declaring string and ostream isn't too much of an issue in practice since they're generally small and widely-used. The same is true for most standard headers. However, beware the pitfalls, and don't be tempted to start forward-declaring templates \-\- or anything else \-\- that belongs to namespace std... that's reserved for the compiler and library writers, and them alone.



# 035 Typename
**Difficulty: 9.5 / 10**
"What's in a (type) name?" Here's an exercise that demonstrates why and how to use typename, using an idiom that's common in the standard library.

----

**Problem**
**Guru Question**
**1.** What, if anything, is wrong with the code below?

``` cpp
template<class T>
struct X_base {
    typedef T instantiated_type;
};

template<class A, class B>
struct X : public X_base<B> {
    bool operator()( const instantiated_type& i ) {
        return ( i != instantiated_type() );
    }
    // ... more stuff ...
};
```

----

**Solution**
**1.** What, if anything, is wrong with the code below?

This example illustrates the issue of why and how to use "typename" to refer to dependent names, and may shed some light on the question: "What's in a name?"

``` cpp
template<class T>
struct X_base {
    typedef T instantiated_type;
};

template<class A, class B>
struct X : public X_base<B> {
    bool operator()( const instantiated_type& i ) {
      return ( i != instantiated_type() );
    }
    // ... more stuff ...
};
```

**1.** Use "typename" for Dependent Names

The problem with X is that "instantiated_type" is meant to refer to the typedef supposedly inherited from the base class X_base<B>. Unfortunately, at the time that the compiler has to parse the inlined definition of X<A,B>::operator()(), dependent names (i.e., names that depend on the template parameters, such as the inherited X_Base::instantiated_type) are not visible, and so the compiler will complain that it doesn't know what "instantiated_type" is supposed to mean. Dependent names only become visible later, at the point where the template is actually instantiated.

If you're wondering why the compiler couldn't just figure it out anyway, pretend that you're a compiler and ask yourself how you would figure out what "instantiated_type" means here. Bottom line, you can't figure it out because you don't know what B is yet, and whether later on there might not be a specialization for X_base<B> that makes X_base<B>::instantiated_type something unexpected \-\- any type name, or even a member variable. In the unspecialized X_base template above, X_base<T>::instantiated_type will always be T, but there's nothing preventing someone from changing that when specializing, for example:

``` cpp
template<>
struct X_base<int> {
    typedef Y instantiated_type;
};
```

Granted, the typedef's name would be a little misleading if they did that, but it's legal. Or even:

``` cpp
template<>
struct X_base<double> {
    double instantiated_type;
};
```

Now the name is less misleading, but template X cannot work with X_base<double> as a base class because instantiated_type is a member variable, not a type name.

Bottom line, the compiler won't know how to parse the definition of X<A,B>::operator()() unless we tell it what instantiated_type is... at minimum, whether it's a type or something else. Here, we want it to be a type.

The way to tell the compiler that something like this is supposed to be a type name is to throw in the keyword "typename". There are two ways we could go about it here. The less elegant is to simply write typename wherever we refer to instantiated_type:

``` cpp
    template<class A, class B>
    struct X : public X_base<B> {
      bool operator()
        ( const typename X_base<B>::instantiated_type& i )
      {
        return
          ( i != typename X_base<B>::instantiated_type() );
      }
      // ... more stuff ...
    };
```

I hope you winced when you read that. As usual, typedefs make this sort of thing much more readable, and by providing another typedef the rest of the definition works as originally written:

``` cpp
    template<class A, class B>
    struct X : public X_base<B> {
      typedef typename X_base<B>::instantiated_type
              instantiated_type;
      bool operator()( const instantiated_type& i ) {
        return ( i != instantiated_type() );
      }
      // ... more stuff ...
    };
```

Before reading on, does anything about adding this typedef seem unusual to you?

**2.** The Secondary (and Subtle) Point

I could have used simpler examples to illustrate this (several appear in the standard in section 14.6.2), but that wouldn't have pointed out the unusual thing: The whole reason the empty base X_base appears to exist is to provide the typedef. However, derived classes usually end up just typedef'ing it again anyway.

Doesn't that seem redundant? It is, but only a little... after all, it's still the specialization of X_base<> that's responsible for determining what the appropriate type should be, and that type can change for different specializations.

The standard library contains base classes like this: "bags-o-typedefs" that are intended to be used in just this way. Hopefully this issue of GotW will help avert some of the questions about why derived classes re-typedef those typedefs, seemingly redundantly, and show that this effect is not really a language design glitch as much as it is just another facet of the age-old question:

"What's in a name?"

**Code Joke**
As a bonus, here's a little code joke:

``` cpp
#include <iostream>
using std::cout;
using std::endl;

struct Rose {};

struct A { typedef Rose rose; };

template<class T>
struct B : T { typedef typename T::rose foo; };

template<class T>
void smell( T ) { cout << "awful" << endl; }

void smell( Rose ) { cout << "sweet" << endl; }

int main() {
    smell( A::rose() );
    smell( B<A>::foo() );
}
```

:-)


# 036 Initialization
**Difficulty: 3 / 10**
What's the difference between direct initialization and copy initialization, and when are they used?

----

**Problem**
**JG Question**
**1.** What is the difference between "direct initialization" and "copy initialization"?

(Hint: See an earlier GotW.)

**Guru Question**
**2.** Which of the following cases use direct initialization, and which use copy initialization?

``` cpp
struct T : S {
    T() : S(1),             // base initialization
          x(2) {}           // member initialization
    X x;
};

T f( T t ) {              // passing a function argument
    return t;               // returning a value
}

S s;
T t;
S& r = t;
reinterpret_cast<S>(t);   // performing a reinterpret_cast
static_cast<S>(t);        // performing a static_cast
dynamic_cast<T&>(r);      // performing a dynamic_cast
const_cast<const T&>(t);  // performing a const_cast

try {
    throw T();              // throwing an exception
} 
catch( T t ) {          // handling an exception
}

f( T(s) );                // functional-notation type conversion
S a[3] = { 1, 2, 3 };     // brace-enclosed initializers
S* p = new S(4);          // new expression
```

**Solution**
**1.** What is the difference between "direct initialization" and "copy initialization"?

(Hint: See an earlier GotW.)

Direct initialization means the object is initialized using a single (possibly conversion) constructor, and is equivalent to the form "T t(u);":
``` cpp
U u;
T t1(u); // calls T::T( U& ) or similar
```
Copy initialization means the object is initialized using the copy constructor, after first calling a user-defined conversion if necessary, and is equivalent to the form "T t = u;":
``` cpp
T t2 = t1;  // same type: calls T::T( T& ) or similar
T t3 = u;   // different type: calls T::T( T(u) )
            //  or T::T( u.operator T() ) or similar
```
[Aside: The reason for the "or similar"s above is that the copy and conversion constructors could take something slightly different from a plain reference (the reference could be const or volatile or both), and the user-defined conversion constructor or operator could additionally take or return an object rather than a reference.]

NOTE: In the last case ("T t3 = u;") the compiler could call both the user-defined conversion (to create a temporary object) and the T copy constructor (to construct t3 from the temporary), or it could choose to elide the temporary and construct t3 directly from u (which would end up being equivalent to "T t3(u);"). Since July 1997 and in the final draft standard, the compiler's latitude to elide temporary objects has been restricted, but it is still allowed for this optimization and for the return value optimization. For more details, see GotW #1 (about the basics) and GotW #27 (about the 1997 change).

**Guru Question**
**2.** Which of the following cases use direct initialization, and which use copy initialization?

Section 8.5 [dcl.init] covers most of these. There were also three tricks that actually don't involve initialization at all... did you catch them?

``` cpp
struct T : S {
    T() : S(1),             // base initialization
          x(2) {}           // member initialization
    X x;
};
```

Base and member initialization both use direct initialization.

``` cpp
T f( T t ) {              // passing a function argument
    return t;               // returning a value
}
```

Passing and returning values both use copy initialization.
​
``` cpp
S s;
T t;
S& r = t;
reinterpret_cast<S>(t);   // performing a reinterpret_cast
```

Trick: A reinterpret_cast does no initialization of a new object at all, but just causes t's bits to be reinterpreted (in-place) as an S.
​
``` cpp
static_cast<S>(t);        // performing a static_cast
```

A static_cast uses direct initialization.

``` cpp
dynamic_cast<T&>(r);      // performing a dynamic_cast
const_cast<const T&>(t);  // performing a const_cast
```

More tricks: No initialization of a new object is involved in either of these cases.

``` cpp
try {
    throw T();              // throwing an exception
} 
catch( T t ) {          // handling an exception
}
```

Throwing and catching an exception object both use copy initialization.

Note that in this particular code there are two copies, for a total of three T objects: A copy of the thrown object is made at the throw site, and in this case a second copy is made because the handler catches the thrown object by value.

``` cpp
f( T(s) );                // functional-notation type conversion
```

"Constructor syntax" type conversion uses direct initialization.

``` cpp
S a[3] = { 1, 2, 3 };     // brace-enclosed initializers
```

Brace-enclosed initializers use copy initialization.

``` cpp
S* p = new S(4);          // new expression
```

New-expressions use direct initialization.



# 037 Multiple Inheritance - Part I
**Difficulty: 6 / 10**
Some languages, including the emerging SQL3 standard, continue to struggle with the decision of whether to support single or multiple inheritance. This GotW invites you to consider the issues.

----

**Problem**
**JG Question**
**1.** What is multiple inheritance (MI), and what extra possibilities or complications does allowing MI introduce into C++?

**Guru Question**
**2.** Is MI ever necessary? If yes, show as many different situations as you can and argue why MI should be in a language. If no, argue why SI (possibly combined with Java-style interfaces) is equal or superior and why MI should not be in a language.

[NOTE: This GotW is not intended to resurrect long-dead arguments about what C++ should do. It is appropriate to consider the multiple inheritance (MI) issue now, however, because another popular language is currently facing the same decision: Less than two weeks ago in Portland OR, the ANSI SQL3 committee voted to remove MI as a feature of standard OO/ORDBMS. The corresponding ISO committee, meeting in Australia later this month, will consider the same paper and is likely to follow suit. If it goes there as it did at our ANSI meeting, SQL3 will include only SI (without even the compromise of Java-style interfaces). -hps]

----

**Solution**
**1.** What is multiple inheritance (MI), and what extra possibilities or complications does allowing MI introduce into C++?

Very briefly:

MI means the ability to inherit from more than a single direct base class. For example:

``` cpp
class Derived : public Base1, private Base2
{
  //...
};
```

Allowing MI introduces the possibility that a class may have the same (direct or indirect) base class appear more than once as an ancestor. A simple example of this is the "Diamond of Death" shape:

``` cpp
        B
       / \
     C1   C2
       \ /
        D
```

Here B is an indirect base class of D twice, once via C1 and once via C2.

This situation introduces the need to an extra feature in C++: virtual inheritance. Does the programmer want D to have one B subobject, or two? If one, B should be a virtual base class; if two, B should be a normal (nonvirtual) base class.

Finally, the main complication of virtual base classes is that they must be initialized directly by the most-derived class. For more information on this and other aspects of MI, see a good text like C++PL3 or Meyers' "Effective C++" books.

 

**2.** Is MI ever necessary?

Short answer: No feature is strictly "necessary" insofar as any program can be written in assembler (or lower). However, just as most people would rather not code their own virtual function mechanism in plain C, in some cases not having MI requires painful workarounds.

If yes, show as many different situations as you can and argue why MI should be in a language. If no, argue why SI (possibly combined with Java-style interfaces) is equal or superior and why MI should not be in a language.

Short answer: Yes and no.

Like any tool, MI should be used carefully. Using MI always adds complexity (see JG question above), but there are situations when it is still simpler and more maintainable than the alternatives. As some have put it: "You only need MI rarely, but when you need it you REALLY need it."

There are many situations where MI is a convenient and appropriate tool. I'll just cover three (in fact, most uses fall into these three categories):

*1. Interface Classes (Pure Abstract Base Classes)*

In C++, MI's best and safest use is to define interface classes, that is, classes composed of nothing but pure virtual functions. In particular, it's the absence of data members in the base class that avoids MI's more famous complexities.

Interestingly, different languages/models support this kind of "MI" through non-inheritance mechanisms. Two examples are Java and COM: Java has only SI, but supports the notion that a class can implement multiple "interfaces" where the interfaces are very similar to C++ pure ABCs. COM does not include inheritance but likewise has a notion of composition of interface, and this model is similar to a combination of Java interfaces and C++ templates.

*2. Combining Modules/Libraries*

Many classes are designed to be base classes; that is, to use them you are intended to inherit from them. The natural consequence: What if you want to write a class that extends two libraries, and you are required to inherit from a class in each? Because you usually don't have the option of changing the library code (if you purchased the library from a third-party vendor, or it is a module produced by another project team inside your company), MI is necessary.

*3. Ease of (Polymorphic) Use*

There are examples where allowing MI greatly simplifies using the same object polymorphically in different ways. One good example is found in C++PL3 14.2.2 which demonstrates an MI-based design for exception classes, where a most-derived exception class may have a polymorphic IS-A relationship with multiple direct base classes.

Note that #1 overlaps greatly with #3.

Finally, don't forget that "polymorphic LSP IS-A public inheritance" isn't the only game in town; there are many other possible reasons to use inheritance. The result is that sometimes it's not just necessary to inherit from multiple base classes, but to do so from each one for different reasons. For example, a class may need to inherit privately from base class A to gain access to protected members of class A, but at the same time inherit publicly from base class B to polymorphically implement a virtual function of class B.



# 038 Multiple Inheritance - Part II
**Difficulty: 8 / 10**
If you couldn't use multiple inheritance, how would you emulate it? Don't forget to emulate as natural a syntax as possible for the client code.



**Problem**
**1.** Consider the following example:

``` cpp
    struct A { 
        virtual ~A() { }                
        virtual string Name() { return "A";  } 
    };
    
    struct B1 : virtual A { string Name() { return "B1"; } };
    struct B2 : virtual A { string Name() { return "B2"; } };
    struct D  : B1, B2    { string Name() { return "D";  } };
```

Demonstrate the best way you can find to "work around" not using multiple inheritance by writing an equivalent (or as near as possible) class D without using MI. How would you get the same effect and usability for D with as little change as possible to syntax in the client code?

**Starter:** You can begin by considering the cases in the following test harness.

``` cpp
void f1( A&  x ) { cout << "f1:" << x.Name() << endl; }
void f2( B1& x ) { cout << "f2:" << x.Name() << endl; }
void f3( B2& x ) { cout << "f3:" << x.Name() << endl; }

void g1( A   x ) { cout << "g1:" << x.Name() << endl; }
void g2( B1  x ) { cout << "g2:" << x.Name() << endl; }
void g3( B2  x ) { cout << "g3:" << x.Name() << endl; }

int main() {
    D   d;
    B1* pb1 = &d;   // D* -> B* conversion
    B2* pb2 = &d;
    B1& rb1 = d;    // D& -> B& conversion
    B2& rb2 = d;

    f1( d );        // polymorphism
    f2( d );
    f3( d );
    
    g1( d );        // slicing
    g2( d );
    g3( d );
                    // dynamic_cast/RTTI
    cout << ( (dynamic_cast<D*>(pb1) != 0)
            ? "ok " : "bad " );
    cout << ( (dynamic_cast<D*>(pb2) != 0)
            ? "ok " : "bad " );
    
    try {
        dynamic_cast<D&>(rb1);
        cout << "ok ";
    } catch(...) {
        cout << "bad ";
    }
    try {
        dynamic_cast<D&>(rb2);
        cout << "ok ";
    } catch(...) {
        cout << "bad ";
    }
}
```

----

**Solution**
**1.** Consider the following example:

``` cpp
struct A      { virtual ~A() { }
                virtual string Name() { return "A";  } };
struct B1 : virtual A { string Name() { return "B1"; } };
struct B2 : virtual A { string Name() { return "B2"; } };

struct D  : B1, B2    { string Name() { return "D";  } };
```

Demonstrate the best way you can find to "work around" not using multiple inheritance by writing an equivalent (or as near as possible) class D without using MI. How would you get the same effect and usability for D with as little change as possible to syntax in the client code?

There are a few strategies, each with their weaknesses, but here's one that gets quite close:

``` cpp
struct D : B1 {
    struct D2 : B2 {
        void   Set ( D* d ) { d_ = d; }
        string Name();
        D* d_;
    } d2_;

    D()                 { d2_.Set( this ); }

    D( const D& other ) : B1( other ), d2_( other.d2_ )
                        { d2_.Set( this ); }

    D& operator=( const D& other ) {
                          B1::operator=( other );
                          d2_ = other.d2_;
                          return *this;
                        }

    operator B2&()      { return d2_; }

    B2& AsB2()          { return d2_; }

    string Name()       { return "D"; }
};

string D::D2::Name()    { return d_->Name(); }
```

Some drawbacks are that:

    - providing operator B2& arguably gives references special (inconsistent) treatment over pointers
    
    - you need to call D::AsB2() explicitly to use a D as a B2 (in the test harness, this means changing "B2* pb2 = &d;" to "B2* pb2 = &d.AsB2();")
    
    - dynamic_cast from D* to B2* still doesn't work (it's possible to work around this if you're willing to use the preprocessor to redefine dynamic_cast calls)

Interestingly, you may have observed that the D object layout in memory is probably identical to what multiple inheritance would give. That's because we're trying to simulate MI, just without all of the syntactic sugar and convenience that builtin language support would provide.




# 039 Multiple Inheritance - Part III
**Difficulty: 4 / 10**
Overriding inherited virtual functions is easy \-\- as long as you're not trying to override a virtual function that has the same signature in two base classes. This can happen even when the base classes don't come from different vendors!

----

**Problem**
**1.** Consider the following two classes:

``` cpp
class B1 {
public:
    virtual int ReadBuf( const char* );
    // ...
};

class B2 {
public:
    virtual int ReadBuf( const char* );
    // ...
};
```

Both are clearly intended to be used as base classes but they are otherwise unrelated \-\- their ReadBuf functions are intended to do different things, and the classes come from different library vendors.

Demonstrate how to write a class D, publicly derived from both B1 and B2, which overrides both ReadBufs independently to do different things.

----

**Solution**
Demonstrate how to write a class D, publicly derived from both B1 and B2, which overrides both ReadBufs independently to do different things.

Here's the naive attempt that won't work:

``` cpp
class D : public B1, public B2 {
public:
    int ReadBuf( const char* );
    // overrides both B1::ReadBuf and B2::ReadBuf
};
```

This overrides BOTH functions with the same implementation, whereas the point of the question was to override the two functions to do different things. You can't just "switch" behaviours inside this D::ReadBuf depending which way it gets called, either, because once you're inside D::ReadBuf there's no way of telling which base interface was used (if any).

**Renaming Virtual Functions**
If the two inherited functions had different signatures, there would be no problem: We would just override them independently as usual. The trick, then, is to somehow change the signature of at least one of the two inherited functions.

The way to change a base class function's signature is to create an intermediate class which derives from the base class, declares a new virtual function, and overrides the inherited version to call the new function:

``` cpp
class D1 : public B1 {
public:
    virtual int ReadBufB1( const char* p ) = 0;
    int ReadBuf( const char* p ) // override inherited
      { return ReadBufB1( p ); } // to call new func
};

class D2 : public B2 {
public:
    virtual int ReadBufB2( const char* p ) = 0;
    int ReadBuf( const char* p ) // override inherited
      { return ReadBufB2( p ); } // to call new func
};
```

D1 and D2 may also need to duplicate constructors of B1 and B2 so that D can invoke them, but that's it. D1 and D2 are abstract classes, so they do NOT need to duplicate any other B1/B2 functions or operators, such as assignment operators.

Now we can simply write:

``` cpp
class D : public D1, public D2 {
public:
    int ReadBufB1( const char* );
    int ReadBufB2( const char* );
};
```

Derived classes only need to know that they must not further override ReadBuf itself.


# 040 Controlled Polymorphism
**Difficulty: 8 / 10**
IS-A polymorphism is a very useful tool in OO modeling, but sometimes you may want to restrict which code can use certain classes polymorphically. This issue gives an example, and shows how to get the intended effect.

----

**Problem**
**1.** Consider the following code:

``` cpp
class Base {
public:
    virtual void VirtFunc();
    // ...
};

class Derived : public Base {
public:
    void VirtFunc();
    // ...
};
```

  void SomeFunc( const Base& );
There are two other functions. The goal is to allow f1 to use Derived object polymorphically where a Base is expected, yet prevent all other functions (including f2) from doing so.

``` cpp
void f1() {
    Derived d;
    SomeFunc( d ); // works, OK
}

void f2() {
    Derived d;
    SomeFunc( d ); // we want to prevent this
}
```

Demonstrate how to achieve this effect.

----

**Solution**
**1.** Consider the following code:

``` cpp
class Base {
public:
    virtual void VirtFunc();
    // ...
};

class Derived : public Base {
public:
    void VirtFunc();
    // ...
};

void SomeFunc( const Base& );
```

The reason why all code can use Derived objects polymorphically where a Base is expected is because Derived inherits publicly from Base (no surprises here).

If instead Derived inherited privately from Base, then "almost" no code could use Deriveds polymorphically as Bases. The reason for the "almost" is that code with access to the private parts of Derived CAN access the private base classes of Derived and can therefore use Deriveds polymorphically in place of Bases. Normally, only the member functions of Derived have such access. However, we can use "friend" to extend similar access to other outside code.

Putting the pieces together, we get:

There are two other functions. The goal is to allow f1 to use Derived object polymorphically where a Base is expected, yet prevent all other functions (including f2) from doing so.

``` cpp
void f1() {
    Derived d;
    SomeFunc( d ); // works, OK
}

void f2() {
    Derived d;
    SomeFunc( d ); // we want to prevent this
}
```

Demonstrate how to achieve this effect.

The answer is to write:

``` cpp
class Derived : private Base {
public:
    void VirtFunc();
    // ...
    friend void f1();
};
```

This solves the problem cleanly, although it does give f1 greater access than f1 had in the original version.



# 041 Using the Standard Library
**Difficulty: 9 / 10**
How well do you know the standard library's algorithms? This puzzle requires a "master mind."

----

**Problem**
**1.** Write a program that plays simplified Mastermind, using only Standard Library containers, algorithms and streams. The challenge is to use as few "if"s, "while"s, "for"s and other builtin constructs as possible, and have the program take up as few statements as possible.

Simplified Rules Summary
To start the game, the program randomly chooses a string of four pegs in some internal order. Each peg can be one of three colours: red (R), green (G) or blue (B). For example, the program might pick "RRBB" or "GGGG" or "BGRG".

The player makes successive guesses until he figures out the order. For each guess, the program tells the player two numbers: the first is the number of pegs that are the correct colour (independent of order); and the second is the number of pegs with the correct colour AND the correct location.

Here is an example session, where the program chose the combination "RRBB":

``` sh
guess--> rbrr
3 1

guess--> rbgg
2 1

guess--> bbrr
4 0

guess--> rrbb
4 4 - solved!
```

(A more complex form of this puzzle was proposed by Tom Holaday at the November 1997 meeting: To write a program that plays the full advanced Mastermind, using only one expression. It kept half a dozen committee members amused for hours during the evenings.)

-----

**Solution**
**1.** Write a program that plays simplified Mastermind, using only Standard Library containers, algorithms and streams. The challenge is to use as few "if"s, "while"s, "for"s and other builtin constructs as possible, and have the program take up as few statements as possible.

The solution presented below is not the only right answer. In fact, it may not even be the cleanest, and it's relatively unimaginative. Each of the solutions that were posted to the newsgroup have aspects that I like better (such as creative use of count<> and inner_product<>) and aspects that I don't (such as randomizing the combination poorly and hardcoding the peg colours within the algorithm). All of the solutions, including this one, do insufficient error checking.

This solution avoids hardcoding peg-colour and combination-size restrictions within the algorithm; peg colours can be extended simply by changing the 'colors' string, and the combination length can be changed simply by changing the '4' literal. It does use one "while," however, which can replaced (at further cost of clarity) by "find_if( istream_iterator<string>(cin), ... );".

  string colors("BGR"), comb(4, '.'), l(comb), guess;

``` cpp
typedef map<int,int> M;

struct Color {
    Color( M& cm, M& gm, int& color )
        : cm_(cm), gm_(gm), color_(color=0) { }
        
    void operator()( char c ) 
    {
        color_ += min( cm_[c], gm_[c] ); 
    }
    
    M &cm_, &gm_;
    int& color_;
};

struct Count {
    Count( int& color, int& exact )
        : color_(color), exact_(exact=0) { }
        
    char operator()( char c, char g ) { 
        return ++cm_[c], ++gm_[toupper(g)], exact_ += c == toupper(g), '.'; 
    }
               
    ~Count() {
        for_each(colors.begin(), colors.end(), Color(cm_, gm_, color_)); 
    }
    
    M cm_, gm_;
    int &color_, &exact_;
};

char ChoosePeg()
{
    return colors[rand() % colors.size()]
}

int main() 
{
    int color, exact = 0;
    srand( time(0) ),
        generate( comb.begin(), comb.end(), ChoosePeg );
    while( exact < comb.length() ) {
        cout << "\n\nguess--> ", cin >> guess;
        transform( comb.begin(), comb.end(),
                   guess.begin(), l.begin(),
                   Count( color, exact ) );
        cout << color << ' ' << exact;
    }
    cout << " - solved!\n";
}
```



# 042 Using auto_ptr
**Difficulty: 5 / 10**
This issue illustrates a common pitfall with using auto_ptr. What is the problem, and how can you solve it?

----

**Problem**
**JG Question**
**1.** Consider the following function, which illustrates a common mistake when using auto_ptr:

``` cpp
    template <class T>
    void f( size_t n ) {
        auto_ptr<T> p1( new T );
        auto_ptr<T> p2( new T[n] );
        //
        // ... more processing ...
        //
    }
```

What is wrong with this code? Explain.

**Guru Question**
**2.** How would you fix the problem? Consider as many options as possible, including: the Adapter pattern; alternatives to the problematic construct; and alternatives to auto_ptr.

----

**Solution**
**Problem: Arrays and auto_ptr Don't Mix**
**1.** Consider the following function, which illustrates a common mistake when using auto_ptr:

``` cpp
    template <class T>
    void f( size_t n ) {
        auto_ptr<T> p1( new T );
        auto_ptr<T> p2( new T[n] );
        //
        // ... more processing ...
        //
    }
```

What is wrong with this code? Explain.

Every "delete" must match the form of its "new". If you use single-object new, you must use single-object delete; if you use the array form of new, you must use the array form of delete. Doing otherwise yields undefined behaviour, as illustrated in the following slightly-modified code:

``` cpp
    T* p1 = new T;
    // delete[] p1; // error
    delete p1;      // ok - this is what auto_ptr does
    
    T* p2 = new T[10];
    delete[] p2;    // ok
    // delete p2;   // error - this is that auto_ptr does
```

The problem with p2 is that auto_ptr is intended to contain only single objects, and so it always calls "delete" \-\- not "delete[]" \-\- on the pointer that it owns. This means that p1 will be cleaned up correctly, but p2 will not.

What will actually happen when you use the wrong form of delete depends on your compiler. The best you can expect is a resource leak, but a more typical result is memory corruption soon followed by a core dump. To see this effect, try the following complete program on your favourite compiler:

``` cpp
#include <iostream>
#include <memory>
#include <string>
using namespace std;

int c = 0;

struct X {
    X() : s( "1234567890" ) { ++c; }
    ~X()                    { \-\-c; }
    string s;
};

template <class T>
void f( size_t n ) {
    {
        auto_ptr<T> p1( new T );
        auto_ptr<T> p2( new T[n] );
    }
    cout << c << " ";   // report # of X objects
}                       // that currently exist

int main() {
    while( true ) {
        f<X>(100);
    }
}
```

This will either crash, or else output a running update of the number of leaked X objects. (For extra fun, try running a system monitoring tool in another window that shows your system's total memory usage. It will help you to appreciate how bad the leak can be if the program doesn't just crash right off.)

**Non-Problem: Zero-Length Arrays Are Okay**
What if f's parameter is zero (e.g., in the call f<int>(0))? Then the second new turns into "new T[0]" and often programmers will wonder: "Hmm, is this okay? Can we have a zero-length array?"

The answer is Yes. Zero-length arrays are perfectly okay, kosher, and fat-free. The result of "new T[0]" is just a pointer to an array with zero elements, and that pointer behaves just like any other result of "new T[n]" including the fact that you may not attempt to access more than n elements of the array... in this case, you may not attempt to access any elements at all, because there aren't any.

From 5.3.4 [expr.new], paragraph 7:

    When the value of the expression in a direct-new-declarator is zero, the allocation function is called to allocate an array with no elements. The pointer returned by the new-expression is non-null. [Note: If the library allocation function is called, the pointer returned is distinct from the pointer to any other object.]

"Well, if you can't do anything with zero-length arrays (other than remember their address)," you may wonder, "why should they be allowed?" One important reason is that it makes it easier to write code that does dynamic array allocation. For example, the function f above would be needlessly more complex if it was forced to check the value of its n parameter before performing the "`new T[n]`" call.

**2.** How would you fix the problem? Consider as many options as possible, including: the Adapter pattern; alternatives to the problematic construct; and alternatives to auto_ptr.

There are several options (some better, some worse). Here are four:

**Option 1:** Roll Your Own auto_array
This can be both easier and harder than it sounds:

**Option 1 (a):** ... By Deriving From auto_ptr (Score: 0 / 10)

Bad idea. For example, you'll have a lot of fun reproducing all of the ownership and helper-class semantics. This might only be tried by true gurus, but true gurus would never try it because there are easier ways.

Advantages: Few.

Disadvantages: Too many to count.

**Option 1 (b):** ... By Cloning auto_ptr Code (Score: 8 / 10)

The idea here is to take the code from your library's implementation of auto_ptr, clone it (renaming it auto_array or something like that), and change the "`delete`" statements to "`delete[]`" statements.

Advantages:

    a) EASY TO IMPLEMENT (ONCE). We don't need to hand-code our own auto_array, and we keep all the semantics of auto_ptr automatically, which helps avoid surprises for future maintenance programmers who are already familiar with auto_ptr.
    
    b) NO SIGNIFICANT SPACE OR TIME OVERHEAD.

Disadvantages:

    a) HARD TO MAINTAIN. You'll probably want to be careful to keep your auto_array in sync with your library's auto_ptr when you upgrade to a new compiler/library version or switch compiler/library vendors.

**Option 2: Use the Adapter Pattern (Score: 7 / 10)**
This option came out of a discussion I had with C++ World attendee Henrik Nordberg after one of my talks. Henrik's first reaction to the problem code was to wonder whether it would be easiest to write an adapter to make the standard auto_ptr work correctly, instead of rewriting auto_ptr or using something else. This idea has some real advantages, and deserves analysis despite its few drawbacks.

The idea is as follows: Instead of writing

``` cpp
    auto_ptr<T> p2( new T[n] );
```

we write

``` cpp
    auto_ptr<ArrDelAdapter<T> > p2(new ArrDelAdapter<T>(new T[n]));
```

where the ArrDelAdapter ("array deletion adapter") has a constructor that takes a `T*` and a destructor that calls `delete[]` on that pointer:

``` cpp
template <class T>
class ArrDelAdapter {
public:
    ArrDelAdapter( T* p ) : p_(p) { }
    ~ArrDelAdapter() { delete[] p_; }
    // operators like "->" "T*" and other helpers
private:
    T* p_;
};
```

Since there is only one `ArrDelAdapter<T>` object, the single-object delete statement in `~auto_ptr` is fine; and since `~ArrDelAdapter<T>` correctly calls `delete[]` on the array, the original problem has been solved.

Sure, this may not be the most elegant and beautiful approach in the world, but at least we didn't have to hand-code our own auto_array template!

Advantages:

    a) EASY TO IMPLEMENT (INITIALLY). We don't need to write an auto_array. In fact, we get to keep all the semantics of auto_ptr automatically, which helps avoid surprises for future maintenance programmers who are already familiar with auto_ptr.
    
    Disadvantages:
    
    a) HARD TO READ. This solution is rather verbose.
    
    b) (POSSIBLY) HARD TO USE. Any code later in f that uses the value of the p2 auto_ptr will need syntactic changes, which will often be made more cumbersome by extra indirections.
    
    c) INCURS SPACE AND TIME OVERHEAD. This code requires extra space to store the required adapter object for each array. It also requires extra time because it performs twice as many memory allocations (this can be ameliorated by using an overloaded operator new), and then an extra indirection each time client code accesses the contained array.

Having said all that: Even though the other alternatives turn out to be better in this particular case, I was very pleased to see people immediately think of using the Adapter pattern. Adapter is widely applicable, and one of the core patterns that every programmer should know.

FINAL NOTE ON 2: It's worth pointing out that writing

``` cpp
    auto_ptr<ArrDelAdapter<T> > p2(new ArrDelAdapter<T>(new T[n]));
```

isn't much different from writing

``` cpp
    auto_ptr< vector<T> > p2( new vector<T>(n) );
```

Think about that for a moment \-\- for example, ask yourself, "What if anything am I gaining by allocating the vector dynamically that I wouldn't have if I just wrote "vector p2(n);"? --, then see Option 4.

**Option 3: Replace auto_ptr With Hand-Coded EH Logic (Score: 1 / 10 )**
Function f uses auto_ptr for automatic cleanup, and probably for exception safety. Instead, we could drop auto_ptr for the p2 array and hand-code our own exception-handling (EH) logic.

The idea is as follows: Instead of writing

``` cpp
    auto_ptr<T> p2( new T[n] );
    //
    // ... more processing ...
    //
```

we write something like

``` cpp
    T* p2( new T[n] );
    try {
        //
        // ... more processing
        //
    }
    delete[] p2;
```

Advantages:

a) EASY TO USE. This solution has little impact on the code in "more processing" that uses p2; probably all that's necessary is to remove ".get()" wherever it occurs.

b) NO SIGNIFICANT SPACE OR TIME OVERHEAD.

Disadvantages:

    a) HARD TO IMPLEMENT. This solution probably involves many more code changes than are suggested by the above. The reason is that, while the auto_ptr for p1 will automatically clean up the new T no matter how the function exits, to clean up p2 we now have to write cleanup code along every code path that might exit the function. For example, consider the case where "more processing" includes more branches, some of which end with "return;".
    
    b) BRITTLE. See (a): Did we put the right cleanup code along all code paths?
    
    c) HARD TO READ. See (a): The extra cleanup logic is likely to obscure the function's normal logic.

**Option 4: Use a vector<> Instead Of an Array (Score: 9.5 / 10 )**
Most of the problems we've encountered have been due to the use of C-style arrays. If appropriate \-\- and it's almost always appropriate \-\- it would be better to use a vector instead of a C-style array. After all, a major reason why vector exists in the standard library is to provide a safer and easier-to-use replacement for C-style arrays!

The idea is as follows: Instead of writing

``` cpp
    auto_ptr<T> p2( new T[n] );
```

we write

``` cpp
    vector<T> p2( n );
```

Advantages:

    a) EASY TO IMPLEMENT (INITIALLY). We still don't need to write an auto_array.
    
    b) EASY TO READ. People who are familiar with the standard containers (and that should be everyone, by now!) will immediately understand what's going on.
    
    c) LESS BRITTLE. Since we're pushing down the details of memory management, our code is (usually) further simplified. We don't need to manage the buffer of T objects... that's the job of the vector<T> object.
    
    d) NO SIGNIFICANT SPACE OR TIME OVERHEAD.

Disadvantages:

    a) SYNTACTIC CHANGES. Any code later in f that uses the value of the p2 auto_ptr will need syntactic changes, although the changes will be fairly simple and not as drastic as those required by Option 2.
    
    b) (SOMETIMES) USABILITY CHANGES. You can't instantiate any standard container (including a vector) of T's if T objects are not copy-constructible and assignable. Most types are both copy-constructible and assignable, but if they are not then this solution won't work.

Note that passing or returning a vector by value is much more work than passing or returning an auto_ptr. I consider this objection somewhat of a red herring, however, because it's an unfair comparison... if you wanted to get the same effect, you would simply pass a pointer or reference to the vector.

From the GotW coding standards:

    - prefer using vector<> instead of built-in (C-style) arrays



# 043 Reference Counting - Part I
**Difficulty: 4 / 10**
Reference counting is a common optimization (also called "lazy copy" and "copy on write"). Do you know how to implement it?

----

**Problem**
**JG Question**
**1.** Consider the following simplified String class:

``` cpp
namespace Original {
    class String {
    public:
        String();                // start off empty
        ~String();               // free the buffer
        String( const String& ); // take a full copy
        void Append( char );     // append one character
    private:
        char*    buf_;           // allocated buffer
        size_t   len_;           // length of buffer
        size_t   used_;          // # chars actually used
    };
}
```

This is a simple String that does not contain any fancy optimizations. When you copy an Original::String, the new object immediately allocates its own buffer and you immediately have two completely distinct objects.

Your assignment: Implement Original::String.

Guru Question
**2.** Unfortunately, sometimes copies of string objects are taken, used without modification, and then discarded. "It seems a shame," the implementer of Original::String might frown to herself, "that I always do all the work of allocating a new buffer (which can be expensive) when it may turn out that I never needed to, if all the user does is read from the new string and then destroy it. I could have just let the two string objects share a buffer under the covers, avoiding the copy for a while, and only really take a copy when I know I need to because one of the objects is going to try to modify the string. That way, if the user never modifies the copy, I never need to do the extra work!"

With a smile on her face and determination in her eyes, the implementer designs an Optimized::String that uses a reference-counted implementation (also called "lazy copy" or "copy on write"):

``` cpp
namespace Optimized {
    struct StringBuf {
        StringBuf();             // start off empty
        ~StringBuf();            // delete the buffer
        void Reserve( size_t n );// ensure len >= n
    
        char*    buf;            // allocated buffer
        size_t   len;            // length of buffer
        size_t   used;           // # chars actually used
        unsigned refs;           // reference count
    };
    
    class String {
    public:
        String();                // start off empty
        ~String();               // decrement reference count
                                 //  (delete buffer if refs==0)
        String( const String& ); // point at same buffer and
                                 //  increment reference count
        void   Append( char );   // append one character
    private:
        StringBuf* data_;
    };
}
```

Your assignment: Implement Optimized::StringBuf and Optimized::String. You may want to add a private String::AboutToModify() helper function to simplify the logic.

Author's Notes
**1.** I don't show operator= because it's essentially the same as copy construction.

**2.** This issue of GotW incidentally illustrates another use for namespaces, namely clearer exposition. Writing the above was much nicer than writing about differently-named "OriginalString" and "OptimizedString" classes (which would have made reading and writing the example code a little harder). Also, the namespace syntax is very natural here in the expository paragraphs. Using namespaces judiciously can likewise improve readability in your production code and "talkability" during design meetings and code reviews.

----

**Solution**
This is a simple String that does not contain any fancy optimizations. When you copy an Original::String, the new object immediately allocates its own buffer and you immediately have two completely distinct objects.

Your assignment: Implement Original::String.

Here is a straightforward implementation.

``` cpp
namespace Original {

    String::String() : buf_(0), len_(0), used_(0) { }
    
    String::~String() { delete[] buf_; }
    
    String::String( const String& other )
    : buf_(new char[other.len_]),
      len_(other.len_),
      used_(other.used_)
    {
        copy( other.buf_, other.buf_ + used_, buf_ );
    }
```

I've chosen to implement an additional Reserve helper function for code clarity, since it will be needed by other mutators besides Append. Reserve ensures that our internal buffer is at least n bytes long, and allocates additional space using an exponential-growth strategy:

``` cpp
    void String::Reserve( size_t n ) {
        if( len_ < n ) {
            size_t newlen = max( len_ * 1.5, n );
            char*  newbuf = new char[ newlen ];
            copy( buf_, buf_+used_, newbuf );
            
            delete[] buf_;  // now all the real work is
            buf_ = newbuf;  //  done, so take ownership
            len_ = newlen;
        }
    }
    
    void String::Append( char c ) {
        Reserve( used_+1 );
        buf_[used_++] = c;
    }

}
```

**Aside: What's the Best Buffer Growth Strategy?**
When a String object runs out of room in its currently- allocated buffer, it needs to allocate a larger one. The key question is: "How big should the new buffer be?" [Note: The analysis that follows holds for other structures and containers that use allocated buffers, such as a standard vector<>.]

There are several popular strategies. I'll note each strategy's complexity in terms of the number of allocations required, and the average number of times a given character must be copied, for a string of eventual length N.

a) Exact growth. In this strategy, the new buffer is made exactly as large as required by the current operation. For example, appending one character and then appending another will force two allocations; first a new buffer is allocated with space for the existing string and the first new character; next, another new buffer is allocated with space for that and the second new character.

- Advantage: No wasted space.

- Disadvantage: Poor performance. This strategy requires O(N) allocations and an average of O(N) copy operations per char, but with a high constant factor in the worst case (same as (b) with an increment size of 1). Control of the constant factor is in the hands of the user code, not controlled by the String implementer.

b) Fixed-increment growth. The new buffer should be a fixed amount larger than the current buffer. For example, a 64-byte increment size would mean that all string buffers would be an integer multiple of 64 bytes long.

- Advantage: Little wasted space. The amount of unused space in the buffer is bounded by the increment size, and does not vary with the length of the string.

- Disadvantage: Moderate performance. This strategy requires both `O(N)` allocations and an average of `O(N)` copy operations per char. That is, both the number of allocations and the average number of times a given char is copied vary linearly with the length of the string. However, control of the constant factor is in the hands of the String implementer.

c) Exponential growth. The new buffer is a factor of F larger than the current buffer. For example, with F=.5, appending one character to full string which is already 100 bytes long will allocate a new buffer of length 150 bytes.

- Advantage: Good performance. This strategy requires O(logN) allocations and an average of O(k) copy operations per char. That is, the number of allocations varies linearly with the length of the string, but the average number of times a given char is copied is constant(!) which means that the amount of copying work to create a string using this strategy is at most a constant factor greater than the amount of work that would have been required had the size of the string been known in advance.

- Disadvantage: Some wasted space. The amount of unused space in the buffer will always be strictly less than `N*F` bytes, but that's still `O(N)` average space wastage.

The following chart summarizes the tradeoffs:

|Growth Strategy|  Allocations | Char Copies|  Wasted Space|
|---------------|  ----------- | -----------|  ------------|
|Exact          |O(N) with high const.|O(N) with high const.|none|
|Fixed-increment|       O(N)          |         O(N)        |O(k)|
|Exponential    |       O(logN)       |         O(k)        |O(N)|

In general, the best-performing strategy is exponential growth. Consider a string that starts empty and grows to 1200 characters long. Fixed-increment growth requires O(N) allocations and on average requires copying each character O(N) times (in this case, using 32-byte increments, that's 38 allocations and on average 19 copies of each character). Exponential growth requires O(logN) allocations and on average requires making only O(k) \-\- one or two \-\- copies of each character (yes, really; see the reference below); in this case, using a factor of 1.5, that's 10 allocations and on average 2 copies of each character.

```
               1,200-char string       12,000-char string
             ======================  =======================
             Fixed-incr Exponential  Fixed-incr  Exponential
               growth     growth       growth      growth
             (32 bytes)   (1.5x)     (32 bytes)    (1.5x)
             ---------- -----------  ----------  -----------

of memory        38         10          380          16
allocations

avg # of         19          2          190           2
copies made
of each char
```

This result can be surprising. For more information, see Andrew Koenig's column in the September 1998 issue of JOOP (Journal of Object-Oriented Programming). Koenig also shows why, again in general, the best growth factor is not 2 but probably about 1.5. He also shows why the average number of times a given char is copied is constant \-\- i.e., doesn't depend on the length of the string.

Your assignment: Implement Optimized::StringBuf and Optimized::String. You may want to add a private String::AboutToModify() helper function to simplify the logic.

First, consider StringBuf. Note that the default memberwise copying and assignment don't make sense for StringBufs, so both operations should also be suppressed (declared as private and not defined).

``` cpp
namespace Optimized {

    StringBuf::StringBuf() : buf(0), len(0), used(0), refs(1) { }
    
    StringBuf::~StringBuf() { delete[] buf; }
    
    void StringBuf::Reserve( size_t n ) {
        if( len < n ) {
            size_t newlen = max( len * 1.5, n );
            char*  newbuf = new char[ newlen ];
            copy( buf, buf+used, newbuf );
            
            delete[] buf;   // now all the real work is
            buf = newbuf;   //  done, so take ownership
            len = newlen;
        }
    }
```

Next, consider String itself. The default constructor and the destructor are easy to implement.

``` cpp
    String::String() : data_(new StringBuf) { }
    
    String::~String() {
        if( --data_->refs < 1 ) {
            delete data_;
        }
    }
```

Next, we write the copy constructor to implement the "lazy copy" semantics by simply updating a reference count. We will only actually split off the shared representation if we discover that we need to modify one of the strings that share this buffer.

``` cpp
    String::String( const String& other )
    : data_(other.data_)
    {
        ++data_->refs;
    }
```

I've chosen to implement an additional AboutToModify helper function for code clarity, since it will be needed by other mutators besides Append. AboutToModify ensures that we have an unshared copy of the internal buffer; that is, it performs the "lazy copy" if it has not already been performed. For convenience, AboutToModify takes a minimum buffer size, so that we won't needlessly take our own copy of a full string only to turn around and immediately perform a second allocation to get more space.

``` cpp
    void String::AboutToModify( size_t n ) {
        if( data_->refs > 1 ) {
            auto_ptr<StringBuf> newdata( new StringBuf);
            newdata.get()->Reserve(max(data_->len, n));
            copy(data_->buf, data_->buf+data_->used, newdata.get()->buf);
            newdata.get()->used = data_->used;
            
            --data_->refs;             // now all the real work is
            data_ = newdata.release(); //  done, so take ownership
        }
        else {
            data_->Reserve( n );
        }
    }
    
    void String::Append( char c ) {
        AboutToModify( data_->used+1 );
        data_->buf[data_->used++] = c;
    }
}
```



# 044 Reference Counting - Part II
**Difficulty: 6 / 10**
In the second of this three-part miniseries, we examine the effect of references and iterators into a reference-counted string. Can you spot the issues?

----

**Problem**
Consider the reference-counted Optimized::String class from GotW #43, but with two new functions: Length() and operator[].

``` cpp
namespace Optimized {
    struct StringBuf {
        StringBuf();              // start off empty
        ~StringBuf();             // delete the buffer
        void Reserve( size_t n ); // ensure len >= n
    
        char*    buf;             // allocated buffer
        size_t   len;             // length of buffer
        size_t   used;            // # chars actually used
        unsigned refs;            // reference count
    };
    
    class String {
    public:
        String();                 // start off empty
        ~String();                // decrement reference count
                                  //  (delete buffer if refs==0)
        String( const String& );  // point at same buffer and
                                  //  increment reference count
        void   Append( char );    // append one character
        size_t Length() const;    // string length
        char&  operator[](size_t);// element access
    private:
        void AboutToModify( size_t n );
                                  // lazy copy, ensure len>=n
        StringBuf* data_;
    };
}
```

This allows code like the following:

``` cpp
  if( s.Length() > 0 ) {
    cout << s[0];
    s[0] = 'a';
  }
```

**Guru Questions**
1. Implement the new members of Optimized::String.

2. Do any of the other members need to be changed because of the new member functions? Explain.

----

**Solution**
Consider the reference-counted Optimized::String class from GotW #43, but with two new functions: Length() and operator[].

The point of this GotW is to demonstrate why adding `operator[]` changes `Optimized::String`'s semantics enough to impact other parts of the class. But, first things first:

**1.** Implement the new members of Optimized::String.

The `Length()` function is simple:

``` cpp
  namespace Optimized {
    size_t String::Length() const {
        return data_->used;
    }
```

There's more to `operator[]`, however, than meets the casual eye. In the following, what I want you to notice is that what `operator[]` does (it returns a reference into the string) is really no different from what `begin()` and `end()` do for standard strings (they return iterators that "point into" the string). Any copy-on-write implementation of `std::basic_string` will run into the same considerations that we do below.

Writing `operator[]` for Shareable Strings
Here's a naive first attempt:

``` cpp
    //  BAD: Naive attempt #1 at operator[]

    char& String::operator[]( size_t n ) {
        return *(data_->buf+n);
    }
```

This isn't good enough by a long shot. Consider:

``` cpp
    //  Example 1a: Why attempt #1 doesn't work

    void f( Optimized::String& s ) {
        Optimized::String s2( s ); // take a copy of the string
        s[0] = 'x';                // oops: also modifies s2!
    }
```

You might be thinking that the poor programmer of Example 1a might be a little unhappy about this side effect. You would be right.

So, at the very least, we'd better make sure that the string isn't being shared, else the caller might inadvertently be modifying what look to him like two unrelated strings. "Aha," thinks the once-naive programmer, "I'll call `AboutToModify(`) to ensure I'm using an unshared representation:"

``` cpp
    //  BAD: Inadequate attempt #2 at operator[]

    char& String::operator[]( size_t n ) {
        AboutToModify( data_->len );
        return *(data_->buf+n);
    }
```

This looks good, but it's still not enough. The problem is that we only need to rearrange the Example 1a problem code slightly to get back into the same situation as before:

``` cpp
    //  Example 2a: Why attempt #2 doesn't work either

    void f( Optimized::String& s ) {
        char& rc = s[0];  // take a reference to the first char
        Optimized::String s2( s ); // take a copy of the string
        rc = 'x';                  // oops: also modifies s2!
    }
```

You might be thinking that the poor programmer of Example 2a might be a little perturbed about this surprise, too. You would be right, but as of this writing certain popular implementations of basic_string have precisely this copy-on-write-related bug.

The problem is that the reference was taken while the original string was unshared, but then the string became shared and the single update through the reference modified both String objects' visible state.

**Key Notion: An "Unshareable" String**
When `operator[]` is called, besides taking an unshared copy of the StringBuf, what also need to mark the string "unshareable," just in case the user remembers the reference (and later tries to use it).

Now, marking the string "unshareable for all time" will work, but that's actually a little excessive: It turns out that all we really need to do is mark the string "unshareable for a while." To see what I mean, consider that it's already true that references returned by `operator[]` into the string must be considered invalid after the next mutating operation. That is, code like this:

``` cpp
    //  Example 3: Why references are invalidated
    //             by mutating operations

    void f( Optimized::String& s ) {
        char& rc = s[0];
        s.Append( 'i' );
        rc = 'x';   // error: oops, buffer may have moved
    }               //        if s did a reallocation
```

should already be documented as invalid, whether or not the string uses copy-on-write. In short, calling a mutator clearly invalidates references into the string because you never know if the underlying memory may move (invisibly, from the point of view of the calling code).

Given that fact, in Example 2a, rc would be invalidated anyway by the next mutating operation on s. So, instead of marking s as "unshareable for all time" just because someone might have remembered a reference into it, we could just mark it "unshareable until after the next mutating operation," when any such remembered reference would be invalidated anyway. To the user, the documented behaviour is the same.

**2.** Do any of the other members need to be changed because of the new member functions? Explain.

As we can now see, the answer is: Yes.

First, we need to be able to remember whether a given string is currently unshareable (so that we won't use reference counting when copying it). We could throw in a bool flag, but to avoid even that overhead let's just encode the "unshareable" state directly in the refs count by agreeing that "refs == the biggest unsigned int there can possibly be" means "unshareable." We'll also add a flag to `AboutToModify()` that says whether to mark the string unshareable.

``` cpp
    //  GOOD: Correct attempt #3 at operator[]

    //  Add a new static member for convenience, and
    //  change AboutToModify appropriately. Because now
    //  we'll need to clone a StringBuf in more than
    //  one function (see the String copy constructor,
    //  below), we'll also move that logic into a
    //  single function... it was time for StringBuf to
    //  have its own copy constructor, anyway.

    size_t String::Unshareable = numeric_limits<unsigned>::max();
    
    StringBuf::StringBuf( const StringBuf& other, size_t n )
      : buf(0), len(0), used(0), refs(1)
    {
        Reserve( max( other.len, n ) );
        copy( other.buf, other.buf+other.used, buf );
        used = other.used;
    }
    
    void String::AboutToModify(size_t n,
                               bool bMarkUnshareable /* = false */)
    {
        if( data_->refs > 1 && data_->refs != Unshareable ) {
            StringBuf* newdata = new StringBuf( *data_, n );
            --data_->refs;   // now all the real work is
            data_ = newdata; //  done, so take ownership
        } 
        else {
            data_->Reserve( n );
        }
        data_->refs = bMarkUnshareable ? Unshareable : 1;
    }
    
    char& String::operator[]( size_t n ) {
        AboutToModify( data_->len, true );
        return *(data_->buf+n);
    }
```

Note that all of the other calls to `AboutToModify()` continue to work as originally written.

Now all we need to do is make String's copy constructor respect the unshareable state, if it's set:

``` cpp
    String::String( const String& other )
    {
        //  If possible, use copy-on-write.
        //  Otherwise, take a deep copy immediately.

        if( other.data_->refs != Unshareable ) {
          data_ = other.data_;
          ++data_->refs;
        } else {
          data_ = new StringBuf( *other.data_ );
        }
    }
```

The destructor needs a small tweak:

``` cpp
    String::~String() {
        if( data_->refs == Unshareable || --data_->refs < 1 ) {
          delete data_;
        }
    }
```

All other functions (both of them!) work as originally written:

``` cpp
    String::String() : data_(new StringBuf) { }
    
    void String::Append( char c ) {
        AboutToModify( data_->used+1 );
        data_->buf[data_->used++] = c;
    }
}
```

And that's all(!) there is to it. In the final GotW of this reference-counting miniseries, we'll consider how multithreading affects our reference-counted string. See GotW #45 for the juicy details.

----

Here's all the code together.

Note that I've also taken this opportunity to implement a slight change to `StringBuf::Reserve()`. It now rounds up the chosen "`new buffer size`" to the next multiple of four, in order to ensure that the size of the memory buffer size is always a multiple of four bytes.

This is in the name of efficiency: Many popular operating systems don't allocate memory in chunks smaller than this, anyway, and this code is faster than the original one particularly for small strings. (The original code would allocate a 1-byte buffer, then a 2-byte buffer, then a 3-byte buffer, then a 4-byte buffer, and then a 6-byte buffer before the exponential-growth strategy would really kick in. The code below goes straight to a 4-byte buffer, then an 8-byte buffer, and so on.)

``` cpp
namespace Optimized {
    struct StringBuf {
        StringBuf();              // start off empty
        ~StringBuf();             // delete the buffer
        StringBuf( const StringBuf& other, size_t n = 0 );
                                  // initialize to copy of other,
                                  //  and ensure len >= n
    
        void Reserve( size_t n ); // ensure len >= n
    
        char*    buf;             // allocated buffer
        size_t   len;             // length of buffer
        size_t   used;            // # chars actually used
        unsigned refs;            // reference count
    };
    
    class String {
    public:
        String();                 // start off empty
        ~String();                // decrement reference count
                                  //  (delete buffer if refs==0)
        String( const String& );  // point at same buffer and
                                  //  increment reference count
        void   Append( char );    // append one character
        size_t Length() const;    // string length
        char&  operator[](size_t);// element access
    private:
        void AboutToModify( size_t n, bool bUnshareable = false );
                                  // lazy copy, ensure len>=n
                                  //  and mark if unshareable
        static size_t Unshareable;// ref-count flag for "unshareable"
        StringBuf* data_;
    };
    
    
    StringBuf::StringBuf() : buf(0), len(0), used(0), refs(1) { }
    
    StringBuf::~StringBuf() { delete[] buf; }
    
    StringBuf::StringBuf( const StringBuf& other, size_t n )
      : buf(0), len(0), used(0), refs(1)
    {
        Reserve( max( other.len, n ) );
        copy( other.buf, other.buf+other.used, buf );
        used = other.used;
    }
    
    void StringBuf::Reserve( size_t n ) {
        if( len < n ) {
            //  Same growth code as in GotW #43, except now we round
            //  the new size up to the nearest multiple of 4 bytes.
            size_t needed = max<size_t>( len*1.5, n );
            size_t newlen = needed ? 4 * ((needed-1)/4 + 1) : 0;
            char*  newbuf = newlen ? new char[ newlen ] : 0;
            if( buf ) {
              copy( buf, buf+used, newbuf );
            }
            
            delete[] buf;   // now all the real work is
            buf = newbuf;   //  done, so take ownership
            len = newlen;
        }
    }
    
    
    size_t String::Unshareable = numeric_limits<unsigned>::max();
    
    String::String() : data_(new StringBuf) { }
    
    String::~String() {
        if( data_->refs == Unshareable || --data_->refs < 1 ) {
            delete data_;
        }
    }
    
    String::String( const String& other )
    {
        //  If possible, use copy-on-write.
        //  Otherwise, take a deep copy immediately.
        //
        if( other.data_->refs != Unshareable ) {
            data_ = other.data_;
            ++data_->refs;
        } else {
            data_ = new StringBuf( *other.data_ );
        }
    }
    
    void String::AboutToModify(size_t n, 
                               bool bMarkUnshareable /* = false */) 
    {
        if( data_->refs > 1 && data_->refs != Unshareable ) 
        {
            StringBuf* newdata = new StringBuf( *data_, n );
            --data_->refs;   // now all the real work is
            data_ = newdata; //  done, so take ownership
        } 
        else {
            data_->Reserve( n );
        }
        data_->refs = bMarkUnshareable ? Unshareable : 1;
    }
    
    void String::Append( char c ) {
        AboutToModify( data_->used+1 );
        data_->buf[data_->used++] = c;
    }
    
    size_t String::Length() const {
        return data_->used;
    }
    
    char& String::operator[]( size_t n ) {
        AboutToModify( data_->len, true );
        return *(data_->buf+n);
    }
}
```


# 045 Reference Counting - Part III
**Difficulty: 9 / 10**
In this final chapter of the miniseries, we consider the effects of thread safety on reference-counted strings. Is reference counting really an optimization? The answer will likely surprise you.

----

**Problem**
**JG Question**
**1.** Why is Optimized::String not thread-safe? Give examples.

**Guru Questions**
**2.** Demonstrate how to make Optimized::String (from GotW #44) thread-safe:

    a) assuming that there atomic operations to get, set, and compare integer values; and
    
    b) assuming that there aren't.

**3.** What are the effects on performance? Discuss.

----

**Solution**
**Introduction**
Standard C++ is silent on the subject of threads. Unlike Java, C++ has no built-in support for threads, and does not attempt to address thread safety issues through the language. So why a GotW issue on threads? Simply because more and more of us are writing multithreaded (MT) programs, and no discussion of reference-counted String implementations is complete if it does not cover thread safety issues.

I won't go into a long discussion about what's involved with writing thread-safe code in detail; see a good book on threads for more details.

**1.** Why is Optimized::String not thread-safe? Give examples.

It is the responsibility of code that uses an object to ensure that access to the object is serialized as needed. For example, if a certain String object could be modified by two different threads, it's not the poor String object's job to defend itself from abuse; it's the job of the code that uses the String to make sure that two threads never try to modify the same String object at the same time.

The problem with the code in GotW #44 is twofold: First, the reference-counted (copy-on-write, COW) implementation is designed to hide the fact that two different visible String objects could be sharing a common hidden state; hence it's the String class's responsibility to ensure that the calling code never modifies a String whose representation is shared. The String code shown already does that, by performing a deep copy ("un-sharing" the representation) if necessary the moment a caller tries to modify the String. This part is fine in general.

Unfortunately, it now brings us to the second problem: The code in String that "un-shares" the representation isn't thread-safe. Imagine that there are two String objects s1 and s2:

    a) s1 and s2 happen to share the same representation under the covers (okay, because String is designd for this);
    
    b) thread 1 tries to modify s1 (okay, because thread 1 knows that no one else is trying to modify s1);
    
    c) thread 2 tries to modify s2 (okay, because thread 2 knows that no one else is trying to modify s2);
    
    d) at the same time (error)

The problem is (d): At the same time, both s1 and s2 will attempt to "un-share" their shared representation, and the code to do that is is not thread-safe. Specifically, consider the very first line of code in String::AboutToModify():

``` cpp
void String::AboutToModify(
    size_t n,
    bool   bMarkUnshareable /* = false */
) {
    if( data_->refs > 1 && data_->refs != Unshareable ) {
       /* ... etc. ... */
```

This if-condition is not thread-safe. For one thing, evaluating even "`data_->refs > 1`" may not be atomic; if so, it's possible that if `thread 1` tries to evaluate "`data_->refs > 1`" while `thread 2` is updating the value of refs, the value read from `data_->refs` might be anything \-\- 1, 2, or even something that's neither the original value nor the new value. The problem is that String isn't following the basic thread-safety requirement that code that uses an object must ensure that access to the object is serialized as needed. Here, String has to ensure that no two threads are going to use the same "refs" value in a dangerous way at the same time. The traditional way to accomplish this is to serialize access to the shared StringBuf (or just its .refs member) using a critical section or semaphore. In this case, at minimum the "comparing an int" operation must be guaranteed to be atomic.

This brings us to the second issue: Even if reading and updating "refs" were atomic, there are two parts to the if condition. The problem is that the thread executing the above code could be interrupted after it evaluates the first part of the condition but before it evaluates the second part. In our example:

```
        Thread 1                      |            hread 2
                                      |
>> enter s1's AboutToModify()         |
                                      |
evaluate "data_->refs > 1"            |
(true, because data_->refs is 2)      |
                                      |
                      ******** context switch ********
                                      |
                                      |    >> enter s2's AboutToModify()
                                      |    (runs all the way to completion,
                                      |    including that it decrements
                                      |    data_->refs to the value 1)
                                      |    
                                      |    << exit s2's AboutToModify()
                                      |        
                      ******** context switch ********
                                      |
evaluate "data_->refs != Unshareable" |
(true, because data_->refs is now 1)  | 
```

enters AboutToModify's "I'm shared and need to unshare" block, which clones the representation, decrements data_->refs to the value 0, and gives up the last pointer to the StringBuf... poof, we have a memory leak because the StringBuf that had been shared by s1 and s2 can now never be deleted


Having covered that, we're ready to see how to solve these safety problems.

**2.** Demonstrate how to make Optimized::String (from GotW #44) thread-safe:

    a) assuming that there atomic operations to get, set, and compare integer values; and
    
    b) assuming that there aren't.

I'm going to answer b) first because it's more general. What's needed here is a lock-management device like a critical section or a mutex. The code is equivalent in either case, so below I'll use a critical section, which is usually a more efficient synchronization device than a general-purpose mutex. The key to what we're about to do is quite simple: It turns out that if we do things right we only need to lock access to the reference count itself.

Before doing anything else, we need to add a CriticalSection member object into Optimized::StringBuf. Let's call the member cs:

``` cpp
namespace Optimized {
    struct StringBuf {
        StringBuf();              // start off empty
       ~StringBuf();              // delete the buffer
        StringBuf( const StringBuf& other, size_t n = 0 );
                                  // initialize to copy of other,
                                  //  and ensure len >= n
    
        void Reserve( size_t n ); // ensure len >= n
    
        char*    buf;             // allocated buffer
        size_t   len;             // length of buffer
        size_t   used;            // # chars actually used
        unsigned refs;            // reference count
        CriticalSection cs;       // serialize work on this object
    };
```

The only function that necessarily operates on two StringBuf objects at once is the copy constructor. String only calls StringBuf's copy constructor from two places (from String's own copy constructor, and from AboutToModify()). Note that String only needs to serialize access to the reference count, because by definition no String will do any work on a StringBuf that's shared (if it is shared, it will be read in order to take a copy, but we don't need to worry about anyone else trying to change or Reserve() or otherwise alter/move the buffer).

The default constructor needs no locks:

``` cpp
    String::String() : data_(new StringBuf) { }
```

The destructor need only lock its interrogation and update of the refs count:

``` cpp
    String::~String() {
      bool bDelete = false;
      data_->cs.Lock(); //---------------------------
      if( data_->refs == Unshareable || --data_->refs < 1 ) {
        bDelete = true;
      }
      data_->cs.Unlock(); //-------------------------
      if( bDelete ) {
        delete data_;
      }
    }
```

For the String copy constructor, note that we can assume that the other String's data buffer won't be modified or moved during this operation, since it's the responsibility of the caller to serialize access to visible objects. We must still, however, serialize access to the reference count itself, as we did above:

``` cpp
    String::String( const String& other )
    {
      bool bSharedIt = false;
      other.data_->cs.Lock(); //---------------------
      if( other.data_->refs != Unshareable ) {
        bSharedIt = true;
        data_ = other.data_;
        ++data_->refs;
      }
      other.data_->cs.Unlock(); //-------------------
    
      if( !bSharedIt ) {
        data_ = new StringBuf( *other.data_ );
      }
    }
```

So making the String copy constructor safe wasn't very hard at all. This brings us to AboutToModify(), which turns out to be very similar, but notice that this sample code actually acquires the lock during the entire deep copy operation... really, the lock is strictly only needed when looking at the refs value, and again when updating the refs value at the end, but I decided to lock the whole operation instead of getting slightly better concurrency by releasing the lock during the deep copy and then reacquiring it just to update refs:

``` cpp
    void String::AboutToModify(
      size_t n,
      bool   bMarkUnshareable /* = false */
    ) {
      data_->cs.Lock(); //---------------------------
      if( data_->refs > 1 && data_->refs != Unshareable ) {
        StringBuf* newdata = new StringBuf( *data_, n );
        --data_->refs;   // now all the real work is
        data_->cs.Unlock(); //-----------------------
        data_ = newdata; //  done, so take ownership
      }
      else {
        data_->cs.Unlock(); //-----------------------
        data_->Reserve( n );
      }
      data_->refs = bMarkUnshareable ? Unshareable : 1;
    }
```

None of the other functions need to be changed. Append() and operator[]() don't need locks because once AboutToModify() completes we're guaranteed that we're not using a shared representation. Length() doesn't need a lock because by definition we're okay if our StringBuf is not shared (there's no one else to change the used count on us), and we're okay if it is shared (the other thread would take its own copy before doing any work and hence still wouldn't modify our used count on us):

``` cpp
    void String::Append( char c ) {
      AboutToModify( data_->used+1 );
      data_->buf[data_->used++] = c;
    }
    
    size_t String::Length() const {
      return data_->used;
    }
    
    char& String::operator[]( size_t n ) {
      AboutToModify( data_->len, true );
      return *(data_->buf+n);
    }

  }
```

Again, note the interesting thing in all of this: The only locking we needed to do involved the refs count itself.

With that observation and the above general-purpose solution under our belts, let's look back to the a) part of the question:

a) assuming that there atomic operations to get, set, and compare integer values; and

Some operating systems provide these kinds of functions.

NOTE: These functions are usually much more efficient than general-purpose synchronization primitives like critical sections and mutexes. It is, however, a fallacy so say that we can use atomic integer operations "instead of locks" because locking is still required \-\- the locking is just generally less expensive than other alternatives, but it's not free by a long shot.

Here is a thread-safe implementation of String that assumes we have three functions: an IntAtomicGet, and IntAtomicDecrement and IntAtomicIncrement that safely return the new value. We'll do essentially the same thing we did above, but use only atomic integer operations to serialize access to the refs count:
``` cpp
  namespace Optimized {

    String::String() : data_(new StringBuf) { }
    
    String::~String() {
      if( IntAtomicGet( data_->refs ) == Unshareable ||
          IntAtomicDecrement( data_->refs ) < 1 ) {
        delete data_;
      }
    }
    
    String::String( const String& other )
    {
      if( IntAtomicGet( other.data_->refs ) != Unshareable ) {
        data_ = other.data_;
        IntAtomicIncrement( data_->refs );
      }
      else {
        data_ = new StringBuf( *other.data_ );
      }
    }
    
    void String::AboutToModify(
      size_t n,
      bool   bMarkUnshareable /* = false */
    ) {
      int refs = IntAtomicGet( data_->refs );
      if( refs > 1 && refs != Unshareable ) {
        StringBuf* newdata = new StringBuf( *data_, n );
        if( IntAtomicDecrement( data_->refs ) < 1 ) {
          delete newdata;  // just in case two threads
        }                  //  are trying this at once
        else {             // now all the real work is
          data_ = newdata; //  done, so take ownership
        }
      }
      else {
        data_->Reserve( n );
      }
      data_->refs = bMarkUnshareable ? Unshareable : 1;
    }
    
    void String::Append( char c ) {
      AboutToModify( data_->used+1 );
      data_->buf[data_->used++] = c;
    }
    
    size_t String::Length() const {
      return data_->used;
    }
    
    char& String::operator[]( size_t n ) {
      AboutToModify( data_->len, true );
      return *(data_->buf+n);
    }

  }
```

3. What are the effects on performance? Discuss.

Without atomic integer operations, copy-on-write typically incurs a significant performance penalty. Even with atomic integer operations, COW can make common String operations take nearly 50% longer \-\- even in single-threaded programs.

In general, copy-on-write is often a bad idea in multithread-ready code. In short, the reason is that the calling code can no longer know whether two distinct String objects actually share the same representation under the covers, and so String must incur overhead to do enough serialization that calling code can take its normal share of responsibility for thread safety. Depending on the availability of more-efficient options like atomic integer operations, the impact on performance ranges from "moderate" to "profound."

SOME EMPIRICAL RESULTS
In this test environment I tested six main flavours of string implementations:

| Name | Description |
| ---- | ---- |
| Plain | Non-use-counted string; all others are modeled on this (a refined version of the GotW #43 answer)|
|COW_Unsafe|  Plain + COW, not thread-safe (a refined version of the GotW #44 answer)|
|COW_AtomicInt|  Plain + COW + thread-safe (a refined version of this GotW #45 1(a) answer above)|
|COW_AtomicInt2|  COW_AtomicInt + StringBuf in same buffer as the data (another refined version of this GotW #45 #1(a) above)|
|COW_CritSec|  Plain + COW + thread-safe (Win32 critical sections) (a refined version of this GotW #45 #1(b) answer above)|
|COW_Mutex|  Plain + COW + thread-safe (Win32 mutexes) (COW_CritSec with mutexes instead of critical sections)|

I also threw in a seventh flavour to measure the result of optimizing memory allocation instead of optimizing copying:

  Plain_FastAlloc  Plain + an optimized memory allocator
I focused on comparing Plain with COW_AtomicInt. COW_AtomicInt was generally the most efficient thread-safe COW implementation. The results were as follows:

1. For all mutating and possibly-mutating operations, COW_AtomicInt was always worse than Plain. This is natural and expected.

2. COW should shine when there are many unmodified copies, but for an average string length of 50:

a) When 33% of all copies were never modified, and the rest were modified only once each, COW_AtomicInt was still slower than Plain.

b) When 50% of all copies were never modified, and the rest were modified only thrice each, COW_AtomicInt was still slower than Plain.

This result may be more surprising to many \-\- particularly that COW_AtomicInt is slower in cases where there are more copy operations than mutating operations in the entire system!

Note that, in both cases, traditional thread-unsafe COW did perform better than Plain. This shows that indeed COW can be an optimization for purely single- threaded environments, but it is less often appropriate for thread-safe code.

3. It is a myth that COW's principal advantage lies in avoiding memory allocations. Especially for longer strings, COW's principal advantage is that it avoids copying the characters in the string.

4. Optimized allocation, not COW, was a consistent true speed optimization in all cases (but note that it does trade off space). Here is perhaps the most important conclusion from the Detailed Measurements section:

"\* Most of COW's primary advantage for small strings could be gained without COW by using a more efficient allocator. (Of course, you could also do both \-\- use COW and an efficient allocator.)"

Q: Why measure something as inefficient as COW_CritSec? A: Because at least one popular commercial basic_string implementation used this method as recently as a year ago (and perhaps still does, I haven't seen their code lately), despite the fact that COW_CritSec is nearly always a pessimization. Be sure to check whether your library vendor is doing this, because if the library is built for possible multithreaded use then you will bear the performance cost all the time \-\- even if your program is single-threaded.

Q: What's COW_AtomicInt2? A: It's the same as COW_AtomicInt except that, instead of allocating a StringBuf and then separately allocating the data buffer, the StringBuf and data are in the same allocated block of memory. Note that all other `COW_*` variants use a fast allocator for the StringBuf object (so that there's no unfair "double allocation" cost), and the purpose of COW_AtomicInt2 is mainly to demonstrate that I have actually addressed that concern... COW_AtomicInt2 is actually slightly slower than COW_AtomicInt for most operations (because of the extra logic).

I also tested the relative performance of various integer operations (incrementing int, incrementing volatile int, and incrementing int using the Win32 atomic integer operations), to ensure that COW_AtomicInt results were not unfairly skewed by poor implementations or function call overhead.

APPROACH
To assess COW, I performed measurements of three kinds of functions:

- copying (where COW shines, its raison d'etre)

- mutating operations that could trigger reallocation (represented by Append, which gradually lengthens; this is to make sure any conclusions drawn can take into account periodic reallocation overhead due to normal string use)

- possibly-mutating operations that do not change length enough to trigger reallocation, or that do not actually mutate the string at all (represented by operator[])

It turns out that the last two both incur a constant (and similar, within ~20%) cost per operation, and can be roughly considered together. Assuming for simplicity that mutating-and-extending operations like Append (235ms overhead) and possibly-mutating operations like operator[] (280ms overhead) will be about equally frequent, the COW_AtomicInt overhead for mutating and possibly-mutating operations is about 260ms per 1,000,000 operations in this implementation.

Finally, for each of 2(a) and 2(b), I first used the "Raw Measurements" section below to hand-calculate a rough prediction of expected relative performance, then ran the test to check actual performance.

``` assembly
SUMMARY FOR CASE 2(a):

    PREDICTION
    
      COW_AtomicInt Cost         Plain Cost
      -------------------------  ----------------------
      1M shallow copies          1M deep copies
       and dtors            400   and dtors        1600
      667K mutations        ???                     ???
      667K deep copies     1060
      extra overhead on
       667K deep copies     ???
      extra overhead on
       667K mutations       175
                          -----                   -----
                           1635+                   1600+
    
    TEST
        (program that makes copies in a tight loop, and
         modifies 33% of them with a single Append and
         another 33% of them with a single op[])
    
      Running 1000000 iterations with strings of length 50:
        Plain_FastAlloc    642ms  copies: 1000000  allocs: 1000007
                  Plain   1726ms  copies: 1000000  allocs: 1000007
             COW_Unsafe   1629ms  copies: 1000000  allocs:  666682
          COW_AtomicInt   1949ms  copies: 1000000  allocs:  666682
         COW_AtomicInt2   1925ms  copies: 1000000  allocs:  666683
            COW_CritSec  10660ms  copies: 1000000  allocs:  666682
              COW_Mutex  33298ms  copies: 1000000  allocs:  666682

SUMMARY FOR CASE 2(b):

    PREDICTION
    
      COW_AtomicInt Cost         Plain Cost
      -------------------------  ----------------------
      1M shallow copies          1M deep copies
       and dtors            400   and dtors        1600
      1.5M mutations        ???                     ???
      500K deep copies      800
      extra overhead on
       500K deep copies     ???
      extra overhead on
       1.5M mutations       390
                          -----                   -----
                           1590+                   1600+
    
    TEST
        (program that makes copies in a tight loop, and
         modifies 25% of them with three Appends and
         another 25% of them with three operator[]s)
    
      Running 1000000 iterations with strings of length 50:
        Plain_FastAlloc    683ms  copies: 1000000  allocs: 1000007
                  Plain   1735ms  copies: 1000000  allocs: 1000007
             COW_Unsafe   1407ms  copies: 1000000  allocs:  500007
          COW_AtomicInt   1838ms  copies: 1000000  allocs:  500007
         COW_AtomicInt2   1700ms  copies: 1000000  allocs:  500008
            COW_CritSec   8507ms  copies: 1000000  allocs:  500007
              COW_Mutex  31911ms  copies: 1000000  allocs:  500007
```

RAW MEASUREMENTS

``` assembly
TESTING CONST COPYING + DESTRUCTION: The target case of COW

  Notes:
   - COW_AtomicInt always took over twice as long to create and
      destroy a const copy as did plain thread-unsafe COW.
   - For every copy of a string that was later modified,
      COW_AtomicInt added constant unrecoverable overhead
      (400ms per 1,000,000) not counting the overhead on other
      operations.
   * Most of COW's primary advantage for small strings could be
      gained without COW by using a more efficient allocator.
      (Of course, you could also do both -- use COW and an
      efficient allocator.)
   * COW's primary advantage for large strings lay, not in
      avoiding the allocations, but in avoiding the char copying.

Running 1000000 iterations with strings of length 10:
  Plain_FastAlloc    495ms  copies: 1000000  allocs: 1000003
            Plain   1368ms  copies: 1000000  allocs: 1000003
       COW_Unsafe    160ms  copies: 1000000  allocs:       3
    COW_AtomicInt    393ms  copies: 1000000  allocs:       3
   COW_AtomicInt2    433ms  copies: 1000000  allocs:       4
      COW_CritSec    428ms  copies: 1000000  allocs:       3
        COW_Mutex  14369ms  copies: 1000000  allocs:       3

Running 1000000 iterations with strings of length 50:
  Plain_FastAlloc    558ms  copies: 1000000  allocs: 1000007
            Plain   1598ms  copies: 1000000  allocs: 1000007
       COW_Unsafe    165ms  copies: 1000000  allocs:       7
    COW_AtomicInt    394ms  copies: 1000000  allocs:       7
   COW_AtomicInt2    412ms  copies: 1000000  allocs:       8
      COW_CritSec    433ms  copies: 1000000  allocs:       7
        COW_Mutex  14130ms  copies: 1000000  allocs:       7

Running 1000000 iterations with strings of length 100:
  Plain_FastAlloc    708ms  copies: 1000000  allocs: 1000008
            Plain   1884ms  copies: 1000000  allocs: 1000008
       COW_Unsafe    171ms  copies: 1000000  allocs:       8
    COW_AtomicInt    391ms  copies: 1000000  allocs:       8
   COW_AtomicInt2    412ms  copies: 1000000  allocs:       9
      COW_CritSec    439ms  copies: 1000000  allocs:       8
        COW_Mutex  14129ms  copies: 1000000  allocs:       8

Running 1000000 iterations with strings of length 250:
  Plain_FastAlloc   1164ms  copies: 1000000  allocs: 1000011
            Plain   5721ms  copies: 1000000  allocs: 1000011 [*]
       COW_Unsafe    176ms  copies: 1000000  allocs:      11
    COW_AtomicInt    393ms  copies: 1000000  allocs:      11
   COW_AtomicInt2    419ms  copies: 1000000  allocs:      12
      COW_CritSec    443ms  copies: 1000000  allocs:      11
        COW_Mutex  14118ms  copies: 1000000  allocs:      11

Running 1000000 iterations with strings of length 1000:
  Plain_FastAlloc   2865ms  copies: 1000000  allocs: 1000014
            Plain   4945ms  copies: 1000000  allocs: 1000014
       COW_Unsafe    173ms  copies: 1000000  allocs:      14
    COW_AtomicInt    390ms  copies: 1000000  allocs:      14
   COW_AtomicInt2    415ms  copies: 1000000  allocs:      15
      COW_CritSec    439ms  copies: 1000000  allocs:      14
        COW_Mutex  14059ms  copies: 1000000  allocs:      14

Running 1000000 iterations with strings of length 2500:
  Plain_FastAlloc   6244ms  copies: 1000000  allocs: 1000016
            Plain   8343ms  copies: 1000000  allocs: 1000016
       COW_Unsafe    174ms  copies: 1000000  allocs:      16
    COW_AtomicInt    397ms  copies: 1000000  allocs:      16
   COW_AtomicInt2    413ms  copies: 1000000  allocs:      17
      COW_CritSec    446ms  copies: 1000000  allocs:      16
        COW_Mutex  14070ms  copies: 1000000  allocs:      16



TESTING APPEND: An always-mutating periodically-reallocating operation

  Notes:
   - Plain always outperformed COW.
   - The overhead of COW_AtomicInt compared to Plain did not
      vary greatly with string lengths: It was fairly constant
      at about 235ms per 1,000,000 operations.
   - The overhead of COW_AtomicInt compared to COW_Unsafe did not
      vary greatly with string lengths: It was fairly constant
      at about 110ms per 1,000,000 operations.
   * The overall ever-better performance for longer strings was
      due to the allocation strategy (see GotW #43), not COW vs.
      Plain issues.

Running 1000000 iterations with strings of length 10:
  Plain_FastAlloc    302ms  copies:       0  allocs:  272730
            Plain    565ms  copies:       0  allocs:  272730
       COW_Unsafe    683ms  copies:       0  allocs:  272730
    COW_AtomicInt    804ms  copies:       0  allocs:  272730
   COW_AtomicInt2    844ms  copies:       0  allocs:  363640
      COW_CritSec    825ms  copies:       0  allocs:  272730
        COW_Mutex   8419ms  copies:       0  allocs:  272730

Running 1000000 iterations with strings of length 50:
  Plain_FastAlloc    218ms  copies:       0  allocs:  137262
            Plain    354ms  copies:       0  allocs:  137262
       COW_Unsafe    474ms  copies:       0  allocs:  137262
    COW_AtomicInt    588ms  copies:       0  allocs:  137262
   COW_AtomicInt2    536ms  copies:       0  allocs:  156871
      COW_CritSec    607ms  copies:       0  allocs:  137262
        COW_Mutex   7614ms  copies:       0  allocs:  137262

Running 1000000 iterations with strings of length 100:
  Plain_FastAlloc    182ms  copies:       0  allocs:   79216
            Plain    257ms  copies:       0  allocs:   79216
       COW_Unsafe    382ms  copies:       0  allocs:   79216
    COW_AtomicInt    492ms  copies:       0  allocs:   79216
   COW_AtomicInt2    420ms  copies:       0  allocs:   89118
      COW_CritSec    535ms  copies:       0  allocs:   79216
        COW_Mutex   7810ms  copies:       0  allocs:   79216

Running 1000000 iterations with strings of length 250:
  Plain_FastAlloc    152ms  copies:       0  allocs:   43839
            Plain    210ms  copies:       0  allocs:   43839
       COW_Unsafe    331ms  copies:       0  allocs:   43839
    COW_AtomicInt    438ms  copies:       0  allocs:   43839
   COW_AtomicInt2    366ms  copies:       0  allocs:   47825
      COW_CritSec    485ms  copies:       0  allocs:   43839
        COW_Mutex   7358ms  copies:       0  allocs:   43839

Running 1000000 iterations with strings of length 1000:
  Plain_FastAlloc    123ms  copies:       0  allocs:   14000
            Plain    149ms  copies:       0  allocs:   14000
       COW_Unsafe    275ms  copies:       0  allocs:   14000
    COW_AtomicInt    384ms  copies:       0  allocs:   14000
   COW_AtomicInt2    299ms  copies:       0  allocs:   15000
      COW_CritSec    421ms  copies:       0  allocs:   14000
        COW_Mutex   7265ms  copies:       0  allocs:   14000

Running 1000000 iterations with strings of length 2500:
  Plain_FastAlloc    122ms  copies:       0  allocs:    6416
            Plain    148ms  copies:       0  allocs:    6416
       COW_Unsafe    279ms  copies:       0  allocs:    6416
    COW_AtomicInt    380ms  copies:       0  allocs:    6416
   COW_AtomicInt2    304ms  copies:       0  allocs:    6817
      COW_CritSec    405ms  copies:       0  allocs:    6416
        COW_Mutex   7281ms  copies:       0  allocs:    6416



TESTING OPERATOR[]: A possibly-mutating operation, never does mutate

  Notes:
   - Plain always vastly outperformed COW.
   - Results were independent of string lengths.
   - The overhead of COW_AtomicInt compared to Plain was
      constant at about 280ms per 1,000,000 operations.
   - COW_AtomicInt2 fared better in this test case, but
      COW_AtomicInt did better overall and so I am focusing
      on comparing that with Plain.

[10x iterations] Running 10000000 iterations with strings of length 10:
  Plain_FastAlloc      3ms  copies:       0  allocs:       3 [*]
            Plain      2ms  copies:       0  allocs:       3 [*]
       COW_Unsafe   1698ms  copies:       0  allocs:       3
    COW_AtomicInt   2833ms  copies:       0  allocs:       3
   COW_AtomicInt2   2112ms  copies:       0  allocs:       4
      COW_CritSec   3587ms  copies:       0  allocs:       3
        COW_Mutex  71787ms  copies:       0  allocs:       3

   [*] within measurement error margin, both varied from 0ms to 9ms



TESTING VARIOUS INTEGER INCREMENT/DECREMENT OPERATIONS

  Test Summary:
   - "plain" performs the operations on normal
      nonvolatile ints
   - "volatile" is the only case to use volatile ints
   - "atomic" uses the Win32 InterlockedXxx operations
   - "atomic_ass" uses inline x86 assembler locked
      integer operations

  Notes:
   - ++atomic took only three times as long as either
      ++volatile and unoptimized ++plain
   - ++atomic does not incur function call overhead

[100x iterations] Running 100000000 iterations for integer operations:

          ++plain   2404ms, counter=100000000
          --plain   2399ms, counter=0
    
       ++volatile   2400ms, counter=100000000
       --volatile   2405ms, counter=0
    
         ++atomic   7480ms, counter=100000000
         --atomic   9340ms, counter=0
    
     ++atomic_ass   8881ms, counter=100000000
     --atomic_ass  10964ms, counter=0
```

Here are a few extra notes on the relative timings of various flavours of x86 assembler implementations of IntAtomicIncrement (these timings were taken under the same conditions as above and can be compared directly):

``` assembly
    Instructions                    Timing
    ---------------------------     --------
    __asm mov       eax, 1
    __asm lock xadd i, eax
    __asm mov       result, eax     ~11000ms
    
    __asm mov       eax, 1
    __asm lock xadd i, eax          ~10400ms
    
    __asm lock inc i                 ~8900ms
      (this is the one actually used above)
```

Note that the non-atomic versions are much better, and map directly onto the "plain" timings:

``` assembly
    __asm inc i                      ~2400ms
```
Conclusion: So there is indeed overhead introduced by the x86 LOCK instruction, even on a single-CPU machine. This is natural and to be expected, but I point it out because some people said there was no difference.

I am very impressed that Win32's InterlockedIncrement is actually faster at 765ms than my hand-coded assembler at 900ms, even though my hand-coded version actually does less work (only a single instruction!) because it doesn't return the new value. Of course, I'm no x86 assembler expert; the explanation is certainly that the OS's wrapper is using a different opcode than my hand-coded version.

Finally, of course, note that the Win32 atomic int functions clearly are not incurring function-call overhead. Never assume \-\- measure.

A few important points about this test harness:

1. CAVEAT LECTOR: Take this for what it is: A first cut at a test harness. Comments and corrections are welcome. I'm showing raw performance numbers here; I haven't inspected the compiled code, and I've made no attempt to determine the impact of cache hits/misses and other secondary effects. (Even so, this GotW took much more effort than usual to produce, and I guarantee that the next few issues will feature simpler topics!)

2. TANSTAAFL ("there ain't no such thing as a free lunch" -R.A.Heinlein). Even with atomic integer operations, it is incorrect to say "there's no locking required" because the atomic integer operations clearly do perform serialization, and do incur significant overhead.

3. The test harness itself is SINGLE-threaded. A thread-safe COW implementation incurs overhead even in programs that are not themselves multithreaded. At best, COW could be a true optimization only when the COW code does not need to be made thread-safe at all (even then, see Rob Murray's "C++ Strategies and Tactics" book, pages 70-72, for more empirical tests that show COW is only beneficial in certain situations). If thread safety is required, COW imposes a significant performance penalty on all users, even users running only single-threaded code.

[Click here to download the test harness source code.]: http://www.gotw.ca/gotw/045code.zip



# 046 Typedefs
**Difficulty: 3 / 10**
Why use typedef? Besides the many traditional reasons, we'll consider typedef techniques that make using the C++ standard library safer and easier.

----

**Problem**
**JG Question**
**1.** What does typedef do?

**Guru Questions**
**2.** Why use typedef? Name as many situations/reasons as you can.

**3.** Why is typedef such a good idea in code that uses standard (STL) containers?

----

**Solution**
**1.** What does typedef do?

Writing "typedef" allows you to assign another equivalent name for a type. For example:

``` cpp
typedef vector< vector<int> > IntArray;
```

lets you write the simpler "IntArray" in place of the more verbose "`vector< vector<int> >`".

**2.** Why use typedef? Name as many situations/reasons as you can.

Here are several "abilities":

**Typeability**
Shorter names are easier to type.

**Readability**
Typedefs can make code, especially long template type names, much more readable. For a simple example, consider the following from a public newsgroup posting asking what this code meant:
``` cpp
int ( *t(int) )( int* );
```

If you're used to reading C declarations just like you'd read a Victorian English novel (i.e., dense and verbose prose that at times feels like slogging through ankle-deep sucking muck), you know the answer and you're fine. If you're not, typedefs really help, even with as meaningless a typedef name as "`Func`":
``` cpp
typedef int (*Func)( int* );
Func t( int );
```

Now it's clearer that this is a function declaration, for a function named "t" that takes an int and returns a pointer to a function that takes an `int*` and returns an int. (Say that three times fast.) In this case, the typedef is easier to read than the English.

Typedefs can also add semantic meaning. For example, "PhoneBook" is much easier to understand than "map< string, string>" (which could mean anything!).

**Portability**
If you use typedef'd names for platform-specific or otherwise nonportable names, you'll find it easier to move to new platforms. After all, it's easier to write this:
``` cpp
#if defined USING_COMPILER_A
    typedef __int32 Int32;
    typedef __int64 Int64;
#elif defined USING_COMPILER_B
    typedef int       Int32;
    typedef long long Int64;
#endif
```

than search-and-replace for one of the system-specific names throughout your code. The typedef names insulate you from simple platform dependencies.

**3.** Why is typedef such a good idea in code that uses standard (STL) containers?

**Flexibility**
Changing a typedef name in one place is easier than changing all of its uses throughout the code. For example, consider the following code:
``` cpp
void f( vector<Customer>& vc ) {
    vector<Customer>::iterator i = vc.begin();
    ...
}
```

What if a few months later you find that vector isn't the right container? If you're storing huge numbers of Customer objects, the fact that vector's storage is contiguous[1] may be a disadvantage and you'd like to switch to deque instead. Or, if you're frequently inserting/removing elements from the middle, you'd like to switch to list instead.

In the above code, you'd have to make that change everywhere "`vector<Customer>`" appears. How much easier it would be if you had only written:
``` cpp
typedef vector<Customer> Customers;

...

void f( Customers& vc ) {
    Customers::iterator i = vc.begin();
    ...
}
```

and only needed to change the typedef to `list<Customer>` or `deque<Customer>`! It's not always this easy \-\- for example, your code might be relying on Customers::iterator being a random-access iterator, which a `list<Customer::iterator` isn't \-\- but this does insulate you from a lot of otherwise-tedious changes.

**Traitability**
The traits idiom is a powerful way to associate information with a type, and if you want to customize standard containers or algorithms you'll often need to provide traits. Consider the case-insensitive string example from GotW #29, where we defined our own char_traits replacement.

Most uses of typedef fall into one of these categories.

**Notes**
1. Yes, I'm aware of the debate, and yes, it should be contiguous. (Later note: See also [Standard Library News, Part 1: Vectors and Deques](http://www.gotw.ca/publications/mill10.htm).)



# 047 Uncaught Exceptions
**Difficulty: 6 / 10**
What is the standard function `uncaught_exception()`, and when should it be used? The answer given here isn't one that most people would expect.

----

**Problem**
**JG Question**
**1.** What does std::uncaught_exception() do?

**Guru Questions**
**2.** Consider the following code:

``` cpp
T::~T() {
    if( !std::uncaught_exception() ) {
        // ... code that could throw ...
    } else {
        // ... code that won't throw ...
    }
}
```

Is this a good technique? Present arguments for and against.

**3.** Is there any other good use for uncaught_exception? Discuss and draw conclusions.

----

**Solution**
**1.** What does std::uncaught_exception() do?

It provides a way of knowing whether there is an exception currently active. (Note that this is not the same thing as knowing whether it is safe to throw an exception.)

To quote directly from the standard (15.5.3/1):

The function bool uncaught_exception() returns true after completing evaluation of the object to be thrown until completing the initialization of the exception-declaration in the matching handler (_lib.uncaught_). This includes stack unwinding. If the exception is rethrown (_except.throw_), uncaught_exception() returns true from the point of rethrow until the rethrown exception is caught again.

As it turns out, this specification is deceptively close to being useful.

**2.** Consider the following code:

``` cpp
T::~T() {
    if( !std::uncaught_exception() ) {
        // ... code that could throw ...
    } else {
        // ... code that won't throw ...
    }
}
```

Is this a good technique? Present arguments for and against.

In short: No, even though it attempts to solve a problem. There are technical grounds why it shouldn't be used (i.e., it doesn't always work), but I'm much more interested in arguing against this idiom on moral grounds.

**Background: The Problem**
If a destructor throws an exception, Bad Things can happen. Specifically, consider code like the following:

``` cpp
//  The problem

class X {
public:
    ~X() { throw 1; }
};

void f() {
    X x;
    throw 2;
} // calls X::~X (which throws), then calls terminate()
```

If a destructor throws an exception while another exception is already active (i.e., during stack unwinding), the program is terminated. This is usually not a good thing.

**The Wrong Solution**
"Aha," many people \-\- including many experts \-\- have said, "let's use uncaught_exception() to figure out whether we can throw or not!" And that's where the code in Question 2 comes from... it's an attempt to solve the illustrated problem:

``` cpp
//  The wrong solution

T::~T() {
    if( !std::uncaught_exception() ) {
        // ... code that could throw ...
    } else {
        // ... code that won't throw ...
    }
}
```

The idea is that "we'll use the path that could throw as long as it's safe to throw." This philosophy is wrong on two counts: first, this code doesn't do that; second (and more importantly), the philosophy itself is in error.

**The Wrong Solution: Why the Code Is Unsound**
One problem is that the above code won't actually work as expected in some situations. Consider:

``` cpp
//  Why the wrong solution is wrong

U::~U() {
    try {
        T t;
        // do work
    } catch( ... ) {
        // clean up
    }
}
```

If a U object is destroyed due to stack unwinding during to exception propagation, T::~T will fail to use the "code that could throw" path even though it safely could.

Note that none of this is materially different from the following:

``` cpp
//  Variant: Another wrong solution

Transaction::~Transaction() {
    if( uncaught_exception() ) {
        RollBack();
    }
}
```

Again, note that this doesn't do the right thing if a transaction is attempted in a destructor that might be called during stack unwinding:

``` cpp
//  Variant: Why the wrong solution is still wrong

U::~U() {
    try {
        Transaction t( /*...*/ );
        // do work
    } catch( ... ) {
        // clean up
    }
}
```

**The Wrong Solution: Why the Approach Is Immoral**
In my view, however, the "it doesn't work" problem isn't even the main issue here. My major problem with this solution is not technical, but moral: It is poor design to give T::~T() two different semantics, for the simple reason that it is always poor design to allow an operation to report the same error in two different ways. Not only does it complicate the interface and the semantics, but it makes the caller's life harder because the caller must be able to handle both flavours of error reporting \-\- and this when far too many programmers don't check errors well in the first place!

**The Right Solution**
The right answer to the problem is much simpler:

``` cpp
//  The right solution

T::~T() /* throw() */ {
    // ... code that won't throw ...
}
```

If necessary, T can provide a "pre-destructor" function (e.g., "T::Close()") which can throw and performs all shutdown of the T object and any resources that it owns. That way, the calling code can call T::Close() if it wants to detect hard errors, and T::~T() can be implemented in terms of T::Close() plus a try/catch block:

``` cpp
//  Alternative right solution

T::Close() {
    // ... code that could throw ...
}

T::~T() /* throw() */ {
    try {
        Close();
    } 
    catch( ... ) {
    }
}
```

(Later note: See GotW #66 as to why this try block is inside the destructor body, and must not be a destructor function try block.)

This nicely follows the principle of "one function, one responsibility"... a problem in the original code was that the same function was responsible for both destroying the object and final cleanup/reporting.

From the GotW coding standards:

    - destructors:
    
        - never throw an exception from a destructor (Meyers96: 58-61)
    
        - if a destructor calls a function that might throw, always wrap the call in a try/catch block that prevents the exception from escaping
    
        - prefer declaring destructors as "throw()"

If this GotW issue hasn't convinced you to follow this coding standard, then maybe my other articles on exception safety will. See Scott Meyers' soon-to-be-published CD of his "Effective C++" and "More Effective C++" books, which also includes [exception handling articles](http://www.awlonline.com/cseng/meyerscddemo/DEMO/MAGAZINE/SU_FRAME.HTM) that I and Jack Reeves have written. In my section, note particularly the part entitled "Destructors That Throw and Why They're Evil" to see why you can't even reliably new[] an array of objects whose destructors can throw. (If you keep back issues of C++ Report, you can find the same articles in the October 1997 and November/December 1997 issues.)

**3.** Is there any other good use for uncaught_exception? Discuss and draw conclusions.

Unfortunately, I do not know of any good and safe use for std::uncaught_exception. My advice: Don't use it.





# 048 Switching Streams
**Difficulty: 2 / 10**
What's the best way to switch between different stream sources and targets, including the standard console streams and files?



**Problem**
**JG Question**
**1.** What are the types of std::cin and std::cout?

**Guru Question**
**2.** Write an ECHO program that simply echoes its input and that can be invoked equivalently in two ways:

``` sh
ECHO <infile >outfile

ECHO infile outfile
```

In most popular command-line environments, the first command assumes that the program takes input from std::cin and sends output to std::cout. The second command tells the program to take its input from the file named "infile" and to produce output in the file named "outfile". The program should be able to support all of the above input/output options.



**Solution**
**1.** What are the types of std::cin and std::cout?

Short answer is that cin is:

``` cpp
std::basic_istream<char, std::char_traits<char> >
```

and cout is:

``` cpp
std::basic_ostream<char, std::char_traits<char> >
```

Longer answer: std::cin and std::cout have type std::istream and std::ostream, respectively. In turn, those are typdef'd as std::basic_istream<char> and std::basic_ostream<char>. Finally, after accounting for the default template arguments, we get the above.

Note: If you are using an older implementation of the iostreams subsystem, you might still see intermediate classes like istream_with_assign. Those classes do not appear in the final standard, and your implementation should eliminate them soon as it catches up, if it hasn't already.

**2.** Write an ECHO program that simply echoes its input and that can be invoked equivalently in two ways:

``` sh
ECHO <infile >outfile

ECHO infile outfile
```

In most popular command-line environments, the first command assumes that the program takes input from std::cin and sends output to std::cout. The second command tells the program to take its input from the file named "infile" and to produce output in the file named "outfile". The program should be able to support all of the above input/output options.

**Method 0: The Tersest Solution**
The tersest solution is a single statement:

``` cpp
#include <fstream>
#include <iostream>
using namespace std;

int main( int argc, char* argv[] ) {
    (argc>2
       ? ofstream(argv[2], ios::out | ios::binary)
       : cout)
    <<
    (argc>1
       ? ifstream(argv[1], ios::in | ios::binary)
       : cin)
    .rdbuf();
}
```

This works because basic_ios provides a convenient rdbuf() member function that returns the streambuf used inside a given stream object. Pretty much everything in the iostreams subsystem is derived from the basic_ios class template. In particular, that includes cin, cout, and the ifstream and ofstream templates.

**More Flexible Solutions**
Method 0 has two major drawbacks: First, the terseness is borderline, and extreme terseness is not suitable for production code. From the GotW coding standards:

- programming style:

- prefer readability:

- avoid writing terse (brief but difficult to understand and maintain) code; eschew obfuscation (Sutter97b)

Second, although Method 0 answers the immediate question, it's only good for when you want to copy the input verbatim. That may be enough today, but what if tomorrow you need to do other processing on the input, like converting it to upper case or calculating a total or removing every third character? That may well be a reasonable thing to want to do in the future, so it would be better right now to encapsulate the processing work in a separate function that can use the right kind of input or output object polymorphically:

``` cpp
#include <fstream>
#include <iostream>
using namespace std;

int main( int argc, char* argv[] ) {
    Process(
        (argc>1
           ? ifstream(argv[1], ios::in  | ios::binary)
           : cin ),
        (argc>2
           ? ofstream(argv[2], ios::out | ios::binary)
           : cout)
    );
}
```

But how do we implement Process()? In C++, there are two useful ways to express polymorphism:

**Method 1: Templates (Compile-Time Polymorphism)**
The first way is to use compile-time polymorphism using templates, which merely requires the passed objects to have a suitable interface (such as a member function named rdbuf):

``` cpp
template<class In, class Out>
void Process( In& in, Out& out ) {
  // ... do something more sophisticated,
  //     or just plain "out << in.rdbuf();"...
}
```

**Method 2: Virtual Functions (Run-Time Polymorphism)**
The second way is to use run-time polymorphism, which makes use of the fact that there is a common base class with a suitable interface:

``` cpp
//  Method 2(a): First attempt, sort of okay.
//
void Process( basic_istream<char>& in,
              basic_ostream<char>& out ) {
  // ... do something more sophisticated,
  //     or just plain "out << in.rdbuf();"...
}
```

(The parameters are not of type basic_ios<char>& because that wouldn't permit the use of operator<<.)

Of course, this depends on the input and output streams being derived from basic_istream<char> and basic_ostream<char>. That happens to be good enough for our example, but not all streams are based on plain chars or even on char_traits<char>. For example, wide character streams are based on wchar_t, and GotW #29 showed the usefulness of user-defined traits with different behaviour (ci_char_traits, for case insensitivity).

Even Method 2 ought to use templates, and let the compiler deduce the arguments appropriately:

``` cpp
//  Method 2(b): Better solution.
//
template<class C = char, class T = char_traits<C> >
void Process( basic_istream<C,T>& in,
              basic_ostream<C,T>& out ) {
  // ... do something more sophisticated,
  //     or just plain "out << in.rdbuf();"...
}
```

**Sound Engineering Principles**
All of these answers are "right" as far as they go, but in this situation I personally tend to prefer Method 1. This is because of two valuable guidelines:

    Guideline: Prefer extensibility.

Avoid writing code that only solves the immediate problem. Writing an extensible solution is almost always far better (just don't go overboard).

Balanced judgment is one hallmark of the experienced programmer. In particular, experienced programmers understand how to strike the right balance between writing special-purpose code that solves only the immediate problem (shortsighted, hard to extend) and writing a gradiose general framework to solve what should be a simple problem (rabid overdesign).

Method 1 is only slightly more complex than Method 0, but that extra complexity buys you better extensibility. It is at once both simpler and more flexible than Method 2; it is more adaptable to new situations because it avoids being hardwired to work with the iostreams hierarchy only.

So, prefer extensibility. Note that this is NOT an open license to go overboard and overdesign what ought to be a simple system. It is, however, encouragement to do more than just solve the immediate problem, when a little thought lets you discover that the problem you're solving is a special case of a more general problem. This is especially true because designing for extensibility often implicitly means designing for encapsulation:

    Guideline: Prefer encapsulation. Separate concerns.

As far as possible, one piece of code \-\- function, or class \-\- should know about and be responsible for one thing.

Arguably best of all, Method 1 exhibits good separation of concerns: The code that knows about the possible differences in input/output sources and sinks is separated from the code that knows how to actually do the work. This is a second hallmark of sound engineering.



# 049 Template Specialization and Overloading
**Difficulty: 6 / 10**
How do you specialize and overload templates? When you do, how do you determine which template gets called? Try your hand at analyzing these twelve examples.

----

**Problem**
**JG Questions**
**1.** What is template specialization? Give an example.

**2.** What is partial specialization? Give an example.

**Guru Question**
**3.** Consider the following declarations:

``` cpp
template<typename T1, typename T2>
void f( T1, T2 );                       // 1
template<typename T> void f( T );       // 2
template<typename T> void f( T, T );    // 3
template<typename T> void f( T* );      // 4
template<typename T> void f( T*, T );   // 5
template<typename T> void f( T, T* );   // 6
template<typename T> void f( int, T* ); // 7
template<> void f<int>( int );          // 8
void f( int, double );                  // 9
void f( int );                          // 10
```

Which of the above functions are invoked by each of the following? Be specific by identifying the template parameter types, where appropriate.

``` cpp
    int             i;
    double          d;
    float           ff;
    complex<double> c;

    f( i );         // a
    f<int>( i );    // b
    f( i, i );      // c
    f( c );         // d
    f( i, ff );     // e
    f( i, d );      // f
    f( c, &c );     // g
    f( i, &d );     // h
    f( &d, d );     // i
    f( &d );        // j
    f( d, &i );     // k
    f( &i, &i );    // l
```

----

**Solution**
Templates provide C++'s most powerful form of genericity. They allow you to write generic code that works with many kinds of unrelated objects; for example, strings that contain various kinds of characters, containers that can hold arbitrary types of objects, and algorithms that can operate on arbitrary types of sequences.

**1.** What is template specialization? Give an example.

Template specialization lets templates deal with special cases. Sometimes a generic algorithm can work much more efficiently for a certain kind of sequence (for example, when given random-access iterators), and so it makes sense to specialize it for that case while using the slower but more generic approach for all other cases. Performance is a common reason to specialize, but it's not the only one; for example, you might also specialize a template to work with certain objects that don't conform to the normal interface expected by the generic template.

These special cases can be handled using two forms of template specialization: explicit specialization, and partial specialization.

**Explicit Specialization**
Explicit specialization lets you write a specific implementation for a particular combination of template parameters. For example, given the function template:

``` cpp
template<class T> void sort(Array<T>& v) { /*...*/ };
```

If we have a faster (or other specialized) way we want to deal specifically with arrays of `char*`'s, we could explicitly specialize:

``` cpp
template<> void sort<char*>(Array<char*>&);
```

The compiler will then choose the most appropriate template:

``` cpp
    Array<int>   ai;
    Array<char*> apc;
    
    sort( ai );       // calls sort<int>
    sort( apc );      // calls specialized sort<char*>
```

**Partial Specialization**
**2.** What is partial specialization? Give an example.

For class templates only, you can define partial specializations that don't have to fix all of the primary (unspecialized) class template's parameters.

Here is an example from 14.5.4 [temp.class.spec]. The first template is the primary class template:

``` cpp
template<class T1, class T2, int I>
class A             { };             // #1
```

We can specialize this for the case when T2 is a T1*:

``` cpp
template<class T, int I>
class A<T, T*, I>   { };             // #2
```

Or for the case when T1 is any pointer:

``` cpp
template<class T1, class T2, int I>
class A<T1*, T2, I> { };             // #3
```

Or for the case when T1 is int and T2 is any pointer and I is 5:

``` cpp
template<class T>
class A<int, T*, 5> { };             // #4
```

Or for the case when T2 is any pointer:

``` cpp
template<class T1, class T2, int I>
class A<T1, T2*, I> { };             // #5
```

Declarations 2 to 5 declare partial specializations of the primary template. The compiler will then choose the appropriate template. From 14.5.4.1 [temp.class.spec.match]:

``` cpp
    A<int, int, 1>   a1;  // uses #1
    A<int, int*, 1>  a2;  // uses #2, T is int,
                          //          I is 1
    A<int, char*, 5> a3;  // uses #4, T is char
    A<int, char*, 1> a4;  // uses #5, T1 is int,
                          //          T2 is char,
                          //          I is 1
    A<int*, int*, 2> a5;  // ambiguous:
                          // matches #3 and #5
```

**Function Template Overloading**
Now let's consider function template overloading. It isn't the same thing as specialization, but it's related to specialization.

C++ lets you overload functions, yet makes sure the right one is called:

``` cpp
    int  f( int );
    long f( double );
    
    int    i;
    double d;
    
    f( i );   // calls f(int)
    f( d );   // calls f(double)
```

Similarly, you can also overload function templates, which brings us to the final question:

**3.** Consider the following declarations:

``` cpp
template<typename T1, typename T2>
void f( T1, T2 );                       // 1
template<typename T> void f( T );       // 2
template<typename T> void f( T, T );    // 3
template<typename T> void f( T* );      // 4
template<typename T> void f( T*, T );   // 5
template<typename T> void f( T, T* );   // 6
template<typename T> void f( int, T* ); // 7
template<> void f<int>( int );          // 8
void f( int, double );                  // 9
void f( int );                          // 10
```

First, let's simplify things a little by noticing that there are two groups of overloaded f's here: Those that take a single parameter, and those that take two parameters. [Note: I deliberately didn't muddy the waters by including an overload with two parameters where the second parameter had a default. Had there been such a function, then for the purposes of determining the correct ordering it should be considered in both lists: once as a single-parameter function (using the default), and once as a two-parameter funtion (not using the default).]

``` cpp
template<typename T1, typename T2>
void f( T1, T2 );                       // 1
template<typename T> void f( T, T );    // 3
template<typename T> void f( T*, T );   // 5
template<typename T> void f( T, T* );   // 6
template<typename T> void f( int, T* ); // 7
void f( int, double );                  // 9

template<typename T> void f( T );       // 2
template<typename T> void f( T* );      // 4
template<> void f<int>( int );          // 8
void f( int );                          // 10
```

Now let's consider each of the calls in turn:

Which of the above functions are invoked by each of the following? Be specific by identifying the template parameter types, where appropriate.

``` cpp
    int             i;
    double          d;
    float           ff;
    complex<double> c;
    
    f( i );         // a
```

A. This calls #10, because it's an exact match for #10 and such non-templates are always preferred over templates (see 13.3.3).

``` cpp
    f<int>( i );    // b
```

B. This calls #8, because `f<int>` is being explicitly requested.

``` cpp
    f( i, i );      // c
```

C. This calls #3 (T is int), because that is the best match.

``` cpp
    f( c );         // d
```

D. This calls #2 (T is `complex<double>`), because no other f can match.

``` cpp
    f( i, ff );     // e
```

E. This calls #1 (T1 is int, T2 is float). You might think that #9 is very close \-\- and it is \-\- but a nontemplate function is preferred only if it is an exact match.

``` cpp
    f( i, d );      // f
```

F. This one does call #9, because now #9 is an exact match and the nontemplate is preferred.

``` cpp
    f( c, &c );     // g
```

G. This calls #6 (T is complex<double>), because #6 is the closest overload. #6 provides an overload of f where the second parameter is a pointer to the same type as the first parameter.

``` cpp
    f( i, &d );     // h
```

H. This calls #7 (T is double), because #7 is the closest overload.

``` cpp
    f( &d, d );     // i
```

I. This calls #5 (T is double). #5 provides an overload of f where the first parameter is a pointer to the same type as the second parameter.

Only a few more to go...

``` cpp
    f( &d );        // j
```

J. Clearly (by now), we're asking for #4 (T is double).

``` cpp
    f( d, &i );     // k
```

K. Several other overloads are close, but only #1 fits the bill here (T1 is double, T2 is `int*`).

And finally...

``` cpp
    f( &i, &i );    // l
```

L. This calls #3 (T is `int*`), which is the closest overload even though some of the others explicitly mention a pointer parameter.

The Good News:

Compiler vendors will soon be supporting templates better, so that you can make use of features like the above more reliably and portably.

The Bad News:

If you got all of the above answers right, you probably know the rules better than your current compiler.

 



# 050 Using Standard Containers 
**Difficulty: 6 / 10**
Oil and water just don't mix. Do pointers and standard containers mix any better?

----

**Problem**
**JG Questions**
Consider the following code:

``` cpp
void f( vector<char>& v ) {
    char* pc = &v[0];
    // ... later, uses pc ...
}
```

**1.** Is this code valid?

**2.** Whether it's valid or not, how could it be improved?

Guru Question
**3.** Consider the following code:

``` cpp
template<class T = std::vector<T> >
void f( T& t ) {
    typename T::value_type* p1 = &t[0];
    typename T::value_type* p2 = &*t.begin();
    // ... later, uses p1 and p2 ...
}
```

Is this code valid? Discuss.

----

**Solution
Consider the following code:

``` cpp
void f( vector<char>& v ) {
    char* pc = &v[0];
    // ... later, uses pc ...
}
```

**1.** Is this code valid?

Yes, as long as v is nonempty.

Why is this code legal? The reason comes right out of the standard's Container and Sequence requirements: If `operator[]` is provided for a given kind of sequence (which it is in the case of `std::vector<char>`), it must return a "reference". In turn, the "reference" must be an lvalue of type T (here char), which can have its address taken.

**Why Take Pointers or References Into Containers?**
Many programmers are surprised by this kind of code the first time they see it, but there's nothing wrong with this technique in itself. As long as you remain aware of when the pointers might be invalidated, which is pretty much whenever iterators would be invalidated, this technique is not evil, it is not immoral, and it's not even fattening. On the contrary, it can be quite useful.

There are cases where it can be legal (and, more to the point, where it makes perfect sense) to have pointers or references into containers. A common case is when you read a data structure into memory on program startup and then never modify it, but you need to access it frequently in different ways. In such cases it can make sense to have additional data structures that contain pointers into the main container to optimize different access methods. I'll give an example as part of the discussion of Question #2.

**2.** Whether it's valid or not, how could it be improved?

In general, it's not a bad guideline to prefer to use iterators instead of pointers when you want to point at an object that's inside a container. After all, iterators are invalidated at mostly the same times and the same ways as pointers.

There are two main potential drawbacks to this method. If either applies in your case, continue to use pointers.

1. You can't always conveniently use an iterator where you can use a pointer. (See example below.)

2. Using iterators might incur extra space and performance overhead, in cases where the iterator is an object and not just a bald pointer.

Example: Say that you have a `map<Name,PhoneNumber>` that, given a name, makes it easy to look up a phone number. What if you need to do the reverse lookup too? One solution is to build a second structure, perhaps a `map<PhoneNumber*,Name*,DerefFunctor>` that enables the reverse lookup but avoids doubling the storage overhead (no need to store each name and phone number twice; the second structure simply has pointers into the first).

Note that, in the above example, it would be difficult to use iterators instead of pointers. Why? Because the natural candidate, `map<Name,PhoneNumber>::iterator`, points to a `pair<Name,PhoneNumber>`, and there's no handy way to get an iterator to just the name or phone number part individually.

This brings us to the next (and in my opinion most interesting) point of today's lesson:

**When Is a Container Not a Container?**
**3.** Consider the following code:

``` cpp
template<class T>
```

The above line has two problems:

a) The easy one is that it doesn't make sense to define T in terms of itself. That red herring may have successfully tricked you into missing the more fundamental problem, namely:

b) Function templates can't have default arguments.

Now consider the rest of the function as if it were introduced with "template<class T>" only:

``` cpp
void f( T& t ) {
    typename T::value_type* p1 = &t[0];
    typename T::value_type* p2 = &*t.begin();
    // ... later, uses p1 and p2 ...
}
```
Is this code valid? Discuss.

The short answer is: Sometimes.

One instructive way to look at this problem is to think about what kinds of T and t make the code valid. At compile time, what characteristics and abilities must T have? At runtime, what characteristics must t have? Let's do some detective work.

At compile time:

a) To make the expression &t[0] valid, T::operator[] must exist and must return something that understands operator&.

In particular, this is true of containers that meet the standard's Container requirements and implement the optional operator[], because that operator must return a reference to the contained object. By definition, you can then take the contained object's address.

b) To make the expression &*t.begin() valid, T::begin() must exist, and it must return something that understands operator*, which in turn must return something that understands operator&.

In particular, this is true of containers that meet the standard's iterator requirements, because the iterator returned by begin() must, when dereferenced using operator*, return a reference to the contained object. By definition, you can then take the contained object's address.

Further, at runtime:

c) To make the expression &t[0] safe, the t object must be in a suitable state for calling t[0]. In particular, if T is something like std::vector<int>, then the vector must not be empty.

d) To make the expression &*t.begin() safe, the t object must be in a suitable state for calling t.begin() and dereferencing the result. In particular, if T is something like std::vector<int>, then the vector must be nonempty.

**Which Brings Us to the Embarrassing Part**
The code in Question #3 will work for every container in the standard library that supports operator[] \-\- and, if you take away the line containing "&t[0]", it will work for every container in the standard library, bar none \-\- EXCEPT for std::vector<bool>. In fact, the following template:

``` cpp
template<class T>
void g( vector<T>& v ) {
  T* p = &*t.begin();
  // ... do something with p ...
}
```

works for every type EXCEPT bool. If this seems a little strange to you, you're not alone.

**What About bool?**
Unfortunately, there's something a little embarrassing about the C++ standard library: not all of the templates it defines that look like containers actually are Containers. In particular,

   ***std::vector<bool> IS NOT a Container***

because it does not meet the standard library's requirements for Containers. It appears in the Containers and Sequences section of the standard without any note to indicate that it is neither.

(If anyone else had written vector<bool>, it would have been called "nonconforming" and "nonstandard." Well, it's in the standard, so that makes it a little bit harder to call it those names at this point, but some of us try anyway in the hopes that it will eventually get cleaned up. The correct solution is to remove the vector<bool> specialization requirement so that vector<bool> really is a vector of plain old bools. Besides, it's mostly redundant: std::bitset was designed for this kind of thing.)

The reason std::vector<bool> is nonconforming is that it pulls tricks under the covers in an attempt to optimize for space: Instead of storing a full char or int for every bool[1] (taking up at least 8 times the space, on platforms with 8-bit chars), it packs the bools and stores them as individual bits (inside, say, chars) in its internal representation. One consequence of this is that it can't just return a normal bool& from its operator[] or its dereferenced iterators[2]; instead, it has to play games with a helper "proxy" class that is bool-like but is definitely not a bool. Unfortunately, that also means that access into a vector<bool> is slower, because we have to deal with proxies instead of direct pointers and references.

All of the above trickery has the following disadvantages:

1. std::vector<bool> is not a Container. 'Nuff said.

2. std::vector<bool> attempts to illustrate how to write proxied containers that pull tricks under the covers. Unfortunately, that's not a sound idea, because by definition that violates the Container requirements. (See #1.) (The Container requirements were never changed to enable proxied containers to actually be conforming, and as far as I know that decision was deliberate.)

3. std::vector<bool>'s name is misleading because the things inside aren't even standard bools. A standard bool is at least as big as a char, so that it can be used "normally." So, in fact, std::vector<bool> does not even store bools, despite the name.

4. std::vector<bool> forces a specific optimization on all users by enshrining it in the standard. That's not a good idea; different users have different requirements, and now all users of vector<bool> must pay the performance penalty even if they don't want or need the space savings.

Bottom line: If you care more about speed than you do about size, you shouldn't use std::vector<bool>. Instead, you should hack around this optimization by using a std::vector<char> or the like instead, which is unfortunate but still the best you can do.

**Notes**
1. sizeof(bool) is implementation-defined, but it must be at least 1.

2. That's because there is not now a standard way to express a pointer or a reference to a bit.




# 051 Extending the Standard Library - Part I 
**Difficulty: 7 / 10**
This issue lets you test your standard algorithm skills. What does the standard library function remove() actually do, and how would you go about writing a generic function to remove only the third element in a container?



**Problem**
**JG Questions**
**1.** What does the std::remove() function do? Be specific.

**2.** Write code that removes all values equal to 3 from a std::vector<int>.

**Guru Question**
**3.** A programmer working on your team wrote the following alternative pieces of code to remove the n-th element of a container.

``` cpp
//  Method 1: Write a special-purpose remove_nth algorithm.
template<class FwdIter>
FwdIter remove_nth( FwdIter first, FwdIter last, size_t n )
{
    /* ... */
}

//  Method 2: Write a functor which returns true the nth time it's applied, 
//            and use that as a predicate for remove_if.
template<class T>
class FlagNth
{
public:
    FlagNth( size_t n ) : i_(0), n_(n) { }
    bool operator()( const T& ) { return i_++ == n_; }
private:
    size_t       i_;
    const size_t n_;
};

// Example invocation
... remove_if( v.begin(), v.end(), FlagNth<int>(3) ) ...
```

a) Implement the missing part of Method 1.

b) Which method is better? Why? Discuss anything that might be problematic about either solution.



**Solution**
**1.** What does the std::remove() function do? Be specific.

std::remove() does not physically remove objects from a container; the size of the container is unchanged. Rather, remove() shuffles up the "unremoved" objects to fill in the gaps left by removed objects, leaving at the end one "dead" object for each removed object. Finally, remove() returns an iterator pointing at the first "dead" object (if no objects were removed, remove() returns the end iterator).

For example, consider a vector<int> v that contains the following:

```
    1 2 3 1 2 3 1 2 3
```

Say that you called remove( v.begin(), v.end(), 2 ). What would happen? The answer is something like this:

```
    1 3 1 3 1 3 ? ? ?
    ^^^^^^^^^^^ ^^^^^
    unremoved   "dead"
    objects     objects
    
                ^
                iterator returned by
                remove() points to
                the third-last object
                (because three were
                removed)
```

Three objects had to be removed, and the rest were copied to fill in the gaps. The objects at the end of the container may have their original values (1 2 3), or they may not; don't rely on that. Again, note that the size of the container is left unchanged.

If you're wondering "why does remove() work that way?", see Andy Koenig's thorough treatment of this topic in the February 1999 C++ Report. [Disclaimer: I wrote this GotW issue several months ago but, this time, Andy beat me to publication... His column appeared on my desk within days of my getting around to posting GotW #51.]

**2.** Write code that removes all values equal to 3 from a std::vector<int>.

Here's a one-liner to do it, where v is a vector<int>:

``` cpp
    v.erase( remove( v.begin(), v.end(), 3 ), v.end() );
```
The call to remove( v.begin(), v.end(), 3 ) does the actual work, and returns an iterator pointing to the first "dead" element. The call to erase() from that point until v.end() shrinks the vector so that it contains only the unremoved objects.

**3.** A programmer working on your team wrote the following alternative pieces of code to remove the n-th element of a container.

``` cpp
//  Method 1: Write a special-purpose remove_nth algorithm.
//
template<class FwdIter>
FwdIter remove_nth( FwdIter first, FwdIter last, size_t n )
{
    /* ... */
}

//  Method 2: Write a functor which returns true the nth time it's applied, 
//            and use that as a predicate for remove_if.
template<class T>
class FlagNth
{
public:
    FlagNth( size_t n ) : i_(0), n_(n) { }
    bool operator()( const T& ) { return i_++ == n_; }
private:
    size_t       i_;
    const size_t n_;
};

// Example invocation
... remove_if( v.begin(), v.end(), FlagNth<int>(3) ) ...
```
**a)** Implement the missing part of Method 1.

People often propose implementations that have the same bug as the following code. Did you?

``` cpp
//  Can you see the problem(s)?
template<class FwdIter>
FwdIter remove_nth( FwdIter first, FwdIter last, size_t n )
{
    for( ; n > 0; ++first, \-\-n ) ;
    
    if( first != last )
    {
        FwdIter dest = first;
        return copy( ++first, last, dest );
    }
    
    return last;
}
```

There is one problem here, and the one problem hath two aspects:

    1. The initial loop may move first past last, and then \[first,last) is no longer a valid iterator range. If so, then in the remainder of the function, Bad Things(tm) will happen.
    
    2. It would be fine to leave that potential problem in the function, as long as you also documented a precondition that n be valid for the given range. But this opens you up to some further criticism:
    
        a) Not all who proposed such code documented the precondition.
    
        b) If you're going to do that, you should dispense with the iterator-advancing loop and simply write advance( first, n ). The standard advance() function for iterators is already aware of iterator categories, and is automatically optimized for random-access iterators.

Here is a reasonable implementation:

``` cpp
//  Preconditions:
//  - first != last
//  - n must not exceed the size of the range
template<class FwdIter>
FwdIter remove_nth( FwdIter first, FwdIter last, size_t n )
{
    advance( first, n );
    if( first != last )
    {
        FwdIter dest = first;
        return copy( ++first, last, dest );
    }
    return last;
}
```

**b)** Which method is better? Why? Discuss anything that might be problematic about either solution.

Method 1 has two main advantages:

1. It is correct.

2. It can take advantage of iterator traits, specifically the iterator category, and so can perform better for random-access iterators.

Method 2 has corresponding disadvantages, which we'll analyze in detail in the second part of this two-part GotW.



# 052 Extending the Standard Library - Part II 
**Difficulty: 7 / 10**
Following up from the introduction given in #51, we now examine "stateful" predicates. What are they? When are they useful? How compatible are they with standard containers and algorithms?

----

**Problem**
**JG Question**
**1.** What are predicates, and how are they used in STL? Give an example.

**Guru Questions**
**2.** When would a "stateful" predicate be useful? Give examples.

**3.** What requirements on algorithms are necessary in order to make stateful predicates work correctly?

----

**Solution**
**1.** What are predicates, and how are they used in STL?

A predicate is a pointer to a function, or a functor (an object that supplies the function call operator, operator()()), that an algorithm can apply to each element it operates on. Algorithms normally use the predicate to make a decision about the element, so a predicate 'pred' should work correctly when used as follows:

``` cpp
    //  Example 1(a).

    if(pred(*first))
    {
        /* ... */
    }
```

As you can see from this example, 'pred' should return a value that can be tested as true. Note that a predicate is not allowed to use any non-const function through the dereferenced iterator.

Some predicates are binary; that is, they take two dereferenced iterators as arguments. This means that a binary predicate 'bpred' should work correctly when used as follows:

``` cpp
    //  Example 1(b).

    if(bpred(*first1, *first2))
    {
        /* ... */
    }
```

Give an example.

Consider the following implementation of the standard algorithm find_if():

``` cpp
//  Example 1(c).

template<class Iter, class Pred> inline
Iter find_if( Iter first, Iter last, Pred pred )
{
    for(; first != last; ++first)
    {
        if(pred(*first))
        {
            break;
        }
    }
    return (first);
}
```

This implementation of the algorithm visits every element in the range \[first, last) in order, applying the predicate function pointer or object 'pred' to each element. If there is an element for which the predicate evaluates to true, find_if() returns an iterator pointing to the first such element. Otherwise, find_if() returns 'last' to signal that an element satisfying the predicate was not found.

We can use find_if() with a function pointer predicate as follows:

``` cpp
//  Example 1(d): Using find_if() with a
//                function pointer.

bool GreaterThanFive( int i )
{
    return i > 5;
}
bool IsAnyElementGreaterThanFive( vector<int>& v )
{
    return find_if(v.begin(), v.end(), GreaterThanFive) != v.end();
}
```

Here's the same example, only using find_if() with a functor instead of a free function:

``` cpp
//  Example 1(e): Using find_if() with a
//                functor.
//
struct GreaterThanFive
{
    bool operator()( int i )
    {
      return i > 5;
    }
};
bool IsAnyElementGreaterThanFive( vector<int>& v )
{
    return find_if(v.begin(), v.end(), GreaterThanFive()) != v.end();
}
```

In this example, there's not much benefit to using a functor over a free function, is there? But this leads us nicely into our guru questions, where the functor starts showing much greater flexibility:

**2.** When would a "stateful" predicate be useful? Give examples.

Continuing on from examples 1(d) and 1(e), here's something that a free function can't do as easily without using something like a static variable:

``` cpp
//  Example 2(a): Using find_if() with a
//                more general functor.

class GreaterThan
{
public:
    GreaterThan( int value ) : value_( value ) { }
    bool operator()( int i ) const
    {
      return i > value_;
    }
private:
    const int value_;
};

bool IsAnyElementGreaterThanFive( vector<int>& v )
{
    return find_if(v.begin(), v.end(), GreaterThan(5)) != v.end();
}
```

This GreaterThan predicate has member data that remembers a value, in this case the value that it should compare each element against. You can already see that this version is much more usable \-\- and, therefore, reusable \-\- than the special-purpose code in examples 1(d) and 1(e), and a lot of the power comes from the ability to store local information inside the object like this. (This object is not "stateful" yet, though, because once constructed it does not change.)

Taking it one step further, we end up with something even more generalized:

``` cpp
//  Example 2(b): Using find_if() with a
//                fully general functor.

template<class T>
class GreaterThan
{
public:
    GreaterThan( T value ) : value_( value ) { }
    bool operator()( const T& t ) const
    {
      return t > value_;
    }

private:
    const T value_;
};

bool IsAnyElementGreaterThanFive( vector<int>& v )
{
    return find_if(v.begin(), v.end(), GreaterThan<int>(5)) != v.end();
}
```

So we can see some usability benefits from using predicates that store value.

**The Next Step: Stateful Predicates**
The predicates in both examples 2(a) and 2(b) have an important property: Copies are equivalent. That is, if you make a copy of a GreaterThan<int> object, it behaves in all respects just like the original one and can be used interchangeably. This turns out to be important, as we shall see in Question #3.

Some people have tried to write "stateful" predicates that go further by changing as they're used; that is, the result of applying a predicate depends on its history of previous applications.[1]

Examples of such stateful predicates appear in books. In particular, people have tried to write predicates that keep track of various information about the elements that they were applied to. For example, people have proposed predicates that remember the values of the objects they were applied to in order to perform calculations (e.g., a predicate that returns true as long as the average of the values it was applied to so far is more than 50, or the total is less than 100, and so on). We just saw a specific example of this kind of stateful predicate in GotW #51, Question #3:

``` cpp
//  Example 2(c)
//  (From GotW #51, Question #3)

//  Method 2: Write a functor which returns true
//            the nth time it's applied, and use
//            that as a predicate for remove_if.

template<class T>
class FlagNth
{
public:
    FlagNth( size_t n ) : i_(0), n_(n) { }
    bool operator()( const T& ) { return i_++ == n_; }
private:
    size_t       i_;
    const size_t n_;
};
```

Stateful predicates are sensitive to the way they are applied to elements in the range being operated on: This one in particular depends on both the number of times it has been applied (this should be obvious), and on the order in which it is applied to the elements in the range (if used in conjunction with something like remove_if(); this may be less obvious at first).

The major difference between predicates that are stateful and those that aren't are that, for stateful predicates, copies are NOT equivalent. Clearly an algorithm couldn't make a copy of a FlagNth<T> object, and apply one object to some elements and the other object to other elements. That wouldn't give the expected results at all, because the two predicate objects would update their counts independently and neither would be able to flag the correct n-th element; each could only flag the n-th element that it happened to be applied to.

The problem is that, in GotW #51's Question #3, our Method 2 tried to use a FlagNth<T> object in just such a way (possibly):

``` cpp
    // Example invocation
    ... remove_if( v.begin(), v.end(), FlagNth<int>(3) ) ...
```

"Looks reasonable, and I've used this technique," you say? "I just read a C++ book that demonstrates this technique, so it must be fine," you say? Well, the truth is that this technique may happen to work on your implementation (or on the implementation that the author of book with the error in it was using), but it is NOT guaranteed to work portably on all implementations, or even on the next version of the implementation you are (or that author is) using now.

Let's see why, by examining remove_if() in a little more detail in Question #3:

**3.** What requirements on algorithms are necessary in order to make stateful predicates work correctly?

For stateful predicates to be really useful with an algorithm, the algorithm must generally guarantee two things about how it uses the predicate:

a) that it does not make copies of the predicate (that is, it should consistently use the same object that it was given); and

b) that the predicate is applied to the elements in the range in some known order (usually, first to last).

Alas, the standard does not require that the standard algorithms meet these two guarantees (yes, even though stateful predicates have appeared in books; in a battle between the standard and a book, the standard wins). The standard does mandate other things for standard algorithms, such as the performance complexity and the number of times a predicate is applied, but in particular it never specifies requirement (a) for any algorithm.

For example, consider std::remove_if():

a) It's common for standard library implementations implement remove_if() in terms of find_if(), and pass the predicate along to find_if() by value. This will make the predicate behave unexpectedly, because the predicate object actually passed to remove_if() is not necessarily applied once to every element in the range... "the predicate object _or a copy of the predicate object_" is what is guaranteed to be applied once to every element. This is because a conforming remove_if() is allowed to assume that copies of the predicate are equivalent. [As an aside, note that this means that auto_ptrs can never be used as predicates.]

b) The standard requires that the predicate supplied to remove_if() be applied exactly "last - first" times, but that doesn't mean that the predicate will necessarily be applied to the elements in any given order. It's possible (if a little obnoxious) to write a conforming implementation of remove_if() that doesn't apply the predicate to the elements in order. The point is that, if it's not required by the standard, you can't portably depend on it.

"Well," you ask, "isn't there ANY way to make stateful predicates like FlagNth<> work reliably with the standard algorithms?" Unfortunately, the answer is No.

...All right, all right, quiet down! I can already hear the howls from the folks who write predicates that use reference-counting techniques to solve the predicate-copying problem (a) above. Yes, you can share the predicate state so that a predicate can be safely copied without changing its semantics when it is applied to objects. The following code uses this technique (for a suitable CountedPtr template; follow-up question: provide a suitable implementation of CountedPtr):

``` cpp
//  Example 3(a): A (partial) solution that
//                shares state between copies.

template<class T>
struct FlagNthImpl
{
    FlagNthImpl( size_t nn ) : i(0), n(nn) { }
    size_t       i;
    const size_t n;
};

template<class T>
class FlagNth
{
public:
    FlagNth( size_t n )
        : pimpl_( new FlagNthImpl<T>( n ) )
    {
    }
    
    bool operator()( const T& )
    {
        return (pimpl_->i)++ == pimpl_->n;
    }
private:
    CountedPtr<FlagNthImpl<T> > pimpl_;
};
```

But this doesn't (and can't) solve the ordering problem (b) above. That is, you the programmer are entirely dependent on the order in which the predicate is applied by the algorithm. There's no way around this, not even with fancy pointer techniques, unless the algorithm itself guarantees a traversal order.

**Follow-Up Question**
The follow-up question, above, was to provide a suitable implementation of CountedPtr, a smart pointer for which making a new copy just points at the same representation, and the last copy to be destroyed cleans up the allocated object. Here is a skeletal one I just made up on the spot:

``` cpp
template<class T>
class CountedPtr
{
private:
    struct Impl
    {
        Impl( T* pp ) : p( pp ), refs( 1 ) { }
        
        ~Impl() { delete p; }
        
        T*     p;
        size_t refs;
    };
    Impl* impl_;

public:
    CountedPtr( T* p )
        : impl_( new Impl( p ) )
    {
    }
    
    ~CountedPtr()
    {
        Decrement();
    }
    
    CountedPtr( const CountedPtr& other )
        : impl_( other.impl_ )
    {
        Increment();
    }
    
    CountedPtr& operator=( const CountedPtr& other )
    {
        if( impl_ != other.impl_ )
        {
            Decrement();
            impl_ = other.impl_;
            Increment();
        }
        return *this;
    }
    
    T* operator->()
    {
        return impl_->p;
    }

private:
    void Decrement()
    {
        if( --(impl_->refs) == 0 )
        {
            delete impl_;
        }
    }
    
    void Increment()
    {
        ++(impl_->refs);
    }
};
```

**Notes**
1. As Astute Reader John D. Hickin so elegantly describes it: "The input \[first, last) is somewhat like the tape fed to a Turing machine and the stateful predicate is like a program."



# 053 Migrating to Namespaces 
**Difficulty: 4 / 10**
Standard C++ includes support for namespaces and name visibility control with using declarations and directives. What's the best way to use these powerful facilities, and what's the most effective way to initially migrate your existing C++ code to a namespace-aware compiler and library?



**Problem**
**JG Question**
1. What are using-declarations and using-directives, and how are they used? Give examples.

Bonus: What does MLOC stand for?

**Guru Question**
2. You are working on a MLOC project with over 1,000 .h and .cpp files. The project team is just upgrading to the latest version of your compiler, which finally both supports namespaces and puts all of the standard library features into namespace std. Unfortunately, this conformance boon has the side effect of breaking your current build. As always, there is never enough time allocated to do a detailed job; you would like to analyze every source file and write exactly the needed using-declarations in each one, but that's infeasible at this time.

What is the most effective way to deal with this problem and safely migrate your code base to the new (and more standard) environment? Discuss alternatives, and prefer the quickest approach that gets the job done without compromising future safety and usability. How can you best defer unnecessary (for now) migration work to the future without increasing the overall migration workload later?



**Solution**
**1.** What are using-declarations and using-directives, and how are they used? Give examples.

**Using Declarations**
A using declaration creates a local synonym for a specific name actually declared in another namespace. For the purposes of overloading and name resolution, a using declaration works just like any declaration.

``` cpp
//  Example 1a

namespace A
{
    int f( int );
    int i;
}
using A::f;
int f( int );
int main()
{
    f( 1 );     // ambiguous: A::f() or ::f()?
    int i;
    using A::i; // error, double declaration (just as
                // writing "int i;" again would be)
}
```

A using declaration only brings in names whose declarations have already been seen. For example:

``` cpp
//  Example 1b
//
namespace A
{
    class X {};
    int Y();
    int f( double );
}

using A::X;
using A::Y; // only function A::Y(), not class A::Y
using A::f; // only A::f(double), not A::f(int)

namespace A
{
    class Y {};
    int f( int );
}
int main()
{
    X x;      // OK, X is a synonym for A::X
    Y y;      // error, A::Y not visible because the using declaration preceded the
              //  actual declaration that's wanted
              
    f( 1 );   // oops: this uses a silent implicit conversion and calls A::f(double), 
              // NOT A::f(int), because only the first A::f() declaration had been
              // seen at the point of the using declaration for the name A::f
}   
```

**Using Directives**
A using directive allows all names in another namespace to be used in the scope of the using directive. Unlike a using declaration, a using directive brings in names declared after the using directive. For example:

``` cpp
//  Example 1c

namespace A
{
    class X {};
    int f( double );
}

using namespace A;

namespace A
{
    class Y {};
    int f( int );
}

int main()
{
    X x;      // OK, X is a synonym for A::X
    Y y;      // OK, Y is a synonym for A::Y
    f( 1 );   // OK, calls A::f(int)
}
```

Bonus: What does MLOC stand for?

MLOC stands for millions of lines of code. Typically, the measurement counts only noncomment, nonblank lines.

**Migrating To Namespaces, Safely and Effectively**
Background: The situation described in Question #2 is pretty typical for projects whose compilers are supporting namespaces for the first time.

Note that in such a situation by definition there can't already be name conflicts in the existing project (including both custom code and libraries) because namespaces didn't yet exist, so the main problem that needs a solution is that now the standard library is in namespace std and so unqualified uses of "std::" names in the project code will now fail to compile.

**2.** You are working on a MLOC project with over 1,000 .h and .cpp files. The project team is just upgrading to the latest version of your compiler, which finally both supports namespaces and puts all of the standard library features into namespace std. Unfortunately, this conformance boon has the side effect of breaking your current build.

First, let's go on a short tangent to analyze the best long-term solution. Then we'll know better what to look for in a good short-term solution that won't end up being just thrown away and reworked when we get to the long-term solution.

**Characteristics of a Good Long-Term Solution**
As always, there is never enough time allocated to do a detailed job; you would like to analyze every source file and write exactly the needed using-declarations in each one, but that's infeasible at this time.

In short, a good long-term solution for namespace usage should follow at least these rules:

1. Using directives should be avoided entirely, especially in header files.[1]

The reason for Guideline #1 is that using directives cause wanton namespace pollution by bringing in potentially huge numbers of names, many (usually the vast majority) of which are unnecessary. The presence of the unnecessary names greatly increases the possibility of unintended name conflicts \-\- not just in the header, but in every module that #includes the header.

2. Namespace using declarations should never appear in header files.

The reason for Guideline #2 is less obvious.

Most authors recommend that using declarations never appear AT FILE SCOPE in shared header files. That's good advice, as far as it goes, because at file scope a using declaration causes the same kind of namespace pollution as using directives, only less of it.

Unfortunately, in my opinion the above "usual advice" doesn't go far enough. Here is why you should never write using declarations in header files at all, not even in a namespace scope: The meaning of the using declaration may change depending on what other headers happen to be #included before it in any given module. (I'll come back to this point again when I get to Example 2c later on.)

Think about what this means. To illustrate, consider the following example module and two good long-term approaches to migrating the module to namespaces:
``` cpp
//  Example 2a: Original namespace-free code

//--- file x.h ---
//
#include "y.h"  // declares Y
#include <deque>
#include <iosfwd>
ostream& operator<<( ostream&, const Y& );
int f( const deque<int>& );

//--- file x.cpp ---
//
#include "x.h"
#include "z.h"  // declares Z
#include <ostream>
ostream& operator<<( ostream& o, const Y& y )
{
    // ... uses Z in the implementation ...
    return o;
}
int f( const deque<int>& d )
{
    // ...
}
```

How can we best migrate the above code to a compiler that respects namespaces and puts standard names in namespace std? Before reading on, please stop for a moment and think about what alternatives you might consider. Which ones are good? Which ones aren't? Why?

----

There are several ways you might approach this. I'll describe two common strategies, only one of which is good form.

**A Good Long-Term Solution**
A good long-term solution is to explicitly qualify every standard name wherever it is mentioned in x.h, and to write just the using declarations that are needed inside each source (.cpp) file for convenience, since those names are likely to be widely used in the source file:

``` cpp
//  Example 2b: Good long-term solution

//--- file x.h ---
//
#include "y.h"  // declares Y
#include <deque>
#include <iosfwd>
std::ostream& operator<<( std::ostream&, const Y& );
int f( const std::deque<int>& );

//--- file x.cpp ---
//
#include "x.h"
#include "z.h"  // declares Z
#include <ostream>
using std::deque;   // using declarations appear
using std::ostream; //  AFTER all #includes
ostream& operator<<( ostream& o, const Y& y )
{
  // ... uses Z in the implementation ...
  return o;
}
int f( const deque<int>& d )
{
  // ...
}
```

Note that the using declarations inside x.cpp had better come after all #include directives, otherwise they may introduce name conflicts into other header files depending on #include orders.

[An alternative good long-term solution would be to do the above but eschew the using declarations entirely and explicitly qualify each standard name, even inside the .cpp file. I don't recommend doing that, because it's a lot of extra work that doesn't deliver any real advantage in a typical situation like this.]

**A Not-So-Good Long-Term Solution**
In contrast, a bad long-term solution would be to put all of the project's code into its own namespace (which is not objectionable in itself, but isn't as useful as you might think) and write the using declarations or directives in the headers (which opens the door for potential problems). The reason some people have proposed this method is that it requires less typing than some other namespace-migration methods:

``` cpp
//  Example 2c: Bad long-term solution (or, Why to
//              avoid using declarations in
//              headers, even not at file scope)

//--- file x.h ---
//
#include "y.h"  // declares MyProject::Y and adds
                //  using declarations/directives
                //  in namespace MyProject
#include <deque>
#include <iosfwd>
namespace MyProject
{
  using std::deque;
  using std::ostream;
  // or, "using namespace std;"
  ostream& operator<<( ostream&, const Y& );
  int f( const deque<int>& );
}

//--- file x.cpp ---
//
#include "x.h"
#include "z.h"  // declares MyProject::Z and adds
                //  using declarations/directives
                //  in namespace MyProject
  // error: potential future name ambiguities in
  //        z.h's declarations, depending on what
  //        using declarations exist in headers
  //        that happen to be #included before z.h
  //        in any given module (in this case,
  //        x.h or y.h may cause potential changes
  //        in meaning)
#include <ostream>
namespace MyProject
{
  ostream& operator<<( ostream& o, const Y& y )
  {
    // ... uses Z in the implementation ...
    return o;
  }
  int f( const deque<int>& d )
  {
    // ...
  }
}
```

Note the error. The reason, as stated, is that the meaning of a using declaration in a header can change \-\- even when the using declaration is inside a namespace, and not at file scope \-\- depending on what else a client module may happen to #include before it. It's ALWAYS bad form to write any kind of code that can silently change meaning depending on the order in which headers are #included. (It's also bad form to write header code that requires the user to #include other headers manually ahead of it; headers should always be self-contained and work correctly even if #included alone.)

**An Effective Short-Term Solution**
The reason why I've spent time explaining desirable long-term solutions is so that, when we pick a simpler short-term solution, we pick one that won't compromise a good long-term solution:

What is the most effective way to deal with this problem and safely migrate your code base to the new (and more standard) environment? Discuss alternatives, and prefer the quickest approach that gets the job done without compromising future safety and usability. How can you best defer unnecessary (for now) migration work to the future without increasing the overall migration workload later?

Adding using declarations or directives in the headers would be the least work, but it's work that would have to be undone when we implement the correct long-term solution. We've already seen enough reasons not to go down that road. Given that we mustn't add using declarations or directives in headers, our only alternative is to add "std::" in the right places in the headers.

We can, however, get away with writing a using directive in each implementation file, which avoids/defers the tedious work of figuring out the (possibly lengthy) correct set of using declarations that will eventually go into each implementation file. Note that this doesn't compromise \-\- or change the meaning \-\- of any other code in any other modules, including the headers we #include, as long as we're careful to write the using directive after all of the #include statements. For convenience, and for a good reason that will become clearer in a moment, I'll put the using directive into a separate header to be #included after all others.

Finally, we can defer the work of moving to the new <cheader> C header style too, which also defers all of the associated namespace issues.

Here's the result:

``` cpp
//  Example 2d: Good short-term solution is to:
//              - in headers, write std:: as needed
//              - in .cpp files, #include
//                "myproject_last.h" after all
//                other headers
//
//--- file x.h ---
//
#include "y.h"  // declares Y
#include <deque>
#include <iosfwd>
std::ostream& operator<<( std::ostream&, const Y& );
int f( const std::deque<int>& );

//--- file x.cpp ---
//
#include "x.h"
#include "z.h"  // declares Z
#include <ostream>
#include "myproject_last.h"
                // AFTER all other #includes
ostream& operator<<( ostream& o, const Y& y )
{
  // ... uses Z in the implementation ...
  return o;
}
int f( const deque<int>& d )
{
  // ...
}
//--- common file myproject_last.h ---
//
using namespace std;
```

This does not compromise the long-term solution in that it doesn't do anything that will need to be "undone" for the long-term solution. At the same time, it's simpler and requires fewer code changes than the full long-term solution would.

**Migrating To the Long-Term Solution**
In the future, this allows a simple migration to the full long-term solution. Simply follow these steps:

- in each header: change lines that #include C headers to the new <cheader> style; e.g., change "#include <stdio.h>" to "#include <cstdio>"

- in myproject_last.h: comment out the using directive

- rebuild; in each implementation file, see what breaks and add the correct using declarations (after all #includes)

If you follow this advice, then even with looming project deadlines you'll be able to quickly \-\- and effectively \-\- migrate to a namespace-aware compiler and library, all without compromising your long-term solution.

**Notes**
1. Unless of course the header is only #included by one module, but this case is rare enough that I didn't want to cause possible confusion by writing "...in shared header files," lest someone think this advice applies only to headers shared between projects. It applies to all headers that are #included by more than one module.



# 054 Using Vector and Deque 
**Difficulty: 8 / 10**
What is the difference between vector and deque? When should you use each one? And how can you properly shrink such containers when you no longer need their full capacity? These answers and more, as we consider news updates from the standards front.

----

**Problem**
**JG Question**
**1.** In the standard library, vector and deque provide similar services. Which should you typically use? Why? Under what circumstances would you use the other?

**Guru Questions**
**2.** What does the following code do?

``` cpp
vector<C> c( 10000 );
c.erase( c.begin()+10, c.end() );
c.reserve( 10 );
```

**3.** A vector or deque typically reserves extra internal storage as a hedge against future growth, to prevent too-frequent reallocation as new elements are added. Is it possible to completely clear a vector or deque (that is, remove all contained elements AND free all internally reserved storage)? Demonstrate why or why not.

Warning: Answers 2 and 3 may be subtle. Each has a facile answer, but don't stop at the surface; try to be as detailed as possible.

----

**Solution**
**In Most Cases, Prefer Using deque (Controversial)**
**1.** In the standard library, vector and deque provide similar services. Which should you typically use? Why? Under what circumstances would you use the other?

The C++ Standard, in section 23.1.1, offers some advice on which containers to prefer. It says:

    vector is the type of sequence that should be used by default... deque is the data structure of choice when most insertions and deletions take place at the beginning or at the end of the sequence.

I'd like to present an amiably dissenting point of view: I recommend that you consider preferring deque by default instead of vector, especially when the contained type is a class or struct and not a builtin type, unless you really need the container's memory to be contiguous.

vector and deque offer nearly identical interfaces and are generally interchangeable. deque also offers push_front() and pop_front(), which vector does not. (True, vector offers capacity() and reserve(), which deque does not, but that's no loss \-\- those functions are actually a weakness of vector, as I'll demonstrate in a moment.)

The main structural difference between vector and deque lies in the way the two containers organize their internal storage: Under the covers, a deque allocates its storage in pages, or "chunks," with a fixed number of contained elements in each page; this why a deque is often compared to (and pronounced as) a "deck" of cards, although its name originally came from "double-ended queue" because of is ability to insert elements efficiently at either end of the sequence. On the other hand, a vector allocates a contiguous block of memory, and can only insert elements efficiently at the end of the sequence.

The paged organization of a deque offers significant benefits:

1. A deque offers constant-time insert() and erase() operations at the front of the container, whereas a vector does not \-\- hence the note in the Standard about using a deque if you need to insert or erase at both ends of the sequence.

2. A deque uses memory in a more operating system-friendly way, particularly on systems without virtual memory. For example, a 10-megabyte vector uses a single 10-megabyte block of memory, which is usually less efficient in practice than a 10-megabyte deque that can fit in a series of smaller blocks of memory.

3. A deque is easier to use, and inherently more efficient for growth, than a vector. The only operations supplied by vector that deque doesn't have are capacity() and reserve() \-\- and that's because deque doesn't need them! For vector, calling reserve() before a large number of push_back()s can eliminate reallocating ever-larger versions of the same buffer every time it finds out that the current one isn't big enough after all. A deque has no such problem, and having a deque::reserve() before a large number of push_back()s would not eliminate any allocations (or any other work) because none of the allocations are redundant; the deque has to allocate the same number of extra pages whether it does it all at once or as elements are actually appended.

Interestingly, the Standard stack adapter, which can only grow in one direction and so never needs insertion in the middle or at the other end, has its default implementation as a deque:

``` cpp
template <class T, class Container = deque<T> >
class stack {
    // ...
};
```

**Aside: For Those Concerned About the Above**
Some readers are going to be concerned about my advice to prefer deque, saying perhaps: "But deque is a more complex container than vector, and so deque must be very inefficient compared to vector, right?" As always, beware premature optimization before you actually measure: I have found the "deque must be inefficient" assumption to be generally untrue on popular implementations. Using MSVC 5.0 SP3 with the current patches for its bundled Dinkumware C++ Standard Library implementation (probably by far the most widely-used compiler and library configuration), I tested the performance of the following operations, in order:

| Operation| Description                                 |
| :------: | :-----------------------------------------: |
| grow     | first, perform 1,000,000 push_back()s       |
| traverse | then traverse, simply incrementing iterators, from begin() to end() |
| at       | then access each element in turn using at() |
| shuffle  | then random_shuffle() the entire container  |
| sort     | then sort() the entire container (for list, uses list::sort()) |

I tested each operation against several standard containers including deque. I expected vector to outperform deque in "traverse" and "at", and deque to win in "grow" for reason #3 above. In fact, note that deque did outperform vector on the "grow" test even though in fairness I gave vector special treatment by calling vector::reserve(1000000) first to avoid any reallocations. All timings varied less than 5% across multiple runs, and all runs executed fully in-memory without paging or other disk accesses.

First, consider what happened for containers of a simple builtin type, int:

| Times in ms | grow  | traverse |   at  | shuffle |   sort |
| :---:       | :---: | :---:    | :---: | :---:   | :---:  |
| vector<int> | 1015  |    55    |   219 |     621 |   3624 |
| deque<int>  |  761  |   265    |   605 |    1370 |   7820 |
| list<int>   | 1595  |   420    |   n/a |     n/a |  16070 |

Here list was always the slowest, and the difference between vector and deque wasn't as great as several people had led me to expect. Of course, the performance differences between the containers will fade when you use a contained type that is more expensive to manipulate. What's interesting is that most of the differences go away even when the contained type is as simple (and common) as a struct E { int i; char buf[100]; };:

| Times in ms | grow  | traverse |   at  | shuffle |   sort |
| :-----:     | :---: | :----:   | :---: | :-----: | :----: |
| vector<E>   | 3253  |    60    |   519 |    3825 |  17546 |
| deque<E>    | 3471  |   274    |   876 |    4950 |  21766 |
| list<E>     | 3740  |   497    |   n/a |     n/a |  15134 |

Now deque's performance disadvantage for even an intensive operation like sort is less than 25%.

Finally, note that the popular library implementation that I tested has since been revised and now includes a streamlined version of deque with simplified iterators. I do not yet have a copy of that library, but it will be interesting to see how much of the deque disadvantage in even the raw iterator "traverse" and element-accessing "at" tests will remain compared to vector.

So, for the three reasons cited earlier: Consider preferring deque by default in your programs, especially when the contained type is a class or struct and not a builtin type, unless the actual performance difference in your situation is known to be important or you specifically require the contained elements to be contiguous in memory (which typically means that you intend to pass the contents to a function that expects an array).

NOTE: If you think that relying on vector storage to be contiguous is a Bad Thing, in light of recent newsgroup discussions, see my [upcoming article](http://www.gotw.ca/publications/mill10.htm) in the July/August 1999 C++ Report... I give reasons why doing so is arguably okay, safe, and even portable.

**The Incredible Shrinking vector**
One of std::vector's most endearing features, at least compared to C-style arrays, is its encapsulated storage management. As we push elements onto a vector it just grows its storage automatically, and we can even give the vector hints about how much capacity to keep ready under the covers as it grows (by first calling reserve()) and only incur at most a single reallocation hit. This allows optimally efficient growth.

But what if we're doing the opposite? What if we're using a vector that's pretty big, and then we remove elements that we no longer need and want the vector to shrink to fit; that is, we want it to get rid of the now-unneeded extra capacity? You might think that the following would work:

**2.** What does the following code do?

``` cpp
vector<C> c( 10000 );
```

Line 1 creates a `vector<C>` object named c that initially contains 10,000 default-constructed C objects. At this point, we know that `c.capacity() >= 10,000`.

``` cpp
c.erase( c.begin()+10, c.end() );
```

Line 2 erases all but the first 10 elements in c. At this point, c's capacity is probably unchanged.

``` cpp
c.reserve( 10 );
```

Alas, line 3 does NOT shrink c's internal buffer to fit! Now `c.capacity()` is still `>= 10000` as before.

This example doesn't do what you might expect because calling `reserve()` will never shrink the vector's capacity; it can only increase the capacity, or do nothing if the capacity is already sufficient.

The Right Way To "Shrink-To-Fit" a vector or deque
So, can we write code that does shrink a vector "to fit" so that its capacity is just enough to hold the contained elements? Obviously `reserve()` can't do the job, but fortunately there is indeed a way:

``` cpp
vector<Customer>( c ).swap( c );
// ...now c.capacity() == c.size(), or perhaps a little more than c.size()
```

Do you see how the shrink-to-fit statement works? It's a little subtle:

1. First, we create a temporary (unnamed) `vector<Customer>` and initialize it to hold the same contents as c. The salient difference between the temporary vector and c is that, while c still carries around a lot of extra capacity in its oversize internal buffer, the temporary vector has just enough capacity to hold its copy of c's contents. (Some implementations may choose to round up the capacity slightly to their next larger internal "chunk size," with the result that the capacity actually ends up being slightly larger than the size.)

2. Next, we call `swap()` to exchange the internals of c with the temporary vector. Now the temporary vector owns the oversize buffer with the extra capacity that we're trying to get rid of, and c owns the "rightsized" buffer with the just-enough capacity.

3. Finally, the temporary vector goes out of scope, carrying away the old oversize buffer with it; the old buffer is deleted when the temporary vector is destroyed. Now all we're left with is c itself, but now c has a "rightsized" capacity.

Note that this procedure is not needlessly inefficient. Even if vector had a special-purpose `shrink_to_fit()` member function, it would have to do pretty much all of the same work just described above.

**The Right Way To Completely Clear a vector or deque**
**3.** A vector or deque typically reserves extra internal storage as a hedge against future growth, to prevent too-frequent reallocation as new elements are added. Is it possible to completely clear a vector or deque (that is, remove all contained elements AND free all internally reserved storage)? Demonstrate why or why not.

Again, the answer is Yes, it is possible. If you want to completely clear a vector, so that it has no contents and no extra capacity at all, the code is nearly identical to the shrink-to-fit code... you just initialize the temporary vector to be empty instead of making it a copy of c:

``` cpp
vector<Customer>().swap( c );
// ...now c.capacity() == 0, unless the implementation happens to enforce a
// minimum size even for empty vectors
```

Again, note that the vector implementation you're using may choose to make even empty vectors have some slight capacity, but now you're guaranteed that c's capacity will be the smallest possible allowed by your implementation: it will be the same as the capacity of an empty vector.

These techniques work for deque, too, but you don't need to do this kind of thing for list, set, map, multiset, or multimap because they allocate storage on an "exactly-as-needed" basis and so should never have excess capacity lying around.



# 055 Equivalent Code? 
**Difficulty: 5 / 10**
Can subtle code differences really matter, especially in something as simple as postincrementing a function parameter? This issue explores an interesting interaction that becomes important in STL-style code.

----

**Problem**
**JG Question**
1. Describe what the following code does:

``` cpp
//  Example 1
f( a++ );
```

Be as complete as you can about all possibilities.

**Guru Questions**
2. What is the difference, if any, between the following two code fragments?

``` cpp
//  Example 2(a)
f( a++ );

//  Example 2(b)
f( a );
a++;
```

3. In Question #2, make the simplifying assumption that f() is a function that takes its argument by value, and that a is an object of class type that has an operator++(int) with natural semantics. Now what is the difference, if any, between Example 2(a) and Example 2(b)?



**Solution**
**1. Describe what the following code does:**

``` cpp
//  Example 1
f( a++ );
```

Be as complete as you can about all possibilities.

A comprehensive list would be daunting, but here are the main possibilities.

First, f could be any of the following:

1. A macro. In this case the statement could mean just about anything, and "a++" could be evaluated many times or not at all. For example:

``` cpp
#define f(x) x                      // once
#define f(x) (x,x,x,x,x,x,x,x,x)    // 9 times
#define f(x)                        // not at all
```

Guideline: Avoid using macros. They usually make code more difficult to understand, and therefore more troublesome to maintain.

2. A function. In this case, first "a++" is evaluated and then the result is passed to the function as its parameter. Normally, postincrement returns the previous value of a in the form of a temporary object, so f() could take its parameter either by value or by reference to const, but not by reference to non-const because a reference to non-const cannot be bound to a temporary object.

3. An object. In this case, f() would be a functor, that is, an object for which operator()() is defined. Again, if postincrement returns the previous value of a (as postincrement always should) then f's operator()() could take its parameter either by value or by reference to const.

4. A type name. In this case, the statement first evaluates "a++" and uses the result of that expression to initialize a temporary object of type f.

Next, a could be:

   1. A macro. In this case, again, a could mean just about anything.
   
   2. An object (possibly of a built-in type). In this case, it must be a type for which a suitable operator++(int) postincrement operator is defined.

Normally, postincrement should be implemented in terms of preincrement and should return the previous value of a:

``` cpp
// Canonical form of postincrement:
T T::operator++(int)
{
    T old( *this ); // remember our original value
    ++*this;        // always implement postincrement
                    //  in terms of preincrement
    return old;     // return our original value
}
```

When you overload an operator, of course, you do have the option of changing its normal semantics to do "something unusual." For example, the following is likely to break the Example 1 code for most kinds of f, assuming a is of type A:

``` cpp
void A::operator++(int) // doesn't return anything
```

Don't do that. Instead, follow this sound advice:

Guideline: Always preserve natural semantics for overloaded operators. "Do as the ints do," that is, follow the semantics of the builtin types.

3. A value, such as an address. For example, a could be the name of an array, or it could be a pointer.

For the remaining questions, I will make the simplifying assumptions that:

   - f() is not a macro; and
   
   - a is an object with natural postincrement semantics.

**2. What is the difference, if any, between the following two code fragments?**

``` cpp
//  Example 2(a)
f( a++ );
```

This performs the steps:

   1. a++: Increment a and return the old value.
   
   2. f(): Execute f(), passing it the old value of a.

Example 2(a) ensures that the postincrement is performed, and therefore a gets its new value, before f() executes. As noted above, f() could still be a function, a functor, or a type name which leads to a constructor call.

Some coding standards state that operations like ++ should always appear on separate lines, on the grounds that it can be dangerous to perform multiple operations like ++ in the same statement because of sequence points (more about this in GotW #56). Instead, such coding standards would recommend:

``` cpp
//  Example 2(b)
f( a );
a++;
```

This performs the steps:

   1. f(): Execute f(), passing it the old value of a.
   
   2. a++: Increment a and return the old value, which is then ignored.

In both cases, f() gets the old value of a. "So what's the big difference?" you may ask. Well, Example 2(b) will not always have the same effect as that in Example 2(a), because Example 2(b) ensures that the postincrement is performed, and therefore a gets its new value, after f() executes. This has two major consequences: First, in the case where f() emits an exception, Example 2(a) ensures that a++ and all of its side effects have already been completed successfully, whereas Example 2(b) ensures that a++ has not been performed and none of its side effects have occurred.

Second, even in non-exceptional cases, if f() and a.operator++(int) have visible side effects, the order in which they are executed can matter. But, more specifically, consider what happens if f() has a side effect that affects the state of a itself: That's neither farfetched nor unlikely, and it can happen even if f() doesn't and can't directly change a, as I'll illustrate with an example:

**3. In Question #2, make the simplifying assumption that f() is a function that takes its argument by value, and that a is an object of class type that has an operator++(int) with natural semantics. Now what is the difference, if any, between Example 2(a) and Example 2(b)?**

The difference is that, for perfectly normal C++ code, Example 2(a) can be legal when Example 2(b) is not. Specifically, consider what happens when we replace f with list::erase(), and a with list::iterator. Now the first form is valid:

``` cpp
//  Example 3(a)
//  l is a list<int>
//  i is a valid non-end iterator into l
l.erase( i++ ); // OK
```

But the second form is not:

``` cpp
//  Example 3(b)
//  l is a list<int>
//  i is a valid non-end iterator into l
l.erase( i );
i++;            // error, i is not a valid iterator
```

The reason that Example 3(b) is incorrect is that the call to "l.erase( i )" invalidates i, and therefore you can no longer call operator++ on i afterwards.

**Scissors, Traffic, and Iterators**


Warning: Some programmers routinely write code like Example 3(b), perhaps because of coding guidelines that have a blanket policy of discouraging operations like ++ in function call statements.

If you're one of the programmers who writes code like Example 3(b), you may even be routinely getting away with it (and not realizing the danger) just because it happens to work on the current version of your compiler and library. But be warned: Code like Example 3(b) is not portable, it is not sanctioned by the Standard, and it's likely to turn and bite you when you port to another compiler platform or even just upgrade the one you're working on today. When it does bite, it will bite hard, because "using-an-invalid-iterator" bugs can be very difficult to find (unless you have the joy of working with a good checked library implementation during debugging \-\- but if you're in this situation you must not be using a checked implementation or else it would already have warned you about this!).

Some mothers (who are also software engineers) give the following three pieces of good advice, and we should always strive to follow them for our own good:

   1. Don't run with scissors.
   
   2. Don't play in traffic.
   
   3. Don't use invalid iterators.

Next time: With the exception of examples like the above, we'll see why it's still a good idea in general to avoid writing operations like ++ in function calls.



# 056 Exception-Safe Function Calls 
**Difficulty: 8 / 10**
Regular readers of GotW know that exception safety is anything but trivial. This puzzle points out an exception safety problem that was discovered only weeks before posting, and shows how best to avoid it in your own code.

----

**Problem
**JG Question**
**1.** In each of the following statements, what can you say about the order of evaluation of the functions f, g, and h and the expressions expr1 and expr2? Assume that expr1 and expr2 do not contain more function calls.

``` cpp
//  Example 1(a)
f( expr1, expr2 );

//  Example 1(b)
f( g( expr1 ), h( expr2 ) );
```

**Guru Questions**
**2.** In your travels through the dusty corners of your company's code archives, you find the following code fragment:

``` cpp
//  Example 2
//  In some header file:
void f( T1*, T2* );
//  In some implementation file:
f( new T1, new T2 );
```

Does this code have any potential exception safety problems? Explain.

**3.** As you continue to root through the archives, you see that someone must not have liked Example 2 because later versions of the files in question were changed as follows:

``` cpp
//  Example 3
//  In some header file:
void f( auto_ptr<T1>, auto_ptr<T2> );
//  In some implementation file:
f( auto_ptr<T1>( new T1 ), auto_ptr<T2>( new T2 ) );
```

What improvements does this version offer over Example 2, if any? Do any exception safety problems remain? Explain.

**4.** Demonstrate how to write an auto_ptr_new facility that solves the safety problems in Question 3 and can be invoked as follows:

``` cpp
//  Example 4
//  In some header file (same as in Example 2b):
void f( auto_ptr<T1>, auto_ptr<T2> );
//  In some implementation file:
f( auto_ptr_new<T1>(), auto_ptr_new<T2>() );
```

----

**Solution**
**Recap: Evaluation Orders and Disorders**
**1. In each of the following statements, what can you say about the order of evaluation of the functions f, g, and h and the expressions expr1 and expr2? Assume that expr1 and expr2 do not contain more function calls.**

Ignoring threads (which are beyond the scope of the C++ Standard), the answer to the first question hinges on these basic rules:

1. All of a function's arguments must be fully evaluated before the function is called. This includes the completion of any side effects of expressions used as function arguments.

2. Once the execution of a function begins, no expressions from the calling function begin, or continue, to be evaluated until execution of the called function has completed. Function executions never interleave with each other.

3. Expressions used as function arguments may generally be evaluated in any order, including interleaved, except as otherwise restricted by the other rules.

Given those rules, let's see what happens in our opening examples:

``` cpp
//  Example 1(a)
f( expr1, expr2 );
```

All we can say is: Both expr1 and expr2 must be evaluated before f() is called.

That's it. The compiler may choose to perform the evaluation of expr1 before, after, or interleaved with the evaluation of expr2. There are enough people who find this surprising that it comes up as a regular question on the newsgroups, but it's just a direct result of the C and C++ rules about sequence points.

``` cpp
//  Example 1(b)
f( g( expr1 ), h( expr2 ) );
```

The functions and expressions may be evaluated in any order that respects the following rules:

   - expr1 must be evaluated before g() is called.
   
   - expr2 must be evaluated before h() is called.
   
   - both g() and h() must complete before f() is called.
   
   - The evaluations of expr1 and expr2 may be interleaved with each other, but nothing may be interleaved with any of the function calls. For example, no part of the evaluation of expr2 nor the execution of h() may occur from the time g() begins until it ends.

That's it. For example, this means that any one or more of the following are possible:

  - Either g() or h() could be called first.
  
  - Evaluation of expr1 could begin, then be interrupted by h() being called, then complete. (Likewise for expr2 and g().)

**Some Function Call Exception Safety Problems**
**2. In your travels through the dusty corners of your company's code archives, you find the following code fragment:**

``` cpp
//  Example 2
//  In some header file:
void f( T1*, T2* );
//  In some implementation file:
f( new T1, new T2 );
```

Does this code have any potential exception safety problems? Explain.

Yes, there are several potential exception safety problems.

Brief recap: An expression like "new T1" is called, simply enough, a new-expression. Recall what a new-expression really does (I'll ignore placement and array forms for simplicity, since they're not very relevant here):

   - it allocates memory
   
   - it constructs a new object in that memory
   
   - if the construction fails because of an exception the allocated memory is freed

So each new-expression is essentially a series of two function calls: one call to operator new() (either the global one, or one provided by the type of the object being created), and then a call to the constructor.

For Example 1, consider what happens if the compiler decides to generate code as follows:

  1: allocate memory for T1
  2: construct T1
  3: allocate memory for T2
  4: construct T2
  5: call f()

The problem is this: If either step 3 or step 4 fails because of an exception, the C++ standard does not require that the T1 object be destroyed and its memory deallocated. This is a classic memory leak, and clearly not a Good Thing.

Another possible sequence of events is this:

   1: allocate memory for T1
   2: allocate memory for T2
   3: construct T1
   4: construct T2
   5: call f()

This sequence has, not one, but two different exception safety problems with different effects:

a) If step 3 fails because of an exception, then the memory allocated for T1 is automatically deallocated (step 1 is undone), but the standard does not require that the memory allocated for the T2 object be deallocated. The memory is leaked.

b) If step 4 fails because of an exception, then the T1 object has been allocated and fully constructed, but the standard does not require that it be destroyed and its memory deallocated. The T1 object is leaked.

"Hmm," you might wonder, "then why does this exception safety loophole exist at all? Why doesn't the standard just prevent the problem by requiring compilers to Do The Right Thing when it comes to cleanup?"

The basic answer is that it wasn't noticed, and even now that it has been noticed it might not be desirable to fix it. The C++ standard allows the compiler some latitude with the order of evaluation of expressions because this allows the compiler to perform optimizations that might not otherwise be possible. To permit this, the expression evaluation rules are specified in a way that is not exception-safe, and so if you want to write exception-safe code you need to know about, and avoid, these cases. (See below for how best to do this.)

Remember that C++ exception safety in general has not been well understood until fairly recently (namely the summer of 1997, just months before the standard was completed). Even so, the committee managed to add exception safety guarantees to the standard library, albeit necessarily at the eleventh hour in the last two meetings before the standard was cast in stone; many of the guarantees were added literally days before the concrete set. The problem with new-expressions has now been noticed, but for the reasons above the committee will have to decide whether the hole should be "fixed" or whether it should continue to be allowed in the name of compiler optimization flexibility.

**3. As you continue to root through the archives, you see that someone must not have liked Example 2 because later versions of the files in question were changed as follows:**

``` cpp
//  Example 3

//  In some header file:
void f( auto_ptr<T1>, auto_ptr<T2> );
//  In some implementation file:
f( auto_ptr<T1>( new T1 ), auto_ptr<T2>( new T2 ) );
```

What improvements does this version offer over Example 2, if any? Do any exception safety problems remain? Explain.

This code attempts to "throw auto_ptr at the problem." Many people believe that a smart pointer like auto_ptr is an exception safety panacea.

It is not. Nothing has changed. Example 3 is still not exception-safe, for exactly the same reasons as before.

Specifically, the problem is that the resources are safe only if they make it into a managing auto_ptr, but the same problems already noted above can still occur here before either auto_ptr constructor is ever reached. This is because both of the two problematic execution orders mentioned above are still possible, just with the auto_ptr constructors tacked onto the end before f(). For one example:

   1: allocate memory for T1
   2: construct T1
   3: allocate memory for T2
   4: construct T2
   5: construct auto_ptr<T1>
   6: construct auto_ptr<T2>
   7: call f()

In the above case, the same problems as before are still present if either of steps 3 or 4 throws. Similarly:

   1: allocate memory for T1
   2: allocate memory for T2
   3: construct T1
   4: construct T2
   5: construct auto_ptr<T1>
   6: construct auto_ptr<T2>
   7: call f()

Again, the same problems as before are still present if either of steps 3 or 4 throws.

Fortunately, though, this is not a problem with auto_ptr; auto_ptr is just being used the wrong way, that's all. In a moment, we'll see several ways to use it better.

**Aside: A Non-Solution**
Note that the following is not a solution:

``` cpp
//  In some header file:
void f(auto_ptr<T1> = auto_ptr<T1>(new T1), auto_ptr<T2> = auto_ptr<T1>(new T2));
//  In some implementation file:
f();
```

Why is this code not a solution? Because it's identical to Example 3 in terms of expression evaluation. Default arguments are considered to be created in the function call expression, even though they're actually written somewhere else entirely (namely, in the function declaration).

**A Limited Solution**
4. Demonstrate how to write an auto_ptr_new facility that solves the safety problems in Question 3 and can be invoked as follows:

``` cpp
//  Example 4

//  In some header file (same as in Example 2b):
void f( auto_ptr<T1>, auto_ptr<T2> );
//  In some implementation file:
f( auto_ptr_new<T1>(), auto_ptr_new<T2>() );
```

The simplest solution is to provide a function template like the following:

``` cpp
//  Example 4(a): Partial solution

template<class T>
auto_ptr<T> auto_ptr_new()
{
  return auto_ptr<T>( new T );
}
```

This solves the exception safety problems with Examples 2 through 4. No sequence of generated code can cause resources to be leaked, because now all we have is two functions, and we know that one must be executed entirely before the other. Consider the following evaluation order:

    1: call auto_ptr_new<T1>()
    2: call auto_ptr_new<T2>()

If step 1 throws, there are no leaks because the auto_ptr_new() template is itself strongly exception-safe.

If step 2 throws, then is the temporary auto_ptr<T1> created by step 1 guaranteed to be cleaned up? The answer is: Yes, it is. Now, one might wonder, isn't this pretty much the same as the "new T1" object being created in the corresponding case in Example 2, which isn't correctly cleaned up? No, this time it's not quite the same, because here the auto_ptr<T1> is actually a temporary object, and cleanup of temporary objects is correctly specified in the Standard. From the Standard, in 12.2/3:

    Temporary objects are destroyed as the last step in evaluating the full-expression (_intro.execution_) that (lexically) contains the point where they were created. This is true even if that evaluation ends in throwing an exception.

But Example 4(a) is only a limited solution: It only works with a default constructor, which breaks if a given type T doesn't have a default constructor, or if you don't want to use it. A more general solution is still needed.

**Generalizing the auto_ptr_new() Solution**
As pointed out by Dave Abrahams, we can extend the solution to support non-default constructors by providing a family of overloaded function templates:

``` cpp
//  Example 4(b): Improved solution

template<class T>
auto_ptr<T> auto_ptr_new()
{
    return auto_ptr<T>( new T );
}
template<class T, class Arg1>
auto_ptr<T> auto_ptr_new(const Arg1& arg1)
{
    return auto_ptr<T>( new T( arg1 ) );
}
template<class T, class Arg1, class Arg2>
auto_ptr<T> auto_ptr_new(const Arg1& arg1, const Arg2& arg2)
{
    return auto_ptr<T>( new T( arg1, arg2 ) );
}
// etc.
```

Now auto_ptr_new() fully and naturally supports non-default construction.

**A Better Solution**
Although auto_ptr_new() is nice, is there any way we could have avoided all of the exception safety problems without writing such helper functions? Could we have avoided the problems with better coding standards? The answer is yes, and here is one possible standard that would have eliminated the problem:

POSSIBLE GUIDELINE (Alternative #1):

Never allocate resources (e.g., via new) in the same expression as any other code that could throw an exception. This applies even if the new resource will immediately be managed (e.g., passed to an auto_ptr constructor) in the same expression.

In the Example 3 code, the way to satisfy this guideline is to move one of the temporary auto_ptrs into a separate named variable:

``` cpp
//  Example 3(a): A solution

//  In some header file:
void f( auto_ptr<T1>, auto_ptr<T2> );
//  In some implementation file:
{
    auto_ptr<T1> t1( new T1 );
    f( t1, auto_ptr<T2>( new T2 ) );
}
```

This satisfies Guideline #1 because, although we are still allocating a resource, it can't be leaked due to an exception because it's not being created in the same expression as any other code that could throw.[1]

Here is another possible coding standard, which is even simpler and easier to get right (and easier to catch in code reviews):

POSSIBLE GUIDELINE (Alternative #2):

Perform every resource allocation (e.g., new) in its own code statement which immediately gives the new resource to a manager object (e.g., auto_ptr).

In Example 3, the way to satisfy this guideline is to move both of the temporary auto_ptrs into separate named variables:

``` cpp
//  Example 3(b): A simpler solution

//  In some header file:
void f( auto_ptr<T1>, auto_ptr<T2> );
//  In some implementation file:
{
    auto_ptr<T1> t1( new T1 );
    auto_ptr<T2> t2( new T2 );
    f( t1, t2 );
}
```

This satisfies Guideline #2, and it required a lot less thought to get it right. Each new resource is created in its own expression and is immediately given to a managing object.

**Summary**
My recommendation is:

    GUIDELINE:
    
    Perform every resource allocation (e.g., new) in its own code statement which immediately gives the new resource to a manager object (e.g., auto_ptr).

This guideline is easy to understand and remember, it neatly avoids all of the exception safety problems in the original problem, and by mandating the use of manager objects it helps to avoid many other exception safety problems as well. This guideline is a good candidate for inclusion in your team's coding standards.

**Acknowledgments**
This GotW was prompted by a discussion thread on comp.lang.c++.moderated. This solution draws on observations presented by gurus James Kanze, Steve Clamage, and Dave Abrahams in that and other threads, and in private correspondence.

**Notes**
1. Yes, I'm being a little fuzzy, because I know that the body of f() is included in the expression evaluation and we don't care whether or not it throws.



# 057 Recursive Declarations 
**Difficulty: 6 / 10**
Can you write a function that returns a pointer to itself? If so, why would you want to?

----

**Problem**
**JG Question**
1. What is a pointer to function? How can it be used?

**Guru Questions**
2. Assume that it is possible to write a function that can return a pointer to itself. Then that function could equally well return a pointer to any function with the same signature as itself. When might this be useful? Explain.

3. Is it possible to write a function f() that returns a pointer to itself? It should be usable in the following natural way:
``` cpp
//  Example 3

//  FuncPtr is a typedef for a pointer to a
//  function with the same signature as f()
FuncPtr p = f();    // executes f()
(*p)();             // executes f()
```
If it is possible, demonstrate how. If it is not possible, explain why.

----

**Solution**
**Function Pointers: Recap**
**1. What is a pointer to function? How can it be used?**

Just as a pointer to an object lets you dynamically point at an object of a given type, a pointer to a function lets you dynamically point at a function with a given signature. For example:

``` cpp
//  Example 1

// Create a typedef called FPDoubleInt for a function
// signature that takes a double and returns an int.

typedef int (*FPDoubleInt)( double );
// Use it.

int f( double ) { /* ... */ }
int g( double ) { /* ... */ }
int h( double ) { /* ... */ }
FPDoubleInt fp;
fp = f;
fp( 1.1 );     // calls f()
fp = g;
fp( 2.2 );     // calls g()
fp = h;
fp( 3.14 );    // calls h()
```

When applied well, pointers to functions can allow significant runtime flexibility. For example, the standard C function qsort() takes a pointer to a comparison function with a given signature, which allows calling code to extend qsort()'s behavior by providing a custom comparison function.

**A Brief Look at State Machines**
**2. Assume that it is possible to write a function that can return a pointer to itself. Then that function could equally well return a pointer to any function with the same signature as itself. When might this be useful? Explain.**

Many situations come to mind, but one common example is when implementing a state machine.

In brief, a state machine is composed of a set of possible states along with a set of legal transitions between those states. For example, a simple state machine might look something like this:
```
  // Figure: A small state machine
          "a"
  Start -------> S2
    |             |
    |             | "see"
    |   "be"      v
    `---------> Stop
````
Here, when we are in the Start state, if we receive the input "a" we transition to state S2, otherwise if we receive the input "be" we transition to state Stop; any other input is illegal in state Start. In state S2, if we receive the input "see" we transition to state Stop; any other input is illegal in state S2. For this state machine, there are only two valid input streams: "be" and "asee".

To implement a state machine, one method that is sometimes sufficient is to make each state a function. All of the state functions have the same signature and each one returns a pointer to the next function (state) to be called. For example, here is an oversimplified code fragment that illustrates the idea:

``` cpp
//  Example 2

StatePtr Start( const string& input );
StatePtr S2   ( const string& input );
StatePtr Stop ( const string& input );
StatePtr Error( const string& input ); // error state
StatePtr Start( const string& input )
{
    if( input == "a" )
    {
        return S2;
    }
    else if( input == "be" )
    {
        return Stop;
    }
    else
    {
        return Error;
    }
}
```

See your favorite computer science textbook for more information about state machines and their uses. [A future GotW issue may address the question: "What is the best way to implement a state machine in C++?"]

Of course, the above code doesn't say what "StatePtr" is, and it's not necessarily obvious how to say what it should be. This leads us nicely into the final question:

**How Can a Function Return a Pointer to Itself?**
**3. Is it possible to write a function f() that returns a pointer to itself? It should be usable in the following natural way:**

``` cpp
//  Example 3

//  FuncPtr is a typedef for a pointer to a
//  function with the same signature as f()
FuncPtr p = f();    // executes f()
(*p)();             // executes f()
```

If it is possible, demonstrate how. If it is not possible, explain why.

Short answer: Yes, it's possible, but it's not obvious.

For example, one might first try something like this:

``` cpp
//  Example 3(a): Naive attempt
//                (doesn't work)

typedef FuncPtr (*FuncPtr)(); // error
```

Alas, this kind of recursive typedef isn't allowed. Some people, thinking that the type system is just getting in the way, try to do an end run around the type system "nuisance" by casting to and from void*:

``` cpp
//  Example 3(b): A nonstandard and
//                nonportable hack

typedef void* (*FuncPtr)();
void* f() { return (void*)f; }  // cast to void*
FuncPtr p = (FuncPtr)(f());     // cast from void*
p();
```

This isn't a solution because it doesn't satisfy the criteria of the question. Worse, it is dangerous because it deliberately defeats the type system, and it is onerous because it forces all users of `f()` to write casts. Worst of all, Example 3(b) is not supported by the standard. Although a `void*` is guaranteed to be big enough to hold the value of any object pointer, it is not guaranteed to be suitable to hold a function pointer; for example, on some platforms a function pointer is larger than an object pointer. Even if code like Example 3(b) happens to work on the compiler you're using today, it's not portable and may not work on another compiler, or even on the next release of your current compiler.

Of course, one way around this is to cast to and from another function pointer type instead of a simple void*:

``` cpp
//  Example 3(c): Standard and portable, but
//                nonetheless still a hack

typedef void (*VoidFuncPtr)();
typedef VoidFuncPtr (*FuncPtr)();
VoidFuncPtr f() { return (VoidFuncPtr)f; }
                          // cast to VoidFuncPtr
FuncPtr p = (FuncPtr)f(); // cast from VoidFuncPtr
p();
```

This code is technically legal, but it has all but one of the major disadvantages of Example 3(b): It is a dangerous hack because it deliberately defeats the type system. It imposes an unacceptable burden on all users of f() by requiring them to write casts. And, of course, it's still not really a solution at all because doesn't meet the requirements of the problem.

Can we really do no better?

**A Correct and Portable Way**
Fortunately, we can indeed get exactly the intended effect required by Question #3 in a completely type-safe and portable way, without relying on nonstandard code or type-unsafe casting. The way to do it is to use a proxy class that takes, and has an implicit conversion to, the desired pointer type:
``` cpp
//  Example 3(d): The correct solution

struct FuncPtr_;
typedef FuncPtr_ (*FuncPtr)();
struct FuncPtr_
{
    FuncPtr_( FuncPtr pp ) : p( pp ) { }
    operator FuncPtr() { return p; }
    FuncPtr p;
};
```

Now we can declare, define, and use f() naturally:

``` cpp
FuncPtr_ f() { return f; } // natural return syntax
int main()
{
    FuncPtr p = f();  // natural usage syntax
    p();
}
```

This solution has three main strengths:

1. It solves the problem as required. Better still, it's type-safe and portable.

2. Its machinery is transparent: You get natural syntax for the caller/user, and natural syntax for the function's own "return myname;" statement.

3. It probably has zero overhead: On modern compilers, the proxy class, with its storage and functions, should inline and optimize away to nothing.

**Coda**
Of course, normally a special-purpose FuncPtr_ proxy class like this (that contains some old object and doesn't really care much about its type) just cries out to be templatized into a general-purpose Holder proxy. Alas, you can't just templatize the FuncPtr_ class above, because then the typedef would have to look something like:

  typedef Holder<FuncPtr> (*FuncPtr)();
which is self-referential.



# 058 Nested Functions 
**Difficulty: 4 / 10**
C++ has nested classes, but not nested functions. When might nested functions be useful, and can they be simulated in C++?

----

**Problem**
**JG Questions**
1. What is a nested class? Why can it be useful?

2. What is a local class? Why can it be useful?

**Guru Question**
3. C++ does not support nested functions. That is, we cannot write something like:
``` cpp
//  Example 3
//
int f( int i )
{
    int j = i*2;
    
    int g( int k )
    {
      return j+k;
    }
    
    j += 4;
    return g( 3 );
}
```

Demonstrate how it is possible to achieve the same effect in standard C++, and show how to generalize the solution.

----

**Solution**
**Recap: Nested and Local Classes**
C++ provides many useful tools for information hiding and dependency management. As we recap nested classes and local classes, don't fixate too much on the syntax and semantics; rather, focus on how these features can be used to write solid, maintainable code and express good object designs \-\- designs that prefer weak coupling and strong cohesion.

**1. What is a nested class? Why can it be useful?**

A nested class is a class enclosed within the scope of another class. For example:
``` cpp
//  Example 1: Nested class

class OuterClass
{
    class NestedClass
    {
      // ...
    };
    // ...
};
```

Nested classes are useful for organizing code and controlling access and dependencies. Nested classes obey access rules just like other parts of a class do; so, in Example 1, if NestedClass is public then any code can name it as OuterClass::NestedClass. Often nested classes contain private implementation details, and are therefore made private; in Example 1, if NestedClass is private, then only OuterClass's members and friends can use NestedClass.

Note that you can't get the same effect with namespaces alone, because namespaces merely group names into distinct areas. Namespaces do not by themselves provide access control, whereas classes do. So, if you want to control access rights to a class, one tool at your disposal is to nest it within another class.

**2. What is a local class? Why can it be useful?**

A local class is a class defined within the scope of a function \-\- any function, whether a member function or a free function. For example:

``` cpp
  //  Example 2: Local class
  //
  int f()
  {
    class LocalClass
    {
      // ...
    };
    // ...
  };
```

Like nested classes, local classes can be a useful tool for managing code dependencies. In Example 2, only the code within f() itself is able to use LocalClass, which can be useful when LocalClass is, say, an internal implementation detail of f() that should never be exposed to other code.

You can use a local class in most places where you can use a nonlocal class, but there is a major restriction to keep in mind: A local or unnamed class cannot be used as a template parameter. From the C++ standard (14.3.1/2):

    A local type, a type with no linkage, an unnamed
    type or a type compounded from any of these types
    shall not be used as a template-argument for a
    template type-parameter.  \[Example:
    
    ``` cpp
     template <class T>
     class X { /* ... */ };
     void f()
     {
       struct S { /* ... */ };
       X<S> x3;  // error: local type used as
                 //  template-argument
       X<S*> x4; // error: pointer to local type
                 //  used as template-argument
     }
     ```
     
    \-\-end example]
Both nested classes and local classes are among C++'s many useful tools for information hiding and dependency management.

**Nested Functions: Overview**
Some languages (but not C++) allow nested functions, which are similar in concept to nested classes. A nested function is defined inside another function (the "enclosing function"), such that:
- the nested function has access to the enclosing function's variables; and

- the nested function is local to the enclosing function, that is, it can't be called from elsewhere unless the enclosing function gives you a pointer to the nested function.

Just as nested classes can be useful because they help control the visibility of a class, nested functions can be useful because they help control the visibility of a function.

C++ does not have nested functions. But can we get the same effect? This brings us to the question:

**3. C++ does not support nested functions. That is, we cannot write something like:**

``` cpp
//  Example 3

int f( int i )
{
    int j = i*2;
    int g( int k )
    {
        return j+k;
    }
    
    j += 4;
    return g( 3 );
}
```

Demonstrate how it is possible to achieve the same effect in standard C++, and show how to generalize the solution.

Short answer: Yes, it is possible, with only a little code reorganization and a minor limitation. The basic idea is to turn a function into a functor \-\- and this discussion, not coincidentally, will also serve to nicely illustrate some of the power of functors.

Attempts at Simulating Nested Functions in C++
To solve a problem like Question #3 above, most people start out by trying something like the following:

``` cpp
//  Example 3(a): Naive "local functor"
//                approach (doesn't work)

int f( int i )
{
    int j = i*2;
    
    class g_
    {
    public:
        int operator()( int k )
        {
          return j+k;   // error: j isn't accessible
        }
    } g;
    
    j += 4;
    return g( 3 );
}
```

In Example 3(a), the idea is to wrap the function in a local class, and call the function through a functor object.

It's a nice idea, but it doesn't work for a simple reason: The local class object doesn't have access to the enclosing function's variables.

"Well," one might say, "why don't we just give the local class pointers or references to all of the function's variables?" Indeed, that's usually the next attempt:

``` cpp
//  Example 3(b): Naive "local functor plus
//                references to variables"
//                approach (complex, fragile)

int f( int i )
{
    int j = i*2;
    
    class g_
    {
    public:
        g_( int& j ) : j_( j ) { }
        int operator()( int k )
        {
          return j_+k;  // access j via a reference
        }
    private:
        int& j_;
    } g( j );
    
    j += 4;
    return g( 3 );
}
```

Well, all right, I have to admit that this "works"... but only barely. This solution is fragile, difficult to extend, and can rightly be considered a hack. For example, consider that just to add a new variable requires four changes:

    a) add the variable;
    
    b) add a corresponding private reference to g_;
    
    c) add a corresponding constructor parameter to g_; and
    
    d) add a corresponding initialization to g_::g_().

That's not very maintainable. It also isn't easily extended to multiple local functions. Couldn't we do better?

An Somewhat Improved Solution
We can do better by moving the variables themselves into the local class:

``` cpp
//  Example 3(c): A better solution

int f( int i )
{
    class g_
    {
    public:
        int j;
        int operator()( int k )
        {
          return j+k;
        }
    } g;
    
    g.j = i*2;
    g.j += 4;
    return g( 3 );
}
```

This is a solid step forward. Now g_ doesn't need pointer or reference data members to point at external data, it doesn't need a constructor, and everything is much more natural all around. Note also that the technique is now extensible to any number of local functions, so let's add some more local functions called x(), y(), and z() while we're at it, and let's rename g_ to be more indicative of what the local class is actually doing:

``` cpp
//  Example 3(d): Nearly there!

int f( int i )
{
    //  Define a local functor that wraps all
    //  local data and functions.

    class Local_
    {
    public:
        int j;
        // All local functions go here:
        //
        int g( int k )
        {
          return j+k;
        }
        void x() { /* ... */ }
        void y() { /* ... */ }
        void z() { /* ... */ }
    } local;

    local.j = i*2;
    local.j += 4;
    local.x();
    local.y();
    local.z();
    return local.g( 3 );
}
```

This still has the problem that, if you need to initialize j using something other than default initialization, you have to add a clumsy constructor to the local class just to pass through the initialization value. The original question tried to initialize j to the value of `i*2`; here, we've had to create `j` and then assign the value, which could be difficult for more complex types.

A Complete Solution
If you don't need f itself to be a bona fide function (e.g., if you don't take pointers to it), you can turn the whole thing into a functor and elegantly support non-default initialization into the bargain:

``` cpp
//  Example 3(e): A complete and nicely
//                extensible solution

class f
{
    int  retval; // f's "return value"
    int  j;
    int  g( int k ) { return j + k; };
    void x() { /* ... */ }
    void y() { /* ... */ }
    void z() { /* ... */ }
public:
    f( int i )   // original function, now a constructor
    : j( i*2 )
    {
        j += 4;
        x();
        y();
        z();
        retval = g( 3 );
    }
    operator int() // returning the result
    {
        return retval;
    }
};
```

The code is shown inline for brevity, but all of the private members could also be hidden behind a Pimpl, giving the same full separation of interface from implementation that we had with the original simple function.

Note that this approach is easily extensible to member functions. For example, suppose that f() was a member function instead of a free function, and we would like to write a nested function g() inside f() as follows:

``` cpp
//  Example 4: This isn't legal C++, but it
//             illustrates what we want: a local
//             function inside a member function

class C
{
    int data_;
public:
    int f(int i)
    {
        // a hypothetical nested function
        int g( int i ) { return data_ + i; }
        return g( data_ + i*2 );
    }
};
```

We can express this by turning f() into a class as demonstrated in Example 3(e), except that whereas in Example 3(e) the class was at global scope it is now instead a nested class and accessed via a passthrough helper function:

``` cpp
//  Example 4(a): The complete and nicely
//                extensible solution, now
//                applied to a member function

class C
{
    int data_;
    friend class C_f;
public:
    int f( int i );
};

class C_f
{
    C*  self;
    int retval;
    int g( int i ) { return self->data_ + i; }
public:
    C_f( C* c, int i ) : self( c )
    {
      retval = g( self->data_ + i*2 );
    }
    operator int() { return retval; }
};

int C::f( int i ) { return C_f( this, i ); }
```

**Summary**
This approach illustrated above in Examples 3(e) and 4(a) simulates most of the features of local functions, and is easily extensible to any number of local functions. Its main weakness is that it requires that all variables be defined at the start of the 'function,' and doesn't (easily) allow us to declare variables closer to point of first use. It's also more tedious to write than a local function would be, and clearly a workaround for a language limitation. But, in the end, it's not bad, and it demonstrates some of the power inherent in functors over plain functions.

The purpose of this GotW wasn't just to learn about how nice it might be to have local functions in C++. It was, as always, about setting a specific design problem, exploring alternative methods for solving it, and weighing the solutions to pick the best-engineered option. Along the way, we also got to experiment with various C++ features and see through use what makes functors useful.

When designing your programs, strive for simplicity and elegance wherever possible. Some of the intermediate above solutions would arguably "work," yet should never be allowed to see the light of day in production code because they are complex, difficult to understand, and therefore difficult and expensive to maintain.

Simpler designs are easier to code and test. Avoid complex solutions, which are almost certain to be fragile and harder to understand and maintain. While being careful not to fall into the trap of over engineering, do recognize that even if coming up with a judicious simple design takes a little extra time up front, the long-term benefits usually make that time well spent. Often putting in a little time now means saving more time later \-\- and most people would agree that the "more time later" is better spent with the family at the cottage than slaving away over a hot keyboard trying to find those last few bugs in a complicated rat's nest of code.



# 059 Exception-Safe Class Design, Part 1: Copy Assignment 
**Difficulty: 7 / 10**
Is it possible to make any C++ class strongly exception-safe, for example for its copy assignment operator? If so, how? What are the issues and consequences?

----

**Problem**
**JG Questions**
**1.** What are the three common levels of exception safety? Briefly explain each one and why it is important.

**2.** What is the canonical form of strongly exception-safe copy assignment?

**Guru Questions**
**3.** Consider the following class:

``` cpp
//  Example 3: The Cargill Widget Example

class Widget
{
    // ...
private:
    T1 t1_;
    T2 t2_;
};
```
Assume that any T1 or T2 operation might throw. Without changing the structure of the class, is it possible to write a strongly exception-safe Widget::operator=( const Widget& )? Why or why not? Draw conclusions.

**4.** Describe and demonstrate a simple transformation that works on any class in order to make strongly exception-safe copy assignment possible and easy for that class. Where have we seen this transformation technique before in other contexts? Cite GotW issue number(s).

----

**Solution**
**Review: Exception Safety Canonical Forms**
**1. What are the three common levels of exception safety? Briefly explain each one and why it is important.**

The canonical Abrahams Guarantees are as follows.

1. Basic Guarantee: If an exception is thrown, no resources are leaked, and objects remain in a destructible and usable \-\- but not necessarily predictable \-\- state. This is the weakest usable level of exception safety, and is appropriate where client code can cope with failed operations that have already made changes to objects' state.

2. Strong Guarantee: If an exception is thrown, program state remains unchanged. This level always implies global commit-or-rollback semantics, including that no references or iterators into a container be invalidated if an operation fails.

In addition, certain functions must provide an even stricter guarantee in order to make the above exception safety levels possible:

3. Nothrow Guarantee: The function will not emit an exception under any circumstances. It turns out that it is sometimes impossible to implement the strong or even the basic guarantee unless certain functions are guaranteed not to throw (e.g., destructors, deallocation functions). As we will see below, an important feature of the standard auto_ptr is that no auto_ptr operation will throw.

**2. What is the canonical form of strongly exception-safe copy assignment?**

It involves two steps: First, provide a nonthrowing Swap() function that swaps the guts (state) of two objects:

``` cpp
void T::Swap( T& other ) throw()
{
    // ...swap the guts of *this and other...
}
```

Second, implement operator=() using the "create a temporary and swap" idiom:

``` cpp
T& T::operator=( const T& other )
{
    T temp( other ); // do all the work off to the side
    Swap( temp );    // then "commit" the work using
    return *this;    //  nonthrowing operations only
}
```

**The Cargill Widget Example**
This brings us to the Guru questions, starting with a new exception safety challenge proposed by Tom Cargill:

**3. Consider the following class:**
``` cpp
//  Example 3: The Cargill Widget Example

class Widget
{
    // ...
private:
    T1 t1_;
    T2 t2_;
};
```
Assume that any T1 or T2 operation might throw. Without changing the structure of the class, is it possible to write a strongly exception-safe Widget::operator=( const Widget& )? Why or why not? Draw conclusions.

Short answer: In general, no, it can't be done without changing the structure of Widget.

In the Example 3 case, it's not possible to write a strongly exception-safe Widget::operator=() because there's no way that we can change the state of both of the t1_ and t2_ members atomically. Say that we attempt to change t1_, then attempt to change t2_. The problem is twofold:

1. If the attempt to change t1_ throws, t1_ must be unchanged. That is, to make Widget::operator=() strongly exception-safe relies fundamentally on the exception safety guarantees provided by T1, namely that T1::operator=() (or whatever mutating function we are using) either succeeds or does not change its target. This comes close to requiring the strong guarantee of T1::operator=(). (The same reasoning applies to T2::operator=().)

2. If the attempt to change t1_ succeeds, but the attempt to change t2_ throws, we've entered a "halfway" state and cannot in general roll back the change already made to t1_.

Therefore, the way Widget is currently structured, its operator=() cannot be made strongly exception-safe.

    Note also that Cargill's Widget Example isn't all that different from the following simpler case:
    
    ``` cpp
    class Widget2  
    {              
        // ...       
    private:       
        T1 t1_;      
    };
    ```
    
    In the above code, problem #1 above still exists. If T1::operator=() can throw in such a way that it has already started to modify the target, there is no way to write a strongly exception-safe Widget2::operator=() unless T1 provides suitable facilities through some other function (but if T1 can do that, why doesn't it for T1::operator=()?).

Our goal: To write a Widget::operator=() that is strongly exception-safe, without making any assumptions about the exception safety of any T1 or T2 operation. Can it be done? Or is all lost?

**A Complete Solution: Using the Pimpl Idiom**
The good news is that, even though Widget::operator=() can't be made strongly exception-safe without changing Widget's structure, the following simple transformation always works:

**4. Describe and demonstrate a simple transformation that works on any class in order to make strongly exception-safe copy assignment possible and easy for that class. Where have we seen this transformation technique before in other contexts? Cite GotW issue number(s).**

The way to solve the problem is hold the member objects by pointer instead of by value, preferably all behind a single pointer with a Pimpl transformation (described in GotW issues like 7, 15, 24, 25, and 28).

Here is the canonical Pimpl exception-safety transformation:
``` cpp
//  Example 4: The canonical solution to
//             Cargill's Widget Example

class Widget
{
    Widget();  // initializes pimpl_ with new WidgetImpl
    ~Widget(); // must be provided, because the implicit
               //  version causes usage problems (see GotW #62)
    // ...
private:
    class WidgetImpl;
    auto_ptr<WidgetImpl> pimpl_;
    // ... provide copy construction and assignment
    //     that work correctly, or suppress them ...
};

class Widget::WidgetImpl
{
public:
    // ...
    T1 t1_;
    T2 t2_;
};
```
Now we can easily implement a nonthrowing Swap(), which means we can easily implement exception-safe copy assignment: First, provide the nonthrowing Swap() function that swaps the guts (state) of two objects (note that this function can provide the nothrow guarantee because no auto_ptr operations are permitted to throw exceptions):
``` cpp
void Widget::Swap( Widget& other ) throw()
{
    auto_ptr<WidgetImpl> temp( pimpl_ );
    pimpl_ = other.pimpl_;
    other.pimpl_ = temp;
}
```
Second, implement the canonical form of operator=() using the "create a temporary and swap" idiom:
``` cpp
Widget& Widget::operator=( const Widget& other )
{
    Widget temp( other ); // do all the work off to the side
    Swap( temp );    // then "commit" the work using
    return *this;    //  nonthrowing operations only
}
```
**A Potential Objection, and Why It's Unreasonable**
Some may object: "Aha! Therefore this proves exception safety is unattainable in general, because you can't solve the general problem of making any arbitrary class strongly exception-safe without changing the class!"

Such a position is unreasonable and untenable. The Pimpl transformation, a minor structural change, IS the solution to the general problem. To say, "no, you can't do that, you have to be able to make an arbitrary class exception-safe without any changes," is unreasonable for the same reason that "you have to be able to make an arbitrary class meet New Requirement #47 without any changes" is unreasonable.

For example:

Unreasonable Statement #1: "Polymorphism doesn't work in C++ because you can't make an arbitrary class usable in place of a Base& without changing it (to derive from Base)."

Unreasonable Statement #2: "STL containers don't work in C++ because you can't make an arbitrary class usable in an STL container without changing it (to provide an assignment operator)."

Unreasonable Statement #3: "Exception safety doesn't work in C++ because you can't make an arbitrary class strongly exception-safe without changing it (to put the internals in a Pimpl class)."

Clearly all the above arguments are equally bankrupt, and the Pimpl transformation is indeed the general solution to strongly exception-safe objects.

So, what have we learned?

**Conclusion 1: Exception Safety Affects a Class's Design**
Exception safety is never "just an implementation detail." The Pimpl transformation is a minor structural change, but still a change. GotW #8 shows another example of how exception safety considerations can affect the design of a class's member functions.

**Conclusion 2: You Can Always Make Your Code (Nearly) Strongly Exception-Safe**
There's an important principle here:

   > Just because a class you use isn't in the least exception-safe is no reason that YOUR code that uses it can't be (nearly) strongly exception-safe.

Anybody can use a class that lacks a strongly exception-safe copy assignment operator and make that use exception-safe. The "hide the details behind a pointer" technique can be done equally well by either the Widget implementor or the Widget user... it's just that if it's done by the implementor it's always safe, and the user won't have to do this:

``` cpp
class MyClass
{
    auto_ptr<Widget> w_; // hold the unsafe-to-copy
                         //  Widget at arm's length
public:
    void Swap( MyClass& other ) throw()
    {
        auto_ptr<Widget> temp( w_ );
        w_ = other.w_;
        other.w_ = temp;
    }
    MyClass& operator=( const MyClass& other )
    {
        /* canonical form */
    }
    // ... destruction, copy construction,
    //     and copy assignment ...
};
```

**Conclusion 3: Use Pointers Judiciously**
To quote Scott Meyers:

   > "When I give talks on EH, I teach people two things:

   > "- POINTERS ARE YOUR ENEMIES, because they lead to the kinds of problems that auto_ptr is designed to eliminate.

To wit, bald pointers should normally be owned by manager objects that own the pointed-at resource and perform automatic cleanup. Then Scott continues:

   > "- POINTERS ARE YOUR FRIENDS, because operations on pointers can't throw.

   > "Then I tell them to have a nice day :-)

Scott captures a fundamental dichotomy well. Fortunately, in practice you can and should get the best of both worlds:

- USE POINTERS BECAUSE THEY ARE YOUR FRIENDS, because operations on pointers can't throw.

- KEEP THEM FRIENDLY BY WRAPPING THEM IN MANAGER OBJECTS like auto_ptrs, because this guarantees cleanup. This doesn't compromise the nonthrowing advantages of pointers because auto_ptr operations never throw either (and you can always get at the real pointer inside an auto_ptr whenever you need to).

Indeed, often the best way to implement the Pimpl idiom is exactly as shown in Example 4 above, by using a pointer (in order to take advantage of nonthrowing operations) while still wrapping the dynamic resource safely in an auto_ptr manager object. Just remember that now your object must provide its own copy construction and assignment with the right semantics for the auto_ptr member, or disable them if copy construction and assignment don't make sense for the class.



# 060 Exception-Safe Class Design, Part 2: Inheritance 
**Difficulty: 7 / 10**
What does IS-IMPLEMENTED-IN-TERMS-OF mean? It may surprise you to learn that there are definite exception-safety consequences when choosing between inheritance and delegation. Can you spot them?

----

**Problem**
**JG Question**
1. What does IS-IMPLEMENTED-IN-TERMS-OF mean?

**Guru Question**
2. In C++, IS-IMPLEMENTED-IN-TERMS-OF can be expressed by either nonpublic inheritance or by containment/ delegation. That is, when writing a class T that is implemented in terms of a class U, the two main options are to either inherit privately from U or to contain a U member object.

Does the choice between these techniques have exception safety implications? Explain. (Ignore any issues not related to exception safety.)

----

**Solution**
**IS-IMPLEMENTED-IN-TERMS-OF**
**1. What does IS-IMPLEMENTED-IN-TERMS-OF mean?**

A type T is IS-IMPLEMENTED-IN-TERMS-OF (IIITO) type U if T uses U in its implementation in some form. This can run the gamut from T being an adapter or proxy or wrapper for U, to T simply using U incidentally to implement some details of T's own services.

Typically "T IIITO U" means that either T HAS-A U:

``` cpp
//  Example 1(a): IIITO using HAS-A

class T
{
    // ...
private:
    U* u_;  // or by value or by reference
};
```

or that T is derived from U nonpublicly:

``` cpp
//  Example 1(b): IIITO using derivation

class T : private U
{
    // ...
};
```

Arguably, public derivation also models IIITO incidentally, but the primary meaning of public derivation is IS-A (in the sense of LSP, IS-SUBSTITUTABLE-FOR-A).

**Inheritance vs. Delegation**
**2. In C++, IS-IMPLEMENTED-IN-TERMS-OF can be expressed by either nonpublic inheritance or by containment/ delegation. That is, when writing a class T that is implemented in terms of a class U, the two main options are to either inherit privately from U or to contain a U member object.**

As I've argued before:[1] [2]

"Inheritance is often overused, even by experienced developers. Always minimize coupling: If a class relationship can be expressed in more than one way, use the weakest relationship that's practical. Given that inheritance is nearly the strongest relationship you can express in C++ (second only to friendship), it's only really appropriate when there is no equivalent weaker alternative."

"If you can express a class relationship using containment alone, you should always prefer that. If you need inheritance but aren't modeling [Liskov] IS-A, use nonpublic inheritance."

It turns out that the above "minimize coupling" principle also relates to exception safety, because a design's coupling has a direct impact on its possible exception safety.

**Exception Safety Consequences**
Does the choice between these techniques have exception safety implications? Explain. (Ignore any issues not related to exception safety.)

The short answer is this: Lower coupling promotes program correctness (including exception safety), and tight coupling reduces the maximum possible program correctness (including exception safety).

This state of affairs shouldn't be surprising. After all, the less tightly real-world objects are related, the less effect they necessarily have on each other. That's why we put firewalls in buildings and bulkheads in ships; if there's a failure in one compartment, the more we've isolated the compartments the less likely the failure is to spread to other compartments before things can be brought back under control.

Now consider against a class `T` that is IIITO another type U. How does the choice of how to express the IIITO relationship affect how we write `T::operator=()`? First, consider HAS-A:

``` cpp
//  Example 2(a): IIITO using HAS-A

class T
{
    // ...
private:
    U* u_;
};

T& T::operator=( const T& other )
{
    U* temp = new U( *other.u_ );   // do all the work
                                    //  off to the side
    delete u_;      // then "commit" the work using
    u_ = temp;      //  nonthrowing operations only
    return *this;
}
```

Therefore we can write a "nearly" strongly exception-safe `T::operator=()`... and we haven't made any assumptions at all about `U`. (See the next GotW for more about what "nearly strongly exception-safe" actually means.)

Even if the `U` object were contained by value instead of by pointer, it could be easily transformed into being held by pointer as above. The `U` object could also be put into a Pimpl using the transformation described in GotW #59. It is precisely the fact that containment (HAS-A) gives us this flexibility that allows us to easily write an exception-safe `T::operator=()` without making any assumptions about U.

Consider next how the problem changes once the relationship between `T` and `U` involves any kind of inheritance:

``` cpp
//  Example 2(b): IIITO using derivation

class T : private U
{
    // ...
};
T& T::operator=( const T& other )
{
    U::operator=( other );  // ???
    return *this;
}
```

The problem is the call to `U::operator=()`. As noted in GotW #59, if `U::operator=()` can throw in such a way that it has already started to modify the target, there is no way to write a strongly exception-safe `T::operator=()` unless `U` provides suitable facilities through some other function (but if `U` can do that, why doesn't it for `U::operator=()`?).

In other words, now T's ability to make an exception safety guarantee for its `T::operator=()` depends implicitly on `U`'s own safety and guarantees. But, really, should this surprise us? No, it shouldn't, because Example 2(b) uses the tightest possible relationship, and hence the highest possible coupling, between `T` and `U`.

**Summary**
Lower coupling promotes program correctness (including exception safety), and tight coupling reduces the maximum possible program correctness (including exception safety).

Inheritance is often overused, even by experienced developers. See the references[1] [2] for more information about many other reasons (besides exception safety) why and how you should use delegation instead of inheritance wherever possible. Always minimize coupling: If a class relationship can be expressed in more than one way, use the weakest relationship that's practical. In particular, never use inheritance except where containment/delegation won't suffice.

**Notes**
1. H. Sutter. ["Uses and Abuses of Inheritance, Part 1"](http://www.gotw.ca/publications/mill06.htm) and ["Uses and Abuses of Inheritance, Part 2"](http://www.gotw.ca/publications/mill07.htm) (C++ Report, October 1998 and January 1999).

2. H. Sutter. Exceptional C++, Item 24 (Addison-Wesley, 2000)



# 061 CHALLENGE EDITION: ACID Programming 
**Difficulty: 10 / 10**
What are the similarities and differences in reasoning about exception safety in particular, and program safety in general? Is it possible to generalize an approach that encompasses general program correctness analysis, both in C++ and in other languages? And what guidelines can we extrapolate that will help improve our day-to-day programming? For answers, see this first-ever "10/10 difficult" issue of GotW.

----

**Problem**
**Preface**
In this issue of GotW, I want to suggest a new approach to program correctness analysis that covers both "normal" correctness (i.e., in the absence of exceptions) and exception safety analysis in C++ and other languages. Hopefully it is useful, and if so then it will doubtless be subject to refinement; no one ever bothers to refine unuseful ideas.

The purpose of this GotW is to define a taxonomy to better describe exception safety, and to see whether it can provide us with new ways of reasoning about program correctness. If successful, the method should meet the following goals:
   - It should be inclusive of existing work, covering existing exception safety analysis methods.
   
   - It should help to fill holes in existing methods, such as a situation where given sample code was known to support some exception-safety guarantee but that guarantee could not be adequately described using the existing models alone.
   
   - It should help us reason about and analyze the safety of code, making it less a craft and more a methodical engineering task.
This brings us to the GotW questions:

**JG Questions**
1. Consider again the Abrahams exception safety guarantees, as frequently described in GotW, and any others that you may know of. Then, review the technique described in GotW #59 (Question #4 and Conclusion #2). Which Abrahams (or other) guarantee does that technique provide? Discuss.

2. In the context of databases, what do the following terms mean:
   a) transaction

   b) ACID

**Guru Questions**
3. The key insight is to notice that ACID transactional analysis can be applied to program correctness, including C++ exception safety. Demonstrate a parallel between the two by showing a programming analog for each aspect of ACID.

4. Treat the analysis from Question #3 as a taxonomy and use it to describe the Abrahams and other prior guarantees. Discuss.

5. Does the answer from Question #4 equip us to give a better answer to Question #1? Explain.

----

**Solution**
**INTRODUCTION**
**Exception Safety Recap**

**1. Consider again the Abrahams exception safety guarantees, as frequently described in GotW, and any others that you may know of.**

Recapping from GotW #59, Dave Abrahams' guarantees are usually stated as follows.

    A1. Basic Guarantee: If an exception is thrown, no resources are leaked, and objects remain in a destructible and usable \-\- but not necessarily predictable \-\- state. This is considered by many to be the weakest usable exception safety guarantee, and is appropriate where client code can cope with failed operations that have already made changes to objects' state.

A point I want to make here is that this essentially means "there are no bugs." It's not much different from saying: "If an error occurs, no resources are leaked, and objects remain in a destructible and usable \-\- not not necessarily predictable \-\- state."

For example, consider a Container::AppendElements() operation that appends multiple elements onto the end of an existing container. If called to append 10 elements, and it successfully adds 6 elements before encountering an error and stopping, then the operation provides only the basic guarantee: The container is still in a consistent state (it still meets the Container invariants, and further operations on the container will work fine), but the state is not necessarily predictable (the caller probably has no way to know in advance what the final state will be if an error is encountered; it may or may not be the initial state).

    A2. Strong Guarantee: If an exception is thrown, program state remains unchanged. This guarantee always implies global commit-or-rollback semantics, including that no references or iterators into a container be invalidated if an operation fails.

Note that here we are already heading toward transactional semantics.

In addition, certain functions must provide an even stricter guarantee in order to make the above exception safety guarantees possible in higher functions:

    A3. Nothrow Guarantee: The function will not emit an exception under any circumstances. It turns out that it is sometimes impossible to implement the strong or even the basic guarantee unless certain functions are guaranteed not to throw (e.g., destructors, deallocation functions). For example, an important feature of the standard auto_ptr is that no auto_ptr operation will throw.

Other experts have also proposed exception safety guarantees. Building on earlier work by Harald Mueller[2], Jack Reeves[3] suggested three guarantees regarding the state of an object after an exception had occurred:

    R1. Good: The object meets its invariants.
    
    R2. Bad: The object has been corrupted in that it no longer meets its invariants, and the only thing that you are guaranteed to be able to safely do with it is destroy it.
    
    R3. Undefined: The object is throughly corrupted and cannot be safely used or even destroyed.

Over the years, other possible guarantees have been suggested, notably the "destructible guarantee," which is another way of saying that if an exception occurs then an object is in Reeves' "Bad" state.

**Analyzing a Motivating Example**
Then, review the technique described in GotW #59 (Question #4 and Conclusion #2). Which Abrahams (or other) guarantee does that technique provide? Discuss.

In GotW #59, the main technique presented was that a class contain its member objects by auto_ptr<Pimpl> rather than directly by value:

``` cpp
// The general solution to
// Cargill's Widget Example

class Widget
{
    // ...
private:
    class Impl;
    auto_ptr<Impl> pimpl_;
    // ... provide destruction, copy construction
    // and assignment that work correctly, or
    // suppress them ...
};

class Widget::Impl
{
public:
    // ...
    T1 t1_;
    T2 t2_;
};

void Widget::Swap( Widget& other ) throw()
{
    auto_ptr<Impl> temp( pimpl_ );
    pimpl_ = other.pimpl_;
    other.pimpl_ = temp;
}

Widget& Widget::operator=( const Widget& other )
{
    Widget temp( other ); // do all work off to the side
    
    Swap( temp ); // then "commit" the work using
    return *this; // nonthrowing operations only
}
```

See GotW #59 for further details.

Now we get to the crux of the matter: That GotW then made the claim that the above Widget::operator=() provides the strong guarantee without any exception-safety requirements on T1 and T2. As Dave Abrahams has pointed out, this isn't quite true: For example, if constructing a T1 or T2 object causes side effects (such as changing a global variable or launching a rocket) and then throws an exception, and that side effect is not cancelled by the corresponding T1 or T2 destructor, then Widget::operator=() has caused a side effect and so doesn't fully meet the requirements of the strong guarantee. (It does fully meet the requirements of the strong guarantee in all other ways besides the possibility of side effects.)

So here's the rub: The above solution to Cargill's Widget Example is an important technique, it does provide a safety guarantee, and that guarantee appears to be a fundamental concept \-\- it is the strongest exception safety guarantee one can make for assignment without requiring any assumptions at all about the used type (more on this below). Better still, it is a useful guarantee to be able to make; for example, it is useful to know that upon failure a change to a container object will produce no local effects on the container, even though it might invalidate existing iterators into the container. More on this in the final section.

So, since it is a guarantee, what guarantee is it? It doesn't meet the requirements of the Abrahams strong or nothrow guarantees, but at the same time it seems to give more than just the Abrahams basic guarantee or the Reeves Good guarantee.

There's got to be a name for a concept as useful and as fundamental as "the strongest guarantee you can make without requiring assumptions about the used type" \-\- and that's not a name, that's just an explanation. It bothered me greatly that we had no succinct name for it.

**TOWARD TRANSACTIONAL PROGRAMMING **
**Ideas From the Database World**
Could another area of computer science yield any insights that would help? To introduce the answer, let's take a quick trip into the database world and its existing methods of analyzing program correctness with respect to database manipulation:

**2. In the context of databases, what do the following terms mean:**

    a) transaction

A transaction is a unit of work. That is, it accomplishes some task (typically involving multiple database changes) in such a way that it meets the ACID requirements to a specified degree appropriate for its particular task:

    b) ACID

ACID is an acronym for a set of four requirements:

| Requirement |	Description (Databases)|
| ---- | ---- |
|Atomicity|The operation is indivisible, all-or- nothing. A transaction can be "committed" or "rolled back"; if it is committed all of its effects must complete fully, otherwise if it is rolled back it must have no effect.|
|Consistency|The transaction transforms the database from one consistent state to another consistent state. Here "consistent" implies obeying not just relational integrity constraints but also business rules and semantics, such as that a given Customer record's Address and ZIP fields must not specify inconsistent information (e.g., an address in New York and a ZIP code in Alabama).|
|Isolation|Two transactions are fully isolated if they can be started simultaneously and the result is guaranteed to be the same as though one was run entirely before the other ("Serializable"). In practice, databases support various levels of isolation, each of which trades off better isolation at the expense of concurrency or vice versa. Some typical isolation levels, in order from best to worst concurrency, are: Read Uncommitted (a.k.a. Dirty Read), Read Committed, Repeatable Read, and Serializable (highest isolation), and describe what kinds of intermediate states of other transactions might be visible.|
|Durability|If a transaction is committed, its effects are not lost even if there is a subsequent system crash. This requirement is usually implemented using a transaction log which is used to "roll forward" the state of the database on system restart.|

Note 1: Two transactions operating on nonoverlapping parts of the database are by definition always fully isolated.

The following notes show why isolation is disproportionately important:

Note 2: Isolation interacts with atomicity. For example, consider three transactions T1, T2, and T3 that run concurrently. All three are atomic. T1 and T2 run at Serializable isolation and do not interact with each other; that is, because of their isolation levels, T1 and T2 are also guaranteed to be atomic with respect to each other. T3, however, runs at Read Uncommitted isolation, and it is possible for T3 to see some but not all of the changes made by T1 and T2 before those other transactions commit; that is, because of its isomation level, T1 and T2 are NOT atomic as seen by T3.

Note 3: Because isolation interacts with atomicity, it also transitively interacts with consistency. This is because lower isolation levels may permit states to be seen by other transactions that would not otherwise be visible, and relying carelessly on such intermediate states can create inconsistencies in the database. For example, consider what could happen if transaction T4 observes some intermediate state of transaction T5, makes a change based on that observed state, but T5 is subsequently rolled back. Now T4 has made further changes based on phantom data that, as it turns out, never really existed.

For more information, see a standard database text like Date's An Introduction to Database Systems.[4]

In the database world, atomicity, consistency, and durability are always fully required in most real-world database systems. It is the isolation aspect that offers tradeoffs for concurrency (and, as noted, the tradeoffs can indirectly affect atomicity and consistency too). In practice, running at anything less than Serializable isolation opens windows wherein the database could get into a state that violates business rules (and hence the consistency requirement), but it's done all the time. In short, we live with this quite happily today, even in some financial database systems with stringent integrity requirements, because we can measure the exposure and we accept the tradeoff in order to achieve better concurrency. Nearly all production databases in the world run at less than the strongest isolation levels at least some of the time.

**Mapping ACID Concepts to Non-SQL Programming**
Note that in the following discussion, for an operation to "fail" can mean fail in any way, so that the analysis is the same whether the failure is reported by an error return code or by an exception.

**3. The key insight is to notice that ACID transactional analysis can be applied to program correctness, including C++ exception safety. Demonstrate a parallel between the two by showing a programming analog for each aspect of ACID.**

The elements of ACID were developed to describe requirements in database environments, but each has analogs that make sense in programming environments. Note that these analogs apply to reasoning about program correctness in general, both with and without exceptions and including exception support in other languages, not just to C++ exception handling.

The following analysis distinguishes between local effects and side effects:

    **1. Local effects** include all effects on the visible state of the immediate object(s) being manipulated. Here the visible state is defined as the state visible through member functions as well as free functions (e.g., operator<<()) that form part of the interface. For more on what constitutes the interface of a type, see the Interface Principle as described in my original IP article[5] and in Items 31-34 of Exceptional C++[6].
    
    **2. Side effects** include all other changes, specifically changes to other program states and structures as well as any changes caused outside the program (such as invalidating an iterator into a manipulated container, writing to a disk file, or launching a rocket), that affect the system's business rules, as they are expressed in object and system invariants. So, for example, a counter of the number of times a certain function is called would normally not be considered a side effect on the state of the system if the counter is simply a debugging aid; it would be a side effect on the state of the system if it recorded something like the number of transactions made in a bank account, because that affects business rules or program invariants. Note: In this article I use the terms "business rules" and "invariants" interchangeably.

Of course, if an effect is visible as a side effect but also through the manipulated object's interface (such as via ReadByte() or WasLaunched() functions), it is both a local effect and a side effect.

I propose that the following descriptions of the ACID requirements may be suitable for describing program correctness in general and exception safety in particular. They form four fundamental axes of ACID exception safety analysis:

|Requirement | Description (Programming) |
| ---- | ---- |
|Atomicity | The operation is indivisible, all-or- nothing, with respect to local effects. |
|Consistency | The operation transforms system from one consistent state to another consistent state. Here "consistent" implies obeying not just object integrity constraints (i.e., objects continue to fulfill their invariants) but also wider system invariants. Note "no \[memory or resource] leaks" is usually a required program invariant. |

Before tackling Isolation, a note: In the database world, isolation is often all about concurrency \-\- that is, access and manipulation of the same data elements by different transactions operating concurrently. The same concepts we use to describe database transaction isolation map directly onto multithreaded and multiprocess access and manipulation of shared resources; the same issues apply \-\- read locks, write locks, read/write locks, deadlocks, livelocks, and similar issues.

There is a difference, however, between relational databases and programs. Standard relational database systems provide direct support in the database engine to enforce isolation options such as "Read Committed" by automatically creating, maintaining, and freeing locks on individual data objects. They can do this because the engine "owns," or acts as an arbiter and access point for, all the data. No such support exists in most programming languages (including C++ and Java; Java has some thread safety primitives, but they are not very useful and do not actually solve the thread safety problem even in simple cases). In languages like C++ and Java, if you want locking you have to do it yourself in an application-specific way by manually writing calls to acquire and release locks on program resources like shared variables.

|Requirement | Description (Programming) |
| ---- | ---- |
|Isolation | Two operations are fully isolated if they can be started simultaneously and the result is guaranteed to be the same as though one was run entirely before the other ("Serializable"). In practice, programs implement various levels of isolation, each of which trades off better isolation at the expense of concurrency or vice versa. Some typical isolation levels, in order from best to worst concurrency, are: Read Uncommitted (a.k.a. Dirty Read), Read Committed, Repeatable Read, and Serializable (highest isolation), and describe what kinds of intermediate states of other operations might be visible. |

Interestingly, however, note that concurrency isolation issues only arise between transactions manipulating the same data; so, as already noted earlier, isolation is fundamentally concerned with not only concurrency, but interaction of side effects. As Note 1 above stated: two transactions operating on nonoverlapping parts of the database are by definition always fully isolated.

In the interests of space, this article does not pursue concurrency issues in depth (which would be a lengthy discussion beyond the immediate scope of the article), except to note the following: The ACID program analysis method proposed in this article can be directly extended to address concurrency issues in the same way that those issues are addressed in database environments, with the caveat that most programming languages do not provide the same native support provided inherently by relational database management systems for basic concurrency control and that these must therefore be implemented by the application. A later article will expand on this wider aspect of isolation.

This article then focuses specifically on the aspects of isolation of interest in a nonshared/serialized environment, namely the interaction of side effects:

|Requirement | Description (Programming)|
| ---- | ---- |
|Isolation (continued) | Regardless of threading and other (continued) concurrency issues, an operation on one or more objects is fully isolated if it does not cause side effects (i.e., only causes local effects on those objects). |
|Durability | If an operation completes successfully, all its effects are preserved even if there is a subsequent system crash. For example, a rocket once fired stays in flight even if the launching software in the submarine crashes afterwards.|

Here's a difference: In the database world, atomicity, consistency, and durability are always fully required and it is usually isolation that is traded off (with some indirect effects on atomicity and consistency). In the programming world, atomicity is normally assumed (but if there are asynchronous interrupts or exceptions, atomicity has to be explicity programmed for), consistency in terms of maintain invariants is also normally assumed, and durability is not often considered except in database-like applications, where the durability is made the responsibility of an underlying database or file system.

Really, it looks like what we are building up to is a transactional approach to program correctness, including C++ exception safety, arrived at by adapting concepts that are already well-understood in another area of computer science.

**A Notation for Safety Guarantees**
Next, let's analyze the possible statements we can make about each axis. For each axis, we can make one of three statements:
    a) that there is no guarantee at all (designated by a dash, '-', or simply omitting mention of the guarantee);
    
    b) a statement about effects if an operation may fail (designated by a lowercase letter, such as 'i'); or
    
    c) in addition to b), a statement about effects if an operation does not or cannot fail (designated by an uppercase letter, such as 'I').

Note that these are true "levels" in that each subsumes the one before it. I omit a fourth possible case ("a statement about effects if an operation succeeds \[but with no statement about what happens on failure]") because it is no better than a) for analyzing failure cases.

Here's what we get:

| Level | Description |
| ---- | ---- |
| Atomicity | |
| - | No guarantee. |
| a | If the operation fails, the manipulated object(s) are in their initial state, otherwise the manipulated object(s) are in their final state. |
| A | The operation is guaranteed to succeed and the manipulated object(s) are in their final state. (For exception- related failures, this is spelled "throw()" in C++; but an operation could still fail in ways other than throwing an exception.) |

Note that atomicity as defined here concerns itself only with local effects, that is, with the state of a directly manipulated object.

| Level | Description |
| ---- | ---- |
| Consistency | |
| - | No guarantee.|
| c | If the operation fails, the system is in a consistent state. (Depending whether or not the operation also makes some atomicity guarantee, the consistent state may be the initial state, the final state, or possibly an intermediate state, depending on whether and how a failure occurred.) |
| C | If the operation succeeds or fails, the system is in a consistent state. |

Any operation that can result in objects no longer meeting their invariants should be considered buggy. No program safety, let alone exception safety, is possible without a guarantee of consistency on success, so the rest of this article will generally rely on C. Further, we require that at the beginning of the operation the system is in a consistent state, else no reasoning about consistency is possible or useful.

| Level | Description |
| ---- | ---- |
| Isolation | |
| - | No guarantee. |
| i | If the operation fails, there will be no side effects. |
| I | If the operation either succeeds or fails, there will be no side effects. |

Note that isolation as defined here concerns itself only with side effects. In many ways it is a mirror of atomicity.

Note that `'a'+'i' => 'c'`, because we require that the objects and the system were initially in a consistent state.

| Level | Description |
| ---- | ---- |
| Durability | |
| - | No guarantee. |
| d | If the operation fails, whatever state was achieved will persist. |
| D | If the operation succeeds or fails, the state persists. |

**Interlude: FAQs and Objections**
At this point, several questions or objections arise, and it makes sense to pause briefly and handle them now:

Q: "I think 'c' and 'd' may not be useful, on the grounds that they seem backward: Shouldn't operations first specify whether they produce consistent and/or durable results if they succeed, and then optionally what they do if they fail?"

A: An interesting outgrowth of the above ACID view, then, is that it is actually more important first to state what operations do if they fail, and only then to state in addition what they do if they succeed. (I grant that programmers who find it annoying that their companies force them to document their functions' failure modes may also dislike the previous statement.)

Q: "The database version of 'atomicity' includes all effects, whereas the above doesn't. Is that right?"

A: Yes. It is useful in programming to distinguish between local effects and side effects, and the global atomicity guarantee still exists and is expressed as "both 'a' and 'i'."

This last point leads nicely into the question of combining guarantees:

**A Notation for Combining Guarantees**
The four axes of correctness analysis are not entirely independent; one can affect the other, as for example above 'a'+'i' => 'c'.

For notational convenience I will designate each combination of guarantee levels as a 4-tuple (such as "-CID" or "aC-D"), where each position specifies the level of the atomicity, consistency, isolation, and durability axis (respectively) that is guaranteed, if any. For convenience, the '-'s could also be omitted.

This article focuses primarily on the first three axes, so in the following discussion the durability guarantee level will often be shown as - or omitted. Because the durability axis is independent of the others, this can be done without loss of generality.

Note that, if the operation guarantees that it will always succeed (A), then it is not very meaningful to make a guarantee about what happens if the operation fails without a guarantee about what happens if the operation succeeds (c, i, or d). So, for example, in every case where ACi is guaranteed, AC is equivalent and should be specified instead.

**(RE)VIEWING EXISTING APPROACHES THROUGH THE LENS OF TRANSACTIONAL PROGRAMMING**
**Describing Existing Guarantees as ACID Guarantees**
The first goal of this taxonomy was that "\[i]t should be inclusive of existing work, covering existing exception safety analysis methods."

Hence Question #4:

**4. Treat the analysis from Question #3 as a taxonomy and use it to describe the Abrahams and other prior guarantees. Discuss.**

The ACID model does indeed provide a concise and explicit notation to describe the Abrahams guarantees, Reeves' Good, Bad, and Undefined states, and the destructible guarantee:
| ACID | Guarantee |
| ---- | ---- |
| ---- | Reeves Undefined state guarantee<br/>Reeves Bad state guarantee<br/>Destructible guarantee |
| -C\-\- | Abrahams basic guarantee<br/>Reeves Good state guarantee |
| aCi- | Abrahams strong guarantee |
| AC\-\- | Abrahams nothrow guarantee |

Note that this analysis points out at least two weaknesses in prior models:

First, the prior models generally ignored the issue of isolation. Note that none of the above-listed prior guarantees included I. When using the prior models we frequently found that we had to add amplifications like, "it meets the strong guarantee except there could be side effects." Descriptions like that implicitly point to missing guarantees, which are now accounted for above.

Second, the above ACID analysis makes it clear why Mueller's and Reeves' "Bad" and "Undefined" states, and the sometimes-cited "destructible" guarantee, are all the same thing from a program correctness point of view.

I have always thought that the "Bad," "Undefined," and "destructible" ideas had problems. There's nothing magical about exiting a function via a C++ exception compared to returning an error value indicating the operation failed, so consider: Say that a programmer on your team wrote a function that could never throw and just returned a code to indicate success or failure, and that could leave an object in a state where it no longer met its invariants (or, even worse, couldn't even be destroyed). Would you accept the work, or would you send the programmer back to his desk to fix his bug? I think most of us would agree that the function just described is plainly buggy (and so is its specification if the specification says it can render objects invariant-breaking). So why should the answer be any different in the case of failure reported by throwing an exception? Clearly it shouldn't, and in fact none of the Bad, Undefined, or destructible "guarantees" really gives any kind of useful program correctness or safety guarantee, because all useful program correctness guarantees include at least the concept of preserving object and system invariants.

\[Aside: Consider "Bad" and "destructible" a little further. If a function f() can leave any object t in a state wherein all you can ever do is call t's destructor, then f() is a sink in all but name, and might as well destroy t too.]

**Filling Holes: ACID and GotW #59**
The second goal of this taxonomy was that "[i]t should help to fill holes in existing methods, such as a situation where given sample code was known to support some exception-safety guarantee but that guarantee could not be adequately described using the existing models alone."

This brings us to Question #5:

**5. Does the answer from Question #4 equip us to give a better answer to Question #1? Explain.**

When we considered Question #1, I said that it seems like there ought to be a name for a concept as fundamental as "the strongest guarantee you can make (for assignment) without requiring assumptions about the used type." In the ACID model, it turns out that there is:

| ACID | Guarantee |
| ---- | ---- |
| aC\-\- | The guarantee provided by the technique shown in GotW #59, if T construction and destruction provide at least -C\-\- (which should be a given for correct programs). |

Without making any assumptions about the used type(s), is it possible to make a stronger guarantee than aC for a generalized assignment operator? I believe it is possible to show fairly rigorously that the answer is No, as follows:

1. Consider a simplified version of the T::operator=() code.

``` cpp
// (Simplified) solution to Cargill's Widget Example
//
class Widget { /*...*/ struct Impl* pimpl_; };

class Widget::Impl { /*...*/ T1 t1_; T2 t2_; };

Widget& Widget::operator=( const Widget& other )
{
    Impl* temp = new Impl( *other.pimpl_ );
    // construction could fail
    
    delete pimpl_;
    pimpl_ = temp;
    return *this;
}
```

As before, the above code invokes the Widget::Impl constructor to create a new object from the existing one. If the construction fails, C++'s language rules automatically destroy any fully-constructed subobjects (such as t1_, if the t2_ construction fails). So we could end up getting any of the following function call sequences:

``` cpp
Case 1 (Failure): T1::T1() fails

Case 2 (Failure): T1::T1(), T2::T2() fails, T1::~T1()

Case 3 (Success): T1::T1(), T2::T2()
```

2. Consider the C++ language rules. The C++ language rules strictly define the meaning of construction and destruction (in both cases we assume the Consistency guarantee because, for example, any type's constructor does not yield a valid object of that type is just buggy):


    C++ Constructor Guarantee: aC\-\-
    A constructor, if successful, must create a valid object of the indicated type; if it fails, it has not created anything.
     
    C++ Destructor Guarantee: AC\-\-
    A destructor destroys a valid object of the indicated type, and so is an inverse of any constructor of the same type. Further, a destructor is guaranteed to succeed, at least in the sense that the object it operates on is considered fully-destroyed, or as-destroyed-as- it-can-ever-be because you can't re-run a failed destructor. (Further, the standard requires that any destructor used in the standard library may not throw; this includes the destructor of any type you can legally instantiate a container with.)
     
    At any rate, the above solution assumes implicitly that at least T1::~T1() guarantees A---, else there is no way to write a correct Widget::operator=() anyway. For more on these issues, see Item 16 in Exceptional C++[6].

3. Analyze Atomicity (i.e., local effects). In each of the two failure cases, the only local effects are the possible construction or destruction of the t1_ and t2_ member objects, because these are the only objects being directly manipulated.


    Case 1 (Failure): T1::T1() fails The only attempted operation promises at least 'a' atomicity, so no local effects occur and the entire assignment has no local effects.
    
    Case 2 (Failure): T1::T1(), T2::T2() fails, T1::~T1() T2::T2() promises at least 'a' atomicity, so does not have any local effects. But T1::~T1() is guaranteed to succeed and undoes all local effects of T1::T1(), and so the entire assignment has no local effects.
    
    Case 3 (Success): T1::T1(), T2::T2() Success, all local effects have been successfully produced.

Failure is possible, but all failure cases have no local effects. Therefore the assignment's best possible atomicity guarantee level is 'a' \-\- thanks to the guarantees inherent in the C++ language.

4. Analyze Consistency. Failure is possible, but in each case each suboperation guarantees the progression from one consistent state to another regardless of success or failure, so the entire assignment ends in a consistent state. Therefore the assignment's best possible consistency guarantee level is 'C' \-\- again thanks to the guarantees inherent in the C++ language.

5. Analyze Isolation. No guarantee is possible here, because no suboperation makes any guarantee about side effects. For example, in Case 1, T1::T1() may fail in a way that has already affected global state.

6. Analyze Durability. No guarantee is possible here either, because no suboperation makes any guarantee about durability. For example, in Case 1, T1::T1() may fail in a way that does not achieve persistence of the final state.

So aC is the strongest exception safety guarantee one can make for assignment without requiring any assumptions at all about the used type. That's a useful piece of information, especially in generic programming using language facilities like C++ templates where we cannot always assume knowledge of the guarantees provided by arbitrary and even unknowable types.

Better still, we can make an even stronger statement with a little knowledge about the types we're using:

| ACID | Guarantee |
| ---- | ---- |
| aCI- | The guarantee provided by the technique shown in GotW #59, if T1 and T2 construction and destruction provide at least -CI-. |

Interestingly, note that this guarantee (aCI-) promises more than the Abrahams strong guarantee (aCi-).

In short:

- With no knowledge of the used types T1 and T2, it is always possible to write an assignment operator for T that gives "better than the basic guarantee" \-\- specifically, such an assignment operator is aC-safe.

- With the additional knowledge that T1 and T2 construction and destruction are CI-safe, if is always possible to write an assignment operator for T that gives "better than the strong guarantee" \-\- specifically, such an assignment operator is aCI-safe.

Neither of these guarantees is concisely expressible using prior exception and program safety models.

**TRANSACTIONAL PROGRAM CORRECTNESS ANALYSIS**
The third goal of this taxonomy was that "\[i]t should help us reason about and analyze the safety of code, making it less a craft and more a methodical engineering task."

Let's see how well we can do.

**ACID Safety Analysis**
It's important that the ACID program correctness taxonomy precisely describes the prior Abrahams and other guarantees, but perhaps the best thing about the ACID model is that it simplifies our analysis because the compound guarantees are no longer the important thing at all.

Consider first the following principle, which is perhaps the most important result of this article:

    **The Transactional Principle:**
    All programming operations should be considered as transactions, with associated ACID guarantees. This applies regardless of whether exceptions are possible or absent.

All programming operations are amenable to this new style of ACID transactional analysis. After years of differing with Dave Abrahams on this point, I can now in good conscience come around to his point of view that "exceptions are not magical" \-\- they are simply another failure case, albeit with complex programming semantics in C++ that come into play when you actually use them, but conceptually no different from an error return code. The reason I only now feel I can agree with Dave Abrahams is that, after writing about and advancing this topic for several years, I only now think I finally understand exception safety well enough to be able to reason about it confidently as simply another form of "early return."

By defining distinct and largely independent axes, the ACID model gives us a simpler and systematic \-\- even mechanical \-\- way to analyze an operation's correctness: Analyze it according to the individual safety guarantees, one axis at a time, not according to aggregated "compound guarantees." The way to analyze a function, then, is to examine each axis' guarantee level in turn:

1. If the axis has no guarantee, do nothing.

2. If the axis has a guarantee about a failure case, then for each possible EARLY return (each possible failure return, or each possible throw or interrupt point which means each non-A-safe function call we perform as a suboperation), ensure the guarantee is met if the early return is taken.

3. If the axis makes a guarantee about a success case, then for each possible NORMAL return, ensure the guarantee is met if the return is taken.

Let's consider examples to show why this can make reasoning about exception safety simpler, more systematic, more mechanical \-\- which is to say more like engineering and less like craft:

**Example: Analyzing for aCi-Safety (the Strong Guarantee)**
Say you are writing a function that must guarantee aCi-safety (the Abrahams strong guarantee). How do you analyze whether it meets the guarantee? By considering each axis independently as follows:

1. [a] For each possible EARLY return, ensure the immediate object is returned to the initial state if the early return is taken.

2. [C] For each NORMAL return, ensure the immediate object's state is consistent.

3. [i] For each possible EARLY return, ensure that side effects are undone if the early return is taken.

Note that we didn't need to consider the c-safe aspect because ai-safe => aci-safe since we require that the initial state of the system be consistent.

**Another Example: Analyzing for aC-Safety**
Now say you are writing a function that must guarantee aC-safety. How do you analyze whether it meets the guarantee? By considering each axis independently as follows:

1. [a] For each possible EARLY return, ensure the immediate object is returned to the initial state if the early return is taken.

2. [c] For each possible EARLY return, ensure that the system is in a consistent state if the early return is taken.

3. [C] For each NORMAL return, ensure the immediate object's state is consistent.

**OTHER RESULTS AND CONCLUSIONS**
**The Importance of "Big-A" Atomicity**
As pointed out by Abrahams and Colvin and others, to guarantee any kind of exception safety it is essential that certain operations \-\- notable destructors and deallocation functions \-\- be required not to throw. That is, they must be at least A-safe. We rely on this in all sorts of situations, notably:

- The "do all the work off to the side, and commit using only nonthrowing operations" method is not possible if there are no suitable nonthrowing (A-safe) operations.

- The related "create a temporary and Swap()" method, which is the standard form of exception-safe copy assignment, is not possible without a nonthrowing (A-safe) Swap().

An interesting point raised by Greg Colvin is that for this reason it does not appear to be possible to write exception-safe code in Java. The Java specification currently allows the JVM to throw an exception at any time at arbitrary points in the program, for example asynchronously when stop is called on a Thread object. In practice it isn't that bad, but it is worrisome that the Java specification can apparently be shown to preclude writing exception-safe code because there is no way to absolutely guarantee A-safe operations in Java.

**Side Effects Last**
It seems best to try to perform I-safe operations last, in order to put off side effects. That way, if a failure occurs earlier in the operation, no side effects will have been begun. This helps to limit the scope for side effects after failures.

**Future Directions**
An important area for further study is a rigorous treatment of analyzing compound operations. For example, as illustrated in one of the analysis examples above, the consistency guarantees operate like a chain; an operation is c-safe only if all suboperations are c-safe (any one that isn't becomes the weak link that breaks the chain), and an operation is C-safe only if all suboperations are C-safe. That's fine for consistency... now, is there anything we can conclude similarly about atomicity, isolation, durability, or combinations thereof in suboperations?

**Summary**
When I started this taxonomy, I had in mind applying database concepts to better analyze and describe C++ exception safety. In the process of doing so, however, I kept on coming back to the simple terms "successful operation" and "failed operation," and finally realized (strongly influenced, I am sure, by papers like Dave Abrahams' "Exception Safety in Generic Components"[1]) that the latter applied completely and equally to ALL forms of failure \-\- C++ exceptions, Java exceptions, C return codes, asynchronous interrupts, and any other imaginable failure mode and reporting mechanism. That made me realize that what I was developing might not really be just a taxonomy for analyzing C++ exception safety at all, but rather something much more general: A taxonomy for describing program correctness in all its aspects in all languages. Further use, critique, and development will show whether this approach is indeed that widely applicable or not.

Finally, consider two fundamental implications:

1. The Transactional Principle: All programming operations should be considered as transactions, with associated guarantees, and are amenable to this new style of ACID transactional analysis. This applies regardless of whether exceptions are possible or absent.

2. As pointed out by Dave Abrahams in his "Exception-Safety in Generic Components" paper[1] and other places, "exception safety" is a bit of a misnomer, and the ACID model bears this out by not distinguishing between different kinds of "early returns," whether they be through error codes, through exceptions, or through some other method. Exception-related "safety" is not some magical entity or unknowable phantom; "safety" is just another name for "correctness." When we want to talk about "exception safety," we should instead just talk about "program correctness" without arbitrarily highlighting one form of correctness. Unfortunately, this only underscores that languages that have exceptions but do not allow exception-safe programming (such as by not supporting the strongest no-failure atomicity guarantee, or A-safety) actually have a worse problem, namely insufficient support for general program correctness.

If this taxonomy and its related analysis methods are at all useful, they will be subject to refinement. I invite readers to analyze and expand upon these ideas.

**Acknowledgments**
Thanks to Dave Abrahams, Greg Colvin, and Scott Meyers for incisive reviews of this article, along with much other fun and productive discussion of C++ topics over the years.

**Notes**
1. D. Abrahams. "Exception-Safety in Generic Components," in Generic Programming: Proceedings of a Dagstuhl Seminar, M. Jazayeri, R. Loos, and D. Musser, eds. (Springer Verlag, 1999).

2. H. Mueller. "10 Rules for Handling Exception Handling Successfully" (C++ Report, 8(1), January 1996).

3. J. Reeves. "Coping with Exceptions" (C++ Report, 8(3), March 1996).

4. C.J. Date. An Introduction to Database Systems, 7th Edition (Addison-Wesley, 1999).

5. H. Sutter. "Namespaces and the Interface Principle" (C++ Report, 11(3), March 1999).

6. H. Sutter. Exceptional C++ (Addison-Wesley, 2000).



# 062 Smart Pointer Members 
**Difficulty: 6 / 10**
Most C++ programmers know they have to take special care for classes with pointer members. But what about classes with auto_ptr members? And can we make life safer for ourselves and our users by devising a smart pointer class designed specifically for class membership?

----

**Problem**
**JG Question**
1. Consider the following class:

``` cpp
// Example 1

class X1
{
    // ...
private:
    Y* y_;
};
```

If an X1 object owns its pointed-at Y object, why can't the author of X use the compiler-generated destructor, copy constructor, and copy assignment?

**Guru Questions**
2. What are the advantages and drawbacks of the following approach?

``` cpp
// Example 2

class X2
{
    // ...
private:
    auto_ptr<Y> y_;
};
```

3. Write a suitable HolderPtr template that is used as shown here:

``` cpp
// Example 3

class X3
{
    // ...
private:
    HolderPtr<Y> y_;
};
```

to suit three specific circumstances:

   a) Copying and assigning HolderPtrs is not allowed.

   b) Copying and assigning HolderPtrs is allowed, and has the semantics of creating a copy of the owned Y object using the Y copy constructor.

   c) Copying and assigning HolderPtrs is allowed, and has the semantics of creating a copy of the owned Y object, which is performed using a virtual Y::Clone() method if present and the Y copy constructor otherwise.

----

**Solution**
**Recap: Problems of Pointer Members**
**1. Consider the following class:**

``` cpp
// Example 1

class X1
{
    // ...
private:
    Y* y_;
};
```

If X1 owns its pointed-at Y, the compiler-generated versions will do the wrong thing. First, note that some function (probably a constructor) has to create the owned Y object, and there has to be another function (likely the destructor, X1::~X1()) that deletes it:

``` cpp
// Example 1(a): Ownership semantics.

{
    X1 a; // allocates a new Y object and points at it

    // ...
}   // as a goes out of scope and is destroyed, it
    // deletes the pointed-at Y
```

Then use of the default memberwise copy construction will cause multiple X1 objects to point at the same Y object, which will cause strange results as modifying one X1 object changes the state of another, and which will also cause a double delete to take place:

``` cpp
// Example 1(b): Sharing, and double delete.

{
    X1 a;   // allocates a new Y object and points at it
    
    X1 b( a ); // b now points at the same Y object as a
    
    // ... manipulating a and b modifies
    // the same Y object ...
}   // as b goes out of scope and is destroyed, it
    // deletes the pointed-at Y... and so does a, oops
```

And use of the default memberwise copy assignment will also cause multiple X1 objects to point at the same Y object, which will cause the same state sharing, the same double delete, and as an added bonus will also cause leaks when some objects are never deleted at all:

``` cpp
// Example 1(c): Sharing, double delete, plus leak.

{
    X1 a;  // allocates a new Y object and points at it
    
    X1 b;  // allocates a new Y object and points at it
    
    b = a; // b now points at the same Y object as a,
           // and no one points at the Y object that
           // was created by b
    
    // ... manipulating a and b modifies
    // the same Y object ...
}   // as b goes out of scope and is destroyed, it
    // deletes the pointed-at Y... and so does a, oops
    // the Y object allocated by b is never deleted
```

In other code, we normally apply the good practice of wrapping bald pointers in manager objects that own them and simplify cleanup. If the Y member was held by such a manager object, instead of by a bald pointer, wouldn't that ameliorate the situation? This brings us to our Guru Questions:

**What About auto_ptr Members?**
**2. What are the advantages and drawbacks of the following approach?**

``` cpp
// Example 2

class X2
{
    // ...
private:
    auto_ptr<Y> y_;
};
```

In short, this has some benefits but it doesn't do a whole lot to solve the problem that the automatically generated copy construction and copy assignment functions will do the wrong thing. It just makes them do different wrong things.

First, if X2 has any user-written constructors, making them exception-safe is easier because if an exception is thrown in the constructor the auto_ptr will perform its cleanup automatically. The writer of X2 is still forced, however, to allocate his own Y object and hold it, however briefly, by a bald pointer before the auto_ptr object assumes ownership.

Next, the automatically generated destructor now does in fact do the right thing. As an X2 object goes out of scope and is destroyed, the auto_ptr<Y> destructor automatically performs cleanup by deleting the owned Y object. Even so, there is a subtle caveat here that has already caught me once: If you rely on the automatically generated destructor, then that destructor will be defined in each translation unit that uses X2, which means that the definition of Y must be visible to pretty much anyone who uses an X2 (this is not so good if Y is a Pimpl, for example, and the whole point is to hide Y's definition from clients of X2). So you can rely on the automatically generated destructor, but only if the full definition of Y is supplied along with X2 (for example, if x2.h includes y.h):

``` cpp
// Example 2(a): Y must be defined.

{
    X2 a; // allocates a new Y object and points at it
    
    // ...
    
}   // as a goes out of scope and is destroyed, it
    // deletes the pointed-at Y, and this can only
    // happen if the full definition of Y is available
```

If you don't want to provide the definition of Y, then you must write the X2 destructor explicitly, even if it is just empty.

Next, the automatically generated copy constructor will no longer have the double-delete problem described in Example 1(b). That's the good news. The not-so-good news is that the automatically generated version introduces another problem, namely grand theft: The X2 object being constructed rips away the Y belonging to copied-from X2 object, including all knowledge of the Y object:

``` cpp
// Example 2(b): Grand theft pointer.

{
    X2 a; // allocates a new Y object and points at it
    
    X2 b( a ); // b rips away a's Y object, leaving a's
               // y_ member with a null auto_ptr
    
    // if a attempts to use its y_ member, it won't
    // work; if you're lucky, the problem will manifest
    // as an immediate crash, otherwise it will likely
    // manifest as a difficult-to-diagnose intermittent
    // failure
}
```

The only redeeming point about the above grand theft, and it isn't much, is that at least the automatically generated X2 copy constructor gives some fair warning that theftlike behavior may be impending. Why? Because its signature will be X2::X2( X2& ) \-\- note that it takes it parameter by reference to non-const. That's what auto_ptr's copy constructor does, after all, and so X2's automatically generated one has to follow suit. This is pretty subtle, though, but at least it prevents copying from a const X2.

Finally, the automatically generated copy assignment operator will no longer have either the double-delete problem or the leak problem, both of which were described in Example 1(c). That's the good news. Alas, again, there's some not-so-good news, because the same grand theft occurs: The assigned-to X2 object rips away the Y belonging to assigned-from X2 object, including all knowledge of the Y object, and in addition it (possibly prematurely) deletes the Y object that it originally owned:

``` cpp
// Example 2(c): More grand theft pointer.

{
    X2 a;  // allocates a new Y object and points at it
    
    X2 b;  // allocates a new Y object and points at it
    
    b = a; // b deletes its pointed-at Y, rips away
           // a's Y, and leaves a with a null auto_ptr
           // again
    
    // as in Example 2(b), any attempts by a to use its
    // y_ member will be disastrous
}
```

Similarly above, at least the theftish behavior is hinted at, because the automatically generated function will be declared as X2& X2::operator=( X2& ), thus advertising (albeit in the fine print, not in a front-page banner) that the operand can be modified.

In summary, then, auto_ptr does give some benefits, particularly by automating cleanup for constructors and the destructor. It does not, however, of itself answer the main original problems in this case: That we have to write our own copy construction and copy assignment for X2, or else disable them if copying doesn't make sense for the class. For that, we can do better with something a little more special-purpose.

**Variations On HolderPtr**
The meat of this article involves development successively refined versions of a HolderPtr template that is more suitable than auto_ptr for the kinds of uses outlined above.

A note on exception specifications: For reasons I won't go into here (see a coming issue of GotW), exception specifications are not as useful as you might think. On the other hand, it is important to know what exceptions a function might throw, especially if it is AC-safe (always succeeds and leaves the system in a consistent state) which means no exception can occur; this is also known as a nothrow guarantee. Well, you don't need exception specifications to document behavior, and so I am going to assert the following:
   > For every version of HolderPtr<T> presented in this article, all member functions are AC-safe (nothrow) except that construction or assignment from a HolderPtr<U> (where U could be T) might cause an exception to be thrown from a T constructor.

Now let's get into the meat of it:

**A Simple HolderPtr: Strict Ownership Only**
**3. Write a suitable HolderPtr template that is used as shown here:**

``` cpp
// Example 3

class X3
{
    // ...
private:
    HolderPtr<Y> y_;
};
```

We are going to consider three cases. In all three, the constructor benefit still applies: Cleanup is automated and there's less work for the writer of X3::X3() to be exception-safe and avoid leaks from failed constructors.[2] Also, in all three, the destructor restriction still applies: Either the full definition of Y must accompany X3, or the X3 destructor must be explicitly provided, even if it's empty.

to suit three specific circumstances:

   a) Copying and assigning HolderPtrs is not allowed.

There's really not much to it:

``` cpp
// Example 3(a): Simple case: HolderPtr without
// copying or assignment.

template<class T>
class HolderPtr
{
public:
    explicit HolderPtr( T* p = 0 ) : p_( p ) { }

    ~HolderPtr() { delete p_; p_ = 0; }
```

Of course, there has to be some way to access the pointer, so add something like the following, which parallels std::auto_ptr:

``` cpp
    T& operator*() const { return *p_; }

    T* operator->() const { return p_; }
```

What else might we need? Well, for many smart pointer types it can make sense to provide additional facilities that parallel auto_ptr's reset() and release() functions to let users arbitrarily change which object is owned by a HolderPtr. It may seem at first like that such facilities would be a good idea because they contribute to HolderPtr's intended purpose and usage as a class member; consider, for example, Example 4 in the solution to GotW #59, where the class member holds a Pimpl pointer and it's desirable to write an exception-safe assignment operator... then you need a way to swap HolderPtrs without copying the owned objects. But providing reset()- and release()-like functions isn't the right way to do it... that would let users do what they need for swapping and exception-safety, but it would also open the door for many other (unneeded) options that don't contribute to the purpose of HolderPtr and can cause problems if abused.

So what to do? Insteading of providing overly general facilities, understand your requirements well enough to provide just the facility you really need:

``` cpp
    void Swap( HolderPtr& other ) { swap( p_, other.p_ ); }

private:
    T* p_;

    // no copying
    HolderPtr( const HolderPtr& );
    HolderPtr& operator=( const HolderPtr& );
};
```

We take ownership of the pointer and delete it afterwards, we handle the null pointer case, and copying and assignment are specifically disabled in the usual way by declaring them private and not defining them. Construction is explicit as good practice to avoid implicit conversions, which are never needed by HolderPtr's intended audience.

There, that was easy. I hope it didn't lull you into a false sense of security, because the next steps have some subtleties attached.

**Copy Construction and Copy Assignment**
   b) Copying and assigning HolderPtrs is allowed, and has the semantics of creating a copy of the owned Y object using the Y copy constructor.

Here's one way to write it that satisfies the requirements but isn't as general-purpose as it could be. It's the same as Example 3(a), but with copy construction and copy assignment defined:

``` cpp
// Example 3(b)(i): HolderPtr with copying and
// assignment, take 1.

template<class T>
class HolderPtr
{
public:
    explicit HolderPtr( T* p = 0 ) : p_( p ) { }
    
    ~HolderPtr() { delete p_; p_ = 0; }
    
    T& operator*() const { return *p_; }
    
    T* operator->() const { return p_; }
    
    void Swap( HolderPtr& other ) { swap( p_, other.p_ ); }
    
    //--- new code begin ------------------------------
    HolderPtr( const HolderPtr& other )
        : p_( other.p_ ? new T( *other.p_ ) : 0 ) { }
```

Note that it's important to check whether other's pointer is null or not. Since, however, operator=() is implemented in terms of copy construction, we only have to put the check in one place.

``` cpp
    HolderPtr& operator=( const HolderPtr& other )
    {
      HolderPtr<T> temp( other );
      Swap( temp );
      return *this;
    }
    //--- new code end --------------------------------

private:
    T* p_;
};
```

This satisfies the stated requirements, because in the intended usage there's no case where we will be copying or assigning from a HolderPtr that manages any type other than T. If that's all we know you'll ever need, then that's fine. But whenever we design a class, we should at least consider designing for extensibility if it doesn't cost us much extra work and could make the new facility more useful to users in the future. At the same time, we need to balance such "design for reusability" with the danger of overengineering, that is, of providing an overly complex solution to a simple problem. This brings us to the next point:

**Templated Construction and Templated Assignment**
One question to consider is this: What is the impact on the Example 3(b)(i) code if we want to allow for the possibility of assigning between different types of HolderPtr in the future? That is, we want to be able to copy or assign a HolderPtr<X> to a HolderPtr<Y> if X is convertible to Y. It turns out that the impact is minimal: Duplicate the copy constructor and the copy assignment operators with templated versions that just add "template<class U>" in front and take a parameter of type "HolderPtr<U>&", as follows:

``` cpp
// Example 3(b)(ii): HolderPtr with copying and
// assignment, take 2.

template<class T>
class HolderPtr
{
public:
    explicit HolderPtr( T* p = 0 ) : p_( p ) { }

    ~HolderPtr() { delete p_; p_ = 0; }

    T& operator*() const { return *p_; }
    
    T* operator->() const { return p_; }
    
    void Swap( HolderPtr& other ) { swap( p_, other.p_ ); }
    
    HolderPtr( const HolderPtr& other )
        : p_( other.p_ ? new T( *other.p_ ) : 0 ) { }
    
    HolderPtr& operator=( const HolderPtr& other )
    {
        HolderPtr<T> temp( other );
        Swap( temp );
        return *this;
    }
    
    //--- new code begin ------------------------------
    template<class U>
    HolderPtr( const HolderPtr<U>& other )
        : p_( other.p_ ? new T( *other.p_ ) : 0 ) { }
    
    template<class U>
    HolderPtr& operator=( const HolderPtr<U>& other )
    {
        HolderPtr<T> temp( other );
        Swap( temp );
        return *this;
    }

private:
    template<class U> friend class HolderPtr;
    //--- new code end --------------------------------

    T* p_;
};
```

Did you notice the trap we avoided? We still need to write the nontemplated forms of copying and assignment too in order to suppress the automatically generated versions, because a templated constructor is never a copy constructor and a templated assignment operator is never a copy assignment operator. For more information about this, see Item 5 in Exceptional C++.[1]

   ***Balancing reusability against overengineering***
   *From what I understand of XP, it seems to me that migration from 3(b)(i) to 3(b)(ii) may happen in an XP environment only if HolderPtr is used in a single project, or is owned by the same team that comes across the new requirement to copy and assign between different kinds of HolderPtr. Part of the core XP philosophy is to only refactor or extend when necessary to implement a particular new feature, and that assumes that the person implementing the feature also has rights to extend the code being reused, here HolderPtr; if not, you'll end up having several project teams each writing their own extensions as needed when a reusable common component is insufficient, instead of upgrading the common component once and saving overall effort across projects. (And that's just the "upgrade/extend" case; it's even harder to refactor common components already in use in multiple projects even if you do have the source rights, because the need to maintain interface stability greatly limits the scope of acceptable changes.)*

   *This kind of situation seems that it might build into XP a systemic impediment to "design for extensibility" at least for library creation and maintenance and other cross-project reuse efforts, and is part of the cost for the short-term rather than long-term thinking encouraged by XP. That's not to say that XP's advantages may not outweigh the disadvantages in a given situation \-\- I'm sure there are situations where they do --, but it is important to understand both sides of a particular methodology before jumping into it with both feet for a particular project. For example, XP has a lot to recommend it for a single, short-term, focused project that does not overlap much or at all with other projects and can be thrown away if needed, but XP might not be as appropriate for writing things like space shuttle onboard systems, or even shared libraries unless the library team was treated as a distinct project.*

   *This is my feeling only, based on what little I know of XP. Corrections and flames are welcome.*

There is still one subtle caveat, though, but fortunately it's not really a big deal (or even, I would say, our responsibility as the authors of `HolderPtr`): With either the templated or nontemplated copy and assignment functions, the source ("other") object could still actually be holding a pointer to a derived type in which case we're slicing. For example:

``` cpp
class A {};
class B : public A {};
class C : public B {};

HolderPtr<A> a1( new B );
HolderPtr<B> b1( new C );

// calls copy ctor,
// slices
HolderPtr<A> a2( a1 );

// calls templated ctor,
// slices
HolderPtr<A> a3( b1 );

// calls copy assignment,
// slices
a2 = a1;

// calls templated
// assignment, slices
a3 = b1;
```

I point this out because this is the sort of thing one shouldn't forget to write up in the HolderPtr documentation to warn users, preferably in a "Don't Do That" section. There's not much else we the authors of `HolderPtr` can do in code to stop this kind of abuse.

So which is the right solution to problem 3(b) \-\- Example 3(b)(i), or Example 3(b)(ii)? Both are good solutions, and it's really a judgment call based on your own experience at balancing design-for-reuse and overengineering-avoidance. I imagine that XP advocates in particular would automatically use 3(b)(i) because it satisfies the minimum requirements. I can also imagine situations where HolderPtr is in a library written by one group and shared by several distinct teams and where 3(b)(ii) will end up saving overall development effort through reuse and the prevention of reinvention.

**Adding Extensibility Using Traits**
But, now, what if Y has a virtual Clone() method? It may seem from Example 1 that X always creates its own owned Y object, but it might get it from a factory or from a new-expression of some derived type. As we've already seen, in such a case the owned Y object might not really be a Y object at all, but be of some type derived from Y, and copying it as a Y would slice it at best and render it unusable at worst. The usual technique in this kind of situation is for Y to provide a special virtual Clone() member function that allows complete copies to be made even without knowing the complete type of the object being pointed at.

So, what if someone wants to use a HolderPtr to hold such an object, that can only be copied using a function other than the copy constructor? This is the point of our final question:

c) Copying and assigning HolderPtrs is allowed, and has the semantics of creating a copy of the owned Y object, which is performed using a virtual Y::Clone() method if present and the Y copy constructor otherwise.

In the HolderPtr template, we don't know what our contained T type really is, we don't know whether it has a virtual Clone() function, and so we don't know the right way to copy it. Or do we?

One solution is to apply a technique widely used in the C++ standard library itself, namely traits. Briefly, a traits class is defined as follows, quoting clause 17.1.18:

a class that encapsulates a set of types and functions necessary for template classes and template functions to manipulate objects of types for which they are instantiated

First, let's change Example 3(b)(ii) slightly to remove some redundancy. You'll notice that both the templated constructor and the copy constructor have to check the source for nullness. Let's put all that work in a single place and have a single Create() function that builds a new T object (we'll see another reason to do this in a minute):

``` cpp
// Example 3(c)(i): HolderPtr with copying and
// assignment, Example 3(b)(ii)
// with a little factoring.
//
template<class T>
class HolderPtr
{
public:
  explicit HolderPtr( T* p = 0 ) : p_( p ) { }

  ~HolderPtr() { delete p_; p_ = 0; }

  T& operator*() const { return *p_; }

  T* operator->() const { return p_; }

  void Swap( HolderPtr& other ) { swap( p_, other.p_ ); }

  HolderPtr( const HolderPtr& other )
    : p_( CreateFrom( other.p_ ) ) { } // changed

  HolderPtr& operator=( const HolderPtr& other )
  {
    HolderPtr<T> temp( other );
    Swap( temp );
    return *this;
  }

  template<class U>
  HolderPtr( const HolderPtr<U>& other )
    : p_( CreateFrom( other.p_ ) ) { } // changed

  template<class U>
  HolderPtr& operator=( const HolderPtr<U>& other )
  {
    HolderPtr<T> temp( other );
    Swap( temp );
    return *this;
  }

private:
  //--- new code begin ------------------------------
  template<class U>
  T* CreateFrom( const U* p ) const
  {
    return p ? new T( *p ) : 0;
  }
  //--- new code end --------------------------------

  template<class U> friend class HolderPtr;

  T* p_;
};
```

Now, `CreateFrom()` gives us a nice hook to encapsulate all knowledge about different ways of copying a T.

**Applying Traits**
Now we can apply the traits technique using something like the following approach. Note that this is not the only way to apply traits, and there are other ways besides traits to deal with different ways of copying a T. I chose to use a single traits class template with a single static `Clone()` member that calls whatever is needed to do the actual cloning, and which can be thought of as an adapter. This follows the style of char_traits, for example, which simply delegates the work to the traits class. (Alternatively, for example, the traits class could provide typedefs and other aids so that the HolderPtr template to figure out what to do, but still have to do it itself, but it's a needless division of responsibilities to have one place find out what the right thing to do is, and a different place to actually do it.)

``` cpp
template<class T>
class HolderPtr
{
  // ...

  template<class U>
  T* CreateFrom( const U* p ) const
  {
    // the "Do the Right Thing" fix... but how?
    return p ? HPTraits<U>::Clone( p ) : 0;
  }
};
```

We want HPTraits to be a template that does the actual cloning work, where the main template's implementation of `Clone(`) uses `U`'s copy constructor. Two notes: First, since `HolderPtr` assumes responsibility for the null check, `HPTraits::Clone()` doesn't have to do it; and second, if `T` and `U` are different this function can only compile if a `U*` is convertible to a `T*`, in order to correctly handle polymorphic cases like `T==Base` `U==Derived`.

``` cpp
template<class T>
class HPTraits
{
  static T* Clone( const T* p ) { return new T( *p ); }
};
```
Then HPTraits is specialized as follows for any given Y that does not want to use copy construction. For example, say that some Y has a virtual Y* CloneMe() function, and some Z has a virtual void CopyTo( Z& ) function; then HPTraits<Y> is specialized so as to let that function do the cloning:
``` cpp
// The most work any user has to do, and it only
// needs to be done once, in one place:
//
template<>
class HPTraits<Y>
{
  static Y* Clone( const Y* p )
    { return p->CloneMe(); }
};

template<>
class HPTraits<Z>
{
  static Z* Clone( const Z* p )
    { Z* z = new Z; p->CopyTo(*z); return z; }
};
```

This is much better, and it will work with whatever flavor and signature of `CloneMe()` is ever invented in the future; `Clone()` only needs to create it under the covers in whatever way it deems desirable, and the only visible result is a pointer to the new object... another good argument for strong encapsulation.

To use HolderPtr with a brand new type Y that was invented centuries after `HolderPtr` was written and its authors turned to dust, the new fourth-millennium user (or author) of `Y` merely needs to specialize `HPTraits` once for all time, then use `HolderPtr<Y>`s all over the place in her code wherever desired. That's pretty easy. And, if Y doesn't have a virtual `Clone()` function, the user (or author) of Y doesn't even need to do that and can just use `HolderPtr` without any work at all.

A brief coda: Since `HPTraits` has only a single static function template, why make it a class template instead of just a function template? The main motive is for encapsulation (in particular, better name management) and extensibility. We want to avoid cluttering the global namespace with free functions; the function template could be put at namespace scope in whatever namespace `HolderPtr` itself is supplied in, but even then this couples it more tightly to `HolderPtr` as opposed to use by perhaps other code also in that namespace. Now, the `Clone()` function template may be the only kind of trait we need today, but what if we need new ones tomorrow? If the additional traits are functions, we'd otherwise have to continue cluttering things up with extra free functions. But what if the additional traits are typedefs or even class types? HPTraits gives us a nice place to encapsulate all of those things.

**HolderPtr With Traits**
Here is the code for a HolderPtr that incorporates cloning traits along with all of the earlier requirements in Question 3.[3] Note that the only change from Example 3(c)(i) is a one-line change to HolderPtr and one new template with a single simple function... now that's what I call minimum-impact given all of the flexibility we've just bought:

``` cpp
// Example 3(c)(ii): HolderPtr with copying and
// assignment and full traits-based
// customizability.
//
//--- new code begin --------------------------------
template<class T>
class HPTraits
{
static T* Clone( const T* p ) { return new T( *p ); }
};
//--- new code end ----------------------------------

template<class T>
class HolderPtr
{
public:
  explicit HolderPtr( T* p = 0 ) : p_( p ) { }

  ~HolderPtr() { delete p_; p_ = 0; }

  T& operator*() const { return *p_; }

  T* operator->() const { return p_; }

  void Swap( HolderPtr& other ) { swap( p_, other.p_ ); }

  HolderPtr( const HolderPtr& other )
    : p_( CreateFrom( other.p_ ) ) { }

  HolderPtr& operator=( const HolderPtr& other )
  {
    HolderPtr<T> temp( other );
    Swap( temp );
    return *this;
  }

  template<class U>
  HolderPtr( const HolderPtr<U>& other )
    : p_( CreateFrom( other.p_ ) ) { }

  template<class U>
  HolderPtr& operator=( const HolderPtr<U>& other )
  {
    HolderPtr<T> temp( other );
    Swap( temp );
    return *this;
  }

private:
  template<class U>
  T* CreateFrom( const U* p ) const
  {
  //--- new code begin ----------------------------
    return p ? HPTraits<U>::Clone( p ) : 0;
  //--- new code end ------------------------------
  }

  template<class U> friend class HolderPtr;

  T* p_;
};
```

**A Usage Example**
Here is an example that shows the typical implementation of the major functions (construction, destruction, copying, and assignment) for class that uses our final version of HolderPtr, ignoring more detailed issues like whether the destructor ought or ought not to be present (or inline) because Y is or is not defined:

``` cpp
// Example 4: Sample usage of HolderPtr.
//
class X
{
public:
  X() : y_( new Y(/*...*/) ) { }

  ~X() { }

  X( const X& other ) : y_( new Y(*(other.y_) ) ) { }

  void Swap( X& other ) { y_.Swap( other.y_ ); }

  X& operator=( const X& other )
  {
    X temp( other );
    Swap( temp );
    return *this;
  }

private:
  HolderPtr<Y> y_;
};
```

**Summary**
One moral I would like you to take away from this issue of GotW is to always be aware of extensibility as you're designing. By default, prefer to design for reuse. While avoiding the trap of overengineering, always be aware of a longer-term view \-\- you can always decide to reject it, but always be aware of it \-\- so that you save time and effort in the long run both for yourself and for all the grateful users who will be happily reusing your code with their own new classes well into the new millennium.

**Notes**
1. H. Sutter. Exceptional C++ (Addison-Wesley, 2000).

2. It is also possible to have HolderPtr itself perform the construction of the owned `Y` object, but I will omit that for clarity and because it pretty much just gives `HolderPtr<Y>` the same `Y` value semantics, begging the question "then why not just use a plain old Y member?"

3. Alternatively, one might also choose to provide a traits object as an additional `HolderPtr` template parameter Traits that just defaults to `HPTraits<T>`, the same way `std::basic_string` does:

``` cpp
   template<class T, class Traits = HPTraits<T> >
   class HolderPtr { /*...*/ };
```

so that it's even possible for users to have different `HolderPtr<X>` objects in the same program copy in different ways. That didn't seem to make a lot of sense in this particular case, so I didn't do it, but it shows what's possible.



# 063 Amok Code 
**Difficulty: 4 / 10**
Sometimes life hands you some debugging situations that seem just plain deeply weird. Try this one on for size, and see if you can reason about possible causes for the problem.

----

**Problem**
1. One programmer has written the following code:
``` cpp
//--- file biology.h
// ... appropriate includes and other stuff ...

class Animal
{
public:
    // Functions that operate on this object:
    //
    virtual int Eat    ( int ) { /*...*/ }
    virtual int Excrete( int ) { /*...*/ }
    virtual int Sleep  ( int ) { /*...*/ }
    virtual int Wake   ( int ) { /*...*/ }
    
    // For animals that were once married, and
    // sometimes dislike their ex-spouses, we provide:
    //
    int EatEx    ( Animal* a ) { /*...*/ };
    int ExcreteEx( Animal* a ) { /*...*/ };
    int SleepEx  ( Animal* a ) { /*...*/ };
    int WakeEx   ( Animal* a ) { /*...*/ };
    
    // ...
};

// Concrete classes.
//
class Cat    : public Animal { /*...*/ };
class Dog    : public Animal { /*...*/ };
class Weevil : public Animal { /*...*/ };
// ... more cute critters ...

// Convenient, if redundant, helper functions.
//
int Eat    ( Animal* a ) { return a->Eat( 1 );     }
int Excrete( Animal* a ) { return a->Excrete( 1 ); }
int Sleep  ( Animal* a ) { return a->Sleep( 1 );   }
int Wake   ( Animal* a ) { return a->Wake( 1 );    }
```

Unfortunately, the code fails to compile. The compiler rejects the definition of at least one of the ...Ex() functions with an error message saying the function has already been defined.

To get around the compile error, the programmer comments out the ...Ex() functions, and now the program compiles and he starts testing the sleeping functions. Unfortunately, the Animal::Sleep() member function doesn't seem to always work correctly; when he tries to call the member function Animal::Sleep() directly, all is well. But when he tries to call it through the Sleep() free function wrapper, which does nothing but call the member function version, sometimes nothing happens... not all the time, only in some cases. Finally, when the programmer goes into the debugger or the linker- generated symbol map in an attempt to diagnose the problem, he can't seem to even find the code for Animal::Sleep() at all.

Is the compiler on the fritz? Should the programmer send the compiler vendor an angry flame e-mail and submit an irate letter to the New York Times? Is it a Year 2000 problem? Or is it just due to a rogue virus caught from the Internet?

What's going on?

----

**Solution**
**1. One programmer has written the following code:**

[...]

What's going on?

In short, there are several things that might be going on to cause these symptoms, but there's one outstanding possibility that would account for all of the observed behavior: Yes, you guessed it, an ugly combination of macros running amok and a side dish of mixed intentional and unintentional overloading.

**Motivation**
Certain popular C++ programming environments give you macros that are deliberately designed to change function names. Usually they do this for "good" or "innocent" reasons, namely backward and forward API compatibility; for example, if a Sleep() function in one version of an operating system is replaced by a SleepEx(), the vendor supplying the header in which the functions are declared might decide to "helpfully" provide a macro that automatically changes Sleep to SleepEx.

This is not a good idea. Macros are the antithesis of encapsulation, because their actual range of effect cannot be controlled, not even by the macro writer.

**Macros Don't Care**
Macros are obnoxious, smelly, sheet-hogging bedfellows for several reasons, most of which are related to the fact that they are a glorified text-substitution facility whose effects are applied during preprocessing, before any C++ syntax and semantic rules can even begin to apply. The following are some of macros' less charming habits.

1. Macros change names \-\- more often than not to harm, not protect, the innocent.

It is an understatement to say that this silent renaming can make debugging somewhat confusing. Such macro renaming means that your functions aren't actually called what you think they're called.

For example, consider our nonmember function Sleep():
``` cpp
int Sleep  ( Animal* a ) { return a->Sleep( 1 );   }
```
You won't find Sleep() anywhere in the object code or the linker map file because there's really no Sleep() function at all. It's really called SleepEx(). At first, when you're wondering where your Sleep() went, you might think, "ah, maybe the compiler is automatically inlining Sleep() for me," because that would explain why the short function doesn't seem to exist \-\- although of course the compiler shouldn't be inlining things without first being so directed. If you jump to conclusions and fire off an angry email to your compiler vendor complaining about aggressive optimizations, though, you're blaming the wrong company (or, at least, the wrong department).

Some of you may already have encountered this unfortunate and demonstrably bad effect. If you're like me, which is to say easily irritated by compiler weirdnesses and not satisfied with simple explanations, your curiosity bump might get the better of you. Then, curious, you may fire up the debugger and deliberately step into the function... only to be taken to the correct source line where the phantom function (which still looks like it has its original name, in the source code) lives, stepping into the phantom function that indeed works and is indeed getting called, but which by all other accounts doesn't seem to exist. Usually at this point it's a short step to figuring out what's really going on and a sotto voce grumbling at stupid macro tricks.

But wait, it gets better:

1(b). C++ already has features to deal with names. This causes what might be best termed an "unhealthy interaction."

You might think that it's not such a big deal to change a function's name. All right, fine, often it's not. But say you change a function's name to be the same as another function that also exists... what does C++ do if it finds two functions with the same name? It overloads them. That's not quite so fine when you don't realize it's happening silently.

This, alas, seems to be the case with Sleep(). The whole reason the library vendor decided to "helpfully" provide a Sleep macro to automatically rename things to SleepEx is because both such functions in fact do already exist in the vendor's library. Consider that the functions may have different signatures; then when we write our own Sleep() function, we might well be aware of the overloading on the library-supplied Sleep() and take care to avoid ambiguities or other errors that might cause us problems. We might even rely on the overloading because we want to provide library-Sleep()-like behavior intentionally. If, instead, our function is getting silently overloaded with some other function, not only is the overload going to behave differently than we expect, but if our original overloading was intentional it's not going to happen at all, at least not in the way we thought.

In the context of our question, such a Sleep-renaming macro can partly explain why different functions could end up being called in different circumstances; which function gets called can depend on how the overload resolution happened to work out for the specific types used at different call sites. Sometimes it's ours, and sometimes it's the library's. It just depends, perhaps in nonobvious ways.

If this sordid tale ended with the above lamentable effects on nonmember functions, that would be bad enough. Unfortunately, like shrapnel from a grenade, there's dangerous stuff flying in several other directions too:

2. Macros don't care about type.

The original intent for the Sleep macro described above was to change a global nonmember function's name. Unfortunately, the macro will change the name Sleep wherever it finds it; if we happen to have a global variable named Sleep, that member's name will silently change too. This is altogether a bad thing.

3. Macros don't care about scope.

Worse still, a macro designed to change a global nonmember function name will happily change any matching function (or other) names that happen to be members of your classes or nicely encapsulated within your own namespaces. In this case, we wrote a class with both Sleep() and SleepEx() functions; many of the described problems could be accounted for at least in part by a Sleep-renaming macro that makes our own functions invisibly overload \-\- with each other. Indeed, as with the invisible overloading mentioned above under point #1, this can explain why sometimes an unexpected member function can be called, depending on how the overload resolution happened to work out for the specific types used at different call sites.

If you think this is a Bad Thing, you're right. It's like having some ungloved uncertified doctor (the injudicious library header writer) with dirty hands (unsterilized macros) slice open your torso (class or namespace) and reach into your body cavity to rearrange things (members and other code)... while sleepwalking (not even realizing they're doing it).

**Summary**
In short, macros don't care about much of anything.

   **Guideline:** Avoid macros.

Your default response to macros should be something like "Macros! Ew, yuck!" unless there's a compelling reason to use them in special cases where they are not a hack. Macros are not type-safe, they're not scope-safe... they're just not safe, period. If you must write macros, avoid putting them in header files, and try to give them long and personalized names that are highly unlikely to ever tromp upon things that they aren't intentionally meant to beat into unwilling submission and grind under their heels into the dust.

   **Guideline:** Prefer to use namespaces.

True, as noted above, macros don't respect scope, and that includes namespace scope. However, had the free functions in the question been put inside a namespace, it would have prevented at least some of the problems; specifically, it would have avoided the global overloading with the vendor-supplied functions.

In short, practice good encapsulation. Not only does good encapsulation make for better designs, but it can defend you against unexpected threats you might not even see coming. Macros are the antithesis of encapsulation, because their actual range of effect cannot be controlled, not even by the macro writer. Classes and namespaces are among C++'s useful tools to help manage and minimize interdependencies between parts of your program that should be unrelated, and judicious use of these and other C++ facilities to promote encapsulation will not just make for superior designs, but will at the same time offer a measure of protection against the shrapnel that ill-considered code from fellow programmers, however well-intentioned, might occasionally send your way.



# 064 Standard Library Member Functions 
**Difficulty: 5 / 10**
Reuse is good, but can you always reuse the standard library with itself? Here is an example that might surprise you, where one feature of the standard library can be used portably with any of your code as much as you like, but it cannot be used portably with the standard library itself.

----

**Problem**
**JG Question**
1. What is std::mem_fun? When would you use it? Give an example.

**Guru Question**
2. Assuming a correct incantation in the indicated comment, is the following expression legal and portable C++? Why or why not?
``` cpp
std::mem_fun</*...*/>( &(std::vector<int>::clear) )
```


**Solution**
**Fun With mem_fun**
**1. What is std::mem_fun? When would you use it? Give an example.**

The standard mem_fun adapter lets you use member functions with standard library algorithms and other code that normally deals with free functions.

For example, given:
``` cpp
class Employee {
public:
    int DoStandardRaise() { /*...*/ }
    //...
};

int GiveStandardRaise( Employee& e )
{
    return e.DoStandardRaise();
}

vector<Employee> emps;
```
We might be used to writing code like the following:
``` cpp
std::for_each( emps.begin(), emps.end(), &GiveStandardRaise );
```
But suppose GiveStandardRaise() didn't exist, or for some other reason we needed to call the member function directly? Then we can write the following:
``` cpp
std::for_each( emps.begin(), emps.end(),
std::mem_fun_ref( &Employee::DoStandardRaise ) );
```
The "_ref" bit at the end of the name mem_fun_ref is a bit of an historical oddity. When writing code like the above, you should just remember to say "mem_fun_ref" if the container is a plain old container of objects, since for_each will be operating on references to those objects, and say "mem_fun" if it's a container of pointers to objects:
``` cpp
std::vector<Employee*> emp_ptrs; // <- note "*"
std::for_each( emp_ptrs.begin(), emp_ptrs.end(),
std::mem_fun( &Employee::DoStandardRaise ) );
```
You'll probably have noticed that, for clarity, I've been showing how to do this with functions that take no parameters. You can use the bind... helpers to deal with some functions that take an argument, and the principle is the same. Unfortunately you can't use this approach for functions that take two or more arguments. Still, it can be useful.

And that, a nutshell, is mem_fun. This brings us to the awkward part:

**Use mem_fun With Anything, Except the Standard Library**
**2. Assuming a correct incantation in the indicated comment, is the following expression legal and portable C++? Why or why not?**
``` cpp
std::mem_fun</*...*/>( &(std::vector<int>::clear) )
```
First, note that no "incantation" should be necessary. I deliberately wrote the question this way because as of this writing some popular compilers cannot correctly deduce the template parameters. For such compilers, depending on your implementation of the standard library, you would have to write something like:
``` cpp
std::mem_fun<void, std::vector<int, std::allocator<int>>>(&(std::vector<int>::clear));
```
Over time, this limitation will go away and compilers will be able to let you reliably omit the template parameters.

You might wonder why in the above I wrote "depending on your implementation of the standard library"... after all, the signature of std::vector<int>::clear() is that it takes no parameters and returns void, right? The standard tells us so, doesn't it?

Wrong (maybe), and that gets us to the crux of the problem.

The standard library specification deliberately gives some leeway to implementers when it comes to member functions. Specifically:

- A member function signature with default parameters may be replaced by "two or more member function signatures with the equivalent behavior."

- A member function signature may have additional defaulted parameters.

Aye, and there, in the second item, is the rub: Those pesky "might-be-there-or-might-not," "now-you-see-them- now-you-don't" extra parameter critters \-\- for short, let's call them "peekaboo" parameters \-\- are what cause our problem in this case.

Much of the time, any extra implementation-specific defaulted "peekaboo" parameters just go unnoticed; for example, when you call a member function you'll get the default values for the peekaboo parameters, so you don't need to ever be aware that the library implementer has thrown a few extra parameters on the end of the member function's signature. Unfortunately, such possible extra parameters do become very noticeable if you need to be sure of the exact signature of the member function \-\- such as when trying to use mem_fun. Note that this is true even if your compiler deduces template arguments correctly, because of two potential problems:

1. If the member function in question actually takes a parameter and you didn't expect one, you need to write something like bind2nd to get rid of it. Of course, now your code won't work on implementations that tack on an extra parameter of a different type, or none at all \-\- but, hey, your code wasn't portable anyway, right?

2. If the member function in question actually has two or more parameters (even if they're defaulted), you can't use it with mem_fun at all. Bummer \-\- but again, your code wasn't portable anyway, right?

\[Note: There's actually yet another final problem lurking here. As currently specified, the standard mem_fun adapters only work with const member functions, and `vector<int>::clear()` is a non-const member function. It seems to be clear that mem_fun was intended to work with non-const member functions, too, and presumably that will soon be addressed in a Technical Corrigendum to the C++ standard and by library implementers (who can do it even in advance of a TC; after all, doing it would be just another extension).]

In practice, though, the problem may not be all that bad. I don't know whether library implementers widely avail themselves of the leeway to add extra parameters, or intend to do so in the future. To the extent that they don't do so, you won't encounter the above difficulties much in practice.

Unfortunately, though, that's not quite the end of the story. Finally, consider a more general consequence of this leeway:

**Use Pointers To Member Functions With Anything, Except the Standard Library**
Alas, there's an even more basic issue: It is impossible to portably create a pointer to a standard library member function, period.

After all, if you want to create a pointer to a function, member or not, you have to know the pointer's type, which means you have to know the function's signature. Because the signatures of standard library member functions are impossible to know exactly \-\- unless you peek in your library implementation's header files to look for any peekaboo parameters, and even then the answer might change on a new release of the same library \-\- the bottom line is that you can't reliably form pointers to standard library member functions and still have portable code.

**Conclusion**
Do you think it's a little odd that you can portably use a standard library facility, namely mem_fun, with everything except the standard library itself? Do you think it's odd that you can portably use a language feature, namely a pointer to member function, with everything except the language's own standard library?

If you do, and you care about this, let the committee know by posting to comp.std.c++. You will no doubt generate, and learn and benefit from, much detailed discussion about the benefits you get from the implementer's leeway to alter function signatures and how those weigh against the portability benefits you might get if that leeway should be revoked in the future.

Let your voice be heard, and also listen to the resulting feedback; dialogue is only useful when it's two-way, but when it is two-way it's very useful indeed \-\- for you the users, and for the committee that serves you.



# 065 Try and Catch Me 
**Difficulty: 3 / 10**
Is exception safety all about writing try and catch in the right places? If not, then what? And what kinds of things should you consider when developing an exception safety policy for your software?

----

**Problem**
**JG Question**
1. What is a try-block?

**Guru Questions**
2. "Writing exception-safe code is fundamentally about writing 'try' and 'catch' in the correct places." Discuss.

3. When should try and catch be used? When should they not be used? Express the answer as a good coding standard guideline.

----

**Solution**
**1. What is a try-block?**

A try-block is a block of code (compound statement) whose execution will be attempted, followed by a series of one or more handler blocks that can be entered to catch an exception of the appropriate type if one is emitted from the attempted code. For example:
``` cpp
// Example 1: A try-block example

try
{
    if( some_condition )
        throw string( "this is a string" );
    else if( some_other_condition )
        throw 42;
}
catch( const string& )
{
    // do something if a string was thrown
}
catch( ... )
{
    // do something if anything else was thrown
}
```
In Example 1, the attempted code might throw a string, an integer, or nothing at all.

**2. "Writing exception-safe code is fundamentally about writing 'try' and 'catch' in the correct places." Discuss.**

Put bluntly, that statement reflects a fundamental misunderstanding of exception safety. Exceptions are just another form of error reporting, and we certainly know that writing error-safe code is not just about where to check return codes and handle error conditions.

Actually, it turns out that exception safety is rarely about writing '`try`' and '`catch`' \-\- and the more rarely the better. Also, never forget that exception safety affects a piece of code's design; it is never just an afterthought that can be retrofitted with a few extra catch statements as if for seasoning.

There are three major considerations when writing exception-safe code:

   a) Where and when should I throw?

This consideration is about writing 'throw' in the right places:

- What code should throw? That is, what errors will we choose to report by throwing an exception instead of by returning a failure value or using some other method?

- What code shouldn't throw? In particular, what code should provide the AC (nothrow) guarantee? (See GotW #61.)

   b) Where and when should I handle an exception?

This consideration is indeed about writing '`try`' and '`catch`' in the right places:

- What code could catch? That is, what code has enough context and knowledge to handle the error being reported by the exception (possibly by translating the exception into another form)? In particular, note that the catching code also needs to have enough knowledge to perform any necessary cleanup, such as of dynamic resources.

- What code should catch? That is, of the code that could catch the exception, which is best suited to do so?

Using the "resource acquisition is initialization" idiom can eliminate many try-blocks. If you wrap dynamically allocated resources in owner-manager objects, typically the destructor can perform automatic cleanup at the right time without any '`try`' or '`catch`' at all. This is clearly desirable, not to mention that it's also usually easier to code now and to read later.

   GUIDELINE: Prefer handling exception cleanup automatically using destructors instead of `try/catch`.

   c) In all other places, is my code going to be safe if an exception comes roaring through out of any given function call?

This consideration is about using good resource management to avoid leaks, maintaining class and program invariants, and other kinds of program correctness. Put another way, it's about keeping the program from blowing up just because an exception happens to pass from its throw site through code that shouldn't have to particularly care about it, before arriving at an appropriate handler. For most programmers I've encountered, it turns out that this is typically by far the most time-consuming and difficult-to-learn aspect of exception safety.

Notice that only one of these three considerations has anything to do with writing '`try`' and '`catch`'. And even that one can often be avoided with the judicious use of destructors to automate cleanup.

**3. When should try and catch be used? When should they not be used? Express the answer as a good coding standard guideline.**

Here's one suggestion. In brief:

1. Determine an overall error reporting and handling policy for your application or subsystem, and stick to it. In particular, the policy should cover the following basic aspects (and generally includes much more):

   a) Error Reporting: Define what kinds of errors are to be reported using exceptions as opposed to other error reporting methods. Generally it's good to choose the most readable and maintainable method for each case by default; for example, exceptions are most useful for constructors and operators that cannot emit return values, or where the throw site and the handler are widely separated.
   
   b) Error Propagation: Among other things, define the boundaries that exceptions shall not cross; typically these are module or API boundaries.
   
   c) Error Handling: Among other things, mandate that owning objects and destructors be used to manage cleanup instead of try/catch, wherever possible.

2. Write 'throw' in the places that detect an error and cannot deal with it themselves. (Clearly, code that can resolve an error immediately doesn't need to report it!)

For every operation, document what exceptions the operation might throw, and why, as part of the documentation for every function and module. You don't need to actually write an exception specification on each function, but you do need to document clearly and rigorously what the caller can expect, because error semantics are part of the function's or module's interface.

3. Write '`try`' and '`catch`' in the places that have sufficient knowledge to handle the error, to translate it, or to enforce boundaries defined in #1. In particular, I've found that there are three main reasons to write "try/catch":

   a) To handle an error. This is the simple case: An error happened, we know what to do about it, and we do it. Life goes on (sans the original exception, which has been safely put to rest). Again, do this in a destructor if possible; if not, go ahead and use try/catch.

   b) To translate an exception. This means catching one exception that is reporting a lower-level problem, and throwing another that is couched in the context of the translating code's own higher-level semantics. Alternatively, the original exception can be translated to some other representation, such as an error code.

For example, consider a communications session utility class that works across many host types and transport protocols: An attempt to open a session to another host can fail for any number of low-level reasons that the session class can detect (for example, a failure to detect the network, or authentication/permission rejection from the remote host). The Open() function can handle these conditions itself, and there's no use reporting them to the caller, who after all has no idea what a Foo packet is or what to do if it Barifies; the session class handles its internal low-level errors directly, keeps itself in a consistent state, and reports its own higher-level error or exception to inform its caller that the session could not be opened.

``` cpp
void Session::Open( /*...*/ )
{
    try
    {
        // entire operation
    }
    catch( const ip_error& err )
    {
        // - do something about an IP error
        // - clean up
        throw Session::OpenFailed();
    }
    catch( const KerberosAuthentFail& err )
    {
        // - do something about an authentication error
        // - clean up
        throw Session::OpenFailed();
    }
    
    // ... etc. ...
}
```

   c) To catch(...) on subsystem boundaries or other runtime firewalls. This usually also involves translating the error, usually to an error code or other non-exceptional representation. For example, when your stack unwinds up to a C API, you only have two choices: Return an error code right away for the current API function, or set an error state that the caller can query later using a complementary GetLastError() API function.

**Summary**
A wise man once said:

"Lead, follow, or get the blazes out of the way!"

In exception safety analysis, we might say instead:

"Throw, catch, or get the blazes out of the way!"

In practice, the last get-out-of-the-way case accounts for the bulk of exception safety analysis and testing. That's the major reason why exception-safe coding is not fundamentally about writing 'try' and 'catch' in the right places. Rather, it's fundamentally about getting out of the bullet's way in the right places!



# 066 Constructor Failures 
**Difficulty: 7 / 10**
What exactly happens when a constructor emits an exception? What if the exception comes from an attempt to construct a subobject or member object? This issue of GotW analyzes one aspect of C++ in detail, shows why it should work the way that it does, and demonstrates the implications for constructor exception specifications.

----

**Problem**
**JG Question**
1. Consider the following class:
``` cpp
// Example 1
//
class C : private A
{
  B b_;
};
```

In the C constructor, how can you catch an exception thrown from the constructor of a base subobject (such as A) or a member object (such as B)?

**Guru Questions**
2. In Example 1, if the A or B constructor throws an exception, is it possible for the C constructor to absorb the exception, and emit no exception at all? Justify your answer, explaining by example why this is as it should be.

3. What are the minimal requirements that A and B must meet in order for us to safely put an empty throw- specification on C's constructor(s)?

----

**Solution**
This issue of GotW was inspired by questions and a lengthy email exchange with Bobby Schmidt, renowned columnist for CUJ and MSDN Online's "Deep C++." For more information about Bobby's treatments of the topics addressed below, be sure to check out the very interesting reading in the online references mentioned in Note 1.[1]
``` cpp
**1. Consider the following class:**

// Example 1
//
class C : private A
{
  B b_;
};
```

In the C constructor, how can you catch an exception thrown from the constructor of a base subobject (such as A) or a member object (such as B)?

The short answer is: By using a function-try-block.

For example, here's how we can write a function-try-block for C's constructor:
``` cpp
// Example 1(a): Constructor function-try-block

C::C()
try
  : A ( /*...*/ ) // optional initialization list
  , b_( /*...*/ )
{
}
catch( ... )
{
  // We get here if either A::A() or B::B() throws.

  // If A::A() succeeds and then B::B() throws, the
  // language guarantees that A::~A() will be called
  // to destroy the already-created A base subobject
  // before control reaches this catch block.
}
```
The more interesting, question, though is: Why would you want to do this? That question is at the heart of this GotW issue.

**Object Lifetimes, and What a Constructor Exception Means**
In order to answer Question #2 correctly, we need only to fully understand object lifetimes[2] and what it means for a constructor to throw an exception.

Take a simple example:
``` cpp
    {
        Parrot parrot;
    }
```
In the above code, when does the object's lifetime begin? When does it end? Outside the object's lifetime, what is the status of the object? Finally, what would it mean if the constructor threw an exception? Take a moment to think about these questions before reading on.

Let's take these items one at a time:

Q: When does an object's lifetime begin?

A: When its constructor completes successfully and returns normally. That is, control reaches the end of the constructor body or an earlier return statement.

Q: When does an object's lifetime end?

A: When its destructor begins. That is, control reaches the beginning of the destructor body.

Q: What is the state of the object after its lifetime has ended?

A: As a well-known software guru[3] once put it, speaking about a similar code fragment and anthropomorphically referring to the local object as a "he":
   > He's not pining! He's passed on! This parrot is no more! He has ceased to be! He's expired and gone to meet his maker! He's a stiff! Bereft of life, he rests in peace! If you hadn't nailed him to the perch he'd be pushing up the daisies! His metabolic processes are now history! He's off the twig! He's kicked the bucket, he's shuffled off his mortal coil, run down the curtain and joined the bleedin' choir invisible! THIS IS AN EX-PARROT!
   >                                                     - Dr. M. Python, BMath, MASc, PhD (CompSci)

   \[The actual code example being commented upon by the above guru was only slightly different:
   ``` cpp
   {
     Parrot();
     const Parrot& perch = Parrot();
   
     // ... more code; at this point, only the first
     // temporary object is pushing up daisies ...
   }
   ```
   Get it? It's a lifetime-of-temporaries-bound-to-references joke. Remember it for Bay area parties.]

Kidding aside, the important point here is that the state of the object before its lifetime begins is exactly the same as after its lifetime ends \-\- there is no object, period. This observation brings us to the key question:

Q: What does emitting an exception from a constructor mean?

A: It means that construction has failed, the object never existed, its lifetime never began. Indeed, the only way to report the failure of construction \-\- that is, the inability to correctly build a functioning object of the given type \-\- is to throw an exception. (I will presently comment on the obsolete "if you get into trouble just set a status flag to 'bad' and let the caller check it via an IsOK() function" programming convention.)

In biological terms, conception took place \-\- the constructor began --, but despite best efforts it was followed by a miscarriage \-\- the constructor never ran to term(ination).

Incidentally, this is why a destructor will never be called if the constructor didn't succeed \-\- there's nothing to destroy. "It cannot die, for it never lived." Note that this makes the phrase "an object whose constructor threw an exception" really an oxymoron. Such a thing is even less than an ex-object... it never lived, never was, never breathed its first.

We might summarize the C++ constructor model as follows:

   Either:

   (a) The constructor returns normally by reaching its end or a return statement, and the object exists.

   Or:

   (b) The constructor exits by emitting an exception, and the object not only does not now exist, but never existed.

There are no other possibilities. Armed with this information, we can now fairly easily tackle Question #2.

**I Can't Keep No Caught Exceptions[7]**
**2. In Example 1, if the A or B constructor throws an exception, is it possible for the C constructor to absorb the exception, and emit no exception at all?**

If you didn't consider the object lifetime rules, you might have tried something like the following:
``` cpp
// Example 2(a): Absorbing exceptions?
//
C::C()
try
  : A ( /*...*/ ) // optional initialization-list
  , b_( /*...*/ )
{
}
catch( ... )
{
  // ?
}
```

If the handler body contained the statement "throw;" then the catch block would obviously rethrow whatever exception A::A() or B::B() had emitted. What's less obvious, but clearly stated in the standard, is that if the catch block does not throw (either rethrow the original exception, or throw something new), and control reaches the end of the catch block of a constructor or destructor, then the original exception is automatically rethrown.

Think about what this means: A constructor or destructor function-try-block's handler code MUST finish by emitting some exception. There's no other way. The language doesn't care what exception it is that gets emitted \-\- it can be the original one, or some other translated exception \-\- but an exception there must be! It is impossible to keep any exceptions thrown by base or member subobject constructors from causing some exception to leak beyond their containing constructors.

In fewer words, it means that:

If construction of any base or member subobject fails, the whole object's construction must fail.

This is no different than saying "there is no way for a human to exist (i.e., be born alive) if any of its vital organs (e.g., heart, liver) were never formed." If you tried to continue by keeping at least those parts you were able to make, the result may be a hunk of flesh, probably rotting, but it was never a human being. There is no such thing as an "optional" base or member (held by value or reference) subobject.

A constructor cannot possibly recover and do something sensible after one of its base or member subobjects' constructors throws. It cannot even put its own object into a "construction failed" state... its object is not constructed, it never will be constructed no matter what Frankensteinian efforts the handler attempts in order to breathe life into the non-object, and whatever destruction can be done has already been done automatically by the language \-\- and that includes all base and member subobjects.

What if your class can honestly have a sensible "construction partially failed" state \-\- i.e., it really does have some "optional" members that aren't strictly required and the object can limp along without them, possibly with reduced functionality? Then use the Pimpl idiom to hold the possibly-bad parts of the object at arm's length. For similar reasoning, see Exceptional C++ Items 31-34 about abuses of inheritance[4]; incidentally, this "optional parts of an object" idea is another great reason to use delegation instead of inheritance whenever possible, because base subobjects can never be made optional because you can't put base subobjects into a Pimpl.

    *Aside: Convergence is funny sometimes. Long after I started pushing the Pimpl idiom and bashing needless inheritance, I kept on coming across new problems that were solved by using Pimpl or removing needless inheritance, especially to improve exception safety. I guess it shouldn't have been a surprise because it's just this whole coupling thing again: Higher coupling means greater exposure to failure in a related component. To this comment, Bobby Schmidt responded:*
    
    *And maybe that's the core lesson to pull out of this \-\- we've really just rediscovered and amplified the old minimal-coupling-maximum-cohesion axiom.[5]*

I've always had a love/hate relationship with exceptions, but even so I've always had to agree that exceptions are the right way to signal constructor failures given that constructors cannot report errors via return codes (ditto for most operators). I have found the "if a constructor encounters an error, set a status bit and let the user call IsOK() to see if construction actually worked" method to be outdated, dangerous, tedious, and in no way better than throwing an exception.

**Toward Some Morals**
Incidentally, this also means that the only (repeat only) possible use for a constructor function-try-block is to translate an exception thrown from a base or member subobject. That's Moral #1. Next, Moral #2 says that destructor function-try-blocks are entirely usele\-\-

"--But wait!" I hear someone interrupting from the middle of the room. "I don't agree with Moral #1. I can think of another possible use for constructor function-try-blocks, namely to free resources allocated in the initializer list or in the constructor body!"

Sorry, nope. After all, remember that once you get into your constructor try-block's handler, any local variables in the constructor body are also already out of scope, and you are guaranteed that no base subobjects or member objects exist any more, period. You can't even refer to their names. Either the parts of your object were never constructed, or those that were constructed have already been destroyed. So you can't be cleaning up anything that relies on referring to a base or member of the class (and anyway, that's what the base and member destructors are for, right?).

**Aside: Why Does C++ Do It That Way?**
To see why it's good that C++ does it this way, let's put that restriction aside for the moment and imagine, just imagine, that C++ did let you mention member names in those handlers. Then imagine the following case, and try to decide: Should the handler delete t_ or z_? (Again, ignore for the moment that in real C++ it can't even refer to t_ or z_.)
``` cpp
// Example 2(b): Very Buggy Class
//
class X : Y {
    T* t_;
    Z* z_;
public:
    X()
    try
      : Y(1)
      , t_( new T( static_cast<Y*>(this) )
      , z_( new Z( static_cast<Y*>(this), t_ ) )
    {
        /*...*/
    }
    catch(...)
    // Y::Y or T::T or Z::Z or X::X's body has thrown
    {
        // Q: should I delete t_ or z_? (note: not legal C++)
    }
};
```

First, we cannot possibly know whether t_ or z_ were ever allocated, so neither delete could be safe.

Second, even if we did know that we had reached one of the allocations, we probably can't destroy *t_ or *z_ because they refer to a Y (and possibly a T) that no longer exists and they may try to use that Y (and possibly T). Incidentally, this means that not only can't we destroy *t_ or *z_, but they can never be destroyed by anyone!

If that didn't just sober you up, it should have. I have seen people write code similar in spirit to the above, never imagining that they were creating objects that, should the wrong things happen, could never be destroyed! The good news is that there's a simple way to avoid the problem: These difficulties would largely go away if the `T*` members were auto_ptrs or similar manager objects.

Finally, if Y::~Y() can throw, it is not possible to reliably create an X object at any time! If you haven't been sobered yet, this should definitely do it. If Y::~Y() can throw, even writing "X x;" is fraught with peril. This reinforces the dictum that destructors must never be allowed to emit an exception under any circumstances, and writing a destructor that could emit an exception is simply an error. Destruction and emitting exceptions don't mix.

The above side discussion was to help better understand why the rules are as they are. After all, as noted, you can't even refer to t_ or z_ inside the handler anyway. I've refrained from quoting standardese elsewhere in this GotW, so here's your dose... from the C++ standard, clause 15.3, paragraph 10:
   > Referring to any non-static member or base class of an object in the handler for a function-try-block of a constructor or destructor for that object results in undefined behavior.

**Some Morals**
Therefore the status quo can be summarized as follows:
   **Moral #1:** Constructor function-try-block handlers have only one purpose \-\- to translate an exception. (And maybe to do logging or some other side effects.) They are not useful for any other purpose.

   **Moral #2:** Since destructors should never emit an exception, destructor function-try-blocks have no practical use at all.[6] There should never be anything for them to detect, and even if there were something to detect because of evil code, the handler is not very useful for doing anything about it because it can not suppress the exception.

   **Moral #3:** Always perform unmanaged resource acquisition in the constructor body, never in initializer lists. In other words, either use "resource acquisition is initialization" (thereby avoiding unmanaged resources entirely) or else perform the resource acquisition in the constructor body.

For example, building on Example 2(b), say T was char and t_ was a plain old `char*` that was `new[]`'d in the initializer-list; then in the handler there would be no way to `delete[]` it. The fix would be to instead either wrap the dynamically allocated memory resource (e.g., change `char*` to `string`) or `new[]` it in the constructor body where it can be safely cleaned up using a local try-block or otherwise.

   **Moral #4:** Always clean up unmanaged resource acquisition in local try-block handlers within the constructor or destructor body, never in constructor or destructor function-try-block handlers.

   **Moral #5:** If a constructor has an exception specification, that exception specification must allow for the union of all possible exceptions that could be thrown by base and member subobjects. As Holmes might add, "It really must, you know." (Indeed, this is the way that the implicitly generated constructors are declared; see GotW #69.)

   **Moral #6:** If a constructor of a member object can throw but you can get along without said member, hold it by pointer and use the pointer's nullness to remember whether you've got one or not, as usual. Use the Pimpl idiom to group such "optional" members so you only have to allocate once.

And finally, one last moral that overlaps with the above but is worth restating in its own right:

   **Moral #7:** Prefer using "resource acquisition is initialization" to manage resources. Really, really, really. It will save you more headaches than you can probably imagine.

**Justifying the Rules**
From legality, we now turn to morality:

Justify your answer, explaining by example why this is as it should be.

In short, the way the language works is entirely correct and easily defensible, once you think about the meaning of C++'s object lifetime model and philosophy.

A constructor exception must be propagated, for there is no other way to signal that the constructor failed. Two cases spring to mind:
``` cpp
// Example 2(c): Auto object
//
{
    X x;
    g( x ); // do something else
}
```
If X's construction fails \-\- whether it's due to X's own constructor body code, or due to some X base subobject or member object construction failure \-\- control must not continue within f(). After all, there is no x object! The only way for control not to continue in f()'s body is to emit an exception. Therefore a failed construction of an auto object must result in some sort of exception, whether it be the same exception that caused the base or member subobject construction failure or some translated exception emitted from an X constructor function try block.

Similarly:
``` cpp
// Example 2(d): Array of objects
//
{
    X ax[10];
    // ...
}
```
If the 5th X object construction fails \-\- whether it's due to X's own constructor body code failing, or due to some X base subobject or member object construction failing \-\- control must not continue within the scope. After all, if you tried to continue, you'd end up with an array not all of whose objects really exist.

**A Final Word: Failure-Proof Constructors?**
We could slyly rephrase Question #2 as follows: Is it possible to write and enforce an empty throw-specification for a constructor of a class, if some base or member constructor could throw? After all, to enforce a "throws nothing" guarantee for any function, we must be able to absorb any possible exceptions that come our way from lower-level code, to avoid accidentally trying to emit them to our own caller.

This, not coincidentally, brings us to the final question:

**3. What are the minimal requirements that A and B must meet in order for us to safely put an empty throw- specification on C's constructor(s)?**

Now that we've done all the hard work, this one's easy: For a constructor to have an empty throw-specification, all base and member subobjects must be known to never throw (whether they have throw-specs that say so or not).

An empty throw-specification on a constructor declares to the world that construction cannot fail. If for whatever reason it can indeed fail, then the empty throw-spec isn't appropriate, that's all.

**Notes**
1. R. Schmidt. ["Handling Exceptions in C and C++, Parts 14-16"](http://msdn.microsoft.com/library/default.asp?url=/library/en-us/dndeepc/html/deep02032000.asp?frame=true). (The link is to Part 15; you can navigate from there.)

2. For simplicity, I'm speaking only of the lifetime of an object of class type that has a constructor.

3. The inventor of the Python programming language?

4. H. Sutter. Exceptional C++ (Addison-Wesley, 2000).

5. R. Schmidt. Private correspondence, February 2, 2000.

6. Not even for logging or other side effects, because there shouldn't be any exceptions from base or member subobject destructors and therefore anything you could catch in a destructor function-try-block could be caught equally well using a normal try-block inside the destructor body.

7. A double pun, can be sung to the chorus of "Satisfaction" or to the opening bars of "Another Brick in the Wall, Part N."


# 067 Double or Nothing 
**Difficulty: 4 / 10**
No, this issue isn't about gambling. It is, however, about a different kind of "float," so to speak, and lets you test your skills about basic floating-point operations in C and C++.

----

**Problem**
**JG Question**
1. What's the difference between "float" and "double"?

**Guru Question**
2. Say that the following program takes 1 second to run, which is not unusual for a modern desktop computer:
``` cpp
int main()
{
    double x = 1e8;
    while( x > 0 )
    {
        --x;
    }
}
```

How long would you expect it to take if you change "double" to "float"? Why?

----

**Solution**
**1. What's the difference between "float" and "double"?**

Quoting from section 3.9.1/8 of the C++ standard:

There are three floating point types: float, double, and long double. The type double provides at least as much precision as float, and the type long double provides at least as much precision as double. The set of values of the type float is a subset of the set of values of the type double; the set of values of the type double is a subset of the set of values of the type long double.

**The Wheel of Time**
**2. Say that the following program takes 1 second to run, which is not unusual for a modern desktop computer:**
``` cpp
int main()
{
    double x = 1e8;
    while( x > 0 )
    {
        --x;
    }
}
```

How long would you expect it to take if you change "double" to "float"? Why?

It will probably take either about 1 second (on a particular implementation floats may be somewhat faster, as fast, or somewhat slower than doubles), or forever, depending whether or not float can exactly represent all integer values from 0 to 1e8 inclusive.

The above quote from the standard means that there may be values that can be represented by a double but that cannot be represented by a float. In particular, on some popular platforms and compilers, double can exactly represent all integer values in [0,1e8] but float cannot.

What if float can't exactly represent all integer values from 0 to 1e8? Then the modified program will start counting down, but will eventually reach a value N which can't be represented and for which N-1 == N (due to insufficient floating-point precision)... and then the loop will stay stuck on that value until the machine on which the program is running runs out of power (due to local power outage or battery life limits), its operating system crashes (more common on some platforms than others), Sol turns out to be a variable star and scorches the inner planets, or the universe dies of heat death, whichever comes first.[1]

**A Word About Narrowing Conversions**
Some people might wonder: "Well, besides universal heat death, isn't there another problem? The constant 1e8 has type double. So if we just changed 'double' to 'float' the program wouldn't compile because of the narrowing conversion, right?" Well, let's quote standardese again, this time from section 4.8/1:

   > An rvalue of floating point type can be converted to an rvalue of another floating point type. If the source value can be exactly rep- resented in the destination type, the result of the conversion is that exact representation. If the source value is between two adjacent destination values, the result of the conversion is an implementation- defined choice of either of those values. Otherwise, the behavior is undefined.

This means that a double constant can be implicitly (i.e., silently) converted to a float constant, even if doing so loses precision (i.e., data). This was allowed to remain for C compatibility and usability reasons, but it's worth keeping in mind when you do floating-point work.

A quality compiler will warn you if you try to do something that's undefined behavior, namely put a double quantity into a float that's less than the minimum, or greater than the maximum, value that a float is able to represent. A really good compiler will provide an optional warning if you try to do something that may be defined but could lose information, namely put a double quantity into a float that is between the minimum and maximum values representable by a float, but which can't be represented exactly as a float.

**Notes**
1. Indeed, because the program keeps the computer running needlessly, it also needlessly increases the entropy of the universe, thereby hastening said heat death. In short, such a program is quite environmentally unfriendly and should be considered a threat to our species. Don't write code like this.[2]

2. Of course, performing any kind of additional work, whether by humans or machines, also increases the entropy of the universe, thereby hastening heat death. This is a good argument to keep in mind for times when your employer requests extra overtime.



# 068 Flavors of Genericity 
**Difficulty: 7 / 10**
How generic is a generic function, really? The answer can depend as much on its implementation as on its interface, and a perfectly generalized interface can be hobbled by simple \-\- and awkward-to-diagnose \-\- programming lapses.

----

**Problem**
**JG Question**
**1.** What's the purpose of C++'s template feature?

**Guru Questions**
The code in the following questions is taken from the box on page 42 of Exceptional C++.[1]

**2.** What are the semantics of the following function? Be as complete as you can, and be sure to explain why there are two template parameters and not just one.
``` cpp
// Example 2(a): construct().

template <class T1, class T2>
void construct(T1* p, const T2& value)
{
    new (p) T1(value);
}
```

**3.** There is a subtle genericity trap in the following functions. What is it, and what's the best way to fix it?
``` cpp
// Example 3(a): destroy().

template <class T>
void destroy(T* p)
{
    p->~T();
}

template <class FwdIter>
void destroy(FwdIter first, FwdIter last)
{
    while(first != last)
    {
        destroy(first);
        ++first;
    }
}
```

**4.** What are the semantics of the following function, including the requirements on `T`? Is it possible to remove any of those requirements? If so, demonstrate how, and argue whether doing so is a good idea or a bad idea.
``` cpp
// Example 4(a): swap().
//
template <class T>
void swap( T& a, T& b )
{
    T temp(a); a = b; b = temp;
}
```

----

**Solution**
**1. What's the purpose of C++'s template feature?**

Templates provide powerful compile-time polymorphism. Misuse of templates can cause really hard-to-read error messages on many compilers, but templates are also one of C++'s most powerful features.

When we think of polymorphism in an object-oriented world, we think of the kind of runtime polymorphism we get from using virtual functions. A base class establishes an interface "contract" as defined by a set of virtual functions, and derived classes may inherit from the base class and override the virtual functions in a way that preserves the contracted semantics. Then other code that expects to work on a Base object (and accepts the Base object by pointer or reference) can work equally well with a Derived object:
``` cpp
// Example 1(a): Ye olde garden-variety runtime
// polymorphism. Doesn't draw big crowds these days.
//
class Base
{
public:
  virtual void f();
  virtual void g();
};

class Derived : public Base
{
  // override f() and/or g() if desired
};

void h( Base& b )
{
  b.f();
  b.g();
}

int main()
{
  Derived d;
  h( d );
}
```

This is great stuff, and gives a lot of runtime flexibility. The main drawbacks of runtime polymorphism are that the types must be related in a hierarchy derived from a common base class, and when the virtual functions are called in a tight loop you may notice some performance penalty because normally each call to a virtual function must be made through an extra pointer indirection, as the compiler figures out the Derived function you "really" mean to call.

If you know the types you're using at compile time, then you can get around both of the above drawbacks: You can use types that are not related by inheritance, as long as they provide the expected operations:
``` cpp
// Example 1(b): Ye newe C++-variety compile-time
// polymorphism. Powerful stuff. We're still
// finding out just what kinds of nifty things
// this makes possible.
//
class Xyzzy
{
public:
    void f( bool someParm = true );
    void g();
    void GoToGazebo();
    // ... more functions ...
};

class Murgatroyd
{
public:
    void f();
    void g( double two = 6.28, double e = 2.71828 );
    int HeavensTo( const Z& ) const;
    // ... more functions ...
};

template<class T>
void h( T& t )
{
    t.f();
    t.g();
}

int main()
{
    Xyzzy x;
    Murgatroyd m;
    
    h( x );
    h( m );
}
```

As long as both objects x and m are of a type that provides member functions `f()` and `g()` that can be called with no parameters,` h()` will work. In Example 1(b), both types actually have different signatures for `f()` and `g()`, and they also provide additional functions beyond just those two, but `h()` doesn't care... as long as `f()` and `g()` can be called without parameters, the compiler will allow `h()` to make the calls. Of course, when called, those functions should also do something that's sensible for `h()`!

**2. What are the semantics of the following function? Be as complete as you can, and be sure to explain why there are two template parameters and not just one.**
``` cpp
// Example 2(a): construct().

template <class T1, class T2>
void construct( T1* p, const T2& value )
{
    new (p) T1(value);
}
```
`construct()` constructs an object in a given memory location using an initial value. The form of new used here is called "`placement new,`" and instead of allocating memory for the new object, it just puts it into the memory pointed at by p. Any object new'd in this way should generally be destroyed by calling its destructor explicitly (as shown in Question 3), rather than by using a delete expression.

Why two template parameters? Isn't one sufficient to make a copy of the value object? Well, if `construct()` had only one template parameter, then you would need to explicitly state the type of that parameter when copying from an object of a different type:
``` cpp
// Example 2(b): A less functional construct(),
// and why it's less functional.

template <class T1>
void construct( T1* p, const T1& value )
{
    new (p) T1(value);
}

// Assume that both p1 and p2 point to raw memory.

void f( double* p1, Base* p2 )
{
    Base b;
    Derived d;
    
    construct( p1, 2.718 );      // ok
    construct( p2, b );          // ok
    
    construct( p1, 42 ); // error: is T1 double or int?
    construct<double>( p1, 42 ); // ok
    
    construct( p2, d );  // error: is T1 Base or Derived?
    construct<Base>( p2, d );    // ok
}
```
The reason the two noted cases are ambiguous is that the compiler doesn't know enough to deduce the template parameter, and so the programmer is forced to nominate a template parameter explicitly. Yet shouldn't we allow programmers to silently construct a double from an integer value? Probably; the worst that could happen is that we might lose some precision. Shouldn't we allow programmers to silently construct a Base from a Derived? Possibly; if Base allows that, then slicing would occur but that can be a legitimate choice of operation.

Assuming that we want to allow the programmer to be able to do such things without explicitly naming types, we need to use the originally-presented version that has two independent template parameters.

**3. There is a subtle genericity trap in the following functions. What is it, and what's the best way to fix it?**
``` cpp
// Example 3(a): destroy().

template <class T>
void destroy( T* p )
{
    p->~T();
}

template <class FwdIter>
void destroy( FwdIter first, FwdIter last )
{
    while( first != last )
    {
        destroy( first );
        ++first;
    }
}
```
`destroy()` destroys an object or a range of objects. The first version takes a single pointer, and calls the pointed-at object's destructor. The second version takes an iterator range, and iteratively destroys the individual objects in the designated range.

Still, there's a subtle trap here. This didn't make a difference in any example in the book, but it's a little odd: The two-parameter `destroy(FwdIter,FwdIter)` version is templatized to take any generic iterator, and yet it calls the one-parameter `destroy(T*)` by passing it one of the iterators \-\- which requires that FwdIter must be a plain old pointer! This needlessly loses some of the generality of templatizing on FwdIter. It also means you can get Really Obscure error messages when compiling code that tries to call `destroy(FwdIter,FwdIter)` with non-pointer iterators, because (at least one of) the actual failure(s) will be on the `destroy(first)` line inside the two-parameter version, which typically generates such useful messages as the following, taken from one popular compiler:
```
'void __cdecl destroy(template-parameter-1,template-parameter-1)' : expects 2 arguments - 1 provided

'void __cdecl destroy(template-parameter-1 *)' : could not deduce template argument for 'template-parameter-1 *' from '[the iterator type I was using]'
```
These error messages aren't as bad as some I've seen, and with only a little bit of extra reading they do actually tell you (mostly) what's going on. The first message indicates that the compiler was trying to resolve the line "`destroy(first)`" as a call to the two-parameter version; the second indicates an attempt instead to resolve it as a call to the one-parameter version. Both attempts failed, each for a different reason: The two-parameter version can take iterators, but needs two of them, not just one; and the one-parameter version can take just one parameter, but needs it to be a pointer. No dice.

Having said all that, in reality we'd almost never want to use `destroy()` with anything but pointers in the first place just because of the way the function is intended to be used, given that it turns things back into raw memory and all. Still, only a simple change is needed to let FwdIter be any iterator type, not just a pointer, so why not do it: Have `destroy(iter,iter)` call the destructor explicitly. In the two-parameter version of `destroy()`, change:
``` cpp
destroy( first );
```
To:
``` cpp
destroy( &*first );
```
This will almost always work. After all, we can usually expect that, for an iterator iter, the expression `&*iter` really does yield a pointer to the `T` object to which the iterator refers. Consider why it would usually work fine: All standard-conforming iterators [2] are required to supply an `operator*()` that returns a true `T&`. This is one of the reasons why proxied containers are not supported by the C++ standard; for more information about this and related issues, see the discussion of the expression `&*t.begin()` in GotW #50 [3] and my May 1999 C++ Report column.[4] (It is possible, if rare, to make "`destroy( &*first );`" not work: As Astute Reader Bill Wade pointed out, that formulation will fail to work if `T` overrides its `operator&()` to return something besides the object's address.)

What's the moral of the story? Beware subtle genericity drawbacks when implementing one generic function in terms of another. In this case, there was a subtle principal drawback: The two-parameter version wasn't as generic for iterators as we originally thought. There was also an even subtler secondary drawback: The quick fix of changing "`destroy( first );`" to "`destroy( &*first );`" introduced a new requirement on `T`, namely that it provide an `operator&()` with normal semantics, namely one that really returns a pointer to the object. Both traps were neatly avoided when we stopped implementing one function in terms of the other.

**4. What are the semantics of the following function, including the requirements on T? Is it possible to remove any of those requirements? If so, demonstrate how, and argue whether doing so is a good idea or a bad idea.**
``` cpp
// Example 4(a): swap().
//
template <class T>
void swap( T& a, T& b )
{
    T temp(a); a = b; b = temp;
}
```
`swap()` just exchanges two values using the copy constructor and copy assignment operator. It therefore requires that `T` have a copy constructor and a copy assignment operator.

If that's all you said, give yourself half marks only. One of the important things to note about the semantics of any function is its exception safety status, including what guarantees it provides: In this case, `swap()` is not at all exception-safe if T's copy assignment operator can throw an exception. In particular, if `T::operator=()` can throw but is atomic (all-or-nothing), then if the second assignment fails we exit via an exception but a has already been modified; if additionally `T::operator=()` can throw but is not atomic, then it is possible for `swap()` to exit via an exception but both parameters may have been modified and one may now contain neither of the two values. Therefore this `swap()` must be documented as follows:

- if `T::operator=()` doesn't throw, `swap()` gives the aC guarantee (all-or-nothing, except for side effects of T operations);[5]

- otherwise, if `T::operator=()` can throw:

- if `T::operator=()` is atomic, and `swap()` exits by means of an exception, the first argument might or might not already have been modified;

- otherwise, if `T::operator=()` isn't atomic, and `swap()` exits by means of an exception, both arguments might or might not already have been modified, and one of them might contain neither of the original two values.

There are two ways to remove the requirement that T have an assignment operator, and the first additionally provides better exception safety:

1. Specialize or overload `swap()`. Say that we have a class MyClass that uses the common idiom of providing a nonthrowing `Swap()`. Then we can specialize standard functions for MyClass as follows.
``` cpp
// Example 4(b): Specializing swap().
//
class MyClass
{
public:
    void Swap( MyClass& ) throw();
    // ...
};

namespace std
{
    template<> swap<MyClass>( MyClass& a, MyClass& b ) // throw()
    {
        a.Swap( b );
    }
}
```
Alternatively, we can overload standard functions for MyClass as follows:
``` cpp
// Example 4(c): Overloading swap().

class MyClass
{
public:
    void Swap( MyClass& ) throw();
    // ...
};

// NOTE: Not in namespace std.
swap( MyClass& a, MyClass& b ) // throw()
{
    a.Swap( b );
}
```
Doing this is usually a good idea \-\- even if T does have an assignment operator that would allow the original version to work!

For example, the standard library itself overloads `swap()` for `vector<>` so that calling swap() actually invokes `vector<>::swap()`. This makes a lot of sense, because `vector<>::swap()` can be much more efficient by avoiding making any copies of the vectors' data at all. The primary template above would create a complete new copy (temp) of one of the vectors, then perform additional copying from one vector to the other, then perform additional copying from temp to the other vector, which results in a lot of T operations and has complexity O(N) where N is the combined size of the vectors being swapped. The specialized version will typically simply just assign a few pointers and integral types, and runs in constant (and usually negligible) time.

So, if you create a type that provides a swap()-like operation, it's usually a good idea to specialize `std::swap()` (or provide your own overloaded swap() in another namespace) that's specific to your new type. It will usually be more efficient than a routine application of the primary `std::swap()` template's brute-force procedure, and will often improve `swap()`'s own exception safety.

2. Destroy-and-reconstruct. The idea here is to write `swap()` in terms of T's copy constructor instead of its copy assignment operator, and of course this only works if T indeed has a copy constructor:
``` cpp
// Example 4(d): swap() without assignment.

template <class T>
void swap( T& a, T& b )
{
    if( &a != &b ) // note: this check is now necessary!
    {
        T temp( a );
        
        destroy( &a );
        construct( &a, b );
        
        destroy( &b );
        construct( &b, temp );
    }
}
```
First, this is never appropriate if `T` copy assignment can throw, because then you get all the exception safety problems of the original version, only in spades: You can get into situations where the objects hold not only indeterminate values, but no longer exist at all!

If we know that T copy assignment is guaranteed not to throw, though, this version does have the extra ability to deal with types that can't be assigned but can be copy-constructed, and there are indeed many such types. Being able to swap such types is -not- necessarily a good thing, though, because if a type can't be assigned then it's probably set up that way for a good reason \-\- for example, it likely doesn't have value semantics, and it may have const or reference members \-\- and so providing a mechanism to "imbue" (or "impose") value semantics may be misguided and produce surprising and incorrect results.

Worse still, this approach plays games with object lifetimes, and that's always questionable. Here by "plays games" I mean that it changes not only the values, but the very -existence-, of the operated-upon objects. Code using the Example 4(d) form of swap() could easily produce surprising results when users forget about the unusual destruction semantics.

A guideline: If you must play games with object lifetimes and you know that doing so is okay, and you're certain that the operated-upon objects' copy constructors can never thow, and you're very sure that the unusually "imposed" value semantics will be all right in your application for those specific objects, then (and only then) you may legitimately decide to use such an approach in those specific situations only \-\- but even then, don't make such an operation a general template that could be accidentally instantiated for -any- type, and be very sure to document the blazes out of it so that the poor unsuspecting user next door knows what to expect, because this technique falls squarely into the "unusual" section of the C++ coding catalog.

**Notes**
1. H. Sutter. [Exceptional C++](http://www.gotw.ca/publications/xc++.htm) (Addison-Wesley, 2000).

2. Note that this does not include vector<bool>::iterator, which is not a conforming iterator.

3. Available at [http://www.gotw.ca/gotw/050.htm](http://www.gotw.ca/gotw/050.htm).

4. Sutter, H. ["When Is a Container Not a Container?"](http://www.gotw.ca/publications/mill09.htm) (C++ Report, 11(5), May 1999). This discusses some of the evils of vector<bool>.

5. See GotW #61.



# 069 Enforcing Rules for Derived Classes 
**Difficulty: 5 / 10**
Too many times, just being at the top of the (inheritance) world doesn't mean that you can save programmers of derived classes from simple mistakes. But sometimes you can! This issue is about safe design of base classes, so that derived class writers have a more difficult time going wrong.

----

**Problem**
**JG Question**
1. When are the following functions implicitly declared and implicitly defined for a class, and with what semantics? Be specific, and describe the circumstances under which the implicitly defined versions cause the program to be illegal (not well-formed).

   a) default constructor
   
   b) copy constructor
   
   c) copy assignment operator
   
   d) destructor

2. What functions are implicitly declared and implicitly defined for the following class X? With what signatures?
``` cpp
class X
{
    auto_ptr<int> i_;
};
```

**Guru Question**
3. Say that you have a base class which requires that all derived classes not use one or more of the implicitly declared and defined functions. For example:
``` cpp
class Count
{
public:
    // The Author of Count hereby documents that derived
    // classes shall inherit virtually, and that all their
    // constructors shall call Count's special-purpose
    // constructor only.
    
    Count( /* special parameters */ );
    Count& operator=( const Count& ); // does the usual
    virtual ~Count();                 // does the usual
};
```
Unfortunately, programmers are human too, and they sometimes forget that they should write two of the functions explicitly.
``` cpp
class BadDerived : private virtual Count
{
    int i_;
    
    // default constructor:
    //   should call special ctor, but does it?
    // copy constructor:
    //  should call special ctor, but does it?
    // copy assignment: ok?
    // destructor: ok?
};
```
In the context of this example, is there a way for the author of Count to force derived classes to be coded correctly \-\- that is, to break at compile time (preferable) or run time (at minimum) if not coded correctly?

More generally, Is there any way that the author of a base class can force authors of derived classes to explicitly write each of these four basic operations? If so, how? If not, why not?

----

**Solution**
**Implicitly Generated Functions (or, What the Compiler Does For/To You)**
In C++, four class member functions can be implicitly generated by the compiler: The default constructor, the copy constructor, the copy assignment operator, and the destructor.

The reason for this is a combination of convenience and backward compatibility with C. Recall that C-style structs are just classes consisting of only public data members; in particular, they don't have any (explicitly defined) member functions, and yet you do have to be able to create, copy, and destroy them. To make this happen, the C++ language automatically generates the appropriate functions (or some appropriate subset thereof) to do the appropriate things, if you don't define appropriate operations yourself.

This issue of GotW is about what all of those "appropriate" words mean.

**1. When are the following functions implicitly declared and implicitly defined for a class, and with what semantics? Be specific, and describe the circumstances under which the implicitly defined versions cause the program to be illegal (not well-formed).**

In short, an implicitly declared function is only implicitly defined when you actually try to call it. For example, an implicitly declared default constructor is only implicitly defined when you try to create an object using no constructor parameters.

Why is it useful to distinguish between when the function is implicitly declared and when it's implicitly defined? Because it's possible that the function might never be called, and if it's never called then the program is still legal even if the function's implicit definition would have been illegal.

For convenience, throughout this article unless otherwise noted "member" means "nonstatic class data member." I'll also say "implicitly generated" as a catchall for "implicitly declared and defined."

**Exception Specifications of Implicitly Declared Functions**
In all four of the cases where a function can be implicitly declared, the compiler will make its exception specification just loose enough to allow all exceptions that could be allowed by the functions the implicit definition would call. For example, given:
``` cpp
// Example 1(a): Illustrating the exception
// specifications of implicitly declared functions
//
class C // ...
{
    // ...
};
```
Because there are no constructors explicitly declared, the implicitly generated default constructor has the semantics of invoking all base and member default constructors. Therefore the exception specification of C's implicitly generated default constructor must allow any exception that any base or member default constructor might emit. If -any- base class or member of C has a default constructor with no exception specification, the implicitly declared function can throw anything:
``` cpp
// public:
inline C::C(); // can throw anything
```
If every base class or member of C has a default constructor with an explicit exception specification, the implicitly declared function can throw any of the types mentioned in those exception specifications:
``` cpp
// public:
inline C::C() throw();
    // anything that a C base or member default
    // constructor might throw; i.e., the union of
    // all types mentioned in C base or member
    // default constructor exception specifications                
```
It turns out that there's a potential trap lurking here. Consider: What if one of the implicitly generated functions overrides an inherited virtual function? This can't happen for constructors (because constructors are never virtual), but it can happen for the copy assignment operator (if you try hard, and make a base version that matches the implictly generated derived version's signature[1]), and it can happen pretty easily for the destructor:
``` cpp
// Example 1(b): Danger, Will Robinson!
class Derived;

class Base
{
public:
    // Somewhat contrived, and almost certainly deplorable,
    // but technically it is possible to use this trick to
    // declare a Base assignment operator that takes a
    // Derived argument; be sure to read [3] before even
    // thinking about trying anything like this:
    //
    virtual Base& /* or Derived& */
    operator=( const Derived& ) throw( B1 );
    
    virtual ~Base() throw( B2 );
};

class Member
{
public:
    Member& operator=( const Member& ) throw( M1 );
    ~Member() throw( M2 );
};

class Derived : public Base
{
    Member m_;
    
    // implicitly declares four functions:
    //   Derived::Derived();                 // ok
    //   Derived::Derived( const Derived& ); // ok
    //   Derived& Derived::operator=( const Derived& )
    //            throw( B1, M1 ); // error, ill-formed
    //   Derived::~Derived()
    //            throw( B2, M2 ); // error, ill-formed
};
```
What's the problem? The two functions are ill-formed because whenever you override any inherited virtual function, your derived function's exception specification must be at least as restrictive as the version in the base class. That only makes sense, after all: If it weren't that way, it would mean that code that calls a function through a pointer to the base class could get an exception that the base class version promised not to emit. For instance, if the context of Example 1(b) were allowed, consider the code:
``` cpp
Base* p = new Derived;

// Ouch -- this could throw B2 or M2, even though
// Base::~Base() promised to throw at most B2:
delete p;
```
Guideline: This is Yet Another Good Reason why every destructor should have either an exception specification of "`throw()`" or none at all. Besides, destructors should never throw anyway, and should always be written as though they had an exception specification of "`throw()`" even if that specification isn't written explicitly.[2]

Guideline: This is also Yet Another Good Reason to be careful about virtual assignment operators. See [3] for more about the hazards of virtual assignment and how to avoid them.

Now let's consider the four implicitly generated functions one at a time:

**Implicit Default Constructor**
a) default constructor

A default constructor is implicitly declared if you don't declare any constructor of your own. An implicitly declared default constructor is public and inline.

An implicitly declared default constructor is only implicitly defined when you actually try to call it, has the same effect as if you'd written an empty default constructor yourself, and can throw anything that a base or member default constructor could throw. It is illegal if that empty default constructor would also have been illegal had you written it yourself (for example, it would be illegal if some base or member doesn't have a default constructor).

**Implicit Copy Constructor**
b) copy constructor

A copy constructor is implicitly declared if you don't declare one yourself. An implicitly declared default constructor is public and inline, and will take its parameter by reference to const if possible (it's possible if and only if every base and member has a copy constructor that takes its parameter by reference to const or const volatile too), and by reference to non-const otherwise.

Yes indeed, just like most C++ programmers, the standard itself pretty much ignores the volatile keyword a lot of the time. Although the compiler will take pains to tack "const" onto the parameter of an implicitly declared copy constructor (and copy assignment operator) whenever possible, frankly \-\- to use the immortal words of Clark Gable in Gone with the Wind \-\- it doesn't give a hoot about tacking on "volatile." Oh well, that's life.

An implicitly declared copy constructor is only implicitly defined when you actually try to call it to copy an object of the given type, performs a memberwise copy of its base and member subobjects, and can throw anything that a base or member copy constructor could throw. It is illegal if any base or member has an inaccessible or ambiguous copy constructor.

**Implicit Copy Assignment Operator**
c) copy assignment operator

A copy assignment operator is implicitly declared if you don't declare one yourself. An implicitly declared default constructor is public and inline, returns a reference to non-const that refers to the assigned-to object, and will take its parameter by reference to const if possible (it's possible if and only if every base and member has a copy assignment operator that takes its parameter by reference to const too), and by reference to non-const otherwise. As already noted above, volatile can go hang.

An implicitly declared copy assignment operator is only implicitly defined when you actually try to call it to assign an object of the given type, performs a memberwise assignment of its base and member subobjects (including possibly multiple assignments of virtual base subobjects), and can throw anything that a base or member copy constructor could throw. It is illegal if any base or member is const, is a reference, or has an inaccessible or ambiguous copy assignment operator.

**Implicit Destructor**
d) destructor

A destructor is implicitly declared if you don't declare any destructor of your own. An implicitly declared destructor is public and inline.

An implicitly declared destructor is only implicitly defined when you actually try to call it, has the same effect as if you'd written an empty destructor yourself, and can throw anything that a base or member destructor could throw. It is illegal if any base or member has an inaccessible destructor, or if any base destructor is virtual and not all base and member destructors have identical exception specifications (see Exception Specifications of Implicitly Declared Functions, above).

**An auto_ptr Member**
**2. What functions are implicitly declared and implicitly defined for the following class X? With what signatures?**
``` cpp
// Example 2
//class X
{
    auto_ptr<int> i_;
};
```
The following functions are implicitly declared as public members. Each is implicitly defined, with the indicated effects, when you write code that tries to use it.
``` cpp
inline X::X() throw()
// : i_()
{
}

inline X::X( X& other ) throw()
  : i_( other.i_ )
{
}

inline X& X::operator=( X&) throw()
{
    i_ = other.i_;
    return *this;
}

inline X::~X() throw()
{
}
```
The copy constructor and copy assignment operators take references to non-const because they can \-\- that's what auto_ptr's versions do. Similarly, all of the above functions have "throws-nothing" specifications because they can \-\- no related auto_ptr operation throws, and indeed no auto_ptr operation at all can throw.

Note that the copy constructor and copy assignment operator transfer ownership. That may not be what the author of X necessarily wants, and so often X should provide its own versions of these functions. (For more details about this and related topics, see also GotW #62.[4])

**Renegade Children and Other Family Problems**
**3. Say that you have a base class which requires that all derived classes not use one or more of the implicitly declared and defined functions. For example:**
``` cpp
// Example 3
//class Count
{
public:
    // The Author of Count hereby documents that derived
    // classes shall inherit virtually, and that all their
    // constructors shall call Count's special-purpose
    // constructor only.
    
    Count( /* special parameters */ );
    Count& operator=( const Count& ); // does the usual
    virtual ~Count();                 // does the usual
```
Here we have a class that wants its derived child classes to play nice and call Count's special constructor, perhaps so that Count can keep track of the number of objects of derived types created in the system. That's a good reason to require virtual inheritance, in order to avoid double-counting if some derived class happens to inherit multiply in such a way that Count is a base class more than once.[5]

Interestingly, did you notice that Count may have a design error already? It has an implicitly generated copy constructor, which probably isn't what is wanted to keep track of a correct count. To disable that, simply declare it private without a definition:
``` cpp
private:
    // undefined, no copy construction
    Count( const Count& );
};
```
So Count wants its derived child classes to behave. But kids don't always play nice, do they? Indeed, we don't have to look far to find an example of badly behaved problem child:

Unfortunately, programmers are human too, and they sometimes forget that they should write two of the functions explicitly.
``` cpp
class BadDerived : private virtual Count
{
    int i_;

    // default constructor:
    //   should call special ctor, but does it?
```
In short, no, the default constructor not only doesn't call the special constructor, but there's an even more fundamental concern: Is there even a BadDerived default constructor at all? The answer, which probably isn't reassuring, is: Sort of. There is an implicitly- declared default constructor (okay), but if you ever try to call it the program becomes ill-formed (oops).

Let's see why this is so. First, BadDerived doesn't define any of its own constructors, so a default constructor will be implicitly declared. That's cool. But, the minute you try to use that constructor (i.e., the minute you try to create a BadDerived object, which you might think is kind of an important thing to be able to do, and you'd be right), that default constructor gets implicitly defined \-\- or at least it should be, but because that implicit definition is supposed to call a base default constructor that doesn't exist, the program is ill-formed. Bottom line, any program that tries to create a BadDerived object is not a conforming C++ program, and for that reason BadDerived is properly viewed as delinquent.

So is there a default constructor? Sort of. It's declared, but you can't call it, which makes it considerably less useful. When kids go renegade like this, it's just not a happy family.
``` cpp
    // copy constructor:
    //  should call special ctor, but does it?
```
For similar reasons, the implicitly generated copy constructor will be declared but, when defined, won't call the special Count constructor. With the Count class as originally shown, this copy constructor will simply call Count's implicitly generated copy constructor.

If we decide to suppress Count's implicitly generated copy constructor, as indicated earlier, then this BadDerived would have a copy constructor implicitly declared, but since it can't be implicitly defined (because Count's wouldn't be accessible) any attempt to use it would make the program not valid C++.

Fortunately, now the news starts getting a little better:
``` cpp
    // copy assignment: ok?
    // destructor: ok?
};
```
Yes, the implicitly generated copy assignment operator and destructor will both do the right thing, namely invoke (and, in the destructor's case, override) the base class versions. So the good news is that at least something worked right.

Still, all is not happy in this class family. Every household must have some minimum of order, after all. Can we not find a way to give the parents better tools to keep the peace?

**Enforcing Rules For Derived Classes**
In the context of this example, is there a way for the author of Count to force derived classes to be coded correctly \-\- that is, to break at compile time (preferable) or run time (at minimum) if not coded correctly?

The idea is not to suppress the implicit declaration (we can't), but to make the implicit definition not well-formed, so that the compiler should emit an understandable error about it.

More generally, Is there any way that the author of a base class can force authors of derived classes to explicitly write each of these four basic operations? If so, how? If not, why not?

Well, as we went through the review of what happens to implicitly declare and define the four basic operations, we kept coming across the words "inaccessible" and "ambiguous." It turns out that adding ambiguous overloads, even with different access specifies, doesn't help much. It's hard to do much better than simply make the base class functions selectively inaccessible, by declaring them private (whether they're actually defined is optional) \-\- and this approach works for all of the functions but one.
``` cpp
// Example 4: Try to force derived classes to not use
// their implicitly generated functions, by making
// Base functions inaccessible.

class Base
{
public:
    virtual ~Base();
    
    // ... other special-purpose named functions that
    // derived classes are supposed to use...

private:
    Base( const Base& ); // undefined
    Base& operator=( const Base& ); // undefined
};
```
This Base has no default constructor (because a user-defined constructor has been declared, if not defined), and it has a hidden copy constructor and copy assignment operator. There's no way we could hide the destructor (which must always be accessible to derived classes after all), so it might as well be public.

The idea is that, even if we do want to support a form of a given operation (for example, copy assignment), if we can't do it with the usual function then we make the usual function inaccessible and provide a named, or otherwise distinguishable, function to do the work the way we want it done.

Where does this get us? Let's see:
``` cpp
class Derived : private Base
{
    int i_;

    // default constructor:
    //   declared, but definition ill-formed
    //   (there is no Base default constructor)
    
    // copy constructor:
    //   declared, but definition ill-formed
    //   (Base's copy constructor is inaccessible)
    
    // copy assignment:
    //   declared, but definition ill-formed
    //   (Base's copy assignment is inaccessible)
    
    // destructor: well-formed, will compile
};
```
Not bad... we got three compile-time errors out of a possible four, and it turns out that's pretty much the best we can do, or indeed need to do.

This simple solution can't handle the destructor case, but that's okay because destructors are less amenable to special-purpose replacement anyway; base versions must always be called, no two ways about it, and after all there can only be one destructor. The difficult part is usually getting any unusual constructors to be correctly called so that the base class is correctly initialized; the base class can then normally save the information it needs to do the right thing in the destructor.

There, that wasn't bad. Simple solutions are usually best. In this case there were some more complex alternatives; let's consider them briefly to reassure ourselves that none of them could do better for the destructor case, or any other case for that matter:

Alternative #1: Make base class functions ambiguous. This isn't any better, still doesn't break the implicitly generated destructor, and it's more work.

Alternative #2: Provide base versions that blow up, for example, by throwing a std::logic_error exception. This also doesn't break the implicitly generated destructor (without breaking all possible destructors), and it turns a compile-time error into a run-time error, which is not as good.

Guideline: Prefer compile-time errors to run-time errors.

Alternative #3: Provide base versions that are pure virtual. This is useless: It doesn't apply to constructors (either default or copy); it won't help with copy assignment, because the derived versions have different signatures; and it won't help with the destructor because the implicitly generated version will satisfy the requirement to define a destructor.

Alternative #4: Use a virtual base class without a default constructor, which forces each most-derived class to explicitly call the virtual base's constructor. This approach has potential for the two constructors, and has the additional advantage of working even for classes that aren't immediately derived from Base, so this is really the only alternative that could be used in conjunction with the above solution. Derived classes' implicitly generated copy assignment operators and destructors will still be valid, though.

**Notes**
1. Thanks to [Joerg Barfurth](joerg.barfurth@germany.sun.com) for pointing out this rare but possible case.

2. H. Sutter. Exceptional C++, "Item 16 (incl. Destructors That Throw and Why They're Evil)" (Addison-Wesley, 2000).

3. S. Meyers. More Effective C++, "Item 33: Make Non-Leaf Classes Abstract"(Addison-Wesley, 1996).

4. Available online at [http://ww.gotw.ca/gotw/062.htm](http://www.gotw.ca/gotw/062.htm).

5. This example is adapted from one by Marco Dalla Gasperina in his article "Counting Objects and Virtual Inheritance" (unpublished). His code didn't include the design errors I talk about next. That article was about something else, not about enforcing rules for derived classes, but his example made me think, "hmm, how could I find a way to prevent authors of derived classes from forgetting to use the special constructor when they use that Counter base class?" Thanks again for the inspiration, Marco!



# 070 Encapsulation 
**Difficulty: 4 / 10**
What exactly is encapsulation as it applies to C++ programming? What does proper encapsulation and access control mean for member data \-\- should it ever be public or protected? This issue focuses on alternative answers to these questions, and shows how those answers can increase either the robustness or the fragility of your code.

----

**Problem**
**JG Question**
1. What does "encapsulation" mean? How important is it to object- oriented design and programming?

**Guru Questions**
2. Under what circumstances, if any, should nonstatic class data members be made public, protected, and private? Express your answer as a coding standard guideline.

3. The std::pair class template uses public data members because it is not an encapsulated class but only a simple way of grouping data. Imagine a class template that is like std::pair but which additionally provides a deleted flag, which can be set and queried but cannot be unset. Clearly the flag itself must be private so as to prevent users from unsetting it directly. If we choose to keep the other data members public, as they are in std::pair, we end up with something like the following:
``` cpp
template<class T, class U>
class Couple
{
public:
  // The main data members are public...
  //
  T first;
  U second;

  // ... but there is still classlike machinery
  // and private implementation.
  //
  Couple() : deleted_(false) {}
  void MarkDeleted() { deleted_ = true; }
  bool IsDeleted() { return deleted_; }
private:
  bool deleted_;
};
```
Should the other data members still be public, as shown above? Why or why not? If so, is this a good example of why mixing public and private data in the same class might sometimes be good design?

----

**Solution**
**1. What does "encapsulation" mean?**

According to Webster's Third New International Dictionary:

   en-cap-su-late vt: to surround, encase, or protect in or as if in a capsule

Encapsulation in programming has precisely the same sense: To protect the internal implementation of a class by hiding those internals behind a surrounding and encasing interface visible to the outside world.

The definition of the word "capsule," in turn, gives good guidance as to what makes a good class interface:
   cap-sule \[F, fr. L /capsula/ small box, dim. of /capsa/ chest, case]
   1a: a membrane or saclike structure enclosing a part or organ ...
   2 : a closed container bearing spores or seeds ...
   4a: a gelatin shell enclosing medicine ...
   5 : a metal seal ...
   6 : ... envelope surrounding certain microscopic organisms ...
   9 : a small pressurized compartment for an aviator or astronaut ...

Note the recurring theme in the words:

a) Surround, encase, enclose, envelope: A good class interface hides the class's internals, presenting a "face" to the outside world that is separate and distinct from the internals. Because a capsule surrounds exactly one cohesive group of subobjects, its interface should likewise be cohesive \-\- its parts should be directly related.

The outer surface of a bacterium's capsule contains its means for sensing, touching, and interacting with the outside world, and the outside world with it. (Those means would be a lot less useful if they were inside.)

b) Closed, seal: A good class interface is complete, and does not expose any internals. The interface acts as a hermetic seal, and often acts as a code firewall (at compile time, runtime, or both) whereby outside code cannot depend on class internals and changes to class internals therefore cause no impact on outside code.

A bacterium whose capsule isn't closed won't live long; its internals will quickly escape, and the organism will die.

c) Protect, shell: A good class interface protects the internals against unauthorized access and manipulation. In particular, a primary job of the interface is to ensure that all access to and manipulation of internal structures is guaranteed to preserve class invariants.

The principal methods for killing bacteria (and humans) involve fashioning devices to break outer and/or inner capsules. On the micro level, these include chemicals, enzymes, or organisms (and possibly eventual nanomachines) responsible capable of making appropriate holes. On the macro level, knives and guns are perennial favorites, and many well-funded lobby groups encourage the "right and duty" to bear capsule-breaching armaments even of types useful only against Kevlar-encapsulated humans of the police variety, on the theory that deer may someday take the anticompetitive action of donning bulletproof vests.

\[See United States v. United Antlered Wildlife (UAW) Oregon Local 507, a landmark Department of Justice deer antitrust case currently under appeal. If the DoJ succeeds, the UAW will be divided into two specialized groups, one for soup and one for hunting trophies, neither of which will be permitted to wear vests or initiate other anticompetitive practices. Mr. H. Eston, prominent leader of pro-personal-ICBM group People for the Eating of Tasty Animals (PETA) announced support for the sanctions, stating that hunters have a constitutional right not to have to cope with prey that might defend itself. In related news, in a move apparently designed to defend against accusations of trademark infringement, UAW has just announced that it will soon ship to its members vests made of Protect# (pronounced "Protect- Sharp") which it judiciously describes without using the name "Kevlar" or the letter "K."]

**Encapsulation's Place in OO**
How important is it to object- oriented design and programming?

Encapsulation is the prime concept in object-oriented programming. Period.

Other OO techniques \-\- such as data hiding, inheritance, and polymorphism \-\- are important principally because they support special cases of encapsulation. For example, encapsulation nearly always implies data hiding; runtime polymorphism using virtual functions more completely separates the interface (provided by a base class) from the implementation (provided by the derived class, which need not even exist at the time that the code which will eventually use it is written); compile-time polymorphism using templates completely divorces interface from implementation, as any class having the required operations can be used interchangeably without requiring any inheritance or other relationship. Encapsulation is not always data hiding, but data hiding is always a form of encapsulation. Encapsulation is not always polymorphism, but polymorphism is always a form of encapsulation.

Object-orientation is often defined as:

   the bundling together of data and the functions that operate on that data.

That definition is true to a point \-\- it excludes nonmember functions that are also logically part of a class, such as operator<<() in C++ \-\- and it stresses high cohesion. It does not, however, adequately emphasize the other essential element of object-orientation, namely:

   the simultaneous separation of data from calling code through an interface of functions that operate on that data.

This complementary aspect stresses low coupling, and that the purpose of the assembled functions is to form a protective interface.

In short, object-orientation is all about separating interfaces from implementation in a way that promotes high cohesion and low coupling \-\- both of which have been known to be sound software engineering goals since long before objects were invented. These concepts address dependency management, which is one of the key concepts in modern software engineering, especially for large systems.[1] [2]

**Public, Protected, or Private Data?**
**2. Under what circumstances, if any, should nonstatic class data members be made public, protected, and private? Express your answer as a coding standard guideline.**

Normally we first look at the rule, then at the exception. This time, let's do things the other way around and consider the exception first.

The only exception to the general rule that follows is the case when all class members (both functions and data) are public, as with a C-style struct. In this case the "class" isn't really a full-fledged class with its interface, behavior, and invariants \-\- it's not even a half-fledged class; it's just a bundle-o-data. The "class" is merely a convenient bundling of objects, and that's fine, especially for backward compatibility with C programs that manipulate C-style structs.

Other than that special case, however, data members should always be private.

Public data is a breach of encapsulation because it permits calling code to manipulate the object's internals directly. This implies a high level of trust! After all, in real life, most other people don't get to manipulate my internals directly (e.g., by operating directly on my stomach) because they might then easily and unintentionally do the wrong thing; at best, they only get to manipulate my internals indirectly by going through my public interface with my knowledge and consent (e.g., by handing me a bottle labeled "Drink Me" which I will then decide to drink, or shampoo my hair with, or wash my car with, according to my own feelings and judgment). Of course, some people really are qualified to manipulate my internals directly (e.g., a surgeon), but even then: a) it's rare; and b) I get to elect whether or not to have the surgery, and if so in which surgeon I will declare the requisite high level of trust.

Similarly, most calling code shouldn't ever manipulate a class's internals directly (e.g., by viewing or changing member data) because they may quite easily and unintentionally do the wrong thing; at best, they only get to manipulate the class's internals indirectly by going through the class's public interface with the class's knowledge and consent (e.g., by handing a Bottle("Drink Me") object to a public member function, which will then decide what, if anything, to do with the object according to the class author's own feelings and judgment). Of course, some nonmember code may really be qualified to manipulate a class's internals directly (usually such code should be a member function, but for example operator<<() cannot be a member), but even then: a) it's rare; and b) the class gets to elect what such outside code will be declared a "friend" with that declaration's attendant high level of trust.

In short, public data is evil (except only for C-style structs).

Likewise, in short, protected data is evil (this time with no exceptions). Why is it evil? Because the same argument above applies to protected data, which is part of an interface too \-\- the protected interface, which is still an interface to outside code, just a smaller set of outside code, namely the code in derived classes. Why is there no exception? Because protected data is never just a bundle-o-data; if it were, it could only be used as such by derived classes, and since when do you use additional instances of one of your base classes as a convenient bundle-o-data? That would be bizarre. For most on the history of why protected data was originally permitted, and why even the person who campaigned for it now agrees it was a bad idea, see Stroustrup's The Design and Evolution of C++.[3]

From the GotW coding standards:

- encapsulation and insulation:

   - always keep class data members private (Lakos96: 65-69; Meyers92: 71-72; Murray93: 33-36)

   - except for the special case where all class members are public (e.g., C-style struct)

Related guidelines include:

- encapsulation and insulation:

   - avoid returning 'handles' to internal data, especially from const member functions (Meyers92: 96-99)

   - where reasonable, avoid showing private members of a class in its declaration: use the Pimpl Idiom

- programming style:

   - never subvert the language; for example, never attempt to break encapsulation by copying a class definition and adding a friend declaration, or by providing a local instantiation of a template member function

- design style:

   - prefer cohesion:

      - prefer to follow the "one class, one responsibility" principle

      - prefer to give each piece of code (e.g., each module, each class, each function) a single well-defined responsibility (Items 10, 12, 19, and 23)

**A General Transformation**
Now let's prove the "member data should always be private" guideline by assuming the opposite (that public/protect member data can be appropriate) and showing that in every such case the data should not in fact be public/protected at all.
``` cpp
// Example 2(a): Nonprivate data (evil)
//
class X
{
    // ...
public:
    T1 t1_;
protected:
    T2 t2_;
};
```
First, we note that this can always be transformed, without loss of either generality or efficiency, to:
``` cpp
// Example 2(b): Encapsulated data (good)
//
class X
{
    // ...
public:
    T1& UseT1() { return t1_; }
protected:
    T2& UseT2() { return t2_; }
private:
    T1 t1_;
    T2 t2_;
};
```
Therefore even if there's a reason to allow direct access to t1_ or t2_, there exists a simple transformation that causes the access to be performed through a(n initially inline) member function. Examples 2(a) and 2(b) are equivalent. But is there any benefit to using the method in Example 2(a)?

To prove that Example 2(a) should never be used, all that remains is to show that that:

1. Example 2(a) has no advantages not present in Example 2(b);

2. Example 2(b) has concrete advantages; and

3. Example 2(b) costs nothing.

Taking them in reverse order:

Point 3 is trivial to show. The inline function, which returns by reference and hence incurs no copying cost, will probably be optimized away entirely by the compiler.

Point 2 is easy: Just look at the source dependencies. In Example 2(a), all calling code that uses t1_ and/or t2_ mentions them explicitly by name; in Example 2(b), all calling code that uses t1_ or t2_ mentions only the names of the functions UseT1() and UseT2(). Example 2(a) is rigid, because any change to t1_ or t2_ (e.g., removing them and replacing them with something else, or just tacking on some instrumentation) requires all calling code to be changed to suit. In Example 2(b), however, instrumentation can be added, and t1_ and/or t2_ can even be removed entirely, without any change to calling code, because the member function completes the class's interface and "surrounds," "seals," and "protects" the internals.

Finally, Point 1 is demonstrated by observing that anything that calling code could do with t1_ or t2_ directly in Example 2(a) it can still do by using the member accessor in Example 2(b). The caller may have to write an extra pair of brackets, but that's it.

Let's consider a concrete example: Say you want to add some instrumentation, perhaps something as simple as counting the number of accesses to t1_ or t2_. If it's a data member, as in Example 2(a), here's what you have to do:

1. You create accessor functions that do what you want, and make the data private. (In other words, you do Example 2(b) anyway, only later as a retrofit.)

2. All your users get to experience the joy of finding and changing every use of t1_ and t2_ in their code to the functional equivalent. This is just bound to cause rejoicing among a user community with pressing deadlines who already have other real work to do. Your users may thank you profusely and buy you gifts as a reward; don't open them if they're ticking.

3. All your users recompile.

4. The compile will break if they missed any instances; fix them by repeating steps 2 and 3 until done.

If you already have simple accessor member functions, as in Example 2(b), here's what you have to do:

1. You make the change inside the existing accessor functions.

2. All your users relink (if the functions are in a separate .cpp and not inline), or at worst recompile (if the functions are in the header).

The worst part is that, in real life, if you started with Example 2(a) you may never even be allowed later to make the change to get to Example 2(b). The more users there are that depend on an interface, the more difficult it is to ever change the interface. This brings us to a law (not just a guideline):

Sutter's Law of Second Chances:

The most important thing to get right is the interface. Everything else can be fixed later.

Get the interface wrong, and you may never be allowed to fix it.

Once an interface is in widespread use, there may be so many people who depend on it that it becomes infeasible to change it. True, interfaces can always be extended (added to instead of changed) without impacting anyone, but just adding member functions doesn't help to fix existing parts that you later decide were a bad idea \-\- at most it lets you add alternative ways of doing things that will confuse your users, who will legitimately ask: "But there are two (or three, or N) ways of doing it... why? Which one do I use?"

In short, a bad interface can be difficult or impossible to fix after the fact. Do your best to get the interface right the first time, and make it surround, seal, and protect its internals.

**A Case In Point**
**3. The std::pair class template uses public data members because it is not an encapsulated class but only a simple way of grouping data.**

Note that the above is the exceptional valid use of public data. Even so, std::pair would have been no worse off with accessors instead of public data.

Imagine a class template that is like std::pair but which additionally provides a deleted flag, which can be set and queried but cannot be unset. Clearly the flag itself must be private so as to prevent users from unsetting it directly. If we choose to keep the other data members public, as they are in std::pair, we end up with something like the following:
``` cpp
// Example 3(a): Mixing public and private data?
//
template<class T, class U>
class Couple
{
public:
    // The main data members are public...
    //
    T first;
    U second;
    
    // ... but there is still classlike machinery
    // and private implementation.
    //
    Couple() : deleted_(false) {}
    void MarkDeleted() { deleted_ = true; }
    bool IsDeleted() { return deleted_; }
private:
    bool deleted_;
};
```
Should the other data members still be public, as shown above? Why or why not? If so, is this a good example of why mixing public and private data in the same class might sometimes be good design?

This Couple class was proposed as a counterexample to the above coding guidelines. It attempts to show a class that is "mostly a struct" but has some private housekeeping data. The housekeeping data (here a simple attribute) has an invariant. The claim is that updates to the attribute flag are totally independent of updates to the Couple's values.

Let's start with the last statement: I don't buy it. The updates may be independent, but the attribute is clearly not independent of the values, else it wouldn't be grouped together cohesively with them. Of course the deleted_ attribute isn't independent of the accompanying objects \-\- it applies to them!

Note how, instead of mixing public and private data, we can model the solution using accessors even if the accessors' initial implementation was to give up references:
``` cpp
// Example 3(b): Proper encapsulation, initially
// with inline accessors. Later in life, these
// may grow into nontrivial functions if needed;
// if not, then not.
//
template<class T, class U>
class Couple
{
    Couple() : deleted_(false) { }
    T& First()         { return first_;   }
    U& Second()        { return second_;  }
    void MarkDeleted() { deleted_ = true; }
    bool IsDeleted()   { return deleted_; }

private:
    T first_;
    U second_;
    bool deleted_;
};
```
"Huh?" someone might say. "Why bother writing do-nothing accessor functions?" Answer: As described with Example 2(b) above. If today calling code can change some aspect of this object (in this example, the tagalong "deleted_" attribute), tomorrow you may well want to add new features even if they do no more than add debugging information or add checks. Example 3(b) lets you do that, and that flexibility doesn't cost you anything in terms of efficiency because of the inline functions.

For example, say that one month in the future you decide that you want to check all attempted accesses to an object marked deleted:

- In Example 3(a), you can't, period \-\- not without changing the design and requiring all code that uses first_ and second_ to be rewritten.

- In Example 3(b), you simply put the check into the First() and Second() members. The change is transparent to all past and present users of Couple (at worst a recompile is needed; no code changes are needed).

It turns out that Example 3(b) has other practical side benefits in the real world. For example, as Nick Mein (nmein@trimble.co.nz) points out: "You can put a breakpoint (or whatever) in the accessor to find out just where and when the value is being modified. This can be pretty helpful in tracking down a bug."

**Summary**
Except for the case of a C-style struct (all members public), all data members should always be private. Doing otherwise violates all of the principles of encapsulation noted at the start of this article, and creates dependencies on the data names which makes it harder to later encapsulate them correctly. There is never a good reason to write public or protected data members because they can always be trivially wrapped in (at first) inline accessor functions at no cost, so it's always right to do the right thing the first time.

Get your interfaces right first. Internals are easy enough to fix later, but if you get the interface wrong you may never be allowed to fix it.

**Notes**
1. Robert C. Martin. Designing Object-Oriented Applications Using the Booch Method (Prentice-Hall, 1995).
2. Various 1996 articles by Martin, especially those with "Principle" in the title, at the [ObjectMentor website](http://www.objectmentor.com/publications/articlesByDate.html).
3. B. Stroustrup. The Design and Evolution of C++, Section 13.9 (Addison-Wesley, 1994).



# 071 Inheritance Traits? 
**Difficulty: 5 / 10**
This issue reviews traits templates, and demonstrates some cool traits techniques. What can a template figure out about its type \-\- and then what can it do about it? The answers are nifty and illuminating, and not just for people who write C++ libraries.

----

**Problem**
**JG Questions**
1. What is a traits class?

2. Demonstrate how to detect and make use of class members within templates, using the following motivating case: You want to write a templated class C that can only be instantiated on types having a function named Clone that takes no parameters and returns a pointer to the same kind of object.
``` cpp
// T must provide T* T::Clone() const
template<class T>
class C
{
    // ...
};
```
Note: It's obvious that if C just writes code that tries to invoke `T::Clone()` without parameters, then such code will fail to compile if there isn't a `T::Clone()` that can be called without parameters. But that's not enough to answer this question: Just trying to call `T::Clone()` without parameters would also succeed in calling a `Clone()` that has defaulted parameters and/or does not return a `T*`. The goal here is to specifically enforce that T provide a function that looks exactly like this: `T* T::Clone()`.

**Guru Questions**
3. A programmer wants to write a template that can require (or just detect) whether the type on which it is instantiated has a `Clone()` member function. The programmer decides to do this by requiring that classes offering such a `Clone()` must derive from a fixed Clonable class. Demonstrate how to write this template:
``` cpp
template<class T>
class X
{
    // ...
};
```
a) to require that T be derived from Clonable; and

b) to provide an alternative implementation if T is derived from Clonable, and work in some default mode otherwise.

4. Is the approach in #3 the best way to require/detect the availability of a `Clone()`? Describe alternatives.

5. Can a template benefit significantly from knowing that is parameter type T is inherited from some other type, in a way that could not be achieved at least as well otherwise without the inheritance relationship?

----

**Solution**
**1. What is a traits class?** 

Quoting 17.1.18 in the C++ standard, a traits class is:

"a class that encapsulates a set of types and functions necessary for template classes and template functions to manipulate objects of types for which they are instantiated."

The idea is that traits classes are templates used to carry extra information \-\- especially information that templates can use \-\- about the classes on which the traits template is instantiated. The nice thing is that the traits class `T<C>` tacks on said extra information to class C without requiring any change at all to C. Despite all the talk about "tacking on," traits are quite useful \-\- not "tacky" at all.

For examples, see:

- Items 2 and 3 in Exceptional C++.[1]

- GotW #62 on "Smart Pointer Members."[2]

- The April, May, and June 2000 issues of C++ Report, which contained several excellent columns about traits.

- The C++ standard library's own char_traits, iterator categories, and similar mechanisms.

**Requiring Member Functions** 
**2. Demonstrate how to detect and make use of class members within templates, using the following motivating case: You want to write a templated class C that can only be instantiated on types having a function named Clone that takes no parameters and returns a pointer to the same kind of object.**

``` cpp
// T must provide T* T::Clone() const
template<class T>
class C
{
    // ...
};
```
Note: It's obvious that if C just writes code that tries to invoke `T::Clone()` without parameters, then such code will fail to compile if there isn't a `T::Clone()` that can be called without parameters.

For an example to illustrate that last note, consider:
``` cpp
// Example 2(a): Initial attempt,
// sort of requires Clone()

// T must provide /*...*/ T::Clone( /*...*/ )
template<class T>
class C
{
public:
    void SomeFunc( T* t )
    {
        // ...
        // ...
        t->Clone();
        t->Clone();
        // ...
        // ...
    }
};
```
The first problem with Example 2(a) is that it doesn't necessarily require anything at all \-\- in a template, only the member functions that are actually used will be instantiated, or even parsed for that matter.[3] If `SomeFunc()` is never used, it will never be instantiated and so C can easily be instantiated with T's that don't have anything resembling `Clone()`.

The solution is to put the code that enforces the requirement into a function that's sure to be instantiated. The first thing most people think of is to put it in the constructor, because of course it's impossible to use C without invoking its constructor somewhere, right? True enough, but there could be multiple constructors and then to be safe we'd have to put the requirement-enforcing code into every constructor. There's a much easier solution, namely: Put it in the destructor. There's only one destructor, and it's equally impossible to use C without invoking its destructor, so that's the simplest place for the requirement-enforcing code to live.
``` cpp
// Example 2(b): Revised attempt, requires Clone()

// T must provide /*...*/ T::Clone( /*...*/ )
template<class T>
class C
{
public:
    ~C()
    {
        // ...
        T t; // kind of wasteful
        t.Clone();
        // ...
    }
};
```
This leaves us with the second problem: Both Examples 2(a) and 2(b) don't so much test the constraint as simply rely on it. (In the case of Example 2(b), it's even worse because 2(b) does it in a wasteful way that adds unnecessary runtime code just to try to enforce a constraint.) As noted in the question statement itself, continuing on:

But that's not enough to answer this question: Just trying to call `T::Clone()` without parameters would also succeed in calling a `Clone()` that has defaulted parameters and/or does not return a T*.

The code in Examples 2(a) and 2(b) will indeed work most swimmingly if there is a function "`T* T::Clone();`". The problem is that it will also work most swimmingly if there is a function "`void T::Clone();`", or "`T* T::Clone( int = 42 );`", or other variant signature, as long as it can be called without parameters. (For that matter, it will also work even if there isn't a `Clone()` member function at all, as long as there's a macro that changes the name Clone to something else, but there's little we can do about that.)

All that may be fine in some applications, but it's not what the question asked for. What we want to achieve is stronger:

The goal here is to specifically enforce that T provide a function that looks exactly like this: `T* T::Clone()`.

So here's one way we can do it:
``` cpp
// Example 2(c): Better, requires
// exactly T* T::Clone() const
//
// T must provide T* T::Clone() const
template<class T>
class C
{
public:
    // in C's destructor (easier than putting it
    // in every C ctor):
    ~C()
    {
        T* (T::*test)() const = T::Clone;
        test; // suppress warnings about unused variables
        // this unused variable is likely to be optimized
        // away entirely
        
        // ...
    }
    
    // ...
};
```
Or, a little more cleanly and extensibly:
``` cpp
// Example 2(d): Alternative way of requiring
// exactly T* T::Clone() const

// T must provide T* T::Clone() const
template<class T>
class C
{
    bool ValidateRequirements()
    {
        T* (T::*test)() const = T::Clone;
        test; // suppress warnings about unused variables
        // ...
        return true;
    }

public:
    // in C's destructor (easier than putting it
    // in every C ctor):
    ~C()
    {
        assert( ValidateRequirements() );
    }
    
    // ...
};
```
Having a ValidateRequirements() function is extensible \-\- it gives us a nice clean place to add any future requirements checks. Calling it within an `assert()` further ensures that all traces of the requirements machinery will disappear from release builds.

**Requiring Inheritance**
NOTE: I have seen the following ideas in several places \-\- I think that two of those places were in published or unpublished articles by Andrei Alexandrescu and Bill Gibbons. My apologies if the code I'm about to show looks really similar to someone's published code; I'm reusing other people's good ideas, but writing this code off the top of my head. (Later note: see Andrei's "Mappings Between Types and Values."[4])

**3. A programmer wants to write a template that can require (or just detect) whether the type on which it is instantiated has a `Clone()` member function. The programmer decides to do this by requiring that classes offering such a `Clone()` must derive from a fixed Clonable class. Demonstrate how to write this template:**
``` cpp
template<class T>
class X
{
    // ...
};
```
a) to require that T be derived from Clonable; and

First, we define a helper template that tests whether a candidate type D is derived from B. It determines this by determining whether a pointer to D can be converted to a pointer to B:
``` cpp
// Example 3(a): An IsDerivedFrom helper
//
template<class D, class B>
class IsDerivedFrom
{
private:
    class Yes { char a[1]; };
    class No { char a[10]; };
    
    static Yes Test( B* ); // undefined
    static No Test( ... ); // undefined

public:
    enum { Is = sizeof(Test(static_cast<D*>(0))) == sizeof(Yes) ? 1 : 0 };
};
```
Get it? Think about this code for a moment before reading on.

The above trick relies on three things:

- Yes and No have different sizes. I may just be being paranoid, but the reason I don't use just char[1] and char[2] is the off chance that the sizes of Yes and No might be the same, for example if the compiler happened to require that an object's size be a multiple of four bytes. I doubt it would ever happen, but I can't see any wording in the standard that would prohibit it.

- Overload resolution and determining the value of sizeof are both performed at compile time, not runtime.

- Enums are initialized, and values can be used, at compile time.

Let's analyze the enum definition in a little more detail. First, the innermost part is:
``` cpp
Test(static_cast<D*>(0))
```
All this does is mention a function named Test and pretend to pass it a `D*` \-\- in this case, a suitably cast null pointer will do. Note that nothing is actually being done here, and no code is being generated, so the pointer is never dereferenced or for that matter even ever actually created. All we're doing is creating a typed expression. Now, the compiler knows what D is, and will apply overload resolution at compile time to decide which of the two overloads of Test() ought to be chosen: If a `D*` can be converted to a `B*`, then `Test( B* `), which returns a Yes, would get selected; otherwise, `Test( ... )`, which returns a No, would get selected.

The next step is to check which overload would get selected:
``` cpp
sizeof(Test(static_cast<D*>(0))) == sizeof(Yes) ? 1 : 0
```
This expression, still evaluated entirely at compile time, will yield 1 if a `D*` can be converted to a `B*`, and 0 otherwise. And that's pretty much all we want to know, because a `D*` can be converted to a `B*` if and only if D is derived from B (or D is the same as B, but we'll plug that hole presently).

So, now that we've calculated what we need to know, we just need to store the result someplace. The said "someplace" has to be a place that can be set and the value used all at compile time. Fortunately, an enum fits the bill nicely:
``` cpp
enum { Is = sizeof(Test(static_cast<D*>(0))) == sizeof(Yes) ? 1 : 0 };
```
Finally, there is still that potential hole when D is the same as B, and depending on the way we plan to use IsDerivedFrom we may not want this template to report that a class is derived from itself (a lie, if perhaps a benign one in some applications). If we do need to plug the hole, we can do it easily by partially specializing the template to say that a class is not derived from itself:
``` cpp
// Example 3(a), continued
//
template<class T>
class IsDerivedFrom<T,T>
{
public:
    enum { Is = 0 };
};
```
That's it. We can now use this facility to help build an answer to the question, to wit:
``` cpp
// Example 3(a), continued: Using IsDerivedFrom
// to enforce derivation from Clonable
//
template<class T>
class X
{
    bool ValidateRequirements() const
    {
        // typedef needed because of the ,
        typedef IsDerivedFrom<T, Clonable> Y; 
        
        // a runtime check, but one that can be turned
        // into a compile-time check without much work
        assert( Y::Is );
        
        return true;
    }

public:
    // in X's destructor (easier than putting it
    // in every X ctor):
    ~X()
    {
        assert( ValidateRequirements() );
    }
    
    // ...
};
```
**Selecting Alternative Implementations**
Well, the solution in Example 3(a) is nice and all, and it'll make sure T must be a Clonable, but what if T isn't a Clonable? What if there were some alternative action we could take? Perhaps we could make things even more flexible \-\- which brings us to the second part of the question.

b) to provide an alternative implementation if T is derived from Clonable, and work in some default mode otherwise.

To do this, we introduce the proverbial extra level of indirection, in this case a helper template. In short, X will use IsDerivedFrom, and use partial specialization of the helper to switch between is-Clonable and isn't-Clonable implementations:
``` cpp
// Example 3(b): Using IsDerivedFrom to make use of
// derivation from Clonable if available, and do
// something else otherwise.
//
template<class T, int>
class XImpl
{
    // general case: T is not derived from Clonable
};

template<class T>
class XImpl<T, 1>
{
    // T is derived from Clonable
};

template<class T>
class X
{
    XImpl<T, IsDerivedFrom<T, Clonable>::Is> impl_;
    // ... delegates to impl_ ...
};
```
Do you see how this works? Let's work through it with a quick example:
``` cpp
class MyClonable : public Clonable { /*...*/ };

X<MyClonable> x1;
```
`X<T>`'s impl_ has type `XImpl<T, IsDerivedFrom<T, Clonable>::Is>`. In this case, `T` is MyClonable, and so `X<MyClonable>`'s impl_ has type `XImpl<MyClonable, IsDerivedFrom<MyClonable, Clonable>::Is>`, which evaluates to `XImpl<MyClonable, 1>`, which uses the specialization of `XImpl` that makes use of the fact that `MyClonable` is derived from Clonable. But what if we instantiate `X` with some other type? Consider:
``` cpp
X<int> x2;
```
Now `T` is int, and so `X<int>`'s `impl_` has type `XImpl<MyClonable, IsDerivedFrom<int, Clonable>::Is>`, which evaluates to `XImpl<MyClonable, 0>`, which uses the unspecialized `XImpl`. Nifty, isn't it?

Note that at most `XImpl<T,0>` and `XImpl<T,1>` will ever be instantiated for any given T. Even though XImpl's second parameter could theoretically take any integer value, the way we've set things up here the integer can only ever be 0 or 1. (In that case, why not use a bool instead of an int? Extensibility: It doesn't hurt to use an int, and doing so allows additional alternative implementations to be added easily in the future.)

**Requirements vs. Traits**
**4. Is the approach in #3 the best way to require/detect the availability of a `Clone()`? Describe alternatives.**

The approach in #3 is nifty, but I tend to like traits better in many cases \-\- they're about as simple (except when they have to be specialized for every class in a hierarchy), and they're more extensible as shown in GotW #62.

The idea is to create a traits template whose sole purpose in life, in this case, is to implement a Clone() operation. The traits template looks a lot like XImpl, in that there'll be a general-purpose unspecialized version that does something general-purpose, and possibly multiple specialized versions that deal with classes that provide better or just different ways of cloning.
``` cpp
// Example 4: Using traits instead of IsDerivedFrom
// to make use of Clonability if available, and do
// something else otherwise. Requires writing a
// specialization for each Clonable class.
//
template<class T>
class XTraits
{
    // general case: use copy constructor
    static T* Clone( const T* p )
    { return new T( *p ); }
};

template<>
class XTraits<MyClonable>
{
    // MyClonable is derived from Clonable, so use Clone()
    static MyClonable* Clone( const MyClonable* p )
    { return p->Clone(); }
};

// ... etc. for every class derived from Clonable
```
`X<T>` then simply calls `XTraits<T>::Clone()` where appropriate, and it will do the right thing.

The main difference between traits and the plain old XImpl shown in Example 3(b) is that, with traits, when the user defines some new type the most work they have to do to use it with X is to specialize the traits template to "do the right thing" for the new type. That's more extensible than the relatively hard-wired approach in #3 above, which does all the selection inside the implementation of XImpl instead of opening it up for extensibility. It also allows for other cloning methods besides a function specifically named `Clone()` inherited from a specifically named base class, and this too provides extra flexibility.

For more details, including a longer sample implementation of traits for a very similar example, see GotW #62, Examples 3(c)(i) and 3(c)(ii).

**Hierarchy-Wide Traits**
The main drawback of the traits approach above is that it requires individual specializations for every class in a hierarchy. There are ways to provide traits for a whole hierarchy of classes at a time, instead of tediously writing lots of specializations. See Andrei Alexandrescu's excellent column in the June 2000 C++ Report, where he describes a nifty technique to do just this.[5] Andrei's technique requires minor surgery on the base class of the outside class hierarchy, in this case Clonable. It would be nice if we could specialize XTraits for the whole Clonable hierarchy in one shot without requiring any change to Clonable \-\- this is a topic for a potential future issue of GotW.

**Inheritance vs. Traits**
**5. Can a template benefit significantly from knowing that is parameter type T is inherited from some other type, in a way that could not be achieved at least as well otherwise without the inheritance relationship?**

As far as I can tell at this writing, there is little extra benefit a template can gain from knowing that one of its template parameters is derived from some given base class that the template couldn't gain more extensibly via traits. The only real drawback to using traits is that it can require writing lots of traits specializations to handle many classes in a big hierarchy, but there are techniques that mitigate or eliminate this drawback.

A principal motivator for this GotW issue was to demonstrate that "using inheritance for categorization in templates" is perhaps not as necessary a reason to use inheritance as some have thought. Traits provide a more general mechanism that's much more extensible when it comes time to instantiate an existing template on new types, such as types that come from a third-party library, that may not be easy to derive from a foreordained base class.

**Notes**
1. H. Sutter. [Exceptional C++](http://www.gotw.ca/publications/xc++.htm) (Addison-Wesley, 2000).
2. Available at [http://www.gotw.ca/gotw/062.htm](http://www.gotw.ca/gotw/062.htm).
3. I'm not sure that all compilers get this rule right yet. Yours may well instantiate all functions, not just the ones that are used.
4. A. Alexandrescu. ["Mappings Between Types and Values"](http://www.cuj.com/experts/1810/alexandr.htm) (C/C++ Users Journal C++ Experts Forum, October 2000).
5. A. Alexandrescu. "Traits on Steroids" (C++ Report 12(6), June 2000).



# 072 Data Formats and Efficiency 
**Difficulty: 8 / 10**
How good are you at choosing highly compact and memory-efficient data formats? How good are you at writing bit-twiddling code? This GotW gives you ample opportunity to exercise both skills, as we consider efficient representations of chess games and a BitBuffer to hold them.

----

Background: I assume you know the basics of chess.

**Problem**
**JG Question**
**1.** Which standard container uses the least memory to store the same number of objects of the same type T: deque, list, set, or vector? Explain.

**Guru Questions**
**2.** You are creating a worldwide chess server that stores all games ever played on it. Because the database can get very large, you want to represent each game using as few bytes as possible. For this problem, consider only the actual game moves and ignore extra information such as the players' names and comments.

For each of the following data sizes, demonstrate a format for representing a chess game that requires the indicated amount of data to store each half-move (a half-move is one move played by one side). For this question, assume 1 byte == 8 bits.
   a) 128 bytes per half-move
   b) 32 bytes per half-move
   c) minimum 4 bytes and maximum 8 bytes per half-move
   d) minimum 2 bytes and maximum 4 bytes per half-move
   e) 12 bits per half-move

**3.** To implement solution 2(e), you decide to create the following class that manages a buffer of bits. Implement it portably, so that it will work correctly on all conforming C++ compilers regardless of platform.
``` cpp
class BitBuffer
{
public:
    // ... add other functions as needed ...
    
    // Append num bits starting with the first bit of p.
    //
    void Append( unsigned char* p, size_t num );
    
    // Query #bits in use (initially zero).
    //
    size_t Size() const;
    
    // Get num bits starting with the start-th bit,
    // and store the result starting with the first
    // bit of p.
    //
    void Get( size_t start, size_t num, unsigned char* dest ) const;

private:
    // ... add details here ...
};
```
**4.** Is it possible to store a chess game using fewer than 12 bits per half-move? If so, demonstrate how. If not, why not?

----

**Solution**
**1. Which standard container uses the least memory to store the same number of objects of the same type T: deque, list, set, or vector? Explain.**

Each kind of container chooses a different space/performance tradeoff:
   a) A `vector<T>` internally stores a contiguous array of `T` objects, and so has no per-element overhead at all.

   b) A `deque<T>` can be thought of as a `vector<T>` whose internal storage is broken up into chunks. A `deque<T>` stores chunks, or "pages," of objects. This requires storing one extra pointer of management information per page, which usually works out to a fraction of a bit per element. There's no other per-element overhead because `deque<T>` doesn't store any extra pointers or other information for individual `T` objects.

   c) A `list<T>` is a doubly-linked list of nodes that hold `T` elements. This means that for each `T` element, `list<T>` also stores two pointers, which point to the previous and next nodes in the list. Every time we insert a new `T` element, we also create two more pointers, so a `list<T>` requires two pointers' worth of overhead per element.

   d) Finally, a `set<T>` (and, for that matter, a `multiset<T>`, `map<Key,T>`, or `multimap<Key,T>`) also stores nodes that hold `T` (or `pair<const Key,T>`) elements. The usual implementation of a set is as a tree with three extra pointers per node. Often people see this and think, 'why three pointers? isn't two enough, one for the left child and one for the right child?' Because it must be possible to efficiently iterate over the set, we also need an "up" pointer to the parent node, otherwise determining the '`next`' element starting from some arbitrary iterator can't be done efficiently. (Besides trees, other internal implementations of set are possible; for example, an alternating skip list can be used, although this still requires at least three pointers per element in the set.[1])

Part of choosing an efficient in-memory representation of data is choosing the right (read: most space- and time-efficient) container that supports the functionality you need. But that's not the end of it by any means: Another big part of choosing an efficient in-memory representation of data is determining how to represent the data that will be put into those containers. This question brings us to the meat of this issue of GotW.

**Different Ways To Represent Data**
The point of Question 2 is to demonstrate that there can be a plethora of ways to represent the same information:

**2. You are creating a worldwide chess server that stores all games ever played on it. Because the database can get very large, you want to represent each game using as few bytes as possible. For this problem, consider only the actual game moves and ignore extra information such as the players' names and comments.**

The rest of this article uses the following terms and abbreviations:
|terms|abbreviations|
|----|----|
|K|	King  |
|Q|	Queen |
|R|	Rook  |
|B|	Bishop|
|N|	Knight|
|P|	Pawn  |
|rank|	row on the chessboard, typically numbered 1 (White's home row)|
|to 8| (Black's home row)|
|file|	column on the chessboard, typically numbered a (left) to h (right)|

For each of the following data sizes, demonstrate a format for representing a chess game that requires the indicated amount of data to store each half-move (a half-move is one move played by one side). For this question, assume 1 byte == 8 bits.

   a) 128 bytes per half-move

One representation that would take this amount of space would be to assume that the program already knows the current board position (which it can deduce from the previous moves) and store the entire new board position using two bytes per square. In this case, we are mimicking one of the standard online notations, which uses a 'W' or 'B' or '.' to designate the side that owns the piece in the given square, and a 'K', 'Q', 'R', 'B', 'N', 'P', or '.' to designate the type of piece in the given square.

Using this scheme, and storing the board from rank 1 to rank 8, and file a to file h within each rank, one possible half-move representation might be:
```
  WRWNWBWQWKWBWNWRWPWPWP..WPWPWPWP......WP........................\
  ................................BPBPBPBPBPBPBPBPBRBNBBBQBKBBBNBR
```
If this is the first move, it represents "1. d4" (or, "1. P-Q4"), my usual opening move.

   b) 32 bytes per half-move

The representation in (a) seems a little wasteful, since it's in ASCII text that humans can read, but we really only need a machine-readable format. After all, the software running in front of the chess database is going to take care of displaying positions to the user.

We can get down to 32 bytes per half-move by keeping the basic strategy above of storing the entire new board position, but this time storing only 4 bits for each square: we need 3 bits to store the piece type (e.g., 0 for K, 1 for Q, 2 for R, 3 for B, 4 for N, 5 for P, or 6 for none), and 1 bit to store the color (which can be ignored if there is no piece on the square).

Using this scheme, and storing the board from rank 1 to rank 8, and file a to file h within each rank, one possible half-move representation might consume this many bytes:
```
  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

   c) minimum 4 bytes and maximum 8 bytes per half-move

We can achieve this by storing the half-move as text in old-style chess notation.

Old-style "descriptive" chess notation identifies squares using variable-length tags like K3 and QN8, instead of using two-character tags like e3 and b8. To write down a half-move this way requires at least 4 characters (e.g., P-Q4) and possibly as much as 8 characters (e.g., RKN1-KB1, P-KB8(Q)). Note that no extra trailing null or other delimiter is needed because the move format can be decoded unambiguously.

Using this scheme, one possible half-move representation might be:
```
  P-KB8(Q)
```

   d) minimum 2 bytes and maximum 4 bytes per half-move

We can achieve this by storing the half-move as text in modern chess notation.

Modern "algebraic" chess notation is more compact, and any half-move can be written using at least 2 characters (e.g., d4) and at most 4 characters (e.g., Rgf1, gh=Q). Again, no special move delimiter is needed because the format can be decoded unambiguously.

Using this scheme, one possible half-move representation might be:
```
  gh=Q
```
(Incidentally, a major advantage of this representation outside the computing world is that it can be written down quickly on paper by a human, even under time pressure. The reduction from a maximum of 8 characters to a maximum of 4 characters, coupled with some improved conceptual simplicity, turns out to make a big difference to users \-\- also known as players.)


   e) 12 bits per half-move

We can get more compact still by taking a different approach: What if we were to store just the moving piece's origin and destination squares? To encode one square location requires 6 bits because there are 64 possibilities; so to encode two square locations to allow for both the origin and the destination to be recorded requires 12 bits. That suffices for usual moves, but in the case of a pawn promotion, however, this scheme will need more than 12 bits.

That's already a lot better than the earlier attempts. Let's stop here for a moment, though, and consider how we might create auxiliary data structures to store such bits of information that won't play nice and fall on even byte boundaries.

**BitBuffer, the Binary Slayer**
**3. To implement solution 2(e), you decide to create the following class that manages a buffer of bits. Implement it portably, so that it will work correctly on all conforming C++ compilers regardless of platform.**

First, note that the directive "assume that 1 byte == 8 bits" applied only to Question #2 \-\- it does not apply here. We need a solution that will compile and run correctly on any conforming C++ implementation, no matter what kind of underlying platform it's running on.

The required interface boiled down to:
``` cpp
class BitBuffer
{
public:
  void Append( unsigned char* p, size_t num );
  size_t Size() const;
  void Get( size_t start, size_t num, unsigned char* dest ) const;
  // ...
};
```
You might wonder why the BitBuffer interface was specified in terms of pointers to unsigned char. First off, there's no such thing as a pointer to a bit in standard C++, so that's out. Second, the C++ standard guarantees that operations on unsigned types (including unsigned char, unsigned short, unsigned int, and unsigned long) won't run afoul of "hey, you didn't initialize that byte!" or "hey, that's not a valid value!" messages from your compiler. As Bjarne Stroustrup writes:[2]

   > The unsigned integer types are ideal for uses that treat storage as a bit array.

So compilers are required to treat unsigned char (and the other unsigned integer types) as raw bits of storage \-\- which is just what we want. There are other approaches, but this is a reasonable one that lets us exercise our bit-fiddling coding skills, which happens to be a major goal of this GotW.

The main question in implementing BitBuffer is: What internal representation should we use? I'll consider two major alternatives.

**Attempt #1: Bit-Fiddling Into an unsigned char Buffer**
The first idea is to implement the BitBuffer via an internal big block of unsigned chars, and fiddle the bits ourselves when we put them in and take them out. We could let BitBuffer have a member of type unsigned char* that points to the buffer, but let's at least use a vector<unsigned char> so that we don't have to worry as much about basic memory management.

Do you think the above sounds pretty easy? If you do, and you haven't yet tried to implement (and test!) it, take an hour or three and try it now. I bet you'll find it's not as simple as one might think.

I'm not entirely ashamed to report that this version took me quite a bit of effort to write. Just drafting the initial version of the code took me more programming effort than I expected, and then it took a lot of debugging effort to find and fix all the bugs. I didn't keep track of the development effort, but in retrospect I estimate it took me several score compiles, including several to add reporting cout's to analyze intermediate values and see where things were going wrong, plus half a dozen sessions in the debugger stepping through code, to determine and fix all the problems.

Here's the result. I don't claim it's perfect, but it passed the unit tests I threw at it, including single- and multi-byte appends and boundary cases. (You always write unit test harnesses for your code, right? And make sure your code passes them all, before you check the code in?) Note that this version of the code operates on chunks of bytes at a time \-\- for example, if we're using 8-bit bytes and have an offset of 3 bits, then we'll copy the first three bits as a unit and copy the last 5 bits as a unit, for two operations per byte. For simplicity, I also require the user to provide buffers that are a byte bigger than might otherwise be necessary, just so that I can simplify my own code by allowing a little running off the end.
``` cpp
// Example 3(a): BitBuffer implemented in terms of
// vector<unsigned char>. Hard work. Ugh.
//
class BitBuffer
{
public:
    BitBuffer() : buf_(0), size_(0) { }
    
    // Append num bits starting with the first bit of p.
    //
    void Append( unsigned char* p, size_t num )
    {
        int bits = numeric_limits<unsigned char>::digits;
        
        // first destination byte & bit offset
        int dst = size_ / bits;
        int off = size_ % bits;
        
        while( buf_.size() < (size_+num) / bits + 1 )
            buf_.push_back( 0 );
        
        for( int i = 0; i < (num+bits-1)/bits; ++i )
        {
            unsigned char mask = FirstBits(num - bits*i);
            buf_[dst+i] |= (*(p+i) & mask) >> off;
            if( off > 0 )
                buf_[dst+i+1] = (*(p+i) & mask) << (bits - off);
        }
        
        size_ += num;
    }
    
    // Query #bits in use (initially zero).
    //
    size_t Size() const
    {
        return size_;
    }
    
    // Get num bits starting with the start-th bit
    // (0-based), and store the result starting with
    // the first bit of dst. The buffer pointed at
    // by dst should be at least one byte bigger
    // than the minimum needed to hold num bits.
    //
    void Get( size_t start, size_t num, unsigned char* dst ) const
    {
        int bits = numeric_limits<unsigned char>::digits;
        
        // first source byte & bit offset
        int src = start / bits;
        int off = start % bits;
        
        for( int i = 0; i < (num+bits-1)/bits; ++i )
        {
            *(dst+i) = buf_[src+i] << off;
            if( off > 0 )
                *(dst+i) |= buf_[src+i+1] >> (bits - off);
        }
    }

private:
    vector<unsigned char> buf_;
    size_t size_; // in bits
    
    // Build a mask where the first n bits are 1 and
    // the rest are 0.
    //
    unsigned char FirstBits( size_t n )
    {
        int num = min( n, numeric_limits<unsigned char>::digits );
        unsigned char b = 0;
        while( num-- > 0 )
          b = (b >> 1) | (1 << (numeric_limits<unsigned char>::digits-1));
        return b;
    }
};
```
The above code is nontrivial. Take some time to read it and to convince yourself that it's doing the right thing. (If you think you've found a bug, first write a test harness that attempts to demonstrate the bug; once the bug has been confirmed, please do go ahead and send me both the bug report and the test harness that tickles the problem behavior.)

**Attempt #2: Reusing a Standard Bit-Packed Containers**
The second idea is to note that the standard library already includes two containers that store bits: bitset and vector<bool>. Now, bitset is a bad choice simply because bitset<N> has fixed length N and we'll be encoding variable-length bitstreams. No dice. Here vector<bool>, for all its other faults, is a tempting choice and in this case turns out to be just what the doctor ordered.

The most important thing I can say about the following code is this:

The Example 3(b) code was essentially correct on first writing.

Yes, really. Between my first compile and the final code, all I did was fix a few syntax typos like a missing semicolon (these are, after all, things the compiler is supposed to find for you) and add parentheses in two places where I'd forgotten that % has higher precedence than +. That's it.
``` cpp
// Example 3(b): BitBuffer implemented in terms of
// vector<bool>.
//
class BitBuffer
{
public:
  // Append num bits starting with the first bit of p.
  //
  void Append( unsigned char* p, size_t num )
  {
    int bits = numeric_limits<unsigned char>::digits;

    for( int i = 0; i < num; ++i )
    {
      buf_.push_back( *p & (1 << (bits-1 - i%bits)) );
      if( (i+1) % bits == 0 )
        ++p;
    }
  }

  // Query #bits in use (initially zero).
  //
  size_t Size() const
  {
    return buf_.size();
  }

  // Get num bits starting with the start-th bit
  // (0-based), and store the result starting
  // with the first bit of dst.
  //
  void Get( size_t start, size_t num, unsigned char* dst ) const
  {
    int bits = numeric_limits<unsigned char>::digits;

    *dst = 0;
    for( int i = 0; i < num; ++i )
    {
      *dst |= unsigned char(buf_[start+i]) << (bits-1 - i%bits);
      if( (i+1) % bits == 0 )
        *++dst = 0;
    }
  }

private:
  vector<bool> buf_;
};
```
That writing this version was so much easier than writing Example 2(a) shouldn't be surprising. This version reuses existing bit-fiddling code instead of writing its own, it uses about 50% fewer lines of code \-\- and it's disproportionately less buggy as a result! This time I didn't even have to ask the user to supply a bit of extra output space just to make my Get() code simpler, as I did in the first version.

I suspect that both solutions, especially the first, could probably be further improved \-\- there may be bugs I didn't detect, and maybe the code could be simplified a bit in ways I didn't see \-\- but I think they're pretty close to optimal in terms of both correctness and style.

**The Big Squeeze**
Let's take one final look at the compressed representation and see if there's anything more we can do to squeeze it down.

**4. Is it possible to store a chess game using fewer than 12 bits per half-move? If so, demonstrate how. If not, why not?**

Yes. For example, here are three ways:

We can get down to 10 bits per half-move by encoding the destination square and the piece that moved there. Encoding a square requires 6 bits, as above. Encoding which piece moved there can be done by simply identifying the number of the piece, assigning an arbitrary ordering to the squares, say from rank 1 to rank 8 and file a to file h within each rank, and numbering the pieces in the order their current squares appear in that ordering. There can only be 16 pieces on the board, so the piece can be identified using 4 bits, for a total of 10 bits.

Could we do better still? Let's reason it through: We can encode all possible squares as destination, but usually only a minority of squares could actually be moved to with a legal move, so there must be some redundancy left in that part of our encoding. Similarly, we can represent all pieces moving to the given square, even though almost certainly not all pieces could move to that square \-\- indeed, some pieces may not even have a legal move at all to any square --, so somehow we're probably encoding more than we need to encode. For example, we could compress the second part further by storing from 0 to 4 bits to identify the piece that moved: there are 1 to 16 possibilities, and if there is only one piece that could move to the square then we don't need to encode any bits at all. On decoding, we know how many possible pieces could have moved to the square, and so we know how many bits to pull from the input format for that half-move.

We can get down to an encoding that uses at least 0 and at most 8 bits per half-move as follows: First, invent an ordering of legal moves; for example, we could order the pieces according to their squares as above, and for each piece order its possible moves as possible destination squares according to the same square ordering. Then, store the number of the actual move made using the minimum number of bits required; for example, the opening position has 20 legal moves, and to store them as a plain binary number requires ceiling(log2(20)) = 5 bits.

The result is that we need a minimum of 0 bits to represent each half-move. Zero bits are needed if there is only one possibility, a forced move. But how many bits could be needed in the worst case? This corresponds directly to the question: How many moves could there be in a legal chess position? At [http://www.drb.insel.de/~heiner/Chess/README_LONG](http://www.drb.insel.de/~heiner/Chess/README_LONG), the author writes: "The current known maximum is 218 (Dickins 1968): 3Q4/1Q4Q1/4Q3/2Q4R/ Q4Q2/3Q4/1Q4Rp/1K1BBNNk w - - 0 1." The board looks like this:[3]
```
  - - - Q - - - -   3Q4
  - Q - - - - Q -   1Q4Q1
  - - - - Q - - -   4Q3
  - - Q - - - - R   2Q4R
    Q - - - - Q - -   Q4Q2
  - - - Q - - - -   3Q4
  - Q - - - - R p   1Q4Rp
  - K - B B N N k   1K1BBNNk
```
In this worst case, 8 bits are needed to encode the move as a plain binary number. On average, probably 5 bits will be required to store a typical move; the opening position has 20 moves, and a typical endgame with the side to move having, say, K+R+2P on an open board can yield about 30 legal moves if the Pawns are getting ready to promote, both of which cases require 5 bits to store using this method.

Thinking about this briefly should convince us that this encoding ought to be pretty close to optimal, because it is representing directly and exactly the answer to the question at hand: "Which legal move was made?" We are using the minimum number bits to represent the possibilities for any given move as a plain binary number, with full knowledge of what has gone before.

Can we do better still? Yes, although now we'll start to see diminishing returns as the further gains become incremental. This is because further gains rely on having more knowledge and/or saving only partial bits. To illustrate how we could do a bit better still, consider that the opening position has 20 moves, which we under the above scheme we would store using ceiling(log2(20)) = 5 bits \-\- yet really that choice of first move theoretically holds only log2(20) = 4.3 bits of actual information, even assuming that all moves are equally likely, and on average we should require even fewer bits because the two most popular opening moves for White account for the majority of all chess games. In brief, if we can additionally gain knowledge about the relative probabilities of each move (for example, by building into the compression engine a deterministic chess-playing program that can guess which moves are better or more likely than others for any given position), then we could use variable-length encodings such as Huffman and arithmetic compression that use fewer bits to store the more likely moves. This trades off computing time using domain-specific knowledge in return for better compression.

**Conclusion**
This GotW shows how domain-specific knowledge can be applied to make a significant difference in the solution of a given problem.

In summary, even without any knowledge of which moves are more likely in any given position, a typical 40-move (80-half-move) game can be stored in about 50 bytes. To get a sense of how tight a representation that really is, consider:

This concluding sentence is exactly 50 bytes long.

 

**Notes**
1. L. Marrie. "Alternating Skip Lists" (Dr. Dobb's Journal, 25(8), August 2000).

2. B. Stroustrup. The C++ Programming Language, Special Edition (Addison-Wesley, 2000)

3. I believe it should be straightforward to show that this is a legally reachable position: Black moved last, and the only legal move could have been to move the P on h3 to h2 (the P on h2 it could not have come from g3 because there's no White piece it could have captured). Note that, sometime in the past, all white Ps Queened, and the easiest way to let them do that with minimal captures is to have every 2nd black P be captured by a white P, where each captured black P allows two white Ps (the capturing P and the P on the capturee's file) to advance to the 8th rank with no further captures. Therefore White's move before that could have been to capture any of Black's 10 other pieces on any of the 16 squares White now controls, etc. If we just say that White's last 10 moves were each to capture a Black piece that had itself just moved to the target square, and we've accounted for the last 21 half-moves leading to the position shown.



# 073 Style Case Study #1: Index Tables 
**Difficulty: 5 / 10**
This GotW introduces a new theme that we'll see again from time to time in future Style Case Study issues: We examine a piece of published code, critique it to illustrate proper coding style, and develop an improved version. You may be amazed at just how much can be done even with code that has been written, vetted, and proofread by experts.

----

**Problem**
**JG Question**
1. Who benefits from clear, understandable code?

Guru Question
2. The following code presents an interesting and genuinely useful idiom for creating index tables into existing containers. For a more detailed explanation, see the original article.[1]

Critique this code and identify:
   a) Mechanical errors, such as invalid syntax or nonportable conventions.
   b) Stylistic improvements that would improve code clarity, reusability, and maintainability.

``` cpp
// program sort_idxtbl(...) to make a permuted array of indices
#include <vector>
#include <algorith>

template <class RAIter>
struct sort_idxtbl_pair
{
    RAIter it;
    int i;
    
    bool operator<(const sort_idxtbl_pair& s)
    {   
        return (*it) < (*(s.it)); 
    }
    
    void set(const RAIter& _it, int _i) { it=_it; i=_i; }
    sort_idxtbl_pair() {}
};

template <class RAIter>
void sort_idxtbl(RAIter first, RAIter last, int* pidxtbl)
{
    int iDst = last-first;
    typedef std::vector< sort_idxtbl_pair<RAIter> > V;
    V v( iDst );
    
    int i=0;
    RAIter it = first;
    V::iterator vit = v.begin();
    for( i=0; it<last; it++, vit++, i++ )
        (*vit).set(it,i);
    
    std::sort(v.begin(), v.end());
    
    int *pi = pidxtbl;
    vit = v.begin();
    for( ; vit<v.end(); pi++, vit++ )
        *pi = (*vit).i;
}

main()
{
    int ai[10] = { 15,12,13,14,18,11,10,17,16,19 };
    
    cout << "#################" << endl;
    std::vector<int> vecai(ai, ai+10);
    int aidxtbl[10];
    sort_idxtbl(vecai.begin(), vecai.end(), aidxtbl);
    
    for (int i=0; i<10; i++)
    cout << "i=" << i
         << ", aidxtbl[i]=" << aidxtbl[i]
         << ", ai[aidxtbl[i]]=" << ai[aidxtbl[i]]
         << endl;
    cout << "#################" << endl;
}
```

----

**Solution**
**1. Who benefits from clear, understandable code?**

In short, just about everyone benefits.

First, clear code is easier to follow while debugging, and for that matter is less likely to have as many bugs in the first place, so writing clean code makes your own life easier even in the very short term. (For a case in point, see the GotW #72 solution for Question 3.[2]) Further, when you return to the code a month or a year later \-\- as you surely will if the code is any good and is actually being used \-\- then it's much easier to pick it up again and understand what's going on. Most programmers find it difficult to keep full details of code in their heads for even a few weeks, especially after having moved on to other work; after a few months, or even a few years, it's too easy to go back to your own code and imagine it was written by a stranger \-\- albeit a stranger who curiously happened to follow your personal coding style.

But enough about selfishness. Let's turn to altruism: Those who have to maintain your code also benefit from clarity and readability. After all, to maintain code well one must first grok the code. "To grok," as coined by Robert Heinlein, means to comprehend deeply and fully; in this case, that includes understanding the internal workings of the code itself, as well as its side effects and interactions with other subsystems. It is altogether too easy to introduce new errors when changing code one does not fully understand. Code that is clear and understandable is easier to grok, and therefore fixes to such code become less fragile, less risky, less likely to have unintended side effects.

Most importantly, however, your end users benefit from clear and understandable code for all of the above reasons: Such code is likely to have had fewer initial bugs in the first place, and it's likely to have been be maintained more correctly without as many new bugs being introduced.

**2. The following code presents an interesting and genuinely useful idiom for creating index tables into existing containers. For a more detailed explanation, see the original article.[1]**

Critique this code and identify:

Again, let me repeat that which bears repeating: This code presents an interesting and genuinely useful idiom. I've frequently found it necessary to access the same container in different ways, such as using different sort orders. For this reason it can be useful indeed to have one principal container that holds the data (for example, a vector<Employee>) and secondary containers of iterators into the main container that support variant access methods (for example, a set<vector<Employee>::iterator, Funct> where Funct is a functor that compares Employee objects indirectly yielding a different ordering than the order in which the objects are physically stored in the vector).

Having said that, style matters too. The original author has kindly allowed me to use his code as a case in point, and I'm not trying to pick on him here; I'm just adopting the technique, pioneered long ago by luminaries like P.J. Plauger, of expounding coding style guidelines via the dissection and critique of published code. I've critiqued other published material before, have had other people critique my own, and I'm positive that further dissections will no doubt follow.

Having said all that, let's see what we might be able to improve in this particular piece of code:

**Correcting Mechanical Errors**
   a) Mechanical errors, such as invalid syntax or nonportable conventions.

The first area for constructive criticism is mechanical errors in the code, which on most platforms won't compile as shown.
``` cpp
#include <algorith>
```
1. Spell standard headers correctly: Here the header `<algorithm>` is misspelled as `<algorith>`. My first guess was that this is probably an artifact of an 8-character file system used to test the original code, but even my old version of MSVC on an old version of Windows (based on the 8.3 filename system) rejected this code. Anyway, it's not standard, and even on hobbled file systems the compiler itself is required to support any standard long header names even if it silently maps it onto a shorter filename (or onto no file at all).

``` cpp
main()
```
2. Define `main()` correctly: The signature "`main()`" has never been valid C++. That used to be valid in pre-1999 C, which had an implicit int rule, but it's nonstandard in both C++ (which never had implicit int) and C99 (which as far as I can tell didn't merely deprecate implicit int, but actually removed it outright). In the C++ standard, see:
- 3.6.1/2: portable code must define main() as either "`int main()`" or "`int main( int, char*[] )`"
- 7/7 footnote 78, and 7.1.5/2 footnote 80: implicit int banned
- Annex C (Compatibility), comment on 7.1.5/4: explicitly notes that "`main()`" is invalid C++, and must be written "`int main()`"

``` cpp
    cout << "#################" << endl;
```
3. Always #include the headers for the types whose definitions you need: The program uses cout and endl, but fails to #include <iostream>. Why did this probably work on the original developer's system? Because C++ standard headers can #include each other, but unlike C, C++ does not specify which standard headers #include which other standard headers.

In this case, the program does `#include <vector>` and `<algorithm>`, and on the original system it probably just so happened that one of those headers happened to indirectly `#include <iostream>` too. That may work on the library implementation used to develop the original code, and it happens to work on mine too, but it's not portable and not good style.

4. Follow the guidelines in my Migrating to Namespaces article[3] about using namespaces: Speaking of cout and endl, the program must also qualify them with `std::`, or write "`using std::cout; using std::endl;`". Unfortunately it's still common for authors to forget namespace scope qualifiers \-\- I hasten to point out that this author did correctly scope vector and sort.

**Improving Style**
   b) Stylistic improvements that would improve code clarity, reusability, and maintainability.

Beyond the above mechanical errors, there were several things I personally would have done differently in the code example. First, a couple of comments about the helper struct:
``` cpp
template <class RAIter>
struct sort_idxtbl_pair
{
  RAIter it;
  int i;
  bool operator<( const sort_idxtbl_pair& s )
    { return (*it) < (*(s.it)); }
  void set( const RAIter& _it, int _i ) { it=_it; i=_i; }
  sort_idxtbl_pair() {}
};
```
5. Be const-correct: In particular, `sort_idxtbl_pair::operator<()` doesn't modify `*this`, so it ought to be declared as a const member function.

6. Remove redundant code: The program explicitly writes class `sort_idxtbl_pair`'s default constructor even though it's no different from the implicitly generated version. There doesn't seem to be much point to this. Also, as long as `sort_idxbl_pair` is a struct with public data, having a distinct `set()` operation adds a little syntactic sugar but since it's only called in one place the minor extra complexity doesn't gain much.

Next, we come to the core function, `sort_idxtbl()`:
``` cpp
template <class RAIter>
void sort_idxtbl( RAIter first, RAIter last, int* pidxtbl )
{
    int iDst = last-first;
    typedef std::vector< sort_idxtbl_pair<RAIter> > V;
    V v( iDst );
    
    int i=0;
    RAIter it = first;
    V::iterator vit = v.begin();
    for( i=0; it<last; it++, vit++, i++ )
        (*vit).set(it,i);
    
    std::sort(v.begin(), v.end());
    
    int *pi = pidxtbl;
    vit = v.begin();
    for( ; vit<v.end(); pi++, vit++ )
        *pi = (*vit).i;
}
```
7. Choose meaningful and appropriate names: In this case, `sort_idxtbl` is a misleading name because the function doesn't sort an index table... it creates one! On the other hand, the code gets good marks for using the template parameter name RAIter to indicate a random access iterator, which is what's required in this version of the code and so naming the parameter to indicate this is a good reminder.

8. Be consistent: In the function above, sometimes variables are initialized (or set) in for-init-statements, and sometimes they aren't. This just makes things harder to read, at least for me. Your mileage may vary on this one.

9. Remove gratuitous complexity: This function adores gratuitous local variables! It contains three examples. First, the variable iDst is initialized to last-first, and then used only once; why not just write last-first where it's used and get rid of clutter? Second, the vector iterator vit is created where a subscript was already available and could have been used just as well, and the code would have been clearer. Third, the local variable it gets initialized to the value of a function parameter, after which the function parameter is never used; my personal preference in that case is just to use the function parameter (even if you change its value \-\- that's okay!) instead of introducing another name.

10. Reuse Part 1: Reuse more of the standard library. Now, the original program gets good marks for reusing std::sort() \-\- that's good. But why handcraft that final loop to perform a copy when std::copy() does the same thing? Why reinvent a special-purpose sort_idxtbl_pair class when the only thing it has that std::pair doesn't is a comparison function? Besides being easier, reuse also makes our own code more readable. Humble thyself and reuse!

11. Reuse Part 2: Kill two birds with one stone by making the implementation itself more reusable. Of the original implementation, nothing is directly reusable other than the function itself. The helper sort_idxtbl_pair class is hardwired for its purpose and is not independently reusable.

12. Reuse Part 3: Improve the signature. The original signature
``` cpp
template <class RAIter>
void sort_idxtbl( RAIter first, RAIter last, int* pidxtbl )
```
takes a bald `int*` pointer to the output area, which I generally try to avoid in favor of managed storage (say, a vector). But at the end of the day the user ought to be able to call sort_idxtbl and put the output into a plain array, or a vector, or anything else. Well, the requirement "be able to put the output into any container" simply cries out for an iterator, doesn't it?
``` cpp
template< class RAIn, class Out >
void sort_idxtbl( RAIn first, RAIn last, Out result )
```
13. Reuse Part 4, or Prefer comparing iterators using `!=`: When comparing iterators, always use `!=` (which works for all kinds of iterators) instead of `< `(which works only for random-access iterators), unless of course you really need to use `<` and only intend to support random-access iterators. The original program uses < to compare the iterators it's given to work on, which is fine for random access iterators, which was the program's initial intent \-\- to create indexes into vectors and arrays, both of which support random-access iteration. But there's no reason we may not want to do exactly the same thing for other kinds of containers, like lists and sets, that don't support random-access iteration, and the only reason the original code won't work for such containers is that it uses `<` instead of `!=` to compare iterators.

As Scott Meyers puts it eloquently, "program in the future tense." He elaborates:

"Good software adapts well to change. It accommodates new features, it ports to new platforms, it adjusts to new demands, it handles new inputs. Software this flexible, this robust, and this reliable does not come about by accident. It is designed and implemented by programmers who conform to the constraints of today while keeping in mind the probable needs of tomorrow. This kind of software \-\- software that accepts change gracefully \-\- is written by people who program in the future tense."[4]

14. Prefer preincrement unless you really need the old value: Here, for the iterators, writing preincrement (`++i`) should habitually be preferred over writing postincrement (`i++`).[5] True, that probably doesn't make a material difference in the original code because `vector<T>::iterator` does not have to be a cheap-to-copy `T*` (although it almost always will be); but if we implement point 13 we may no longer be limited to just `vector<T>::iterators`, and most other iterators are of class type \-\- perhaps often still not too expensive to copy, but why introduce this possible inefficiency needlessly?

That covers most the important issues. There are other things we could critique but that I didn't consider important enough to call attention to here; for example, production code should have comments that document each class's and function's purpose and semantics, but that doesn't apply to code accompanying magazine articles where the explanation is typically written in better English and in greater detail than code comments have any right to expect.

I'm deliberately not critiquing the mainline for style (as opposed to the mechanical errors already noted above that would cause the mainline to fail to compile) because after all this particular mainline is only meant to be a demonstration harness to help readers of the magazine article see how the index table apparatus is meant to work, and it's the index table apparatus that's the intended focus.

**Conclusion**
Let's preserve the original code's basic interface choice instead of straying far afield and proposing alternate design choices.[6] Limiting our critique just to correcting the code for mechanical errors and basic style, then, consider the three alternative improved versions below. Each has its own benefits, drawbacks, and style preferences as explained in the accompanying comments. What all three versions have in common is that they are clearer, more understandable, and more portable code \-\- and that ought to count for something, in your company and in mine.
``` cpp
// An improved version of the code originally
// published in [1].
//
#include <vector>
#include <map>
#include <algorithm>


// Solution 1 does some basic cleanup but still
// preserves the general structure of the
// original's approach. We're down to 17 lines
// (even if you count "public:" and "private:"
// as lines), where the original had 23.
//
namespace Solution1
{
    template<class Iter>
    class sort_idxtbl_pair
    {
    public:
        void set( const Iter& it, int i ) { it_ = it; i_ = i; }
        
        bool operator<( const sort_idxtbl_pair& other ) const
          { return *it_ < *other.it_; }
        operator int() const { return i_; }
    
    private:
        Iter it_;
        int i_;
    };
    
    // This function is where most of the clarity
    // savings came from; it has 5 lines, where
    // the original had 13. After each code line,
    // I'll show the corresponding original code
    // for comparison. Prefer to write code that
    // is clear and concise, not unnecessarily
    // complex or obscure!
    //
    template<class IterIn, class IterOut>
    void sort_idxtbl( IterIn first, IterIn last, IterOut out )
    {
        std::vector<sort_idxtbl_pair<IterIn> > v(last-first);
            // int iDst = last-first;
            // typedef std::vector< sort_idxtbl_pair<RAIter> > V;
            // V v(iDst);
      
      
        for( int i=0; i < last-first; ++i )
            v[i].set( first+i, i );
            // int i=0;
            // RAIter it = first;
            // V::iterator vit = v.begin();
            // for (i=0; it<last; it++, vit++, i++)
            // (*vit).set(it,i);
      
      
        std::sort( v.begin(), v.end() );
            // std::sort(v.begin(), v.end());
      
      
        std::copy( v.begin(), v.end(), out );
            // int *pi = pidxtbl;
            // vit = v.begin();
            // for (; vit<v.end(); pi++, vit++)
            // *pi = (*vit).i;
    }
}


// Solution 2 uses a pair instead of reinventing
// a pair-like helper class. Now we're down to
// 13 lines, from the original 23. Of the 14
// lines, 9 are purpose-specific, and 5 are
// directly reusable in other contexts.
//
namespace Solution2
{
    template<class T, class U>
    struct ComparePair1stDeref
    {
        bool operator()(const std::pair<T,U>& a,
                        const std::pair<T,U>& b) const
        { return *a.first < *b.first; }
    };
    
    template<class IterIn, class IterOut>
    void sort_idxtbl( IterIn first, IterIn last, IterOut out )
    {
        std::vector< std::pair<IterIn,int> > s( last-first );
        for( int i=0; i < s.size(); ++i )
            s[i] = std::make_pair(first+i, i);
            
        std::sort(s.begin(), s.end(),
                  ComparePair1stDeref<IterIn,int>() );
                  
        for( int i=0; i < s.size(); ++i, ++out)
            *out = s[i].second;
    }
}


// Solution 3 just shows a couple of alternative
// details -- it uses a map to avoid a separate
// sorting step, and it uses std::transform()
// instead of a handcrafted loop. Here we still
// have 15 lines, but more are reusable. This
// version uses more space overhead and probably
// more time overhead too, so I prefer Solution 2,
// but this is an example of finding alternative
// approaches to a problem.
//
namespace Solution3
{
    template<class T>
    struct CompareDeref
    {
        bool operator()( const T& a, const T& b ) const
        { return *a < *b; }
    };
    
    template<class T, class U>
    struct Pair2nd
    {
        const U& operator()( const std::pair<T,U>& a ) const
        { return a.second; }
    };
    
    template<class IterIn, class IterOut>
    void sort_idxtbl( IterIn first, IterIn last, IterOut out )
    {
        std::multimap<IterIn, int, CompareDeref<IterIn> > v;
        for( int i=0; first != last; ++i, ++first )
            v.insert( std::make_pair( first, i ) );
        std::transform(v.begin(), v.end(), out,
                       Pair2nd<IterIn const,int>() );
    }
}

// I left the test harness essentially unchanged,
// except to demonstrate putting the output in an

// output iterator (instead of necessarily an
// int*) and using the source array directly as a
// container.
//
#include <iostream>

int main()
{
    int ai[10] = { 15,12,13,14,18,11,10,17,16,19 };
    
    std::cout << "#################" << std::endl;
    std::vector<int> aidxtbl( 10 );
    
    // use another namespace name to test a different solution
    Solution3::sort_idxtbl( ai, ai+10, aidxtbl.begin() );
    
    
    for( int i=0; i<10; ++i )
    std::cout << "i=" << i
              << ", aidxtbl[i]=" << aidxtbl[i]
              << ", ai[aidxtbl[i]]=" << ai[aidxtbl[i]]
              << std::endl;
    std::cout << "#################" << std::endl;
}
```

**Notes**
1. C. Hicks. "Creating an Index Table in STL" (C/C++ Users Journal, 18(8), August 2000).
2. Available at http://www.gotw.ca/gotw/072.htm.
3. H. Sutter. ["Migrating to Namespaces"](http://www.gotw.ca/publications/migrating_to_namespaces.htm) (Dr. Dobb's Journal, 25(10), October 2000).
4. S. Meyers. More Effective C++, Item 32 (Addison-Wesley, 1996).
5. H. Sutter. Exceptional C++ (Addison-Wesley, 2000).
6. The original author also reports separate feedback from [Steve Simpson](ssimpson@mediaone.net) demonstrating another elegant, but substantially different, approach: Simpson creates a containerlike object that wraps the original container, including its iterators, and allows iteration using the alternative ordering.



# 074 Uses and Abuses of Vector 
**Difficulty: 4 / 10**
Almost everybody uses std::vector, and that's good. Unfortunately, many people misunderstand some of its semantics and end up unwittingly using it in surprising and dangerous ways. How many of the subtle problems illustrated in this issue might be lurking in your current program?

----

**Problem**
**JG Question**
1. Given a vector<int> v, what is the difference between the lines marked A and B?
``` cpp
// Example 1: [] vs. at()

void f( vector<int>& v )
{
    v[0];      // A
    v.at(0);   // B
}
```
**Guru Question**
2. Consider the following code:
``` cpp
// Example 2: Some fun with vectors

vector<int> v;

v.reserve(2);
assert(v.capacity() == 2);
v[0] = 1;
v[1] = 2;
for(vector<int>::iterator i = v.begin(); i < v.end(); i++)
{
    cout << *i << endl;
}

cout << v[0];
v.reserve( 100 );
assert( v.capacity() == 100 );
cout << v[0];

v[2] = 3;
v[3] = 4;
// ...
v[99] = 100;
for(vector<int>::iterator i = v.begin(); i < v.end(); i++)
{
    cout << *i << endl;
}
```
Critique this code. Consider both style and correctness.

----

**Solution**
**Accessing Vector Elements**
**1. Given a vector<int> v, what is the difference between the lines marked A and B?**
``` cpp
// Example 1: [] vs. at()

void f( vector<int>& v )
{
    v[0];      // A
    v.at(0);   // B
}
```
In Example 1, if v is not empty then there is no difference between lines A and B. If v is empty, then line B is guaranteed to throw a `std::out_of_range` exception, but there's no telling what line A might do.

There are two ways to access contained elements within a `vector`. The first, `vector<T>::at()`, is required to perform bounds checking to ensure that the `vector` actually contains the requested element. It doesn't make sense to ask for, say, the 100th element in a `vector` that only contains 10 elements at the moment, and if you try to do such a thing, `at()` will protest by throwing a `std::out_of_range` hissy fit (a.k.a. "exception").

On the other hand, `vector<T>::operator[]()` is allowed, but not required, to perform bounds checking. There's not a breath of wording in the standard's specification for `operator[]()` that says anything about bounds checking, but neither is there any requirement that it have an exception specification, so your standard library implementer is free to add bounds-checking to `operator[]()`, too. So, if you use `operator[]()` to ask for an element that's not in the vector, you're on your own, and the standard makes no guarantees about what will happen (although your standard library implementation's documentation might) \-\- your program may crash immediately, the call to `operator[]()` might throw an exception, or things may seem to work and occasionally and/or mysteriously fail.

Given that bounds checking protects us against many common problems, why isn't `operator[]()` required to perform bounds checking? The short answer is: Efficiency. Always checking bounds would cause a (possibly slight) performance overhead on all programs, even ones that never violate bounds. The spirit of C++ includes the dictum that, by and large, you shouldn't have to pay for what you don't use, and so bounds checking isn't required for `operator[]()`. In this case we have an additional reason to want the efficiency: vectors are intended to be used instead of built-in arrays, and so should be as efficient as built-in arrays, which don't do bounds-checking. If you want to be sure that bounds get checked, use `at()` instead.

**Size()ing Up Vector**
Let's turn now to Example 2, which manipulates a `vector<int>` using a few simple operations.

2. Consider the following code:
``` cpp
// Example 2: Some fun with vectors

vector<int> v;

v.reserve( 2 );
assert( v.capacity() == 2 );
```
This assertion has two problems, one substantive and one stylistic.

**2.1. The substantive problem is that this assertion might fail.**

Why might it fail? Because the call to `reserve()` will guarantee that the vector's `capacity()` is at least 2 \-\- but it might also be greater than 2, and indeed is likely to be greater because a vector's size must grow exponentially and so typical implementations may choose to always grow the internal buffer on exponentially increasing boundaries even when specific sizes are requested via `reserve()`.

So this comparison should instead test using `>=`, not strict equality:
``` cpp
assert( v.capacity() >= 2 );
```
This leaves us with the other potential issue:

**2.2. The stylistic problem is that the (corrected) assertion is redundant.**

The standard already guarantees what is here being asserted. Why add needless clutter by testing for it explicitly? This doesn't make sense unless you suspect a bug in the standard library implementation you're using, in which case you have bigger problems.
``` cpp
v[0] = 1;
v[1] = 2;
```
Both of the above lines are flat-out errors, but they might be hard-to-find flat-out errors because they'll likely "work" after a fashion on your implementation of the standard library.

There's a big difference between `size()` (which goes with `resize()`) and `capacity()` (which goes with `reserve()`):
- `size()` tells you how many elements are currently actually present in the container, and `resize()` adjusts the actual contents of the container to be the specified size by adding or removing elements at the end. Both functions are available for list, vector, and deque, not other containers.
- `capacity()` tells you how many elements have room before adding another would force the vector to allocate more space, and `reserve()` grows (never shrinks) into a larger internal buffer if necessary to ensure at least the specified space is available. Both functions are available only for vector.

In this case, we used `v.reserve(2)` and therefore know that `v.capacity() >= 2`, and that's fine as far as it goes `--` but we never actually added any elements to `v`, so `v` is still empty! At this point `v` just happens to have room for two or more elements.

We can only safely use `operator[]()` (or `at()`) to modify elements that are actually present, which means that they count towards `size()`. At first you might wonder why `operator[]()` couldn't just be smart enough to add the element if it's not already there, but if `operator[]()` were allowed to work this way, you could create a vector with "holes" in it! For example, consider:
``` cpp
vector<int> v;
v.reserve( 100 );
v[99] = 42; // if this were allowed, then...
// ...now what are the values of v[0..98]?
```
Alas, because `operator[]()` isn't required to perform range checking, on most implementations the expression "`v[0]`" will simply return a reference to the not-yet-used space in the vector's internal buffer where the first element would eventually go. Therefore, the statement "`v[0] = 1;`" will probably appear to work, kind of, sort of, in that if the program were to `cout << v[0]` now the result would probably be 1, just as (mis)expected. Again, whether this will actually happen on the implementation you're using isn't guaranteed; it's just one typical possibility. The standard puts no requirements on what the implementation should do with flat-out errors like writing "`v[0]`" for an empty vector v, because the programmer is assumed to know enough not to write such things \-\- and after all, if the programmer had wanted the library to perform bounds checking, he would have written "`v.at(0)`", right?

Of course, the assignments "`v[0] = 1; v[1] = 2;`" would have been fine if the earlier code had performed a `v.resize(2)` instead of just a `v.reserve(2)` \-\- but it didn't, so they're not. Alternatively, it would have been fine to replace them with "`v.push_back(1); v.push_back(2);`" which is the always-safe way to tack elements onto the end of a container.

``` cpp
for(vector<int>::iterator i = v.begin(); i < v.end(); i++)
{
    cout << *i << endl;
}
```
First, note that this loop prints nothing because of course the vector is still empty. This might surprise the original programmer, until that programmer realizes that the above code never really added anything to the vector \-\- it was just (dangerously) playing around with some of the reserved but still-officially-unused space hanging around inside the vector.

Having said that, there is no outright error in the above loop, but there are several stylistic problems that I would comment on if I saw this code in a code review setting. Most are low-level comments:

1. Be as const-correct as possible.

The iterator is never used to modify the vector's contents, so it should be a const_iterator.

2. Prefer comparing iterators with `!=`, not `<`.

True, because `vector<int>::iterator` happens to be a random-access iterator (not necessarily an `int*`, of course!), there's no downside to using `<` in the comparison with `v.end()` above. But `<` only works with random-access iterators, whereas `!=` works with other iterator types too, so we should routinely use `!=` all the time unless we really need `<`.

3. Prefer using prefix `--` and `++`, instead of postfix.

Get in the habit of by default writing `++i` instead of `i++` in loops unless you really need the old value of `i`. For example, postfix is natural and fine when you're writing something like "`v[i++]`" to access the i-th element and at the same time increment a loop counter.

In this case, there's almost certainly no performance difference because often `vector<int>::iterator` really is just a plain old `int*` which is cheap to copy and anyway can have extra copies optimized away by the compiler, because the compiler is allowed to know about the semantics of `int*`'s.

4. Avoid needless recalculations.

In this case, `v.end()` doesn't change during the loop, so instead of recalculating it on every iteration it might be worthwhile to precalculate it before the loop begins.

Note: If your implementation's `vector<int>::iterator` is just an `int*`, and your implementation inlines end() and does reasonable optimization, it's probable that there's zero overhead here anyway because the compiler will probably be able to see that the value of `end()` doesn't change and that it can therefore safely hoist the code out of the loop. This is a pretty common case. However, if your implementation's `vector<int>::iterator` is not an `int*` (for example, if it's a debugging implementation it could be of class type), the `function(s)` are not inlined, and/or the compiler doesn't perform the suggested optimizations, then hoisting the calculation code out of the loop yourself can make a performance difference.

5. Prefer `'\n'` to endl.

Using endl forces the stream to flush its internal output buffers. If the stream is buffered and you don't really need a flush each time, just write a flush once at the end of the loop and your program will perform that much faster.

Finally, the last comment hits at a higher level:

6. Prefer reusing the standard library's `copy()` and `for_each()` instead of handcrafting your own loops, where using the standard facilities is clean and easy. Season to taste.

I say "season to taste" because here's one of those places where taste really does matter. In simple cases, `copy()` and `for_each()` really can and do improve readability over handcrafted loops. Beyond those simple cases, though, unless you have a nice expression template library available, code written using `for_each()` can get unreadable pretty quickly because the code in the loop body has to be split off into functors. Sometimes that kind of factoring is still a good thing; other times it's merely obscure.

That's why your tastes may vary. Still, in this case I would be tempted to replace the above loop with something like:
``` cpp
copy( v.begin(), v.end(),
      ostream_iterator<int>(cout, "\n") );
```
Besides, when you reuse copy() like this, you can't get the `!=`, `++`, `end()`, and endl parts wrong because they're done for you. (Of course, this assumes that you don't want to flush the output stream after each int is output; if you do, there's no way to do it without writing your own loop instead of reusing `std::copy()`.) Reuse, when applied well, not only makes code more readable but can also make it better by avoiding some opportunities for pitfalls.

You can take it a step further and write a container-based algorithm for copying \-\- that is, an algorithm that operates on an entire container, not just an iterator range. Doing this would also automatically get the `const_iterator` part right. For example:
``` cpp
template<class Container, class OutputIterator>
OutputIterator copy( const Container& c, OutputIterator result )
{
    return std::copy( c.begin(), c.end(), result );
}
```
Here we simply wrap `std::copy()` for an entire container, and because the container is taken by const& the iterators will automatically be `const_iterators`.

``` cpp
cout << v[0];
```
When the program performs "`cout << v[0]`" now, it will probably produce a 1. This is because the program scribbled over memory in a way that it shouldn't have, but that will probably not cause an immediate fault \-\- more's the pity.

``` cpp
v.reserve( 100 );
assert( v.capacity() == 100 );
```
Again, this assertion should use `>=`, and then becomes redundant anyway, as before.
``` cpp
cout << v[0];
```
Surprise! This time, the "`cout << v[0]`" will probably produce a 0 \-\- the value 1 that we just set has mysteriously vanished!

Why? Assuming that the `reserve(100)` actually did trigger a reallocation of v's internal buffer (i.e., if the initial call to `reserve(2)` didn't already raise `capacity()` to 100 or more), v would only copy over into the new buffer the elements it actually contains \-\- and it doesn't actually think it contains any! The new buffer initially holds zeroes, and that's what stays there.
``` cpp
v[2] = 3;
v[3] = 4;
// ...
v[99] = 100;
```
No doubt you are even now shaking your head sadly at this deplorable code. This is bad, bad, bad... but because `operator[]()` isn't required to perform bounds-checking, on most implementations this will probably silently appear to work and won't cause an immediate exception or memory trap.

If instead the user had written:
``` cpp
v.at(2) = 3;
v.at(3) = 4;
// ...
v.at(99) = 100;
```
then the problem would have become obvious, because the very first call would have thrown an `out_of_range` exception.

``` cpp
for(vector<int>::iterator i = v.begin(); i < v.end(); i++)
{
    cout << *i << endl;
}
```

Again this prints nothing, and I'd consider replacing it with:
``` cpp
copy(v.begin(), v.end(), ostream_iterator<int>(cout, "\n"));
```
Notice again that this reuse automatically solves the `!=`, prefix `++`, `end()`, and endl comments too \-\- the opportunity to get them wrong never arises! Good reuse often makes code automatically faster and safer, too.

**Summary**
Know the difference between `size()` and `capacity()`. Know also the difference between `operator[]()` and `at()`, and always use the latter if you want bounds-checked access. Doing so can easily save you long hours of sweat and tears inside a debugger.



# 075 Istream Initialization? 
**Difficulty: 3 / 10**
Most people know the famous quote: "What if they gave a war and no one came?" This time, we consider the question: "What if we initialized an object and nothing happened?" As Scarlett might say in such a situation: "This isn't right, I do declare!"

----

**Problem**
Assume that the relevant standard headers are included and that the appropriate using-declarations are made.

**JG Question**
1. What does the following code do?
``` cpp
deque<string> coll1;

copy(istream_iterator<string>(cin), istream_iterator<string>(), back_inserter(coll1));
```
**Guru Questions**
2. What does the following code do?
``` cpp
deque<string> coll2(coll1.begin(), coll1.end());

deque<string> coll3(istream_iterator<string>(cin), istream_iterator<string>());
```
3. What must be changed to make the code do what the programmer probably expected?

----

**Solution**
**1. What does the following code do?**
``` cpp
// Example 1(a)

deque<string> coll1;

copy(istream_iterator<string>(cin), istream_iterator<string>(), back_inserter(coll1));
```
This code declares a deque of strings called coll1 that is initially empty. It then copies every whitespace-delimited string in the standard input stream (`cin`) into the deque using `deque::push_back(`), until there is no more input available.

The above code is equivalent to:
``` cpp
// Example 1(b): Equivalent to 1(a)

deque<string> coll1;

istream_iterator<string> first(cin), last; 
copy(first, last, back_inserter(coll1));
```
The only difference is that in Example 1(a) the `istream_iterator` objects are created on the fly as unnamed temporary objects, and so they are destroyed at the end of the `copy()` statement. In Example 1(b), the `istream_iterator` objects are named variables, and survive the `copy()` statement; they won't be destroyed until the end of whatever scope surrounds the above code.

**2. What does the following code do?**
``` cpp
// Example 2(a): Declaring another deque

deque<string> coll2(coll1.begin(), coll1.end());
```
This code declares a second deque of strings called coll2, and initializes it using the deque constructor that takes a pair of iterators corresponding to a range from which the contents should be copied. In this case, we're initializing coll2 from an iterator range
that happens to correspond to "everything that's in coll1."

The code so far in Example 2(a) is nearly equivalent to:
``` cpp
// Example 2(b): Almost the same as Example 2(a)

// extra step: call default constructor
deque<string> coll2;

// append elements using push_back()
copy(coll1.begin(), coll1.end(), back_inserter(coll2));
```
The (minor) difference is that coll2's default constructor is called first and then the elements are pushed into the collection as a separate step, using push_back(). The original code simply did it all using the constructor that takes an iterator pair, which probably (though not necessarily) does exactly the same thing under the covers.

You might wonder why I've belabored this syntax. The reason will become clear as we take a look at the last part of the code, which is unfortunately much more benign than some might think:
``` cpp
// Example 2(c): Declaring yet another deque?

deque<string> coll3(istream_iterator<string>(cin), istream_iterator<string>());
```
The above code looks at first blush like it's trying to do the same thing as Example 1(a), namely create a deque of strings populated from the standard input, except that it's trying to do it using the syntax of Example 2(a), namely using the iterator range constructor. This has one potential problem, and one actual problem: The potential problem is that cin is exhausted, so there's no input left to read as was probably intended, which may be a logical problem.

The big problem, though, is that the code doesn't actually do anything at all. Why not? Because it doesn't actually declare a `deque<string>` object named coll3. What it actually declares is (take a deep breath here):

        a function named coll3
            that returns a deque<string> by value
            and takes two parameters:
                an istream_iterator<string> with a formal parameter name of cin,
                and a function with no formal parameter name
                    that returns an istream_iterator<string>
                    and takes no parameters.

(Say that three times fast.)

What's going on here? Basically, we're running across one of the painful rules that C++  inherited from C, to maintain C compatibility: If a piece of code can be interpreted as a declaration, it will be. In the words of the C++ standard, clause 6.8:

There is an ambiguity in the grammar involving expression-statements and declarations: An expression-statement with a function-style explicit type conversion (_expr.type.conv_) as its leftmost subexpression can be indistinguishable from a declaration where the first declarator starts with a (. In those cases the statement is a declaration.

Without going into the gory details, the reason why this is the way that it is comes down to helping compilers deal with C's horrid declaration syntax, which can be ambiguous \-\- and so to make things manageable the compiler resolves such ambiguities by universally assuming that "if in doubt, it must be a function declaration." 'Nuff said.

If you haven't already, take a quick look at GotW #1 (Exceptional C++ Item 42)[1], which contains a similar but simpler example. Let's dissect the declaration step by step to see what's going on:
``` cpp
// Example 2(d): Identical to Example 2(c), removing
// redundant parentheses and adding a typedef

typedef istream_iterator<string> (Func)();

deque<string> coll3(istream_iterator<string> cin, Func);
```
Does that look more like a function declaration? Maybe so, maybe not, so let's take another step and remove the formal parameter name `"cin"` which is ignored anyway, and change the name "coll3" to something that we usually expect to see as a function name:
``` cpp
// Example 2(e): Still identical to Example 2(c)
//
typedef istream_iterator<string> (Func)();

deque<string> f(istream_iterator<string>, Func);
```
Now it's pretty clear: The above "could be" a function declaration, and so according to the C and C++ syntax rules, it is one. What makes it confusing is that it looks a lot like constructor syntax; what makes it downright obscure is that the formal parameter name, cin, happens to resemble the name of a variable that is indeed in scope and is even defined by the standard \-\- because that's what it was in fact intended to be \-\- but, misleading as it is, that doesn't matter, for the formal parameter name and std::cin have nothing in common other than the accident that they happen to be spelled the same way.

People still run across this problem from time to time in real-world coding, and that's the reason why this problem deserves its own article. Because the code is (probably surprisingly) just a function declaration, it doesn't actually do anything \-\- no code gets generated by the compiler, no actions are performed, no deque constructors are called, no objects are created.

It wouldn't be fair to throw up an example like this, however, without also showing how you can fix it. This brings us to the final question:

**3. What must be changed to make the code do what the programmer probably expected?**

All we need is something that makes it impossible for the compiler to treat the code as a function declaration. There are two easy ways to do it. Here's the seductive way:
``` cpp
// Example 3(a): Disambiguate the syntax,
// say by adding extra parens
// (okay solution, score 7/10)
//
deque<string> coll3((istream_iterator<string>(cin)), istream_iterator<string>());
```
Here just adding the extra parentheses around the parameters makes it clear to the compiler that what we intend to be constructor parameter names can't be parameter declarations. This is because although `"istream_iterator<string>(cin)"` can be a variable (or parameter declaration, as noted above, `"(istream_iterator<string>(cin))"` can't \-\- the code in Example 3(a) can't be a function declaration for the same reason that `"void f((int i))"` can't be, namely because of the extra parentheses which are illegal around a whole parameter declaration.

There are other ways to try to disambiguate this by forcing the statement out of the declaration syntax, but I won't present them for a simple reason: They only work if both you and your compiler understand this corner case of the standard very well.

This declaration-vs.-constructor syntax ambiguity is by its nature such a thorny edge case that the best thing to do is just avoid the ambiguity altogether, and not rely on methods that essentially amount to coaxing and wheedling a surly three-year-old compiler into treating it as a declaration. Put another way, if you were talking to someone, would you purposely say something ambiguous and then change it slightly by adding, "well, what I really meant was..."? Hardly.

It's far better to avoid the ambiguity in the first place. I prefer and recommend the following alternative because it's much easier to get right, it's utterly understandable to even the weakest compilers, and it makes the code clearer to read to boot:
``` cpp
// Example 3(b): Use named variables
// (recommended solution, score 10/10)
//
istream_iterator<string> first(cin), last;
deque<string> coll3(first, last);
```
Actually, in both Example 3(a) and 3(b), making the suggested change to just one of the parameters would have been sufficient, but for consistency I've treated both parameters the same way.

Guideline: Prefer using named variables as constructor parameters. This avoids possible declaration ambiguities.

Notes
1. H. Sutter. [Exceptional C++](http://www.gotw.ca/publications/xc++.htm) (Addison-Wesley, 2000).




# 076 Uses and Abuses of Access Rights 
**Difficulty: 6 / 10**
Who really has access to your class's internals? This issue is about liars, cheats, pickpockets, and thieves, and how to recognize and avoid them.

----

**Problem**
**JG Question**
**1.** What code can access the following parts of a class:
   a) public
   b) protected
   c) private

**Guru Questions**
**2.** Consider the following header file:
``` cpp
// File x.h 

class X 
{ 
public:
    X() : private_(1) { /*...*/ }
    
    template<class T>
    void f( const T& t ) { /*...*/ }
    
    int Value() { return private_; }
    // ...
private: 
    int private_; 
};
```
Demonstrate:
    a. a non-standards-conforming and non-portable hack; and
    b. a fully standards-conforming and portable technique

​    for any calling code to get direct access to this class's private_ member.

**3.** Is this a hole in C++'s access control mechanism, and therefore a hole in C++'s encapsulation? Discuss.

----

**Solution**
This issue of GotW is about liars, cheats, pickpockets, and thieves.

**1. What code can access the following parts of a class:**
   a) public
   b) protected
   c) private

In short:
   a) Public members can be accessed by any code.
   b) Protected members can be accessed by the class's own member functions and friends, and by the member functions and friends of derived classes.
   c) Private members can be accessed by the class's own member functions and friends only.

That's the usual answer, and it's true as far as it goes. In this issue of GotW, we consider a special case where the above doesn't quite hold, because C++ sometimes provides a way that makes it legal (if not moral) to subvert access to a class's private members.

**2. [...] Demonstrate:**
   **a) a non-standards-conforming and non-portable hack; and**
   **b) a fully standards-conforming and portable technique**

for any calling code to get direct access to this class's private_ member.

There's a strange and perverse fascination that makes people stare at car wrecks, underworld hit men, and evil code hackery, so we might as well begin with a visit to a tragic "hit" scene in (a) and get it out of the way.

For a non-standards-conforming and non-portable hack, several ideas come to mind. Here are three of the more infamous offenders, all probable cousins of the hit man:

**Criminal #1: The Liar**

The Liar's hack of choice is duplicate a forged class definition to make it say what he wants it to say. For example:
``` cpp
// Example 1: Lies and forgery

class X
{
    // instead of including x.h, manually duplicates X's
    // definition, and adds a line like:
    friend ::Hijack( X& );
};

void Hijack( X& x )
{
    x.private_ = 2; // evil laughter here
}
```
This man is a Liar. Mark him well, for he cannot be trusted.

This is illegal because it violates the One Definition Rule, which says that if a type (here X) is defined more than once, the definitions must be identical. The object being used above may be called an X and may look like an X, but it's not the same kind of X all the other code in the program is using.

Still, this hack will work on most compilers because usually the underlying object data layout will still be the same, and if so then the Liar may be able to get away with his lie for a time.

**Criminal #2: The Pickpocket**

The Pickpocket's hack of choice is to silently change the meaning of the class definition. For example:
``` cpp
// Example 2: Evil macro magic

#define private public // illegal
#include "x.h"

void Hijack( X& x )
{
    x.private_ = 2; // evil laughter here
}
```
This man is a pickpocket. Mark him well, for his fingers are light.

The code in Example 2 is nonportable for two reasons:
   a) It is illegal to #define a reserved word.
   b) It violates the One Definition Rule, as above. Still, if the object's data layout is unchanged, the hack may seem to work for a while.

**Criminal #3: The Cheat**

The Cheat's modus operandi is to substitute one item when you're expecting another. For example:
``` cpp
// Example 3: Nasty attempt to simulate the object layout
// (cross your fingers and toes).

class BaitAndSwitch
{    // hopefully has the same data layout as X, so we can pass him off as one 
public:
    int notSoPrivate;
};

void f( X& x )
{
    // evil laughter here
    (reinterpret_cast<BaitAndSwitch&>(x)).notSoPrivate = 2;
}
```
This man is a cheat. Mark him well, for he's the kind of bait-and-switch artist who runs newspaper ads just to get you into his store, and then claims not to have the advertised item and tries to fob off something else of lesser value and higher price.

The code in Example 3 is illegal for two reasons:
   a) The object layouts of X and BaitAndSwitch are not guaranteed to be the same, although in practice they probably always will be.
   b) The results of the reinterpret_cast are undefined, although most compilers will let you try to use the resulting reference in the way the hacker intended.

We're also asked to look for "a fully standards-conforming and portable technique." Alas, while many criminals and hackers are smelly and unwashed and nonconforming, some do conform and have an air of respectability:

**Person #4: The Language Lawyer**

Consider the following code:
``` cpp
// Example 4: The legal weasel

namespace
{
    struct Y {};
}

template<>
void X::f( const Y& )
{
    private_ = 2; // evil laughter here
}

void Test()
{
    X x;
    cout << x.Value() << endl; // prints 1
    x.f( Y() );
    cout << x.Value() << endl; // prints 2
}
```
This man is a language lawyer who knows the loopholes. He will never be caught, for he is careful to obey the letter of the law while pillaging its spirit. Mark and avoid such ungentlemen.

Example 4 exploits the fact that X has a member template. The code is entirely conforming and is guaranteed by the standard to work as expected. The reason is twofold:

a) It's legal to specialize a member template on any type.
   The only room for error would be if you tried to specialize it on the same type twice in different ways, which would be a One Definition Rule violation, but we get around that because:
b) The code uses a type that's guaranteed to have a unique name, because it's in the hacker's unnamed namespace. Therefore it is guaranteed to be legal and won't tromp on anyone else's specialization.

There remains only one question:

**3. Is this a hole in C++'s access control mechanism, and therefore a hole in C++'s encapsulation? Discuss.**

This demonstrates an interesting interaction between two C++ features: The access control model, and the template model. It turns out that member templates appear to implicitly "break encapsulation" in the sense that they effectively provide a portable way to bypass the class access control mechanism.

This isn't actually a problem. The issue here is of protecting against Murphy vs. protecting against Machiavelli... that is, protecting against accidental misuse (which the language does very well) vs. protecting against deliberate abuse (which is effectively impossible). In the end, if a programmer wants badly enough to subvert the system, he'll find a way, as demonstrated above in Examples 1 to 3.

The real answer to the issue is: Don't do that! Admittedly, even Scott Meyers has said publicly that there are times when it's tempting to have a quick way to bypass the access control mechanism temporarily, such as to produce better diagnostic output during debugging... but it's just not a habit you want to get into for production code, and it should appear on the list of "one-warning offences" in your development shop.

From the GotW coding standards:  

    never subvert the language; for example, never attempt to break encapsulation by copying a class definition and adding a friend declaration, or by providing a local instantiation of a template member function (GotW #76)



# 077 #Definition 
**Difficulty: 4 / 10**
What can and can't macros do? Not all compilers agree.

----

**Problem**
**JG Question**
1. Demonstrate how to write a simple `max()` preprocessor macro that takes two arguments and evaluates to the one that is greater, using normal `<` comparison. What are the usual pitfalls in writing such a macro?

**Guru Question**
2. What can a preprocessor macro not create? Why not?

----

**Solution**
**Common Macro Pitfalls**
**1. Demonstrate how to write a simple `max()` preprocessor macro that takes two arguments and evaluates to the one that is greater, using normal `<` comparison. What are the usual pitfalls in writing such a macro?**

There are four major pitfalls, besides several further drawbacks. Focusing on the pitfalls first, here are common ways to go wrong when writing a macro.

**1. Don't forget to put parentheses around arguments.**
``` cpp
// Example 1(a): Paren pitfall #1: arguments

#define max(a,b) a < b ? b : a
```
The problem here is that the uses of the parameters a and b are not fully parenthesized. Macros only do straight textual substitution, so this can cause some unexpected results. For example:
``` cpp
    max( i += 3, j )
```
expands to the following:
``` cpp
    i += 3 < j ? j : i += 3
```
which, because of operator precedence and language rules, actually means:
``` cpp
    i += ((3 < j) ? j : i += 3)
```
This could cause some long debugging sessions.

**2. Don't forget to put parentheses around the whole expression.**

Fixing the first problem, we still fall prey to another subtlety:
``` cpp
// Example 1(b): Paren pitfall #2: expansion

#define max(a,b) (a) < (b) ? (b) : (a)
```
The problem now is that the entire expansion is not correctly parenthesized. For example:
``` cpp
    k = max( i, j ) + 42;
```
expands to the following:
``` cpp
    k = (i) < (j) ? (j) : (i) + 42;
```
which, because of operator precedence, actually means:
``` cpp
    k = (((i) < (j)) ? (j) : ((i) + 42));
```
If i >= j, k is assigned the value of i+42, as intended. But if `i < j`, `k` is assigned the value of `j`.

**3. Be aware of the results of any possible multiple evaluation.**

We can fix problem #2 by putting parentheses around the entire macro expansion, but this leaves us with yet another problem:
``` cpp
// Example 1(c): Multiple argument evaluation

#define max(a,b) ((a) < (b) ? (b) : (a))
```
Now, consider what happens if one or both of the expressions has side effects:
``` cpp
    max( ++i, j )
```
If the result of `++i >= j`, `i` gets incremented twice, which is probably not what the programmer intended:
``` cpp
    ((++i) < (j) ? (j) : (++i))
```
Similarly, consider the code:
``` cpp
    max( f(), pi )
```
which expands to:
``` cpp
    ((f()) < (pi) ? (pi) : (f()))
```
If the result of `f()` `>=` `pi`, `f()` gets executed twice, which is almost certainly inefficient and often actually wrong.

Alas, although we could work around the first two problems, this one is a corker `--` there is no solution as long as max is a macro.

**4. Beware scope.**

Finally, macros don't care about scope. (They don't care about much of anything; see GotW #63.) They just perform textual substitution no matter where the text may be. This means that, if we use macros at all, we have to be very careful about what we name them. In particular, the biggest problem with the max macro is that it is highly likely to interfere with the standard max() function template:
``` cpp
// Example 1(d): Name tromping

#define max(a,b) ((a) < (b) ? (b) : (a))

#include <algorithm> // oops!
```
The problem is that, inside header `<algorithm>`, there will be something like the following:
``` cpp
template<class T> const T& 
max(const T& a, const T& b);
```
which the macro "helpfully" turns into an uncompilable mess:
``` cpp
template<class T> const T& 
((const T& a) < (const T& b) ? (const T& b) : (const T& a));
```
If you think that's easy to avoid by putting your macro definition after all #included header files (which really is a good idea in any case), just imagine what the macro does to all your other code that happens to have variables or other things that just happen to be named max.

If you have to write a macro, try to give it an unusual and hard-to-spell name that will be less likely to tromp on other names.

**Other Macro Drawbacks**
There are a few other major things a macro can't do:

**5. Macros can't recurse.**

We can write a recursive function, but it's impossible to write a recursive macro. As the C++ standard says, in 16.3.4/2:

If the name of the macro being replaced is found during this scan of the replacement list (not including the rest of the source file's pre- processing tokens), it is not replaced. Further, if any nested replacements encounter the name of the macro being replaced, it is not replaced. These nonreplaced macro name preprocessing tokens are no longer available for further replacement even if they are later (re)examined in contexts in which that macro name preprocessing token would otherwise have been replaced.

**6. Macros don't have addresses.**

It's possible to form a pointer to any free or member function (for example, to use it as a predicate), but it's not possible to form a pointer to a macro because a macro has no address. Why not should be obvious: Macros aren't code. A macro doesn't have any existence of its own, because all it is is a glorified (and not particularly glorious) text substitution rule.

**7. Macros are debugger-unfriendly.**

Besides the fact that macros change the underlying code before the compiler gets a chance to see it, and therefore can wreak havoc with variable and other names, a macro can't be stepped into during debugging.

Have you heard the one about the scientists who started experimenting on lawyers instead of on laboratory macros? It was because...

**There Are Some Things Even a Macro Just Won't Do**
There are valid reasons to use macros (see GotW #32), but there are limits. This brings us to the final question:

**2. What can a preprocessor macro not create? Why not?**

In the standard, clause 2.1 defines the phases of translation. Preprocessing directives and macro expansions take place in phase 4. Thus, on a compliant compiler, it is not possible for a macro to create:
- a trigraph (trigraphs are replaced in phase 1);
- a universal character name (\uXXX, replaced in phase 1);
- an end-of-line line-splicing backslash (replaced in phase 2);
- a comment (replaced in phase 3);
- another macro; or
- changes to a character literal or string literal via macro names inside the strings. For this last point, as noted in 16.3/8 footnote 7:  


    Since, by macro-replacement time, all character literals and string literals are preprocessing tokens, not sequences possibly containing identifier-like subsequences (see 2.1.1.2, translation phases), they are never scanned for macro names or parameters.

A recent CUJ article[1] claimed that it's possible for a macro to create a comment:
``` cpp
#define COMMENT SLASH(/)
#define SLASH(s) /##s
```
This is nonstandard and not portable, but it actually works on some compilers. Why? Because those compilers don't implement the phases of translation exactly correctly. Here are the results from four compilers I tried:

| Compiler | Accepts Comment Macro? |
| ---- | ---- |
| Microsoft Visual Studio.NET<br/>(Visual C++ version 7), beta 1 | Yes (wrong) |
| Borland BCC 5.5.1 | Yes (wrong) |
| GCC 2.95.2 | No (correct) |
| Comeau 4.2.44 | No (correct) |

**Notes**
1. M. Timperley. "A C/C++ Comment Macro" (C/C++ Users Journal, 19(1), January 2001).



# 078 Operators, Operators Everywhere
**Difficulty: 4 / 10**
How many operators can you put together, when you really put your mind to it? This issue takes a break from production coding to get some fun C++ exercise.

----

**Problem**
**JG Question**
1. What is the greatest number of plus signs (`+`) that can appear consecutively, without whitespace, in a valid C++ program?

Note: Of course, plus signs in comments, preprocessor directives and macros, and literals don't count. That would be too easy.

**Guru Question**
2. Similarly, what is the greatest number of each the following characters that can appear consecutively, without whitespace, outside comments in a valid C++ program?
   a. &
   b. <
   c. |

For example, for (a), the code `"if( a && b )"` trivially demonstrates two consecutive & characters in a valid C++ program.

----

**Solution**
**Who Is Max Munch, and What's He Doing In My C++ Compiler?**
Max Munch is the only C++ standards committee member to have a perfect attendance record at all meetings from the very beginning to date. (If you don't believe me, check the attendance list in each meeting's minutes.)

More seriously, the "max munch" rule says that, when interpreting the characters of source code into tokens, the compiler does it greedily \-\- it makes the longest tokens possible. Therefore `>>` is always interpreted as a single token, the stream extraction (right-shift) operator, and never as two individual `>` tokens even when the characters appear in a situation like this:
``` cpp
  template<class T = X<Y>> ...
```
That's why such code has to be written with an extra space, as:
``` cpp
  template<class T = X<Y> > ...
```
Similarly, `>>>` is always interpreted as `>> >`, never as `> >>`, and so on.

**Some Fun With Operators**
1. What is the greatest number of plus signs (`+`) that can appear consecutively, without whitespace, in a valid C++ program?

It is possible to create a source file containing arbitrarily long sequences of consecutive `+` characters, up to a compiler's limits (such as the maximum source file line length the compiler can handle).

If the sequence has an even number of `+` characters, it will be interpreted as `++ ++ ++ ++ ... ++`, a sequence of binary `++` tokens. To make this work, and have well-defined semantics because of sequence points, all we need is a class with a user-defined prefix `++` operator that allows chaining. For example:
``` cpp
  // Example 1(a)
  //
  class A
  {
  public:
    A& operator++() { return *this; }
  };
```
Now we can write code like:
``` cpp
  A a;
  ++++++a; // meaning: ++ ++ ++ a;
```
which works out to:
``` cpp
  a.operator++().operator++().operator++()
```
What if the sequence has an odd number of `+` characters? Then it will be interpreted as `++ ++ ++ ++ ... ++ +`, a series of binary `++` tokens ending with a final unary `+`. To make this work, we just need to additionally provide a unary `+` operator. For example:
``` cpp
  // Example 1(b)
  //
  class A
  {
  public:
    A& operator+ () { return *this; }
    A& operator++() { return *this; }
  };
```
Now we can write code like:
``` cpp
  A a;
  +++++++a; // meaning: ++ ++ ++ + a;
```
which works out to:
``` cpp
  a.operator+().operator++().operator++().operator++()
```
This trick is fairly simple. Creating longer-than-usual sequences of other characters turns out to be a little more challenging, but still possible.

**Abuses of Operators**
The code in Examples 1(a) and 1(b) don't especially abuse the `++` and `+` operators' usual semantics. What we're going to do next, however, really goes beyond anything you'd ever want to see in production code; this is for fun only.

2. Similarly, what is the greatest number of each the following characters that can appear consecutively, without whitespace, outside comments in a valid C++ program?

For this question, we'll create and use the following helper class:
``` cpp
  // Example 2
  //
  class A
  {
  public:
    void operator&&( int ) { }
    void operator<<( int ) { }
    void operator||( int ) { }
  };

  typedef void (A::*F)(int);
```
**a) &**

Answer: Five.

Well, `&&` is easy and `&&&` not too much harder, so let's go right to the next level: Can we create a series of four `&`'s, namely `&&&&`? Well, if we did, they would be interpreted as `"&& &&"`, but expressions like `"a && && b"` are syntactically illegal; we can't have two binary `&&` operators immediately after each other. The trick is to see that we can use the second `&&` as an operator, and make the first `&&` come out as the end of something that's not an operator. With that in mind, it doesn't take too long to see that the first `&&` could be the end of a name, specifically the name of a function, and so all we need is an `operator&&()` that can accept a pointer to some other `operator&&()` as its first parameter:
``` cpp
  void operator&&( F, A ) { }
```
This lets us write:
```
  &A::operator&&&&a; // && &&
```
which means:
``` cpp
  operator&&(&A::operator&&, a);
```
That's the longest even-numbered run of `&`'s we can make, because `&&&&&&` has to be illegal. Why? Because it would mean `&& && &&`, and even after making the first `&&` part of a name again, we can't make the final `&&` the beginning of another name, so we're left with two binary `&&` operators in a row which is illegal.

But can we squeeze in one more `&` by going to an odd number of `&`'s? Yes, indeed. Specifically, `&&&&&` means `&& && &`, we already have a solution for the first part, and with not much thought it's easy to tack on a unary `&`:
``` cpp
  &A::operator&&&&&a; // && && &
```
which uses the builtin operator&&() that can take pointers:
``` cpp
  operator&&(&A::operator&&, &a);
```
**b) <**

**c) |**

Answer for both: Four.

Having seen the solution to 2(a), this one should be easy. To make a series of four, we just use the same trick as before, defining a:
``` cpp
  void operator<<( F, A ) { }
  void operator||( F, A ) { }
```
which lets us write:
``` cpp
  &A::operator<<<<a; // << <<
  &A::operator||||a; // || ||
```
which means:
``` cpp
  operator<<(&A::operator<<, a);
  operator||(&A::operator||, a);
```
That's the longest even-numbered runs we can make, because `<<<<<<` and `||||||` have to be illegal, just as we've already noted that `&&&&&&` has to be illegal. But this time we can't manage an extra `<` or `|` to make a series of five, because there is no unary `<` or `|` operator.

**Bonus Challenge Question**
Here's a bonus question: How many question mark (?) characters can appear in a row in a valid C++ program?

Think about the question before reading on. It's a lot harder than it looks.

----

Do you have an answer?

You might think that the answer must be one, because the only legitimate token in the C++ language that includes a ? is the ternary ?: operator. It's true that that's the only legitimate language feature that includes a ?, but "there are more things in the translator and preprocessor, Horatio, than are dreamt of in your language syntax rules..." In particular, there's more to C++ than just the language, or even just the preprocessor.

For ?, the correct answer is three. For example:

   1???-0:0;

This question is harder than the others in part because it's the only one that doesn't follow the maximal munch rule. The three question marks, ???, are not interpreted as the tokens ?? and ?. Why not? Because ??- happens to be a trigraph, and trigraphs are replaced very early during source code processing \-\- before tokenization begins, even before preprocessor instructions are handled. If you haven't heard about trigraphs before, don't worry; that just means that you don't use an exotic foreign-language keyboard. A trigraph is an alternate way to spell certain unusual source code characters (specifically #, \, ^, [, ], |, {, }, and ~), provided for the benefit of programmers whose keyboards don't happen to have a key for that character.

In this case, long before any tokenization can occur the trigraph ??- is replaced with ~, the one's-complement operator. Therefore the statement above becomes:
   1?~0:0;

which is tokenized as:
   1 ? ~ 0 : 0 ;

and means:
   1 ? (~0) : 0 ;

Trigraphs are a feature inherited from C, and they really are rare in practice. Just to give you an idea of how rare, note that as of this writing several compilers I know of do not have trigraph support turned on by default, and one of those documents it as "enables the undesirable and rarely used ANSI trigraph feature." At least one compiler provides a separate filter to process trigraphs before running the code through the compiler.



# 079 Template Typedef
**Difficulty: 3 / 10**
This GotW exists to answer a recurring question about C++ syntax: When, and how, could you make use of a template typedef?

----

**Problem**
**JG Question**
1. The following code is not legal C++. Why not?
``` cpp
// Example 1

template<class T>
typedef std::map<std::string, T> Registry;

Registry<Employee> employeeRoster;
Registry<void (*)(int)> callbacks;
Registry<ClassFactory*> factories;
```
**Guru Question**
2. Demonstrate a C++ technique that lets you get the same effect.

----

**Solution**
1. The following code is not legal C++. Why not?
``` cpp
// Example 1

template<class T>
typedef std::map<std::string, T> Registry;

Registry<Employee> employeeRoster;
Registry<void (*)(int)> callbacks;
Registry<ClassFactory*> factories;
```
Plain old typedefs are useful beasts; see also GotW #46 for a broader discussion of what typedef is good for. Alas, the code in question above isn't legal for a simple reason: Today's C++ doesn't support template typedefs.

The good news is that it's not hard to simulate this feature:

2. Demonstrate a C++ technique that lets you get the same effect.

According to today's C++ language, typedefs can't be templated directly. What saves the day is that: a) classes can be templated; and b) classes can contain typedefs. Putting this together, we get the following solution, which is the usual technique for working around the absence of template typedefs:
``` cpp
// Example 2

template<class T>
struct Registry
{
  typedef std::map<std::string, T> Type;
};
```
This solution isn't quite the same as having a template typedef, but it's close. The main difference is that we had to give up some of the syntactic niceness, because we now have to mention the name of the helper typedef Type:
``` cpp
Registry<Employee>::Type employeeRoster;
Registry<void (*)(int)>::Type callbacks;
Registry<ClassFactory*>::Type factories;
```
It's Deja Vu All Over Again
As noted above, Example 2 is "the usual workaround." In fact, it's so usual that it's even  used in the standard library itself. All STL-compliant allocators have to provide a template class member called rebind whose entire purpose in life is to simulate a template typedef:
``` cpp
// Example 3

// The type of af is SomeAllocator<float>
SomeAllocator<float> af;

// The type of ad is SomeAllocator<double>
SomeAllocator<float>::rebind<double>::other ad;
```
One way to think of rebind is as a way to get "an allocator kind of sort of just like this one that I've already got, except different" \-\- specifically, an allocator that allocates objects of a different type but behaves the same way in other respects. For an allocator `A<T>`, it's guaranteed that `A<T>::rebind<U>::other` is the same type as `A<U>`.

The reason why allocators have to be able to jump through these hoops is mildly arcane. Consider a `vector<int, SomeAllocator<int> >` called `vec_of_ints`. This `vec_of_ints` will use the provided allocator directly to get memory for its internal buffer of ints, because the standard requires that it must store a real array of ints internally. Fair enough, and simple enough.

But now consider a `list<int, SomeAllocator<int> >` called `list_of_ints`. Unlike vector, list is a node-based container. This means that, unlike `vec_of_ints`, `list_of_ints` doesn't allocate ints directly; rather, it allocates nodes containing an int plus forward and back pointers for navigation to other nodes in the list. But that means that `SomeAllocator<int>`, which allocates ints, isn't at all what `list_of_ints` needs; what `list_of_ints` needs is something that will allocate its own `ListNode<int>s`. Enter rebind: To solve the problem, list_of_ints will internally use rebind to get a related allocator of the type it needs that does what it wants. Assuming its second template parameter is called Allocator, it just asks for an object of type `Allocator::rebind<ListNode<int> >::other`.



# 080 Order, Order!
**Difficulty: 2 / 10**
Programmers learning C++ often come up with interesting misconceptions of what can and can't be done in C++. In this example, contributed by Jan Christiaan van Winkel, a student makes a basic mistake \-\- but one that many compilers allow to pass with no warnings at all.

----

**Problem**
**JG Question**
1. The following code was actually written by a student taking a C++ course, and the compiler the student was using issued no warnings about it. Indeed, several popular compilers issue no warnings for this code. What's wrong with it, and why?
``` cpp
// Example 1
#include <string>
using namespace std;
class A
{
public:
    A( const string& s ) { /* ... */ }   
    string f() { return "hello, world"; }
};

class B : public A
{
public:
    B() : A( s = f() ) {}

private:
    string s;
};

int main()
{
    B b;
}
```
**Guru Question**
2. When you create a C++ object of class type, in what order are its various parts initialized? Be as specific and complete as you can.

----

**Solution**
1. [...]  What's wrong with [this code], and why?
``` cpp
  // ...
    B() : A( s = f() ) {}
  // ...
```
This line harbors a couple of related problems, both associated with object lifetime and the use of objects before they exist. Note that the expression "s = f()" appears as the argument to the A base subobject constructor, and hence will be executed before the A base subobject (or, for that matter, any part of the B object) is constructed.

First, this line of code tries to use the A base subobject before it exists. This particular compiler did not flag the (ab)use of A::f(), in that the member function f() is being called on an A subobject that hasn't yet been constructed. Granted, the compiler is not required to diagnose such an error, but this is the kind of thing standards folks call "a quality of implementation issue" \-\- something that a compiler is not required to do, but that better compilers could be nice enough to do.

Second, this line then goes on and merrily tries to use the s member subobject before it exists, namely by calling the member function operator=() is being called on a string member subobject that hasn't yet been constructed.

2. When you create a C++ object of class type, in what order are its various parts initialized? Be as specific and complete as you can.

The following set of rules is applied recursively:
- First, the most derived class's constructor calls the constructors of the virtual base class subobjects. Virtual base classes are initialized in depth-first, left-to-right order.
- Next, direct base class subobjects are constructed in the order they are declared in the class definition.
- Next, (nonstatic) member subobjects are constructed, in the order they were declared in the class definition.
- Finally, the body of the constructor is executed.

For example, consider the following code. Whether the inheritance is public, protected, or private doesn't affect initialization order, so I'm showing all inheritance as public.
``` cpp
  // Example 2
  //
  class B1 { };
  class V1 : public B1 { };
  class D1 : virtual public V1 { };

  class B2 { };
  class B3 { };
  class V2 : public B1, public B2 { };
  class D2 : public B3, virtual public V2 { };

  class M1 { };
  class M2 { };

  class X : public D1, public D2 { M1 m1_; M2 m2_; };
```
The inheritance hierarchy looks like this:

  B1          B1    B2
   |            |       /
   |            |    /
   |            | /
  V1           V2    B3
   |            |       /
   |v        v|    /
   |            | /
  D1           D2
    \             /
       \       /
          \ /
           X

The initialization order for a X object in Example 2 is as follows, where each constructor call shown represents the execution of the body of that constructor:
``` cpp
    // first, construct the virtual bases:
    // construct V1:
    B1::B1()
    V1::V1()
    
    // construct V2:
    B1::B1()
    B2::B2()
    V2::V2()
    
    // next, construct the nonvirtual bases:
    // construct D1:
    D1::D1()
    
    // construct D2:
    B3::B3()
    D2::D2()
    
    // next, construct the members:
    M1::M1()
    M2::M2()
    
    // finally, construct X itself:
    X::X()
```
This should make it clear why in Example 1 it's illegal to call either `A::f()` or the s member subobject's construct.

**A(nother) Word About Inheritance**
Of course, although the main point of this issue of GotW was to understand the order in which objects are constructed (and, in reverse order, destroyed), it doesn't hurt to repeated a tangentially related guideline:

Guideline: Avoid overusing inheritance.

Except for friendship, inheritance is the strongest relationship that can be expressed in C++, and should be only be used when it's really necessary. For more details, see also:
- [GotW #60: Exception-Safe Class Design, Part 2: Inheritance](http://www.gotw.ca/gotw/060.htm)
- [Sutter's Mill #6: Uses and Abuses of Inheritance, Part 1](http://www.gotw.ca/publications/mill06.htm)
- [Sutter's Mill #7: Uses and Abuses of Inheritance, Part 2](http://www.gotw.ca/publications/mill07.htm)
- The updated material in [Exceptional C++](http://www.gotw.ca/publications/xc++.htm) and [More Exceptional C++](http://www.gotw.ca/publications/mxc++.htm)


# 081 Constant Optimization?
**Difficulty: 3 / 10**
Does const-correctness help the compiler to optimize code? Most programmers' reaction is that, yes, it probably does. Which brings us to the interesting thing...

----

**Problem**
**JG Question**
1. Consider the following code:
``` cpp
  // Example 1
  //
  const Y& f( const X& x )
  {
    // ... do something with x and find a Y object ...
    return someY;
  }
```
Does declaring the parameter and/or the return value as const help the compiler to generate more optimal code or otherwise improve its code generation? Why or why not?

**Guru Questions**
2. In general, explain why or why not the presence or absence of const can help the compiler to enhance the code it generates.

3. Consider the following code:
``` cpp
  // Example 3
  //
  void f( const Z z )
  {
    // ...
  }
```
The questions:

a) Under what circumstances, and for what kinds of class Z, could this particular const qualification help to generate different and better code? Discuss.

b) For the kinds of circumstances mentioned in (a), are we talking about a compiler optimization or some other kind of optimization? Explain.

c) What's a better way to get the same effect?

----

**Solution**
1. Consider the following code:
``` cpp
// Example 1
//
const Y& f( const X& x )
{
  // ... do something with x and find a Y object ...
  return someY;
}
```
Does declaring the parameter and/or the return value as const help the compiler to generate more optimal code or otherwise improve its code generation? Why or why not?

In short, no, it probably doesn't.

What could the compiler do better? Could it avoid a copy of the parameter or the return value? No, because the parameter is already passed by reference, and the return is already by reference. Could it put a copy of x or someY into read-only memory? No, for both x and someY live outside its scope and come from and/or are given to the outside world. Even if someY is dynamically allocated on the fly within `f()` itself, it and its ownership are given up to the caller.

But what about possible optimizations of code that appears inside the body of `f()`? Because of the const, could the compiler somehow improve the code it generates for the body of `f()`? This leads into the second and more general question:

2. In general, explain why or why not the presence or absence of const can help the compiler to enhance the code it generates.

Referring again to Example 1, of course the usual reason that the parameter's constness doesn't usually let the compiler do fancy things still applies here: Even when you call a const member function, the compiler can't assume that the bits of object x or object someY won't be changed. Further, there are additional problems (unless the compiler performs global optimization): The compiler also may not know for sure that no other code might have a non-const reference that aliases the same object as x and/or someY, and whether any such non-const references to the same object might get used incidentally during the execution of `f()`; and the compiler may not even know whether the real objects, to which x and someY are merely references, were actually declared const in the first place.

Just because x and y are declared const doesn't necessarily mean that their bits are physically const. Why not? Because either class may have mutable members \-\- or their moral equivalent, namely const_casts inside member functions. Indeed, the code inside `f()` itself might perform `const_casts` or `C-style` casts that turn the const declarations into lies.

There is one case where saying "const" can really mean something, and that is when objects are made const at the point they are defined. In that case, the compiler can often successfully put such "really const" objects into read-only memory, especially if they are PODs whose memory images can be created at compile time and therefore can be stored right inside the program's executable image itself. Such objects are colloquially called "ROM-able."

3. Consider the following code:
``` cpp
// Example 3
//
void f( const Z z )
{
  // ...
}
```
The questions:

a) Under what circumstances, and for what kinds of class Z, could this particular const qualification help to generate different and better code? Discuss.

The short answer is: Yes and no.

First, the Yes part: Because the compiler knows that z truly is a const object, it could perform some useful optimizations even without global analysis. For example, if the body of `f()` contains a call like `g(&z)`, the compiler can be sure that the non-mutable parts of z do not change during the call to `g()`.

Other than that, however, writing const in Example 3 is not an optimization for most classes Z \-\- and in those cases where it is an optimization, it's not a compiler code generation optimization.

For compiler code generation, the question mostly comes down to whether the compiler could elide copy construction, or could put z into read-only memory. That is, it would be nice if we knew that z was really untouched by what goes on inside `f()`, because theoretically that would mean we might be able to just directly use the outside object that the caller passed into this function instead of making a copy of it, or else we could make a copy but put that copy into read-only memory if that happens to be faster or otherwise more desirable.

In general, the compiler can't use the parameter's constness to elide the copy construction or assume bitwise constness. As noted above, too many things can go wrong. In particular, Z might have mutable members, or someone somewhere (in `f()` itself, in some other function, or in some directly or indirectly called Z member function) might perform const_casts or other chicanery.

There is one case where the compiler might be able to generate better code. If:
- the definitions for Z's copy constructor, and for all of Z's functions that are used directly or indirectly in the body of f(), are visible at this point;
- those functions are all pretty simple and free of side effects; and	
- the compiler has an aggressive optimizer

then the compiler can be sure of what exactly is going on every step of the way, and might choose to elide the copy under the "as if" rule, which says that a compiler is allowed to do something different from what the standard technically says it must as long as a conforming program can't tell the difference.

As an aside, here it's worth mentioning one small red herring: Some people may say there's another case where the compiler could generate better code with const, namely if the compiler performs global optimization. The thing is that that sentence is true even if you remove the "with const." Never mind that global optimization is rare and very expensive; the real issue here is that global optimization makes use of all knowledge about how an object is actually used, and therefore will do the same thing whether the object is actually declared const or not \-\- it makes decisions based on what you really do, not on what you promised to do, and so it doesn't matter if the two happen to be the same thing. If you're getting true global optimization anyway, then promising constness definitely doesn't help the optimizer do its job any better in this respect.

Note that above I said that "writing const in Example 3 is not an optimization for most classes Z," and "for compiler code generation." There are, however, more optimizations that are possible than are dreamt of in your compiler's optimizer! And indeed const can enable some real optimizations:
   b) For the kinds of circumstances mentioned in (a), are we talking about a compiler optimization or some other kind of optimization? Explain.

In particular, there are also programmatic optimizations, where the author of Z can intelligently choose to do things a different way for const objects.

As a case in point, let's say that Example 3's Z is a handle/body class, such as a String class that uses reference counting to perform lazy copying:
``` cpp
// Example 4
void f( const String s ) {
    // ...
    s[4]; // or use iterators
    // ...
}
```

Such a String, knowing that calling `operator[]()` on a const String shouldn't be able to change the contents of the string, might decide to provide a const overload for `operator[]()` that returns a `char` by value instead of a `char&`:
``` cpp
class String {
    // ...

public:
    char  operator[]( size_t ) const;
    char& operator[]( size_t );
    // ...
}
```

Similarly, String could also provide const_iterators whose `operator*()` returns a `char` by value instead of a `char&`:

If so, and if you happen to use the smartened-up `operator[]()` or iterators, and you declare the pass-by-value parameter as const, then \-\- wonder of wonders! \-\- the String can, with no further help from you, automagically and wholesomely optimize your code by avoiding a deep copy...

c) What's a better way to get the same effect?

...but you get all of this and more by simply writing
``` cpp
// Example 5: Just do it \-\- better than Example 3
void f( const Z& z ) {
    // ...
}
```
and it works whether the object is handle/body or reference-counted or not, so just do that!

Guideline: Avoid passing objects by const value. Prefer passing them by const reference instead, unless they're cheap-to-copy objects like ints.

It's a common belief that const-correctness helps compilers generate tighter code. Const is indeed a Good Thing, but the point of this issue of GotW is that const is mainly for humans, rather than for compilers and optimizers. When it comes to writing safe code, const is a great tool that lets programmers write safer code with compiler checking and enforcement. Even when it comes to optimization, const is still principally useful as a tool that lets human class designers better implement handcrafted optimizations, and less so as a tag for omniscient compilers to automatically generate better code.

**Acknowledgments**
Thanks to Kevlin Henney for suggesting this topic and some of the cases, and to Bill Wade for an amplification to 3(a)..



# 082 Exception Safety and Exception Specifications: Are They Worth It?
**Difficulty: 8 / 10**
Is it worth the effort to write exception-safe code? Are exception specifications worthwhile? It may surprise you that these are still disputed and debated points, and ones where even experts may sometimes disagree.

----

**Problem**
**JG Questions**
1. Recap: Briefly define the Abrahams exception safety guarantees (basic, strong, and nothrow).

2. What happens when an exception specification is violated? Why? Discuss the basic rationale for this C++ feature.

**Guru Questions**
3. When is it worth it to write code that meets:
   a. the basic guarantee?
   b. the strong guarantee?
   c. the nothrow guarantee?

4. When is it worth it to write exception specifications on functions? Why would you choose to write one, or why not?

----

**Solution**
**JG Questions**
1. Recap: Briefly define the Abrahams exception safety guarantees (basic, strong, and nothrow).

The basic guarantee is that failed operations may alter program state, but no leaks occur and affected objects/modules are still destructible and usable, in a consistent (but not necessarily predictable) state.

The strong guarantee involves transactional commit/rollback semantics: failed operations guarantee program state is unchanged with respect to the objects operated upon. This means no side effects that affect the objects, including the validity or contents of related helper objects such as iterators pointing into containers being manipulated.

The nothrow guarantee means that failed operations will not happen. The operation will not throw an exception.

2. What happens when an exception specification is violated? Why? Discuss the basic rationale for this C++ feature.

The idea of exception specifications is to do a run-time check that guarantees that only exceptions of certain types will be emitted from a function (or that none will be emitted at all). For example, the following function's exception specification guarantees that f() will emit only exceptions of type A or B:
``` cpp
int f() throw( A, B );
```
If an exception would be emitted that's not on the "invited-guests" list, the function unexpected() will be called. For example:
``` cpp
int f() throw( A, B )
{
  throw C(); // will call unexpected()
}
```
You can register your own handler for the unexpected-exception case by using the standard set_unexpected() function. Your replacement handler must take no parameters and it must have a void return type. For example:
``` cpp
void MyUnexpectedHandler() { /*...*/ }

std::set_unexpected( &MyUnexpectedHandler );
```
The remaining question is, what can your unexpected handler do? The one thing it can't do is return via a usual function return. There are two things it may do:

1. It could decide to translate the exception into something that's allowed by that exception-specification, by throwing its own exception that does satisfy the exception-specification list that caused it to be called, and then stack unwinding continues from where it left off.

2. It could call terminate(). (The terminate() function can also be replaced, but must always end the program.)

**Guru Questions**
3. When is it worth it to write code that meets:

a) the basic guarantee?

b) the strong guarantee?

c) the nothrow guarantee?

It is always worth it to write code that meets at least one of these guarantees. There are several good reasons:

**1. Exceptions happen.** (To paraphrase a popular saying.)

They just do. The standard library emits them. The language emits them. We have to code for them. Fortunately, it's not that big a deal, because we now know how to do it. It does require adopting a few habits, however, and following them diligently \-\- but then so did learning to program with error codes.

The big thorny problem is, as it ever was, the general issue of error handling. The detail of how to report errors, using return codes or exceptions, is almost entirely a syntactic detail where the main differences are in the semantics of how the reporting is done, and so each approach requires its own style.

**2. Writing exception-safe code is good for you.**

Exception-safe code and good code go hand in hand. The same techniques that have been popularized to help us write exception-safe code are, pretty much without exception, things we usually ought to be doing anyway. That is, exception-safety techniques are good for your code in and of themselves, even if exception safety weren't a consideration. 

To see this in action, consider the major techniques I and others have written about to make exception safety easier:

Use “resource acquisition is initialization” (RAII) to manage resource ownership. Using resource-owning objects like Lock classes and auto_ptrs is just a good idea in general. It should come as no surprise that among their many benefits we should also find "exception safety." How many times have you seen a function (here we're talking about someone else's function, of course, not something you wrote!) where one of the code branches that leads to an early return fails to do some cleanup, because cleanup wasn't being managed automatically using RAII? 

Use “do all the work off to the side, then commit using nonthrowing operations only” to avoid changing internal state until you’re sure the whole operation will succeed. Such transactional programming is clearer, cleaner, and safer even with error codes. How many times have you seen a function (and naturally here again we're talking about someone else's function, of course, not something you wrote!) where one of the code branches that leads to an early return fails to preserve the object's state, because some fiddling with internal state had already happened before a later operation failed?

Prefer “one class (or function), one responsibility.” Functions that have multiple effects, such as the Stack::Pop() and EvaluateSalaryAndReturnName() functions described in Items 10 and 18 of Exceptional C++ [1], are difficult to make strongly exception-safe. Many exception safety problems can be made much simpler, or eliminated without conscious thought, simply by following the "one function, one responsibility" guideline. And that guideline long predates our knowledge that it happens to also apply to exception safety; it's just a good idea in and of itself.

Doing these things is just plain good for you.

Having said that, then, which guarantee should we use when? In brief, here's the guideline followed by the C++ standard library, and one that you can profitably apply to your own code:

Guideline:
   **A function should always support the strictest guarantee that it can support without penalizing users who don't need it.**

So if your function can support the nothrow guarantee without penalizing some users, it should do so. Note that a handful of key functions, such as destructors and deallocation functions, simply must be nothrow-safe operations because otherwise it's impossible to reliably and safely perform cleanup.

Otherwise, if your function can support the strong guarantee without penalizing some users, it should do so. Note that vector::insert() is an example of a function that does not support the strong guarantee in general because doing so would force us to make a full copy of the vector's contents every time we insert an element, and not all programs care so much about the strong guarantee that they're willing to incur that much overhead. (Those programs that do can "wrap" `vector::insert()` with the strong guarantee themselves, trivially: take a copy of the vector, perform the insert on the copy, and once it's successful `swap()` with the original vector and you're done.)

Otherwise, your function should support the basic guarantee.

For more information about some of the above concepts, such as what a nonthrowing swap() is all about or why destructors should never emit exceptions, see further reading in Exceptional C++ [1] and More Exceptional C++ [2].

4. When is it worth it to write exception specifications on functions? Why would you choose to write one, or why not?

In brief, don't bother. Even experts don't bother.

Slightly less briefly, the major issues are:

Exception specifications can cause surprising performance hits, for example if the compiler turns off inlining for functions with exception specifications.

A runtime unexpected() error is not always what you want to have happen for the kinds of mistakes that exception specifications are meant to catch.

You generally can't write useful exception specifications for function templates anyway because you generally can't tell what the types they operate on might throw.

For more, see for example the Boost exception specification rationale available via [http://www.gotw.ca/publications/xc++s/boost_es.htm](http://www.gotw.ca/publications/xc++s/boost_es.htm) (it summarizes to "Don't!").

**References**
[1] H. Sutter. [Exceptional C++](http://www.gotw.ca/publications/xc++.htm) (Addison-Wesley, 2000).
[2] H. Sutter. [More Exceptional C++](http://www.gotw.ca/publications/mxc++.htm) (Addison-Wesley, 2002).



# 083 Style Case Study #2: Generic Callbacks
**Difficulty: 5 / 10**
Part of the allure of generic code is its usability and reusability in as many kinds of situations as reasonably possible. How can the simple facility presented in the cited article be stylistically improved, and how can it be made more useful than it is and really qualify as generic and widely-usable code?

----

**Problem**
**JG Question**
1. What qualities are desirable in designing and writing generic facilities? Explain.

**Guru Question**
2. The following code presents an interesting and genuinely useful idiom for wrapping callback functions. For a more detailed explanation, see the original article.[1]

Critique this code and identify:

a) Stylistic choices that could be improved to make the design better for more idiomatic C++ usage.

b) Mechanical limitations that restrict the usefulness of the facility.
``` cpp
template < class T, void (T::*F)() >
class callback
{
public:
    callback(T& t) : object(t) {} // assign actual object to T
    void execute() {(object.*F)();}// launch callback function
private:
    T& object;
};
```
(For an idea of the kinds of things I'm looking for, see also Style Case Study #1.)


**Solution**
**JG Question**
1. What qualities are desirable in designing and writing generic facilities? Explain.

Generic code should above all be usable. That doesn't mean it has to include all options up to and including the kitchen sink. What it does mean is that generic code ought to make a reasonable and balanced effort to avoid at least three things:

**a) Avoid undue type restrictions.**

For example, are you writing a generic container? Then it's perfectly reasonable to require that the contained type have, say, a copy constructor and a nonthrowing destructor. But what about a default constructor, or an assignment operator? Many useful types that users might want to put into our container don't have a default constructor, and if our container uses it then we've eliminated such a type from being used with our container. That's not very generic. (For a complete example, see Item 15 of Exceptional C++. [2])

**b) Avoid undue functional restrictions.**

If you're writing a facility that does X and Y, then what if some user wants to do Z, and Z isn't so much different from Y? Sometimes you'll want to make your facility flexible enough to support Z; sometimes you won't. Part of good generic design is choosing the ways and means by which your facility can be customized or extended. That this is important in generic design should hardly be a surprise, though, because the same principle applies to object-oriented class design.

Policy-based design is one of several important techniques that allow "pluggable" behavior with generic code. For examples of policy-based design, see any of several chapters in Alexandrescu's Modern C++ Design [3]; the SmartPtr and Singleton chapters are a good place to start. 

This leads to a related issue:

**c) Avoid unduly monolithic designs.**

I'll break out discussion of this third item to a separate GotW #84. The issue of "unduly monolithic designs" doesn't arise as directly in our style example under consideration below, and it deserves some dedicated consideration in its own right, hence it gets its own article.

Above, you'll note the recurring word "undue." That means just what it says: Good judgment is needed when deciding where to draw the line between failing to be sufficiently generic (the "I'm sure nobody would want to use it with anything but char" syndrome) on the one hand, and overengineering (the "what if someday some wants to use this toaster-oven LED display routine to control the booster cutoff on an interplanetary spacecraft?" misguided fantasy) on the other.

**Guru Question**
2. The following code presents an interesting and genuinely useful idiom for wrapping callback functions. For a more detailed explanation, see the original article.[1]

Here again is the code:
``` cpp
template < class T, void (T::*F)() >
class callback
{
public:
    callback(T& t) : object(t) {} // assign actual object to T
    void execute() {(object.*F)();}// launch callback function
private:
    T& object;
};
```
Now, really, how many ways are there to go wrong in a simple class with just two one-liner member functions? Well, as it turns out, its extreme simplicity is part of the problem. This class template doesn't need to be heavyweight, not at all, but it could stand to be a little less lightweight.

Critique this code and identify:
   a) Stylistic choices that could be improved to make the design better for more idiomatic C++ usage.

How many did you spot? Here's what I came up with:


**The constructor should be explicit.**

The author probably didn't mean to provide an implicit conversion from T to callback<T>. Well-behaved classes avoid creating the potential for such problems for their users. So what we really want is more like this:
``` cpp
    explicit callback(T& t) : object(t) {}
                                  // assign actual object to T
```
While we're already looking at this particular line, there's another stylistic issue that's not about the design per se, but about the description:


**(Nit) The comment is wrong.**

The word "assign" in the comment is incorrect and so somewhat misleading. More correctly, in the constructor we're "binding" a T object to the reference, and by extension to the callback object. Also, after many rereadings I'm still not sure what the "to T" part means. So better still would be "bind actual object."
``` cpp
    explicit callback(T& t) : object(t) {} // bind actual object
```
But then all that comment is saying is what the code already says, which is faintly ridiculous and a stellar example of a useless comment, so best of all would be:
``` cpp
    explicit callback(T& t) : object(t) {}
```

**The execute() function should be const.**

The execute() function isn't doing anything to the callback<T> object's state, after all! This is a "back to basics" issue: Const-correctness may be an oldie, but it's a goodie. The value of const-correctness has been known in C and C++ since at least the early 1980s, and that value didn't just evaporate when we clicked over to the new millennium and started writing lots of templates.
``` cpp
    void execute() const {(object.*F)();}
                               // launch callback function
```
While we're already beating on the poor execute() function, there's an arguably more serious idiomatic problem:


**(Idiom) And the execute() function should be spelled "operator()()".**

In C++, it's idiomatic to use the function-call operator for executing a function-style operation. Indeed, then the comment, already somewhat redundant, becomes completely so and can be removed without harm because now our code is already idiomatically commenting itself. To wit:
``` cpp
    void operator()() const {(object.*F)();}
```
"But," you might be wondering, "if we provide the function-call operator, then isn't this some kind of function object?" That's an excellent point, which leads us to observe that, as a function object, maybe callback instances ought to be adaptable too:


**Pitfall: (Idiom) Should this callback should be derived from std::unary_function?**

See Item 40 in Meyers' Effective STL [4] for a more detailed discussion about adaptability and why it's a Good Thing in general. Alas, here, there are two excellent reasons why callback should not be derived from std::unary_function, at least not yet:
- It's not a unary function. It takes no parameter, and unary functions take a parameter. (No, "void" doesn't count.)
- Deriving from std::unary_function isn't going to be extensible anyway. Later on, we're going to see that callback perhaps ought to work with other kinds of function signatures too, and depending on the number of parameters involved, there may well be no standard base class to derive from. For example, if we supported callback functions with three parameters, we have no std::ternary_function to derive from.

Deriving from std::unary_function or std::binary_function is a convenient way to give callback a handful of important typedefs that binders and similar facilities often rely upon, but it only matters if you're going to use the function objects with those facilities. Because of the nature of these callbacks and how they're intended to be used, it's unlikely that this will be needed. (If in the future it turns out that they ought to be usable this way for the common one- and two-parameter cases, then the one- and two-parameter versions we'll mention later can be derived from std::unary_function and std::binary_function, respectively.)

   b) Mechanical limitations that restrict the usefulness of the facility.

**Consider making the callback function a normal parameter, not a template parameter.**

Non-type template parameters are rare in part because there's rarely much benefit in so strictly fixing a type at compile time. That is, we could instead have:
``` cpp
template < class T >
class callback
{
public:
    typedef void (T::*Func)();

    callback( T& t, Func func ) : object(t), f(func) { }
    void operator()() { (object.*f)(); }

private:
    T& object;
    Func f;
};
```
Now the function to be used can vary at runtime, and it would be simple to add a member function that allowed the user to change the function that an existing callback object was bound to, something not possible in previous versions of the code.

Guideline:
It's usually a good idea to prefer making non-type parameters into normal function parameters, unless they really need to be template parameters.


**Containerization**

If a program wants to keep one callback object for later use, it's likely to want to keep more of them. What if it wants to put the callback objects into a container, like a vector or a list? Currently that's not possible, because callback objects aren't assignable \-\- they don't support operator=(). Why not? Because they contain a reference, and once that reference is bound during construction it can never be rebound to something else.

Pointers, however, have no such compunction, and are quite happy to point at whatever you'd ask them to. In this case it's perfectly safe for callback instead to store a pointer, not a reference, to the object it's to be called on, and then to use the default compiler-generated copy constructor and copy assignment operator:
``` cpp
template < class T >
class callback
{
public:
    typedef void (T::*Func)();

    callback( T& t, Func func ) : object(&t), f(func) { }
    void operator()() { (object->*f)(); }

private:
    T* object;
    Func f;
};
```
Now it's possible to have, for example, a list< callback< Widget, &Widget::SomeFunc > >.

"But wait," you might wonder at this point, "if I could have that kind of a list, why couldn't I have a list of arbitrary kinds of callbacks of various types, so that I can remember them all, and go execute them all when I want to?" Indeed, you can, if you add a base class:


**Provide a common base class for callback types.**

If we want to let users have a list<callbackbase*>, we can do it by providing just such a base class, which by default happens to do nothing in its operator()():
``` cpp
class callbackbase
{
public:
    virtual void operator()() const { };
    virtual ~callbackbase() = 0;
};

callbackbase::~callbackbase() { }

template < class T >
class callback : public callbackbase
{
public:
    typedef void (T::*Func)();

    callback( T& t, Func func ) : object(&t), f(func) { }
    void operator()() const { (object->*f)(); }

private:
    T* object;
    Func f;
};
```
Now anyone who wants to can keep a `list<callbackbase*>` and polymorphically invoke `operator()()` on its elements. Of course, a `list<boost::shared_ptr<callback> >` would be even better.

Note that adding a base class is a tradeoff, but only a small one: We've added the overhead of a second indirection, namely a virtual function call, when the callback is triggered through the base interface. But that overhead only actually manifests when you use the base interface. Code that doesn't need the base interface doesn't pay for it.


**(Idiom, Tradeoff) There could be a helper make_callback function to aid in type deduction.

After a while, users may get tired of explicitly specifying template parameters for temporary objects:
``` cpp
    list< callback< Widget > > l;
    l.push_back( callback<Widget>( w, &Widget::SomeFunc ) );
```
Why write Widget twice? Doesn't the compiler know? Well, no, it doesn't, but we can help it to know. in contexts where only a temporary object like this is needed. Instead, we could provide a helper so that they need only type:
``` cpp
    list< callback< Widget > > l;
    l.push_back( make_callback( w, &Widget::SomeFunc ) );
```
This make_callback works just like the standard make_pair(). The missing make_callback() helper should be a function template, because that's the only kind of template for which compiler can deduce types. Here's what the helper looks like:
``` cpp
template<typename T >
callback<T> make_callback( T& t, void (T::*f) () )
{
    return callback<T>( t, f );
}
```

**(Tradeoff) Add support for other callback signatures.**

I've left the biggest job for last. As the Bard might have put it, "There are more function signatures in heaven and earth, Horatio, than are dreamt of in your `void (T::*F)()`!"

If enforcing that signature for callback functions is sufficient, then by all means stop right there. There's no sense in complicating a design if we don't need to  \-\- for complicate it we will, if we want to allow for more function signatures!

I won't write out all the code, because it's significantly tedious. (If you really want to see code this repetitive, or are having trouble with insomnia, see books and articles like [3] for similar examples.) What I will do is briefly sketch the main things you'd have to support, and how you'd have to support them:

First, what about const member functions? The easiest way to deal with this one is to provide a parallel callback that uses the const signature type, and in that version remember to take and hold the T by reference or pointer to const.

Second, what about non-void return types? The simplest way to allow the return type to vary is by adding another template parameter.

Third, what about callback functions that take parameters? Again, add template parameters, remember to add parallel function parameters to `operator()()`, and stir well. Remember to add a new template to handle each potential number of callback arguments.

Alas, the code explodes, and you have to do things like set artificial limits on the number of function parameters that callback supports. Perhaps in a future C++0x language we'll have features like template "varargs" that will help to deal with this, but not today.


**Summary**
Putting it all together, and making some purely stylistic adjustments like using "typename" consistently and naming conventions and whitespace conventions that I happen to like better, here's what we get:
``` cpp
class CallbackBase
{
public:
    virtual void operator()() const { };
    virtual ~CallbackBase() = 0;
};

CallbackBase::~CallbackBase() { }

template<typename T>
class Callback : public CallbackBase
{
public:
    typedef void (T::*F)();

    Callback( T& t, F f ) : t_(&t), f_(f) { }
    void operator()() const { (t_->*f_)(); }

private:
    T* t_;
    F  f_;
};

template<typename T>
Callback<T> make_callback( T& t, void (T::*f) () )
{
    return Callback<T>( t, f );
}
```

**References**
[1] D. Kalev. ["Designing a Generic Callback Dispatcher"](http://www.gotw.ca/publications/xc++s/dk_callbacks.htm) (DevX).
[2] H. Sutter, [Exceptional C++](http://www.gotw.ca/publications/xc++.htm) (Addison-Wesley, 2000).
[3] A. Alexandrescu. Modern C++ Design (Addison-Wesley, 2001).
[4] S. Meyers. Effective STL (Addison-Wesley, 2001).



# 084 Monoliths "Unstrung"
**Difficulty: 3 / 10**
"All for one, and one for all" may work for Musketeers, but it doesn't work nearly as well for class designers. Here's an example that is not altogether exemplary, and it illustrates just how badly you can go wrong when design turns into overdesign. The example is, unfortunately, taken from a standard library near you...


**Problem**
**JG Question**
1. What is a "monolithic" class, and why is it bad? Explain.

Guru Questions
2. List all the member functions of std::basic_string.

3. Which ones should be members, and which should not? Why?


**Solution**
**Avoid Unduly Monolithic Designs**
1. What is a "monolithic" class, and why is it bad? Explain.

The word "monolithic" is used to describe software that is a single big, indivisible piece, like a monolith. The word "monolith" comes from the words "mono" (one) and "lith" (stone) whose vivid image of a single solid and heavy stone well illustrates the heavyweight and indivisible nature of such code.

A single heavyweight facility that thinks it does everything is often a dead end. After all, often a single big heavyweight facility doesn't really do more \-\- it often does less, because the more complex it is, the narrower its application and relevance is likely to be.

In particular, a class might fall into the monolith trap by trying to offer its functionality through member functions instead of nonmember functions, even when nonmember nonfriend functions would be possible and at least as good. This has at least two drawbacks:
- (Major) It isolates potentially independent functionality inside a class. The operation in question might otherwise be nice to use with other types, but because it's hardwired into a particular class that won't be possible, whereas if it were exposed as a nonmember function template it could be more widely usable.
- (Minor) It can discourage extending the class with new functionality. "But wait!" someone might say. "It doesn't matter whether the class's existing interface leans toward member or nonmember functions, because I can equally well extend it with my own new nonmember functions either way!" That's technically true, and misses the semantic point: If the class's built-in functionality is offered mainly (or only) via member functions and therefore present that as the class's natural idiom, then the extended nonmember functions cannot follow the original natural idiom and will always remain visibly second-class johnny-come-latelies. That the class presents its functions as members is a semantic statement to its users, who will be used to the member syntax that extenders of the class cannot use. (Do we really want to go out of our way to make our users commonly have to look up which functions are members and which ones aren't? That already happens often enough even in better designs.)

Where it's practical, break generic components down into pieces.

Guideline: Prefer "one class (or function), one responsibility."

Guideline: Where possible, prefer writing functions as nonmember nonfriends.

The rest of this article will go further toward justifying the latter guideline.

 

**The Basics of Strings**
2. Name all the member functions of std::basic_string.

It's, ahem, a fairly long list. Counting constructors, there are no fewer than 103 member functions. Really. If that's not monolithic, it's hard to imagine what would be.

Imagine yourself in an underground cavern, at the edge of a subterranean lake. There is a submerged channel that leads to the next air-filled cavern. You prepare to dive into the black water and swim through the flooded tunnel...

Take a deep breath, and repeat after ISO/IEC 14882:1998(E) [1]:
``` cpp
namespace std {
    template<class charT, class traits = char_traits<charT>,
  	   class Allocator = allocator<charT> >
    class basic_string {
        // ... some typedefs ...
        
        explicit basic_string(const Allocator& a = Allocator());
        basic_string(const basic_string& str, size_type pos = 0, size_type n = npos,
                     const Allocator& a = Allocator());
        basic_string(const charT* s, size_type n,
                     const Allocator& a = Allocator());
        basic_string(const charT* s,
                     const Allocator& a = Allocator());
        basic_string(size_type n, charT c,
                     const Allocator& a = Allocator());
					 
        template<class InputIterator>
        basic_string(InputIterator begin, InputIterator end,
        	         const Allocator& a = Allocator());
					 
        ~  basic_string();
        basic_string& operator=(const basic_string& str);
        basic_string& operator=(const charT* s);
        basic_string& operator=(charT c);
	    
        iterator       begin();
        const_iterator begin() const;
        iterator       end();
        const_iterator end() const;
        
        reverse_iterator       rbegin();
        const_reverse_iterator rbegin() const;
        reverse_iterator       rend();
        const_reverse_iterator rend() const;
    
        size_type size() const;
        size_type length() const;
        size_type max_size() const;
        void resize(size_type n, charT c);
        void resize(size_type n);
        size_type capacity() const;
        void reserve(size_type res_arg = 0);
        void clear();
        bool empty() const;
        
        const_reference operator[](size_type pos) const;
        reference       operator[](size_type pos);
        const_reference at(size_type n) const;
        reference       at(size_type n);
        
        basic_string& operator+=(const basic_string& str);
        basic_string& operator+=(const charT* s);
        basic_string& operator+=(charT c);
        basic_string& append(const basic_string& str);
        basic_string& append(const basic_string& str,
                             size_type pos, size_type n);
        basic_string& append(const charT* s, size_type n);
        basic_string& append(const charT* s);
        basic_string& append(size_type n, charT c);
		
        template<class InputIterator>
        basic_string& append(InputIterator first, InputIterator last);
		
        void push_back(const charT);
        
        basic_string& assign(const basic_string&);
        basic_string& assign(const basic_string& str,
                             size_type pos, size_type n);
        basic_string& assign(const charT* s, size_type n);
        basic_string& assign(const charT* s);
        basic_string& assign(size_type n, charT c);
		
        template<class InputIterator>
        basic_string& assign(InputIterator first, InputIterator last);
        
        basic_string& insert(size_type pos1, const basic_string& str);
        basic_string& insert(size_type pos1, const basic_string& str,
                             size_type pos2, size_type n);
        basic_string& insert(size_type pos, const charT* s, size_type n);
        basic_string& insert(size_type pos, const charT* s);
        basic_string& insert(size_type pos, size_type n, charT c);
        iterator insert(iterator p, charT c);
        void insert(iterator p, size_type n, charT c);
		
        template<class InputIterator>
        void insert(iterator p, InputIterator first, InputIterator last);
```
You break surface at a small air pocket and gasp. Don't give up \-\- we're halfway there! Another deep breath, and:
``` cpp
        basic_string& erase(size_type pos = 0,
                            size_type n = npos);
        iterator erase(iterator position);
        iterator erase(iterator first, iterator last);
        
        basic_string& replace(size_type pos1, size_type n1,
        		              const basic_string& str);
        basic_string& replace(size_type pos1, size_type n1, const basic_string& str,
        		              size_type pos2, size_type n2);
        basic_string& replace(size_type pos, size_type n1, const charT* s, size_type n2);
        basic_string& replace(size_type pos, size_type n1, const charT* s);
        basic_string& replace(size_type pos, size_type n1, size_type n2, charT c);
        
        basic_string& replace(iterator i1, iterator i2, const basic_string& str);
        basic_string& replace(iterator i1, iterator i2, const charT* s, size_type n);
        basic_string& replace(iterator i1, iterator i2, const charT* s);
        basic_string& replace(iterator i1, iterator i2, size_type n, charT c);
	    
        template<class InputIterator>
        basic_string& replace(iterator i1, iterator i2,
        		              InputIterator j1, InputIterator j2);
        
        size_type copy(charT* s, size_type n,
                       size_type pos = 0) const;
        void swap(basic_string<charT,traits,Allocator>&);
        
        const charT* c_str() const;         //  explicit
        const charT* data() const;
        allocator_type get_allocator() const;
      
        size_type find (const basic_string& str,
                        size_type pos = 0) const;
        size_type find (const charT* s,
                        size_type pos, size_type n) const;
        size_type find (const charT* s,
                        size_type pos = 0) const;
        size_type find (charT c, size_type pos = 0) const;
        size_type rfind(const basic_string& str,
                        size_type pos = npos) const;
        size_type rfind(const charT* s,
                        size_type pos, size_type n) const;
        size_type rfind(const charT* s,
                        size_type pos = npos) const;
        size_type rfind(charT c, size_type pos = npos) const;
        
        size_type find_first_of(const basic_string& str,
        		    size_type pos = 0) const;
        size_type find_first_of(const charT* s,
        		    size_type pos,
                                size_type n) const;
        size_type find_first_of(const charT* s,
                                size_type pos = 0) const;
        size_type find_first_of(charT c,
                                size_type pos = 0) const;
        size_type find_last_of (const basic_string& str,
        		    size_type pos = npos) const;
        size_type find_last_of (const charT* s,
        		    size_type pos,
                                size_type n) const;
        size_type find_last_of (const charT* s,
                                size_type pos = npos) const;
        size_type find_last_of (charT c,
                                size_type pos = npos) const;
        
        size_type find_first_not_of(const basic_string& str,
        			size_type pos = 0) const;
        size_type find_first_not_of(const charT* s,
                                    size_type pos,
        			size_type n) const;
        size_type find_first_not_of(const charT* s,
                                    size_type pos = 0) const;
        size_type find_first_not_of(charT c,
                                    size_type pos = 0) const;
        size_type find_last_not_of (const basic_string& str,
        			size_type pos = npos) const;
        size_type find_last_not_of (const charT* s,
                                    size_type pos,
        			size_type n) const;
        size_type find_last_not_of (const charT* s,
        			size_type pos = npos) const;
        size_type find_last_not_of (charT c,
                                    size_type pos = npos) const;
        
        basic_string substr(size_type pos = 0,
                            size_type n = npos) const;
        int compare(const basic_string& str) const;
        int compare(size_type pos1, size_type n1,
        	const basic_string& str) const;
        int compare(size_type pos1, size_type n1,
        	const basic_string& str,
        	size_type pos2, size_type n2) const;
        int compare(const charT* s) const;
        int compare(size_type pos1, size_type n1,
        	const charT* s, size_type n2 = npos) const;
    };
}
```
Whew \-\- 103 member functions or member function templates! The rocky tunnel expands and we break through a new lake's surface just in time. Somewhat waterlogged but better persons for the experience, we look around inside the new cavern to see if the exercise has let us discover anything interesting.

Was the recitation worth it? With the drudge work now behind us, let's look back at what we've just traversed and use the information to analyze Question 3:


**Membership Has Its Rewards \-\- and Its Costs**
3. Which ones should be members, and which should not? Why?

Recall the advice from #1: Where it's practical, break generic components down into pieces.

Guideline: Prefer "one class (or function), one responsibility."

Guideline: Where possible, prefer writing functions as nonmember nonfriends.

For example, if you write a string class and make searching, pattern-matching, and tokenizing available as member functions, you've hardwired those facilities so that they can't be used with any other kind of sequence. (If this frank preamble is giving you an uncomfortable feeling about basic_string, well and good.) On the other hand, a facility that accomplishes the same goal but is composed of several parts that are also independently usable is often a better design. In this example, it's often best to separate the algorithm from the container, which is what the STL does most of the time.

I [2,3] and Scott Meyers [4] have written before on why some nonmember functions are a legitimate part of a type's interface, and why nonmember nonfriends should be preferred (among other reasons, to improve encapsulation). For example, as Scott wrote in his opening words of the cited article:
    I'll start with the punchline: If you're writing a function that can be implemented as either a member or as a non-friend non-member, you should prefer to implement it as a non-member function. That decision increases class encapsulation. When you think encapsulation, you should think non-member functions. [4]

So when we consider the functions that will operate on a basic_string (or any other class type), we want to make them nonmember nonfriends if reasonably possible. Hence, here are some questions to ask about the members of basic_string in particular:
- **Always make it a member if it has to be one:** Which operations must be members, because the C++ language just says so (e.g., constructors) or because of functional reasons (e.g., they must be virtual)? If they have to be, then oh well, they just have to be; case closed.
- **Prefer to make it a member if it needs access to internals:** Which operations need access to internal data we would otherwise have to grant via friendship? These should normally be members. (There are some rare exceptions such as operations needing conversions on their left-hand arguments and some like operator<<() whose signatures don't allow the *this reference to be their first parameters; even these can normally be nonfriends implemented in terms of (possibly virtual) members, but sometimes doing that is merely an exercise in contortionism and they're best and naturally expressed as friends.)
- **In all other cases, prefer to make it a nonmember nonfriend:** Which operations can work equally well as nonmember nonfriends? These can, and therefore normally should, be nonmembers. This should be the default case to strive for.

It's important to ask these and related questions because we want to put as many functions into the last bucket as possible.

A word about efficiency: For each function, we'll consider whether it can be implemented as a nonmember nonfriend function as efficiently as a member function. Is this premature optimization (an evil I often rail against)? Not a bit of it. The primary reason why I'm going to consider efficiency is to demonstrate just how many of basic_string's member functions could be implemented "equally well as nonmember nonfriends." I want to specifically shut down any accusations that making them nonmember nonfriends is a potential efficiency hit, such as preventing an operation from being performed in constant time if the standard so requires. A secondary reason is optimizing a library is not premature when the library's implementer has concrete information from past experience (e.g., past user reports, or his own experience in the problem domain) about what operations are being used by users in time-sensitive ways. So don't get hung up about premature optimization \-\- we're not falling into that trap here. Rather, we are primarily investigating just how many of basic_string's members could "equally well" be implemented as nonmember nonfriends. Even if you are expecting many to be implementable as nonmembers, the actual results may well surprise you.

Operations That Must Be Members
As a first pass, let's sift out those operations that just have to be members. There are some obvious ones at the beginning of the list.
- constructors (6)
- destructor
- assignment operators (3)
- [] operators (2)

Clearly the above functions must be members. It's impossible to write a constructor, destructor, assignment operator, or [] operator in any other way!

12 functions down, 91 to go...


**Operations That Should Be Members**
We asked: Which operations need access to internal data we would otherwise have to grant via friendship? Clearly these have a reason to be members, and normally ought to be. This list includes the following, some of which provide indirect access to (e.g., begin()) or change (e.g., reserve()) the internal state of the string:
- begin (2)
- end (2)
- rbegin (2)
- rend (2)
- size
- max_size
- capacity
- reserve
- swap
- c_str
- data
- get_allocator

The above ought to be members not only because they're tightly bound to basic_string, but they also happen to form the public interface that nonmember nonfriend functions will need to use. Sure, you could implement these as nonmember friends, but why?

There are a few more that I'm going to add to this list as fundamental string operations:
- insert (three-parameter version)
- erase (1 \-\- the "iter, iter" version)
- replace (2 \-\- the "iter, iter, num, char" and templated versions)

We'll return to the question of insert(), erase(), and replace() a little later. For replace() in particular, it's important to be able to choose well and make the most flexible and fundamental version(s) into member(s).


**Into the Fray: Possibly-Contentious Operations That Can Be Nonmember Nonfriends**
First in this section, allow me to perform a stunning impersonation of a lightning rod by pointing out that all of the following functions have something fundamental in common, to wit: Each one could obviously as easily \-\- and as efficiently \-\- be a nonmember nonfriend.
- at (2)
- clear
- empty	
- length

Sure they can be nonmember nonfriends, that's obvious, no sweat. Of course, the above functions also happen to have something else pretty fundamental in common: They're mentioned in the standard library's container and sequence requirements, as member functions. Hence the lightning-rod effect...

"Yeah, wait!" I can already hear some standardiste-minded people saying, heading in my direction and resembling the beginnings of a brewing lynch mob. "Not so fast! Don't you know that basic_string is designed to meet the C++ Standard's container and sequence requirements, and those requirements require or suggest that some of those functions be members? So quit misleading the readers! They're members \-\- live with it!" Indeed, and true, but for the sake of this discussion I'm going to waive those requirements with a dismissive wave of a hand and a quick escape across some back yards past small barking dogs and various clothes drying on the line.

Having left my pursuers far enough behind to resume reasoned discourse, here's the point: For once, the question I'm considering here isn't what the container requirements say. It's which functions can without loss of efficiency be made nonmember nonfriends, and whether there's any additional benefit to be gained from doing so. If those benefits exist for something the container requirements say must be a member, well, why not point out that the container requirements could be improved while we're at it? And so we shall...

Take empty() as a case in point. Can we implement it as a nonmember nonfriend? Sure... the standard itself requires the following behavior of basic_string::empty(), in the C++ Standard, subclause 21.3.3, paragraph 14:
``` cpp
Returns: size() == 0.
```
Well, now, that's pretty easy to write as a nonmember without loss of efficiency:
``` cpp
template<class charT, class traits, class Allocator>
bool empty( const basic_string<charT, traits, Allocator>& s )
{
  return s.size() == 0;
}
```
Notice that, while we can make size() a member and implement a nonmember empty() in terms of it, we could not do the reverse. In several cases here there's a group of related functions, and perhaps more than one could be nominated as a member and the others implemented in terms of that one as nonmembers. Which function should we nominate to be the member? My advice is to choose the most flexible one that doesn't force a loss of efficiency \-\- that will be the enabling flexible foundation on which the others can be built. In this case, we choose size() as the member because its result can always be cached (indeed, the standard encourages that it be cached because size() "should" run in constant time), in which case an empty() implemented only in terms of size() is no less efficient than anything we could do with full direct access to the string's internal data structures.

For another case in point, what about at()? The same reasoning applies. For both the const and non-const versions, the standard requires the following:
``` cpp
Throws: out_of_range if pos >= size().

Returns: operator[]( pos ).
```
That's easy to provide as a nonmember, too. Each is just a two-line function template, albeit a bit syntactically tedious because of all those template parameters and nested type names \-\- and remember all your typenames! Here's the code:
``` cpp
template<class charT, class traits, class Allocator>
typename basic_string<charT, traits, Allocator>
         ::const_reference
at( const basic_string<charT, traits, Allocator>& s,
    typename basic_string<charT, traits, Allocator>
             ::size_type pos )
{
  if( pos >= s.size() ) throw out_of_range( "don't do that" );
  return s[pos];
}

template<class charT, class traits, class Allocator>
typename basic_string<charT, traits, Allocator>::reference
at( basic_string<charT, traits, Allocator>& s,
    typename basic_string<charT, traits, Allocator>
             ::size_type pos )
{
  if( pos >= s.size() )
    throw out_of_range( "I said, don't do that" );
  return s[pos];
}
```
What about clear()? Easy, that's the same as erase(begin(),end()). No fuss, no muss. Exercise for the reader, and all that.

What about length()? Easy, again \-\- it's defined to give the same result as size(). What's more, note that the other containers don't have length(), and it's there in the basic_string interface as a sort of "string thing", but by making it a nonmember suddenly we can consisently say "length()" about any container. Not too useful in this case because it's just a synonym for size(), I grant you, but a noteworthy point in the principle it illustrates \-\- making algorithms nonmembers immediately also makes them more widely useful and usable.

In summary, let's consider the benefits and drawbacks of providing functions like at() and empty() as members vs. nonmembers. First, just because we can write these members as nonmembers (and nonfriends) without loss of efficiency, why should we do so? What are the actual or potential benefits? Here are several:
- Simplicity. Making them nonmembers lets us write and maintain less code. We can write empty() just once and be done with it forever. Why write it many times as basic_string::empty() and vector::empty() and list::empty() and so forth, including writing them over and over again for each new STL-compliant container that comes along in third-party or internal libraries and even in future versions of the C++ standard? Write it once, be done, move on.
- Consistency. It avoids gratuitous incompatibilities between the member algorithms of different containers, and between the member and nonmember versions of algorithms (some existing inconsistencies between members and nonmembers are pointed out in [5]). If customized behavior is needed, then specialization or overloading of the nonmember function templates should be able to accommodate it.
- Encapsulation. It improves encapsulation (as argued by Meyers strongly and at length in [4]).

Having said that, what are the potential drawbacks? Here are two, although in my opinion they are outweighed by the above advantages:
- Consistency. You could argue that keeping things like empty() as members follows the principle of least surprise \-\- similar functions are members, and in other class libraries things like IsEmpty() functions are commonly members, after all. I think this argument is valid, but weakened when we notice that this wouldn't be surprising at all if people were in the habit of following Meyers' advice in the first place, routinely writing functions as nonmembers whenever reasonably possible. So the question really comes down to whether we ought to trade away real benefits in order to follow a tradition, or to change the tradition to get real benefits. (Of course, if we didn't already have size(), then implementing empty() in particular as a nonmember nonfriend would not be possible. The class's public member functions do have to provide the necessary and sufficient functionality already.) I think this argument is weak because writing them as nonmembers yields a greater consistency, as noted above, than the questionable inconsistency being claimed here.
- Namespace pollution. Because empty() is such a common name, putting it at namespace scope risks namespace pollution \-\- after all, will every function named empty() want to have exactly these semantics? This argument is also valid to a point, but weakened in two ways: First, by noting that encouraging consistent semantics for functions is a Good Thing; and second, by noticing that overload resolution has turned out to be a very good tool for disambiguation, and namespace pollution has never been as big a problem as some have made it out to be in the past. Really, by putting all the names in one place and sharing implementations, we're not polluting that one namespace as much as we're sanitizing the functions by gathering them up and helping to avoid the gratuitous and needless incompatibilities that Meyers mentions (see above).

With perhaps these more contentious choices out of the way, then, let's continue with other operations that can be nonmember nonfriends. Some, like those listed above, are mentioned in the container requirements as members; again, here we're considering not what the standard says we must do, but rather what given a blank page we might choose to do in designing these as members vs. nonmember nonfriends.

**More Operations That Can Be Nonmember Nonfriends**
Further, all of the remaining functions can be implemented as nonmember nonfriends:
- resize (2)
- assign (6)
- += (3)
- append (6)
- push_back
- insert (7 \-\- all but the three-parameter version)
- erase (2 \-\- all but the "iter, iter" version)
- replace (8 \-\- all but the "iter, iter, num, char" and templated versions)	
- copy
- substr
- compare (5)
- find (4)
- rfind (4)
- find_first_of (4)	
- first_last_of (4)
- find_first_not_of (4)
- find_last_not_of (4)

Consider first resize():
``` cpp
    void resize(size_type n, charT c);
    void resize(size_type n);
```
Can each `resize()` be a nonmember nonfriend? Sure it can, because it can be implemented in terms of basic_string's public interface without loss of efficiency. Indeed, the standard's own functional specifications express both versions in terms of the functions we've already considered above. For example:
``` cpp
template<class charT, class traits, class Allocator>
void resize( basic_string<charT, traits, Allocator>& s,
             typename Allocator::size_type n, charT c)
{
  if( n > s.max_size() ) throw length_error( "won't fit" );
  if( n <= s.size() )
  {
    basic_string<charT, traits, Allocator> temp( s, 0, n );
    s.swap( temp );
  }
  else
  {
    s.append( n - s.size(), c );
  }
}

template<class charT, class traits, class Allocator>
void resize( basic_string<charT, traits, Allocator>& s,
             typename Allocator::size_type n )
{
  resize( s, n, charT() );
}
```
Next, assign(): We have six, count 'em, six forms of assign(). Fortunately, this case is simple: most of them are already specified in terms of each other, and all can be implemented in terms of a constructor and operator=() combination.

Next, operator+=(), append(), and push_back(): What about all those pesky flavors of appending operations, namely the three forms of operator+=(), the six-count'em-six forms of append(), and the lone push_back()? Just the similarity ought to alert us to the fact that probably at most one needs to be a member; after all, they're doing about the same thing, even if the details differ slightly, for example appending a character in one case, a string in another case, and an iterator range in still another case. Indeed, as it turns out, all of them can likewise be implemented as nonmember nonfriends without loss of efficiency:
- Clearly operator+=() can be implemented in terms of append(), because that's how it's specified in the C++ standard.
- Equally clearly, five of the six versions of append() can be nonmember nonfriends because they are specified in terms of the three-parameter version of append(), and that in turn can be implemented in terms of insert(), all of which quite closes the append() category.
- Determining the status of push_back() takes only slightly more work, because its operational semantics aren't specified in the basic_string section, but only in the container requirements section, and there we find the specification that a.push_back(x) is just a synonym for a.insert(a.end(),x).

What's the linchpin holding all the members of this group together as valid nonmember nonfriends? It's insert(), hence insert()'s status as a good choice to be the member that does the work and encapsulates the append-related access to the string's internals in one place, instead of spreading the internal access all over the place in a dozen different functions.

Next, insert(): For those of you who might think that six-count'em-six versions of assign() and six-count'em-six versions of append() might have been a little much, those were just the warmup... now we consider the eight-count'em-eight versions of insert(). Above, I've already nominated the three-parameter version of insert() as a member, and now it's time to justify why. First, as noted above, insert() is a more general operation than append(), and having a member insert() allowed all the append() operations to be nonmember nonfriends; if we didn't have at least one insert() as a member, then at least one of the append()s would have had to be a member, and so I chose to nominate the more fundamental and flexible operation. But we have eight-count'em-eight flavors of insert() \-\- which one (or more) ought to be the member(s)? Five of the eight forms of insert() are already specified in terms of the three-parameter form, and the others can also be implemented efficiently in terms of the three-parameter form, so we only need that one form to be a member. The others can all be nonmember nonfriends.

For those of you who might think that the eight-count'em-eight versions of insert() take the cake, well, that was warmup too. In a moment we'll consider the ten-count'em-ten forms of replace(). Before we attempt those, though, let's take a short break to tackle an easier function first, because it turns out that erase() is instructive in building up to dealing with replace()...


**Coffee Break (Sort Of): Erasing erase()**
Next erase(): After talking about the total 30-count'em-30 flavors of assign(), append(), insert(), and replace() \-\- and having dealt with 20 of the 30 already above \-\- you will be relieved to know that there are only three forms of erase(), and that only two of them belong in this section. After what we just went through for the others, this is like knocking off for a well-deserved coffee break...

The troika of erase() members is a little interesting. At least one of these erase() functions must be a member (or friend), there being no other way to erase efficiently using the other already-mentioned member functions alone. There are actually two "families" of erase() functions:
``` cpp
// erase( pos, length )
    basic_string& erase(size_type pos = 0, size_type n = npos);

// erase( iter, ... )
    iterator erase(iterator position);
    iterator erase(iterator first, iterator last);
```
First, notice that the two families' return types are not consistent: the first version returns a reference to the string, whereas the other two return iterators pointing immediately after the erased character or range. Second, notice that the two families' argument types are not consistent: the first version takes an offset and length, whereas the other two take an iterator or iterator range; fortunately, we can convert from iterators to offsets via pos = iter - begin() and from offsets to iterators via iter = begin() + pos.

(Aside: The standard does not require, but an implementer can choose, that basic_string objects store their data in a contiguous charT array buffer in memory. If they do, then the conversion from iterators to positional offsets and vice versa clearly need incur no overhead. (I would argue that even segmented storage schemes could provide for very efficient conversion back and forth between offsets and iterators using only the container's and iterator's public interfaces.)

This allows the first two forms to be expressed in terms of the third (again, remember your typenames and qualifications!):

template<class charT, class traits, class Allocator>
basic_string<charT, traits, Allocator>&
erase( basic_string<charT, traits, Allocator>& s,
       typename Allocator::size_type pos = 0,
       typename Allocator::size_type n =
         basic_string<charT, traits, Allocator>::npos )
{
  if( pos > s.size() )
    throw out_of_range( "yes, we have no bananas" );
  typename basic_string<charT, traits, Allocator>::iterator
    first = s.begin()+pos,
    last = n == basic_string<charT, traits, Allocator>::npos
           ? s.end() : first + min( n, s.size() - pos );
  if( first != last )
    s.erase( first, last );
  return s;
}

template<class charT, class traits, class Allocator>
typename basic_string<charT, traits, Allocator>::iterator
erase( basic_string<charT, traits, Allocator>& s,
       typename basic_string<charT, traits, Allocator>
                ::iterator position )
{
  return s.erase( position, position+1 );
}

OK, coffee break's over...


**Back to Work: Replacing replace()**
Next, replace(): Truth be told, the ten-count'em-ten replace() members are less interesting than they are tedious and exhausting.

At least one of these replace() functions must be a member (or friend), there being no other way to replace efficiently using the other already-mentioned member functions alone. In particular, note that you can't efficiently implement replace() in terms erase() followed by insert() or vice versa because both ways would require more character shuffling, and possibly buffer reallocation.

Note again that we have two families of replace() functions:
``` cpp
// replace( pos, length, ... )
    basic_string& replace(size_type pos1, size_type n1, // #1
                          const basic_string& str);
    basic_string& replace(size_type pos1, size_type n1, // #2
                          const basic_string& str,
                          size_type pos2, size_type n2);
    basic_string& replace(size_type pos, size_type n1,  // #3
                          const charT* s, size_type n2);
    basic_string& replace(size_type pos, size_type n1,  // #4
                          const charT* s);
    basic_string& replace(size_type pos, size_type n1,  // #5
                          size_type n2, charT c);

// replace( iter, iter, ... )
    basic_string& replace(iterator i1, iterator i2,     // #6
                          const basic_string& str);
    basic_string& replace(iterator i1, iterator i2,     // #7
                          const charT* s, size_type n);
    basic_string& replace(iterator i1, iterator i2,     // #8
                          const charT* s);
    basic_string& replace(iterator i1, iterator i2,     // #9
                          size_type n, charT c);
    template<class InputIterator>                       // #10
      basic_string& replace(iterator i1, iterator i2,
                            InputIterator j1,
                            InputIterator j2);
```
This time, the two families' return types are consistent; that's a small pleasure. But, as with erase(), the two families' argument types are not consistent: one family is based on an offset and length, whereas the other is based on an iterator range. As with erase(), because we can convert between iterators and positions, we can easily implement one family in terms of the other.

When considering which must be members, we want to choose the most flexible and fundamental version(s) as members and implement the rest in terms of those. Here are a few pitfalls one might encounter while doing this analysis, and how one might avoid them. Consider first the first family:
- One function (#2)? One might notice that the standard specifies all of the first family in terms of the #2 version. Unfortunately, some of the passthroughs would needlessly construct temporary basic_string objects, so we can't get by with #2 alone even for just the first family. The standard specifies the observable behavior, but the operational specification isn't necessarily the best way to actually implement the a given function.

- Two functions (#3 and #5)? One might notice that all but #5 in the first family can be implemented efficiently in terms of #3, but then #5 would still need to be special-cased to avoid needlessly creating a temporary string object (or its equivalent).

Consider second the second family:
- One function (#6)? One might notice that the standard specifies all of the second family in terms of #6. Unfortunately, again, some of the passthroughs would needlessly construct temporary basic_string objects, so we can't get by with #6 alone even for just the second family.	
- Three functions (#7, #9, #10)? One might notice that most of the functions in the second family can be implemented efficiently in terms of #7, except for #9 (for the same reason that made #5 the outcast in the first family, namely that there was no existing buffer with the correct contents to point to) and #10 (which cannot assume that iterators are pointers, or even for that matter basic_string::iterators!).
- Two functions (#9, #10)! One might then immediately notice that all but #9 in the second family can be implemented efficiently in terms of #10, including #7. In fact, assuming string contiguity and position/iterator convertability as we've already assumed, we could probably even handle all the members of the first family... aha! That's it.

So it appears that the best we can do is two member functions upon which we can then build everything else as nonmember nonfriends. The members are the "iter, iter, num, char" and templated versions. The nonmembers are everything else. (Exercise for the reader: For each of the other eight versions, write sample implementations as efficient nonmember nonfriends.)

Note that #10 well illustrates the power of templates \-\- this one function can be used to implement all but two of the others without any loss of efficiency, and to implement the remaining ones with what would probably be only a minor loss of efficiency (constructing a temporary basic_string containing n copies of a character).

Time for another quick coffee break...

Coffee Break #2: Copying copy() and substr()
Oh, copy(), schmopy(). Note that copy() is a somewhat unusual beast, and that its interface is inconsistent with the std::copy() algorithm. Note again the signature:
``` cpp
size_type copy(charT* s, size_type n, size_type pos = 0) const;
```
The function is const; it does not modify the string. Rather, what the string object has to do is copy part of itself (up to n characters, starting at position pos) and dump it into the target buffer (note, I deliberately did not say "C-style string"), s, which is required to be big enough \-\- if it's not, oh well, then the program will scribble onward into whatever memory happens to follow the string and get stuck somewhere in the Undefined Behavior swamp. And, better still, basic_string::copy() does not, repeat not, append a null object to the target buffer, which is what makes s not a C-style string (besides, charT doesn't need to be char; this function will copy into a buffer of whatever kind of characters the string itself is made of). It is also what makes copy() a dangerous function.

Guideline: Never use functions that write to range-unchecked buffers (e.g., strcpy(), sprintf(), basic_string::copy()). They are not only crashes waiting to happen, but a clear and present security hazard \-\- buffer overrun attacks continue to be a perennially popular pastime for hackers and malware writers.

All of the required work could be done pretty much as simply, and a lot more flexibly, just by using plain old std::copy():
``` cpp
string s = "0123456789";

char* buf1 = new char[5];
s.copy( buf1, 0, 5 );  // ok: buf will contain the chars
                       //     '0', '1', '2', '3', '4'
copy( s.begin(), s.begin()+5, buf1 );
                       // ok: buf will contain the chars
                       //     '0', '1', '2', '3', '4'

int* buf2 = new int[5];
s.copy( buf2, 0, 5 );  // error: first parameter is not char*
copy( s.begin(), s.begin()+5, buf2 );
                       // ok: buf2 will contain the values
                       //     corresponding to '0', '1', '2', '3', '4'
                       //	 (e.g., ASCII values)
```
Incidentally, the above code also shows how basic_string::copy() can trivially be written as a nonmember nonfriend, most trivially in terms of the copy() algorithm \-\- another exercise for the reader, but do remember to handle the n == npos special case correctly.

While we're taking a breather, let's knock off another simple one at the same time: substr(). Recall its signature:
``` cpp
basic_string substr(size_type pos = 0, size_type n = npos) const;
```
We'll neglect for the moment to comment on the irksome fact that this function chooses to take its parameters in the order "position, length", whereas the copy() we just considered takes the very same parameters in the order "length, position". Nor will we mention that, besides being aesthetically inconsistent, trying to remember which function takes the parameters in which order makes for an easy trap for users of basic_string to stumble into because both parameters also happen to be of the same type (size_type), worse luck, so that if the users get it wrong their code will continue to happily compile without any errors or warnings and continue to happily run... well, except only every so often hiccupping and generating odd user support calls when odd strings get emitted after odd and wrong substrings get taken. And never would we state graphically that all of this could be viewed as the moral equivalent of neglectfully leaving a land mine neatly labeled "Design By Committee™" lying around the countryside just waiting to blow up without warning beneath the unwary.

Like I said, we won't comment on all that. Not a word. No, with blinders clamped firmly in place we'll ignore the carnage on either side, the cries of confusion, the smell of powder, and we'll march strictly along the main line of thought we're doggedly pursuing \-\- member vs. nonmember nonfriend functions.

Notice that substr() can be easily implemented as a nonmember nonfriend, because it's syntactic sugar for a string constructor \-\- after all, the standard itself specifies that it must simply return a fresh basic_string object constructed using the equivalent of basic_string<charT,traits,Allocator>( data()+pos, min(n, size()-pos) ). That is, creating a new string that's a substring of an existing one can be done equally well with or without substr(), using the more general string constructor which already does all this and a lot more besides:
``` cpp
string s = "0123456789";

string s2 = s.substr( 0, 5 );        // s2 contains "01234"
string s3( s.data(), 5 );            // s3 contains "01234"
string s4( s.begin(), s.begin()+5 ); // s4 contains "01234"
```

All right, break's over. Back to work again... fortunately we have only two families left to consider, compare() and the *find*()s.

Almost There: Comparing compare()s
The penultimate family is compare(). It has five members, all of which can be trivially shown to be implementable efficiently as nonmember nonfriends. How? Because in the standard they're specified in terms of basic_string's size() and data(), which we already decided to make members, and traits::compare(), which does the real work.

Wow, wait a minute. That was almost easy! Let's not question it, but move right along...

The Home Stretch: Finding the find()s
Our relief doesn't last long, alas. Lest the sight of the eight-count'em-eight flavors of insert() and the ten-count'em-ten versions of replace() wasn't enough to bring you verily to the verge of tears, we end on a truly unsurpassable distressing note, namely this: the 24-count'em-24 (yes, really) variations on find()-like algorithms.

There are six families of find functions, each with exactly four members:
- find \-\- forward search for the first occurrence of a string or character (str, s, or c) starting at point pos
- rfind \-\- backward search for the first occurrence of a string or character (str, s, or c) starting at point pos
- find_first_of \-\- forward search for the first occurrence of any of one or more characters (str, s, or c) starting at point pos
- find_last_of \-\- backward search for the first occurrence of any of one or more characters (str, s, or c) starting at point pos
- find_first_not_of \-\- forward search for the first occurrence of any but one or more characters (str, s, or c) starting at point pos
- find_last_not_of \-\- backward search for the first occurrence of any but one or more characters (str, s, or c) starting at point pos

Each family has four members:
- "str, pos" where str contains the characters to search for (or not) and pos is the starting position in the string
- "ptr, pos, n" where ptr is a charT* pointing to a buffer of length n containing the characters to search for (or not) and pos is the starting position in the string
- "ptr, pos" where ptr is a charT* pointing to a null-terminated buffer containing the characters to search for (or not) and pos is the starting position in the string
- "c, pos" where c the characters to search for (or not) and pos is the starting position in the string

All of these can be written efficiently as nonmember nonfriends; the implementations are left as exercises for the reader. Having said that, we're done!

But let's add one final note about string finding. In fact, you might have noticed that, in addition to the extensive bevy of basic_string::*find*() algorithms, the C++ Standard also provides a not-quite-as-extensive-but-still-plentiful bevy of std::*find*() algorithms. In particular:
- std::find() can do the work of basic_string::find()
- std::find() using reverse_iterators, or std::find_end(), can do the work of basic_string::rfind()
- std::find_first_of(), or std::find() with an appropriate predicate, can do the work of basic_string::find_first_of()	
- std::find_first_of(), or std::find() with an appropriate predicate, using reverse_iterators can do the work of basic_string::find_last_of()
- std::find() with an appropriate predicate can do the work of basic_string::find_first_not_of()	
- std::find() with an appropriate predicate and using reverse_iterators can do the work of basic_string::find_last_not_of()

What's more, the nonmember algorithms are more flexible, because they work on more than just strings. Indeed, all of the basic_string::*find*() algorithms could be implemented using the std::find and std::find_end(), tossing in appropriate predicates and/or reverse_iterators as necessary.

So what about just ditching the basic_string::*find*() families altogether and just telling users to use the existing std::find*() algorithms? One caution here is that, even though the basic_string::*find*() work can be emulated, doing it with the default implementations of std::find*() would incur significant loss of performance in some cases, and there's the rub. The three forms each of find() and rfind() that search for substrings (not just individual characters) can be made much more efficient than a brute-force search that tries each position and compares the substrings starting at those positions. There are well-known algorithms that construct finite state machines on the fly to run through a string and find a substring (or prove is absence) in linear time, and it might be desirable to take advantage of such techniques.

To take advantage of such optimizations, could we provide overloads (not specializations, see [6]) of std::find*() that work on basic_string iterators? Yes, but only if basic_string::iterator is a class type, not a plain charT*. The reason is that we wouldn't want to specialize std::find() for all pointer types is because not all character pointers necessarily point into strings; so we would need basic_string::iterator to be a distinct type that we can detect and partially specialize on. Then those specializations could perform the optimizations and work at full efficiency for matching substrings.


**Summary**
Decomposition and encapsulation are Good Things. In particular, it's often best to separate the algorithm from the container, which is what the STL does most of the time.

It's widely accepted that basic_string has way too many member functions. Of the 103 functions in basic_string, only 32 really need to be members, and 71 could be written as nonmember nonfriends without loss of efficiency. In fact, many of them needlessly duplicate functionality already available as algorithms, or are themselves algorithms that would be useful more widely if only they were decoupled from basic_string instead of buried inside it.

Don't repeat basic_string's mistakes in your design \-\- decouple your algorithms from your containers, use template specialization or overloading to get special-purpose behavior where warranted (as for substring searching), and above all follow the guidelines:

Guideline: Prefer "one class (or function), one responsibility."

Guideline: Where possible, prefer writing functions as nonmember nonfriends.


**References**
[1] A thick tome also known as the ISO C++ Standard.
[2] H. Sutter. ["What's In a Class? The Interface Principle"](http://www.gotw.ca/publications/mill02.htm) (C++ Report, 10(3), March 1998).
[3] H. Sutter. [Exceptional C++](http://www.gotw.ca/publications/xc++.htm), Items 31-34 (Addison-Wesley, 2000).
[4] S. Meyers. [“How Non-Member Functions Improve Encapsulation”](http://www.cuj.com/articles/2000/0002/0002c/0002c.htm) (C/C++ Users Journal, 18(2), February 2000).
[5] S. Meyers. ***Effective STL*** (Addison-Wesley, 2000).
[6] H. Sutter. ["Why Not Specialize Function Templates?"](http://www.gotw.ca/publications/mill17.htm) (C/C++ Users Journal, 19(7), July 2001).



# 085 Style Case Study #3: Construction Unions

**Difficulty: 4 / 10**
No, this issue isn't about organizing carpenters and bricklayers. Rather, it's about deciding between what's cool and what's uncool, good motivations gone astray, and the consequences of subversive activities carried on under the covers. It's about getting around the C++ rule of using constructed objects as members of unions.



**Problem**
**JG Questions**
1. What are unions, and what purpose do they serve?

2. What kinds of types cannot be used as members of unions? Why do these limitations exist? Explain.

 

**Guru Questions**
3. The article in [1] cites the motivating case of writing a scripting language: Say that you want your language to support a single type for variables that at various times can hold an integer, a string, or a list. Creating a union { int i; list<int> l; string s; } doesn't work for the reasons given above. The following code presents a workaround that attempts to support allowing any type to participate in a union. For a more detailed explanation, see the original article.

Critique this code and identify:

a) Mechanical errors, such as invalid syntax or nonportable conventions.

b) Stylistic improvements that would improve code clarity, reusability, and maintainability.

``` cpp
#include <list>
#include <string>
#include <iostream>
using namespace std;

#define max(a,b) (a)>(b)?(a):(b)

typedef list<int> LIST;
typedef string STRING;

struct MYUNION {
    MYUNION() : currtype( NONE ) {}
    ~MYUNION() {cleanup();}
    
    enum uniontype {NONE,_INT,_LIST,_STRING};
    uniontype currtype;
    
    inline int& getint();
    inline LIST& getlist();
    inline STRING& getstring();

protected:
    union {
        int i;
        unsigned char buff[max(sizeof(LIST),sizeof(STRING))];
    } U;
    
    void cleanup();
};

inline int& MYUNION::getint()
{
    if( currtype==_INT ) {
        return U.i;
    } 
	else {
        cleanup();
        currtype=_INT;
        return U.i;
    } // else
}

inline LIST& MYUNION::getlist()
{
    if( currtype==_LIST ) {
        return *(reinterpret_cast<LIST*>(U.buff));
    } else {
        cleanup();
        LIST* ptype = new(U.buff) LIST();
        currtype=_LIST;
        return *ptype;
    } // else
}

inline STRING& MYUNION::getstring()
{
    if( currtype==_STRING) {
        return *(reinterpret_cast<STRING*>(U.buff));
    } else {
        cleanup();
        STRING* ptype = new(U.buff) STRING();
        currtype=_STRING;
        return *ptype;
    } // else
}

void MYUNION::cleanup()
{
    switch( currtype ) {
        case _LIST: {
            LIST& ptype = getlist();
            ptype.~LIST();
            break;
        } // case
        case _STRING: {
            STRING& ptype = getstring();
            ptype.~STRING();
            break;
        } // case
        default: break;
    }
    currtype=NONE;
}
```
(For an idea of the kinds of things I'm looking for, see also Style Case Study #1 and Style Case Study #2.)

4. Show a better way to achieve a generalized variant type, and comment on any tradeoffs you encounter.

----

**Solution**
**Unions Redux**
1. What are unions, and what purpose do they serve?

Unions allow more than one object, of either class or builtin type, to occupy the same space in memory. For example:
``` cpp
// Example 1
//
union U
{
    int i;
    float f;
};

U u;

u.i = 42;    // ok, now i is active
std::cout << u.i << std::endl;

u.f = 3.14f; // ok, now f is active
std::cout << 2 * u.f << std::endl;
```

But only one of the types can be "active" at a time \-\- after all, the storage can after all only hold one value at a time. Also, unions only support some kinds of types, which leads us into the next question:

 

2. What kinds of types cannot be used as members of unions? Why do these limitations exist? Explain.

From the C++ standard:

An object of a class with a non-trivial constructor, a non-trivial copy constructor, a non-trivial destructor, or a non-trivial copy assignment operator cannot be a member of a union, nor can an array of such objects.

In brief, for a class type to be usable in a union, it must meet all of the following criteria:

- The only constructors, destructors, and copy assignment operators are the compiler-generated ones.
- There are no virtual functions or virtual base classes.
- Ditto for all of its base classes and nonstatic members (or arrays thereof).

That's all, but that sure eliminates a lot of types.

Unions were inherited from C. The C language has a strong tradition of efficiency and support for low-level close-to-the-metal programming, which has been compatibly preserved in C++; that's why C++ also has unions. On the other hand, the C language does not have any tradition of language support for an object model supporting class types with constructors and destructors and user-defined copying, which C++ definitely does; that's why C++ also has to define what, if any, uses of such newfangled types make sense with the "oldfangled" unions, and do not violate the C++ object model including its object lifetime guarantees.

If C++'s restrictions on unions did not exist, Bad Things could happen. For example, consider what could happen if the following code were allowed:

``` cpp
// Example 2: Not Standard C++ code, but what if it were allowed?
//
void f()
{
    union IllegalImmoralAndFattening
    {
        std::string s;
        std::auto_ptr<int> p;
    };
    
    IllegalImmoralAndFattening iiaf;
    
    iiaf.s = "Hello, world"; // has s's constructor run?
    iiaf.p = new int(4); // has p's constructor run?
}
// will s get destroyed? should it be?
// will p get destroyed? should it be?
```

As the comments indicate, serious problems would exist if this were allowed. To avoid further complicating the language by trying to craft rules that at best only might partly patch up a few of the problems, the problematic operations were simply banished.

But don't think that unions are only a holdover from earlier times. Unions are perhaps most useful for saving space by allowing data to overlap, and this is still desirable in C++ and in today's modern world. For example, some of the most advanced C++ standard library implementations in the world now use just this technique for implementing the "small string optimization," a great optimization alternative that reuses the storage inside a string object itself: for large strings, space inside the string object stores the usual pointer to the dynamically allocated buffer and housekeeping information like the size of the buffer; for small strings, the same space is instead reused to store the string contents directly and completely avoid any dynamic memory allocation. For more about the small string optimization (and other string optimizations and pessimizations in considerable depth), see Items 13-16 in my book More Exceptional C++ [2], or Scott Meyers' discussion of current commercial std::string implementations in Effective STL [3].

**Toward Dissection and Correction**

3. The article in [1] cites the motivating case of writing a scripting language: Say that you want your language to support a single type for variables that at various times can hold an integer, a string, or a list. Creating a union { int i; list<int> l; string s; } doesn't work for the reasons given above. The following code presents a workaround that attempts to support allowing any type to participate in a union. For a more detailed explanation, see the original article.

On the plus side, the cited article addresses a real problem, and clearly much effort has been put into coming up with a good solution. Unfortunately, from well-intentioned beginnings more than one programmer has gone badly astray.

The problems with the design and the code fall into three major categories: legality, safety, and morality.

Critique this code and identify:

    a) Mechanical errors, such as invalid syntax or nonportable conventions.
    b) Stylistic improvements that would improve code clarity, reusability, and maintainability.

The first overall comment that needs to be made is that the fundamental idea behind this code is not legal in Standard C++. The original article summarizes the key idea:

    "The idea is that instead of declaring object members, you instead declare a raw buffer [non-dynamically, as a char array member inside the object pretending to act like a union] and instantiate the needed objects on the fly [by in-place construction]." [1]

The idea is common, but unfortunately isn't sound. This technique is nonconforming and nonportable because buffers that are not dynamically allocated (e.g., via malloc() or new()) are not guaranteed to be correctly aligned for any other type. Even if this technique happens to accidentally work for some types on someone's current compiler, there's no guarantee it will continue to work for other types, or for the same types in the next version of the same compiler. For more details and some directly related discussion, see for example Item 30 in Exceptional C++, notably the sidebar titled "Reckless Fixes and Optimizations, and Why They're Evil." [4] See also the alignment discussion in [9].

For C++0x, the standards committee is considering adding alignment aids to the language specifically to enable techniques that rely on alignment like this, but that's all still in the future. For now, to make this work reasonably reliably even some of time, you'd have to do one of the following:
- Rely on the max_align hack (see the above citation which footnotes the max_align hack, or do a Google search for max_align); or
- Rely on nonstandard extensions like Gnu's __alignof__ to make this work reliably on a particular compiler that supports such an extension. (Even though Gnu provides an ALIGNOF macro intended to work more reliably on other compilers, it too is admitted "hackery" that relies on the compiler's laying out objects in certain ways and making guesses based on offsetof() inquiries, which may often be a good guess but is not guaranteed by the standard. See for example [5].)

You could work around this by dynamically allocating the array using malloc() or new(), which would guarantee that the char buffer is suitably aligned for object of any type, but that would still be a bad idea (it's still not type-safe) and it wouldn't achieve the potential efficiency gains that the original article was aiming for. An alternative and correct solution would be to use boost::any (see below) which incurs a similar allocation/indirection overhead and is also both safe and correct; more about that later on.

Attempts to work against the language, or to make the language work the way we want it to work instead of the way it actually does work, are often questionable and should be a big red flag. In the Exceptional C++ sidebar cited above, while in an ornery mood I also accused a similar technique of "just plain wrongheadedness" followed by some pretty strong language. There can still be cases where it could be reasonable to use constructs that are known to be nonportable but okay in a particular environment (in this case, perhaps using the max_align hack), but even then I would argue that that fact should be noted explicitly and further that it still has no place in a general piece of code recommended for wide use.

``` cpp 

#include <list>
#include <string>
#include <iostream>
using namespace std;

Since new is going to be used below, also #include <new>. (The <iostream> header was used later in the original code, not shown here, which had a test harness that emitted output.)

#define max(a,b) (a)>(b)?(a):(b)

typedef list<int> LIST;
typedef string STRING;

struct MYUNION {
    MYUNION() : currtype( NONE ) {}
    ~MYUNION() {cleanup();}
```
The first classic mechanical error above is that MYUNION is unsafe to copy because the programmer forgot to provide a suitable copy constructor and copy assignment operator.

MYUNION is choosing to play games that require special work be done in the constructor and destructor, so these are provided as above; that's fine as far as it goes. But it doesn't go far enough, because the same games require special work in the copy constructor and copy assignment operator, which are not provided. The default compiler-generated copying operations do the wrong thing, namely copy the contents bitwise as an array of chars, which is likely to have most unsatisfactory results, in most cases leading straight to memory corruption. Consider the following code:
``` cpp
// Example 3-1: MYUNION is unsafe for copying
//
{
    MYUNION u1, u2;
    u1.getstring() = "Hello, world";
    u2 = u1; // copies the bits of u1 to u2
} // oops, double delete of the string (assuming the bitwise copy even made sense)
```

Guideline: Observe the Law of the Big Three: If a class needs a custom copy constructor, copy assignment operator, or destructor, it probably needs all three.

Passing on from the classic mechanical error, we next encounter a duo of classic stylistic errors:
``` cpp
    enum uniontype {NONE,_INT,_LIST,_STRING};
    uniontype currtype;
    
    inline int& getint();
    inline LIST& getlist();
    inline STRING& getstring();
```

There are two stylistic errors here. First, this struct is not reusable because it is hardcoded for specific types. Indeed, the original article recommended handcoding such a struct every time it was needed. Second, even given its limited intended usefulness, it is not very extensible or maintainable. We'll return to this frailty again later, once we've covered more of the context.

There are also two mechanical problems. The first is that currtype is public for no good reason; this violates good encapsulation and means any user can freely mess with the type, even by accident. The second mechanical problem concerns the names used in the union; I'll cover that in its own section, "Underhanded Names," later on.
``` cpp
protected:
```

Next, we encounter another mechanical error: The internals ought to be private, not protected. The only reason to use protected would be to make the internals available to derived classes, but there had better not be any derived classes because MYUNION is unsafe to derive from for several reasons \-\- not least because of the murky and abstruse games it plays with its internals, and because it lacks a virtual destructor.
``` cpp
    union {
        int i;
        unsigned char buff[max(sizeof(LIST),sizeof(STRING))];
    } U;
    
    void cleanup();
};
```

That's it for the main class definition. Moving on, consider the three parallel accessor functions:
``` cpp
inline int& MYUNION::getint()
{
    if( currtype==_INT ) {
        return U.i;
    } else {
        cleanup();
        currtype=_INT;
        return U.i;
    } // else
}

inline LIST& MYUNION::getlist()
{
    if( currtype==_LIST ) {
        return *(reinterpret_cast<LIST*>(U.buff));
    } else {
        cleanup();
        LIST* ptype = new(U.buff) LIST();
        currtype=_LIST;
        return *ptype;
    } // else
}

inline STRING& MYUNION::getstring()
{
    if( currtype==_STRING) {
        return *(reinterpret_cast<STRING*>(U.buff));
    } else {
        cleanup();
        STRING* ptype = new(U.buff) STRING();
        currtype=_STRING;
        return *ptype;
    } // else
}
```

A minor nit: The `"// else"` comment adds nothing. It's unfortunate that the only comments in the code are useless ones.

More seriously, there are three major problems here. The first is that the functions are not written symmetrically, and whereas the first use of a list or a string yields a default-constructed object, the first use of int yields an uninitialized object. If that is intended, in order to mirror the ordinary semantics of uninitialized int variables, that should be documented; since it is not, the int ought to be initialized. For example, if the caller accesses `getint()` and tries to make a copy of the (uninitialized) value, the result is undefined behavior \-\- not all platforms support copying arbitrary invalid int values, and some will reject the instruction at runtime.

The second major problem is that this code hinders const-correct use. If the code is really going to be written the above way, then at least it would be useful to also provide const overloads for each of these functions; each would naturally return the same thing as its non-const counterpart, but by a reference to const.

The third major problem is that the approach above is fragile and brittle in the face of change. It relies on type switching (see any of Steve Dewhurst's many commentaries against this notion in other contexts in previous issues of CUJ), and it's easy to accidentally fail to keep all the functions in sync when you add or remove new types.

Stop reading here and consider: What do you have to do in the above code if you want to add a new type? Make as complete a list as you can.

----

Are you back? All right, here's the list I came up with. To add a new type, you have to remember to: (a) add a new enum value; (b) add a new accessor member; (c) update the cleanup() function to safely destroy the new type; and (d) add that type to the max() calculation to ensure buff is sufficiently large to hold the new type too.

If you missed one or more of those, well, that just illustrates how difficult this code really is to maintain and extend.

Pressing onward, we come to the final function:
``` cpp
void MYUNION::cleanup()
{
    switch( currtype ) {
        case _LIST: {
            LIST& ptype = getlist();
            ptype.~LIST();
            break;
        } // case
        case _STRING: {
            STRING& ptype = getstring();
            ptype.~STRING();
            break;
        } // case
        default: break;
    } // switch
    currtype=NONE;
}
```

Let's reprise that small commenting nit again: The `"// case"` and `"// switch"` comments add nothing; it's unfortunate that the only comments in the code are useless ones. It is better to have no comments at all than to have comments that are just distractions.

But there's a larger issue here: Rather than having simply `"default: break;"`, it would be good to make an exhaustive list (including the "int" type) and signal a logic error if the type is unknown \-\- perhaps via `"throw std::logic_error(...);"`.

Again, type switching is purely evil. A Google search for "switch C++ Dewhurst" will yield all sorts of interesting references on this topic, including [6]; see those for more details, if you need more ammo to convince colleagues to avoid the type-switching beast.

Guideline: Avoid type switching; prefer type safety.

**Underhanded Names**
There's one mechanical problem I haven't yet covered. This problem first rears its ugly, unshaven, and unshampooed head in the following line:
``` cpp
  enum uniontype {NONE,_INT,_LIST,_STRING};
```
Never, ever, ever create names that begin with an underscore or contain a double underscore; they're reserved for your compiler and standard library vendor's exclusive use, so that they have names that they can use without tromping on your code. Tromp on their names, and their names might just tromp back on you! (The more specific rule is that any name with a double underscore anywhere in it `__like__this` or that starts with an underscore and a capital letter `_LikeThis` is reserved. You can remember that rule if you like, but it's a bit easier to just avoid both leading underscores and double underscores entirely.)

***Don't stop! Keep reading!*** You might have read this advice before. You might even have read it from me. You might even be tired of it, and yawning, and ready to ignore the rest of this section. If so, this one's for you, because this advice is not at all theoretical, and it bites and bites hard in this code.

The above line happens to compile on most of the compilers I tried (Borland 5.5, Comeau 4.3.0.1, Intel 7.0, gcc 2.95.3 / 3.1.1 / 3.2, and Microsoft Visual C++ 6.0, 7.0, and 7.1 RC1). But under two of them \-\- Metrowerks CodeWarrior 8.2, and the EDG 3.0.1 demo front-end used with the Dinkumware 4.0 standard library \-\- the code breaks horribly.

Under Metrowerks CodeWarrior 8, this line breaks noisily with the first of 52 errors. The 225 lines of error messages begin with the following diagnostics:
``` cpp
### mwcc Compiler:
#    File: 1.cpp
# --------------
#      17:      enum uniontype {NONE,_INT,_LIST,_STRING};
#   Error:                                     ^
#   identifier expected
### mwcc Compiler:
#      18:      uniontype currtype;
#   Error:      ^^^^^^^^^
#   declaration syntax error
```
followed by 52 further error messages, and 215 more lines. What's pretty obvious from the second and later errors is that we should ignore them for now because they're just cascades from the first error \-\- since uniontype was never successfully defined, the rest of the code which uses uniontype extensively will of course break too.

But what's up with the definition of uniontype? The indicated comma sure looks like it's in a reasonable place, doesn't it? There's an identifier happily sitting in front of it, isn't there? All becomes clear when we ask the Metrowerks compiler to spit out the preprocessed output... omitting many many lines, here's what the compiler finally sees:

enum uniontype {NONE,_INT, , };

Aha! That's not valid C++, and the compiler rightly complains about the third comma because there's no identifier in front of it.

But what happened to `_LIST` and `_STRING`? You guessed it \-\- tromped on and eaten by the ravenously hungry Preprocessor Beast. It just so happens that Metrowerks' implementation has macros that happily strip away the names `_LIST` and `_STRING`, which is perfectly legal and legitimate because it (the implementation) is allowed to own those `_Names` (as well as `_Other__names`).

So Metrowerks' implementation happens to eat both _LIST and _STRING. What about EDG's/Dinkumware's? Judge for yourself:
``` sh
"1.cpp", line 17: error: trailing comma is nonstandard
      enum uniontype {NONE,_INT,_LIST,_STRING};
                                     ^
"1.cpp", line 58: error: expected an expression
      if( currtype==_STRING) {
                           ^
"1.cpp", line 63: error: expected an expression
          currtype=_STRING;
                          ^
"1.cpp", line 76: error: expected an expression
          case _STRING: {
                      ^
4 errors detected in the compilation of "1.cpp".
```
This time, even without generating and inspecting a preprocessed version of the file, we can see what's going on: The compiler is behaving as though the word "_STRING" wasn't there. That's because it was \-\- you guess it \-\- tromped on, not to mention thoroughly chewed up and spat out, by the still-peckish Preprocessor Beast.

I hope that this will convince you that when some writers natter on about not using `_Names` like`__`these, the problem is far from theoretical. It's practical indeed, because the naming restriction directly affects your relationship with your compiler and standard library writer. Trespass on their turf, and you might get lucky and remain unscathed; on the other hand, you might not.

The C++ landscape is wide-open and clear and lets you write all sorts of wonderful and flexible code and wander in pretty much whatever direction your development heart desires, including that it lets you choose pretty much whatever names you like outside of namespace std. But when it comes to names, C++ also has one big fenced-off grove, surrounded by gleaming barbed wire and signs that say things like "Employees__Only \-\- Must Have Valid `_Badge` To Enter Here" and "Violators May Be Tromped and Eaten." The above is a stellar example of the tromping one gets for disregarding the `_` Warnings.

Guideline: Never use "underhanded names" \-\- ones that begin with an underscore, or that contain a double underscore.

**Toward a Better Way: boost::any**
4. Show a better way to achieve a generalized variant type, and comment on any tradeoffs you encounter.

The original article says:

    "[Y]ou might want to implement a scripting language with a single variable type that can either be an integer, a string, or a list." [1]

This is true, and there's no disagreement so far. But the article then continues:

    "A union is the perfect candidate for implementing such a composite type." [1]

Rather, the article has served to show in some considerable detail just why a union is not suitable at all.

But if not a union, then what? One very good candidate for implementing such a variant type is Boost's "any" facility, along with its "many" and "any_cast".[7] Jim Hyslop and I discussed it in our article "I'd Hold Anything For You."[8] Interestingly, the complete implementation for the fully general "any" (covering any number/combination of types and even some platform-specific #ifdefs) is about the same amount of code as the sample MYUNION solution for the special case of the three types int, list<int>, and string \-\- and it's fully general, extensible, type-safe, and part of a healthy low-cholesterol diet.

There is still a tradeoff, however, and it is this: Dynamic allocation. The boost::any facility does not attempt to achieve the potential efficiency gain of avoiding a dynamic memory allocation, which was part of the motivation in the original article. Note too that the boost::any dynamic allocation overhead is more than if the original article's code was just modified to use (and reuse) a single dynamically allocated buffer that's acquired once for the lifetime of MYUNION, because boost::any performs a dynamic allocation every time the contained type is changed, too.

Here's how the article's demo harness would look if it instead used boost::any. The old code that uses the original article's version of MYUNION is shown in comments for comparison:
``` cpp
// MYUNION u;
any u;
```
Instead of a handwritten struct, which has to be written again for each use, just use any directly. Note that any is a plain class, not a template.
``` cpp
// access union as integer
// u.getint() = 12345;
u = 12345;
```
The assignment shows any's more natural syntax.
``` cpp
// cout << "int=" << u.getint() << endl;
cout << "int=" << any_cast<int>(u) << endl;
               // or just "int(u)"
```
I like any's cast form better because it's more general (including that it is a nonmember) and more natural to C++ style; you could also use the less verbose "int(u)" without an any_cast if you know the type already. On the other hand, get[type]() is more fragile, harder to write and maintain, and so forth.
``` cpp
// access union as std::list
// LIST& list = u.getlist();
// list.push_back(5);
// list.push_back(10);
// list.push_back(15);

u = list<int>();
list<int>& l = *any_cast<list<int> >(&u);
l.push_back(5);
l.push_back(10);
l.push_back(15);
```
I think any_cast could be improved to make it easier to get references, but this isn't too bad. (Aside: I'd discourage using 'list' as a variable name when it's also the name of a template in scope; too much room for expression ambiguity.)

So far, we've achieved some typability and readability savings. The remaining differences are more minor:
``` cpp
// LIST::iterator it = list.begin();
list<int>::iterator it = l.begin();
while( it != l.end() ) {
  cout << "list item=" << *(it) << endl;
  it++;
} // while
```
Pretty much unchanged.
``` cpp
// access union as std::string
// STRING& str = u.getstring();
// str = "Hello world!";
u = string("Hello world!");
```
Again, about a wash; I'd say the any version is slightly simpler than the original, but only slightly.
``` cpp
// cout << "string='" << str.c_str() << "'" << endl;
cout << "string='" << any_cast<string>(u) << "'" << endl;
                   // or just "string(u)"
```
As before.

Alexandrescu's Discriminated Unions
Is it possible to fully achieve both of the original goals \-\- safety and avoiding dynamic memory \-\- in a conforming Standard C++ implementation? That sounds like a problem that someone like Andrei Alexandrescu would love to sink his teeth into, especially if it could somehow involve complicated templates. As evidenced in [9], [10], and [11], where Andrei describes his discriminated unions (a.k.a. Variant) approach, it turns out that:
- it is (something he would love to tackle), and
- it can (involve weird templates, and just one quote from [9] says it all: "Did you know that unions can be templates?"), so
- he does.

In short, by performing heroic efforts to push the boundaries of the language as far as possible, Alexandrescu's Variant comes very close to being a truly portable solution. It falls only slightly short, and is probably portable enough in practice even though it goes beyond the pale of what the Standard guarantees. Its main problem is that, even ignoring alignment-related issues, the Variant code is so complex and advanced that it actually works on very few compilers \-\- in my testing, I only managed to get it to work with one.

A key part of Alexandrescu's Variant approach is an attempt to generalize the max_align idea to make it a reusable library facility that can itself still be written in conforming Standard C++. The reason for wanting this is specifically to deal with the alignment problems in the code we've been analyzing above, so that a non-dynamic char buffer can continue to be used in relative safety. Alexandrescu makes heroic efforts to use template metaprogramming to calculate a safe alignment. Will it work portably? His discussion of this question follows:

    "Even with the best Align, the implementation above is still not 100-percent portable for all types. In theory, someone could implement a compiler that respects the Standard but still does not work properly with discriminated unions. This is because the Standard does not guarantee that all user-defined types ultimately have the alignment of some POD type. Such a compiler, however, would be more of a figment of a wicked language lawyer's imagination, rather than a realistic language implementation.
    "[...] Computing alignment portably is hard, but feasible. It never is 100-percent portable." [10]

There are other key features in Alexandrescu's approach, notably a union template that takes a typelist template of the types to be contained, visitation support for extensibility, and an implementation technique that will "fake a vtable" for efficiency to avoid an extra indirection when accessing a contained type. These parts are more heavyweight than boost::any, but are portable in theory. That "portable in theory" part is important \-\- as with Andrei's great work in Modern C++ Design [12] [13], the implementation is so heavy on templates that the code itself contains comments like: "Guaranteed to issue an internal compiler error on: [various popular compilers, Metrowerks, Microsoft, Gnu gcc]", and the mainline test harness contains a commented-out test helpfully labeled "The construct below didn't work on any compiler."

That is Variant's major weakness: Most real-world compilers don't even come close to being able to handle this implementation, and the code should be viewed as important but still experimental. I attempted to build Alexandrescu's Variant code using all of the compilers that I have available: Borland 5.5; Comeau 4.3.0.1; EDG 3.0.1; Intel 7.0; gcc 2.95, 3.1.1, and 3.2; Metrowerks 8.2; and Microsoft VC++ 6.0, 7.0, and 7.1 RC1. As some readers will know, some of the products in that list are very strong and standards-conforming compilers. None of these compilers could successfully compile Alexandrescu's template-heavy source as it was provided.

I tried to massage the code by hand to get it through any of the compilers, but was only successful with Microsoft VC++ 7.1 RC1. Most of the compilers didn't stand a chance, because they did not have nearly strong enough template support to deal with Alexandrescu's code. (Some emitted a truly prodigious quantity of warnings and errors \-\- Intel 7.0's response to compiling main.cpp was to spew back an impressive 430K's worth \-\- really, nearly half a megabyte! \-\- of diagnostic messages.)

I had to make three changes to get the code to compile without errors (although still with some narrowing-conversion warnings at the highest warning level) under Microsoft VC++ 7.1 RC1:


- Added a missing "typename" in class AlignedPOD.
- Added a missing "this->" to make a name dependent in ConverterTo<>::Unit<>::DoVisit().
- Added a final newline character at the end of several headers, as required by the C++ standard (some conforming compilers aren't strict about this and allow the absence of a final newline as a conforming extension; VC++ is stricter and requires the newline). [14]

As the author of [1] commented further about tradeoffs in Alexandrescu's design: "It doesn't use dynamic memory, and it avoids alignment issues and type switching. Unfortunately I don't have access to a compiler that can compile the code, so I can't evaluate its performance vs. myunion and any. Alexandrescu's approach requires 9 supporting header files totaling ~80KB, which introduces its own set of maintenance problems." [15]

I won't try to summarize Andrei's three articles further here, but I encourage readers who are interested in this problem to look them up. They're available online as indicated in the references below.

Guideline: If you want to represent variant types, for now prefer to use boost::any (or something equally simple).

Once the compiler you are using catches up (in template support) and the Standard catches up (in true alignment support) and Variant libraries catch up (in mature implementations), it will be time to consider using Variant-like library tools as type-safe replacements for unions.

**Summary**
Even if the design and implementation of MYUNION are lacking, the motivating problem is both real and worth considering. I'd like to thank Mr. Manley for taking the time to write this article and raise awareness of the need for variant type support, and Kevlin Henney and Andrei Alexandrescu for contributing their own solutions to this area. It is a hard enough problem that Manley's and Alexandrescu's approaches are not strictly portable, standards-conforming C++, although Alexandrescu's Variant makes heroic efforts to get there \-\- Alexandrescu's design is very close to portable in theory, although the implementation is still far from portable in practice because very few compilers can handle the advanced template code it uses.

For now, an approach like Henney's boost::any is the preferred way to go. If in certain places your measurements tell you that you really need the efficiency or extra features provided by something like Alexandrescu's Variant, and you have time on your hands and some template know-how, you might experiment with writing your own scaled-back version of the full-blown Variant by applying only the ideas in [9], [10], and [11] that are applicable to your situation.

 

**References**
[1] K. Manley. "Using Constructed Types in Unions" (C/C++ Users Journal, 20(8), August 2002).
[2] H. Sutter. [More Exceptional C++](http://www.gotw.ca/publications/mxc++.htm) (Addison-Wesley, 2002).
[3] S. Meyers. Effective STL (Addison-Wesley, 2001).
[4] H. Sutter. [Exceptional C++](http://www.gotw.ca/publications/xc++.htm) (Addison-Wesley, 2000).
[5] [http://list-archive.xemacs.org/xemacs-patches/200101/msg00183.html](http://list-archive.xemacs.org/xemacs-patches/200101/msg00183.html)
[6] S. Dewhurst. "C++ Hierarchy Design Idioms", available online at [www.semantics.org/talknotes/SD2002W_HIERARCHY.pdf](http://www.semantics.org/talknotes/SD2002W_HIERARCHY.pdf).
[7] K. Henney. C++ Boost any class, [www.boost.org/libs/any](http://www.boost.org/libs/any).
[8] H. Sutter and J. Hyslop. "I'd Hold Anything For You" (C/C++ Users Journal, 19(12), December 2001), available online at http://www.cuj.com/experts/1912/hyslop.htm.
[9] A. Alexandrescu. ["Discriminated Unions (I)"](http://www.ddj.com/cpp/184403821) (C/C++ Users Journal, 20(4), April 2002).
[10] A. Alexandrescu. ["Discriminated Unions (II)"](http://www.ddj.com/cpp/184403828) (C/C++ Users Journal, 20(6), June 2002).
[11] A. Alexandrescu. ["Discriminated Unions (III)"](http://www.ddj.com/cpp/184403834) (C/C++ Users Journal, 20(8), August 2002).
[12] A. Alexandrescu. Modern C++ Design (Addison-Wesley, 2001).
[13] H. Sutter. ["Review of Alexandrescu's Modern C++ Design"](http://www.gotw.ca/publications/mcd_review.htm) (C/C++ Users Journal, 20(4), April 2002), available online at [http://www.gotw.ca/publications/mcd_review.htm](http://www.gotw.ca/publications/mcd_review.htm).
[14] Thanks to colleague Jeff Peil for pointing out this requirement in clause 2.1/1, which states: "If a source file that is not empty does not end in a new-line character, or ends in a new-line character immediately preceded by a backslash character, the behavior is undefined."
[15] K. Manley, private communication.


# 086 Slight Typos? Graphic Language and Other Curiosities
**Difficulty: 5 / 10**
Sometimes even small and hard-to-see typos can accidentally have a significant effect on code. To illustrate how hard typos can be to see, and how easy phantom typos are to see accidentally even when they're not there, consider these examples.

----

**Problem**
**Guru Questions**
Answer the following questions without using a compiler.

1. What is the output of the following program on a standards-conforming C++ compiler?
``` cpp
#include <iostream>
#include <iomanip>

int main()
{
  int x = 1;
  for( int i = 0; i < 100; ++i );
    // What will the next line do? Increment???????????/
    ++x;
  std::cout << x << std::endl;
}
```

2. How many distinct errors should be reported when compiling the following code on a conforming C++ compiler?
``` cpp
struct X {
  static bool f( int* p )
  {
    return p && 0[p] and not p[1:>>p[2];
  };
};
```


**Solution**
1. What is the output of the following program on a standards-conforming C++ compiler?
``` cpp
#include <iostream>
#include <iomanip>

int main()
{
  int x = 1;
  for( int i = 0; i < 100; ++i );
    // What will the next line do? Increment???????????/
    ++x;
  std::cout << x << std::endl;
}
```
Assuming that there is no invisible whitespace at the end of the comment line, the output is `"1"`.

There are two tricks here, one obvious and one less so.

First, consider the for loop line:
``` cpp
    for( int i = 0; i < 100; ++i );
                                  ^
```

There's a semicolon at the end, a "curiously recurring typo pattern" that (usually accidentally) makes the body of the for loop just the empty statement. Even though the following lines may be indented, and may even have braces around them, they are not part of the body of the for loop. This was a deliberate red herring \-\- in this case, because of the next point, it doesn't matter that the for loop never repeats any statements because there's no increment statement to be repeated at all (even though there appears to be one). This brings us to the second point:

Second, consider the comment line. Did you notice that it ends oddly, with a "/"?

    // What will the next line do? Increment???????????/
                                                       ^

Nikolai Smirnov writes:

    "Probably, what's happened in the program is obvious for you but I lost a couple of days debugging a big program where I made a similar error. I put a comment line ending with a lot of question marks accidentally releasing the `'Shift'` key at the end. The result is unexpected trigraph sequence `'??/'` which was converted to `'\'` (phase 1) which was annihilated with the following `'\n'` (phase 2)." [1]

The `"??/"` sequence is converted to `'\'` which, at the end of a line, is a line-splicing directive (surprise!). In this case, it splices the following line `"++x;"` to the end of the comment line and thus makes the increment part of the comment. The increment is never executed.

Interestingly, if you look at the Gnu g++ documentation for the -Wtrigraphs command-line switch, you will encounter the following statement:

    "Warnings are not given for trigraphs within comments, as they do not affect the meaning of the program." [2]

That may be true most of the time, but here we have a case in point \-\- from real-world code, no less \-\- where this expectation does not hold.

2. How many distinct errors should be reported when compiling the following code on a conforming C++ compiler?
``` cpp
struct X {
    static bool f(int* p)
    {
        return p && 0[p] and not p[1:>>p[2];
    };
};
```

The short answer is: Zero. This code is perfectly legal and standards-conforming (whether the author might have wanted it to be or not).

Let's consider in turn each of the expressions that might be questionable, and see why they're really okay:

- `0[p]` is legal and is defined to have the same meaning as `p[0]`. In C (and C++), an expression of the form `x[y]`, where one of `x` and `y` is a pointer type and the other is an integer value, always means `*(x+y)`. In this case, `0[p]` and `p[0]` have the same meaning  because they mean `*(0+p)` and `*(p+0)`, respectively, which comes out to the same thing. For more details, see clause 6.5.2.1 in the C99 standard [3].
- `and` and `not` are valid keywords that are alternative spellings of `&&` and `!`, respectively.
- `:>` is legal. It is a digraph for the `']'` character, not a smiley (smileys are unsupported in the C++ language outside comment blocks, which is rather a shame). This turns the final part of the expression into `p[1]>p[2]`.
- The `"extra"` semicolon is allowed at the end of a function declaration.

Of course, it could well be that the colon `":"` was a typo and the author really meant `"p[1]>>p[2]"`, but even if it was a typo it's still (unfortunately, in that case) perfectly legal code.

**Acknowledgements**
Thanks to Nikolai Smirnov for contributing part of the Example 1 code; I added the for loop line.

**References**
[1] N. Smirnov, private communication.
[2] A Google search for "trigraphs within comments" yields this and several other interesting and/or amusing hits.
[3] ISO/IEC 9899:1999 (E), International Standard, Programming Languages \-\- C.


# 087 Two-Phase or Not Two-Phase: The Story of Dependent Names in Templates

**Difficulty: 9 / 10**
In the land of C++, there are two towns: The village of traditional nontemplate C++ code, and the hamlet of templates. The two are tantalizingly similar, and so many things are the same in both places that visitors to the templates hamlet might be forgiven for thinking they're right at home, that the template programming environment is the same as it was in their familiar nontemplate surroundings. That is a natural mistake… but some of the local laws are different, often in subtle ways, and some classic assumptions must be unlearned before one can learn to write template code correctly and portably. This articles explores these small but important differences, and how they will soon increasingly affect your code.

----

**Problem**
**JG Questions**
1. In the following code statement, what could f be? List as many possibilities as you can.
``` cpp
    f( a, b );
```

2. Describe at a high level how a C++ compiler performs name lookup on a name like f from Question 1.

**Guru Questions**
3. In a C++ template:

a) What are dependent names and nondependent names?

b) In the following code, for each line, answer whether or not the line contains a dependent name:
``` cpp
template<typename T>
class X : public Y<T> {  // 1 ?
    E f( int ) {           // 2 ?
        g();                 // 3 ?
        this->h();           // 4 ?
	    
        T t1, t2;
        cout << t1;          // 5 ?
        swap( t1, t2 );      // 6 ?
        std::swap( t1, t2 ); // 7 ?
    }
};
```
4. In a C++ template, what do "point of definition" and "point of instantiation" mean?

----

This marks the final issue of GotW. A written-out solution was not posted, but here are brief notes:

1. f could be a function or a function object.

2. See [these other articles](http://www.google.com/search?hl=en&q=%22name+lookup%22+site:gotw.ca) on this site for details on name lookup.

3. Dependent names are names that depend in some way (e.g., qualification) on a template parameter. In this example, lines 1, 4, 5, and 6 contain dependent names and expressions.

4. A template's point of definition is the place where the template code is define. Informally, the point of instantiation is the first point where you use the template or a member of the template; the instantiation could be implicit or explicit.

I hope you've enjoyed GotW over the years! Best wishes,

Herb

