Per-CPU variables
================================================================================

**In Progress**

Per-CPU variables are one of the kernel features. You can understand what this feature means by reading its name. We can create a variable and each processor core will have its own copy of this variable. We take a closer look on this feature and try to understand how it is implemented and how it works in this part.

The kernel provides API for creating per-cpu variables - `DEFINE_PER_CPU` macro:

```C
#define DEFINE_PER_CPU(type, name) \
        DEFINE_PER_CPU_SECTION(type, name, "")
```

This macro defined in the [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) as many other macros for work with per-cpu variables. Now we will see how this feature is implemented.

Take a look at the `DECLARE_PER_CPU` definition. We see that it takes 2 parameters: `type` and `name`, so we can use it to create per-cpu variable, for example like this:

```C
DEFINE_PER_CPU(int, per_cpu_n)
```

We pass the type and the name of our variable. `DEFI_PER_CPU` calls `DEFINE_PER_CPU_SECTION` macro and passes the same two paramaters and empty string to it. Let's look at the definition of the `DEFINE_PER_CPU_SECTION`:

```C
#define DEFINE_PER_CPU_SECTION(type, name, sec)    \
         __PCPU_ATTRS(sec) PER_CPU_DEF_ATTRIBUTES  \
         __typeof__(type) name
```

```C
#define __PCPU_ATTRS(sec)                                                \
         __percpu __attribute__((section(PER_CPU_BASE_SECTION sec)))     \
         PER_CPU_ATTRIBUTES
```

where section is:

```C
#define PER_CPU_BASE_SECTION ".data..percpu"
```

After all macros are expanded we will get global per-cpu variable:

```C
__attribute__((section(".data..percpu"))) int per_cpu_n
```

It means that we will have `per_cpu_n` variable in the `.data..percpu` section. We can find this section in the `vmlinux`:

```
.data..percpu 00013a58  0000000000000000  0000000001a5c000  00e00000  2**12
              CONTENTS, ALLOC, LOAD, DATA
```

Ok, now we know that when we use `DEFINE_PER_CPU` macro, per-cpu variable in the `.data..percpu` section will be created. When the kernel initilizes it calls `setup_per_cpu_areas` function which loads `.data..percpu` section multiply times, one section per CPU. After the kernel finished the initialization process, we have loaded N `.data..percpu` sections, where N is the number of CPU, and section used by bootstrap processor will contain uninitialized variable created with `DEFINE_PER_CPU` macro.

The kernel provides API for per-cpu variables manipulating:

* get_cpu_var(var)
* put_cpu_var(var)


Let's look at `get_cpu_var` implementation:

```C
#define get_cpu_var(var)     \
(*({                         \
         preempt_disable();  \
         this_cpu_ptr(&var); \
}))
```

Linux kernel is preemptible and accessing a per-cpu variable requires to know which processor kernel running on. So, current code must not be preempted and moved to the another CPU while accessing a per-cpu variable. That's why first of all we can see call of the `preempt_disable` function. After this we can see call of the `this_cpu_ptr` macro, which looks as:

```C
#define this_cpu_ptr(ptr) raw_cpu_ptr(ptr)
```

and

```C
#define raw_cpu_ptr(ptr)        per_cpu_ptr(ptr, 0)
```

where `per_cpu_ptr` returns a pointer to the per-cpu variable for the given cpu (second parameter). After that we got per-cpu variables and made any manipulations on it, we must call `put_cpu_var` macro which enables preemption with call of `preempt_enable` function. So the typical usage of a per-cpu variable is following:

```C
get_cpu_var(var);
...
//Do something with the 'var'
...
put_cpu_var(var);
```

Let's look at `per_cpu_ptr` macro:

```C
#define per_cpu_ptr(ptr, cpu)                             \
({                                                        \
        __verify_pcpu_ptr(ptr);                           \
         SHIFT_PERCPU_PTR((ptr), per_cpu_offset((cpu)));  \
})
```

As I wrote above, this macro returns per-cpu variable for the given cpu. First of all it calls `__verify_pcpu_ptr`:

```C
#define __verify_pcpu_ptr(ptr)
do {
	const void __percpu *__vpp_verify = (typeof((ptr) + 0))NULL;
	(void)__vpp_verify; 
} while (0)
```

which makes given `ptr` type of `const void __percpu *`, 

After this we can see the call of the `SHIFT_PERCPU_PTR` macro with two parameters. At first parameter we pass our ptr and sencond we pass cpu number to the `per_cpu_offset` macro which:

```C
#define per_cpu_offset(x) (__per_cpu_offset[x])
```

expands to getting `x` element from the `__per_cpu_offset` array:


```C
extern unsigned long __per_cpu_offset[NR_CPUS];
```

where `NR_CPUS` is the number of CPUs. `__per_cpu_offset` array filled with the distances between cpu-variables copies. For example all per-cpu data is `X` bytes size, so if we access `__per_cpu_offset[Y]`, so `X*Y` will be accessed. Let's look at the `SHIFT_PERCPU_PTR` implementation:

```C
#define SHIFT_PERCPU_PTR(__p, __offset)                                 \
         RELOC_HIDE((typeof(*(__p)) __kernel __force *)(__p), (__offset))
```

`RELOC_HIDE` just returns offset `(typeof(ptr)) (__ptr + (off))` and it will be pointer of the variable.

That's all! Of course it is not the full API, but the general part. It can be hard for the start, but to understand per-cpu variables feature need to understand mainly [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) magic.

Let's again look at the algorithm of getting pointer on per-cpu variable:

* The kernel creates multiply `.data..percpu` sections (ones perc-pu) during initialization process;
* All variables created with the `DEFINE_PER_CPU` macro will be reloacated to the first section or for CPU0;
* `__per_cpu_offset` array filled with the distance (`BOOT_PERCPU_OFFSET`) between `.data..percpu` sections;
* When `per_cpu_ptr` called for example for getting pointer on the certain per-cpu variable for the third CPU, `__per_cpu_offset` array will be accessed, where every index points to the certain CPU.

That's all.
