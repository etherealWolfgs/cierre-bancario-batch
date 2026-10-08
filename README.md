# Cierre bancario con Spring Batch

**Autor:** Genaro Salvador Morales Paoli

## Cómo correrlo

    docker compose up -d --wait
    ./correr.sh 2026-09-30 prueba
    ./ver-batch.sh

## Día 1 · Mi primer Job

### Boleto de salida

1. ¿Qué diferencia hay entre un proceso batch y la API REST de la Semana 3? Da dos.

   Respuesta: La primera es quién lo inicia. La API REST arranca con cada petición de una persona o un sistema, y el batch lo lanza un horario o un operador una sola vez (el cierre de la noche). La segunda es cuándo termina. La API se queda escuchando siempre, y el batch procesa muchos datos de un jalón, sin que nadie interactúe con él, y termina solo al acabar el lote (lo vi en la MP-1, cuando la aplicación arrancó y me devolvió la terminal).

2. ¿Qué es un Job, qué es un Step y qué es un Tasklet?

   Respuesta: El Job es el proceso completo y el contenedor de los steps (aquí, `cierreDelDiaJob`). Un Step es una fase del Job, y un Job tiene uno o más (`saludoStep` y `verificarArchivoStep`). Un Tasklet es una sola operación que corre dentro de un step, y al step que la corre se le llama step de tipo Tasklet.

3. Con tus tablas: ¿qué diferencia hay entre una **JobInstance** y una **JobExecution**?

   Respuesta: La JobInstance es el Job para una fecha concreta, por ejemplo "el cierre del 28", y los parámetros con `identifying=true` son los que dicen de qué instancia se trata. La JobExecution es cada intento de correr esa instancia. En mis tablas, la instancia 1 (28 de septiembre) tiene la ejecución 1, que terminó `COMPLETED`. Una instancia puede tener varias ejecuciones cuando un intento falla.

4. ¿Por qué Spring Batch no deja correr dos veces el cierre del 28?

   Respuesta: Porque ya existe una instancia con `fecha=2026-09-28` y quedó completa, así que lanzó `JobInstanceAlreadyCompleteException`. Es una protección: en un banco, correr de nuevo un cierre que ya se hizo podría cobrar dos veces las comisiones de ese día. Para correrlo otra vez hay que cambiar los parámetros, o sea que sería otro cierre.

5. (MP-4, paso 6) Si mañana llega el archivo del 25 y corres otra vez el cierre del 25, ¿será otra instancia u otra ejecución de la misma? ¿Por qué lo crees?

   Respuesta: Creo que será otra ejecución de la misma instancia. La instancia del 25 ya existe (mismo Job y misma fecha), y quedó `FAILED`, no `COMPLETED`. Spring Batch solo bloquea las instancias completas, así que debería dejarme reintentarla, y eso dejaría una segunda ejecución ligada a la misma instancia. 

   ## Día 2 · El primer chunk

### Boleto de salida

1. ¿Qué diferencia hay entre un step de tipo Tasklet y uno de tipo chunk?
Un tasklet es un step simple, es decir, que no esta orientado a items (o unidad de datos como la fila de un archivo CSV). Ejecuta una tarea concreta como borrar un archivo, ejecutar un SQL o preparar directorios. Un chunk lee, procesa y escribe cada item y lo acumula hasta el tamaño de chunk para posteriormente hacer un commit y repite la accion hasta que el reader devuevle un null.  
2. ¿Qué hace cada una de las tres piezas de un chunk? ¿Cuál es opcional?
El ItemReader es OBLIGATORIO, lo que hace es leer el item una vez y cuando ya no hay más, devuelve un null y termina el step
El ItemProcessor es OPCIONAL, transforma, valida y filtra items, si devuelve un null, ese ítem se descarta y no llega al writer 
El ItemWriter es OBLIGATORIO. Recibe el chunk completo y lo escribre en lote.
Si no se configura ItemProcessor, el item leido pasa directamente al writer.
3. Con 45 movimientos y chunks de 10, ¿cuántos commits habría? ¿Y con chunks de 50?
Con chunk de 10= Se forman 5 commits de 4 de 10 y 1 de 5
Mientras que con el chunk de 50= los 45 movimientos caben en un solo bloque, por lo que es solo 1 commit 
4. ¿Por qué el Escritor recibe el chunk completo y no un movimiento a la vez?
Porque el ItemWriter está diseñado para escribir todo en lote. Su API recibe un chunk lo que permite operaciones eficientes como escritura de archivos en bloque, inserciones masivas, llamadas agrupadas, etc. 
5. Mi predicción de la MP-3, paso 1: ¿qué habría pasado sin el Procesador?
El 2 de octubre llegaron movimientos con valores de tipo “sucios”. El Procesador es la pieza que los limpia y normaliza antes de que el Writer los inserte en MySQL. Si se corre el cierre sin el Procesador, esos valores crudos llegan tal cual a la tabla, sin normalizar. Es decir, veriamos distintas variantes de los valores de tipo, unas con mayusculas otras totalmente en minisculas, sin tildes, con tildes, espacios al inicio o al final o con sinónimos. En lugar de existir 2 grupos, existirian muchos más, haciendo que las sumas queden incorrectas  


## Día 3 · Parámetros, fallas y reinicio

### Boleto de salida

1. ¿Qué diferencia hay entre una JobInstance y una JobExecution? Usa como ejemplo el cierre del 25.
El JobInstance es la definición lógica de la ejecución de un proceso para un conjunto específico de parámetros de entrada por ejemplo: 2026-12-25, el JobExecutio es cada vez que ese Job se ejecuta realemnte con esa configuración 
2. ¿En qué caso Spring Batch se niega a correr un cierre, y en qué caso lo reinicia?
Se niega a correr cuando el JobInstance ya se completó exitosamente y se reinicia cuando la ejecución anterior quedo en estado de FAILED o STOPPED desde el step que falló 
3. En el reinicio del día 5, ¿por qué el step de carga leyó 10 movimientos y no 20?
Porque cuando falla un renglon, el chunk completo donde este venía es descartado completamente 
4. ¿Qué diferencia hay entre un movimiento **filtrado** y uno **omitido**? 
Un movimiento filtrado ocurre dentro del ItemProcessor al retornar un null. El framework detecta este valor y no envía el registro al ItemWriter sin considerarlo un error y el movimiento omitido se activa al capturar una excepción durante la lectura, procesamiento o escritura de un step evitando que el Job aborte y "guardandolo" para su revisión a parte  
5. ¿Por qué importa el código de salida, si el estado ya queda en las tablas?
Porque para Batch, aunque se haya encontrado un error en un Job, este va a sacar siempre un Código de salida 0 que quiere decir que el Job se completó, pero para el planificador del banco (Control-M) todo está ok. 
El código de salida del Job no muestra el error, solo que se completó, aunque haya un error, cosa que no ve el planificador. En este ejemplo se agrega SpringApplication.Exit para obtener un valor de 5 para cuando falla el job y ese valor se interpreta como un FAILED.