  Nos encargaron el desarrollo de un kernel para la nueva consola Orga Génesis que cuenta con un procesador x86. Para ahorrar trabajo vamos a tomar nuestro kernel y expandirlo para permitir la ejecución de juegos mediante cartuchos, que contendrán el código y los recursos gráficos (sprites de personajes, fondos). Se incorpora entonces un lector de cartuchos que, entre otras cosas, copiará los gráficos del cartucho a un buffer de video de 4 kB en memoria. Cada vez que el buffer esté lleno y listo para su procesamiento, el lector se lo informará al kernel mediante una interrupción externa mapeada al IRQ 40 de nuestro x86.
  
  Distintas tareas se ocuparán de mostrar los gráficos en pantalla y de realizar otros pre/postprocesamientos de imagen on-the-fly. Para esto, el sistema debe soportar que las tareas puedan acceder al buffer de video a través de los siguientes mecanismos:

- DMA (Direct Memory Access): se mappea la dirección virtual `0xBABAB000` del espacio de direcciones de la tarea directamente al buffer de video.

- Por copia: se realiza una copia del buffer en una página física específica y se mapea en una dirección virtual provista por la tarea. Cada tarea debe tener una copia única.

Dicho buffer se encuentra en la dirección física de memoria `0xF151C000` y solo debe ser modificable por el lector de cartuchos.

 Para solicitar acceso al buffer, las tareas deberán informar que desean acceder a él mediante una syscall `opendevice` habiendo configurado el tipo de acceso al buffer en la dirección virtual `0xACCE5000` (mapeada como r/w para la tarea). Allí almacenará una variable  `uint8_t acceso` con posibles valores `0`, `1` y `2`. El valor `0` indica que la tarea no accederá al buffer de video, `1` que accederá mediante DMA y `2` que accederá por copia. De acceder por copia, la dirección virtual donde realizar la copia estará dada por el valor del registro `ECX` al momento de llamar a `opendevice`, y sus permisos van a ser de r/w. Asumimos que las tareas tienen esa dirección virtual mapeada a alguna dirección física.

 El sistema no debe retomar la ejecución de estas tareas hasta que se detecte que el buffer está listo y se haya realizado el mapeo DMA o la copia correspondiente. Una vez que la tarea termine de utilizar el buffer, deberá indicarlo mediante la syscall `closedevice`. En ésta se debe retirar el acceso al buffer por DMA o dejar de actualizar la copia, según corresponda.

 La interrupción de buffer completo será la encargada de dar el acceso correspondiente a las tareas que lo hayan solicitado y actualizar las copia del buffer "vivas". Es deseable que cada tarea que accede por copia mantenga una única copia del buffer para no ocupar la memoria innecesariamente.

Como las direcciones que utilizamos viven por fuera de los 817MB definidos en los segmentos, asumimos que los segmentos de codigo y datos de nivel 0 y 3 ocupan toda la memoria (4 GB)

![Flujo del sistema](./img/esquema_cartucho.png)

## Ejercicio 1:
- a) Programar la rutina que atenderá la interrupción que el lector de cartuchos generará al terminar de llenar el buffer.
	- Consejo: programar una función deviceready y llamarla desde esta rutina.
- b) Programar las syscalls opendevice y closedevice.
- Cuentan con las siguientes funciones ya implementadas:
	- void buffer_dma(pd_entry_t* pd) que dado el page directory de una tarea realice el mapeo del buffer en modo DMA.
	- void buffer_copy(pd_entry_t* pd, paddr_t phys, vaddr_t virt) que dado el page directory de una tarea realice la copia del buffer a la dirección física pasada por parámetro y realice el mapeo a la dirección virtual pasada por parámetro.
## Ejercicio 2:
- a) Programar la función void buffer_dma(pd_entry_t* pd)
- b) Programar la función void buffer_copy(pd_entry_t* pd, paddr_t phys, vaddr_t virt)

	
 		 Entonces tenemos que:
 		 - resolver deviceready
 		 - implementar las syscalss de acceso al buffer
    			* opendevice
    			* closedevice
 		 - Implementar los mapeos de paginacion void buffer_dma(pd_entry_t* pd) y buffer_copy(pd_entry_t* pd, paddr_t phys, vaddr_t virt)

		
## Resolusion:
	Veamos que estructuras vamos a tener que editar:
	-IDT -> Tenemos que agregar una interrupcion por hardware y dos syscalls
	*	Como la interrupcion es de hardware, entonces el kernel es el unico que 
	puede atender a la misma(ring/nivel 0)
	*	Como las syscalls son de servicios que provee el SO al usuario, entonces estas
	deben ser capaces de ser llamadas por un usuario, y el codigo  que ejecuten es de nivel 0.

	-TASK_STATE -> Tenemos que agregarle informacion a la tarea sobre la direccion virtual
	que va a usar para mapear el buffer. En particular , podemos agregar un estado a una tarea
	a la cual sea BLOCKED, la cual da un indicio de que esta bloqueada esperando a tener acceso al device.

	-SCHED_ENTRY -> Esta estructura posse toda la informacion de la tarea. 
	Podemos agregarle la informacion sobre el modo de acceso que va a tener la tarea y la
	direccion virtual en donde va a poder encontrar el buffer


	Ediciones para las estructuras:
	 **idt.c**
	  void idt_init() {
 	 //… interrupcion externa mapeada al IRQ 40
 	 IDT_ENTRY0(40); 
 	 // …… Syscalls
 	 IDT_ENTRY3(90);
 	 IDT_ENTRY3(91);
	}

	**task.c**
	typedef struct {
	  int16_t selector;
	  task_state_t state;

	  //Agrego lo q mencione arriba
	  uint32_t copyDir;
	  uint8_t mode;
	} sched_entry_t;


	typedef enum {
	  TASK_SLOT_FREE,
 	 TASK_RUNNABLE,
 	 TASK_PAUSED,
  	// Nuevo estado de tarea
 	 TASK_BLOCKED,
 	 //Estado para usar despues
 	 TASK_KILLED
	} task_state_t;

    typedef enum {
     NO_ACCESS,
 	 ACCESS_DMA,
	 ACCESS_COPIA
	} task_access_mode_t;

## Deviceready:
Cuando se ejecuta esta funcion, entonces el kernel toma posesion de la tarea que estaba ejecutando, y empieza a mapear a todas las tarea que hayan solicitado acceso al area del buffer.

	Para eso la funcion deberia:
	- Iterar sobre todas las tareas definas en el scheduler.
	- Comprobar si esta esperando para acceder o si ya tiene acceso al buffer
	- Actualiazar las estructuras de paginacion segun corresponda:
		* La tarea esta solicitando acceso:
			# Si es por DMA entonces tenemos que mapear la direccion virtual 0xBABAB000 a la 
			
			direccion fisica 0xF151C0000 con permisos de usuraio y Read-Only.
			# Si es por copia, mapeamos la direccion virtual pasada por parametro a una nueva
			
			direccion fisica(en caso de primer mapeo) y hacemos la copia de datos.
		* La tarea ya tiene acceso:
			# Si es pod DMA no tenemos que hacer nada.
			# Si es por copia actualizamos la copia.


