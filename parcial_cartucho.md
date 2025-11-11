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
   			// (uint32_t)pd CONTENIDO A CARGAR EN CR3
   			//(vaddr_t)0xBABAB000 la direccion virtual que se ha de traducir de phy
   			//(paddr_t)0xF151C000 la direccion fisica que debe ser accedida(direc destinio)
   			// MMU_U | MMU_P atributos
   		}

   		void buffer_copy(pd_entry_t pd, paddr_t phyDir , vaddr_t copyDir){
   			mmu_map_page((uint32_t)pd, copyDir, phyDir, MMU_U | MMU_W | MMU_P);
   			// (uint32_t)pd CONTENIDO A CARGAR EN CR3
   			//copyDir la direccion virtual que se ha de traducir de phy
   			//phyDir la direccion fisica que debe ser accedida(direc destinio)
   			// MMU_U | MMU_W | MMU_P atributos
   			copy_page(phyDir, (paddr_t)0xF151C000);
   			//phyDir la direccion fisica , pagina donde queremos copiar el comtenido
   			//(paddr_t)0xF151C000 la direccion fisica cuyo contenido queremos copiar
   		}
   


## Funciones auxiliares

Nos que dan la funciones auxiliares:

 - uint32_t task_selector_to_cr3(uint16_t selector); nos permite entrar al directorio depaginas de una tarea cualquiera en base a su task segment.

 - paddr_t virt_to_phy(unint32_t cr3, vaddr_t virt); devuelvela direccion fisica asociada al cr3 y la direccion virtual pasadas por parametro





 		uint32_t task_selector_to_cr3(uint16_t selector){
   			uint16_t index = selector >> 3; // sacamos los atributos
   			gdt_entry_t* taskDescriptor = &gdt[index]; // indexamos en la gdt
   			//taskDescriptor guarda la dirección de la entrada gdt[index](apunta a una gdt_entry_t)
   			tss_t* tss = (tss_t*)((taskDescriptor->base_15_0)|
			                      (taskDescriptor->base_23_16 << 16) |
                                  (taskDescriptor->base_31_24 << 24) )
   			//recontruimos la base del tss decriptor que apunta al comienzo de la tss
   			return tss-> cr3; // agarro el cr3 de la tss
   			}



   		paddr_t virt_to_phy(uint32_t cr3 , vaddr_t virt){

   			uint32_t pd_index = VIRT_PAGE_DIR(virt); //elige qué PDE usar.
   			uint32_t pt_index = VIRT_PAGE_TABLE(virt); //elige qué PTE usar dentro de esa PT.

   			pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3); //Convierte el cr3 en un puntero al Page Directory de ESA tarea
   			// pd = puntero a la page directpry de esa tarea

   			pt_entry_t* pt = pd[pd_index].pt << 12;
   			//Toma el PDE correspondiente y obtiene la dirección base de la Page Table
   			// el campo .pt guarda el los 20 bits mas altos(address of page table) entonces
   			// << 12 shifteamos 12 para convertirlo en direccion base alineada a la pt

   			return (paddr_t)(pt[pt_index].page << 12);
   			//Toma el PTE dentro de esa PT y devuelve la dirección base física de la página de datos
   		}
   







# EXTRA

Queremos agregar una nueva tarea

Pasos a realizar

(Estructuras) Debemos: 

● Agregar una entrada en la GDT con la TSS de la nueva tarea a ejecutar. 

● Agregar una entrada de tarea en el scheduler. 

● Agregar alguna estructura adicional para que se vayan contando los ticks que pasan. 

(Funciones): 

● Tenemos que implementar el código para la tarea, la cual ejecuta siempre el mismo código y 
permite “matar” a aquellas tareas que se aprovechen de la CPU. 

● Tenemos que modificar la rutina del clock para que cuente la cantidad de ciclos que pasaron 
para cada tarea.


Creamos el tss de una tarea nivel 0(en tss.c):

	tss_t tss_create_kernel_task(paddr_t code_start) {
 	 uint32_t stak = mmu_next_free_kernel_page();
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

cr3 = create_cr3_for_kernel_task() → CR3 con mapeo de kernel. Por qué el killer es una tarea 
propia del kernel, no hereda nada de userland y debe tener su CR3 de kernel, sus segmentos ring 0
y su pila ring 0.

Creamos (en tasks.c)

	static int8_t create_task_kill(){
		//declara una variable para guardar en qué entrada de la GDT vamos a poner el descriptor TSS del killer
		size_t gdt_is; 
		//recorre la GDT desde GDT_TSS_START hasta GDT_COUNT buscando un hueco libre
		for(gdt_id = GDT_TSS_START; gdi_id < GDT_COUNT; gdr_id++){
			if (gdt[gdt_id].p == 0) break;//gdt[gdt_id].p == 0 significa “entry no presente” → libre para usar.
			//cuando encuentra una libre, hace break y se queda con ese índice.
		}
		// explota si la condicion es falsa
		kasset(gdt_id < GDT_COUNT, "No hay entradas disponibles en la GDT");

		int8_i task_id = sched_add_task(gdt_id << 3); 
		//crea una entrada en el scheduler para esta nueva tarea y obtiene su task_id
		//le pasa al scheduler el selector de TSS que vamos a usar: gdt_id << 3

		tss_task[task_id] = tss_create_kernel_task(&killer_main_loop);
		//construye la TSS real en memoria para esta tarea de kernel:

		gdt[gdt_id] = tss_gdt_entry_for_task(&tss_task[task_id]);
		//escribe en la GDT el descriptor de TSS que apunta a la TSS que acabamos de crear

		// sched_tasks[task_id].status = TASK_RUNNABLE; (lo sumo el chat tiene sentido pero nose)
        // sched_tasks[task_id].mode   = NO_ACCESS;

		return task_id;
	}
Busca un slot libre en la GDT para la TSS del killer, mete la TSS, y agrega la tarea al sched_tasks	
El killer es una tarea más a ojos del scheduler (round-robin).

Creamos (en mmu.c)

	paddr_t create_cr3_for_kernel_task(){
	//inicializamos el directorio de paginas
	paddr_t task_page_dir = mmu_next_free_kernel_page(); //ide una página física libre para usarla como Page Directory (PD).
	zero_page(task_page_dir);

	//Hacemos el identity mapping
	for( uint32_t i = 0; i < identity_mapping_end; i += PAGE_SIZE){
		mmu_map_page(task_page_dir; i; i; MMU_P | MMU_W);
	}

	return task_page_dir;
	}

Define una función que arma un nuevo directorio de páginas para una tarea de kernel y 
devuelve su dirección física (eso es lo que va en CR3).
Crea un Page Directory para el killer e identidad mapea (VA=PA) el rango de kernel que necesita
El killer debe poder leer/escribir estructuras del kernel y ver memoria física del kernel sin depender de CR3 de userland.
Cómo lo hubiera deducido: “El killer no puede usar CR3 de usuario; le hago un CR3 propio con identity mapping del kernel”.
Hacemos identity mapping para que el CR3 del kernel tenga un mapa directo y simple: cualquier 
página física importante del kernel es accesible en la misma dirección virtual, con permisos solo
de ring 0. Eso simplifica muchísimo el código (y el TP), y permite que el task_killer y las
rutinas de MMU trabajen sin mapeos temporales a cada rato.

Rutina del clock

	isr.asm

	extern add_tick_to_task
	global _isr32

	_ir32:
		pushad

		call pic_finish1
		call next_clock

		call add_tick_to_task

		call sched_next_task ;Invoca al scheduler para decidir quién corre ahora

		;...

		popad
		iret


		en sched.c

		int8_t current_task = 0;

		//Un array de contadores: uno por tarea.
		//Cada entrada acumula cuántos ticks de reloj estuvo en ejecución (RUNNABLE y elegida) desde la última vez que lo reseteaste.
		static  uint8_t contador_de_ticks[MAX_TASK] = {0};

		void add_tick_to_task(){
		//Suma 1 al contador de la tarea que estaba corriendo cuando sonó el tick
			contador_de_ticks[current_task]++;
		}



TASK_KILLER

tasks.c

	void killer_main_loop(voud){
		while(true){ //Bucle eterno. Siempre repite el chequeo
			for(int i = 0; i < MAX_TASKS; i++){ //Recorre todas las posiciones del arreglo de tareas
				if(i == current) continue; //Si i es la tarea que está corriendo ahora (el killer mismo o la actual), la salta.

				sched_entry_t* task = &sched_task[i];
				//task apunta a la entrada del scheduler para esa i

				if (task->status == TASK_PAUSED) continue;//Si la tarea está pausada, la ignora
 				if (task->mode != NO_ACCESS && task->status != TASK_BLOCKED) continue;// creo q es not esto
 				if (task_ticks[i] <= 100) continue;//Si no superó los 100 ticks todavía, no la consideres ociosa → salta.

				vaddr_t vaddr_to_check = 0;//Variable para guardar la VA a chequear (dónde debería haber leído). Arranca en 0

				//Define qué VA mirar según el modo
				if (task->mode == ACCESS_DMA){
					vaddr_to_check = VADDR_DMA;//la VA fija del buffer, por ej. 0xBABAB000 (VADDR_DMA)
				}else{
					vaddr_to_check = task->copyDir;//la VA donde la tarea pidió su copia (task->copyDir
				}

				//Pide el PTE de esa tarea (task->selector → su TSS/CR3) para la VA elegida. Con el PTE podés mirar el bit Accessed (A).
				pte_t* pte = mmu_get_pte_for_task(task->selector, vaddr);
				if(pte->accessed == 0){
					task->status = TASK_KILLED; //Marca la tarea como “matada”
				{else{//Si sí accedió (no ociosa)
					tasks_ticks[i] = 0;//Resetea su contador de ticks
					pte->accessed = 0;//Limpia el bit Accessed para medir de nuevo la próxima vez
				//coment del chat Falta: hacer invlpg(vaddr) para que el TLB no mantenga el bit viejo en caché.
				}
			}
		}
	}

Es la “tarea asesina” del kernel. Corre en un bucle infinito y va revisando a las demás tareas
para ver si alguna está “al pepe” (ociosa) y hay que matarla.

		
mmu.c

	pt_entry_t* mmu_get_pte_for_task(uint16_t task_selector; vaddr_t virtual_address) {

		//Con el selector de tarea miro la GDT → descriptor TSS → BASE → TSS, y leo el CR3 de esa tarea.
		uint32_t* cr3 = task_selector_to_cr3(task_selector);//Ese cr3 es la PA del Page Directory de la tarea

		uint32_t pd_index = VIRT_PAGE_DIR(virtual_address);//pd_index = bits 31..22 (elige qué PDE mirar).
		uint32_t pt_index = VIRT_PAGE_TABLE(virtual_address);//pt_index = bits 21..12 (elige qué PTE de esa PT).

		//Consigo un puntero al Page Directory de ESA tarea, accesible desde el kernel.
		//CR3_TO_PAGE_DIR(cr3) te da una VA del kernel que “ve” la página física cuyo inicio es cr3
		pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

		pt_entry_t* pt = pd[pd_index].pt << 12;//Tomo el PDE (entrada del PD) que apunta a la Page Table

		return (pt_entry_t*) &(pt[pt_index]);
		//Devuelve la dirección del PTE que corresponde a virtual_address en ESA tarea
	}

Define una función que devuelve un puntero al PTE (entrada de tabla de páginas) correspondiente a 
la dirección virtual vaddr dentro del espacio de direcciones de ESA tarea (identificada por su 
selector de TSS).
