# Documento de Diseño Técnico (Design Doc): Calculadora-IVA

## 1. Visión General del Proyecto
**Calculadora-IVA** es una herramienta automatizada diseñada para extraer, procesar y analizar los acuses de declaraciones fiscales de IVA directamente desde el portal del Servicio de Administración Tributaria (SAT) en México. Su objetivo principal es calcular el saldo a favor actual del usuario, gestionar los acreditamientos históricos bajo una metodología FIFO (*First In, First Out*) y ofrecer una interfaz de centro de control para la toma de decisiones sobre acreditamientos futuros.

---

## 2. Alcance y Objetivos
- **Automatización de Web Scraping:** Autenticación y navegación automatizada mediante e.firma usando Selenium.
- **Procesamiento de Documentos:** Descarga masiva y parsing de archivos PDF correspondientes a los últimos 10 años de declaraciones.
- **Lógica Financiera FIFO:** Rastreo del saldo restante por cada declaración y recomendación de números de operación para cubrir montos a acreditar específicos.
- **Interfaz de Usuario (UI):** Panel de control tabular con un resumen ejecutivo de saldos.

---

## 3. Arquitectura del Sistema y Stack Tecnológico

El sistema se propone bajo una arquitectura monolítica modular en Python, facilitando la ejecución local por parte del contribuyente o despacho contable.

*   **Lenguaje:** Python 3.10+
*   **Automatización / Browser:** Selenium WebDriver (con soporte para Headless Chrome / undetected-chromedriver para evitar bloqueos por Cloudflare/Bot-detection del SAT).
*   **Procesamiento de PDF:** `pdfplumber` o `PyPDF2` / `PyMuPDF` para la extracción de texto estructurado de los acuses.
*   **Almacenamiento (Fase Inicial):** Archivos locales estáticos (`JSON`) y sistema de archivos plano para PDFs.
*   **Interfaz / Frontend:** Streamlit o Flask (para el centro de control tabular y visualización rápida).

---

## 4. Estructura de Directorios del Proyecto

```text
calculadora-iva/
│
├── credentials/          # Almacenamiento local seguro de la e.firma (.cer, .key)
├── files/                # Descarga de PDFs de acuses de declaración
├── data/
│   └── dump.json         # Almacenamiento persistente estructurado inicial
├── src/
│   ├── scraper.py        # Automatización con Selenium y e.firma
│   ├── parser.py         # Extracción de datos de los PDFs
│   ├── calculator.py     # Lógica contable y algoritmo FIFO
│   └── app.py            # Interfaz de usuario (Dashboard)
├── requirements.txt
└── README.md
```

---

## 5. Metodología y Flujo Técnico por Módulos

### Módulo 1: Autenticación y Extracción (Web Scraping)
1. **Credenciales:** El sistema lee los archivos `.cer` y `.key` ubicados en el directorio `credentials/`, junto con la contraseña de clave privada provista por variables de entorno o prompt seguro.
2. **Navegación:** Se conecta a `https://pstcdypisr.clouda.sat.gob.mx/`.
3. **Manejo de Sesión y Retries:** Debido a la inestabilidad de los portales de gobierno, el script incluirá bloques `try-except` con reintentos exponenciales y manejo de captchas si el flujo lo requiere (requiriendo intervención manual temporal si el portal implementa validación estricta).
4. **Descarga:** Los archivos PDF de los acuses de los últimos 10 años se descargan y guardan en el directorio `files/` con nomenclatura estandarizada (ej. `acuse_IVA_YYYY_MM.pdf`).

### Módulo 2: Procesamiento de Documentos (PDF Parsing)
El script escanea los PDFs en `files/` para extraer los campos clave mediante expresiones regulares (`regex`) y búsqueda de patrones de texto:
- **Número de Operación**
- **Fecha** (Mes y año de la declaración)
- **IVA a favor generado** en el mes
- **Acreditamiento de IVA** de periodos anteriores utilizado

Los datos extraídos se consolidan y vacían en `data/dump.json` con la siguiente estructura de ejemplo:
```json
[
  {
    "numero_operacion": "123456789",
    "periodo": "2025-05",
    "iva_generado": 15000.00,
    "acreditamiento_aplicado": 5000.00
  }
]
```

### Módulo 3: Lógica Financiera y Algoritmo FIFO
1. **Cálculo del Saldo Total:** Sumatoria de los saldos a favor generados menos los acreditamientos utilizados reportados en el periodo global.
2. **Método FIFO (First In, First Out):**
   - Las declaraciones con saldo a favor se ordenan cronológicamente de la más antigua a la más reciente.
   - Cuando el usuario ingresa un monto a acreditar deseado, el sistema recorre la lista ordenada consumiendo el saldo remanente completo de la declaración más antigua antes de pasar a la siguiente.
   - El sistema devuelve una propuesta de asignación detallando: `[Número de Operación, Fecha, Monto a Utilizar de esta declaración, Saldo resultante]`.

### Módulo 4: Centro de Control (Interfaz de Usuario)
- Se presenta un dashboard interactivo web (ej. usando Streamlit).
- **Parte Superior:** Tarjetas de resumen con:
  - Saldo Total a Favor Actualizado.
  - Total de declaraciones analizadas.
- **Sección Tabular:** Tabla interactiva con una fila por cada mes/declaración, mostrando columnas de Operación, Periodo, IVA Generado, Remanente y Estado.
- **Simulador de Acreditamiento:** Un campo de entrada numérica donde el usuario ingresa el monto que desea acreditar y el sistema despliega dinámicamente la lista de sugerencia de folios fiscales a utilizar bajo la regla FIFO.

---

## 6. Consideraciones de Seguridad y Privacidad
- **Datos Sensibles:** Las llaves privadas de la e.firma (`.key`) y certificados (`.cer`) **nunca** deben subirse a repositorios remotos (se debe configurar un `.gitignore` estricto para `credentials/` y `files/`).
- **Perspectiva de Base de Datos Futura:** Si el volumen de datos crece o se requiere concurrencia multi-usuario, el archivo estático `dump.json` migrará de manera transparente a una base de datos embebida como SQLite mediante un ORM ligero (como SQLAlchemy).