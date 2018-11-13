Sidekiq 如何处理异步任务
=====================

### 1. 概述
  ```ruby
    class HardWorker
      include Sidekiq::Worker
      def perform(name, count)
        # do something
      end
    end

    HardWorker.perform_async('bob', 5)
    # 此方法调用时，Sidekiq 的 Worker 会将一个异步任务以 JSON 的形式将相关的信息加入 Redis 中并等待消费者对任务的拉取和处理。
  ```
  - `Sidekiq` 的消费者组成：
    - 1. Manager
    - 2. Processor   负责执行执行任务
    - 3. Poller

### 2. 异步任务的入队
  - `perform_async`
    - 当调用 `Worker.perform_async` 时，`Sidekiq` 并不会真正去创建一个 `HardWorker` 的对象。
    - 它实际上会调用 `Worker.client_push` 方法并将当前的 `class` 和 `args` 参数传进去。
    ```ruby
      def perform_async(*args)
        client_push('class'.freeze => self, 'args'.freeze => args)
      end
    ```
  - `perform_in` and `perform_at`
    - `Sidekiq` 还提供了一对用于`在一段时间之后或者某个时间点`执行相应任务的方法。
    ```ruby
      def perform_in(interval, *args)
        int = interval.to_f
        now = Time.now.to_f
        ts = (int < 1_000_000_000 ? now + int : int)
        item = { 'class'.freeze => self, 'args'.freeze => args, 'at'.freeze => ts }
        item.delete('at'.freeze) if ts <= now
        client_push(item)
      end
      alias_method :perform_at, :perform_in
    ```
  - `client_push`
    - 它获取了上下文中的 `Redis` 池并将传入的 `item` 对象传入 `Redis` 中。
    ```ruby
      def client_push(item)
        pool = Thread.current[:sidekiq_via_pool] || get_sidekiq_options['pool'.freeze] || Sidekiq.redis_pool
        item.kyes.each do |key|
          item[key.to_s] = item.delete(key)
        end
        Sidekiq::Client.new(pool).push(item)
      end
    ```
  - ```Client#push```
    ```ruby
      # 
      def push(item)
        normed = normalize_item(item)
        payload = process_single(item['class'.freeze], normed)

        if payload
          raw_push([payload])
          payload['jid'.freeze]
        end
      end
    ```
  - 整理一下思绪，从 `Worker.perform_async` 到 `Client#push` 整个过程都在对即将加入到 Redis 中队列的哈希进行操作，从添加 `at` 字段到字符串化，再到 `Client#normalize_item` 中添加 `jid` 和 `created_at` 字段。
  - 当异步任务需要在未来的某一时间点进行安排时，它会加入 `Redis` 的一个有序集合。
    ```ruby
      def atomc_push(conn, payloads)
        if payloads.first['at'.freeze]
          conn.zadd('schedule'.freeze, payloads.map do |hash|
                      at = hash.delete('at'.freeze).to_s
                      [at, Sidekiq.dump_json(hash)]
                    end)
        else
          # ...
        end
      end

      # 这个有序集合中，Sidekiq 将 schedule 作为权重，而其他的全部字段都以 JSON 的格式作为负载传入。
    ```
  - 但当 `Sidekiq` 遇到需要立即执行的异步任务时，实现有所不同。
    ```ruby
      def atomc_push(conn, payloads)
        if payloads.first['at'.freeze]
          # ...
        else
          q = payloads.first['queue'.freeze]
          now = Time.now.to_f
          to_push = payloads.map do |entry|
            entry['enqueued_at'.freeze] = now
            Sidekiq.dump_json(entry)
          end
          conn.sadd('queues'.freeze, q)
          conn.lpush("queue.#{q}", to_push)
        end
      end

      # 除了设置当前任务的入队时间 enqueued_at 之外， Sidekiq 将队列加入到一个大队列 queues 的集合中，并且将负载直接推到 "queue:#{q}" 数组中等待消费者的拉取。
    ```

### 3. Redis 中的存储
  - 无论是立即执行还是需要安排的异步任务都会进入 Redis 的队列。
  - `Worker.perform_in/at`
    - 会将任务以 `[at, args]` 的形式加入到 `schedules` 有序集中。
  - `Worker.perform_async` 将负载加入到指定的队列，并向整个 `Sidekiq` 的队列集合 `queues` 中添加该队列。
  - 所有的 `payload` 中都包含了一个异步任务需要执行的全部信息，包括该任务的执行的队列 `queue`，异步队列的类 `class`, 参数 `args` 以及 `sidekiq_options` 中的全部参数。
   - 异步任务包含的参数：[jid, queue, class, at, args, sidekiq_options, created_at, enqueued_at]

### 4. Sidekiq Worker 的启动过程
  - `bin/sidekiq` 启动文件
    ```ruby
      begin
        cli = Sidekiq::CLI.instance # 实例化一个 CLI 对象
        cli.parse                   # 对参数进行解析
        cli.run                     # 
      rescue => e
        # ...
      end
    ```
  - ```cli#run```
    ```ruby
      def run
        print_banner

        self.read, self.write = IO.pipe
        # ...

        launcher = Sidekiq::Launcher.new(options)
        begin
          launcher.run
          while readable_io = IO.select([self_read])
            signal = readable_io.first[0].gets.strip
            handle_signal(signal)
          end
        rescue Interrupt
          launcher.stop
        end
      end
    ```

### 5. 从 Launcher 到 Manager
  - 调用 `Launcher#run` 运行用于处理异步任务的 `Processor` 等对象。
    ```ruby
      {
        "Launcher" => [
          "Poller",
          "Manager" => [
            "processor1",
            "processor2",
            "processor3",
            "processor4"
          ]
        ]
      }
    ```
    - 每一个 `Launcher` 都会启动一个 `Manager` 对象和一个 `Poller`, 其中 `Manager` 同时管理了多个 `Processor` 对象，这些不同的类之间有着如上图所示的关系。
    ```ruby
      # Launcher#run
      def run
        @thread = safe_thread("heartbeat", &method(:start_heartbeat))
        @poller.start
        @manager.start

        # Manager 会在初始化时根据传入的 concurrency 的值创建对应数量的 Processor。
        # 当执行 Manager#start 时，就会启动对应数量的线程和处理器开始对任务进行处理：
      end
    ```
    ```ruby
      class Manager
        def start
          @workers.each do |x|
            x.start
          end
        end
      end

      class Processor
        def start
          @thread ||= safe_thread("processor", &method(:run))
        end
      end
    ```

### 6. 并行模型
  - 当处理器开始执行 `Processor#run` 方法时，就开始对所有的任务进行处理了。
  - Sidekiq 使用多线程的模型对任务进行处理，每一个 `Processor` 都是使用了 `safe_thread` 方法在一个新的线程里面运行。
    ```ruby
      def safe_thread(name, &block)
        Thread.new do
          Thread.current['sidekiq_label'.freeze] = name
          watchdog(name, &block)
        end
      end
    ```
  - Sidekiq 可以以多进程、多线程的方式运行，同时处理大量的异步任务。
    ```ruby
      'Sidekiq-Process-1' => [
        'processor-Thread-1'
        'processor-Thread-2'
        'processor-Thread-3'
        'processor-Thread-4'
        'heartbeat-Thread-1'
        'scheduler-Thread-1'
        ]

        'Sidekiq-Process-2' => [
        'processor-Thread-1'
        'processor-Thread-2'
        'processor-Thread-3'
        'processor-Thread-4'
        'heartbeat-Thread-1'
        'scheduler-Thread-1'
        ]

        # ...
    ```

## 异步任务的处理
### 7. 【主题】的订阅
  - 一个 Sidekiq Worker 进程，它在启动时就会决定选择订阅哪些『主题』去执行。
    ```bash
      > sidekiq -q critical, 2 -q default

      # CLI#parse 会对传入的 -q 参数进行解析。
    ```
    ```ruby
      def parse(args=ARGV)
        # ...
        validate!
        # ...
      end

      # 没有传入队列参数时，Sidekiq 只会订阅 default 队列中的任务。
      def validate!
        options[:queues] << 'default' if options[:queues].empty?
      end
    ```
  - 默认，队列的优先级都为 1， 高优先级的队列能得到更多的执行机会。
  - 实现方法：通过增加同一个 `queues` 集合中高优先级队列的数量。
    ```ruby
      def parse_queue(opts, q, weight=nil)
        [weight.to_i, 1].max.times do
          (opts[:queues] ||= []) << q
        end
        opts[:strict] = false if weight.to_i > 0
      end
    ```

### 8. 异步任务的处理
  - `#perform_in`: 定时任务
    - `Sidekiq` 使用 `Scheduled::Poller` 对 `Redis` 中 `schedules` 有序集合中的负载进行处理。
    - 其中包括 `retry` 和 `schedule` 两个`有序集合`中的内容。
      ```ruby
        {
          "Redis" => [ "Retry", "Schedule" ]
        }
        # Retry: 包含了所有重试的任务
        # Schedule: 就是被安排到指定时间执行的定时任务了
      ```
    - 在 `Poller` 被 `Scheduled::Poller` 启动时会调用 `#start` 方法，开始对上述两个有序集合轮训。
      ```ruby
        # Scheduled::Poller#start
        def start
          @thread ||= safe_thread("scheduler") do
            initial_wait
            while !@done
              enqueue   # 入队[会调用下面的 Scheduled::Poll::Enq#enqueue_jobs()]
              wait      # 等待
            end
          end
        end
      ```
      ```ruby
        # SETS 即是 retry 和 schedule 构成的数组
        def enqueue_jobs(now=Time.now.to_f.to_s, sorted_sets=SETS)
          Sidekiq.redis do |conn|
            sorted_sets.each do |sorted_set|
              while job = conn.zrangebyscore(sorted_set, '-inf'.freeze, now, :limit => [0, 1]).first do
                if conn.zrem(sorted_set, job)
                  Sidekiq::Client.push(Sidekiq.load_json(job))
                end
              end
            end
          end
        end
        # Sidekiq 通过 `Redis#zrangebyscore` 和 `Redis#zrem` 将集合中小于当前时间的任务全部加到立即任务中。
        # 最终调用 Client#push，将任务推到指定的队列中。
      ```
    - `Scheduled::Poller` 并不是不停地对 `Redis` 中的数据进行处理的。
    - 因为当前进程一直都在执行 `Poller#enqueue` 其实是一个非常低效的方式。
    - 所以 `Sidekiq` 会在每次执行 `Poller#enqueue` 之后，执行 `Poller#wait` 随机等待一段时间。
      ```ruby
        def wait
          @sleeper.pop(random_poll_interval)
          # ...
        end

        def random_poll_interval
          poll_interval_average * rand + poll_interval_average.to_f / 2
        end

        # 随机等待时间的范围在 [0.5 * poll_interval_average, 1.5 * poll_interval_average] 之间；
        # 通过随机的方式，Sidekiq 可以避免在多个线程处理任务时，短时间内 Redis 接受大量的请求发生延迟等问题，能够保证从长期来看 Redis 接受的请求数是平均的；
      ```
    - 因为 `Scheduled::Poller` 使用了 `#enqueue` 加 `#wait` 对 Redis 中的数据进行消费，所以**没有办法保证任务会在指定的时间点执行**，**执行的时间一定比安排的时间要晚**，此一点需要特别注意。
    > 随机等待的时间不止与 `poll_interval_average` 有关，在默认情况下，它是当前进程数的 15 倍，在有 30 个 Sidekiq 线程时，每个线程会每隔 225 ～ 675s 的时间请求一次。
  - `#perform_async`: 立即任务

### 9. 执行任务
  - 定时任务是由 `Scheduled::Poller` 进行处理的，将需要执行的异步任务加入到指定的队列中。
  - 这些任务最终都会在 `Processor#run` 真正被执行：
    ```ruby
      def run
        begin
          while !@done
            process_one
          end
          @mgr.processor_stopped(self)
        rescue Exception => ex
          # ...
        end
      end
    ```
  - 当处理结束或者发生异常时会调用 `Manager#processor_stopped` 或者 `Manager#processor_died` 方法对 Processor 进行处理。
  - 在处理任务时其实也分为两个部分，也就是 `#fetch` 和 `#process`。
    ```ruby
      def process_one
        @job = fetch
        process(@job) if @job
        @job = nil
      end
    ```
  - 整个方法的调用栈：
    - 任务的获取从 `Processor#process_one`
    - 直到 `BasicFetch#retrive_work` 返回了 `UnitWork` 对象
    - 返回的对象会经过分发最后执行对应类的 `#perform` 传入参数真正运行该任务
    ```ruby
      # Processor#process_one
      # |____ Processor#fetch
      # |    |____ Processor#get_one
      # |       |____ BasicFetch#retrive_work
      # |           |---- Redis#brpop
      # |           |____ UnitOfWork#new
      # |
      # |_____ Processor#process
      #      |---- Processor#dispatch
      #      |---- Processor#execute_job
      #      |---- Worker#perform
    ```
  - 任务的获取：
    - `BasicFetch#retrive_work` 方法，他会从 Redis 中相应队列的有序数组中 `Redis#brpop` 出一个任务，然后封装成 `UnitOfWork` 对象后返回。
      ```ruby
        def retrive_work
          work = Sidekiq.redis { |conn| conn.brpop(*queues_cmd) }
          UnitOfWork.new(*work) if work
        end

        # #queues_cmd 方法用到了在第7节，主题的订阅一节中的 queues 参数。
        # 该参数会在 Processor 初始化时创建一个 BasicFetch 策略对象，最终在 BasicFetch#queues_cmd 方法调用时返回一个类似下面的数组。
        queue:hign
        queue:hign
        queue:hign
        queue:low
        queue:low
        queue:default
      ```
    - 如上所述，就可以实现队列的优先级这一个功能了。
    - 返回的 `UnitOfWork` 其实是一个通过 `Struct.new` 创建的结构体，它会在 `Processor#process` 方法中作为资源倍处理。
      ```ruby
        def process(work)
          jobstr = work.job
          queue = work.queue_name

          begin
            # ...

            job_hash = Sidekiq.load_json(jobstr)
            dispatch(job_hash, queue) do |worker|
              Sidekiq.server_middleware.invoke(worker, job_hash, queue) do
                execute_job(worker, cloned(job_hash['args'.freeze]))
              end
            end
          rescue Exception => ex
            # ...
          end
        end

        ## 上面程序的调用步骤：
        # 1. 将 Redis 中存储的字符串加载为 JSON；
        # 2. 执行 Processor#dispatch 方法并在内部提供方法重试等功能，同时也实例化一个 Sidekiq::Worker 对象；
        # 3. 依次执行服务端的中间件，可能会对参数进行更新；
        # 4. 调用 Processor#execute_job 方法执行任务；

        def execute_job(worker, cloned_args)
          worker.perform(*cloned_args)
        end
        # 该方法在**线程**中执行了客户端创建的 Worker 类的实例方法 #perform 并传入了经过两侧中间件处理后的参数。
      ```

### 10. 小结
  - 到目前为止，**Sidekiq Worker** 对任务的消费过程就是圆满的了，
  - 从客户端创建一个拥有 `#perform` 方法的 `Worker` 到消费者去执行该方法形成了一个闭环，完成了对任务的调度。
  - Sidekiq 是一个非常轻量级的任务调度系统。
  - 它使用 Redis 作为整个系统的消息队列，在两侧分别建立了生产者和消费者的模块。

### 11. 中间件
  - 中间件模块为整个任务的处理流程提供两个钩子，一个在客户端，另一个在 **Sidekiq Worker** 中。
    ```ruby
      # 默认所有的中间件都会拥有一个实例方法 #call 并接受 worker, job 和 queue 三个参数
      # 使用时也只需要直接调用 Chain#add 方法将其加入数组就可以了
      class AcmeCo::MyMiddleware
        def call(worker, job, queue)
          # ...
        end
      end

      # config/initializers/sidekiq.rb
      Sidekiq.configure_server do |config|
        config.server_middleware do |chain|
          chain.add AcmeCo::MyMiddleware
        end
      end
    ```
  - 服务端中间件：是【包围】了任务执行过程的，可以在中间件中使用 `begin`, `rescue` 语句，当任务出现问题时，我们就能拿到异常了。
  - 客户端中间件：在任务即将倍推入 `Redis` 之前运行，它能够阻止任务进入 `Redis` 并且允许我们在任务入队前对其进行修改了停止。

### 12. 中间件的实现
  - 跟任务有关的信息都会先通过一个预处理流程，c 端和 s 端两个中间件的链式调用都使用 `Middleware::Chain` 中的类进行处理的。
    ```ruby
      class Chain
        include Enumerable
        attr_reader :entries

        def initialize
          @entries = []
          yield self if block_given?
        end

        def remove(klass); end
        def add(klass, *args); end
        def prepend(klass, *args); end
        def insert_before(oldklass, newklass, *args); end
        def insert_after(oldklass, newklass, *args); end
      end
    ```
  - `Middleware::Chain` 中包含一系列的 `Entry`, 其中存储了中间件的相关信息。
    - 无论是 c 端还是 s 端都会在执行之前对每一个异步任务的参数执行 `invoke` 方法，
    - 调用 `Middleware::Chain` 对象中的所有中间件。
    ```ruby
      def invoke(*args)
        chain = retrieve.dup
        traverse_chain = lambda do
          if chain.empty?
            yield
          else
            chain.shift.call(*args, &traverse_chain)
          end
        end
        traverse_chain.call
      end

      # Chain#invoke 会对其持有的每一个中间件都执行 #call 方法。
      # 中间件都可以对异步任务的参数进行改变或者进行一些记录日志等操作。
      # 最后执行传入的 block 并返回结果。

      # args ----> ActiveRecord Middleware ----> I18n Middleware ----> Other Middleware ----> yield
    ```
  - 当异步队列入队时，会执行 `Client#process_single` 方法
  - 调用 **Sidekiq** 载入中的全部中间件
  - 最后返回新的 `item` 对象。
    ```ruby
      def process_single(worker_class, item)
        queue = item['queue'.freeze]
        middleware.invoke(worker_class, item, queue, @redis_pool) do
          item
        end
      end
    ```
  - 每一个 **Sidekiq Worker** 在处理中间件时也基本遵循相同的逻辑
  - 如 `#process` 方法会先执行各种中间件，最后再运行 `block` 中的内容。
    ```ruby
      def process(work)
        jobstr = work.job
        queue = work.queue_name

        begin
          # ...
          
          job_hash = Sidekiq.load_json(jobstr)
          Sidekiq.server_middleware.invoke(worker, job_hash, queue) do
            execute_job(worker, cloned(job_hash['args'.freeze]))
          end
        rescue Exception => ex
          # ...
        end
      end

      # 在 #execute_job 方法执行期间，由于异步任务可能抛出异常，在这时，
      # 我们注册的中间件就可以根据情况对异常进行捕获并选择是否对异常进行处理或者抛给上层。
    ```

### 13. 任务的重试
  - `JobRetry`
    - `Processor#dispatch` 调用了 `JobRetry#global` 方法捕获在异步任务执行过程中发生的错误：
      ```ruby
        def dispatch(job_hash, queue)
          pristine = cloned(job_hash)

          # ...
          @retrier.global(pristine, queue) do
            klass = constantize(job_hash['class'.freeze])
            worker = klass.new
            worker.jid = job_hash['jid'.freeze]
            @retrier.local(worker, pristine, queue) do
              yield worker
            end
          end
        end
      ```
    - `JobRetry#local` 和 `JobRetry#global`
      - 实现：将执行异步任务的 `block` 包在一个 `begin`, `rescue` 中，选择在合适的时间重试：
        ```ruby
          def local(worker, msg, queue)
            yield
          # ...
          rescue Exception => e
            raise Sidekiq::Shutdown if exception_caused_by_shutdown?(e)

            if msg['retry'] == nil
              msg['retry'] = worker.class.get_sidekiq_options['retry']
            end

            raise e unless msg['retry']
            attempt_retry(worker, msg, queue, e)
            raise Skip
          end
        ```
      - 如果在定义 `Worker` 时就禁用了重试，那么在这里就会直接抛出上册的异常，否则就会进入 `#attempt_retry` 方法安排任务进行重试：
        ```ruby
          def attempt_retry(worker, msg, queue, exception)
            max_retry_attempts = retry_attempts_from(msg['retry'], @max_retries)

            msg['queue'] =  if msg['retry_queue']
                              msg['retry_queue']
                            else
                              queue
                            end
            
            count = if msg['retry_count']
                      msg['retried_at'] = Time.now.to_f
                      msg['retry_count'] += 1
                    else
                      msg['failed_at'] = Time.now.to_f
                      msg['retry_count'] = 0
                    end

            if count < max_retry_attempts
              delay = delay_for(worker, count, exception)
              retry_at = Time.now.to_f + delay
              payload = Sidekiq.dump_json(msg)
              Sidekiq.redis do |conn|
                conn.zadd('retry', retry_at.to_s, payload) # 加入重试队列
              end
            else
              retries_exhausted(worker, msg, exception)
            end
          end
        ```
      - 任务的重试次数超过了限定的重试次数之后，会调用：
        - `#retries_exhausted`
        - `# send_to_morgue`
          ```ruby
            # 将任务的负载(payload)加入 DeadSet 对象中：
            def send_to_morgue(msg)
              payload = Sidekiq.dump_json(msg)
              DeadSet.new.kill(payload)
            end

          # 至此整个任务的重试过程就结束了。
          ```
        - **Sidekiq** 使用 `begin`, `rescue` 捕获整个流程中出现的异常
        - 并根据传入的 `retry_count` 参数进行重试。
      