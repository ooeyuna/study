5.22－6.4这段时间后台活动记录异常，体现为大多数管理的操作记录丢失。原因是sweeper采用单例模式，多线程共享了同一个对象变量导致的记录错误。

我们的activity代码类似于
  
```ruby
class StudyObserver < ActionController::Caching::Sweeper
  observe Subtitle

  def after_create(obj)
    p "call after_create admin:#{admin_id}"
    p "#{self}"
    p "#{controller}"
  end
  
  def after_update(obj)
    p "call after_update admin:#{admin_id}"
    p "#{self}"
    p "#{controller}"
  end
  
  private
  def admin_id
    @admin_id ||= "#{Random.new.rand(1..100)}"
  end
end

```

  最大的问题就出在
  ```
      @admin_id ||= "#{Random.new.rand(1..100)}"
  ```
  这一段，@admin_id作为对象的变量在第一次赋值后始终存在，不会再进行第二次赋值，导致其他管理员在create subtitle触发callback的时候记录的始终是第一个记录的管理员，现象上表现的也就是大部分管理员操作记录的丢失。
  
  在这个例子里我尝试输出了controller和observer的对象自身，结论与猜测的一致，controller针对于每一个请求都是一个新的对象，而sweeper每个请求都对应的同一个对象，即使是不同的controller对应的也是同一个sweeper对象，只有在rails初始化reload class的时候才会更改。
  
  ---
  
  后面是深入研究心得（吐槽）
   
sweeper注册代码如下

```ruby
    module Sweeping
      module ClassMethods #:nodoc:
        def cache_sweeper(*sweepers)
          configuration = sweepers.extract_options!

          sweepers.each do |sweeper|
            ActiveRecord::Base.observers << sweeper if defined?(ActiveRecord) and defined?(ActiveRecord::Base)
            sweeper_instance = (sweeper.is_a?(Symbol) ? Object.const_get(sweeper.to_s.classify) : sweeper).instance

            if sweeper_instance.is_a?(Sweeper)
              around_filter(sweeper_instance, :only => configuration[:only])
            else
              after_filter(sweeper_instance, :only => configuration[:only])
            end
          end
        end
      end
    end
```

  可以看到sweeper是new了一个observer对象并且作用于全局。也就是所有的controller都用的同一个sweeper
  
  sweeper的github的文档资料有点少，像代码里的变量controller我起初就完全不知道是从哪来的用来做啥，于是还是翻了一下源码：
  
  ```ruby
    class Sweeper < ActiveRecord::Observer #:nodoc:
      attr_accessor :controller

      def initialize(*args)
        super
        @controller = nil
      end

      def before(controller)
        self.controller = controller
        callback(:before) if controller.perform_caching
        true # before method from sweeper should always return true
      end

      def after(controller)
        self.controller = controller
        callback(:after) if controller.perform_caching
      end

      def around(controller)
        before(controller)
        yield
        after(controller)
      ensure
        clean_up
      end

      protected
      # gets the action cache path for the given options.
      def action_path_for(options)
        Actions::ActionCachePath.new(controller, options).path
      end

      # Retrieve instance variables set in the controller.
      def assigns(key)
        controller.instance_variable_get("@#{key}")
      end

      private
      def clean_up
        # Clean up, so that the controller can be collected after this request
        self.controller = nil
      end

      def callback(timing)
        controller_callback_method_name = "#{timing}_#{controller.controller_name.underscore}"
        action_callback_method_name     = "#{controller_callback_method_name}_#{controller.action_name}"

        __send__(controller_callback_method_name) if respond_to?(controller_callback_method_name, true)
        __send__(action_callback_method_name)     if respond_to?(action_callback_method_name, true)
      end

      def method_missing(method, *arguments, &block)
        return super unless @controller
        @controller.__send__(method, *arguments, &block)
      end
    end
  ```
比较有用的方法

  - before(controller)是在action执行之前，在这里用于为sweeper注册controller。在callback里按顺序先执行before\_controller，然后执行before\_action
  - after(controller)是在action执行之后，这里先执行after\_controller,再执行after\_action（有点不符合常理）
  - arround，可以在before和after中间的yield位置插入&block
  - assgins，获得controller对象的变量
  
比较有用的变量
  
  - controller，获得当前controller


这里为了实现多线程共享sweeper单例，在before和after都传入了controller并执行了赋值操作，就实现上来说还是比较丑陋的，同时也可能有隐患，在多线程的场景下，如果在第一个线程才执行完
  ``` self.controller = controller ```
还没来得及执行下一句的时候第二个线程也执行了这赋值语句，会导致第一个线程记录的controller对象错误。不过如果在上层调用的时候加了锁的话也就没太大问题了（但翻了一下\_run\_callback那部分的源码并没有发现加锁的操作，不知道是怎么解决这个问题的）。
  
  在StudyObserver实例里的after\_\*方法的对象实际和父类ActiveRecord::Observer的方法作用一致，监视的是model对象的行为，相当于在model对象上加上了after\_\*方法。虽然说是“观察者模式”，但sweeper依旧支持了before\_\*方法，支持回调的总列表应该是Activerecord::Callbacks中定义的array：
  
  ```ruby

    CALLBACKS = [
      :after_initialize, :after_find, :after_touch, :before_validation, :after_validation,
      :before_save, :around_save, :after_save, :before_create, :around_create,
      :after_create, :before_update, :around_update, :after_update,
      :before_destroy, :around_destroy, :after_destroy, :after_commit, :after_rollback
    ]
  ```    
不知道为什么after\_commit和after\_rollback始终不生效，可能是全局关掉了事务skip掉了这两个callback？

sweeper的初衷应该还是用before和after方法为controller提供切入点，不应该关心model对象的状态，但却又因继承自Observer，使得sweeper变得非常“万金油”，可以同时用于model对象的callback。

rails的callbacks的实现也很有意思，ActionRecord里大量的include module，然后再各个模块里定义了许多同名方法（比如save），然后用super关键字层层向上追溯父模块的调用，Callbacks模块放在比较前面的位置，实现了回调。callback的实现分为两部分，一部分是充斥大量class_eval魔法一般的set\_callback，另一部分是执行rails初始化后生成的\_run\_callback。代码层级错综复杂，也就只有炸堆栈（raise大法）才能看到堆栈调用的全貌了。
