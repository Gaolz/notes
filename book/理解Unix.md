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
  - 软限制 【可通过 ```setrlimit(2)``` 修改，超过会抛出异常】
  - 硬限制

### 7. 进程皆有环境
  - 环境变量【包含进程数据的键-值对】
  - 所有进程都从其父进程处继承环境变量
  - ```$ MESSAGE='wing it'ruby -e "puts ENV['MESSAGE']" ```
  - ```$ RAILS_ENV=production rails server```
  - 比起解析命令行选项，使用环境变量的开销通常更小一些

### 8. 进程皆有参数
  - 所有进程都可以访问名为 ARGV 的特殊数组

### 9. 进程皆有名
  - 进程名的妙处在于它可以在运行期间被修改并作为一种通信手段
  - 可通过访问 ```$PROGRAM_NAME``` 获取当前进程的名称

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
    - fork(2)系统调用 允许运行中的进程以编程的形式创建新的进程。这个新进程和原始进程一模一样。
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

  - ```Process.wait``` 一家子
    - 如果父进程拥有不止一个子进程， 并且使用了 ```Process.wait```，那么你就需要知道究竟是哪个子进程退出了。
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
  - 使用 ```Process.wait2``` 进行通信
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
    - 如果父进程还没来得及从 ```Process.wait``` 返回，另一个子进程也退出了，这会怎样?
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

  - 实践领域
    - ```Unicorn``` 运行模式：```master/workder``` or ```preforking```
    - 你有一个衍生出多个并发子进程的进程，这个进程看管着这些子进程，确保它们能够保持响应，并对子进 程的退出做出回应，等等

### 15 僵尸进程
  - 等待终有果
    - 内核会一直保留已退出的子进程的状态信息，直到父进程使用 Process.wait 请求这些信息。
    - 如果父进程一直不发出请求，那么状态信息就会被内核一直保留着。
    - 因此创建即发即弃的子进程，却不去读取状态信息，便是在浪费内核资源。
    ```ruby
      message = 'Good Morning'
      recipient = 'tree@mybackyard.com'

      pid = fork do

        # 在这个人为设计的例子中，父进程衍生出一个子进程来负责将数据发送给统计收集器。
        # 同时，父进程继续进行自己实际的数据发送工作。
        # 父进程不希望自身被这项任务所拖缓，即便任务出于某种原因失败， 
        # 父进程也不会受到影响。
        
        StatsCollector.record message, recipient
      end

      # 这一行代码确保进行统计收集的进程不会变成僵尸。 
      Process.detach(pid)
    ```
    - Process.detach做了些什么? 它不过是生成了一个新线程。
    - 这个线程的唯一工作就是等待由 pid 所指定的那个子进程退出。
    - 这确保了内核不会一直保留那些我们不需要的状态信息。

  - 僵尸长什么样子
    ```ruby
      # 创建一个子进程，1 秒钟之后退出。
      pid = fork { sleep 1 }

      # 打印出该子进程的pid。
      puts pid
      
      # 让父进程长眠，以便于我们检查子进程的进程状态信息。
      sleep
    ```
    ```bash
      # 状态为“z”或“Z+”就表示这是一个僵尸进程。
      $ ps -ho pid, state -p [pid of zombie process]
    ```

### 16. 进程皆可获得信号
  - 为何要有信号？
    - ```Process.wait``` 为父进程提供了一种很好的方式来监管子进程。
    - 但它是一个阻塞调用:直到子进程结束，调用才会返回。
    - 一个繁忙的父进程不会闲暇到一直等着自己的子进程结束。
    - 对此倒是有一个解决方案，这就是我们要介绍的 ```Unix``` 信号。
  - 捕获 SISCHLD
    ```ruby
      child_processes = 3
      dead_processes = 0
      # 衍生出3个子进程
      child_processes.times do
        fork do
          # 各自休眠3秒钟
          sleep 3
        end
      end

      # 父进程忙于执行一些密集的数学运算，但是仍想知道子进程何时退出。

      # 通过捕获 :CHLD 信号，内核会提醒父进程它的子进程何时退出。
      trap(:CHLD) do
        # 由于 Process.wait 将它获得的数据都加入了队列，因此可以在此进行查询，
        # 因为我们知道其中一个子进程已经退出了。

        puts Process.wait
        dead_processes += 1
        # 一旦所有的子进程统计完毕，就直接退出。
        exit if dead_processes == child_processes
      end

      # 父进程需要执行的密集数学运算。
      loop do
        (Math.sqrt(rand(44)) ** 8).floor
        sleep 1
      end
    ```

  - SIGCHLD 与并发
    - 信号投递是不可靠的。
    - 如果你的代码正在处理 CHLD 信号，这时另一个子进程结束了，那么未必能收到第二个 CHLD 信号。
    - 这会导致上面的代码片段可能产生不一致的后果。
      ```ruby
        # 要正确地处理 CHLD，你必须在一个循环中调用 Process.wait，查找所有已经结束的子进程，
        # 这是因为在进入信号处理程序之后，你可能会收到多个 CHILD 信号。
        # 但是，Process.wait 不是一个阻塞调用吗?
        # 如果只有一个已结束的子进程，而我却调用了两次 Process.wait，又该如何避免阻塞整个进程呢?

        # 现在我们得派上 Process.wait 的第二个参数了。
        # 在上一章我们将一个 pid 传给 Process.wait 作为首个参数，不过它也可以将标志作为第二个参数。
        # 这样的标志可以告诉内核，如果没有子进程退出，那么就不需要进行阻塞。
        # 这恰恰就是我们需要的东西!

        # 常量 Process::WNOHANG 描述了这个标志的值，它可以像这样来使用:
        Process.wait(-1, Process::WNOHANG)
      ```
    - 下面对开头的代码片段进行重构，重构后的代码不会“错过”任何子进程的死亡：
      ```ruby
        # 衍生出 3 个子进程。
        child_processes = 3
        dead_processes = 0
        
        child_processes.times do
          fork do
            # 各自休眠3秒钟。
            sleep 3
          end
        end

        # 设置 $stdout 的 sync, 使得在 CHLD 信号处理程序中不会对 #puts 调用进行缓冲。
        # 如果信号处理程序在调用 #puts 之后被中断，则会引发一个 ThreadError。
        # 如果你的信号处理程序需要执行 IO 操作，那么这不失为一个好方法。
        $stdout.sync = true

        # 父进程忙于执行一些密集的数学运算，但是仍想知道子进程何时退出。

        # 通过捕获:CHLD 信号，内核会提醒父进程它的子进程何时退出。
        trap(:CHLD) do
          # 由于 Process.wait 将它获得的数据都加入了队列，因此可以在此进行查询
          # 因为我们知道其中一个子进程已经退出了。

          # 我们需要执行一个非阻塞的 Process.wait 以确保统计每一个结束的子进程。
          begin
            while pid = Process.wait(-1, Process::WHOHANG)
              puts pid
              dead_processes += 1
              # 一旦所有的子进程统计完毕就退出。
              exit if dead_processes == child_processes
            end
          rescue Errno::ECHILD
          end
        end


        # 父进程需要执行的密集数学运算。
        loop do
          (Math.sqrt(rand(44)) ** 8).floor
          sleep 1
        end
      ```
      - 如果没有子进程存在，Process.wait 乃至其变量，将会抛出 Errno::ECHILD 异常。
      - 因为信号可以在任何时间到达， 很可能最后一个 CHLD 信号到达的时候，
      - 之前的 CHLD 处理程序已经先后两次调用 Process.wait，并得到了最后的可用状态信息。
      - 任何一行代码都能够被信号中断。
      - 你必须在自己的 CHLD 信号处理程序中处理 Errno::ECHILD 异常。
      - 同样地，如果你不知道需要等待多少个子进程，那么你就应该捕获这个异常并正确处理它。
  
  - 信号入门
    - 信号是一种```异步```通信。(当进程从内核那里接收到一个信号时，它可以执行下列某一操作:)
      - 1. 忽略该信号。
      - 2. 执行特定的操作。
      - 3. 执行默认的操作。

  - 信号来自何方？
    - 信号是由一个进程发送到另一个进程，只不过是借用内核作为中介。
      - 不加任何参数执行ruby程序(进入一个 ruby 会话)。输入一些代码，然后点击 Ctrl-D。
      - 这样会执行你刚才输入的代码并退出。
      ```ruby
        # 第一个 ruby 会话
        puts Process.pid
        sleep # 以便于有时间发送信号

        # 第二个 ruby 会话中发出下列命令来使用信号终结第一个会话：
        Process.kill(:INT, <pid of first session>)

        # 因此第二个进程会向第一个进程发送一个 INT 信号，使其退出。
        # INT 是 INTERRUPT(中断)的缩写。
      ```
  
  - 信号一览
    - 表中的“动作”一列 描述了每个信号的默认操作。
      - Term: 进程会立即结束
      - Core: 进程会立即结束并进行核心转储(栈跟踪)
      - Ign:  进程会忽略该信号
      - Stop: 进程会停止运行(暂停)
      - Cont: 进程会恢复运行(继续)
    
  - 重定义信号
    ```ruby
      trap(:INT) { print "其他进程不能通过 INT 信号使该进程退出。但 2.5.0 版 Ruby 是可以退出的" }
    ```

  - 忽略信号
    ```ruby
      # 第一个 ruby 会话
      puts Process.pid
      trap(:INT, "IGNORE") # ignore
      sleep # 以便于有时间发送信号

      # 第二个 ruby 会话中使用下面的命令，注意第一个进程并没有受到影响。
      Process.kill(:INT, <pid of first session>)
    ```

  - 信号处理程序是全局性的
    - 捕获一个信号有点像使用一个全局变量，你有可能把其他代码所依赖的东西给修改了。
  
  - 恰当地重定义信号处理程序
    - 如果你只是想加入一些操作，以在退出之前能够清理资源，那就可以使用一个 at_exit 钩子。

  - 何时收不到信号？
    - 进程可以在任何时候接收到信号。这就是信号的美之所在! 它们是异步的。

  - 实践领域
    - Unicorn:
      - 通过终止其所有进程并立即关闭来响应 INT 信号。
      - 通过重新执行来响应 USR2 信号，从而实现零关闭时间重启(zero-downtime restart)。
      - 通过增加运行的工作进程数量来响应 TTIN 信号。

### 17. 进程皆可互通
  - 进程间通信（IPC）
    - 实现方式：管道； 套接字对（socket pairs）。

  - 管道（单向数据流）：
    - 你可以打开一个管道，一个进程拥有管道的一端，另一个进程拥有另一端。
    - 然后数据就沿着管道单向传递。
    - 因此如果某个进程将自己作为 reader(读者)，而非 writer(写者)，那么它就无法向管道中写入数据。
    - 反之亦然。
    ```ruby
      # 创建一个管道
      reader, writer = IO.pipe #=> [#<IO:fd 5>, #<IO:fd 6>]

      # Ruby 神奇的 IO 类是 File、TCPSocket、UDPSocket 等的超类。
      # 所有这些资源都有一些通用的接口（#write, #read, #close）。
      # 从 IO.pipe 返回的 IO 对象可以看作是类似于匿名文件的东西。
      # 基本上你可以将其视为 File 来对待。

      reader, writer = IO.pipe
      writer.write("Into the pipe I go...")
      writer.close
      puts reader.read #=> Into the pipe I go..

      # 当 reader 调用 IO#read 时，它会不停地试图从管道中读取数据，
      # 直到读到一个 EOF(文件结束标志2)。这个标志告诉 reader 已经没有数据可读了。

      # 只要 writer 仍旧保持打开，那么 reader 就可能读取到更多的数据，因此它就会一直等待。
      # 在读取之前关闭 writer，将一个 EOF 放入管道中，这样一来，reader 获得原始数据之后就会停止读取。
      # 要是你跳过关闭 writer 这一步，那么 reader 就会被阻塞并不停地试图继续读取数据。
    ```
  
  - 管道是单向的
    ```ruby
      # reader 只能读取；writer 只能写入
      reader, writer = IO.pipe
    ```
  
  - 共享管道
    - 下面是一个使用管道在父进程与子进程之间进行通信的简单例子。
    - 子进程通过向管道写入信息来告诉父进程它已经完成了自己的一轮工作:
      ```ruby
        reader, writer = IO.pipe

        fork do
          reader.close

          10.times do
            # 写入数据
            writer.puts "Another one bites the dust"
          end
        end

        writer.close
        while message = reader.gets
          $stdout.puts message
        end

        # 输出十次 Another one bites the dust。

        # 如今涉及到两个进程，在考虑 EOF 时就需要再多考虑一层。
        # 因为文件描述符会被复制，所以现在就出现了 4 个文件描述符。
        # 其中只有两个会被用于通信，其他两个必须关闭。
      ```
    - 管道中流淌的是数据流。
  
  - 流与消息
    - 流：(需要分隔符)
      - 在管道中读写数据时，并没有开始和结束的概念。
      - 当使用诸如管道或 TCP 套接字这样的 IO 流时，
      - 你将数据写入流中，之后跟着一些特定协议的分隔符(delimiter)。
      - 比如 HTTP 使用一连串行终止符来分隔头部和主体。

    - 消息：(不需要分隔符)
      - 没法在管道中使用消息，但可以在 Unix 套接字中使用。
      - Unix 套接字是一种只能用于在同一台物理主机中进行通信的套接字。
      ```ruby
        # 这段代码创建了一对已经相互连接好的 UNIX 套接字。
        # 这些套接字并不使用流，而是使用数据报(datagram)通信。
        # 在这种方式中，你向其中一个套接字写入整个消息，
        # 然后从另一个套接字中读取整个消息，不需要分隔符。

        # 下面的例子是：子进程等待父进程告诉它要做什么，并在工作完成后向父进程报告：
        require 'socket'

        child_socket, parent_socket = Socket.pair(:UNIX, :DGRAM, 0)
        maxlen = 1000

        fork do
          parent_socket.close

          4.times do
            instruction = child_socket.recv(maxlen)
            child_socket.send("#{instruction} accomplished!", 0)
          end
        end
        child_socket.close

        2.times do
          parent_socket.send("Heavy lifting", 0)
        end

        2.times do
          parent_socket.send("Feather lifting", 0)
        end

        4.times do
          $stdout.puts parent_socket.recv(maxlen)
        end
      ```
    
    - 管道提供的是单向通信，套接字对提供的是双向通信。父套接字可以读写子套接字，反之亦然。

  - 远程 IPC
    - IPC 意味着运行在同一台机器上的进程间的通信。
    - 如果你想从单机扩展到多台机器，且同时实现类似于 IPC 的功能（可选方案）：
      - TCP 套接字。
      - RPC2(远程过程调用)。
      - 分布式系统。

### 18. 守护进程
  - 介绍：
    - 守护进程是在后台运行的进程，不受终端用户控制。
    - Web 服务器或数据库服务器都属于常见的守护进程，它们一直在后台运行响应请求。

  - 首个进程：
    - 当内核被引导时会产生一个叫做 init 的进程。
    - 这个进程的 ppid 是 0，作为所有进程的祖父。它是首个进程，没有祖先。它的 pid 是 1。

  - 创建第一个守护进程
    - rack
  
  - 深入 Rack
    ```ruby
      def daemonize_app
        if RUBY_VERSION < "1.9"
          exit if fork    # 父进程衍生出第一个子进程，然后退出
          Process.setsid  # 第一个子进程成为一个新进程组和新会话组的组长兼领导
          exit if fork    # 第一个子进程进行衍生(出第一个孙进程)，然后退出
          Dir.chdir "/"
          STDIN.reopen "/dev/null"
          STDOUT.reopen "/dev/null", "a"
          STDERR.reopen "/dev/null", "a"
        else
          Process.daemon # 将当前进程变成守护进程
        end
      end
    ```
  
  - 逐步将进程变成守护进程
    - 
      ```ruby
        exit if fork
        # fork 会返回两次
        # 父进程中返回子进程 pid, 父进程退出
        # 子进程返回 nil, 故成为孤儿进程继续运行
      ```
    -
      ```ruby
        Process.setsid
        # 1. 该进程变成一个新会话的会话领导
        # 2. 该进程变成一个新进程组的组长
        # 3. 该进程没有控制终端
      ```
  
  - 进程组和会话组
    - 进程组和会话组都和作业控制有关。

    - 进程组：
      - 每一个进程都属于某个组，每一个组都有唯一的整数 id。
      - 进程组是一个相关进程的集合，通常是父进程与其子进程。
      - 进程组 id 和进程组组长的 pid 相同。
      ```ruby
        puts Process.pid        #=> 51139
        puts Process.getpgrp    #=> 51139

        fork {
          puts Process.pid      #=> 57767
          puts Process.getpgrp  #=> 51139
        }

        # 子进程的组 id 是从父进程中继承而来。
        # 因此这两个进程都是同一个进程组的成员。
      ```
      - 终端接收信号，并将其转发给前台进程组中的所有进程。
      - 因此同一个进程组的组员，会被同一个信号终止。
    
    - 会话组
      - 进程组的集合。
      ```bash
        git log | grep shipped | less

        # 每个命令都有自己的进程组，这是因为每个命令都可能创建子进程，
        # 但这些子进程并不属于其他命令。
        # 尽管这些命令不属于同一个进程组，Ctrl-C 仍可以将其全部终止。

        # 这些命令都是同一个会话组的成员，在 shell 中的每一次调用都会获得自己的会话组。
        # 一次调用可以是单个命令，也可以是由管道连接的一串命令。
      ```
      - 终端发送给会话领导的信号被转发到该会话中的所有进程组内，
      - 然后再被转发到这些进程组中的所有进程。
      ```ruby
        # 现在返回到 Rack 的例子中，第一行代码衍生出一个子进程，然后父 进程退出。
        # 启动该进程的终端觉察到进程退出后，将控制返回给用户， 但是之前衍生出的子进程# 仍然拥有从父进程中继承而来的组 id 和会话 id。
        # 此时这个衍生进程既非会话领导，也非组长。

        # 因此终端与衍生进程之间仍有牵连，如果它发出信号到衍生进程的会话组，这个信号仍会被接收到，但是我们想要的是完全脱离终端。

        # Process.setsid 会使衍生进程成为一个新进程组和新会话组的组 长兼领导。
        # 注意，如果在某个已经是进程组组长的进程中调用 Process.setsid，则会失败，它只能从子进程中调用。

        # 新的会话组并没有控制终端，不过从技术上来说，可以给它分配一个。
        exit if fork
        # 已成为进程组和会话组组长的衍生进程再次进行衍生，然后退出。

        # 这个新衍生出的进程不再是进程组的组长，也不是会话领导。
        # 由于之前的会话领导没有控制终端，并且此进程也不是会话领导，因此这个 进程绝不会有控制终端。终端只能够分配给会话领导。

        # 如此以来就确保了我们的进程现在完全脱离了控制终端并且可以独自运行。
        Dir.chdir "/"
        # 此行代码将当前工作目录更改为系统的根目录。
        # 并非一定要这么做， 这额外的一步确保了守护进程的当前工作目录在执行过程中不会消失。
        # 这就避免了守护进程的启动目录出于这样或那样的问题被删除或卸载。

        STDIN.reopen "/dev/null"
        STDOUT.reopen "/dev/null", "a"
        STDERR.reopen "/dev/null", "a"
        # 这将所有的标准流设置到/dev/null，也就是将其忽略。
        # 因为守护 程不再依附于某个终端会话，那么这些标准流也就没什么用了。
        # 不能简单地将其关闭，因为一些程序还指望着它们随时可用。
        # 重定向到 /dev/null 确保了它们对于一些程序依然能用，但实际上毫无效果。
      ```
  - 实践领域
    - 如果你想创建一个守护进程，一个基本的问题：这个进程需要一直保持响应吗?
    - 否：可以考虑 定时任务 或 后台作业系统。
    - 是：可能已经有了好的候选方案[daemons](http://rubygems.org/gems/daemons)

### 19. 生成终端进程
  - ```fork``` + ```exec```
    - ```exec(2)```: 
      - 作用：使用另一个进程来替换当前进程。
      - 缺点：替换后当前进程再也无法恢复了，而 ```fork(2)``` 无此问题。
    - 可以用 ```fork(2)``` 创建一个新进程，然后用 ```exec(2)``` 把这个进程变成其他想要的进程。
    - 你的当前进程仍像从前一样运行，也仍可以根据需要生成其他进程。
    ```ruby
      hosts = File.open('/etc/hosts')

      exec 'python', '-c', "import os; print os.fdopen(#{hosts.fileno}).read()"

      # 我们启动了一个 Ruby 程序并打开文件/etc/hosts。
      # 然后 exec(2)产生一个 python 进程，告诉它文件 /etc/hosts 打开时 Ruby 所获得的文件描述符编号。
      # python 识别出了此文件描述符(因为它通过 exec(2)共享)，并能从中进行读取，而无需再次打开文件。
    ```
  
  - ```exec``` 的参数
    - 字符串(形式)：把字符串传递给 ```exec```，它实际上会启动一个 ```shell``` 进程，然后再将这 个字符串交由 ```shell``` 解释。
    - 数组(形式)：传递一个数组的话，它会跳过 shell，直接将此数组作为新进程的 ```ARGV```。
    - 安全：传递一个字符串，如果涉及用户输入，那么用户有可能直接将恶意命令插入到 ```shell``` 中，来获得当前进程所拥有的任何权限。
    - 如果你希望实现类似于 ```exec('ls * | awk '{print($1)}')``` 的效果，那就必须通过字符串 来传递了。
    - ```Kernel#system```
      ```ruby
        system('ls')
        system('ls', '--help')
        system('git log | tail -10')

        # Kernel#system 的返回值用最基本的方式反映了终端命令的退出码。
        # 如果终端命令的退出码是 0，它就返回 true，否则返回 false。
      ```
      - 借助 fork(2) 的魔力，终端命令与当前进程共享标准流，因此来自终端命令的任何输出同样也会出现在当前进程中。
    - Kernel#`
      ```ruby
        `ls`
        `ls --help`
        %x[git log | tail -10]

        # Kernel#` 略有不同，它的返回值是由终端程序的 STDOUT 汇集而成的一个字符串。
      ```
    - ```Process.spawn```
      ```ruby
        # 此调用会启动 rails server 进程并将环境变量 RAILS_ENV 设置为 test
        Process.spawn({'RAILS_ENV' => 'test'}, 'rails server')

        # 该调用在执行 ls --help 阶段将 STDERR 与 STDOUT 进行合并
        Process.spaw('ls', '--help', STDERR => STDOUT)

        # Process.spawn 是非阻塞的；而 Kernel#system 却会阻塞。
        # 以阻塞方式执行
        system 'sleep 5'

        # 以非阻塞方式执行
        Process.spawn 'sleep 5'

        # 使用 Process.spawn 以阻塞方式执行
        # 返回子进程的pid
        pid = Process.spawn 'sleep 5'
        Process.waitpid(pid)
      ```
    - ```IO.popen```
      ```ruby
        # 这个例子会返回一个文件描述符(IO 对象)
        # 对其进行读取会返回该 shell 命令打印到 STDOUT 中的内容
        IO.popen('ls')

        # 一个 IO 对象被传递到代码块中。在本例中我们打开 stream 进行写入，
        # 因此将stream设置为生成进程的STDIN。
        #
        # 如果打开stream进行读取(默认操作)，那么将stream设置为生成进程的STDOUT。
        IO.popen('less', 'w') { |stream|
          stream.puts "some\ndata"
        }

        # 使用 IO.popen 时必须选择访问哪个流。你无法一次访问所有的流。
      ```
    - ```Open3```
      - Open3 允许同时访问一个生成进程的 STDIN、STDOUT 和 STDERR。
      ```ruby
        # Open3 是标准库的一员 
        require 'open3'

        Open3.popen3('grep', 'data') { |stdin, stdout, stderr|
          stdin.puts "some\ndata"
          stdin.close
          puts stdout.read
        }

        # 在可行的情况下，Open3 会使用 Process.spawn
        # 可以像这样将选项传递给Process.spawn:
        Open3.popen3('ls', '-uhh', :err => :out) { |stdin, stdout, stderr|
          puts stdout.read
        }
      ```

  - 实践领域：
    - 子进程 ```fork``` 父进程
      - 1. 获得了一份父进程在内存中所有内容的副本。
      - 2. 获得了父进程已打开的所有文件描述符的副本。
      - 所以：fork(2)是有成本的，有时候它会成为性能瓶颈。
    - ```posix_spawn```
      - posix_spawn(2)只保留了第 2 条，没有保留第 1 条，这是两者最大的不同。
      - 新生成的进程可以访问由父进程打开的所有文件描述符，却无法与父进程共享内存。
      - 这就是为什么 posix_spawn(2) 比 fork(2)更快、更有效率的原因。
      - 但是要记住，这也会使得它缺乏灵活性。

### 20. 尾声
  - 抽象
    - 所有的一切在内核眼中没什么两样。所有的代码都会被编译成内核能够理解的简单形式。
    - 在那个层面工作的时候，所有的进程都被同等对待，所有的一切都会获得数字标识符，都能够平等地访问内核资源。
    - Unix 编程与具体的编程语言无关。
    - 你在 Ruby 中学到的 Unix 编程技巧同样可以用于 Python、node.js 或 C。

  - 通信
    - 系统中任意的两个进程都可以使用信号来通信。
    - 通过为进程命名，你可以同任何在命令行中查看你的程序的用户进行通信。
    - 你可以使用退 出码给所有期待运行结果的进程发送成功/失败的消息。

### 附录A
#### Resque 如何管理进程
  - 1. 架构
    - Resque是一个基于Redis的库，用于创建后台作业，将这些作业放入多个队列并随后处理。
  - 2. 利用进程衍生进行内存管理
    - Resque worker 使用 fork(2)进行内存管理。
      ```ruby
        if @child = fork
          srand # Reseeding

          # procline 是 Resque 更新当前进程名的内部方式。
          procline "Forked #{@child} at #{Time.now.to_i}"

          # 告诉父进程一直阻塞直到子进程结束。
          Process.wait(@child)
        else
          procline "Processing #{job.queue} since #{Time.now.to_i}"

          # 在这个子进程中，作业由 Resque 来执行。
          perform(job, &block)

          # 然后子进程退出。
          exit! unless @cant_fork
        end
      ```
  - 3. 何必自找麻烦？
    - Resque 使用 fork(2) 来确保其工作进程使用的内存不会膨胀。
      - 原始进程预先载入了应用程序环境，fork 之后，会获得一个载入了应用程序环境的新进程。
      - 后台作业可能需要将图像文件载入主存进行处理，从数据库中读入大量的     ActiveRecord 对象，或者执行其他消耗大量内存的操作。
      - 一旦子进程处理完作业并退出，将会释放其所占用的全部内存并交由操作系统进行清理。
      - 然后原始进程就可以恢复运行，同样只载入应用程序环境。
    - 就内存占用而言，每当 Resque 处理完一个作业，都会返回到洁净状态。
    - 处理作业时，内存使用量或会激增，但终会回落到一个适宜的基线水平。

### 附录B
#### Unicorn 如何收割工作进程
  - 1. 收割什么？
    - 从宏观看：Unicorn 是一个 pre-forking Web Server.
      - Unicorn 通过初始化自己的网络套接字开始运行，然后载入你的应用程序。 
      - 随后，它采用 master-worker 模式， 使用 fork(2) 创建工作进程。
    - 退出前的清理工作：
      ```ruby
        # 收割所有还未被收割的工作进程
        def reap_all_workers
          begin
            wpid, status = Process.waitpid2(-1, Process::WNOHANG)
            wpid or return
            if reexec_pid == wpid
              logger.error "reaped #{status.inspect} exec()-ed"
              self.reexec_pid = 0
              self.pid = pid.chomp('.oldbin') if pid
              proc_name 'master'
            else
              worker = WORKERS.delete(wpid) and worker.close rescue nil
              m = "reaped #{status.inspect} worker=#{worker.nr rescue 'unknown'}"
              status.success? ? logger.info(m) : logger.error(m)
            end
          rescue Errno::ECHILD
            # 如果当前进程没有子进程，那么 Process.waitpid2 或是它的任何表亲会产生 Errno::ECHILD 异常。
            # 那就意味着该方法的任务已经完成!
            # 所有的子进程都已经收割完毕，方法顺利返回。
            break
          end while true # 无限循环
        end
      ```

### 附录C
#### preforking 服务器
  - prefoking 模型：
    - 1. 高效的内存利用
    - 2. 高效的负载均衡
    - 3. 高效的系统管理
  - 内存利用：
    - 假设一个进程启动需要 3 秒，载入 Rails 需要从内核处申请资源(内存)50MB。
    - 10 个 ```Mongrel``` 进程启动时，合计会耗时 30 秒，占用内存：500MB。

    - 10 个 ```Unicorn```工作进程会产生11个进程，其中一个是主进程，负责看护其他 10 个工作进程。
    - 而且只有一个进程，也就是主进程，会载入 ```Rails```，不会出现竞争内核资源的情况。
    - 主进程花费 3 秒钟的时间来载入，然后几乎瞬间就衍生出 10 个进程。 
    - 主进程消耗 50MB 的内存来载入 ```Rails```，而由于写时复制技术(```Cow```)，子进程应该不会消耗任何内存。【但 MRI 不支持 ```Cow``` 】
  - 负载均衡：
    - 套接字工作步骤：
      - 1. 打开一个套接字并绑定到特定的端口
      - 2. 在这个套接字上使用```accept(2)```接受一个连接
      - 3. 可以在这个连接上读写数据，最后关闭连接。套接字一直处于打开状态，但连接会被关闭。
    - 通常这都是在同一个进程内发生的。
      - 一个套接字被打开，然后该进程在此套接字上等待、处理并关闭连接，继而开始下一个循环。
    ```ruby
      # 如何让内核在套接字上对繁重的负载进行均衡

      # 像 Unicorn，主进程第一件事就是打开套接字，用于来自 Web 客户端的外部链接。
      # 但主进程不接受链接，由于 fork(2) 的工作方式，当主进程衍生出工作进程时，每个工作进程都获得一个打开的套接字副本。

      # 此即是它的神奇之处。

      # 每个工作进程拥有一个打开的套接字副本，并在此套接字上使用 accept(2) 来接受连接。
      # 这时候由内核接管并在套接字的 10 个副本上均衡负载，它确保只有单个进程接受每个连接。
    ```
  - 系统管理：
    - ```preforking``` 服务器的管理人员一般只需要向主进程发出命令(通常是信号)。
    - 它就会负责处理消息的跟踪并转发给工作进程。
    - 基础样例：
      ```ruby
        require 'socket'

        # 打开一个套接字
        socket = TCPServer.open('0.0.0.0', 8080)

        # 预载入应用代码
        # require 'config/environment'

        # 将相关的信号转发给子进程
        [:INT, :QUIT].each do |signal|
          Signal.trap(signal) {
            wpids.each { |wpid| Process.kill(signal, wpid) }
          }
        end

        # 跟踪子进程的 pid
        wpids = []

        5.times {
          wpids << fork do
            loop {
              connection = socket.accept
              connection.puts 'Hello Readers!'
              connection.close
            }
          end
        }

        Process.waitall # 执行一个循环，等待所有的子进程退出，然后返回一个进程状态数组。
      ```

### 附录D
#### Spyglass
  - ```Spyglass``` 的体系
    - Spyglass 是一个 Web 服务器，它向外部世界打开一个套接字并处理 Web 请求。
  - 启动 ```Sypglass```
    ```bash
      $ spyglass
      $ spyglass -p other_port
      $ spyglass -h # 帮助信息
    ```
  - 请求抵达之前
    - 启动后，控制权交予 `Spyglass::Lookout`。
    - 这个类并不会预载入 `Rack` 应用，对于 `HTTP` 也一无所知，它只是等待某个连接 是一个打开的套接字而已。
  - 建立连接
    - 当 `Spyglass::Lookout` 注意到某个连接已经建立，它便衍生出一个 `Spyglass::Master` 来处理这个连接。
    - 在衍生出主进程之后，`Spyglass::Lookout` 使用 `Process.wait` 保持空闲，直到主进程退出。

    - `Spyglass::Master` 负责预载入 `Rack` 应用并衍生/看护工作进程。
    - 主进程本身并不了解 HTTP 解析或请求处理。

    - 实际的工作是在 `Spyglass::Worker` 中完成的。
    - 它使用 `preforking` 那章介绍的方法来接受连接，依靠内核进行负载均衡。
    - 一旦获得一个连接，它就解析 `HTTP` 请求，调用 `Rack` 应用，并为客服端生成响应。
  - 万事皆毕
    - 工作： 
      - 只要有稳定的流量进入，`Spyglass` 就会一直作为 `preforking` 服务器进行运作。
    - 退出：
      - 如果在内部计时器超时之前没有接收到任何接入请求，那么主进程和它所有的工作进程都会退出。
      - 控制权返回到 `Spyglass:: Lookout`，这个工作流程再次从头开始。
