理解 Unix 进程
=================

### 1. 进程：Unix 之本
  进程乃 Unix 系统的基石，所有的代码都是在进程中执行的。

### 2. pid: 进程的唯一标识符。

### 3. 进程皆有父:
  - 系统中运行的每一个进程都有对应的父进程。每个进程都知道其父进 程的标识符(称为 ppid)。

### 4. 文件描述符：代表已经打开的文件(资源)
  - Unix 中，一切皆是文件，套接字，管道，文件等都是。
  - 无论何时在进程中打开一个资源，你都会获得一个文件描述符编号 (file descriptor number)。文件描述符并不会在无关进程之间共享，它只存在于其所属的进程之中。
  - 进程打开的所有资源都会获得一个用于标识的唯一数字。这便是内核 跟踪进程所用资源的方法。

### 5. 标准流: STDIN, STDOUT, STDERR
  - 对应的文件描述符
  - STDIN => 0
  - STDOUT => 1
  - STDERR => 2

### 6. 进程皆有资源限制
  - 一个进程能打开的文件是有限的, 通过 getrlimit(2) 获取
  - 软限制 【可通过 setrlimit(2) 修改，超过会抛出异常】
  - 硬限制

### 7. 进程皆有环境
  - 环境变量【包含进程数据的键-值对】
  - 所有进程都从其父进程处继承环境变量
  - MESSAGE='wing it'ruby -e "puts ENV['MESSAGE']"
  - RAILS_ENV=production rails server
  - 比起解析命令行选项，使用环境变量的开销通常更小一些

### 8. 进程皆有参数
  - 所有进程都可以访问名为 ARGV 的特殊数组

### 9. 进程皆有名
  - 进程名的妙处在于它可以在运行期间被修改并作为一种通信手段
  - 可通过访问 $PROGRAM_NAME 获取当前进程的名称

### 10. 进程皆有推出码
  - 所有进程在退出的时候都带有数字退出码(0-255)，用于指明进程是否顺利结束
  - 尽管退出码通常用来表明不同的错误，它们其实是一种通信途径。你只需以适合自己程序的方式来处理各种进程退出码，便打破了传统。
    ```ruby
      #这将使你的程序携带顺利状态码(0)退出。
        exit

      #你可以给这个方法传递一个定制的退出码。
        exit 22

      # 当 Kernel#exit 被调用时，在退出之前，
      # Ruby会调用由 Kernel#at_exit #所定义的全部语句块。
        at_exit { puts'Last!' }

        exit

      # 输出 => Last!

      # Kernel#raise 不会立刻结束进程。
      # 它只是抛出一个异常，该异常会沿着调用栈向上传递并可能会得到处理。
      # 如果没有代码对其进行处理，那么这个未处理的异常将会终结该进程。
    ```

### 11. 进程皆可衍生
  - Luke, fork
    fork(2)系统调用 允许运行中的进程以编程的形式创建新的进程。这个新进程和原始进程一模一样。
  - 子进程从父进程处继承了其所占用内存中的所有内容，以及所有属于父进程的已打开的文件描述符。
  - 子进程是一个全新的进程，所以它拥有自己唯一的 pid。

    ```ruby
      if fork
        puts "entered the if block"
      else
        puts "entered the else block"
      end

      # => 输出：
        # entered the if block
        # entered the else block
    ```
  - 原因：
    - fork() 的返回值在父进程中是子进程的 pid, 而在子进程中为 nil。

    - fork() 的一次调用实际上返回了两次。 记住，fork 创造了一个新进程。
          
    - 所以它在调用进程(父进程)中返回一次，在新创建的进程(子进程)中又返回一次。

  - 使用 block
    ```ruby
      fork do
        # 此处的代码仅在子进程中执行
      end

        # 此处的代码仅在父进程中执行
    ```

### 12. 孤儿进程
  - 当父进程结束后，子进程(成为孤儿进程)照常继续运行，父进程不会带着子进程同归于尽。

### 13. 友好的进程
  - CoW (copy-on-write): 写时复制
  - CoW 将实际的内存复制操作推迟到了真正需要写入的时候。
  - 所以说父进程和子进程实际上是在共享内存中的数据，
    直到它们其中的某一个需要对数据进行修改，届时才会进行内存复制，使得两个进程保持适当的隔离。
    ```ruby
      arr = [1,2,3]
      fork do
        # 此时子进程已经完成初始化。
        # 借助CoW，子进程并不需要复制变量arr，因为它并没有修改任何共享变量。
        # 因此可以继续从和父进程同样的内存位置中进行读取。
        p arr 
      end

      arr = [1,2,3]
      fork do
        # 此时子进程已经完成初始化。
        # 由于CoW，变量arr并不会被复制。
        arr << 4
        # 上面的代码修改了数组，因此在进行修改之前需要为子进程创建一个该数组的副本。
        # 父进程中的这个数组并不会受到影响。
      end
    ```

### 14. 进程可待
  - 看顾（Babysitting）
    ```ruby
      fork do
        5.times do
          sleep 1
          puts "I am an orphan!"
        end
      end

      Process.wait
      abort "Parent process died..."

      # => Output
        # I am an orphan!
        # I am an orphan!
        # I am an orphan!
        # I am an orphan!
        # I am an orphan!
        # Parent process died...
        
      # 它是一个阻塞调用，该调用使得父进程一直等到它的某个子进程退出之后才继续执行。
    ```

  - Process.wait 一家子
    - 如果父进程拥有不止一个子进程， 并且使用了 Process.wait，那么你就需要知道究竟是哪个子进程退出了。
    - 这时可以使用返回值来解决这个问题。
    ```ruby
    #创建 3 个子进程。
    3.times do
      fork do
        # 每个子进程随机休眠一段时间(不超过 5 秒)
        sleep rand(5)
      end
    end

    # 等待每个子进程退出并打印其返回的pid。
    3.times do
      puts Process.wait
    end
    ```
  - 使用 Process.wait2 进行通信
    ```ruby
      Process.wait  #=> pid                 只返回进程标示符
      Process.wait2 #=> [pid, status]       返回子进程的标示符和退出状态玛

      # 从 Process.wait2 返回的 status 是 Process::Status 的一个实例。
      # 它包含大量有用的信息，可让我们获知某个进程是如何退出的。
    ```

    ```ruby
      # 创建5个子进程。
      5.times do
        fork do
          # 每个子进程生成一个随机数。如果是偶数，就以退出码 111 退出，否则以
          # 退出码 112 退出。
          if rand(5).even?
            exit 111
          else
            exit 112
          end
        end
      end

      5.times do
        # 等待子进程逐个退出。
        pid, status = Process.wait2

        # 如果子进程的退出码是111，那么我们就知道它生成的是偶数。
        if status.exitstatus == 111
          puts "#{pid} encountered an even number!"
        else
          puts "#{pid} encountered an odd number!"
        end
      end

      # 进程间的通信既不需要文件系统，也不需要网络!
    ```

  - 等待特定的子进程
    ```ruby
      # 只等待由 pid 指定的子进程
      Process.waitpid
      Process.waitpid2

      # 例子
      favourite = fork do
        exit 77
      end

      middle_child = fork do
        abort "I want to be waited on!"
      end

      pid, status = Process.waitpid2 favourite
      puts status.exitstatus

      # extra
      # Process.waitpid 123  <= 等价 => Process.wait 123
      # Process.wait2 和 Process.waitpid2 同理
      # process.wait(-1) 表示等待任意子进程
    ```

  - 竞争条件
    - 如果父进程还没来得及从 Process.wait 返回，另一个子进程也退出了，这会怎样?
      ```ruby
        # 创建两个子进程。
        2.times do
          fork do
            # 两个子进程立即退出。
            abort "Finished!"
          end
        end

        # 父进程等待第一个子进程，然后休眠 5 秒钟。
        # 其间第二个子进程也退出，不再运行。
        puts Process.wait
        sleep 5

        # 父进程会再等待一次，让人惊叹的是，第二个子进程的退出信息会被加入队列并在此返回。
        puts Process.wait

        # 这项技术能够避免竞争条件。
        # 内核将退出的进程信息加入进程可待队列，这样一来父进程就总是能够依照子进程退出的顺序接收到信息。
      ```