---
name: ddd-kotlin-spring-jpa 
description: 当需要生成领域模型及相关代码(聚合 Aggregate, 领域事件 Domain Event, 领域命令 Domain Command, 领域服务 Domain Serivice, 命令处理器 CommandHandler， 领域事件监听器 Domain Event Listener, 值对象 Value Object, 聚合工厂 Factory， 聚合仓储 Repository)，并且使用 kotlin + sping-data-jpa 的时候，使用这个SKILL
---

# Generate kotlin code of DDD model

## 执行说明

所有领域对象的类名都应该贴近领域语言，更容易理解，更不容易混淆，更精确，同时避免冗长。

### 领域模块和非领域模块

项目中必须存在一个子项目作为领域模块。如何寻找领域模块
* 优先使用用户指定的模块作为领域模块
* 一般领域模块子项目名字叫 domain
* 子项目里有领域事件和领域命令对应的类的模块

**如果无法确认哪个模块是领域模块，必须让用户来告诉你**

实现领域模型功能，实现纯领域逻辑的代码应该写到领域模块中。具体的技术实现应该放到非领域模块中。这里的具体技术，比如
* 表达数据库访问的，比如直接拼接sql等
* 表达调用远程接口的，比如调用http接口、grpc接口等
* 使用某个外部程序库提供的通用成熟的算法库的

像 annotation、日志这样的对业务没有什么侵入的代码是可以在领域模块中使用。


### 领域模块内包结构

将某个聚合相关的代码，写入到一个单独的包里， 包结构大概如下 `<prefix>.domain.<agg>` 。这个包应该用这个聚合的名字的小写命名。这个包下有以下文件

#### Events.kt 

里面定义这个聚合要发出的领域事件，及相关的代码。例如

```kotlin
class ProjectModifiedEvt(
    val projectId: ProjectId,
    override val eventId: String,
    override val eventTime: LocalDateTime,
    val oldName: String,
    val newName: String,
    val oldDescription: String?,
    val newDescription: String?
) : DomainEvent {
    override val aggType: String = AGG_TYPE_PROJECT
    override val aggId: String
        get() = projectId.id
}

class ProjectDeletedEvt(
    val projectId: ProjectId,
    override val eventId: String,
    override val eventTime: LocalDateTime,
    val name: String,
    val description: String?
) : DomainEvent {
    override val aggType: String = AGG_TYPE_PROJECT
    override val aggId: String
        get() = projectId.id
}

const val AGG_TYPE_PROJECT = "Project"
```

#### <Agg>.kt 文件

定义聚合的文件，文件名如 `ProjectAgg.kt` `OrderAgg.kt` 。

#### <Factory>.kt 文件

定义聚合工厂的文件，文件名如 `ProjectFactory.kt` `OrderFactory.kt`

#### <Repository>.kt 文件

定义仓储接口的文件，文件名如 `ProjectRepository.kt` `OrderRepository.kt` 。

#### <Cmd>.kt 文件
定义聚合命令的文件，文件名如 `PlaceOrderCmd.kt` `SendMailCmd.kt` 。 聚合命令，及其 `Handler` 类都定义到这个文件里。比如

```kotlin
class CancelOrderCmd(
	val orderId: OrderId 
)

@Component
class CancelOrderCmdHandler(
	private val orderRepository: OrderRepository
) {
    fun handle(cmd: CancelOrderCmd) {
		val order = this.orderRepository.findByIdOrError(cmd.orderId)
		order.cancel()
		this.orderRepository.save(order)
	}
}
```

### 领域模块外包结构

领域模块外采用和领域模块内同样的包，在这个包内定义具体技术实现的类。

### 生成值对象

遇到以下情况，考虑建值对象类
* 直接用原始数据类型不好表达业务概念，不好限制数据格式，比如直接用 Long 表示订单Id，不如定义一个 `class OrderId(val id: Long)` 来封装
* 有多个数值项总是一起被重复使用，这种时候建值对象封装它们，让代码更简洁

### 领域事件的发布和监听

在聚合类内，做完业务动作后，立刻调用 `publishEvent()` 函数去发布领域事件。在仓储的实现类里，持久化聚合后，调用 `DomainEventDispatcher.dispatchNow()` 函数触发领域事件的投递。

注意：
* 禁止在聚合内持有领域事件，然后在仓储实现里从聚合内取出领域事件去发布
* 禁止在聚合以外发布领域事件，除非用户明确说明
* 禁止聚合执行完业务函数后，通过返回值返回领域事件

如果找不到 `publishEvent()` `DomainEventDispatcher.dispatchNow()` 函数，参考  [domain-events-framework](domain-events-framework.md) 去生成框架代码。

使用 spring 框架提供的 `@EventListner` 去监听领域事件。



### 生成领域事件代码

* 领域事件类要以 `Evt` 结尾，前面要用英文的完成时态，表达某个动作已完成，比如  `OrderPlacedEvt`  `MailSentEvt`
* 领域零件里必须包含以下字段
	- eventTime 领域事件发生的时间
	- eventType 领域事件的类型，表示是哪种领域事件，通常是和领域事件名类似的一段文本
	- aggType   发出这个领域事件的聚合的类型
	- aggId     发出这个领域事件的聚合的id
* 领域事件还可以包含其它和业务相关的数据字段
* 领域事件是不可变的
* 领域事件不允许引用聚合、领域服务、命令等其它对象，只能引用值对象
* 生成注释，说明接收到本事件的时候，表示什么事情已经发生了，说明每个字段的含义

例子:
```kotlin
class OrderPlacedEvt(
	val orderId: OrderId,
	val orderItems: List<OrderItem>,
	
	val eventTime: LocalDateTime,
	val eventType: String,
	val aggType: String,
	val aggId: String
)
```

### 生成领域命令代码

* 领域命令类要以 `Cmd` 结尾，前面要用一般现在时态，表达要执行某个动作，比如 `PlaceOrderCmd` `SendMailCmd`
* 根据具体场景设计命令里的字段
* 命令是不可变的对象
* 命令不能引用聚合、领域服务等对象，只能引用值对象
* 如果遗留代码的领域命令都实现某个接口或继承自某个类，你也必须遵循旧的规则，去实现那个接口或者继承那个类
* 生成注释，说明执行该命令要达成的业务意图，说明每个字段的含义


下面是一个kotlin的例子
```kotlin
class PlaceOrderCmd(
	val userId: String,
	val goodsId: Long,
	val goodsCount: Int
)
```

### 生成聚合代码

#### 聚合接口

* 每个聚合必须有一个接口，定义该聚合对外暴露什么
* 聚合的接口名字用 `Agg` 结尾，比如 `OrderAgg`
* 只有聚合的工厂和仓储的具体实现类需要知道聚合的具体类型，其它领域对象只看到聚合的接口类型
* 聚合接口里只定义必须给别的领域对象使用到的属性和函数

例如：
```kotlin

interface OrderAgg {
	val orderId: OrderId
	
	fun cancel()
}
```

#### 聚合实现类

根据如何持久化聚合，可以分为以下 4 种聚合的实现方式
* 直接映射模式
* 子类委托模式
* 全量合并模式
* 纯接口模式

**你必须在生成聚合代码之前，根据当前聚合的情况去选择一种聚合实现的方式。如果不发决定，必须告知用户，让用户来选择**

##### 直接映射模式

在领域模块内定义一个具体类实现聚合接口，同时这个类也是JPA的数据库表映射类。

适用场景：聚合内数据字段比较简单，很容易直接映射到数据库表上。

例如：

```kotlin

@Entity
@Table("t_order")
class OrderAggImpl(

	@EmbeddedId
	val id: OrderId,
	
	@OneToMany
	val items: List<OrderItem>,
	
	@Embedded
	val userId: UserId,
	
	val totalAmount: BigDecimal,
	
	var orderStatus: OrderStatus
): OrderAgg {
	override fun cancel() {
		if (orderStatus == OrderStatus.Canceled) {
			return
		}
		
		this.orderStatus = OrderStatus.Canceled
		
		publishEvent(OrderCanceledEvent(orderId = this.id))
	}
}
```

##### 子类委托模式

在领域模块内定义一个抽象类AggImpl实现聚合接口Agg，AggImpl内完成业务功能，定义抽象属性或者抽象函数来存取数据项。在领域模块外定义一个类Table作为数据库的映射类。在领域模块外定义一个具体类AggDbImpl来继承AggImpl，通过使用Table来实现AggImpl里的数据访问。

适用场景： 
* 领域模块内的聚合类AggImpl无法直接通过JPA映射到某个表上
	* 比如，要映射到多张表
	* 比如，某些类型不好映射
* 类AggImpl对数据的访问比较简单，可以通过委托给数据库表映射类来实现


例如，在领域模块内
```kotlin
abstract class OrderImpl(
	
	abstract val id: OrderId,
	
	abstract val userId: UserId,
	
	abstract var orderStatus: OrderStatus
): OrderAgg {
	override fun cancel() {
		if (orderStatus == OrderStatus.Canceled) {
			return
		}
		
		this.orderStatus = OrderStatus.Canceled
		
		publishEvent(OrderCanceledEvent(orderId = this.id))
	}
}
```


在领域模块外
```kotlin
@Entity
@Table("t_order")
class OrderTable(

	@Id
	val id: Long,
	
	val userId: Long,
	
	@Enumerated(EnumType.STRING)
	var orderStatus: String
)

class OrderAggDBImpl(
	val orderTable: OrderTable
): OrderImpl {
	override val id: OrderId
		get() = OrderId(orderTable.id)

	override val userId: Long
		get() = UserId(orderTable.userId)
		
	override val orderStatus: OrderStatus
		get() = OrderStatus.valueOf(orderTable.orderStatus)
		set(data) { orderTable.orderStatus = data.name() }
}
```

##### 全量合并模式

领域模块内定义类 AggImpl 实现聚合接口，持有需要使用到的数据。领域模块外，定义 Table 类映射到数据库表。领域模块外仓储实现类里查询时，根据Table内的数据去创建AggImpl对象，保存的时候，重新查询Table，把AggImpl持有的数据合并到Table上，再持久化。

适用场景：
* 聚合内数据结构复杂，难以映射到数据库表上
* 聚合的数据来源复杂，难以被JPA直接映射

##### 纯接口模式

领域模块内只定义了聚合接口，没有实现。在领域模块外定义类AggImpl，实现领域接口，该类通过直接操作数据库或者外部接口来实现业务函数。

适用场景：
* 需要通过直接数据库操作等来改进性能
* 聚合功能过于简单，只有CRUD，以后也不太会有复杂业务功能

例如，领域模块内定义聚合接口
```kotlin
interface OrderAgg {
	val orderId: OrderId
	
	fun cancel()
}
```

领域模块外实现
```kotlin
class OrderImpl(
	override val orderId: OrderId,
	val orderDao: OrderDao
): OrderAgg  {
	override fun cancel() {
		orderDao.updateOrderStatus(orderId.id, OrderStatus.Cancel.name())
		publishEvent(OrderCanceledEvent(orderId = this.id))
	}
}
```

### 生成聚合工厂

工厂类的名字必须以 `Factory` 结尾。工厂类只负责从业务角度从无到有创建聚合，仓储对象把已经持久化的聚合查询出来的时候，不要使用工厂。

#### 抽象工厂

##### 何时使用抽象工厂
当聚合的具体实现类不在领域模块内部的时候，采用抽象工厂模式。
比如实现聚合时使用了纯接口模式、子类委托模式的时候。

##### 如何使用抽象工厂

领域模块内聚合工厂类定义成抽象类，定义抽象函数来生成聚合，然后完成初始化聚合。在领域模块外定义聚合工厂的具体类，实现实例化聚合的函数。

例如，领域内

```kotlin
abstract class OrderFactory {
	/**
	 * 负责初始化聚合的业务逻辑，返回聚合的接口类型
	 */
	fun create(cmd: PlaceOrderCmd): OrderAgg {
		val orderAggImpl = newInstance(cmd)
		
		// 此处生成聚合初始化代码
		
		publishEvent(OrderPlacedEvent(orderId = orderAggImpl.id))
		return OrderAggImpl
	}

	/**
	 * 留给子类去实例化聚合
	*/
	protected abstract fun newInstance(cmd: PlaceOrderCmd): OrderAggImpl
}
```

领域模块外
```kotlin
class OrderFactoryImpl: OrderFactory() {
	override fun newInstance(cmd: PlaceOrderCmd): OrderAggImpl {
		// 用领域模块外定义的聚合的具体类来实例化
		return OrderAggDBImpl()
	}
}
```

### 生成聚合仓储

在领域模块内为每个聚合生成一个仓储接口，命名用 `Repository` 结尾。在领域模块外，定义仓储的实现类，实现仓储接口。

注意：
* 仓储接口只服务于领域模块内部存取聚合使用，禁止定义函数专门给领域模块外使用
* 除非用户明确指定，否则禁止在仓储内定义对聚合部分修改的函数
* 保存聚合后，调用 `DomainEventDispatcher.dispatchNow()` 去触发派发领域事件的动作
* 仓储接口中一般有 
  * findById 函数，根据id查询，查询不到的聚合的时候，返回null
  * findByIdOrError 函数，根据id查询，在查询不到的时候抛异常，只会返回不为null的值

根据实现聚合采用的模式，去决定仓储如何存取聚合。


### 生成领域监听器

对于事件风暴建模出的Policy类型的领域对象，生成事件监听对象，命名以 `Listener` 结尾。

例如
```kotlin

@Component
class GiveCreditOnOrderPlacedListener(
	private val giveCreditCmdHandler: GiveCreditCmdHandler
) {
	@EventListener
	fun on(event: OrderPlacedEvent) {
		giveCreditCmdHandler.handle(GiveCreditCmd(event.userId))
	}
}
```

## 最佳实践

* 能够隐藏到聚合中的功能，就不要放到领域服务中(CommandHandler也是一种领域服务)
* 创建聚合时的业务逻辑，优先在聚合工厂里实现，而不是在CommandHandler里实现
* 通过在 DomainRegistry 中定义函数来让聚合可以得到spring 管理的 bean 的引用
* 可以把命令作为传递给聚合、聚合工厂等，以避免传递太多命令里的数据