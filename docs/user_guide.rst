User Guide
==========

Quick Start
-----------

.. include:: ../README.rst
    :start-after: start-quick-start
    :end-before: end-quick-start

Core Concepts
-------------

.. highlight:: pycon

The most important interfaces in drgn are *programs*, *objects*, and *helpers*.

Programs
^^^^^^^^

A program being debugged is represented by an instance of the
:class:`drgn.Program` class. The drgn CLI is initialized with a ``Program``
named ``prog``; unless you are using the drgn library directly, this is usually
the only ``Program`` you will need.

A ``Program`` is used to look up type definitions, access variables, and read
arbitrary memory::


    >>> prog.type("struct list_head")
    struct list_head {
            struct list_head *next;
            struct list_head *prev;
    }
    >>> prog["jiffies"]
    (volatile unsigned long)4416739513
    >>> prog.read(0xffffffffbe411e10, 16)
    b'swapper/0\x00\x00\x00\x00\x00\x00\x00'

The :meth:`drgn.Program.type()`, :meth:`drgn.Program.variable()`,
:meth:`drgn.Program.constant()`, and :meth:`drgn.Program.function()` methods
look up those various things in a program. :meth:`drgn.Program.read()` reads
memory from the program's address space. The :meth:`[]
<drgn.Program.__getitem__>` operator looks up a variable, constant, or
function::

    >>> prog["jiffies"] == prog.variable("jiffies")
    True

It is usually more convenient to use the ``[]`` operator rather than the
``variable()``, ``constant()``, or ``function()`` methods unless the program
has multiple objects with the same name, in which case the methods provide more
control.

Objects
^^^^^^^

Variables, constants, functions, and computed values are all called *objects*
in drgn. Objects are represented by the :class:`drgn.Object` class. An object
may exist in the memory of the program (a *reference*)::

    >>> Object(prog, 'int', address=0xffffffffc09031a0)

Or, an object may be a constant or temporary computed value (a *value*)::

    >>> Object(prog, 'int', value=4)

What makes drgn scripts expressive is that objects can be used almost exactly
like they would be in the program's own source code. For example, structure
members can be accessed with the dot (``.``) operator, arrays can be
subscripted with ``[]``, arithmetic can be performed, and objects can be
compared::

    >>> print(prog["init_task"].comm[0])
    (char)115
    >>> print(repr(prog["init_task"].nsproxy.mnt_ns.mounts + 1))
    Object(prog, 'unsigned int', value=34)
    >>> prog["init_task"].nsproxy.mnt_ns.pending_mounts > 0
    False

Python doesn't have all of the operators that C or C++ do, so some
substitutions are necessary:

* Instead of ``*ptr``, dereference a pointer with :meth:`ptr[0]
  <drgn.Object.__getitem__>`.
* Instead of ``ptr->member``, access a member through a pointer with
  :meth:`ptr.member <drgn.Object.__getattr__>`.
* Instead of ``&var``, get the address of a variable with
  :meth:`var.address_of_() <drgn.Object.address_of_>`.

A common use case is converting a ``drgn.Object`` to a Python value so it can
be used by a standard Python library. There are a few ways to do this:

* The :meth:`drgn.Object.value_()` method gets the value of the object with the
  directly corresponding Python type (i.e., integers and pointers become
  ``int``, floating-point types become ``float``, booleans become ``bool``,
  arrays become ``list``, structures and unions become ``dict``).
* The :meth:`drgn.Object.string_()` method gets a null-terminated string as
  ``bytes`` from an array or pointer.
* The :class:`int() <int>`, :class:`float() <float>`, and :class:`bool()
  <bool>` functions do an explicit conversion to that Python type.

Objects have several attributes; the most important are
:attr:`drgn.Object.prog_` and :attr:`drgn.Object.type_`. The former is the
:class:`drgn.Program` that the object is from, and the latter is the
:class:`drgn.Type` of the object.

Note that all attributes and methods of the ``Object`` class end with an
underscore (``_``) in order to avoid conflicting with structure or union
members. The ``Object`` attributes and methods always take precedence; use
:meth:`drgn.Object.member_()` if there is a conflict.

References vs. Values
"""""""""""""""""""""

The main difference between reference objects and value objects is how they are
evaluated. References are read from the program's memory every time they are
evaluated::

    >>> import time
    >>> jiffies = prog["jiffies"]
    >>> jiffies.value_()
    4391639989
    >>> time.sleep(1)
    >>> jiffies.value_()
    4391640290

Values simply return the stored value (:meth:`drgn.Object.read_()` reads a
reference object and returns it as a value object)::

    >>> jiffies2 = jiffies.read_()
    >>> jiffies2.value_()
    4391640291
    >>> time.sleep(1)
    >>> jiffies2.value_()
    4391640291
    >>> jiffies.value_()
    4391640593

References have a :attr:`drgn.Object.address_` attribute, which is the object's
address as a Python ``int``::

    >>> address = prog["jiffies"].address_
    >>> type(address)
    <class 'int'>
    >>> print(hex(address))
    0xffffffffbe405000

This is slightly different from the :meth:`drgn.Object.address_of_()` method,
which returns the address as a ``drgn.Object``::

    >>> jiffiesp = prog["jiffies"].address_of_()
    >>> print(repr(jiffiesp))
    Object(prog, 'volatile unsigned long *', value=0xffffffffbe405000)
    >>> print(hex(jiffiesp.value_()))
    0xffffffffbe405000

Of course, both references and values can have a pointer type;
``address_`` refers to the address of the pointer object itself, and
:meth:`drgn.Object.value_()` refers to the value of the pointer (i.e., the
address it points to).

.. _absent-objects:

Absent Objects
""""""""""""""

In addition to reference objects and value objects, objects may also be
*absent*.

    >>> Object(prog, "int").value_()
    Traceback (most recent call last):
      File "<console>", line 1, in <module>
    _drgn.ObjectAbsentError: object absent

This represents an object whose value or address is not known. For example,
this can happen if the object was optimized out of the program by the compiler.

Any attempt to operate on an absent object results in a
:exc:`drgn.ObjectAbsentError` exception, although basic information including
its type may still be accessed.

Helpers
^^^^^^^

Some programs have common data structures that you may want to examine. For
example, consider linked lists in the Linux kernel:

.. code-block:: c

    struct list_head {
        struct list_head *next, *prev;
    };

    #define list_for_each(pos, head) \
        for (pos = (head)->next; pos != (head); pos = pos->next)

When working with these lists, you'd probably want to define a function:

.. code-block:: python3

    def list_for_each(head):
        pos = head.next
        while pos != head:
            yield pos
            pos = pos.next

Then, you could use it like so for any list you need to look at::

    >>> for pos in list_for_each(head):
    ...     do_something_with(pos)

Of course, it would be a waste of time and effort for everyone to have to
define these helpers for themselves, so drgn includes a collection of helpers
for many use cases. See :doc:`helpers`.

Other Concepts
--------------

In addition to the core concepts above, drgn provides a few additional
abstractions.

Threads
^^^^^^^

The :class:`drgn.Thread` class represents a thread.
:meth:`drgn.Program.threads()`, :meth:`drgn.Program.thread()`,
:meth:`drgn.Program.main_thread()`, and :meth:`drgn.Program.crashed_thread()`
can be used to find threads::

    >>> for thread in prog.threads():
    ...     print(thread.tid)
    ...
    39143
    39144
    >>> print(prog.main_thread().tid)
    39143
    >>> print(prog.crashed_thread().tid)
    39144

Stack Traces
^^^^^^^^^^^^

drgn represents stack traces with the :class:`drgn.StackTrace` and
:class:`drgn.StackFrame` classes. :func:`drgn.stack_trace()`,
:meth:`drgn.Program.stack_trace()`, and :meth:`drgn.Thread.stack_trace()`
return the call stack for a thread::

    >>> trace = stack_trace(115)
    >>> trace
    #0  context_switch (./kernel/sched/core.c:4683:2)
    #1  __schedule (./kernel/sched/core.c:5940:8)
    #2  schedule (./kernel/sched/core.c:6019:3)
    #3  schedule_hrtimeout_range_clock (./kernel/time/hrtimer.c:2148:3)
    #4  poll_schedule_timeout (./fs/select.c:243:8)
    #5  do_poll (./fs/select.c:961:8)
    #6  do_sys_poll (./fs/select.c:1011:12)
    #7  __do_sys_poll (./fs/select.c:1076:8)
    #8  __se_sys_poll (./fs/select.c:1064:1)
    #9  __x64_sys_poll (./fs/select.c:1064:1)
    #10 do_syscall_x64 (./arch/x86/entry/common.c:50:14)
    #11 do_syscall_64 (./arch/x86/entry/common.c:80:7)
    #12 entry_SYSCALL_64+0x7c/0x15b (./arch/x86/entry/entry_64.S:113)
    #13 0x7f3344072af7

The :meth:`[] <drgn.StackTrace.__getitem__>` operator on a ``StackTrace`` gets
the ``StackFrame`` at the given index::

    >>> trace[5]
    #5 at 0xffffffff8a5a32d0 (do_sys_poll+0x400/0x578) in do_poll at ./fs/select.c:961:8 (inlined)

The :meth:`[] <drgn.StackFrame.__getitem__>` operator on a ``StackFrame`` looks
up an object in the scope of that frame. :meth:`drgn.StackFrame.locals()`
returns a list of the available names::

    >>> prog["do_poll"]
    (int (struct poll_list *list, struct poll_wqueues *wait, struct timespec64 *end_time))0xffffffff905c6e10
    >>> trace[5].locals()
    ['list', 'wait', 'end_time', 'pt', 'expire', 'to', 'timed_out', 'count', 'slack', 'busy_flag', 'busy_start', 'walk', 'can_busy_loop']
    >>> trace[5]["list"]
    *(struct poll_list *)0xffffacca402e3b50 = {
            .next = (struct poll_list *)0x0,
            .len = (int)1,
            .entries = (struct pollfd []){},
    }

Symbols
^^^^^^^

The symbol table of a program is a list of identifiers along with their address
and size. drgn represents symbols with the :class:`drgn.Symbol` class, which is
returned by :meth:`drgn.Program.symbol()`.

Types
^^^^^

drgn automatically obtains type definitions from the program. Types are
represented by the :class:`drgn.Type` class and created by various factory
functions like :meth:`drgn.Program.int_type()`::

    >>> prog.type("int")
    prog.int_type(name='int', size=4, is_signed=True)

You won't usually need to work with types directly, but see
:ref:`api-reference-types` if you do.

Modules
^^^^^^^

drgn tracks executables, shared libraries, loadable kernel modules, and other
binary files used by a program with the :class:`drgn.Module` class. Modules
store their name, identifying information, load address, and debugging symbols.

.. code-block:: pycon
    :caption: Linux kernel example

    >>> for module in prog.modules():
    ...     print(module)
    ...
    prog.main_module(name='kernel')
    prog.relocatable_module(name='rng_core', address=0xffffffffc0400000)
    prog.relocatable_module(name='virtio_rng', address=0xffffffffc0402000)
    prog.relocatable_module(name='binfmt_misc', address=0xffffffffc0401000)
    >>> prog.main_module().debug_file_path
    '/usr/lib/modules/6.13.0-rc1-vmtest34.1default/build/vmlinux'

.. code-block:: pycon
    :caption: Userspace example

    >>> for module in prog.modules():
    ...     print(module)
    ...
    prog.main_module(name='/usr/bin/grep')
    prog.shared_library_module(name='/lib64/ld-linux-x86-64.so.2', dynamic_address=0x7f51772b6e68)
    prog.shared_library_module(name='/lib64/libc.so.6', dynamic_address=0x7f51771af960)
    prog.shared_library_module(name='/lib64/libpcre2-8.so.0', dynamic_address=0x7f5177258c68)
    prog.vdso_module(name='linux-vdso.so.1', dynamic_address=0x7f51772803e0)
    >>> prog.main_module().loaded_file_path
    '/usr/bin/grep'
    >>> prog.main_module().debug_file_path
    '/usr/lib/debug/usr/bin/grep-3.11-7.fc40.x86_64.debug'

drgn normally initializes the appropriate modules and loads their debugging
symbols automatically. Advanced use cases can create or modify modules and load
debugging symbols manually; see the :ref:`advanced usage guide
<advanced-modules>`.

Platforms
^^^^^^^^^

Certain operations and objects in a program are platform-dependent; drgn allows
accessing the platform that a program runs with the :class:`drgn.Platform`
class.

Command Line Interface
----------------------

The drgn CLI is basically a wrapper around the drgn library which automatically
creates a :class:`drgn.Program`. The CLI can be run in interactive mode or
script mode.

Script Mode
^^^^^^^^^^^

Script mode is useful for reusable scripts. Simply pass the path to the script
along with any arguments:

.. code-block:: console

    $ cat script.py
    import sys
    from drgn.helpers.linux import find_task

    pid = int(sys.argv[1])
    uid = find_task(pid).cred.uid.val.value_()
    print(f"PID {pid} is being run by UID {uid}")
    $ sudo drgn script.py 601
    PID 601 is being run by UID 1000

It's even possible to run drgn scripts directly with the proper `shebang
<https://en.wikipedia.org/wiki/Shebang_(Unix)>`_::

    $ cat script2.py
    #!/usr/bin/env drgn

    mounts = prog["init_task"].nsproxy.mnt_ns.mounts.value_()
    print(f"You have {mounts} filesystems mounted")
    $ sudo ./script2.py
    You have 36 filesystems mounted

.. _interactive-mode:

Interactive Mode
^^^^^^^^^^^^^^^^

Interactive mode uses the Python interpreter's `interactive mode
<https://docs.python.org/3/tutorial/interpreter.html#interactive-mode>`_ and
adds a few nice features, including:

* History
* Tab completion
* Automatic import of relevant helpers
* Pretty printing of objects and types

The default behavior of the Python `REPL
<https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop>`_ is to
print the output of :func:`repr()`. For :class:`drgn.Object` and
:class:`drgn.Type`, this is a raw representation::

    >>> print(repr(prog["jiffies"]))
    Object(prog, 'volatile unsigned long', address=0xffffffffbe405000)
    >>> print(repr(prog.type("atomic_t")))
    prog.typedef_type(name='atomic_t', type=prog.struct_type(tag=None, size=4, members=(TypeMember(prog.type('int'), name='counter', bit_offset=0),)))

The standard :func:`print()` function uses the output of :func:`str()`. For
drgn objects and types, this is a representation in programming language
syntax::

    >>> print(prog["jiffies"])
    (volatile unsigned long)4395387628
    >>> print(prog.type("atomic_t"))
    typedef struct {
            int counter;
    } atomic_t

In interactive mode, the drgn CLI automatically uses ``str()`` instead of
``repr()`` for objects and types, so you don't need to call ``print()``
explicitly::

    $ drgn
    >>> prog["jiffies"]
    (volatile unsigned long)4395387628
    >>> prog.type("atomic_t")
    typedef struct {
            int counter;
    } atomic_t

Next Steps
----------

Follow along with a :doc:`tutorial <tutorials>` or :doc:`case study
<case_studies>`. Refer to the :doc:`api_reference` and look through the
:doc:`helpers`. Browse through the `tools
<https://github.com/osandov/drgn/tree/main/tools>`_. Check out the `community
contributions <https://github.com/osandov/drgn/tree/main/contrib>`_.
