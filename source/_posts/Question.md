title: Question
date: 2014-11-24 15:31:22
categories: JAVA
tags: Concurrency
---

最近在准备Tango的面试，猎头给的参考资料中有这么一道题目：

{% blockquote %}
四个线程，一线程输出1，二线程输出2，三线程输出3，四线程输出4。分别向四个文件输出，使文件的内容如下：
* file1：123412341234…………
* file2：234123412341…………
* file3：341234123412…………
* file4：412341234123…………
{% endblockquote %}

后来查了一下，发现这又是一道早些年Google的经典面试题。

为了解决这道题目，也弥补一下自己稀缺的并发知识，于是在一周内啃完了Java Concurrency in Practice这本书。后来在面试中，和面试官聊到并发的话题时，随便扯扯书中说的CAS，面试官就不问什么了，呵呵。

好了，言归正传，下面就来给出我对这道题目的三个解法：

# 利用Object内部的instrinsic lock

```
import java.io.FileNotFoundException;
import java.io.PrintWriter;

class WriteThread extends Thread {
	private int index;
	private int num;
	private PrintWriter[] writers;
	private int[] lastNums;

	public WriteThread(int index, int num, PrintWriter[] writers, int[] lastNums) {
		this.index = index;
		this.num = num;
		this.writers = writers;
		this.lastNums = lastNums;
	}

	public void run() {
		int ln = (num - 2 + 4) % 4 + 1;
		for (int i = 0; i < 10; i++) {
			synchronized (writers[index]) {
				while (!(lastNums[index] == -1 && index == num - 1) && lastNums[index] != ln)
					try {
						writers[index].wait();
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				writers[index].write("" + num);
				lastNums[index] = num;
				writers[index].notifyAll();
			}
			index = (index - 1 + 4) % 4;
		}
	}
}

public class FourThread {
	public static void main(String[] args) throws FileNotFoundException, InterruptedException {
		PrintWriter[] writers = new PrintWriter[4];
		int[] lastNums = new int[4];
		for (int i = 0; i < 4; i++) {
			writers[i] = new PrintWriter("D:/" + (i + 1) + ".txt");
			lastNums[i] = -1;
		}
		WriteThread[] ts = new WriteThread[4];
		for (int i = 0; i < 4; i++) {
			ts[i] = new WriteThread(i, i + 1, writers, lastNums);
			ts[i].start();
		}
		for (int i = 0; i < 4; i++) {
			ts[i].join();
			writers[i].close();
		}
	}
}

```

# 利用Reentrant Lock和Condition
```
import java.io.FileNotFoundException;
import java.io.PrintWriter;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class WriteThreadv3 extends Thread {
	private int index;
	private int num;
	private PrintWriter[] writers;
	private int[] lastNums;
	private Lock lock;
	private Condition[][] cond;

	public WriteThreadv3(int index, int num, PrintWriter[] writers, int[] lastNums, Lock lock, Condition[][] cond) {
		this.index = index;
		this.num = num;
		this.writers = writers;
		this.lastNums = lastNums;
		this.lock = lock;
		this.cond = cond;
	}

	public void run() {
		int ln = (num - 2 + 4) % 4 + 1;
		for (int i = 0; i < 10; i++) {
			lock.lock();
			try {
				while (!(lastNums[index] == -1 && index == num - 1) && lastNums[index] != ln)
					cond[index][ln-1].await();
				writers[index].write("" + num);
				lastNums[index] = num;
				cond[index][num-1].signal();
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				lock.unlock();
			}
			index = (index - 1 + 4) % 4;
		}
	}
}

public class FourThreadv2 {
	public static void main(String[] args) throws FileNotFoundException, InterruptedException {
		PrintWriter[] writers = new PrintWriter[4];
		int[] lastNums = new int[4];
		for (int i = 0; i < 4; i++) {
			writers[i] = new PrintWriter("D:/" + (i + 1) + ".txt");
			lastNums[i] = -1;
		}
		WriteThreadv3[] ts = new WriteThreadv3[4];
		Lock lock = new ReentrantLock();
		Condition[][] cond = new Condition[4][];
		for(int i = 0 ; i < 4; i++) {
			cond[i] = new Condition[4];
			for(int j = 0 ; j < 4; j++)
				cond[i][j] = lock.newCondition();
		}
		
		for (int i = 0; i < 4; i++) {
			ts[i] = new WriteThreadv3(i, i + 1, writers, lastNums, lock, cond);
			ts[i].start();
		}
		for (int i = 0; i < 4; i++) {
			ts[i].join();
			writers[i].close();
		}
	}
}

```

# 利用CyclicBarrier
```
import java.io.FileNotFoundException;
import java.io.PrintWriter;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

class WriteThreadv2 extends Thread {
	private int index;
	private int num;
	private PrintWriter[] writers;
	private int[] lastNums;
	private CyclicBarrier barrier;
	
	public WriteThreadv2(int index, int num, PrintWriter[] writers, int[] lastNums, CyclicBarrier barrier) {
		this.index = index;
		this.num = num;
		this.writers = writers;
		this.lastNums = lastNums;
		this.barrier = barrier;
	}

	public void run() {
		for (int i = 0; i < 10; i++) {
			synchronized (writers[index]) {
				try {
					barrier.await();
				} catch (InterruptedException | BrokenBarrierException e) {
					e.printStackTrace();
				}
				writers[index].write("" + num);
				lastNums[index] = num;
			}
			index = (index - 1 + 4) % 4;
		}
	}
}

public class FourThreadv3 {
	public static void main(String[] args) throws FileNotFoundException, InterruptedException {
		PrintWriter[] writers = new PrintWriter[4];
		int[] lastNums = new int[4];
		for (int i = 0; i < 4; i++) {
			writers[i] = new PrintWriter("D:/" + (i + 1) + ".txt");
			lastNums[i] = -1;
		}
		WriteThreadv2[] ts = new WriteThreadv2[4];
		CyclicBarrier barrier = new CyclicBarrier(4);
		for (int i = 0; i < 4; i++) {
			ts[i] = new WriteThreadv2(i, i + 1, writers, lastNums, barrier);
			ts[i].start();
		}
		for (int i = 0; i < 4; i++) {
			ts[i].join();
			writers[i].close();
		}
	}
}

```