+++
weight = 1
+++

# Overview

Edge-Cloud systems are _layered_ and _heterogeneous_.  
This infrastructures provide opportunities but also challenges.

{{% multicol %}} {{% col %}}

**Cloud**: high scalability and power but high latency in comms  
**Fog**: hierarchical architecture near the edge devices  
**Edge**: low latency in comms but limited computational power  
{{% /col %}}

{{% col %}}

{{< figure src="figures/cloud-fog-edge.svg" width="90%" >}}

{{% /col %}} {{% /multicol %}}

Several approaches have been proposed in literature to manage this complexity.

---

# Pulverization approach

Pulverization breaks down the system into smaller **computational units** that are continuously executed across the available hosts.

{{% multicol %}}
{{% col %}}
{{< figure src="figures/pulverization-model.svg" width="86%" caption="A <b>logical device</b> and one of its neighbours" >}}
{{% /col %}}

{{% col %}}
{{< figure src="figures/peer-to-peer.svg" width="41%" caption="Example of <b>peer-to-peer</b> deployment." >}}
{{% /col %}}

{{% col %}}
{{< figure src="figures/cloud.svg" width="40%" caption="Example of <b>edge-cloud</b> deployment." >}}
{{% /col %}}
{{% /multicol %}}

This approach promotes the **deployment independence** of the system by separating _functional_ aspects from _deployment_ aspects.

---

{{% section %}}

# PulvReAKt

{{% multicol %}}
{{% col %}}
Main features:

- **Simple API**: easy to use
- **Extensibility**: custom user-defined components
- **Flexible**: use different deployments strategies
- **Multi-platform**: multiple architectures
{{% /col %}}

{{% col %}}
LOGO
{{% /col %}}
{{% /multicol %}}

The framework is written in _Kotlin Multiplatform_ supporting:  
JVM, Android, JS, iOS, Linux, macOS, and Windows.

---

## Architecture

{{< figure src="figures/pulverization-framework-overview.svg" >}}

---

## PulvReAKt DSLs

#### System DSL
```kotlin
object EmbeddedDevice : Capability
object HighCpu : Capability

val systemConfig = pulverizationSystem {
    device("wearable") {
        Behaviour and Communication deployableOn setOf(EmbeddedDevice, HighCpu)
        Sensors and Actuators deployableOn setOf(EmbeddedDevice)
    }
    device("other-device") {
        Behaviour and Communication deployableOn HighCpu
    }
}
```

#### Deployment DSL
```kotlin
object Smartphone : Host {
    override val hostname = "android"
    override val capabilities = setOf(EmbeddedDevice)
}

object Laptop : Host {
    override val hostname = "alice"
    override val capabilities = setOf(HighCpu)
}

val infrastructure = setOf(Smartphone, Laptop)

val runtimeConfig = pulverizationRuntime(systemConfig, "wearable", infrastructure) {
    WearableBehaviour() startsOn Smartphone
    WearableComm() startsOn Smartphone
    WearableSensors() startsOn Smartphone
    WearableActuators() startsOn Smartphone
}
```

---

## Reconfiguration

```kotlin
expect fun getBatteryStatus(): Flow<Double>

object OnLowBattery : ReconfigurationEvent<Double> {
    override val events = getBatteryStatus()
    override val predicate = { it < 20.0 }
}

val runtimeConfig = pulverizationRuntime(systemConfig, "wearable", infrastructure) {
    WearableBehaviour() startsOn Smartphone
    WearableComm() startsOn Smartphone
    WearableSensors() startsOn Smartphone
    WearableActuators() startsOn Smartphone

    reconfigurationRules {
        onDevice {
            OnLowBattery reconfigures { Behaviour movesTo Laptop }
        }
    }
}
```

---

## Components communication

{{< figure src="figures/componentref-communicator.svg" >}}

{{% multicol %}} {{% col %}}

**ComponentRef**  
The **reference** to a component abstracting from its physical place.

Some _communication optimizations_ can be performed by the framework.

{{% /col %}}

{{% col %}}

**Communicator**  
Represents the **communication** between components abstracting from the specific protocol.

In-Memory, RabbitMQ and MQTT communicators

{{% /col %}} {{% /multicol %}}

{{% /section %}}

---

{{% section %}}

# PulvReAKt in action

---

## Hot-warm-cold Game

One person looks around and chooses an **object** to be the mystery object, then gives the other players suggestions to find the object.

The game is realized in a modern version using a **smartphone** device.  
The smartphone is used to **sense** the signal strength of the **object** via Bluetooth.  
On the smartphone UI, the user can see the **distance** from the object and the distance of the teammates.

The _goal_ is to find the object.

---

## System definition

We define the structure of each device that took part into the system, by providing the system's capability.

The interface `Capability` represent the capability that a component requires to be executed.

```kotlin
object EmbeddedDevice : Capability
object HighCpu : Capability

val systemConfig = pulverizationSystem {
    device("wearable") {
        Behaviour and Communication deployableOn setOf(EmbeddedDevice, HighCpu)
        Sensors and Actuators deployableOn setOf(EmbeddedDevice)
    }
}
```

In this configuration the `Behaviour` and `Communication` components can be deployed either on an _embedded device_ and a _server_.

`Sensors` and `Actuators` can be deployed only on an _embedded device_.

---

## Behaviour implementation

```kotlin
class WearableBehaviour : Behaviour<Unit, DistanceFromSource, SignalStrengthValue, WearableDisplayInfo, Unit> {
    override val context: Context by inject()
    private val filter = Filter<Int>(WINDOW)

    override fun invoke(
        state: Unit,
        export: List<DistanceFromSource>,
        sensedValues: SignalStrengthValue,
    ): BehaviourOutput<Unit, DistanceFromSource, WearableDisplayInfo, Unit> {
        filter.register(sensedValues.rssi)
        val filteredRssi = filter.get().toInt()
        val distance = 10.0.pow((RSSI_ONE_METER - filteredRssi) / SIGNAL_PATH_LOSS)
        val neighbourDistance = export.associate { it.deviceId to it.distance }
        val displayInfo = WearableDisplayInfo(neighbourDistance, distance)
        return BehaviourOutput(Unit, DistanceFromSource(context.deviceID, distance), displayInfo, Unit)
    }
}
```

---

## Communication implementation

```kotlin
@Serializable
data class DistanceFromSource(val deviceId: String, val distance: Double)

class WearableComm : Communication<DistanceFromSource> {
    override val context: Context by inject()
    private val mqttClient = MqttClient("tcp://192.168.8.1:1883", MqttClient.generateClientId(), MemoryPersistence())
    private val commTopic = "communication"
    private val defaultQoS = 2

    override fun receive(): Flow<DistanceFromSource> = callbackFlow<DistanceFromSource> {
        val callback = object : MqttCallback { ... }
        mqttCliet.setCallback(callback)
    }.filterNot { it.deviceId == context.deviceID }

    override suspend fun send(payload: DistanceFromSource) {
        mqttClient.publish(commTopic, Json.encodeToString(payload).encodeToByteArray(), defaultQoS, false)
    }
}
```

---

## Sensors implementation

```kotlin
@Serializable
data class SignalStrengthValue(val rssi: Int)

class WearableSensor(private val context: AndroidContext) : Sensor<SignalStrengthValue> {
    private var rssi = 0

    override suspend initialize() {
        // Setup Android BT
        // Update the variable `rssi` via callback 
    }

    override suspend fun sense(): SignalStrengthValue = SignalStrengthValue(rssi)
}

class WearableSensorsContainer(private val aContext: AndroidContext) : SensorsContainer() {
    override val context: Context by inject()

    override suspend fun initialize() {
        this += WearableSensor(aContext).apply { initialize() }
    }
}
```

---

## Actuators implementation

```kotlin
class WearableActuator(private val display: DisplayViewModel) : Actuator<WearableDisplayInfo> {
    override suspend fun actuate(payload: WearableDisplayInfo) {
        Log.i("WearableActuator", "Actuate: $payload")
        display.update(payload)
    }
}

class WearableActuatorsContainer(private val display: DisplayViewModel) : ActuatorsContainer() {
    override val context: Context by inject()

    override suspend fun initialize() {
        this += WearableActuator(display)
    }
}
```

---

## Runtime definition

```kotlin
class LowBatteryEvent() : ReconfigurationEvent<Double>(), Initializable {
    override val events: Flow<Double> = generateSequence(FULL) { it - DISCHARGE }.asFlow().onEach { delay(DELAY) }
    override val predicate: (Double) -> Boolean = { it < THRESHOLD }
}

val lowBatteryEvent = LowBatteryEvent().apply { initialize() }

val runtimeConfig = pulverizationRuntime(config, "wearable", infrastructure) {
    WearableBehaviour() withLogic ::wearableBehaviourLogic startsOn Smartphone
    WearableComm() withLogic ::wearableCommLogic startsOn Smartphone
    WearableSensorsContainer(context) withLogic ::wearableSensorsLogic startsOn Smartphone
    WearableActuatorsContainer(display) withLogic ::wearableActuatorsLogic startsOn Smartphone

    reconfigurationRules {
        onDevice {
            lowBatteryEvent reconfigures { Behaviour movesTo Laptop }
        }
    }

    withCommunicator { MqttCommunicator(hostname = "192.168.8.1", port = 1883) }
    withReconfigurator { MqttReconfigurator(hostname = "192.168.8.1", port = 1883) }
    withRemotePlaceProvider { defaultMqttRemotePlace() }
}
```

---

## Deployment

After the generation of the runtime configuration, it will be used by the runtime to execute the _deployment unit_.

#### Android deployment unit

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    
    // Other initializations

    val platform = PulverizationRuntime(deviceId, "android", runtimeConfig)
    platform.start()
}
```

#### Laptop deployment unit
```kotlin
suspend fun main() {
    // Other initializations

    val platform = PulverizationRuntime(deviceId, "laptop", runtimeConfig)
    platform.start()
}
```

{{% /section %}}

---

# Demo

{{% fragment %}}
⚠️ DISCLAIMER ⚠️

To find the object is used the RSSI value of the Bluetooth signal.  
The RSSI value is not a reliable metric to measure the distance between two devices,  
fluctuations in the signal strength can occur due to the environment.

The demo is only a proof of concept.
{{% /fragment %}}

---

# PulvReAKt Roadmap

1. Stabilize the framework's API
2. Support _global_ reconfiguration rules
3. Support _openness_ (new host added at runtime)
4. Other newtwork protocols (ZeroMQ, socket, ...)

---

# References

