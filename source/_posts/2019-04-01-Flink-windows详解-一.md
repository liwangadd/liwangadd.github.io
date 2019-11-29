---
title: Flink windows详解(一)
tags:
  - 分布式计算
  - Flink
categories:
  - 大数据
abbrlink: f5b05e29
date: 2019-04-01 14:27:13
---

#### Window

window代表一个有限数据的集合，一个窗口有一个maxTimestamp，表示在该时刻窗口中的所有元素都已经到达。在Flink内部window有两个实现类：

- GlobalWindow：没有起止时间的窗口，运行期间接收到的所有元素都会被放在同一个窗口中。

- TimeWindow：该窗口会维护窗口的起止时间，同时为了满足sessionWindows的需要，还通过静态函数`mergeWindows()`提供了窗口合并的功能（将具有重合部分的窗口合并为一个窗口）。

  ```java
  	
  	public static void mergeWindows(Collection<TimeWindow> windows, MergingWindowAssigner.MergeCallback<TimeWindow> c) {
  		// 当用户使用继承自MergingWindowAssigner的类时，就需要涉及到窗口的合并，默认是委托给该函数实现的
  		List<TimeWindow> sortedWindows = new ArrayList<>(windows);
          // 根据窗口的左边界时间排序
  		Collections.sort(sortedWindows, new Comparator<TimeWindow>() {
  			@Override
  			public int compare(TimeWindow o1, TimeWindow o2) {
  				return Long.compare(o1.getStart(), o2.getStart());
  			}
  		});
          // key为合并后的窗口，value为被合并的窗口集合
  		List<Tuple2<TimeWindow, Set<TimeWindow>>> merged = new ArrayList<>();
  		Tuple2<TimeWindow, Set<TimeWindow>> currentMerge = null;
  		for (TimeWindow candidate: sortedWindows) {
  			if (currentMerge == null) {
  				currentMerge = new Tuple2<>();
  				currentMerge.f0 = candidate;
  				currentMerge.f1 = new HashSet<>();
  				currentMerge.f1.add(candidate);
  			} else if (currentMerge.f0.intersects(candidate)) {
                  // 窗口存在重叠，得到重叠窗口合并后的窗口
  				currentMerge.f0 = currentMerge.f0.cover(candidate);
  				currentMerge.f1.add(candidate);
  			} else {
                  // 窗口与之前的窗口没有重叠，将之前的窗口添加到List中，开启新一轮的合并
  				merged.add(currentMerge);
  				currentMerge = new Tuple2<>();
  				currentMerge.f0 = candidate;
  				currentMerge.f1 = new HashSet<>();
  				currentMerge.f1.add(candidate);
  			}
  		}
  		if (currentMerge != null) {
  			merged.add(currentMerge);
  		}
  		for (Tuple2<TimeWindow, Set<TimeWindow>> m: merged) {
  			if (m.f1.size() > 1) {
                  // 若该窗口是合并之后得到的，调用传入的方法进行回调通知
  				c.merge(m.f1, m.f0);
  			}
  		}
  	}
  
  	// 发生在指定时间段的元素聚合到同一窗口
  	public static long getWindowStartWithOffset(long timestamp, long offset, long windowSize) {
  		return timestamp - (timestamp - offset + windowSize) % windowSize;
  	}
  ```

<!-- more -->

#### WindowAssigner

WindowAssigner 用于决定将到达的元素分配给哪个window，在flink内部window和key决定了该元素应该存放在哪些窗格中，当trigger被触发时就会将窗口函数应用到窗格中的元素上并输出结果元素。该抽象类内部提供了如下核心函数：

- `assignWindows(T element, long timestamp, WindowAssignerContext context)`：每一个元素的到达都会调用该函数，返回该元素所属窗口的集合。
- `getDefaultTrigger(StreamExecutionEnvironment env)`：当用户没有指定trigger时，通过调用该函数获得默认的trigger。
- `isEventTime()`：返回该`WindowAssigner`是使用`ProcessingTime`还是`EventTime`决定元素窗口的分配

![WindowAssigner](../../../../Pictures/WindowAssigner.jpg)

- `Tumbling Windows`的构造函数需要两个参数：`size`(窗口大小)，`offset`(时间偏移量)。生成的窗口具有固定大小并且窗口间没有重合。![img](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/tumbling-windows.svg)

  ```java
  /**
  * TumblingProcessingTimeWindows类
  */
  public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
      	// 得到当前的机器时间，内部通过System.currentTimeMillis()获取
  		final long now = context.getCurrentProcessingTime();
      	// 计算元素所在窗口的左边界时间
  		long start = TimeWindow.getWindowStartWithOffset(now, offset, size);
  		return Collections.singletonList(new TimeWindow(start, start + size));
  }
  
  /**
  * TumblintEventTimeWindows类
  */
  @Override
  	public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
          // 和TumblingProcessingTimeWindows类似，不过不是获取的当前机器时间，而是直接使用传入的timestamp参数，即元素的EventTime
  		if (timestamp > Long.MIN_VALUE) {
  			// Long.MIN_VALUE is currently assigned when no timestamp is present
  			long start = TimeWindow.getWindowStartWithOffset(timestamp, offset, size);
  			return Collections.singletonList(new TimeWindow(start, start + size));
  		} else {
  			....
  		}
  	}
  ```

- `Sliding Windows`的构造函数需要三个参数：`size`(窗口大小)，`slide`(窗口滑动距离)，`offset`(时间偏移量)。生成的窗口同样具有固定大小，但是窗口每次都滑动距离不再是窗口大小，而是自定义的slide长度，因此窗口之间存在重叠。![img](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/sliding-windows.svg)

  ```java
  /**
  * SlidingProcessingTimeWindows类
  */
  public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
  		timestamp = context.getCurrentProcessingTime();
  		// 使用Sliding Windows时，一个元素可能被划分到多个的窗口中，使用List容器返回该元素所属的窗口
  		List<TimeWindow> windows = new ArrayList<>((int) (size / slide));
      	// 获取该元素所属最后一个窗口的左边界时间
  		long lastStart = TimeWindow.getWindowStartWithOffset(timestamp, offset, slide);
      	// 不同窗口的滑动距离为slide，每次递减slide，依次添加前一个窗口
  		for (long start = lastStart;
  			start > timestamp - size;
  			start -= slide) {
  			windows.add(new TimeWindow(start, start + size));
  		}
  		return windows;
  }
  
  /**
  * SlidingEventTimeWindows类
  */
  public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
  		if (timestamp > Long.MIN_VALUE) {
              // 同理，使用的是timestamp，即元素的EventTime
  			List<TimeWindow> windows = new ArrayList<>((int) (size / slide));
  			long lastStart = TimeWindow.getWindowStartWithOffset(timestamp, offset, slide);
  			for (long start = lastStart;
  				start > timestamp - size;
  				start -= slide) {
  				windows.add(new TimeWindow(start, start + size));
  			}
  			return windows;
  		} else {
              // 如果timestamp == Long.MIN_VALUE，说明元素没有EventTime，抛出异常
  			......
  		}
  	}
  ```

  

- `Session Windows`继承自`MergingWindowAssigner`，从名字就可以看出生成的窗口是可以合并的，使用这个窗口分配类的时候，每个元素的出现都会创建一个新的窗口，并和之前保存的窗口进行比较，当存在窗口重叠时会将重叠的窗口合并。和前两种不同，该类生成的窗口没有固定的起始时间，窗口的大小也是不确定的。从类图中可以看出，`Session Windows`分为两种：Dynamic开头的和不是Dynamic开头的，前者用户需要传入一个`SessionWindowTimeGapExtractor`接口的实现类，根据当前元素动态返回sessionTimeout；后者直接在构造函数中出固定的sessionTimeout。![img](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/session-windows.svg)

- `Global Windows`相同的key只存在一个窗口，所有具有相同key的元素都会进入该窗口，同时默认返回的trigger也是`NeverTrigger`类型，表示永远不会触发窗口计算。

#### Trigger

Trigger 决定了一个窗格 pane 中的元素触发计算窗口函数的时机，按照上面对 WindowAssigner 的描述，一个元素可以被分到多个 window，按照 key和window 做 grouping 的元素集组成一个窗格，每个窗格都有自己的 Trigger 实例。Trigger 管理了 event-timer/processing-timer 的注册；同时在处理元素以及对应的 timer event 在 WindowOperator 上触发时**决策**是否真正触发计算[ FIRE/CONTINUE ]。该抽象类提供了如下函数：

- onElement: 处理每个添加到窗格中的元素
- onEventTime: TriggerContext 注册的 event-time timer 被触发时调用
- onProcessingTime: TriggerContext 注册的 processing-time timer 被触发时调用
- onMerge: window 被 merge 时触发

Flink根据trigger的返回值即`TriggerResult`来决定接下来如何操作，`TriggerResult`是一个枚举类，内部有两个参数：`fire`(触发窗口计算)、`purge`(清空窗口元素)。共定义了4个枚举常量：`CONTINUE`(不触发计算也不清空元素，继续等待下一个元素)、`FIRE_AND_PURGE`(触发计算并清空元素)、`FIRE`(触发计算，但不清空)、`PURGE`(清空窗口中的元素)。

![trigger](../../../../Pictures/trigger.jpg)

典型的 trigger 实现用途：

- ContinuousEventTimeTrigger：指定的 interval 或 event window 的边界小于当前 watermark 时触发 onEventTime 计算。

  ```java
  public class ContinuousEventTimeTrigger<W extends Window> extends Trigger<Object, W> {
      ......
      @Override
  	public TriggerResult onElement(Object element, long timestamp, W window, TriggerContext ctx) throws Exception {
  		if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
  			// 如果窗口的右边界大于小于等于当前的watermark触发计算
  			return TriggerResult.FIRE;
  		} else {
              // 否则将窗口右边界注册为触发时间
  			ctx.registerEventTimeTimer(window.maxTimestamp());
  		}
  		// 获取保存的由Interval触发计算的时间
  		ReducingState<Long> fireTimestamp = ctx.getPartitionedState(stateDesc);
  		if (fireTimestamp.get() == null) {
              // 之前没有保存的状态，计算Interval触发计算的时间并注册
  			long start = timestamp - (timestamp % interval);
  			long nextFireTimestamp = start + interval;
  			ctx.registerEventTimeTimer(nextFireTimestamp);
  			fireTimestamp.add(nextFireTimestamp);
  		}
  		return TriggerResult.CONTINUE;
  	}
  
  	@Override
  	public TriggerResult onEventTime(long time, W window, TriggerContext ctx) throws Exception {
  		if (time == window.maxTimestamp()){
              // 窗口右边界触发计算
  			return TriggerResult.FIRE;
  		}
  		ReducingState<Long> fireTimestampState = ctx.getPartitionedState(stateDesc);
  		Long fireTimestamp = fireTimestampState.get();
  		// interval触发，清空之前保存的状态并保存计算出的新的interval触发时间
  		if (fireTimestamp != null && fireTimestamp == time) {
  			fireTimestampState.clear();
  			fireTimestampState.add(time + interval);
  			ctx.registerEventTimeTimer(time + interval);
  			return TriggerResult.FIRE;
  		}
  
  		return TriggerResult.CONTINUE;
  	}
      ......
  }
  ```

- ContinuousProessingTimeTrigger：每隔Interval触发一次，触发后将当前时间+interval注册为下一次触发时间。

  ```java
  public class ContinuousProcessingTimeTrigger<W extends Window> extends Trigger<Object, W> {
      ......
      @Override
  	public TriggerResult onElement(Object element, long timestamp, W window, TriggerContext ctx) throws Exception {
          // 由于该触发器是基于机器时间的，故不需要判断watermark，只需要注册interval的触发时间
  		ReducingState<Long> fireTimestamp = ctx.getPartitionedState(stateDesc);
  		timestamp = ctx.getCurrentProcessingTime();
  		if (fireTimestamp.get() == null) {
  			long start = timestamp - (timestamp % interval);
  			long nextFireTimestamp = start + interval;
  			ctx.registerProcessingTimeTimer(nextFireTimestamp);
  			fireTimestamp.add(nextFireTimestamp);
  			return TriggerResult.CONTINUE;
  		}
  		return TriggerResult.CONTINUE;
  	}
  
  	@Override
  	public TriggerResult onProcessingTime(long time, W window, TriggerContext ctx) throws Exception {
  		ReducingState<Long> fireTimestamp = ctx.getPartitionedState(stateDesc);
  		// 判断传入的时间是否等于上次注册的时间，true则触发计算并注册新的触发时间
  		if (fireTimestamp.get().equals(time)) {
  			fireTimestamp.clear();
  			fireTimestamp.add(time + interval);
  			ctx.registerProcessingTimeTimer(time + interval);
  			return TriggerResult.FIRE;
  		}
  		return TriggerResult.CONTINUE;
  	}
      ......
  }
  ```

- CountTrigger: 当窗口内的元素个数大于构造函数传入的maxCount时触发窗口计算。

  ```java
  public class CountTrigger<W extends Window> extends Trigger<Object, W> {
      ......
      @Override
  	public TriggerResult onElement(Object element, long timestamp, W window, TriggerContext ctx) throws Exception {
          // 由于是根据窗口元素个数决定是否触发计算，所以该方法是唯一可能触发计算的地方
          // onEventTime和onProcessingTime都只返回Continue
  		ReducingState<Long> count = ctx.getPartitionedState(stateDesc);
  		count.add(1L);
          // 统计窗口中元素个数，大于maxCount时触发计算
  		if (count.get() >= maxCount) {
              // 清空计数器，用于下一次计数
  			count.clear();
  			return TriggerResult.FIRE;
  		}
  		return TriggerResult.CONTINUE;
  	}
      ......
  }
  ```

- DeltaTrigger: 用户需要提供`deltaFunction`函数，用于计算当前元素和上次触发窗口计算元素之间的距离，当距离大于构造函数传入的`threshold`时，触发计算。

  ```java
  public class DeltaTrigger<T, W extends Window> extends Trigger<T, W> {
      ......
      /**
  	 * @param threshold 触发计算的阀值，当deltaFunction的返回值大于该阀值时触发
  	 * @param deltaFunction delta计算函数，该函数会传入两个参数：上次触发计算的元素和当前进入窗口的元素
  	 */
      private DeltaTrigger(double threshold, DeltaFunction<T> deltaFunction, TypeSerializer<T> stateSerializer) {
  		this.deltaFunction = deltaFunction;
  		this.threshold = threshold;
  		stateDesc = new ValueStateDescriptor<>("last-element", stateSerializer);
  	}
      
  	@Override
  	public TriggerResult onElement(T element, long timestamp, W window, TriggerContext ctx) throws Exception {
  		ValueState<T> lastElementState = ctx.getPartitionedState(stateDesc);
  		if (lastElementState.value() == null) {
              // 状态不存在，则更新为当前元素
  			lastElementState.update(element);
  			return TriggerResult.CONTINUE;
  		}
          // delta的值大于threshold时，触发计算并更新状态为当前元素
  		if (deltaFunction.getDelta(lastElementState.value(), element) > this.threshold) {
  			lastElementState.update(element);
  			return TriggerResult.FIRE;
  		}
  		return TriggerResult.CONTINUE;
  	}
      ......
  }
  ```

- EventTimeTrigger：event window 的边界小于当前 watermark 时触发 onEventTime 计算

  ```java
  public class EventTimeTrigger extends Trigger<Object, TimeWindow> {
      ......
      @Override
  	public TriggerResult onElement(Object element, long timestamp, TimeWindow window, TriggerContext ctx) throws Exception {
          // 当前窗口的右边界小于等于watermark时触发计算
  		if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
  			return TriggerResult.FIRE;
  		} else {
              // 否则注册触发时间，当大于该触发时间的watermark到达时，会回调onEventTime函数
  			ctx.registerEventTimeTimer(window.maxTimestamp());
  			return TriggerResult.CONTINUE;
  		}
  	}
  
  	@Override
  	public TriggerResult onEventTime(long time, TimeWindow window, TriggerContext ctx) {
          // watermark到达后，所有小于watermark的注册事件的回调方法
  		return time == window.maxTimestamp() ?
  			TriggerResult.FIRE :
  			TriggerResult.CONTINUE;
  	}
      ......
  }
  ```

- ProessingTimeTrigger：event window 的边界小于当前机器时间时触发 onProcessingTime 计算

  ```java
  public class ProcessingTimeTrigger extends Trigger<Object, TimeWindow> {
      ......
      @Override
  	public TriggerResult onElement(Object element, long timestamp, TimeWindow window, TriggerContext ctx) {
          // 元素到达时，直接将当前元素所在窗口的右边界注册为触发时间，并不会触发计算
  		ctx.registerProcessingTimeTimer(window.maxTimestamp());
  		return TriggerResult.CONTINUE;
  	}
      ......
      @Override
  	public TriggerResult onProcessingTime(long time, TimeWindow window, TriggerContext ctx) {
          // 触发时间到达后的回调方法，直接触发窗口的计算
  		return TriggerResult.FIRE;
  	}
      ......
  }
  ```

#### Evitor

  用于在窗口函数触发前或者触发后，删除窗口中不符合条件的元素，其提供了如下两个函数用于实现该功能：

- `evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);`在窗口计算前，会首先调用该方法
- `evictAfter(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);`在窗口计算后，调用该方法

- `CountEvictor`：如果窗口中的元素大于maxCount，就从窗口的开头删除元素，直到窗口内的元素等于maxCount
- `TimeEvictor`：首先获取窗口中所有元素的最大时间戳，使用该时间戳减去窗口大小得到窗口中元素应具有的最小时间戳，遍历删除所有时间戳小于该值的元素
- `DeltaEvictor`：首先获取窗口最后一个元素，然后一次遍历窗口中的元素，将该元素和最后一个元素传递给用户定义的`DeltaFunction`，如果返回值大于传入的threshold，则删除