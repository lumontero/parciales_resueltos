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

Definimos el handler en asm

En isr.asm		

		extern deviceready
		global isr_40

		isr_40:
			pushad
			call pic_finish
			call deviceready
			popad
			iret
					
En idt.c

	 void deviceready(void){
 	 	for(int i = 0 ; i < MAX_TASKS ; i++){ //Recorre todo el scheduler.
    		sched_entry_t* tarea = &sched_tasks[i];
    		if(tarea->mode == NO_ACCESS) // No solicita acceso al buffer
      			continue; //si la tarea no pidió el device (NO_ACCESS), no hay nada que hacer.
    		if(tarea->status == BLOCKED){
      			if(tarea->mode == ACCESS_DMA){// Solicita acceso en modo DMA
        			buffer_dma(CR3_TO_PAGE_DIR(task_selecto_to_cr3(tarea->selector)));
      		}
      			if(tarea->mode == ACCESS_COPY){// Solicita acceso en modo por copia
      				  buffer_copy(task_selecto_to_cr3(tarea->selector), mmu_next_user_page(), tarea->copyDir);
     		}
      			tarea->status = TASK_RUNNABLE; // dejamos la tarea lista para correr en una proxima ejecucion
    }
    		else{//la tarea no está BLOCKED (o sea, ya tenía acceso de antes).
      			if(tarea->mode == ACCESS_COPY){
       				 paddr_t destino = virt_to_phy(task_selector_to_cr3(tarea->selector), tarea->copyDir);
       				 copy_page((paddr_t)0xF151C000, destino);
      }
    }
	  }
	} 

Donde usamos las fuciones auxiliares:


-uint32_t task_selector_to_CR3(uint16_t selector);

 Nos permite encontrar el directorio de páginas de una tarea cualquiera en base a su task segment.

-paddr_t virt_to_phy(uint32_t cr3, vaddr_t virt);

 Devuelve la dirección física asociada al cr3 y la dirección virtual pasadas por parámetro.
 
 Esta funcion asume que existe un mapeo bajo la direccion virt.



## OPENDEVICE

Syscall que permite a las tareas solicitar acceso al buffer segun el tipo configurado.

En el caso de acceso por copia, la direccion virtual donde realizar la copia estara por el valor

del registro ECX al momento de llamarla.

El sistema no debe retortnar la ejecucion de las tareas que llaman a la syscall hasta que se 

detecte que el buffer esta listo y se haya realizado el mapeo DMA o la copia correspondiente.

	isr.asm

	extern opendevice
	global isr_90
	
	isr_90:
	pushad  ;salvar todos los registros generales
	
	push ecx  ;pasar el parámetro a C (copyDir va en ECX)
	call opendevice ; opendevice(copyDir)
	add esp, 4 ;balancear la pila del llamado (sacar el parámetro)
	
	call schedd_next_task ;elegir la próxima tarea a ejecutar (RUNNABLE)
	;sched_next_task nos va a devolver en ax un selector de tarea distinto al que está ejecutando.

	mov word [sched_task_selector], ax
    jmp far [sched_task_offset] ;salto FAR: cambia de tarea (con TSS)

	popad
	iret

Recordemos que si sched_next_task no encuentra un selector de una tarea de su lista, entonces devuelve el selector de la tarea idle.
 
	idt.c

	void opendevice(unit32_t copyDir){
		sched_task[current_task].status = BLOCKED; // la bloqueamos
		sched_task[current_task].mode = *(unt8_t*)0xACCE5000; //leer modo pedido (NO/DMA/COPY)
		sched_task[current_task].copyDir = copyDir; //guardar VA destino para COPY
	}

## CLOSEDEVICE

Una vez que la tarea termina de utilizar el buffer, debe indicarlo haciendo usa de esta syscall.

En ella se debe retirar el acceso por DMA o dejar de actualizar la copia, segun corresponda.


	ism.asm

	extern closedevice
	global isr_91

	isr_91:
	pushad

	call closedevice

	popad
	iret


	idt.c

	void closedevice(void){
		if(sched_task[current_task].mode == ACCESS_DMA)
			mmu_unmap_page(rcr3(), (vaddr_t)0xBABAB000); 
			//quita esa traducción VA→PA en el espacio de direcciones de la tarea actual (por eso pasa rcr3()).

		if(sched_task[current_task].mode == ACCESS_COPY)
			mmu_unmap_page(rcr3(),sched_task[current_task].copyDir);
			//la tarea tenía su copia privada mapeada en la VA que ella eligió (copyDir, la guardaste en opendevice).Se desmapea esa VA.

		sched_task[current_task].mode = NO_ACCESS; //esta tarea ya no tiene acceso al device.
		//No toca status (sigue RUNNABLE normalmente; cerrar el device no la bloquea).



## Funciones de mapeo

Se nos pide implementar el mapeo asociado a los modos de acceso al buffer de video del cartucho.

Recordemos:

 - Si es por DMA entonces tenemos que mapear como solo lectura a 0xBABAB000(virtual) a 0xF151C0000(fisica).

 - Si es por copia, mapeamos la direccion virtual a la direccion fisica pasada por parametro y luego hacemos la copia de la pagina que comienda en 0xF151C000

		void buffer_dma(pd_entry_t pd){
			mmu_map_page((uint32_t)pd, (vaddr_t)0xBABAB000, (paddr_t)0xF151C000, MMU_U | MMU_P);
   		}

   		void buffer_copy(pd_entry_t pd, paddr_t phyDir , vaddr_t copyDir){
   			mmu_map_page((uint32_t)pd, copyDir, phyDir, MMU_U | MMU_W | MMU_P);
   			copy_page(phyDir, (paddr_t)0xF151C000);
   		}
   


## Funciones auxiliares

Nos que dan la funciones auxiliares:

 - uint32_t task_selector_to_cr3(uint16_t selector); nos permite entrar al directorio depaginas de una tarea cualquiera en base a su task segment.

 - paddr_t virt_to_phy(unint32_t cr3, vaddr_t virt); devuelvela direccion fisica asociada al cr3 y la direccion virtual pasadas por parametro





 		uint32_t task_selector_to_cr3(uint16_t selector){
   			uint16_t index = selector >> 3; // sacamos los atributos
   			gdt_entry_t* taskDescriptor = &gdt[index]; // indexamos en la gdt
   			tss_t* tss = (tss_t*)((taskDescriptor->base_15_0)|
			                      (taskDescriptor->base_23_16 << 16) |
                                  (taskDescriptor->base_31_24 << 24) )
   			return tss-> cr3;
   			}



   		paddr_t cirt_to_phy(uint32_t cr3 , vaddr_t virt){

			uint32_t* cr3 = task_selector_to_cr3(task_id);

   			uint32_t pd_index = VIRT_PAGE_DIR(virtual_address);
   			uint32_t pt_index = VIRT_PAGE_TABLE(virtual_address);

   			pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

   			pt_entry_t* pt = pd[pd_index].pt << 12;

   			return (paddr_t)(pt[pt_index].page << 12);
   		}
   

   
