**Question** 

G++ now implements the C++11 thread_local keyword; this differs from the GNU __thread keyword primarily in that it allows dynamic initialization and destruction semantics. Unfortunately, this support requires a run-time penalty for references to non-function-local thread_local variables even if they don't need dynamic initialization, so users may want to continue to use __thread for TLS variables with static initialization semantics.

What is precisely the nature and origin of this run-time penalty?

**Answer**

The dynamic thread_local initialization is added in commit 462819c. One of the change is:

* semantics.c (finish_id_expression): Replace use of thread_local
variable with a call to its wrapper.

So the run-time penalty is that, every reference of the thread_local variable will become a function call. Let's check with a simple test case:

```cpp
// 3.cpp
extern thread_local int tls;    
int main() {
    tls += 37;   // line 6
    tls &= 11;   // line 7
    tls ^= 3;    // line 8
    return 0;
}

// 4.cpp

thread_local int tls = 42;
```

When compiled*, we see that every use of the tls reference becomes a function call to _ZTW3tls, which lazily initialize the the variable once:

```
00000000004005b0 <main>:
main():
  4005b0:   55                          push   rbp
  4005b1:   48 89 e5                    mov    rbp,rsp
  4005b4:   e8 26 00 00 00              call   4005df <_ZTW3tls>    // line 6
  4005b9:   8b 10                       mov    edx,DWORD PTR [rax]
  4005bb:   83 c2 25                    add    edx,0x25
  4005be:   89 10                       mov    DWORD PTR [rax],edx
  4005c0:   e8 1a 00 00 00              call   4005df <_ZTW3tls>    // line 7
  4005c5:   8b 10                       mov    edx,DWORD PTR [rax]
  4005c7:   83 e2 0b                    and    edx,0xb
  4005ca:   89 10                       mov    DWORD PTR [rax],edx
  4005cc:   e8 0e 00 00 00              call   4005df <_ZTW3tls>    // line 8
  4005d1:   8b 10                       mov    edx,DWORD PTR [rax]
  4005d3:   83 f2 03                    xor    edx,0x3
  4005d6:   89 10                       mov    DWORD PTR [rax],edx
  4005d8:   b8 00 00 00 00              mov    eax,0x0              // line 9
  4005dd:   5d                          pop    rbp
  4005de:   c3                          ret

00000000004005df <_ZTW3tls>:
_ZTW3tls():
  4005df:   55                          push   rbp
  4005e0:   48 89 e5                    mov    rbp,rsp
  4005e3:   b8 00 00 00 00              mov    eax,0x0
  4005e8:   48 85 c0                    test   rax,rax
  4005eb:   74 05                       je     4005f2 <_ZTW3tls+0x13>
  4005ed:   e8 0e fa bf ff              call   0 <tls> // initialize the TLS
  4005f2:   64 48 8b 14 25 00 00 00 00  mov    rdx,QWORD PTR fs:0x0
  4005fb:   48 c7 c0 fc ff ff ff        mov    rax,0xfffffffffffffffc
  400602:   48 01 d0                    add    rax,rdx
  400605:   5d                          pop    rbp
  400606:   c3                          ret
Compare it with the __thread version, which won't have this extra wrapper:

00000000004005b0 <main>:
main():
  4005b0:   55                          push   rbp
  4005b1:   48 89 e5                    mov    rbp,rsp
  4005b4:   48 c7 c0 fc ff ff ff        mov    rax,0xfffffffffffffffc // line 6
  4005bb:   64 8b 00                    mov    eax,DWORD PTR fs:[rax]
  4005be:   8d 50 25                    lea    edx,[rax+0x25]
  4005c1:   48 c7 c0 fc ff ff ff        mov    rax,0xfffffffffffffffc
  4005c8:   64 89 10                    mov    DWORD PTR fs:[rax],edx
  4005cb:   48 c7 c0 fc ff ff ff        mov    rax,0xfffffffffffffffc // line 7
  4005d2:   64 8b 00                    mov    eax,DWORD PTR fs:[rax]
  4005d5:   89 c2                       mov    edx,eax
  4005d7:   83 e2 0b                    and    edx,0xb
  4005da:   48 c7 c0 fc ff ff ff        mov    rax,0xfffffffffffffffc
  4005e1:   64 89 10                    mov    DWORD PTR fs:[rax],edx
  4005e4:   48 c7 c0 fc ff ff ff        mov    rax,0xfffffffffffffffc // line 8
  4005eb:   64 8b 00                    mov    eax,DWORD PTR fs:[rax]
  4005ee:   89 c2                       mov    edx,eax
  4005f0:   83 f2 03                    xor    edx,0x3
  4005f3:   48 c7 c0 fc ff ff ff        mov    rax,0xfffffffffffffffc
  4005fa:   64 89 10                    mov    DWORD PTR fs:[rax],edx
  4005fd:   b8 00 00 00 00              mov    eax,0x0                // line 9
  400602:   5d                          pop    rbp
  400603:   c3                          ret
```

This wrapper is not needed for in every use case of thread_local though. This can be revealed from decl2.c. The wrapper is generated only when:

***<1>***
It is not function-local, and,

***<2>***
It is extern (the example shown above), or

The type has a non-trivial destructor (which is not allowed for __thread variables), or

The type variable is initialized by a non-constant-expression (which is also not allowed for __thread variables).

**conclusion**
In all other use cases, it behaves the same as __thread. That means, unless you have some extern __thread variables, you could replace all __thread by thread_local without any loss of performance.
