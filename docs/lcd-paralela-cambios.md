# Corrección LCD: de I2C a paralela HD44780

**Archivo modificado:** `firmware/econodo_esp32/econodo_esp32.ino`  
**Fecha:** 2026-06-04

---

## Contexto

La integración anterior asumía incorrectamente que la pantalla era una LCD con módulo I2C/backpack (PCF8574). La pantalla real es una **LCD 1602A paralela (HD44780)** con potenciómetro B100K para contraste. Se corrigió la integración completa.

---

## Qué se eliminó

| Elemento eliminado | Motivo |
|---|---|
| `#include <Wire.h>` | Solo necesario para I2C |
| `#include <LiquidCrystal_I2C.h>` | Librería incorrecta para pantalla paralela |
| `LiquidCrystal_I2C lcd(LCD_I2C_ADDR, LCD_COLS, LCD_ROWS)` | Objeto incorrecto |
| `#define LCD_I2C_ADDR 0x27` | Dirección I2C, no aplica |
| `#define PIN_LCD_SDA 21` | Pin I2C, no aplica |
| `#define PIN_LCD_SCL 22` | Pin I2C, no aplica |
| `Wire.begin(PIN_LCD_SDA, PIN_LCD_SCL)` | Inicialización I2C en setup() |
| `lcd.init()` | Método exclusivo de LiquidCrystal_I2C |
| `lcd.backlight()` | Método exclusivo de LiquidCrystal_I2C |
| Todos los bloques `#if USAR_LCD` / `#endif` | Reemplazados por `if (!USAR_LCD) return;` dentro de funciones |
| `#define USAR_LCD true` | Reemplazado por `const bool USAR_LCD = true` |

---

## Qué se agregó

### Librería

```cpp
#include <LiquidCrystal.h>
```

Incluida en Arduino IDE por defecto. **No requiere instalación adicional.**

---

### Constantes de configuración

```cpp
const bool    USAR_LCD = true;
const uint8_t LCD_COLS = 16;
const uint8_t LCD_ROWS = 2;
const uint8_t LCD_RS   = 13;
const uint8_t LCD_E    = 14;
const uint8_t LCD_D4   = 25;
const uint8_t LCD_D5   = 26;
const uint8_t LCD_D6   = 27;
const uint8_t LCD_D7   = 32;
```

---

### Objeto LCD

```cpp
LiquidCrystal lcd(LCD_RS, LCD_E, LCD_D4, LCD_D5, LCD_D6, LCD_D7);
```

---

### Variables globales nuevas

```cpp
bool bmeListo        = false;  // true tras primera lectura válida del BME680
bool sdsListo        = false;  // true tras primera lectura válida del SDS011
int  ultimoCodigoHTTP = 0;     // último código HTTP recibido (0 = sin envío aún)

int           paginaLCD       = 0;
unsigned long ultimoCambioLCD = 0;
const unsigned long INTERVALO_LCD_MS = 3000;  // rotar página cada 3 segundos
```

---

### Funciones nuevas

#### `inicializarLCD()`
Llama a `lcd.begin(16, 2)` y `lcd.clear()`. Imprime pines en Serial Monitor.

#### `mostrarLCDInicio(const char* linea1, const char* linea2)`
Muestra dos líneas fijas durante el arranque del sistema.

#### `imprimirLCDLinea(uint8_t fila, const String& texto)`
Imprime texto en la fila indicada y rellena el resto con espacios para borrar caracteres residuales. Evita usar `lcd.clear()` en operación normal (evita parpadeo).

#### `limpiarLineaLCD(uint8_t fila)`
Rellena una fila completa con espacios.

#### `actualizarLCD()`
Llamada desde `loop()`. No bloqueante — usa `millis()`. Rota entre 4 páginas cada 3 segundos. Solo actúa si pasó `INTERVALO_LCD_MS` desde el último cambio.

---

### Secuencia de arranque en la pantalla

```
EcoNodo          →  EcoNodo          →  EcoNodo          →  Sistema listo
Iniciando            WiFi...              WiFi OK              ECONODO_001
```

---

### Páginas de operación (rotan cada 3 segundos)

```
Página 0          Página 1          Página 2          Página 3
EcoNodo           PM2.5: 3.4        Pres:1010hPa      WiFi OK
T:30.6C H:31%     PM10:  17.7       Aire: Bueno       HTTP: 201
```

Si aún no hay lectura válida de un sensor, se muestra `--` en lugar del valor.

---

## Conexión física de la pantalla

| Pin LCD | Conectar a | GPIO ESP32 |
|---------|-----------|-----------|
| GND / VSS | GND | — |
| VDD | 5V | — |
| VO | Pin central del potenciómetro B100K | — |
| RS | GPIO 13 | 13 |
| RW | GND directo | — |
| E | GPIO 14 | 14 |
| D0 – D3 | Sin conectar | — |
| D4 | GPIO 25 | 25 |
| D5 | GPIO 26 | 26 |
| D6 | GPIO 27 | 27 |
| D7 | GPIO 32 | 32 |
| A / LED+ | 5V (con resistencia 33–100 Ω recomendada) | — |
| K / LED- | GND | — |

**Potenciómetro B100K:**
- Un extremo → 5V
- Otro extremo → GND
- Pin central → LCD VO

---

## Cambios en setup() y loop()

### Antes (setup)
```cpp
#if USAR_LCD
  Wire.begin(PIN_LCD_SDA, PIN_LCD_SCL);
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("EcoNodo");
  ...
#endif
```

### Después (setup)
```cpp
inicializarLCD();
mostrarLCDInicio("EcoNodo", "Iniciando");
// ... WiFi ...
mostrarLCDInicio("EcoNodo", "WiFi OK");
// ... BME680 ...
mostrarLCDInicio("Sistema listo", NODO_ID);
```

### Antes (loop)
```cpp
#if USAR_LCD
  unsigned long ahoraLCD = millis();
  if (ahoraLCD - ultimoCambioLCD >= INTERVALO_LCD_MS) {
    ...
  }
#endif
```

### Después (loop)
```cpp
actualizarLCD();
```

---

## Qué NO se modificó

- Pines del BME680 (GPIO 5, 18, 19, 23)
- Pines del SDS011 (GPIO 16, 17)
- Lógica de envío a Supabase
- Credenciales en `secrets.h`
- Cálculos de calidad de aire y estado
- Lógica de reintento HTTP

---

## Diagnóstico rápido

| Síntoma | Causa probable | Solución |
|---------|---------------|---------|
| Pantalla no enciende (sin luz) | VDD o K/LED- mal conectado | Verificar 5V en VDD y GND en K |
| Luz encendida pero sin texto | Contraste incorrecto | Girar potenciómetro B100K en VO |
| Cuadros negros en pantalla | Contraste demasiado alto | Girar potenciómetro hacia el otro extremo |
| Texto ilegible / caracteres raros | Pines D4–D7 intercambiados | Verificar que D4→GPIO25, D5→26, D6→27, D7→32 |
| Pantalla congela en "Iniciando" | Wi-Fi no conecta | Verificar SSID y contraseña en secrets.h |
