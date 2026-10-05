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