# Domain Events Framework

## 领域模块内

```kotlin
import org.springframework.context.ApplicationContext
import java.time.LocalDateTime

interface DomainEvent {
    val eventId: String
    val eventTime: LocalDateTime
    val aggType: String
    val aggId: String
}

interface DomainEventPublisher {
    fun publishEvent(event: DomainEvent)
}

object DomainRegistry {
    lateinit var applicationContext: ApplicationContext
}

fun publishEvent(event: DomainEvent) {
    DomainRegistry.applicationContext.getBean(DomainEventPublisher::class.java).publishEvent(event)
}

interface DomainEventIdGenerator {
    fun nextEventId(): String
}

fun nextEventId(): String {
    return DomainRegistry.applicationContext.getBean(DomainEventIdGenerator::class.java).nextEventId()
}
```


## 领域模块外

```kotlin
import com.github.f4b6a3.uuid.UuidCreator
import org.slf4j.LoggerFactory
import org.springframework.context.ApplicationContext
import org.springframework.stereotype.Component
import org.springframework.transaction.TransactionExecution
import org.springframework.transaction.TransactionExecutionListener
import org.springframework.transaction.support.TransactionSynchronization
import org.springframework.transaction.support.TransactionSynchronizationManager

/**
 * 仓储持久化聚合后 ，调用 dispatchNow 触发立即投递领域事件
 */
interface DomainEventDispatcher {
    fun dispatchNow(aggType: String, aggId: String)
}

@Component
class SpringTxResourceDomainEventPublisherDelegate(
    applicationContext: ApplicationContext
) : DomainEventDispatcher {
    init {
        SpringTxResourceDomainEventDispatcher.applicationContext = applicationContext
        // Initialize DomainRegistry for backward compatibility
        DomainRegistry.applicationContext = applicationContext
    }

    override fun dispatchNow(aggType: String, aggId: String) {
        SpringTxResourceDomainEventDispatcher.dispatchNow(aggType, aggId)
    }
}


/**
 * 绑定到spring transaction 上的领域事件投递器
 */
object SpringTxResourceDomainEventDispatcher : DomainEventDispatcher {

    var applicationContext: ApplicationContext? = null
        set(value) {
            logger.info("SpringTxResourceDomainEventDispatcher's applicationContext is set to $value")
            field = value
        }

    override fun dispatchNow(aggType: String, aggId: String) {
        val spring = applicationContext ?: throw IllegalStateException("SpringTxResourceDomainEventDispatcher's applicationContext is null.")
        txResourceEventsMap().remove(AggregateKey(aggType, aggId))?.let { events ->
            events.forEach { spring.publishEvent(it)}
        }
    }

    fun txResourceEventsMap() : MutableMap<AggregateKey, MutableList<DomainEvent>> {
        val txResourceEventsMap = TransactionSynchronizationManager.getResource(DOMAIN_EVENTS_RESOURCE_KEY)
            ?: throw IllegalStateException("Domain events map tx resource not exists. Make sure 1. you publish domain events in a transaction 2. RegisterTxResourceSyncAfterBegin is registered to the transaction manager.")

        return txResourceEventsMap as? MutableMap<AggregateKey, MutableList<DomainEvent>>
            ?: throw IllegalStateException("Domain events map tx resource is not a map.")
    }

    fun addEvent(event: DomainEvent) {
        val events = txResourceEventsMap().getOrPut(AggregateKey(event.aggType, event.aggId)) { ArrayList() }
        events.add(event)
    }

    fun existUnDispatchedEvents(): Boolean {
        return txResourceEventsMap().any { it.value.isNotEmpty() }
    }

    fun dispatchAllNow() {
        val eventsMap = txResourceEventsMap()
        while (eventsMap.isNotEmpty()) {
            val entry = eventsMap.entries.first()
            dispatchNow(entry.key.aggType, entry.key.aggId)
        }
    }

    fun cleanAll() {
        TransactionSynchronizationManager.unbindResource(DOMAIN_EVENTS_RESOURCE_KEY)
    }

    fun initDomainEventsMap() {
        val domainEventsMap = TransactionSynchronizationManager.getResource(DOMAIN_EVENTS_RESOURCE_KEY)
        if (domainEventsMap != null) {
            logger.debug("Domain events map tx resource exists, unbind it.")
            TransactionSynchronizationManager.unbindResource(DOMAIN_EVENTS_RESOURCE_KEY)
        }
        TransactionSynchronizationManager.bindResource(DOMAIN_EVENTS_RESOURCE_KEY, newDomainEventsMap())
    }

    fun newDomainEventsMap(): MutableMap<AggregateKey, MutableList<DomainEvent>> = HashMap()

    data class AggregateKey(val aggType: String, val aggId:String)

    const val DOMAIN_EVENTS_RESOURCE_KEY = "__DOMAIN_EVENTS_MAP__"

    private val logger = LoggerFactory.getLogger(SpringTxResourceDomainEventDispatcher::class.java)
}


/**
 * 实现领域模块定义的 DomainEventPublisher 接口。 发布领域事件的时候，实际上是把领域事件交给 ThreadLocalDomainEventDispatcher 暂存起来，
 */
@Component
class SpringTxResourceDomainEventPublisher: DomainEventPublisher {

    override fun publishEvent(event: DomainEvent) {
        SpringTxResourceDomainEventDispatcher.addEvent(event)
    }
}

/**
 * Implementation of DomainEventIdGenerator using UUID v7
 */
@Component
class UuidEventIdGenerator : DomainEventIdGenerator {
    override fun nextEventId(): String {
        return UuidCreator.getTimeOrderedEpoch().toString()
    }
}

@Component
class RegisterTxResourceSyncAfterBegin: TransactionExecutionListener {
    override fun afterBegin(transaction: TransactionExecution, beginFailure: Throwable?) {
        if (beginFailure != null) {
            logger.debug("Tx failed to begin.")
            return
        }

        if (!TransactionSynchronizationManager.isSynchronizationActive()) {
            logger.warn("Tx sync is not active.")
            return
        }

        SpringTxResourceDomainEventDispatcher.initDomainEventsMap()

        TransactionSynchronizationManager.registerSynchronization(object : TransactionSynchronization {
            override fun beforeCommit(readOnly: Boolean) {
                if (readOnly) {
                    logger.debug("Tx is readOnly. I don't try to dispatch events now.")
                    return
                }

                if (!SpringTxResourceDomainEventDispatcher.existUnDispatchedEvents()) {
                    return
                }
                logger.warn("Unexpected  unDispatched events are found. I will dispatch them now for you. YOU SHOULD CALL DomainEventDispatcher.dispatchNow() after you save your aggregate. ")
                SpringTxResourceDomainEventDispatcher.dispatchAllNow()
            }

            override fun afterCompletion(status: Int) {
                logger.debug("Tx completed, clean the domain events. ")
                SpringTxResourceDomainEventDispatcher.cleanAll()
            }
        })

        logger.debug("Tx sync registered.")
    }

    companion object {
        private val logger = LoggerFactory.getLogger(RegisterTxResourceSyncAfterBegin::class.java)
    }
}
```