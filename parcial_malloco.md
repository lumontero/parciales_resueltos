### Arquitectura y Organización de Computadoras - DC - UBA 
# Segundo Parcial - Primer Cuatrimestre de 2025

## Enunciado

En esta oportunidad, nos encargaron diseñar un sistema de asignación de memoria para nuestro kernel, que permita que cada tarea pueda pedir memoria de forma dinámica y liberarla en caso de que ya no la necesite. Como mecanismo de asignación de memoria, el kernel implementará un sistema de *lazy allocation* que permite que las tareas puedan pedir memoria, pero que el kernel no la reserve hasta que realmente se acceda a dicha memoria.

Contamos con la función `malloco` ya implementada en el kernel que recibe por parámetro la cantidad de memoria a reservar en bytes y devuelve la dirección virtual a partir de la cuál se reservó memoria (Pero aún no estará mapeada a direcciones físicas). Devolverá NULL en caso de que no haya memoria suficiente.

```C
void* malloco(size_t size);
```

Contamos con la función `esMemoriaReservada` que recibe una dirección virtual y retorna `true` si la misma pertenece a un bloque de memoria reservado por malloco, `false`si no.

```C
uint8_t esMemoriaReservada(virtaddr_t virt)
```

Contamos con la función `chau` que recibe una dirección virtual y, si corresponde al comienzo de un bloque reservado con malloco, marca esa reserva para que sea liberada más adelante por una tarea de nivel 0 (desarrollado más adelante). No nos preocuparemos por reciclar la memoria liberada, bastará con liberarla.

```C
void chau(virtaddr_t virt);
```

Para llevar registro de las reservas de memoria, el sistema cuenta con un array alojado estáticamente en la memoria del kernel. Los elementos del array serán  structs de la pinta:

```C
typedef struct {
  uint32_t task_id;
  reserva_t* array_reservas;
  uint32_t reservas_size;
} reservas_por_tarea; 
```

- task_id: identifica a qué tarea corresponden las reservas de este item
- array_reservas: array de reservas en donde una reserva es un struct reserva_t 
- reservas_size: cantidad de elementos de array_reservas
- No nos vamos a preocupar por el tamaño del array_reservas, asumiremos que tendrá espacio suficiente para satisfacer las necesidades del sistema.
  
```C
typedef struct {
  uint32_t virt; //direccion virtual donde comienza el bloque reservado
  uint32_t tamanio; //tamaño del bloque en bytes
  uint8_t estado; //0 si la casilla está libre, 1 si la reserva está activa, 2 si la reserva está marcada para liberar, 3 si la reserva ya fue liberada
} reserva_t; 
```

Contamos con la función `dameReservas` que recibe por parámetro un task_id y devuelve el puntero al elemento del array que corresponde a esa tarea.

```C
reservas_por_tarea* dameReservas(int task_id);
```

Como se implementará un sistema de *lazy allocation*, el kernel no asignará memoria física hasta que la tarea intente acceder, ya sea por escritura o lectura, a la memoria reservada. En el momento del acceso, si la dirección virtual corresponde a las reservadas por la tarea, el kernel deberá asignar memoria física a la dirección virtual que corresponda. La asignación es gradual, es decir, solamente se asignará una única página física por cada acceso a la memoria reservada. A medida que haya más accesos, se irán asignando más páginas físicas. Las páginas de memoria física son obtenidas del area libre de tareas, se asume que habrá suficientes. La memoria asignada por este mecanismo debe estar inicializada a cero (como cuando se reserva memoria con 'calloc'). Si el acceso es incorrecto porque la tarea está leyendo una dirección que no le corresponde, el kernel debe desalojar tarea inmediatamente y asegurarse de que no vuelva a correr, marcar la memoria reservada por la misma para que sea liberada y saltar a la próxima tarea.

La liberación de memoria va a estar a cargo de una tarea de nivel 0 llamada `garbage_collector` que recorrerá continuamente las reservas de las tareas en busca de reservas marcadas para liberar.

Ejercicio 1: (50 puntos)

Detallar todos los cambios que es necesario realizar sobre el kernel para que una tarea de nivel usuario pueda pedir memoria y liberarla, asumiendo como ya implementadas las funciones mencionadas. Para este ejercicio se puede asumir que `garbage_collector` está implementada y funcionando.



# EJERCICIO 1

Recordamos que teenemos:

- void* malloco(size_t size); le damos la cant de memoria a reservar(en bytes) y nos da la dir virtual a partie de la cual se reservo la memoria y da NULL en el caso de que no haya memoria suficiente.

- uint8_t esMemoriaReservada(vaddr_t virt) le damos una direccion virtual y da true si pertenece a un bloque de memoria reservado por malloco, false sino.

- void chau(virtaddr_t virt); le damos una dir virual, si corresponde al comienzo de un bloque reservado con malloco marca esa  reserva para que sea liberada más adelante por una tarea de nivel 0 (garbage_collector)

- reservas_por_tarea* dameReservas(int task_id); le damos un task_id y nos da el puntero al elemento del array que corresponde esa tarea.

La idea seria tener una lista de reservas de tamanio MAX_TASKS que guarde en el lugar de cada tarea otra lista con sus reservas.

Ya contamos con las estructuras reserva_t y estado_reserva_t faltaria agregar la siguiente estructura:

    typedef enum{
      RESERVA_LIBRE;
      RESERVA_ACTIVA;
      RESERVA_MARCADA_LIBERADA;
      RESERVA_LIBERADA;
    } estado_reserva_t

En la cual deficinos los distintos posibles estados de una resera.

Ahora para inicializar nuestra lista de reservas por tareas, la idea seria que la cantidad de filas sea igual a la cantidad de tareas(sin garbage_collector) y para la cantidad de columnas como que cada tarea va a tener su propia cantidad las cuales pueden diferir entre ellas por lo que definimos CANT_BLOQUES_MAX que seria la mayor cantidad de bloque que una tarea podria reservar, entonces las tareas que reserven menos de esa cantidad van a tener sus bloque y en los otros lugares que sobren todos en 0.
Lo inicializo en 0 ya que como que "arrancamos" sin que ninguna tarea haya reservado ningun bloque.
Por lo que nos quedaria algo asi:

    reserva_t reservas[MAX_TASKS][CANT_BLOQUES_MAX] = {{0}},


Tenemos que creer una entrada en la IDT para las syscalls(malloco y chau). Para la syscall malloco, defino que su id es 90 y para la de chau que sea 91. Ninguno de los dos se solapa con ningun codigo de exepcion de ninguna de las interrupciones que tenemos ya definidas en el tp.
Enronces en idt.c nos quedaria agregar esto:

    idt.c

    void idt_t(){
      //....
      IDT_ENTRY3(90); //malloco
      IDT_ENTRY3(91); //chau
      //....
    }

	isr.h
	//....
	void _isr90();
	void _isr91();
	//....

	
Deben ser con DPL = 3 para que puedan ser invocadas por cualquier tarea a nivel de usuario.

La idea de la rutina de atencion de ambas syscalls es que se asume que el parametro de entrada va a estar en el registro edi de la tarea, asique lo pusheamos y luego llamamos a las funciones malloco y chay definidas en c

    isr.asm

    extern malloco
    extern chau

    global _isr90
    global _isr91

    _isr90:
      pushad ;salvar todos los registros generales

      push edi ;pasar el parámetro a C (size va en EDI)
      call malloco ; malloco(size)
      add esp , 4 ;balancear la pila del llamado (sacar el parámetro)

      mov [esp + offset_EAX] , eax ; la funcion malloco devuelve la direccion virtual a partir la cual se reservo la memoria en eax

      popad
      iret
      (offset_EAX = 28)
      _isr91:
      pushad

      push edi
      call chau

      add esp , 4

      popad
      iret

Recordemos que CR2 es un registro de 32 bits que el procesador actualiza automáticamente cuando ocurre un page fault. Guarda la dirección línea (virtual) que provocó el page fault.

Como el kernel no asignara memoria fisica hasta que la tarea intente accedre a la memoria reservada, por lo que va a saltar una page fault porque cuando intente accedre todavia no va a estar mapeada la pagina fisica, por lo tanto voy a modificar ek mecanismo del page_fault_handler para que:
  - si la direccion virtual(CR2) a la cual se intento acceder formaba parte de la memoria que la tarea habia reservado antes con malloco, entonces que mapee la pagina fisica inicializanda a cero obtenida del area libre de tareas.
    en otras palabras si CR2 pertenece a un rango reservado por malloco de esa tarea ⇒ asignar 1 página física, ponerla en cero, mapearla en la página de CR2 (alineada), y volver a user.

  - si la direccion virtual(CR2) a la que se intento acceder no formaba parte de la memoria que la tarea habia reservado antes con malloco, entonces que la tarea se bloquee, toda la memoria que habia reservado se ponga en desuso, o sea, para liberar y se cambie inmediatamente de tarea.
    en otras palabras si no pertenece ⇒ la tarea hizo un acceso ilegal ⇒ sacarla de juego y marcar sus reservas para que el GC libere.

Para poder bloquear a la tarea si esta, es decri, no v a avolver a ser ejecutada ya que no va a estar en estado RUNNABLE por lo que el sheduler no la va a volver a elegir como next task nunca, para poder hacer esto modifico el tipo task_state_t del tp y le agrego el estado BLOCKED.

    typedef enum {
    TASK_SLOT_FREE,
    TASK_RUNNABLE,
    TASK_PAUSED,
    TASK_BLOCKED
    } task_state_t;


    bool page_fault_handler(vaddr_t virt) {

      if(puede_acceder(virt)){
        paddr_t direc_fisica = mmu_next_free_user_page();
        zero_page(direc_fisica);
        mmu_map_page(rcr3(), virt, direc_fisica, MMU_P | MMU_W | MMU_U);
        return true;

      }else{
        sched_tasks[current_task].state = TASK_BLOCKED;//el schedule no la va a volver a elegir
        for(int j = 0; j < CANT_BLOQUES_MAX; j++){
          if(reservas[current_task][j].estado = RESERVA_ACTIVA){
            reservas[current_task][j].estado = RESERVA_MARCADA_LIBERADA;
          }
        
        }
        return false;
       }  
    }

La funcion puede_acceder se fija si la direccion virtual que hizo saltar el page fault pertenece
a algun bloque de reserva de la tarea que quiso acceder a esa misma direccion, devuelve 1 en ese 
caso y 0 si no.

    bool puede_acceder(vaddr_t virt){
      for(int i = 0; i < CANT_BLOQUES_MAX; i++){
        if (reservas[current_task][i].estado != RESERVA_ACTIVA) continue; //Eso evita “acceder” a bloques ya marcados o liberados.(lo recomendo el chat)
        if(virt >= reservas[current_task][i].virt && virt < reservas[current_task][i].virt + reservas[current_task][i].tamanio){
          return true;
        }
      }
      return false;
    }

La _isr14 se encargara de cambiar de tarea de ser necesario.




Ejercicio 2: (25 puntos)

Detallar todos los cambios que es necesario realizar sobre el kernel para incorporar la tarea `garbage_collector` si queremos que se ejecute una vez cada 100 ticks del reloj. Incluir una posible implementación del código de la tarea.


 #EJERCICIO 2 

 Para la parte de la tarea de nivel 0 a la cual voy a llamar tarea_garbage, hay que hacer modificaciones y agregar cosas al kernel ya que en el tp solo tenemos tareas de nivel 3.

Queremos agregar una nueva tarea

Pasos a realizar

(Estructuras) Debemos:

● Agregar una entrada en la GDT con la TSS de la nueva tarea a ejecutar.

● Agregar una entrada de tarea en el scheduler.

● Agregar alguna estructura adicional para que se vayan contando los ticks que pasan.

(Funciones):

● Tenemos que implementar el código para la tarea, la cual ejecuta siempre el mismo código y permite “matar” a aquellas tareas que se aprovechen de la CPU.

● Tenemos que modificar la rutina del clock para que cuente la cantidad de ciclos que pasaron para cada tarea.

Creamos el tss de una tarea nivel 0(en tss.c): 

    tss_t tss_create_kernel_task(paddr_t code_start) {
    vaddr_t stack = mmu_next_free_kernel_page();
    return (tss_t) {
    .cr3 = create_cr3_for_kernel_task(),
    .esp = stack + PAGE_SIZE,
    .ebp = stack + PAGE_SIZE,
    .eip = (vaddr_t)code_start,
    .cs = GDT_CODE_0_SEL,
    .ds = GDT_DATA_0_SEL,
    .es = GDT_DATA_0_SEL,
    .fs = GDT_DATA_0_SEL,
    .gs = GDT_DATA_0_SEL,
    .ss = GDT_DATA_0_SEL,
    .ss0 = GDT_DATA_0_SEL,
    .esp0 = stack + PAGE_SIZE,
    .eflags = EFLAGS_IF,
    };
    }
 

Construye una TSS para un task de kernel (ring 0) que corre siempre en el espacio del kernel.

Reserva una pila de kernel (stack = mmu_next_free_kernel_page()), y setea esp/ebp/esp0 = stack + PAGE_SIZE

cr3 = create_cr3_for_kernel_task() → CR3 con mapeo de kernel. Por qué el killer es una tarea propia del kernel, no hereda nada de userland y debe tener su CR3 de kernel, sus segmentos ring 0 y su pila ring 0.

Crear la tarea:

    // globals  
    static tss_t    tss_garbage;
    uint16_t        garbage_selector;

    static int8_t create_task_garbage(){
      size_t gdt_id;
      for( gdt_id = GDT_TSS_START; gdi_id < GDT_COUNT; gdt_id++){
        if(gdt[gdt_id].p == 0) break;
      }

      kassert(gdt_id < GDT_COUNT, "No hay entradas disponibles en la GDT");

      //contruyo la tss ring 0 del garbage
      tss_t tss_garbage = tss_create_kernel_task((paddr_t)&garbage_main);

      // subo descriptor tss en la gdt y guardo el selector
      gdt[gdt_id] = tss_gdt_entry_for_task(&tss_garbage);
      garbage_selector = (uint16_t)(gdt_id << 3);
    }


Creamos (en mmu.c):

    paddr_t create_cr3_for_kernel_task(){
    //inicializamos el directorio de paginas
    paddr_t task_page_dir = mmu_next_free_kernel_page(); 
    zero_page(task_page_dir);

    //Hacemos el identity mapping
    for( uint32_t i = 0; i < identity_mapping_end; i += PAGE_SIZE){
	  mmu_map_page(task_page_dir; i; i; MMU_P | MMU_W);
    }

    return task_page_dir;
    }

 Define una función que arma un nuevo directorio de páginas para una tarea de kernel y devuelve su dirección física (eso es lo que va en CR3).


  Rutina del clock

    isr.asm

    extern tick_inc
    global _isr32

    _ir32:
	  pushad

	  call pic_finish1
	  call next_clock
    call tick_inc
  
	  call sched_next_task ;Invoca al scheduler para decidir quién corre ahora

	  ;...

	  popad
	  iret

Y  en sched.c

    volatile uint32_t tick_count = 0;

    void tick_inc(void){
      tick_count++;
    }

Ncesitamos que la tarea garbage_collector se ejecute periodicamente, pero sin romper el round-robin normal de las tareas de usuario. Para eso llevamos un contador global de tiks (tiks_count definido arriba) que se incrementa en cada interrupcion del reloj, y en sched_next_task inyectamos esta tarea solo cada 100 tiks.

    extern volatile uint32_t tick_count;
    static sched_entry_t garbage_collector
    
    uit16_t sched_next_task(void){
    //agregamos que cada 100 tikcs haga la tarea garbage_collector
      if(tick_count % 100 == 0){
        return garbage_collector.selector;
      }
    // round-robin normal que teniamos en el tp
    //...
    }

nunca hice la tarealoop(no la cheque es la de ine)

		void tarea_garbage(){
    while(true){
        for(int i = 0; i < MAX_TASKS; i++){
            for(int j = 0; j < CANT_NECESARIA; j++){
                if (reservas[i][j].estado == en_desuso){
                    uint32_t cr3 = tss_tasks[i].cr3;
                    vaddr_t direc_bloque = reservas[i][j].inicio;
                    for(direc_bloque = reservas[i][j].inicio, direc_bloque < reservas[i][j].fin, direc_bloque++){
                        if (esta_mapeada(cr3, direc_bloque)){                     
                            mmu_unmap_page(cr3, direc_bloque);
                        }  
                    }                   
                    reservas[i][j].estado == liberada;
                }
            }
        }
    }
    }
    
Ejercicio 3:

a)Indicar dónde podría estár definida la estructura que lleva registro de las reservas (5 puntos)

b)Dar una implementación para `malloco` (10 puntos)

Considerando:
- Como máximo, una tarea puede tener asignados hasta 4 MB de memoria total. Si intenta reservar más memoria, la syscall deberá devolver `NULL`.
- El área de memoria virtual reservable empieza en la dirección `0xA10C0000`
- Cada tarea puede pedir varias veces memoria, pero no puede reservar más de 4 MB en total.
- No hace falta contemplar los casos en que las reservas tienen tamaños que no son múltiplos de 4KB. Es decir, las reservas siempre van a ocupar una cantidad de páginas, y dos reservas distintas nunca comparten una misma página.

  Entonces:

  - Chequeamos que la tarea no este pidiendo mas de 4MB de memoria (1048576*4 bytes), si es asi returneamos NULL.
  - Si no sucede eso nos fijamos si es el primer bloque que pide reservar la tarea, si es asi el primer bloque empieza en 0xA10C0000 y termina en 0xA10C0000 + size.
    Marcamos el bloque con estado de RESERVA_NOSESILIBREOLIBERADA , nos guardamos su tamanio en size y como esta funcion devuelve la direccion virtuala partir de la cual se reservo memoria returneamos 0xA10C0000.
  - Si no es el primerblorque que pide reservar, buscamos cual fue el ultimo bloque en reservarse y me fijo si pidiendo la cantidad de memoria no me pso de los 4MB. Si nos pasamos rerutneamos NULL, sino significa que podemos reservarlo entonces guardamos en la direccion de inicio del nuevo bloque la direccion siguiente a la la de fin del bloque anterior(el final es virt + tamanio). Y por ultimo ponemos el estado del bloque en RESERVA_??, guardamos su size y delvolvemos el inicio del bloque que acabamos de reservar.



        void* malloco(size_t size){
          if(size > 4MB){
            return NULL;
          }
          else{
             if(reservas[current_task][0].estado == RESERVA_LIBRE){
                reservas[current_task][0].virt = 0xA10C0000;
                reservas[current_task][0].tamanio = size;
                reservas[current_task][0].estado = RESERVA_ACTIVA;
                return (vaddr_t*)0xA10C0000;// o (void*)0xA10C0000;
  
            }
            else{
                uint8_t i = 0;
                size_t catidad_total_reservada = 0;
                while( i < CANT_BLOQUES_MAX && reservas[current_task][i].estado != RESERVA_LIBRE){
                      catidad_total_reservada += reservas[current_task][i].tamanio ;
                      i++; 
                }//i es el primer libre -> i - 1 el la ultima
                if(catidad_total_reservada + size > 4MB){
                  return NULL;
                }
                else{
                reservas[current_task][i].virt =  reservas[current_task][i-1].virt +  reservas[current_task][i-1].tamanio; // aca nose si iria +1 o algo para q no se pisen
                reservas[current_task][i].tamanio = size;
                reservas[current_task][i].estado = RESERVA_ACTIVA;
                return (vaddr_t*)reservas[current_task][i+1].virt;
              }
        }
        }
        }


        //(if (reservas[current_task][0] == 0)  // ❌ no se puede comparar un struct con un numero)

c)Dar una implementación para `chau` (10 puntos)

Considerando:
- Si se pasa un puntero que no fue asignado por la syscall `malloco`, el comportamiento de la syscall `chau` es indefinido.
- Si se pasa un puntero que ya fue liberado, la syscall `chau` no hará nada.
- Si se pasa un puntero que pertenece a un bloque reservado pero no es la dirección más baja, el comportamiento de la syscall `chau` es indefinido.
- Si la tarea continúa usando la memoria una vez liberada, el comportamiento del sistema es indefinido.
- No nos preocuparemos por reciclar la memoria liberada, bastará con liberarla


Idea:
Buecamos la reserva a la cual le pertenece la direccion virtual pasada por parametro (donde comienza el bloque reservado por esa tarea) que queremos liberar.
Para eso recorremos reservas[current_task] y chequeamos cual empieza en la direccion que se pasa por parametro. Asumimos que esa direcicon esta reservada.

      void chau(vaddr_t virt){

        uint8_t i = 0;
        while(i < CANT_BLOQUES_MAX && reservas[current_task][i].virt != virt){
          i++;
        }
        if(reservas[current_task][i].estado = RESERVA_ACTIVA){
           reservas[current_task][i].estado = RESERVA_MARCADA_LIBERADA;
        }
        }
