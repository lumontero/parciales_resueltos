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




#EJERCICIO 1 

Para que una tarea de nivel de usuario pueda pedir memoria y liberarla, se usaran syscalls. El usuario deberá llamar a la syscall correspondiente a malloco dejando en ecx la cantidad de memoria que desea reservar. Esta syscall va a reservar la memoria llamando a la función malloco y dejar el resultado en eax para ser leido por la tarea. Por otro lado, para liberar la memoria, habrá otra syscall que deberá liberar la memoria usando la función chau.

Acá estarían los agregados para hacer funcionar esto:

  isr.asm

      ; ...

    global _isr90
    _isr90:
    pushad

    push ecx
    call malloco ; dejará en eax la dirección reservada
    add esp, 4
    mov [esp + 28], eax 

    popad
    iret

    global _isr91
    _isr91:
    pushad

    push ecx
    call chau
    add esp, 4

    popad
    iret

    ; ...

isr.h

    /* ... */
    void _isr90(); // malloc
    void _isr91(); // chau
    /* ... */

idt.c

    /* ... */
    void idt_init() {
    /* ... */
    IDT_ENTRY3(90);
    IDT_ENTRY3(91);
    }
    /* ... */

Luego de esto, se debe implementar la funcionalidad que asigne la memoria dinámicamente cuando el usuario quiere accederla. Para esto se debe modificar la rutina de atención del page fault, de manera que se asigne la memoria correspondiente en caso de que la dirección que se intenta acceder está reservada por malloco. Para lograr esto se puede usar la función esMemoriaReservada para verificar si la dirección que se quiere acceder está reservada. En caso de que lo esté, usar dameReservas para poder obtener la dirección virtual donde comienza el bloque y mapear la página correspondiente. En caso de que se haya intentado acceder a una página que no está reservada, se debe desalojar a la tarea y liberar toda su memoria reservada.

Primero, se agrega en el scheduler el estado TASK_KILLED y la función sched_kill_task, la cual dejará la tarea en estado TASK_KILLED y marcará toda su memoria reservada para ser desalojada. sched.c

      /* ... */

    typedef enum
    {
    TASK_SLOT_FREE,
    TASK_RUNNABLE,
    TASK_PAUSED,
    TASK_KILLED,//necesitás un estado en el que el scheduler ya no elija a esa tarea.
    }  task_state_t;

    /* ... */

    void sched_kill_task(int8_t task_id)
    {
      kassert(task_id >= 0 && task_id < MAX_TASKS, "Invalid task_id");
      sched_tasks[task_id].state = TASK_KILLED;
      reservas_por_tarea* reservas = dameReservas(current_task);
      for (size_t i = 0; i < reservas->reservas_size; i++){
        reserva_t* reserva = &reservas->array_reservas[i];
        reserva->estado = 2(RESERVA_MARCADA_LIBERADA); //la reserva está marcada para liberar
      }
    }

Luego, debemos agregar al page_fault_handler lo necesario para que se mapee la página de memoria en caso de que la dirección esté en alguna reserva del malloco de la tarea. mmu.c

      bool page_fault_handler(vaddr_t virt) {
      /* ... */

      //si la VA no pertenece a ningún rango reservado por la tarea, ni te gastás en buscar.
      if(esMemoriaReservada(virt)) {
        //Pedís el descriptor de reservas de la tarea que estaba corriendo cuando ocurrió el #PF
        reservas_por_tarea* reservas = dameReservas(current_task);

        
        for (size_t i = 0; i < reservas->reservas_size; i++)//Iterás las reservas de esa tarea.
          reserva_t* reserva = &reservas->array_reservas[i];

          //Chequeás si la VA que falló cae dentro del rango [inicio, inicio+tamaño].
          if(virt >= reserva->virt && virt < reserva->virt + reserva->tamanio)
          {
            paddr_t phy = mmu_next_free_user_page();//Pedís una página física para mapear
            
            mmu_map_zero_page(rcr3(), virt, phy, MMU_P | MMU_W | MMU_U);//Mapeás la página física en la VA
            
            zero_page(phy);// la limpiamos
            
            reserva->estado = 1;//si llego hasta aca ya estaba activa no se si es necesario esto
            return true;
          }
        }
      }
      //Si no estaba reservada o no encontraste un rango que la contenga, devolvés “no atendido”
      return false;
    }

Ahora debemos cambiar el comportamiento del isr14 (page fault) para que en caso de no resolverse el page fault, llame a sched_kill_task para desalojar la tarea junto con su memoria reservada. Luego pasamos a la siguiente tarea disponible. isr.asm


    global _isr14

    _isr14:
    pushad
    mov eax, cr2
    push eax
    call page_fault_handler
    cmp eax, 0
    jne .fin
    .ring0_exception:
    call sched_kill_task
    call sched_next_task

    mov word [sched_task_selector], ax
    jmp far [sched_task_offset]

    .fin:
    add esp, 4 ; cr2
    popad
    add esp, 4 ; error code
    iret

Ejercicio 2: (25 puntos)

Detallar todos los cambios que es necesario realizar sobre el kernel para incorporar la tarea `garbage_collector` si queremos que se ejecute una vez cada 100 ticks del reloj. Incluir una posible implementación del código de la tarea.


#EJERCICIO 2

Para crear la tarea garbage_collector, comenzaré creando el archivo de la tarea:

taskGarbageCollector.c

    #include "task_lib.h"
    #include "mmu.h"
    #include "../sched.h"
    #include "tasks.h"

    void gc_main_loop(void) {//la tarea garbage corre en buble infinito
	    while (true)
	  {
		    for (int i = 0; i < MAX_TASKS; i++) {//recorremos todas las tareas
		    	if (i == current_task) continue; // si es la que esta corriendo ahora no hacemos nada
				//Copia la entrada del scheduler de esa tarea a una variable local task. (Ojo: es copia, no puntero).
		    	sched_entry_t task = sched_tasks[i];

				//Pide la tabla de reservas de esa tarea. Hace selector >> 3 para pasar de selector TSS a índice GDT (quita RPL y TI)
		    	reservas_por_tarea* reservas = dameReservas(task.selector >> 3);

	    		for (int j = 0; j < reservas->reservas_size; j++) {//Recorre todas las reservas de esa tarea
				
		    		reserva_t* reserva = &reservas->array_reservas[j];//puntero al array de reservas de la tarea
					
		    		if(reserva->estado == 2) {//RESERVA_MARCADA_LIBERAR

						//Desmapea página por página todo el rango [virt, virt+tamanio)
		    			for (int page_addr = reserva->virt; page_addr < reserva->virt+reserva->tamanio;       page_addr+=PAGE_SIZE) {
		    				mmu_unmap_page(task_selector_to_cr3(task.selector), page_addr);
		    			}
		    			reserva->estado = 3;//la marco como liberada
		    		}
	    		}
	    	}
    	}
    }


Luego debemos agregar la tarea a la gdt y al scheduler para que sea ejecutada:

tasks.c


      dad
    void tasks_init(void)
    {
        /* ... */
      task_id = create_gc_task();
      sched_enable_task(task_id);
    }

      static int8_t create_gc_task() {
      size_t gdt_id;

      for (gdt_id = GDT_TSS_START; gdt_id < GDT_COUNT; gdt_id++) {
        if (gdt[gdt_id].p == 0)break;
      }

      kassert(gdt_id < GDT_COUNT,
              "No hay entradas disponibles en la GDT");

      int8_t task_id = sched_add_task(gdt_id << 3);
      tss_tasks[task_id] = tss_create_kernel_task(&gc_main_loop);

      gdt[gdt_id] = tss_gdt_entry_for_task(&tss_tasks[task_id]);
      return task_id;
    }

Para crear una tarea a nivel de kernel, debemos crear una nueva función para generar el tss de una tarea de nivel 0. Al vivir en el área del kernel, su stack de nivel 0 va a ser igual a su stack, ya que ambos serán de nivel 0.

tss.c

    tss_t tss_create_kernel_task(paddr_t code_start) {
    uint32_t cr3 = create_cr3_for_kernel_task(code_start);
    vaddr_t code_virt = TASK_CODE_VIRTUAL;
    vaddr_t stack0 = (vaddr_t) mmu_next_free_kernel_page();
    vaddr_t esp0 = stack0 + PAGE_SIZE;
    return (tss_t) {
    .cr3 = cr3,
    .esp = esp0,
    .ebp = esp0,
    .eip = code_start,
    .cs = GDT_CODE_0_SEL,
    .ds = GDT_DATA_0_SEL,
    .es = GDT_DATA_0_SEL,
    .fs = GDT_DATA_0_SEL,
    .gs = GDT_DATA_0_SEL,
    .ss = GDT_DATA_0_SEL,
    .ss0 = GDT_DATA_0_SEL,
    .esp0 = esp0,
    .eflags = EFLAGS_IF,
    };
    }

Ahora debo obtener un cr3 para esta tarea. Al ser una tarea de nivel 0, tanto el código como el stack vivirán en el área del kernel, por lo que basta con pedir una página de kernel para el page directory y realizar el identity mapping del área del kernel:

mmu.c

    paddr_t create_cr3_for_kernel_task() {
    // Inicializamos el directorio de paginas
    paddr_t task_page_dir = mmu_next_free_kernel_page();  
    zero_page(task_page_dir);
    // Realizamos el identity mapping
    for (uint32_t i = 0; i < identity_mapping_end; i += PAGE_SIZE) {
      mmu_map_page(task_page_dir, i, i, MMU_W);
    }
    return task_page_dir;
    }


Finalmente, queremos que la tarea se ejecute una vez cada 100 ticks, por lo que vamos a modificar un poco el scheduler para lograr este comportamiento.

sched.c

    // Defino una variable ticks donde voy a contar los ticks que pasaron desde la ultima ejecución de la tarea
    int8_t ticks = 0;

    /* ... */

    void clock_tick() {
      ticks++;
    }

    /* ... */

    uint16_t sched_next_task(void)
    {
      int8_t i;

      if(ticks > 1000) {
        i = gc_task_id;
      } else {
        // Buscamos la próxima tarea viva (comenzando en la actual)
        for (i = (current_task + 1); (i % MAX_TASKS) != current_task; i++)
        {
          // Si esta tarea está disponible la ejecutamos
          if (sched_tasks[i % MAX_TASKS].state == TASK_RUNNABLE)
          {
            break;
          }
        }
      }

      // Ajustamos i para que esté entre 0 y MAX_TASKS-1
      i = i % MAX_TASKS;

      sched_entry_t task = sched_tasks[i];

      // Si la tarea que encontramos es ejecutable entonces vamos a correrla.
      if (task.state == TASK_RUNNABLE)
      {
        current_task = i;

        return sched_tasks[i].selector;
      }

      // En el peor de los casos no hay ninguna tarea viva. Usemos la idle como
      // selector.
      return GDT_IDX_TASK_IDLE << 3;
    }

Para actualizar correctamente los ticks voy a agregar la llamada a la función clock_tick a la rutina de atención del clock.

isr.asm

    global _isr32
  
    _isr32:
    pushad
    call pic_finish1
  
    call next_clock
    call clock_tick
    call sched_next_task
  
    str cx
    cmp ax, cx
    je .fin
  
    mov word [sched_task_selector], ax
    jmp far [sched_task_offset]
  
    .fin:
      call tasks_tick
      call tasks_screen_update
      popad
      iret


    
Ejercicio 3:

a)Indicar dónde podría estár definida la estructura que lleva registro de las reservas (5 puntos)

  Podría ser mmu.c, tasks.c, sched.c... realmente no importa mucho, yo lo deje en mmu.c
  Y en cuanto a memoria física, simplemente pedía que esté en la parte de kernel si mal no recuerdo, así q literalmente en cualquier lado que elijas de ahí y que no te sobre-escriba otra cosa

b)Dar una implementación para `malloco` (10 puntos)

Considerando:
- Como máximo, una tarea puede tener asignados hasta 4 MB de memoria total. Si intenta reservar más memoria, la syscall deberá devolver `NULL`.
- El área de memoria virtual reservable empieza en la dirección `0xA10C0000`
- Cada tarea puede pedir varias veces memoria, pero no puede reservar más de 4 MB en total.
- No hace falta contemplar los casos en que las reservas tienen tamaños que no son múltiplos de 4KB. Es decir, las reservas siempre van a ocupar una cantidad de páginas, y dos reservas distintas nunca comparten una misma página.


      void* malloco(size_t size){
        if(size > 4MB){
           return NULL;
         }
        else{
          reservas_po_tarea* reservas = dameReservas(current_task);
          reserva_t* reservas_d_tarea = reservas->array_reservas;//ahora si sus elementos son reservas_t tienen vir , estado y tamanio
          uint32_t n = reservas->reservas_size;

          //CASO 1 -> es el primero bloque que quiere reservar
          if(reservas_d_tarea[0].estado == RESERVA_LIBRE){
              reservas_d_tarea[0].virt = 0xA10C0000
              reservas_d_tarea[0].tamanio = (uint32_t)size;
              reservas_d_tarea[0].estado = RESERVA_ACTIVADA;
              return (void*)0xA10C0000
          }else{
              //CASO 2 ->busco el primero libre, acumulo lo reservado y devuelvo
              uint32_t i = 0;
              size_t cantidad_total_reservada = 0;

              while(i < n && reservas_d_tarea[i].estado != RESERVA_LIBRE){
                  if(reservas_d_tarea[i].estado == RESERVA_ACTIVA){
                      cantidad_total_reservada += reservas_d_tarea[i].tamanio;
                  }
                  i++
              }
              // i = primer libre,  i - 1= ultima reserva ocupada
              if(cantidad_total_reservada + size > 4MB){
                return NULL;
              } else{
                  reservas_d_tarea[i].virt = reservas_d_tarea[i-1].virt + reservas_d_tarea[i-1].tamanio
                  reservas_d_tarea[i].tamanio = (uint32_t)size;
                  reservas_d_tarea[i].estado = RESERVA_ACTIVA;
                  return (void*)reservas_d_tarea[i].virt;
              }
          }
        
          }
        }

c)Dar una implementación para `chau` (10 puntos)

Considerando:
- Si se pasa un puntero que no fue asignado por la syscall `malloco`, el comportamiento de la syscall `chau` es indefinido.
- Si se pasa un puntero que ya fue liberado, la syscall `chau` no hará nada.
- Si se pasa un puntero que pertenece a un bloque reservado pero no es la dirección más baja, el comportamiento de la syscall `chau` es indefinido.
- Si la tarea continúa usando la memoria una vez liberada, el comportamiento del sistema es indefinido.
- No nos preocuparemos por reciclar la memoria liberada, bastará con liberarla


       void chau(vaddr_t virt){

        uint8_t i = 0;
        reservas_po_tarea* reservas  = dameReservas(current_task);
        reserva_t* reservas_d_tarea = reservas->array_reservas;
        uint32_t n = reservas->size;
  
        while(i < n && reservas_d_tarea[i].virt != virt){
          i++;
        }
        if(reservas_d_tarea[i].estado = RESERVA_ACTIVA){
           reservas_d_tarea[i].estado = RESERVA_MARCADA_LIBERADA;
        }
        }
