---
title: 循序渐进学习BPF
date: 2021-11-2 19:55:50
tags: bpf
layout: post
---



# 循序渐进学习BPF

本文通过简单、分步的方法使您能够将BPF跟踪技术应用于实际问题，而不需要专门的工具或库。

BPF是Linux内核中用于网络堆栈跟踪的一种跟踪技术，最近由于新的扩展而变得流行起来，这些扩展支持BPF原来范围之外的新用例。今天，它可以用来实现程序性能分析工具、系统和程序动态跟踪工具，以及更多场景。



在这篇文章中，将向您展示如何使用BPF的Linux实现来编写访问系统和程序事件的工具。
来自IO Visor的优秀工具使用户能够轻松地利用BPF技术，而无需投入大量时间用本地代码语言编写专门的工具。



## 什么是BPF

**BPF本身只是一种表达程序的方式，以及用于安全执行程序的运行时解释器。**
它是一组虚拟架构规范，详细描述了用于运行其代码的虚拟机的行为。
BPF的最新扩展不仅引入了新的、真正有用的helper函数(如读取进程内存)，而且还引入了新的寄存器和更多的BPF字节码堆栈空间。



## BPF 程序的限制

即使我们不需要手写BPF程序集，了解代码限制也是很有用的，因为如果我们违反了内核验证器的规则，它将拒绝我们的指令。

BPF程序非常简单，只有一个函数。指令以字节码数组的形式发送到内核，这意味着不涉及可执行文件格式。没有sections，就不可能有全局变量或字符串字面值; 所有东西都必须在堆栈上，堆栈最多只能容纳512字节。分支是允许的，但只有在内核版本5.3之后，只要验证者能够证明代码不会永远执行，跳转字节码才可以执行。

```
 uname -a
Linux ubuntu 5.4.0-58-generic #64-Ubuntu SMP Wed Dec 9 08:16:25 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```

使用循环而不需要使用最新内核版本的唯一另一种方法是将其展开，但是这可能会使用大量的指令，并且较老的Linux版本将不会加载任何超过4096字节码计数限制的程序(参见Linux /bpf_common.h下的BPF_MAXINSNS)。
错误处理在某些情况下是强制性的，并且验证者会阻止你使用那些可能会因为拒绝程序而导致初始化失败的资源。

```
cat /usr/include/linux/bpf_common.h |tail
#ifndef BPF_MAXINSNS
#define BPF_MAXINSNS 4096
#endif
```

这些限制非常重要，因为这些程序可能与内核代码挂钩。当验证器质疑代码的正确性时，可以防止系统崩溃或加载错误代码导致的速度变慢。



## 外部资源

为了使BPF程序真正有用，它们需要通过一些方法与用户模式进程通信并管理长期数据，例如，通过映射和perf事件输出。尽管存在许多映射类型，但它们本质上都类似于键值数据库，通常用于在用户模式和/或其他程序之间共享数据。其中一些类型将数据存储在每个CPU的存储中，当同一个BPF程序从不同的CPU内核并发运行时，可以很容易地保存和检索状态。

Perf事件输出通常用于将数据发送到用户模式程序和服务，并实现为循环缓冲区。



## 事件源

如果没有一些数据要处理，我们的程序就会无所事事。Linux上的BPF探测可以附加到几个不同的事件源。出于我们的目的，我们主要对函数跟踪事件感兴趣。



### 动态插桩

类似于代码挂钩，BPF程序可以附加到任何函数。探测类型取决于目标代码所在的位置。在跟踪内核函数时使用Kprobes，而在处理用户模式库或二进制文件时使用uprobe。

当进入被监视的函数时，会触发Kprobe和Uprobe事件，而当函数返回时，会生成Kretprobe和Uretprobe事件。即使被跟踪的函数有多个出口点，这也能正常工作。这类事件不转发类型化的系统调用参数，只附带一个pt_regs结构，该结构包含调用时的寄存器值。要将函数参数映射回正确的寄存器，需要了解函数原型和系统ABI。



### 静态插桩

在编写工具时，依赖函数挂钩并不总是理想的，因为随着内核或软件的更新，破坏的风险会增加。在大多数情况下，最好使用更稳定的事件源，如跟踪点。有两种类型的跟踪点:一种用于用户模式代码(USDT，又称用户级静态定义跟踪点)，一种用于内核模式代码(有趣的是，它们仅仅被称为跟踪点)。

两种类型的跟踪点都是由程序员在源代码中定义的，本质上定义了一个稳定的接口，除非严格必要，否则不应该更改。

如果已启用并挂载DebugFS，则注册的跟踪点将全部出现在/sys/kernel/debug/tracing文件夹下。与Kprobes和Kretprobes类似，Linux内核中定义的每个系统调用都带有两个不同的跟踪点。第一个是sys_enter，每当系统中的程序转换到内核中的系统调用处理程序时，它就被激活，并携带有关已接收到的参数的信息。第二个(也是最后一个)是sys_exit，它只包含函数的退出代码，并且在syscall函数终止时被调用。



## BPF 开发环境

即使没有使用外部库的计划，我们仍然有一些依赖关系。最重要的事情是能够访问最新的支持BPF编译的LLVM工具链（如果是10以上版本，要在对应文件下改动一下）。您还需要CMake，因为这是我在示例代码中使用的。

当在BPF环境中运行时，我们的程序使用特殊的helper函数，这些函数需要至少高于4.18的内核版本。虽然可以避免使用它们，但这将严重限制我们在代码中所能做的事情。（我用的是5.4，4.18以下可能没有本文用到的部分helper函数）。



## 写第一个程序



有许多新概念需要引入，因此我们将从简单的开始:我们的第一个示例加载一个不做任何操作就返回的程序。

首先，我们创建一个新的LLVM模块和一个包含我们的逻辑的函数。

```C++
std::unique_ptr createBPFModule(llvm::LLVMContext &context) {
  auto module = std::make_unique("BPFModule", context);
  module->setTargetTriple("bpf-pc-linux");
  module->setDataLayout("e-m:e-p:64:64-i64:64-n32:64-S128");
  
  return module;
}
  
std::unique_ptr generateBPFModule(llvm::LLVMContext &context) {
  // Create the LLVM module for the BPF program
  auto module = createBPFModule(context);
  
  // BPF programs are made of a single function; we don't care about parameters
  // for the time being
  llvm::IRBuilder<> builder(context);
  auto function_type = llvm::FunctionType::get(builder.getInt64Ty(), {}, false);
  
  auto function = llvm::Function::Create(
      function_type, llvm::Function::ExternalLinkage, "main", module.get());  
      
  // Ask LLVM to put this function in its own section, so we can later find it
  // more easily after we have compiled it to BPF code
  function->setSection("bpf_main_section");
  
  // Create the entry basic block and assemble the printk code using the helper
  // we have written
  auto entry_bb = llvm::BasicBlock::Create(context, "entry", function);
  
  builder.SetInsertPoint(entry_bb);
  builder.CreateRet(builder.getInt64(0));
  
  return module;
}
```

因为我们不打算处理事件参数，所以我们创建的函数不接受任何参数。除了返回指令，这里没有发生太多其他事情。记住，每个BPF程序只有一个函数，所以最好让LLVM将它们存储在单独的部分中。这使得在模块编译后检索它们变得更容易。

现在我们可以使用LLVM中的ExecutionEngine类将模块JIT为BPF字节码

```c++

SectionMap compileModule(std::unique_ptr module) {
  // Create a new execution engine builder and configure it
  auto exec_engine_builder =
      std::make_unique(std::move(module));
  
  exec_engine_builder->setMArch("bpf");
  
  SectionMap section_map;
  exec_engine_builder->setMCJITMemoryManager(
      std::make_unique(section_map));
      
  // Create the execution engine and build the given module
  std::unique_ptr execution_engine(
      exec_engine_builder->create());
      
  execution_engine->setProcessAllSections(true);
  execution_engine->finalizeObject();
  
  return section_map;
}
```

我们自定义的SectionMemoryManager类主要充当从LLVM到原始SectionMemoryManager类的传递，它只是用来跟踪ExecutionEngine对象在编译IR时创建的section。

一旦构建了代码，我们就会为模块中创建的每个函数返回一个字节向量

```C++
int loadProgram(const std::vector &program) {
  // The program needs to be aware how it is going to be used. We are
  // only interested in tracepoints, so we'll hardcode this value
  union bpf_attr attr = {};
  attr.prog_type = BPF_PROG_TYPE_TRACEPOINT;
  attr.log_level = 1U;
  
  // This is the array of (struct bpf_insn) instructions we have received
  // from the ExecutionEngine (see the compileModule() function for more
  // information)
  auto instruction_buffer_ptr = program.data();
  std::memcpy(&attr.insns, &instruction_buffer_ptr, sizeof(attr.insns));
  
  attr.insn_cnt =
      static_cast(program.size() / sizeof(struct bpf_insn));
      
  // The license is important because we will not be able to call certain
  // helpers within the BPF VM if it is not compatible
  static const std::string kProgramLicense{"GPL"};
  
  auto license_ptr = kProgramLicense.c_str();
  std::memcpy(&attr.license, &license_ptr, sizeof(attr.license));
  
  // The verifier will provide a text disasm of our BPF program in here.
  // If there is anything wrong with our code, we'll also find some
  // diagnostic output
  std::vector log_buffer(4096, 0);
  attr.log_size = static_cast<__u32>(log_buffer.size());
  
  auto log_buffer_ptr = log_buffer.data();
  std::memcpy(&attr.log_buf, &log_buffer_ptr, sizeof(attr.log_buf));
  
  auto program_fd =
      static_cast(::syscall(__NR_bpf, BPF_PROG_LOAD, &attr, sizeof(attr)));
      
  if (program_fd < 0) {
    std::cerr << "Failed to load the program: " << log_buffer.data() << "\n";
  }
  
  return program_fd;
}
```

加载程序并不困难，但您可能已经注意到，我们正在使用的bpf()系统调用没有定义helper函数。跟踪点是最容易设置的事件类型，这也是我们这一次要用到的。

一旦发出BPF_PROG_LOAD命令，内核中的验证器将验证我们的程序，并在我们提供的日志缓冲区中提供它的反汇编。如果内核输出比可用字节长，操作将失败，所以如果加载已经失败，只在生产代码中提供日志缓冲区。

现在我们可以使用所构建的helper来组装main()函数:

```C++
int main() {
  initializeLLVM();
  
  // Generate our BPF program
  llvm::LLVMContext context;
  auto module = generateBPFModule(context);
  
  // JIT the module to BPF code using the execution engine
  auto section_map = compileModule(std::move(module));
  if (section_map.size() != 1U) {
    std::cerr << "Unexpected section count\n";
    return 1;
  }
  
  // We have previously asked LLVM to create our function inside a specific
  // section; get our code back from it and load it
  const auto &main_program = section_map.at("bpf_main_section");
  auto program_fd = loadProgram(main_program);
  if (program_fd < 0) {
    return 1;
  }
  
  releaseLLVM();
  return 0;
}
```

