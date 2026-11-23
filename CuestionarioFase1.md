# Cuestionario de la Fase 1

## 1. Con la analogía del edificio, explique qué es un contenedor y en qué se diferencia de una máquina virtual.

R// Siguiendo la analogía del edificio, un contenedor es un apartamento dentro de un edificio que es mi computador. Las personas que viven en cada apartamento son los procesos, y cada quien vive en lo suyo sin saber que existen los demás inquilinos, porque las paredes (namespaces) los aíslan. Todos comparten los mismos servicios públicos (CPU, RAM) que reparte la administración, que es el kernel de Linux, y cada apartamento tiene su contador con un límite (cgroups). Una máquina virtual sería una casa con cimientos propios: trae su propio kernel completo, por eso pesa más y tarda más en arrancar, mientras que el contenedor comparte el kernel con los demás.

## 2. En la Práctica 2 el mismo proceso tiene dos PID. ¿Qué namespace lo causa? ¿Cuáles fueron sus dos números?

R// Lo causa el namespace PID. Es el mismo proceso de Python, pero cada lado lo ve con un número distinto: dentro del contenedor cree que es el proceso 1 (por eso la API responde `pid_ns=1`) y desde mi Fedora es otro. Mis dos números fueron 23300 en el host y 1 dentro del contenedor, y los saqué con `grep NSpid /proc/$PID/status`.

## 3. ¿Qué significan los dos números de su cpu.max? ¿Cuánta CPU tiene la API cada 100 ms?

R// Mi `cpu.max` dice `25000 100000`. El primer número es cuánto tiempo de CPU puede usar la API en cada ciclo (25 000 microsegundos) y el segundo es lo que dura el ciclo (100 000 microsegundos, o sea 100 ms). Entonces, cada 100 ms la API tiene 25 ms de CPU, que equivale a un cuarto de núcleo. Si se los gasta antes de terminar el ciclo, el kernel la deja esperando hasta el siguiente.

## 4. Según su calculos.py, ¿en qué llamada a /ram muere la API? ¿Por qué el exit code será 137?

R// En mi caso la API muere en la llamada número 4 a `/ram`. Según mi `calculos.py`, la API parte de unos 23,6 MiB y cada llamada le suma 15 MiB, así que llega a 38,6, 53,6 y 68,6 MiB (por debajo de mi límite de 76 MiB) y en la cuarta llega a 83,6 MiB y se pasa. Como no puede usar swap, el kernel la cierra a la fuerza con SIGKILL, y por eso sale exit code 137, que es 128 + 9 (el número de la señal SIGKILL).

## 5. ¿Por qué memswap_limit debe ser igual a mem_limit? ¿Qué pasaría si pudiera usar swap?

R// `memswap_limit` es el límite de memoria más swap juntos. Si lo dejo igual a `mem_limit`, no queda nada para el swap (`memory.swap.max` es 0), y cuando la API se pasa de la RAM permitida el kernel no tiene dónde guardar el exceso y la mata. Si pudiera usar swap, esa memoria de más se iría al disco: la API no moriría en la llamada 4, solo se pondría más lenta, y mi predicción no se cumpliría.

## 6. ¿Qué es un descriptor de archivo? ¿Por qué cada /slow ocupa 2 y cuántas caben a la vez en su API?

R// Un descriptor de archivo es un número que el kernel le da a un proceso cada vez que abre un archivo o una conexión, y cada proceso tiene un máximo (en la API son 256). Cada `/slow` ocupa 2 porque abre una conexión para el cliente que llega desde Nginx y otra hacia Redis, y las mantiene abiertas 15 segundos. Como mi API ya usa 5 descriptores en reposo, caben (256 − 5) ÷ 2 = 125 peticiones a la vez. Si llegan más, aparece el error "Too many open files" y las que sobran esperan en la cola.

## 7. ¿Qué diferencia hay entre SIGTERM y SIGKILL? ¿Por qué la API necesita un manejador de SIGTERM siendo el PID 1?

R// SIGTERM es un aviso amable para que el programa cierre en orden y termine con código 0. SIGKILL es un corte inmediato: el programa no puede atenderlo ni despedirse, y termina con código 137. Como la API es el PID 1 de su contenedor, el kernel no le aplica la reacción normal a SIGTERM, así que si el programa no tiene un manejador para esa señal, simplemente la ignora. Entonces `docker stop` espera 10 segundos y al final manda SIGKILL.

## 8. Describa el camino de curl localhost:8000/ hasta Redis (Sección 1.4), nombrando el puerto 8000, el bridge y los veth.

R// Mi `curl` llega al puerto 8000 de mi computador, y de ahí Docker lo lleva al puerto 80 del contenedor gateway (Nginx). Nginx busca `order-api` por su nombre en el DNS interno y le pasa la petición. Entre contenedores el tráfico viaja por los cables virtuales veth hasta el bridge `br-ae2d723e2b50`, que hace de switch y conecta a todos. La API (puerto 8080) le pide a Redis (puerto 6379) que sume la visita, y la respuesta `OK visita=N` regresa por el mismo camino hasta mi terminal.

## 9. ¿Qué diferencia hay entre un namespace y un cgroup? Dé un ejemplo de cada uno.

R// Un namespace controla qué puede ver un proceso, son sus paredes. Por ejemplo, el namespace PID hace que la API se vea como el proceso 1 aunque en mi Fedora tenga otro número. Un cgroup controla cuánto puede gastar, son sus medidores y límites. Por ejemplo, `memory.max` limita la API a 76 MiB y, si se pasa, el kernel la mata. En corto: el namespace aísla y el cgroup limita.


## 10. ¿Qué diferencias hay entre el hardware de su equipo y lo que ve Linux (Tabla 1)?

R// Mi equipo tiene un Intel Core i5-12450H con 8 núcleos físicos y 12 hilos, y 8 GiB de RAM (4 × 2 GiB LPDDR5). Linux ve 12 CPUs (`nproc`) y 7,4 GiB disponibles. Uso Fedora nativo, no WSL2, así que Linux ve el hardware real al 100%. (I use Arch btw :v)