# CEVAZ Placement Test - Documentación Técnica

## Descripción del Proyecto

Aplicación web de evaluación de nivelación oral de inglés que utiliza:

- **Speech Recognition API** de Google Chrome para transcripción automática
- **getUserMedia API** para captura de foto y video
- **Google Apps Script** como backend para almacenamiento en Google Sheets

**Tecnología:** HTML5 + CSS3 + Vanilla JavaScript (sin frameworks)

---

## Arquitectura

```carpeta
CEVAZ-Placement-Test-Photo-Version/
├── index.html              # Aplicación completa (frontend + lógica)
└── README_DEVELOPERS.md   # Este documento
```

### Flujo de la aplicación

```flujo
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌────────────┐
│ Setup       │───▶│ Test         │───▶│ Recording   │───▶│ Upload   │
│ (registro) │    │ (preguntas)  │    │ (voz)       │    │ (datos)  │
└─────────────┘    └──────────────┘    └─────────────┘    └────────────┘
```

---

## Configuración del entorno de desarrollo

### Requisitos

- VS Code (recomendado)
- Extensión "Live Server" para VS Code
- Google Chrome (para pruebas)

### Ejecución local

````bash
#  Con Live Server (VS Code)
1. Abre el proyecto en VS Code
2. Haz clic derecho en index.html
3. Selecciona "Open with Live Server"

**Importante:** La aplicación debe ejecutarse desde un servidor HTTP, no abriendo el archivo directamente (file://).

### Navegadores soportados

- Chrome (recomendado) - Soporta Web Speech API
- Edge - Soporta Web Speech API
- Safari - Soporte parcial
- Firefox - No soporta Web Speech API

---

## Estructura del Código

### Variables globales

```javascript
const SCRIPT_URL = "https://script.google.com/macros/s/..."; // Google Apps Script
const questions = [...]; // Array de 20 preguntas
let currentIdx = 0;     // Índice de pregunta actual
let results = [];        // Respuestas acumuladas
let stream = null;       // MediaStream de cámara
let studentPhoto = "";   // Foto codificada en base64
let recState = "idle";  // Estado del reconocimiento: "idle" | "starting" | "running" | "stopping"
let recognition = null; // Instancia de SpeechRecognition
````

### Funciones principales

| Función              | Propósito                                                     |
| -------------------- | ------------------------------------------------------------- |
| `initTest()`         | Inicializa el test, captura foto, cambia a vista de preguntas |
| `loadStep()`         | Carga la siguiente pregunta y resetea el micrófono            |
| `saveAndNext()`      | Guarda la respuesta actual y avanza                           |
| `finish()`           | Finaliza el test y prepara el envío                           |
| `upload()`           | Envía los datos al Google Apps Script                         |
| `buildRecognition()` | Crea una instancia limpia de SpeechRecognition                |
| `toggleMic()`        | Alterna el estado del micrófono                               |

### Estados del micrófono

```microfono
idle ──(presionar)───▶ starting ──(onstart)───▶ running ──(presionar)───▶ stopping ──(onend)─── idle
                                              │                       │
                                              └─────(silencio)────────┘
```

### División de pantallas

1. **#setup** - Formulario de registro (nombre, cédula, sede)
2. **#test** - Área de evaluación (pregunta, micrófono, siguiente)
3. **#report** - Confirmación de envío

---

## Solución de problemas técnicos

### Caja de debug

El archivo incluye una `debug-box` oculta que muestra logs en tiempo real:

```javascript
function dbg(msg) {
  console.log("[MIC]", msg);
  const box = document.getElementById("debug-box");
  if (box) box.innerHTML += msg + "<br>";
}
```

**Para activar:** Busca en index.html la línea 293:

```javascript
// box.style.display = "block";  // Descomenta para ver logs
```

### Errores comunes

| Error                 | Causa              | Solución                                                  |
| --------------------- | ------------------ | --------------------------------------------------------- |
| `not-allowed`         | Permiso denegado   | Configuración de Chrome → Privacidad → Permitir micrófono |
| `service-not-allowed` | Servicio bloqueado | Verificar permisos en chrome://settings/content           |
| `audio-capture`       | Micrófono en uso   | Cerrar otras apps que usen micrófono                      |
| `no-speech`           | Silencio detectado | Es normal, el sistema reinicia solo                       |
| `aborted`             | Interrupción       | Ocurre al cambiar de pregunta                             |

### Problemas conocidos

1. **SpeechRecognition se "atasca"** - La función `buildRecognition()` recrea la instancia para evitar esto
2. **Permisos no persisten** - Chrome puede pedir permisos en cada sesión
3. **CORS en desarrollo** - Google Apps Script permite requests desde cualquier origen

---

## Personalización

### Agregar/editar preguntas

Busca el array `questions` en index.html (líneas 197-278):

```javascript
const questions = [
  { lvl: 1, q: "What is your full name...?" },
  { lvl: 2, q: "Describe your typical morning..." },
  // Agrega más preguntas aquí...
];
```

**Formato:** `{ lvl: número, q: "pregunta en inglés" }`

### Agregar nuevas sedes

Busca el elemento `<select id="sede">` en index.html (líneas 133-138):

```html
<select id="sede">
  <option value="LAS MERCEDES">LAS MERCEDES</option>
  <option value="NUEVA SEDE">NUEVA SEDE</option>
</select>
```

### Cambiar el backend

Para usar tu propio Google Apps Script:

1. Crea un nuevo Google Apps Script en script.google.com
2. Despliega como Web App
3. Copia la URL en `index.html` línea 194-195:

```javascript
const SCRIPT_URL = "https://script.google.com/macros/s/TU_NUEVA_URL/exec";
```

### Modificar el diseño

El CSS está embebido en el `<head>` de index.html (líneas 6-117):

```css
:root {
  --blue: #004a99;
  --gold: #f4b400;
  --red: #d9534f;
}
```

---

## Integración con Google Apps Script

### Estructura del payload enviado

```javascript
{
  name: "Nombre completo",
  id: "Cédula",
  sede: "LAS MERCEDES",
  fecha: "DD/MM/AAAA",
  level: 15,
  transcript: "[Lvl 1] respuesta 1\n\n[Lvl 2] respuesta 2...",
  photo: "base64..."
}
```

### Google Apps Script de ejemplo

```javascript
function doPost(e) {
  const data = JSON.parse(e.postData.contents);

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Datos");
  sheet.appendRow([
    new Date(),
    data.name,
    data.id,
    data.sede,
    data.fecha,
    data.level,
    data.transcript,
    data.photo,
  ]);

  return ContentService.createTextOutput("OK");
}
```

---

## Despliegue

### GitHub Pages

1. Sube los archivos a un repositorio público
2. Ve a Settings → Pages
3. Selecciona la rama `main` como source
4. La URL será: `https://tu-usuario.github.io/repo/`

### Actualización

```bash
git add .
git commit -m "Actualización"
git push origin main
```

---

## Contribuir

### Para contribuir al proyecto

1. Haz fork del repositorio
2. Crea una rama feature: `git checkout -b tu-feature`
3. Realiza tus cambios
4. Envía un pull request

### Estándares de código

- Usa indentación con espacios (2 espacios)
- Comenta funciones complejas
- Mantén el debug-box para debugging
- Prueba en Chrome móvil

---

## Créditos

| Nombre               | Rol                          |
| -------------------- | ---------------------------- |
| **Ney Espina**       | Director General / Creador   |
| **Kenny Ruz**        | Desarrollo y Pruebas móviles |
| **Ernesto Bracho**   | Desarrollo                   |
| **José Luis García** | Despliegue y seguimiento     |

---

## Recursos

- [Web Speech API](https://developer.mozilla.org/en-US/docs/Web/API/SpeechRecognition)
- [getUserMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)
- [Google Apps Script](https://developers.google.com/apps-script)

---

_Para preguntas técnicas, contacta al equipo de desarrollo._
