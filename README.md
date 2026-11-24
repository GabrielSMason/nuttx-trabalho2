# nuttx-trabalho2

Trabalho 2 de DEC7562 (Sistemas Operacionais Embarcados – UFSC): detecção de
deadlock em mutex no kernel do Apache NuttX, testado na ESP32-DevKitC (build FLAT).

## O que foi feito

Quando uma tarefa vai bloquear num mutex que está ocupado, o kernel segue a
cadeia de espera: o dono desse mutex, o mutex que esse dono está esperando, o
dono desse outro, e assim por diante. Se a cadeia volta para a própria tarefa,
bloquear fecharia um ciclo e as tarefas ficariam presas para sempre. Nesse caso
o lock retorna `EDEADLK` em vez de bloquear.

Também foram criadas duas chamadas de sistema:

- `int sched_deadlock_enable(int enable)` – liga (1) ou desliga (0) a detecção.
  Retorna o estado anterior, ou -1 com `errno = EINVAL` se o valor não for 0 nem 1.
- `int sched_deadlock_info(struct deadlock_stats_s *info)` – copia para a
  aplicação o número de deadlocks evitados desde o boot e os dados do último
  ciclo (PIDs, mutex e tick). Retorna 0, ou -1 com `errno = EINVAL` se `info`
  for NULL.

O NuttX já tem `nxsched_collect_deadlock()` (`sched/misc/deadlock.c`), mas ela
só é usada no dump de assert, depois que o sistema já travou. Aqui a verificação
é feita na hora do lock e pode ser usada pelas aplicações.

## Arquivos modificados

Kernel (`nuttx`):

| Arquivo | Mudança |
|---|---|
| `sched/Kconfig` | opções `SCHED_DEADLOCK_DETECT` e `SCHED_DEADLOCK_MAXDEPTH` |
| `include/nuttx/sched.h` | `struct deadlock_stats_s` e protótipos das syscalls |
| `sched/semaphore/sem_wait.c` | `dl_next()`, `dl_check()`, verificação em `nxsem_wait_slow()` e as duas syscalls |
| `syscall/syscall.csv` | registro das syscalls |
| `include/sys/syscall_lookup.h` | números das syscalls |

Aplicação (`apps`): `examples/T2/`, comando `t2` no NSH.

Os arquivos modificados estão nas pastas `nuttx/` e `apps/` deste repositório, e
o diff completo do kernel está em `patches/nuttx.patch`.

## Como funciona

O lock de mutex passa por `nxsem_wait()` na libc. Se o mutex está livre, ele é
pego com uma operação atômica e nada muda. Se está ocupado, a chamada vai para
`nxsem_wait_slow()` no kernel, que é onde a tarefa iria dormir. A verificação
foi colocada nesse ponto, dentro da seção crítica, antes de a tarefa entrar na
fila de espera.

Estruturas e funções do NuttX usadas:

- `struct tcb_s`: `task_state` (para saber se a tarefa está em `TSTATE_WAIT_SEM`),
  `waitobj` (o semáforo em que ela está bloqueada) e `pid`.
- `sem_t` de mutex: `val.mholder` guarda o PID do dono, e o bit 31
  (`NXSEM_MBLOCKING_BIT`) indica que há alguém esperando; `waitlist` é a fila.
- `nxsched_get_tcb()`, `nxmutex_get_holder()`, `NXSEM_IS_MUTEX()`,
  `enter_critical_section()` / `leave_critical_section()`.

Como cada mutex tem um único dono e cada tarefa espera um único objeto, basta
seguir uma cadeia simples, sem busca em grafo. A cadeia é limitada por
`CONFIG_SCHED_DEADLOCK_MAXDEPTH` (padrão 8) para que o tempo com interrupções
desligadas seja previsível.

Quando o lock é recusado, a única coisa que já tinha sido alterada é o bit 31
do mutex; ele é limpo se não houver mais ninguém na fila.

## Como compilar e rodar

Base usada: apache/nuttx `6af833774a` e apache/nuttx-apps `c03829f56`.

```bash
mkdir -p ~/nuttxspace && cd ~/nuttxspace
git clone https://github.com/apache/nuttx.git
git clone https://github.com/apache/nuttx-apps.git apps
git -C nuttx checkout 6af833774a
git -C apps checkout c03829f56
git clone https://github.com/GabrielSMason/nuttx-trabalho2.git

git -C nuttx apply ../nuttx-trabalho2/patches/nuttx.patch
cp -r nuttx-trabalho2/apps/examples/T2 apps/examples/

cd nuttx
./tools/configure.sh -l esp32-devkitc:nsh
make menuconfig
```

No menuconfig, habilitar:

- Application Configuration → Examples → T2 - mutex deadlock detection test
- RTOS Features → Tasks and Scheduling → Detect mutex deadlocks at lock time

```bash
make -j$(nproc)
make flash ESPTOOL_PORT=/dev/ttyACM0
picocom -b 115200 /dev/ttyACM0
```

No NSH, rodar `t2`.

## Testes

A aplicação `t2` faz três testes:

1. **Deadlock ABBA**: a thread A pega M1 e depois quer M2; a thread B pega M2 e
   depois quer M1. Uma barreira garante que as duas já pegaram o primeiro mutex
   antes de pedir o segundo. O esperado é que exatamente uma receba `EDEADLK`
   e a outra consiga o mutex.
2. **`sched_deadlock_info`**: mostra o contador, os PIDs do ciclo, o mutex
   recusado e o tick, e confere se o mutex é um dos dois do teste.
3. **Parâmetros inválidos**: `sched_deadlock_info(NULL)` e
   `sched_deadlock_enable(5)` devem retornar -1 com `errno = EINVAL`.

Saída na placa:

```
nsh> t2
[1] Deadlock ABBA (EDEADLK = 35)
  A: lock(M2) = 35
  B: lock(M1) = 0
  OK: uma thread recebeu EDEADLK
[2] sched_deadlock_info
  count=1 len=2 ciclo: 4 5
  mutex=0x3ffb254c (M1=0x3ffb2564 M2=0x3ffb254c) ticks=256
  OK: mutex recusado e um dos dois do teste
[3] Parametros invalidos
  OK: info(NULL) -> EINVAL
  OK: enable(5) -> EINVAL
```

Qual das threads recebe o erro depende da ordem em que o escalonador as executa
depois da barreira: quem fecha o ciclo é quem recebe `EDEADLK`. Rodando `t2` de
novo, o contador aumenta (`count=2`), porque as estatísticas ficam no kernel.

O endereço guardado em `mutex` é o do `mutex_t` que fica dentro do
`pthread_mutex_t` (campo `mutex`), e não o do `pthread_mutex_t` em si, porque o
kernel só enxerga o mutex interno. Por isso o teste compara com `&g_m1.mutex` e
`&g_m2.mutex`.

## Limitações

- Só mutex é verificado. Semáforo contador, variável de condição etc. não têm
  dono, então não entram na cadeia.
- Ciclos com mais tarefas do que `SCHED_DEADLOCK_MAXDEPTH` não são detectados.
- A detecção é global (liga/desliga para o sistema todo) e só o último ciclo
  fica guardado.
- A aplicação precisa tratar o `EDEADLK`, normalmente soltando os mutexes que
  já tem e tentando de novo.
