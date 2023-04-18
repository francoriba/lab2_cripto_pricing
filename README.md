# Cripto pricing calculator
Repositorio para el laboratorio n°2 de la asignatura **"Sistemas de Computación"** de la FCEFyN , Universidad Naconal de Córdoba, Argentina. <br>
Grupo de trabajo: Internautas  <br>
Abril, 2023 <br>
Integrantes: 
 * Careggio, Camila
 * Casanueva, María Constanza
 * Riba, Franco <br>
 
 Profesores:
 * Jorge, Javier Alejandro 
 * Solinas, Miguel

## Ejecución
Para poder ejecutar el script "api.py" es necesario previamente crear la libreria "currencyconverterlib.so", puede hacer esto manualmente mediante los comandos comentados en el archivo "mul64.asm" o bien haciendo uso del Makefile mediante el comando "make" (que creará además el archivo ejecutable apto para ser debugeado mediante gdb y los archivos objeto asociados). Con el comando "make clean" puede eliminar los archivos de salida del Makefile. 

![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/mapa%20conceptual%20.png)

1. Ejecución de ``make`` y si todo sale bien, nos avisa que ya se hizo el build  
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/execution.png)  
2. Ejecutamos el script de python y vemos que nos va a pedir ingresar una crypto y una moneda. Si ingresamos valores válidos, realizará la conversión:  
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/execution2.png)  
3. El programa nos pregunta si queremos realizar otra conversión. Si decimos que no, finalizará, sino vuelve al paso 2.  
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/execution3.png)  
4. En el caso de que se haya ingresado un valor incorrecto, se le avisa al usuario y se vuelve a pedir el valor  
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/execution4.png)  

## Funcionamiento 
Este proyecto se divide en 3 capas. La primer capa utiliza un lenguaje de alto nivel como los es **python** para interactuar con una [API REST](https://www.coinapi.io/) con la que, mediante un esquema HTTP request-response, se obtiene la información correspondiente a:<br>
* Cotización en USD de alguna criptomoneda (BTC, ETH o LTC)<br>
* Cotización en relacion a USD de alguna moneda fiduciaria (USD, ARS, EUR) <br>   

Esta capa interactúa con la capa inferior proporcionandole la información (parametros) tomada de la API REST, esto se logra con ayuda de la libreria ```ctypes``` de Python que permite utlizar las funciones disponibles en una **shared library de c**. Además esta capa implementa las funcionalidades requeridas para que el usuario pueda interactuar con el programa.<br>

La siguiente capa consiste de un **programa de lenguaje c** donde se implementa la función:<br>

```float convert(float crypto_usd, float rate)```<br>

Esta invoca a la subrutina ```mult```, que conforma la capa más baja y se encuentra escrita en **lenguaje ensamblador**, en este caso compatible con arquitecturas x86-64. El interfaceo de estas dos capas se hace posible gracias a la **call convention de c**. La subrutina se encarga de realizar la multiplicación de los valores en el stack frame que fueron traidos como parametros desde las capas superiores y permite obtener el precio de una criptomoneda expresado en unidades de alguna de las monedas fiduciarias. 

```
segment .text
        global  mul
    mul:
        push	rbp
        mov	    rbp, rsp
        push	rbx
        
	    mulss	xmm0, xmm1

        pop		rbx		
        mov		rsp, rbp
        pop		rbp            
        ret
```
<br>

Podemos intuir como se vería nuestra **stackframe** :

![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/stack.png)

Podemos observar como el ```stackframe``` esta conformado los dos parametros del tipo float (ocupando 4Bytes c/u) que ocupan las posiciones ```RBP+24``` (precio en USD de alguna criptomoneda) y ```RBP+16``` (precio de un USD expresado en unidades de alguna moneda fiduciaria). Finalmente se referencia la direccion de retorno```RBP+8``` y el valor original de registro RBP que fue stackeado y es apuntado por el mismo ```RBP```. <rb>

## GDB 32 bits

Para poder inspeccionar la pila usando gdb comenzamos por generar el archivo ejecutable mediante el Makefile, este considera para los comandos el flag -g para poder usar gdb.   

![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/gdb1.png)  
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/gdb2.png)  
A continuación colocamos un break en la función main que contiene el llamado a la función “convert”. Posteriormente a partir de allí comenzamos a inspeccionar los cambios ocurridos a medida que avanzamos en la ejecución mediante el comando “step”  
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/gdb3.png)  
Utilizamos el gdb dashboard para visualizar en cada paso el stack. Vemos que antes de ingresar a la función de assembler, es decir, todavía en el main del código de C, el stack contiene los siguientes registros con sus valores en “Registers”. El ebp tiene un valor de 0xffffcf38 y el esp, 0xffffcf30. Cuando se retorne de la función de assembler, deberían volver a tener estos mismos valores.  
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/gdb4.png)  
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/gdb5.png)  
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/gdb6.png)  
Cuando termina de ejecutarse la función de assembler, se retorna al main y el stack es el siguiente. Vemos que efectivamente se restauran los valores de esp y ebp que tenían antes de ingresar a la función. También vemos que el resultado de la multiplicación de los parámetros ha sido correctamente asignado a la variable conversión.   
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/gdb7.png)  
Por último tambien notamos como el eip cambia su base de `main` a `convert`, la cual es el nombre de la función en la que se encuentra el programa en este momento. A esta base, le suma un dsplazamiento con cada `stepi` puesto que el programa avanza y por lo tanto, también avanza la próxima instrucción a ejecutarse. Lo vemos claramente en la dirección de memoria: `0x56556234: convert`, `0x56556235: convert + 1` y así sucesivamente.  
![](https://github.com/francoriba/lab2_cripto_pricing/blob/x86-64-mejoras/img/stackframe.png)  
