---
layout: default
title: Multiprogramação
parent: Comunicação
nav_order: 2
---

# Multiprogramação e *Multithreading* em Sistemas Distribuídos

É impossível pensar em sistemas distribuídos sem pensar em concorrência na forma de múltiplos processos executando, normalmente, em hosts distintos.
De fato, os exemplos que apresentamos até agora consistem todos em um processo cliente requisitando ações de algum processo servidor.
Apesar disso, a interação entre tais processos aconteceu sempre de forma sincronizada, *lock-step*, em que o cliente requisitava o serviço e ficava bloqueado esperando a resposta do servidor, para então prosseguir em seu processamento. O servidor, de sua parte, fica bloqueado esperando requisições, que atende e então volta a dormir.
Assim, apesar do uso de processadores distintos e da concorrência na execução dos processos, temos um baixo grau de efetivo paralelismo.

![Request/Response](./images/02-03.png)

Para usarmos melhor os recursos disponíveis, uma das razões de ser da computação distribuída, temos então que pensar em termos eventos sendo disparados entre os componentes, que devem ser tratados assim que recebidos ou tão logo quanto haja recursos para fazê-lo. 
Estes eventos correspondem tanto a requisições quanto a respostas (efetivamente tornando difícil a distinção).
Além disso, sempre que possível, um componente não deve ficar exclusivamente esperando por eventos, aproveitando a chance executar outras tarefas até que eventos sejam recebidos.
Dada que processos interagem com a rede usando sockets, cuja operação de leitura é bloqueante, para aumentar a concorrência em um processo, precisamos falar de multi-threading.

Há duas razões claras para estudarmos multi-threading. A primeira, é a discutida acima: permitir o desenvolvimento de componentes que utilizem "melhormente" os recursos em um host.
A segunda é o fato que muitos dos problemas que aparecem em programação multi-thread, aparecem em programação multi-processo (como nos sistemas distribuídos), apenas em um grau de complexidade maior.

## Threads x Processos

| Processo | Thread |
|----------|--------|
| Instância de um programa | "Processo leve"|
| Estado do processo | Estado do thread | 
| Função main | "qualquer" função|
| Memória privada ao processo| Compartilha estado do processo que os contém|
| Código, Stack, Heap, descritores (e.g, file descriptors), controle de acesso | Stack, variáveis locais |
| IPC - Inter process communication  | IPC -- Inter process communication|
| Sistema operacional | Diferentes implementações |
| | Posix, C++, Java, ...|


## Threads em SD

### Cliente multithreaded

Vantagens similares a usar em sistemas centralizado.

* Lida com várias tarefas concorrentemente.
* Esconde latência.
* Separa código em blocos/módulos.


---

### Servidor multithreaded

Há diversas possibilidades de uso de threads em servidores. A mais simples é usar apenas um, com temos feito até agora.

![Single Threaded](./images/singlethreadedserver.gif)

---

Outra opção é criar um novo tthread para cada nova requisição e atende múltiplas requisições concorrentemente.

![Multi Threaded](./images/multithreadedserver.gif)

* Permite tratar requisições enquanto faz IO.
* Número de threads é limitado. %Pelo framework em uso.
* Número de threads deve ser limitado. %Para não saturar o servidor.
* Criação e destruição de threads é cara.

Assim, temos uma outra opção que também usa múltiplos threads, que usa *pools*  de threads para lidar com as requições.

[![Pool Threaded](./images/poolthreadedserver.gif)](https://www3.nd.edu/~dthain/courses/cse30341/spring2009/project4/project4.html)


E se quebrarmos o processamento em vários pools, no que é conhecido como SEDA - **Staged Event-Driven Architecture**.

[![Seda](images/seda1.png)](http://images.cnitblog.com/blog/13665/201306/15180500-a54c8eb3d73246469f1b74ee74f2119b.png)

Assim, cada estágio pode ter seu próprio *pool*, que se escala individualmente.

[![Seda](images/seda2.png)](http://images.cnitblog.com/blog/13665/201306/15180500-a54c8eb3d73246469f1b74ee74f2119b.png)



Que lembra arquitetura de [micro-serviços](http://muratbuffalo.blogspot.com.br/2011/02/seda-architecture-for-well-conditioned.html)

Se você quiser ler mais sobre SEDA, vá [aqui](http://courses.cs.vt.edu/cs5204/fall05-gback/presentations/SEDA_Presentation_Final.pdf).


### Problemas com multithreading

Idealmente, os threads que compartilham variáveis seriam colocados nos mesmos processadores.
E se não houvesse compartilhamento e dado um número adequado de processadores, teríamos paralelismo perfeito.

![Multithreaded](images/multithread2.png)

A realidade, contudo, é outra e simplesmente criar múltiplos threads não garante paralelismo perfeito, pois o SO é quem é responsável por escalonar os mesmos.

![Multithreaded](./images/multithreaded.jpg)

Memes bonitinhos à parte, precisamos enfrentar condições de corrida de forma a não levar **inconsistências**.

![Multithreaded](images/multithread3.png)

Para isso, usamos as primitivas de controle de concorrência que estudaram em SO, que também tem seus problemas em potencial, como **deadlocks** e **inanição**.
Veja o seguinte vídeo para uma análise detalhada do cenário anterior e outros pontos importantes.

<iframe width="560" height="315" src="https://www.youtube.com/embed/JRaDkV0itbM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


### Estado em Servidores


#### Stateless Servers

Não mantém informação após terminar de tratar requisições.

* Fecha todos os arquivos abertos
* Não faz promessas de atualização ao cliente
* Clientes e servidores são independentes
	
* Pouca ou nenhuma inconsistência causada por falhas
* Perda de desempenho (e.g., abertura do mesmo arquivo a cada requisição.)


#### Stateful Servers

Mantém informação dos clientes entre requisições.

* Mantem arquivos abertos
* Sabe quais dados o cliente tem em cache
	
* Possíveis inconsistência causada por falhas (cliente se conecta a servidor diferente)
* Melhor desempenho
	
* Maior consumo de recursos


#### Impacto na Concorrência

| Stateless | Stateful |
|-----------|----------|
| Resultado depende da entrada| Depende do histórico de entradas |
| Qualquer servidor pode atender | Mesmo servidor deve atender |




## Multithread na prática

### PThreads

#### Função de entrada

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
	
int thread_count;

void* hello(void* rank) {
	long my_rank = (long) rank;
	printf("Hello from thread %ld of %d\n", my_rank, thread_count);
	return NULL;
}
```

#### Criação

```c
int main(int argc, char* argv[]) {
	long thread;
	pthread_t* thread_handles;
	
	if(argc < 2) {
		printf("usage: %s <number of threads>", argv[0]); 
		return 1;
	}
	
	thread_count = strtol(argv[1], NULL, 10);
	thread_handles = malloc(thread_count*sizeof(pthread_t));
	
	for (thread = 0; thread < thread_count; thread++)
		pthread_create(&thread_handles[thread], NULL, hello, (void*) thread);
	
	printf("Hello from the main thread\n");
```	

#### Destruição

```c
	for (thread = 0; thread < thread_count; thread++)
		pthread_join(thread_handles[thread], NULL);
	
	free(thread_handles);
```

#### Execução

Compile com
```bash
gcc -pthread teste.c -o teste
````

e execute com 
```bash
./teste 5
```

Observe que a saída das threads é *ordenada*. Agora experimente
```bash
./teste 200
```

Isto acontece porquê a execução das threads independe da ordem de criação. De fato, usando PThreads, temos pouco controle sobre os threads que criamos. Mas isto não quer dizer que estamos "órfãos" de API.

* `pthread_create` - cria novo thread
   * passagem de parâmetros
   * opções
* `pthread_join` - espera thread terminar
   * recebe resultado da thread
* `pthread_tryjoin` - espera thread terminar
* `pthread_exit` - termina a thread e retorna resultado 
   > An implicit call to \lstinline|pthread_exit()| is made when a thread other than the thread in which \lstinline|main()| was first invoked returns from the start routine that was used to create it. The function's return value serves as the thread's exit status. (manual do \lstinline|pthread_exit|)}
	
* pthread_attr_setaffinity_np* - ajusta afinidade dos threads.

#### Threads Java

Neste tutorial, baseado neste [outro](https://docs.oracle.com/javase/tutorial/essential/concurrency/), exploraremos formas de se obter concorrência em Java. Isto é, exploraremos como iniciar múltiplas linhas de execução de instruções, que podem ou não, ser executadas em paralelo.

Em Java, há essencialmente duas formas de se conseguir concorrência. A primeira é via instâncias explícitas da classe `Thread`, e a segunda é via abstrações de mais alto nível, os `Executors`.

---
* Thread
* Executor
---

Além de formas de definir as linhas de execução, Java provê diversas estruturas para comunicação e coordenação destas linhas, desde de a versão 5 da linguagem, no pacote `java.util.concurrent`.


### *Threads*
Há duas formas básicas de se usar a classe `Thread`: extensão ou delegação de um objeto implementando `Runnable`.


---
##### Estender Thread
```Java
public class HelloThread extends Thread {
    public void run() {
        System.out.println("Hello from a thread!");
    }

    public static void main(String args[]) {
        Thread t = new HelloThread();
        t.start();
    }
}
```
---


---
##### Implementar Runnable
```Java
public class HelloRunnable implements Runnable {
    public void run() {
        System.out.println("Hello from a thread!");
    }

    public static void main(String args[]) {
        Thread t = new Thread(new HelloRunnable());
        t.start();
    }
}
```
---

Observe que nos dois exemplos, um método `run()` é implementado com o código a ser executado pelo *thread*. Em nenhum dos exemplos, contudo, o método é invocado diretamente. 
Em vez disto, o método `start()`, sim, é invocado. Isto ocorre pq antes de executar as instruções definidas pelo pelo programador no método `run()`,
a máquina virtual precisa executar alguma "mágica" por baixo dos panos como, por exemplo, solicitar ao sistema operacional a criação de um *thread* do SO, que servirá de hospedeiro para o *thread* Java. 
Isto acontece dentro do `start()`, que em algum ponto de sua execução levará a invocação do método `run()`.

A classe `Thread` também provê uma série de métodos que permitem gerenciar a vida do *thread* criado. 
Por exemplo, o método de classe (`static`) `Thread.sleep()` permite bloquear um *thread*  por um determinado período.


---
##### Thread.sleep()
```Java
public class HelloRunnable implements Runnable {
    public void run() {
        for (int i = 0; i < 10; i ++)
        {
            System.out.println("Hello at instant " + i);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException ie) {
                System.out.println("awoken");
            }
        }
    }

    public static void main(String args[]) {
        Thread t = new Thread(new HelloRunnable());
        t.start();
    }
}
```
---

Observe que a chamada a `sleep()` está dentro de um bloco `try/catch`. Isto é necessário pois é permitido à JVM acordar o *thread* em qualquer instante, antes ou após o tempo especificado. Assim, embora normalmente o tempo "dormido" seja próximo ao especificado, se há requisitos de precisão, é necessário que o *thread*, ao acordar, verifique se já dormiu o suficiente.

---
##### InterruptedException
```Java
public class HelloRunnable implements Runnable {
    public void run() {
        for (int i = 0; i < 10; i ++)
        {
            System.out.println("Hello at instant " + i);
            long before = System.currentTimeMillis();
            long timeout = 1000;
            while(before + timeout > System.currentTimeMillis())
            {
                try {
                    Thread.sleep(Math.max(0,System.currentTimeMillis() - (before + timeout)));
                } catch (InterruptedException ie) {
                    System.out.println("awoken");
                }
            }
        }
    }

    public static void main(String args[]) {
        Thread t = new Thread(new HelloRunnable());
        t.start();
    }
}
```
---

Quando um *thread* está sendo executado, outros podem ter que esperar até que complete. Por exemplo, no caso de um navegador
Web, o *thread* que faz a renderização da página não pode começar a trabalhar enquanto o *thread*  que solicitou o HTML
do servidor não receber sua resposta. Um *thread* indica a intenção de esperar por outro usando o método `join()`.



---
##### Thread.join()
```Java
public class HelloRunnable implements Runnable {
    public void run() {
        Random rand = new Random();
        for (int i = 0; i < 10; i ++)
        {
            System.out.println("Hello at instant " + i);
            long before = System.currentTimeMillis();
            long timeout = 901 + rand.nextInt(200);
            while(before + timeout > System.currentTimeMillis())
            {
                try {
                    Thread.sleep(Math.max(0,System.currentTimeMillis() - (before + timeout)));
                } catch (InterruptedException ie) {
                    System.out.println("awoken");
                }
            }
        }
    }

    public static void main(String args[]) {
        Thread t = new Thread(new HelloRunnable());
        //t.setDaemon(true);
        t.start();
        try {
            t.join();
            //t.join(10000);
        } catch (InterruptedException ie) {
            System.out.println("Waiting was interrupted");
        }
        if (t.isAlive())
            System.out.println("Got tired of waiting");
        else
            System.out.println("Wait is over");
    }
}
```
---

Invocar `t.join()` fará com que o *thread* principal espere indefinidamente até que `t` termine de executar.
Caso seja necessário limitar o tempo de espera, o tempo pode ser especificado como na linha comentada. 
Caso a espera termine por causa de um *timeout*, é possível testar o estado atual do thread com `Thread.isAlive()`.

Outro método interessante, `Thread.setDaemon()`, especifica que o *thread* pode ser terminado quando a *thread* principal terminar. Descomente a invocação e teste o efeito.

##### Exercício

Vejamos um exemplo simples do uso de *threads*.

* Instancie um programa que gere 10 *threads*. 
* Todos os *threads* devem compartilhar **uma mesma instância** de `Counter`
* Cada *thread* deve executar um *loop* em que incrementa o valor do contador 20 vezes 
  * a cada vez, imprime o resultado precedido do identificador do *thread* (use `Thread.getName()` ou `Thread.currentThread().getName()`)
* A *thread*  principal deve esperar todas as outras terminarem antes de terminar (use `Thread.join()`).
* Analise a saída do programa observando a ordem de execução dos *threads*.

---
##### Counter.java
```Java
class Counter {
    private int c = 0;

    public int increment() {
        return ++c;
    }

    public int decrement() {
        return --c;
    }

    public int value() {
        return c;
    }
}
```


É fácil observar que a saída do programa é aleatória nos identificadores e tende a ser incremental nos contadores, mas nem sempre isso é verdade. 
Como discutido anteriormente, frequentemente *threads* tem que coordenar suas ações para que não pisem uns nos outros, por exemplo decidindo quem deve ser o próximo a entrar em uma região crítica ou será o responsável por uma tarefa. 
Em Java, esta coordenação pode ser feita por diversas abstrações: `synchronized`, `Lock`, variáveis atômicas, ...


#### `synchronized`
Ao definir métodos como `synchronized`, garante-se que os mesmos nunca serão executados concorrentemente. 
Observe a classe a seguir, que modifica o contador do exercício anterior.

---
##### synchronized
```Java
public class SynchronizedCounter {
    private int c = 0;

    public synchronized int increment() {
        return ++c;
    }

    public synchronized int decrement() {
        return --c;
    }

    public synchronized int value() {
        return c;
    }
}
```
---

Caso dois *threads* invoquem os métodos `increment` e `decrement` ao mesmo tempo, por exemplo, a JVM fará com que um dos *threads* pare sua execução até que o outro tenha completado a invocação.
Isto não quer dizer que executar o exercício anterior com esta versão do contador não levará a saídas com incrementos completamente sequenciais, pois um *thread*  poderia parar de ser executado logo após incrementar o contador, depois de terminado o método `increment`, e só voltar a executar depois que outro tenha incrementado e impresso na tela o valor obtido. 
O que quer dizer é que, mesmo que saídas estranhas existam, cada operação foi executada integralmente antes da operação seguinte.

##### Exercício
Modifique o código do exercício anterior para usar a versão `synchronized` do contador. Depois de executá-lo, adicione um `println("Dentro: " + c)` **dentro** do método de incremento para verificar que estas saídas acontecem ordenadamente.

#### Blocos `synchronized`
`synchronized` funciona porquê limita a concorrência, e é problemático exatamente pela mesma razão. 
Por isso, é essencial que o `synchronized` seja o mais limitado possível em termos de escopo, o que nos leva ao uso de `synchronized` em blocos de código menores que métodos. Por exemplo:

---
##### blocos `synchronized`
```Java
public class Namer {
    String lastName = null;
    int nameCount = 0;

    public void addName(String name) {
       lastName = name;
        synchronized(this) {
           nameCount++;
        }
        nameList.add(name);
    }
}
```

Neste caso, blocos sincronizados **no mesmo objeto**, não são executados concorrentemente, mas outros blocos sim.

##### Exercício
Neste exercício, use dois objetos para travar o acesso a dois contadores. Instancie um programa com dois *threads*  tal que:
* executem um loop 1000 vezes em que
* o primeiro *thread* primeiro invoca `inc1` e depois `inc2`
* o segundo *thread* primeiro invoca `inc2` e depois `inc1`
* ambos os threads imprimem o valor de `c1` e `c2`

---
##### synchronized
```Java
public class MsLunch {
    private long c1 = 0;
    private long c2 = 0;
    private Object lock1 = new Object();
    private Object lock2 = new Object();

    public void inc1() {
        synchronized(lock1) {
            c1++;
        }
    }

    public void inc2() {
        synchronized(lock2) {
            c2++;
        }
    }
}
```

#### *Deadlock*
O uso dos "locks" em ordens diferentes pode levar a um deadlock, pois o seguinte grafo de dependência poderá ser gerado:

---
##### Deadlock
```plantuml
digraph Test {
T1 -> lock1
lock1 -> T2
T2 -> lock2
lock2 -> T1
}
```
---

#### Sinalização
Usados corretamente, o bloco `synchronized` é executado de forma atômica, isto é, indivisível.
Algumas operações muito simples são naturalmente atômicas, e não precisam ser "protegidas" pelo `synchronized`.
Por exemplo, leituras e escritas de tipos básicos como (`int`, `char`, `byte`, mas não `long` ou `double`), ou variáveis declaradas `volatile`.

Usando estas variáveis, é possível coordenar *threads*, por exemplo, assim:

---
##### Espera ocupada
```Java
boolean condicao = false;

...

public void espereCondicao() {
    while(!condicao) {}
    System.out.println("condicao alcancada.");
}

...

public void satisfacaCondicao() {
    condicao = true;
}
```
---


Embora correto, esta abordagem não é eficiente, pois o primeiro método desperdiça computação. 
Felizmente, em Java, todos os objetos implementam os métodos `wait` e `notify/notifyAll`, que podem ser usados para sincronizar eficientemente *threds*.


---
##### Wait/Notify
```Java
public class Sync{
   Object synch = new Object();
   boolean condicao = false;
   public void espereCondicao() {
      while(!condicao) {
         try {
            synch.wait();
         } catch (InterruptedException e) {}
      }
      System.out.println("Condicao alcancada");
   }
...
   public void satisfacaCondicao() {
      condicao = true;
      synch.notifyAll();
   }
}
```
---


#### *Locks*
Outras abstrações para coordenação de *threads* estão disponíveis no pacote `java.util.concurrent`. 
As mais simples delas são `java.util.concurrent.locks.Lock` e `java.util.concurrent.locks.ReentrantLock`. 
Veja um exemplo de uso, notando o idioma de uso dentro de block `try/catch`.

---
##### Lock
```Java
Lock l = new ReentrantLock();
  l.lock();
  try {
     // access the resource protected by this lock
  } finally {
     l.unlock();
  }
```
---


### Executor
Além de *threads*, Java disponibiliza `Executor` como abstração de mais alto nível para execução de tarefas concorrentes.

---
##### Executor
* `Executor`
* `ExecutorService`
* `ScheduledExecutorService`

```
Executor e = ...;
Runnable r = ...;
e.execute(r);
```
---

Executors normalmente implementam *thread pools*, que podem ser de diferentes tipos. 
O mais simples é o de tamanho fixo em que há um número inicial de *threads* criados e que, no caso de algum ser terminado, por exemplo por causa de uma exceção não tratada, cria substitutos para manter o número constante.

---
##### ThreadPool
`Executor e = java.util.concurrent.Executors.newFixedThreadPool();`

* `newCachedThreadPool()` - *expandable thread pool*
* `newSingleThreadExecutor()` - *single task at a time*
* e outras versões
* `ForkJoinPool`
---

---
##### Fork/Join
```
if (my portion of the work is small enough)
  do the work directly
else
  split my work into two pieces
  invoke the two pieces and wait for the results
```
---

### Estrutura para Coordenação de *Threads*
Finalmente, Java também disponibiliza estruturas de dados que podem ser acessadas concorrentemente por múltiplos *threads* 
sem risco de corrupção. 

---
##### Alguns tipos interessantes

* `BlockingQueue` - bloquei *threads*  se não houver elementos na filq.
* `ConcurrentMap/ConcurrentHashMap` - operações atômicas;
   * `if (!m.containsKey(k)) m.put(k,v);`
   * `vOld = m.putIfAbsent(k,v);`
---

---
##### Tipos Atômicos

```java
import java.util.concurrent.atomic.AtomicInteger;

class AtomicCounter {
    private AtomicInteger c = new AtomicInteger(0);

    public void increment() {
        c.incrementAndGet();
    }

    public void decrement() {
        c.decrementAndGet();
    }

    public int value() {
        return c.get();
    }
}
```
---


---
##### ThreadLocal
```java
private static ThreadLocal<Integer> myId = new ThreadLocal<Integer>() {
   public Integer initialValue() {
      return new Random().nexInt();
   } 
};

public static Integer getMyId() { 
    return myId.get();
}
```
---

Para aprender mais, muito mais sobre concorrência em Java, ótimas referências são:

* [Java Concurrency in Practice](http://jcip.net/)
* [The Well-Grounded Java Developer](https://www.manning.com/books/the-well-grounded-java-developer)
* [Concorrência em Java](http://docs.oracle.com/javase/tutorial/essential/concurrency/simple.html)
* [Futures e Promises](http://winterbe.com/posts/2015/04/07/java8-concurrency-tutorial-thread-executor-examples/)
* [Locks](http://winterbe.com/posts/2015/04/30/java8-concurrency-tutorial-synchronized-locks-examples/)
* [Tipos Atômicos](http://winterbe.com/posts/2015/05/22/java8-concurrency-tutorial-atomic-concurrent-map-examples/)

---

#### Threads em Python

```python
#!/usr/bin/python
import thread
import time

# Define a function for the thread
def print_time( threadName, delay):
   count = 0
   while count < 5:
      time.sleep(delay)
      count += 1
      print "%s: %s" % ( threadName, time.ctime(time.time()) )

# Create two threads as follows
try:
   thread.start_new_thread( print_time, ("Thread-1", 2, ) )
   thread.start_new_thread( print_time, ("Thread-2", 4, ) )
except:
   print "Error: unable to start thread"

while True:
   pass
```

Ou

```python
#!/usr/bin/python

import threading
import time

exitFlag = 0

class myThread (threading.Thread):
   def __init__(self, threadID, name, counter):
      threading.Thread.__init__(self)
      self.threadID = threadID
      self.name = name
      self.counter = counter
   def run(self):
      print "Starting " + self.name
      print_time(self.name, self.counter, 5)
      print "Exiting " + self.name

   def print_time(threadName, counter, delay):
      while counter:
         if exitFlag:
            threadName.exit()
         time.sleep(delay)
         print "%s: %s" % (threadName, time.ctime(time.time()))
         counter -= 1

# Create new threads
thread1 = myThread(1, "Thread-1", 1)
thread2 = myThread(2, "Thread-2", 2)

# Start new Threads
thread1.start()
thread2.start()

print "Exiting Main Thread"
```


[Threads em Python](https://www.tutorialspoint.com/python/python_multithreading.htm)


### Exercício - Anel Multithread

* Usando uma linguagem de alto-nível como C/C++/Java, escrever um programa que crie 30 threads e faça com que uma mensagem circule entre os mesmos. 
* A mensagem é uma string aleatória de pelo menos 80 caracteres. 
* A cada vez que um thread recebe a mensagem ele a imprime, modifica o primeiro caractere minúsculo para maiúsculo, caso exista, dorme por 1 segundo, e repassa a mensagem. 
* Quando todos os caracteres forem maiúsculos, o processo repassa a mensagem e então termina. 
* Antes de terminar, o processo deve imprimir a mensagem resultante.

