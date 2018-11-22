## Guide to the Java Phaser



> Java 7中新增了一个灵活的线程同步栅栏---**Phaser**，如果你需要在执行某些任务之前等待其他任务执行到某个状态，那么**Phaser**是一个很好的选择，这篇文章将介绍`java.util.concurrent.Phaser`，它和`CountDownLatch`具有相似的功能，但是**Phaser**更灵活。

### 基本术语

在理解Phaser类的运行机制之前，我们先来了解Phaser类中的一些基本术语：

- **Registration**

  开篇我们讲过，Phaser比`CountDownLatch`和`CyclicBarrier`更灵活，因为它可以通过`register()`和`bulkRegister(int parties)`来动态调整注册任务的数量，任务也可以通过执行`arriveAndDeregister()`来注销任务，Phaser允许的最大任务注册数量为`65535`。

- **Arrival**

  正如Phaser类的名字所暗示，每个Phaser实例都会维护一个phase number，初始值为0。每当所有注册的任务都到达Phaser时，phase number累加，并在超过Integer.MAX_VALUE后清零。`arrive()`和`arriveAndDeregister()`方法用于记录到 达，`arriveAndAwaitAdvance()`方法用于记录到达，并且等待其它未到达的任务。

- **Termination**

  Phaser支持终止。Phaser终止之后，***调用`register()`和`bulkRegister(int parties)`方法没有任何效果***，`arriveAndAwaitAdvance()`方法也会立即返回。触发终止的时机是在`protected boolean onAdvance(int phase, int registeredParties)`方法返回时，如果该方法返回true，那么Phaser会被终止，默认实现是在注册任务数为0时返回true`（即 return registeredParties == 0;）`，我们也可以通过重写这个方法来自定义的终止逻辑。此外，`forceTermination()`方法用于强制终止，`isTerminated()`方法用于判断是否已经终止。

- **Tiering**

  Phaser支持层次结构，即通过构造函数`Phaser(Phaser parent)`和`Phaser(Phaser parent, int parties)`构造一个树形结构。这有助于减轻因在单个的Phaser上注册过多的任务而导致的竞争，从而提升吞吐量，代价是增加单个操作的开销。

### 核心API

- Phaser(int parties)，构造方法，与CountDownLatch一样，传入同步的线程数，也支持层次构造Phaser(Phaser parent)。
- register()，bulkRegister(int Parties)，动态添加一个或多个参与者。
- arriveAndDeregister()方法，动态撤销线程在phaser的注册，通知phaser对象，该线程已经结束该阶段且不参与后面阶段。
- isTerminated()，当phaser没有参与同步的线程时（或者onAdvance返回true），phaser是终止态（如果phaser进入终止态arriveAndAwaitAdvance()和awaitAdvance()都会立即返回，不在等待）isTerminated返回true。
- arrive()方法，通知phaser该线程已经完成该阶段，但不等待其他线程。
- arriveAndAwaitAdvance()方法，类似await()方法，记录到达线程数，阻塞等待其他线程到达同步点后再继续执行。
- awaitAdvance(int phase) /awaitAdvanceInterruptibly(int phase) 传入阶段数，只有当前阶段等于phase阶段时才阻塞等待。后者如果线程在休眠被中断会抛出InterruptedException异常（phaser的其他方法对中断都不会抛出异常）。
- onAdvance(int phase, int registeredParties)方法。参数phase是阶段数，每经过一个阶段该数加1，registeredParties是当注册的线程数。此方法有2个作用：1、当每一个阶段执行完毕，此方法会被自动调用，因此，重载此方法写入的代码会在每个阶段执行完毕时执行，相当于CyclicBarrier的barrierAction。2、当此方法返回true时，意味着Phaser被终止，因此可以巧妙的设置此方法的返回值来终止所有线程。例如：若此方法返回值为 phase >= 3，其含义为当整个线程执行了3个阶段后，程序终止。
- forceTermination()方法，强制phaser进入终止态。

### 简单示例

```java
/**
 * @author Shuai Junlan[shuaijunlan@gmail.com].
 * @since Created in 2:26 PM 11/7/18.
 */
public class PhaserTest {
    public static void main(String[] args) 
            throws InterruptedException {
        final Phaser phaser = new Phaser(1);
        for (int index = 1; index <= 10; index++){
            phaser.register();
            new Thread(
                    new Player(phaser), 
                    "Player" + index).start();
        }
        System.out.println("Game start");
        phaser.arriveAndDeregister();

        while (!phaser.isTerminated()){
            Thread.sleep(100);
        }
        System.out.println("Game over");
    }
}
class Player implements Runnable{
    private final Phaser phaser;
    Player(Phaser phaser){
        this.phaser = phaser;
    }
    @Override
    public void run() {
        //First step, waiting for all threads be created
        phaser.arriveAndAwaitAdvance();

        try {
            //Second step, waiting for all players be ready
            Thread.sleep(
                    new Random().nextInt(100) * 10L);
            System.out.println(
                    Thread.currentThread().getName() + " ready");
            phaser.arriveAndAwaitAdvance();

            /////////////////running////////////////////

            //Third step, waiting for all players arrived, then competition finishing.
            System.out.println(
                    Thread.currentThread().getName() + " arrived");
            phaser.arriveAndDeregister();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

**输出结果：**

```
Game start
Player10 ready
Player4 ready
Player9 ready
Player1 ready
Player7 ready
Player6 ready
Player8 ready
Player5 ready
Player2 ready
Player3 ready
Player3 arrived
Player5 arrived
Player6 arrived
Player1 arrived
Player4 arrived
Player7 arrived
Player8 arrived
Player9 arrived
Player10 arrived
Player2 arrived
Game over
```

