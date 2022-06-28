# 时间轮

```java
package com.fool.demo.utils;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

/**
 * @author fool
 * @date 2022/2/11 9:43
 */
public class TimerWheel {

    private final TimeUnit timeUnit;

    private final WheelUnit[] wheel;

    private final ExecutorService executorService;

    private static class WheelUnit {


        private final List<UnitContent> unitContents;

        public WheelUnit() {
            unitContents = new ArrayList<>();
        }

        public void addContent(UnitContent unitContent) {
            unitContents.add(unitContent);
        }

        public void execute(ExecutorService executorService) {
            getCurrentRoundUnitContent().forEach(e -> executorService.execute(e::execute));
        }

        public void decrementRoundNumber() {
            unitContents.forEach(UnitContent::decrementRoundNumber);
        }

        public List<UnitContent> getCurrentRoundUnitContent() {
            return unitContents.stream().filter(e -> e.getRoundNumber() == 0).collect(Collectors.toList());
        }
    }

    public interface Executor {
        void execute();
    }


    private static class UnitContent {
        private int roundNumber;

        private final Executor executor;

        public UnitContent(int roundNumber, Executor executor) {
            this.roundNumber = roundNumber;
            this.executor = executor;
        }

        public void execute() {
            executor.execute();
        }

        public void decrementRoundNumber() {
            this.roundNumber--;
        }

        public int getRoundNumber() {
            return this.roundNumber;
        }
    }

    public TimerWheel(int size, TimeUnit timeUnit) {
        this.timeUnit = timeUnit;
        this.wheel = new WheelUnit[size];
        this.executorService = new ThreadPoolExecutor(5, 10, 60L, TimeUnit.SECONDS, new ArrayBlockingQueue<>(10));
        for (int i = 0; i < size; i++) {
            wheel[i] = new WheelUnit();
        }
    }

    public void add(int timeDelay, Executor executor) {
        int roundNum = timeDelay / wheel.length;
        int index = timeDelay % wheel.length;
        UnitContent tUnitContent = new UnitContent(roundNum, executor);
        wheel[index].addContent(tUnitContent);
    }

    public void start(int offset) {
        int index = offset;
        while (true) {
            wheel[index].execute(executorService);
            index = (index + 1) % wheel.length;
            if (index == 0) {
                decrementRoundNumber();
            }
            try {
                timeUnit.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
                break;
            }
        }

    }

    public void decrementRoundNumber() {
        for (WheelUnit wheelUnit : wheel) {
            wheelUnit.decrementRoundNumber();
        }
    }


    public static void main(String[] args) {

        TimerWheel timerWheel = new TimerWheel(12, TimeUnit.SECONDS);

        int[] delays = {1, 10, 12, 15, 24, 30, 50};
        for (int delay : delays) {
            timerWheel.add(delay, () -> System.out.println("delay:" + delay + "\t success"));
        }

        new Thread(() -> {
            int i = 0;
            while (true) {
                System.out.println("Second:" + (i++));
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        timerWheel.start(0);
    }

}

```

参考

[时间轮在Kafka中的运用](https://mp.weixin.qq.com/s/nYokg1QnVrHjNNT36k5tig)
