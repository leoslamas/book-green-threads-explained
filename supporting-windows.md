# Appendix: Supporting Windows

Our example works for both OSX, Linux and Windows, but as I have pointed out, our Windows implementation is not correct even though it's working. Since I've been quite commited to make this work on all three platforms, I'll go through what we need to do in this chapter.

You might wonder why I didn't include this in the original code, and the reason for that is that this is really not at all that relevant for explaining the main concepts I wanted to explore.

### What's special with Windows

The reason I don't consider this important enough to implement in the main example is that that windows has more `callee saved` registers, or `non-volatile`registers as they call it in addition to one rather poorly documented quirk that we need to account for, so what we really do is just to save more data when we do the context switch and that needs more conditional compilation.

{% hint style="info" %}
Conditionally compiling this to support windows correctly bloats our code with almost 50 % without adding much to what we need for a basic understanding.

Now that doesn't mean this isn't interesting, on the contrary, but we'll also experience first hand some of the difficulties of supporting multiple platforms when doing everything from scratch.
{% endhint %}



### Additional callee saved \(non-volatile\) registers

The first thing I mentioned is that windows wants to save more data during context switches, in particular the XMM6-XMM15 registers. It's actually [mentioned specifically in the reference](https://docs.microsoft.com/en-us/cpp/build/x64-software-conventions?view=vs-2019#register-usage) so this is just adding more fields to our `ThreadContext`struct. This is very easy now that we've done it once before.

Our ThreadContext now looks like this:

```rust
#[cfg(target_os="windows")]
#[derive(Debug, Default)]
#[repr(C)] 
struct ThreadContext {
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
    xmm6: u64,
    xmm7: u64,
    xmm8: u64,
    xmm9: u64,
    xmm10: u64,
    xmm11: u64,
    xmm12: u64,
    xmm13: u64,
    xmm14: u64,
    xmm15: u64,
}
```

### The Thread Information Block

The second part is poorly documented. I've actually struggled to verify exactly how skipping this will cause a failure on modern Windows but there's enough references to it around from trustworthy sources that I'm in no doubt we need to go through this.

You see, Windows wants to store some information about the currently running thread in what it calls the `Thread Information Block`, referred to as `NT_TIB`. Specifically it wants access to information about the `Stack Base`and the `Stack Limit`in the `%gs`register.

{% hint style="info" %}
What is the GS register you might ask? 

The answer I found was a bit perplexing. Apparently these segment registers, GS on x64, and FS on x86 was intended by Intel to [allow programs to access many different segments of memory](https://stackoverflow.com/questions/10810203/what-is-the-fs-gs-register-intended-for) that were meant to be part of a persistent virtual store. Modern operating systems doesn't use these registers this way as we can only access our own process memory \(which appear as a "flat" memory to us as programmers\). Back when it wasn't clear that this would be the prevailing model, these registers would allow for different implementations by different operating systems. See the [Wikipedia article on the Multics operating system](https://en.wikipedia.org/wiki/Multics) if you're curious.
{% endhint %}

That means that these segment registers are freely used by operating systems for what they deem appropriate. Windows stores information about the currently running thread in the GS register, and Linux uses these registers for thread local storage. 

When we switch threads, we should provide the information it expects about our [Stack Base and our Stack Limit](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block).

Our `ThreadContext` now looks like this:

{% code-tabs %}
{% code-tabs-item title="ThreadContext" %}
```rust
#[cfg(target_os="windows")]
#[derive(Debug, Default)]
#[repr(C)] 
struct ThreadContext {
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
    xmm6: u64,
    xmm7: u64,
    xmm8: u64,
    xmm9: u64,
    xmm10: u64,
    xmm11: u64,
    xmm12: u64,
    xmm13: u64,
    xmm14: u64,
    xmm15: u64,
    stack_start: u64,
    stack_end: u64,
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% hint style="info" %}
Notice we use the `#[cfg(target_os="windows")]` attribute here on all the Windows specific functions and structs, which mean we need to give our "original" definitions an attribute that makes sure it compiles them for all other targets than Windows: `[cfg(not(target_os="windows"))].`
{% endhint %}

I named the fields `stack_start`and `stack_end`since I find that easier to mentally parse since we know the stack starts on the top and grows downwards to the bottom.

Now to implement this we need to make a change to our `spawn()`function to actually provide this information:

{% code-tabs %}
{% code-tabs-item title="spawn" %}
```rust
    #[cfg(target_os="windows")]
    pub fn spawn(&mut self, f: fn()) {
        let available = self
            .threads
            .iter_mut()
            .find(|t| t.state == State::Available)
            .expect("no available thread.");

        let size = available.stack.len();
        let s_ptr = available.stack.as_mut_ptr();
        unsafe {
            ptr::write(s_ptr.offset((size - 8) as isize) as *mut u64, guard as u64);
            ptr::write(s_ptr.offset((size - 16) as isize) as *mut u64, f as u64);
            available.ctx.rsp = s_ptr.offset((size - 16) as isize) as u64;
            available.ctx.stack_start = s_ptr.offset(size as isize) as u64;
        }
        available.ctx.stack_end = s_ptr as *const u64 as u64;

        available.state = State::Ready;
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

As you see we provide a pointer to the start of our stack and a pointer to the end of our stack.

Last we need to change our `swtich()`function and update our assembly:

```rust
#[cfg(target_os="windows")]
#[naked]
unsafe fn switch(old: *mut ThreadContext, new: *const ThreadContext) {
    asm!("
        mov     $0, %rdi
        mov     %rsp, 0x00(%rdi)
        mov     %r15, 0x08(%rdi)
        mov     %r14, 0x10(%rdi)
        mov     %r13, 0x18(%rdi)
        mov     %r12, 0x20(%rdi)
        mov     %rbx, 0x28(%rdi)
        mov     %rbp, 0x30(%rdi)
        mov     %xmm6, 0x38(%rdi)
        mov     %xmm7, 0x40(%rdi)
        mov     %xmm8, 0x48(%rdi)
        mov     %xmm9, 0x50(%rdi)
        mov     %xmm10, 0x58(%rdi)
        mov     %xmm11, 0x60(%rdi)
        mov     %xmm12, 0x68(%rdi)
        mov     %xmm13, 0x70(%rdi)
        mov     %xmm14, 0x78(%rdi)
        mov     %xmm15, 0x80(%rdi)
        mov     %gs:0x08, %rax     # windows support
        mov     %rax, 0x88(%rdi)   # windows support
        mov     %gs:0x10, %rax     # windows support
        mov     %rax, 0x90(%rdi)   # windows support

        mov     $1, %rsi
        mov     0x00(%rsi), %rsp
        mov     0x08(%rsi), %r15
        mov     0x10(%rsi), %r14
        mov     0x18(%rsi), %r13
        mov     0x20(%rsi), %r12
        mov     0x28(%rsi), %rbx
        mov     0x30(%rsi), %rbp
        mov     0x38(%rsi), %xmm6
        mov     0x40(%rsi), %xmm7
        mov     0x48(%rsi), %xmm8
        mov     0x50(%rsi), %xmm9
        mov     0x58(%rsi), %xmm10
        mov     0x60(%rsi), %xmm11
        mov     0x68(%rsi), %xmm12
        mov     0x70(%rsi), %xmm13
        mov     0x78(%rsi), %xmm14
        mov     0x80(%rsi), %xmm15
        mov     0x88(%rsi), %rax   # windows support
        mov     %rax, %gs:0x08     # windows support
        mov     0x90(%rsi), %rax   # windows support 
        mov     %rax, %gs:0x10     # windows support

        ret
        "
    :
    :"{rdi}"(old), "{rsi}"(new)
    : 
    : "volatile", "alignstack"
    );

}
```

As you see, our code gets just a little bit longer. It's not difficult once you've figured out what to store where, but it does add a lot of code.

{% hint style="warning" %}
You'll notice I've changed the inline assembly slightly here since I've had some issues with the `=m`constraint and compilation in release mode. This code does the same and our original example will be updated once I figure out a solution to why the code will not run as expected in release builds. 

**Meanwhile I'll explain briefly here:**

We use the`{register}`constraint which tells the compiler to put our input variables into a specific register. The `%rdi`and the `%rsi`registers are not randomly chosen, on Linux systems they are the default registers for the first and second argument in a function call. Not strictly needed but a nice convention even though Windows has different default registers for these arguments \(`%rcx`and `%rdx`\).

We also use both our arguments as `inputs`, since we don't really have any output from this function we can avoid the `output`register entirely without any need to worry. However we should enable the `volatile`option to indicate that the assembly has side effects.

Our inline assembly won't let us `mov`from one memory offset to another memory offset so we need to go via a register. I chose the`rax`register \(the default register for the return value\) but could have chosen any general purpose register for this.
{% endhint %}

### Conclusion

So this is all we needed to do. As you see we don't really do anything new here, the difficult part is figuring out how Windows works and what it expects, but now that we have done our job properly we should have a pretty complete context switch for all three platforms.

### Final code

I've collected all the code we need to compile differently for Windows at the bottom so you don't have to read the first 200 lines of code since that is unchanged from what we in the previous chapters. I hope you understand why I chose to dedicate a separate chapter for this.

```rust
#![feature(asm)]
#![feature(naked_functions)]
use std::ptr;

const DEFAULT_STACK_SIZE: usize = 1024 * 1024 * 2;
const MAX_THREADS: usize = 4;
static mut RUNTIME: usize = 0;

pub struct Runtime {
    threads: Vec<Thread>,
    current: usize,
}

#[derive(PartialEq, Eq, Debug)]
enum State {
    Available,
    Running,
    Ready,
}

struct Thread {
    id: usize,
    stack: Vec<u8>,
    ctx: ThreadContext,
    state: State,
}

#[cfg(not(target_os="windows"))]
#[derive(Debug, Default)]
#[repr(C)] 
struct ThreadContext {
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
}

impl Thread {
    fn new(id: usize) -> Self {
        Thread {
            id,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Available,
        }
    }
}

impl Runtime {
    pub fn new() -> Self {
        let base_thread = Thread {
            id: 0,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Running,
        };

        let mut threads = vec![base_thread];
        let mut available_threads: Vec<Thread> = (1..MAX_THREADS).map(|i| Thread::new(i)).collect();
        threads.append(&mut available_threads);

        Runtime {
            threads,
            current: 0,
        }
    }

    pub fn init(&self) {
        unsafe {
            let r_ptr: *const Runtime = self;
            RUNTIME = r_ptr as usize;
        }
    }

    pub fn run(&mut self) -> ! {
        while self.t_yield() {}
        std::process::exit(0);
    }

    fn t_return(&mut self) {
        if self.current != 0 {
            self.threads[self.current].state = State::Available;
            self.t_yield();
        }
    }
    
    fn t_yield(&mut self) -> bool {
        let mut pos = self.current;
        while self.threads[pos].state != State::Ready {
            pos += 1;
            if pos == self.threads.len() {
                pos = 0;
            }
            if pos == self.current {
                return false;
            }
        }

        if self.threads[self.current].state != State::Available {
            self.threads[self.current].state = State::Ready;
        }

        self.threads[pos].state = State::Running;
        let old_pos = self.current;
        self.current = pos;

        unsafe {
            switch(&mut self.threads[old_pos].ctx, &self.threads[pos].ctx);
        }

        true
    }

    #[cfg(not(target_os="windows"))]
     pub fn spawn(&mut self, f: fn()) {
        let available = self
            .threads
            .iter_mut()
            .find(|t| t.state == State::Available)
            .expect("no available thread.");

        let size = available.stack.len();
        let s_ptr = available.stack.as_mut_ptr();
        unsafe {
            ptr::write(s_ptr.offset((size - 8) as isize) as *mut u64, guard as u64);
            ptr::write(s_ptr.offset((size - 16) as isize) as *mut u64, f as u64);
            available.ctx.rsp = s_ptr.offset((size - 16) as isize) as u64;
        }
        available.state = State::Ready;
    }
}

#[cfg_attr(any(target_os="windows", target_os="linux"), naked)]
fn guard() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        let rt = &mut *rt_ptr;
        println!("THREAD {} FINISHED.", rt.threads[rt.current].id);
        rt.t_return();
    };
}

pub fn yield_thread() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_yield();
    };
}

#[cfg(not(target_os = "windows"))]
#[naked]
unsafe fn switch(old: *mut ThreadContext, new: *const ThreadContext) {
    asm!("
        mov     $0, %rdi
        mov     %rsp, 0x00(%rdi)
        mov     %r15, 0x08(%rdi)
        mov     %r14, 0x10(%rdi)
        mov     %r13, 0x18(%rdi)
        mov     %r12, 0x20(%rdi)
        mov     %rbx, 0x28(%rdi)
        mov     %rbp, 0x30(%rdi)

        mov     $1, %rsi
        mov     0x00(%rsi), %rsp
        mov     0x08(%rsi), %r15
        mov     0x10(%rsi), %r14
        mov     0x18(%rsi), %r13
        mov     0x20(%rsi), %r12
        mov     0x28(%rsi), %rbx
        mov     0x30(%rsi), %rbp

        ret
        "
    :
    :"{rdi}"(old), "{rsi}"(new)
    :
    : "volatile", "alignstack"
    );
}

fn main() {
    let mut runtime = Runtime::new();
    runtime.init();
    runtime.spawn(|| {
        println!("THREAD 1 STARTING");
        let id = 1;
        for i in 0..10 {
            println!("thread: {} counter: {}", id, i);
            yield_thread();
        }
    });
    runtime.spawn(|| {
        println!("THREAD 2 STARTING");
        let id = 2;
        for i in 0..15 {
            println!("thread: {} counter: {}", id, i);
            yield_thread();
        }
    });
    runtime.run();
}

// ===== WINDOWS SUPPORT =====
#[cfg(target_os="windows")]
#[derive(Debug, Default)]
#[repr(C)] 
struct ThreadContext {
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
    xmm6: u64,
    xmm7: u64,
    xmm8: u64,
    xmm9: u64,
    xmm10: u64,
    xmm11: u64,
    xmm12: u64,
    xmm13: u64,
    xmm14: u64,
    xmm15: u64,
    stack_start: u64,
    stack_end: u64,
}

impl Runtime {
    #[cfg(target_os="windows")]
    pub fn spawn(&mut self, f: fn()) {
        let available = self
            .threads
            .iter_mut()
            .find(|t| t.state == State::Available)
            .expect("no available thread.");

        let size = available.stack.len();
        let s_ptr = available.stack.as_mut_ptr();
        unsafe {
            ptr::write(s_ptr.offset((size - 8) as isize) as *mut u64, guard as u64);
            ptr::write(s_ptr.offset((size - 16) as isize) as *mut u64, f as u64);
            available.ctx.rsp = s_ptr.offset((size - 16) as isize) as u64;
            available.ctx.stack_start = s_ptr.offset(size as isize) as u64;
        }
        available.ctx.stack_end = s_ptr as *const u64 as u64;

        available.state = State::Ready;
    }
}

// reference: https://probablydance.com/2013/02/20/handmade-coroutines-for-windows/
// Contents of TIB on Windows: https://en.wikipedia.org/wiki/Win32_Thread_Information_Block
#[cfg(target_os="windows")]
#[naked]
unsafe fn switch(old: *mut ThreadContext, new: *const ThreadContext) {
    asm!("
        mov     $0, %rdi
        mov     %rsp, 0x00(%rdi)
        mov     %r15, 0x08(%rdi)
        mov     %r14, 0x10(%rdi)
        mov     %r13, 0x18(%rdi)
        mov     %r12, 0x20(%rdi)
        mov     %rbx, 0x28(%rdi)
        mov     %rbp, 0x30(%rdi)
        mov     %xmm6, 0x38(%rdi)
        mov     %xmm7, 0x40(%rdi)
        mov     %xmm8, 0x48(%rdi)
        mov     %xmm9, 0x50(%rdi)
        mov     %xmm10, 0x58(%rdi)
        mov     %xmm11, 0x60(%rdi)
        mov     %xmm12, 0x68(%rdi)
        mov     %xmm13, 0x70(%rdi)
        mov     %xmm14, 0x78(%rdi)
        mov     %xmm15, 0x80(%rdi)
        mov     %gs:0x08, %rax     # windows support
        mov     %rax, 0x88(%rdi)   # windows support
        mov     %gs:0x10, %rax     # windows support
        mov     %rax, 0x90(%rdi)   # windows support

        mov     $1, %rsi
        mov     0x00(%rsi), %rsp
        mov     0x08(%rsi), %r15
        mov     0x10(%rsi), %r14
        mov     0x18(%rsi), %r13
        mov     0x20(%rsi), %r12
        mov     0x28(%rsi), %rbx
        mov     0x30(%rsi), %rbp
        mov     0x38(%rsi), %xmm6
        mov     0x40(%rsi), %xmm7
        mov     0x48(%rsi), %xmm8
        mov     0x50(%rsi), %xmm9
        mov     0x58(%rsi), %xmm10
        mov     0x60(%rsi), %xmm11
        mov     0x68(%rsi), %xmm12
        mov     0x70(%rsi), %xmm13
        mov     0x78(%rsi), %xmm14
        mov     0x80(%rsi), %xmm15
        mov     0x88(%rsi), %rax   # windows support
        mov     %rax, %gs:0x08     # windows support
        mov     0x90(%rsi), %rax   # windows support 
        mov     %rax, %gs:0x10     # windows support

        ret
        "
    :
    :"{rdi}"(old), "{rsi}"(new)
    : 
    : "volatile", "alignstack"
    );

}

```

