### Arquitectura y Organización de Computadoras - DC - UBA 
# Segundo Parcial - Primer Cuatrimestre de 2025

## Enunciado

# EJERCICIO 1

En un sistema similar al que implementamos en los talleres del curso (modo protegido con paginación activada), se tienen varias tareas en ejecución. Se desea agregar al sistema una syscall que le permita a la tarea que la llama espiar la memoria de las otras tareas en ejecución. En particular queremos copiar 4 bytes de una dirección de la tarea a espiar en una dirección de la tarea llamadora (tarea espía). La syscall tendrá los siguientes parámetros:

  -  El selector de la tarea a espiar.
  -  La dirección virtual a leer de la tarea espiada.
  -  La dirección virtual a escribir de la tarea espía.

Si la dirección a espiar no está mapeada en el espacio de direcciones de la tarea correspondiente, la syscall deberá devolver −1 en eax, por el contrario, si se pudo hacer correctamente la operación deberá devolver 0 en eax.

Se pide:

  a) Definir o modificar las estructuras de sistema necesarias para que dicho servicio pueda ser invocado.
  b) Implementar la syscall y especificar claramente la forma de pasarle los parámetros correspondientes.

  Se recomienda organizar la resolución del ejercicio realizando paso a paso los items mencionados
 anteriormente y explicar las decisiones que toman.


 Recordemos las syscalls son interrupciones llamadas por el usuario(nivel 3), le permiten a las tarea hacer cosas que no pueden hacer poque tienen privilegios de nivel 0.
 Por ejemplo cambiar la paginacion como en este caso.
 Agreguemos esta nueva syscall , para esto tenemos que hacer una entrada de la IDT(moyor a 32 ya que estan reservados, y destino a los que usamos en general, entre 0x80 y 0xff)

 ```C
idt.c
```

```C
void idt_t(){
  //....
  IDT_ENTRY3(99); 
  //....
}
```

La idea de la rutina de atencion de la syscall es que se asume que el parametro de entrada va a estar en el registro edi de la tarea, asique lo pusheamos y luego llamamos a la funcion espia definidas en c.

```C
isr.asm

```

```C
extern espia

global _isr99

_isr99
  pushad

  push esi
  push edi
  push eax
  call espiar
  add esp, 12

  mov  [esp + 28], ax

  popad
  iret
```

```C
int espiar(int16_t selector, vaddr_t* virt_read, vaddr_t* virt_write){
    uint32_t cr3 = task_selector_to_cr3(selector);
    uint32_t cr3_tarea_espia = rcr3();

    paddr_t dir_fisica_a_espiar = virt_to_phy(cr3, virt_read);

    if (dir_fisica_a_espiar == 0) return -1;
    // No está mapeado el origen -> error


     // Mapear ventana temporal en el CR3 del espía al frame físico de origen
    //Solo lectura, user
    mmu_map_page(cr3_tarea_espia, SRC_VIRT_PAGE, dir_fisica_a_espiar, MMU_P | MMU_U)
    /*Nota: Acá usé SRC_VIRT_PAGE definida en copy_page()
 Podría usar otra dirección virtual pero es importante usar una reservada
 para no pisar un mapeo válido *∕

  uint32_t dato_a_copiar = *((SRC_VIRT_PAGE & 0xFFFFFF000) | VIRT_PAGE_OFFSET(direccion_a_espiar));

   //Desmapear ventana temporal
  mmu_unmap_page(cr3_tarea_espia,SRC_VIRT_PAGE);

  virt_write[0] = dato_a_copiar;

  return 0;
  }
```


```C
vaddr_t src_tmp = (SRC_VIRT_PAGE & 0xFFFFF000) | (src_virt & 0x00000FFF);
(src_virt & 0x00000FFF) extrae el offset (los 12 bits bajos).

(SRC_VIRT_PAGE & 0xFFFFF000) deja la base alineada de la ventana (los 12 bits bajos en cero).

El | (OR) combina base + offset ⇒ dirección exacta dentro de la ventana.

uint32_t valor = *(volatile uint32_t*)src_tmp;
Lee los 4 bytes desde esa dirección. volatile es para que el compilador no optimice la lectura (queremos sí o sí tocar memoria).
```

```C
paddr_t virt_to_phy(uint32_t cr3, vaddr_t* dir_a_espiar) {

 uint32_t pd_index = VIRT_PAGE_DIR(virtual_address);
 uint32_t pt_index = VIRT_PAGE_TABLE(virtual_address);
 
 pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

 pt_entry_t* pt = pd[pd_index].pt << 12;
 
 return (paddr_t) (pt[pt_index].page << 12);
}
```
```C
uint32_t task_selector_to_CR3(uint16_t selector) {
 uint16_t index = selector >> 3; // Sacamos los atributos
 gdt_entry_t* taskDescriptor = &gdt[index]; // Indexamos en la gdt
 tss_t* tss = (tss_t*)((taskDescriptor->base_15_0) |
 (taskDescriptor->base_23_16 << 16) |
 (taskDescriptor->base_31_24 << 24));
 return tss->cr3;
}
```

# EJERCICIO 2


Partiendo del sistema trabajado en los talleres, se pide modificar la política del scheduler. El nuevo scheduler distingue tareas prioritarias de no prioritarias. Las tareas prioritarias son aquellas que, al saltar la interrupción del reloj, tengan el valor 0x00FAFAFA en EDX. Las tareas pausadas y/o no ejecutables no pueden ser prioritarias.
La forma en la que el nuevo scheduler determina la siguiente tarea a ejecutar es la siguiente:

  1.  Si hay otra tarea prioritaria distinta se elige esa. En caso de haber más de una se hace de forma round-robin (como en el scheduler de los talleres).
  2.  Si no, se elige la próxima tarea como en el scheduler de los talleres.

La solución propuesta debe poder responder las siguientes preguntas:
 • ¿Dónde se guarda el EDX de nivel de usuario de las tareas desalojadas por el scheduler?
 • ¿Cómo determina el scheduler que una tarea es prioritaria?
 Se recomienda organizar la resolución del ejercicio explicando paso a paso las decisiones que toman.


```C
static sched_entry_t sched_tasks[MAX_TASKS] = {0};
int8_t current_task = 0
int8_t last_task_priority = 0;
int8_t last_task_no_priority = 0;

uint16_t sched_next_task(void) {
    ∕∕ Buscamos la próxima tarea viva con prioridad
  for ( i = (last_task_priority + 1); (i % MAX_TASKS) != last_task_priority; i++) {
    if (sched_tasks[i % MAX_TASKS].state == TASK_RUNNABLE && es_prioritaria(i)) {
    break;
   }




  if(no hay otra con prioridad) hacemos lo de siempre
  // Buscamos la próxima tarea viva (comenzando en la actual)
  int8_t i;
  for (i = (current_task + 1); (i % MAX_TASKS) != current_task; i++) {
    // Si esta tarea está disponible la ejecutamos
    if (sched_tasks[i % MAX_TASKS].state == TASK_RUNNABLE) {
      break;
    }
  }

  // Ajustamos i para que esté entre 0 y MAX_TASKS-1
  i = i % MAX_TASKS;

  // Si la tarea que encontramos es ejecutable entonces vamos a correrla.
  if (sched_tasks[i].state == TASK_RUNNABLE) {
    current_task = i;
    return sched_tasks[i].selector;
  }

  // En el peor de los casos no hay ninguna tarea viva. Usemos la idle como
  // selector.
  return GDT_IDX_TASK_IDLE << 3;
}

```
Vamos a necesitar una rutina de reloj que llame a sched_next_task, podemos usar una
versión simplificada de la del TP. Escribámosla ahora:

```C
global _isr32
_isr32:
    pushad
    call pic_finish1
    call sched_next_task

    ; Si la próxima tarea es 0 o la misma que corre, no saltamos
    cmp ax, 0 ; el scheduler dijo "no cambiamos de tarea"
    je .fin

    str cx ; chequeamos el TR actual
    cmp ax, cx
    je .fin

    mov word [sched_task_selector], ax
    jmp far [sched_task_offset]


    .fin:
    popad
    iret

```

Y oara identificar si una tarea es prioritaria lo que hacemos es:

(nose si esta bien)
```C
bool es_prioritaria(int8_t tarea_id){
      if (sched_tasks[tarea_id].state != TASK_RUNNABLE) return false;

      shced_entry_t* tarea = &sched_tasks[tarea_id];
      uint32_t tarea_edx = task_selector_to_EDX(tarea->selector);
      return tarea_edx == 0x00FAFAFA;
}

uint32_t task_selector_to_EDX(uint16_t selector) {
   uint16_t index = selector >> 3; // Sacamos los atributos
   gdt_entry_t* taskDescriptor = &gdt[index]; // Indexamos en la gdt
   tss_t* tss = (tss_t*)((taskDescriptor->base_15_0) | (taskDescriptor->base_23_16 << 16)|
   (taskDescriptor->base_31_24 << 24));
   return tss->edx;

```

en la clase hicieron:
Como identificar tareas prioritarias ( es_prioritaria(i) )?
Necesitamos revisar el valor de edx que tenía la tarea al momento de ser
interrumpida por el clock. La tarea no está actualmente en ejecución, entonces dónde
está su información? En la TSS.
La TSS se actualiza cuando hacemos jmp far ( jmp sel:offset ), por lo que en la TSS
se guardan los valores de los registros al momento del jmp.
 edx es un registro no volátil, por lo que si recordamos la rutina de reloj de antes,
después de los llamados a funciones de C ( sched_next_task ) es probable que no
tenga el mismo valor que nos había llegado.
Es decir, TSS.edx no tiene el valor que buscamos.

```C
tss_t* obtener_TSS(uint16_t segsel) {
  uint16_t idx = segsel >> 3;
  gdt_entry_t* taskDescriptor = &gdt[idx];
  uint32_t base =((taskDescriptor->base_15_0) |(taskDescriptor->base_23_16 << 16) |(taskDescriptor->base_31_24 << 24));
  return (tss_t*) base;;
}
```
obtener_TSS(segsel): dado un selector de TSS (el de la tarea), quieren obtener el puntero a la estructura tss_t de esa tarea leyendo la base del descriptor en la GDT.

```C
uint8_t es_prioritaria(uint8_t idx) {
  tss_t* tss_task = obtener_TSS(sched_tasks[i].selector);
  uint32_t* pila = tss_task->esp;
  uint32_t edx = pila[5];
  return edx == 0x00FAFAFA;
}
```

es_prioritaria(idx): con la TSS de la tarea, intentan llegar al EDX de nivel usuario mirando la pila y comparar con 0x00FAFAFA.
