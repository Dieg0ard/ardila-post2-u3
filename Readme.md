# Laboratorio: Ensamblado y Ejecución Paso a Paso en DEBUG

**Unidad 3 — Post-Contenido 2 | Arquitectura de Computadores**

---

## Descripción del Laboratorio

En este laboratorio se ensamblan dos programas directamente en memoria usando el comando `A` del DEBUG de DOS, se ejecutan instrucción a instrucción con el comando `T`, y se registra el estado de los registros y banderas tras cada paso en una tabla de traza. La primera parte trabaja con un programa de suma de tres operandos; la segunda introduce la instrucción `LOOP` junto al registro `CX` para controlar un bucle iterativo.

---

## Comandos Utilizados

| Comando | Descripción |
|---------|-------------|
| `A 100` | Ensamblar instrucciones a partir de la dirección 0x0100 |
| `U 100 10E` | Desensamblar y verificar el código ensamblado |
| `R IP` | Leer o modificar el registro IP |
| `T` | Ejecutar una instrucción y mostrar el estado de los registros |
| `D CS:100` | Volcar el contenido de memoria en hexadecimal |

---

## Parte A — Programa de Suma con Traza Completa

### Código ensamblado

```asm
0725:0100  MOV AX, 000A   ; AX = 10
0725:0103  MOV BX, 0005   ; BX = 5
0725:0106  MOV CX, 0003   ; CX = 3
0725:0109  ADD AX, BX     ; AX = AX + BX = 15
0725:010B  ADD AX, CX     ; AX = AX + CX = 18
0725:010D  INT 20         ; Fin del programa
```

### Checkpoint 1 — Tabla de traza: Programa de suma

> Captura: `capturas/CP1_traza_suma.png`

| # | Instrucción ejecutada | AX     | BX     | CX     | IP siguiente | ZF | CF | SF |
|---|-----------------------|--------|--------|--------|--------------|----|----|----|
| 1 | `MOV AX, 000A`        | 000A   | 0000   | 0000   | 0103         | NZ | NC | PL |
| 2 | `MOV BX, 0005`        | 000A   | 0005   | 0000   | 0106         | NZ | NC | PL |
| 3 | `MOV CX, 0003`        | 000A   | 0005   | 0003   | 0109         | NZ | NC | PL |
| 4 | `ADD AX, BX`          | 000F   | 0005   | 0003   | 010B         | NZ | NC | PL |
| 5 | `ADD AX, CX`          | 0012   | 0005   | 0003   | 010D         | NZ | NC | PL |
| 6 | `INT 20`              | 0012   | 0005   | 0003   | —            | —  | —  | —  |

**Verificación:** AX = `0x0012` (18 decimal)

### Observaciones — Parte A

- Las instrucciones `MOV` no afectan las banderas; los valores de ZF, CF y SF permanecen igual que al inicio.
- Tras `ADD AX, BX`: AX pasa de `000A` a `000F` (10 + 5 = 15). La bandera `PE` (paridad par) se activa.
- Tras `ADD AX, CX`: AX pasa de `000F` a `0012` (15 + 3 = 18). La bandera `AC` (acarreo auxiliar) se activa porque hubo acarreo del nibble bajo.
- El resultado final es 18 decimal (`0x0012`), conforme al cálculo esperado: 10 + 5 + 3 = 18.

---

## Parte B — Programa con Bucle usando LOOP y CX

### Código ensamblado

```asm
0725:0100  MOV AX, 0000   ; Acumulador = 0
0725:0103  MOV CX, 0004   ; Contador de iteraciones = 4
0725:0106  ADD AX, +02    ; Suma 2 al acumulador ← inicio del bucle
0725:0109  LOOPW 0106     ; Decrementa CX, salta a 0106 si CX ≠ 0
0725:010B  INT 20         ; Fin (AX debe ser 0x0008 = 8)
```

> **Nota sobre la codificación:** `LOOP 0106` se codifica como `E2 FB`. El byte `E2` es el opcode de `LOOP` y `FB` es el desplazamiento relativo firmado (−5 decimal), calculado como `0x0106 − 0x010B = −5 = 0xFB`. Los saltos cortos del 8086 usan desplazamientos relativos de 8 bits con signo (rango ±127 bytes).

### Checkpoint 2 — Tabla de traza: Programa con bucle LOOP

> Capturas: `capturas/CP2_traza_loop.png` (dos partes)

| Iter. | Instrucción     | AX después | CX después | IP siguiente | ¿LOOP salta? |
|-------|-----------------|------------|------------|--------------|--------------|
| —     | `MOV AX, 0000`  | 0000       | 0000       | 0103         | —            |
| —     | `MOV CX, 0004`  | 0000       | 0004       | 0106         | —            |
| 1     | `ADD AX, +02`   | 0002       | 0004       | 0109         | —            |
| 1     | `LOOPW 0106`    | 0002       | 0003       | 0106         | Sí           |
| 2     | `ADD AX, +02`   | 0004       | 0003       | 0109         | —            |
| 2     | `LOOPW 0106`    | 0004       | 0002       | 0106         | Sí           |
| 3     | `ADD AX, +02`   | 0006       | 0002       | 0109         | —            |
| 3     | `LOOPW 0106`    | 0006       | 0001       | 0106         | Sí           |
| 4     | `ADD AX, +02`   | 0008       | 0001       | 0109         | —            |
| 4     | `LOOPW 0106`    | 0008       | 0000       | 010B         | No           |
| —     | `INT 20`        | 0008       | 0000       | —            | —            |

**Verificación:** AX = `0x0008` (8 decimal) cuando LOOP no salta (CX = `0x0000`)

### Observaciones — Parte B

- `LOOP` decrementa CX automáticamente antes de evaluar la condición de salto; no modifica las banderas.
- El bucle ejecuta exactamente 4 iteraciones (valor inicial de CX), acumulando 2 en AX por cada vuelta.
- Cuando CX llega a `0000`, LOOP no salta y la ejecución continúa en la instrucción siguiente (`INT 20`).
- El resultado 8 confirma la operación: 4 iteraciones × 2 = 8.

---

## Parte C — Análisis del Código Máquina con D

### Volcado de memoria del programa de bucle

```
-D CS:100 L0C
0725:0100  B8 00 00 B9 04 00 83 C0-02 E2 FB CD 20
           ^^^^^^^^ ^^^^^^^^ ^^^^^^^ ^^^^^ ^^^^^
           MOV AX,0 MOV CX,4 ADD AX,2 LOOP  INT 20
```

### Desglose de instrucciones

| Instrucción     | Bytes       | Tamaño | Descripción de la codificación |
|-----------------|-------------|--------|-------------------------------|
| `MOV AX, 0000`  | `B8 00 00`  | 3 B    | Opcode `B8` + inmediato de 16 bits en little-endian |
| `MOV CX, 0004`  | `B9 04 00`  | 3 B    | Opcode `B9` + inmediato de 16 bits en little-endian |
| `ADD AX, +02`   | `83 C0 02`  | 3 B    | Opcode `83` (ADD reg16, imm8-sign-ext) + ModRM `C0` + valor `02` |
| `LOOP 0106`     | `E2 FB`     | 2 B    | Opcode `E2` + desplazamiento relativo firmado `FB` (= −5) |
| `INT 20`        | `CD 20`     | 2 B    | Opcode `CD` (INT) + número de interrupción `20` |

**Total:** 13 bytes para un programa que realiza 4 sumas de forma iterativa.

### Observación sobre el formato little-endian

Los valores inmediatos de 16 bits se almacenan en memoria con el byte menos significativo primero. Por ejemplo, `MOV CX, 0004` se codifica como `B9 04 00`: el byte `04` va antes que `00`, aunque el valor lógico sea `0x0004`.

---

## Estructura del Repositorio

```
apellido-post2-u3/
├── README.md
└── capturas/
    ├── CP1_traza_suma.png
    ├── CP2_traza_loop.png (parte 1)
    └── CP2_traza_loop.png (parte 2)
```

---

*Laboratorio realizado en DOSBox con DEBUG — Arquitectura de Computadores 2026*
