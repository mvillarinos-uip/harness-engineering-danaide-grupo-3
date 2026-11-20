# RF-3105: Criterios de Aceptación con Metodología EARS

## 1. Información General de la Historia
* **Clave de Jira:** RF-3105
* **Título:** Endpoint de validación previa de calidad y detección de rostro sin persistencia
* **Proyecto:** Reconocimiento Facial (RF)
* **Tipo:** Historia
* **Estado:** Finalizada
* **Enfoque Técnico:** API REST, Motor Biométrico (FindFace), Arquitectura Stateless, Umbrales de Calidad de Imagen.

---

## 2. Definición y Propósito
Permitir que los sistemas integradores de capturas forenses y policiales verifiquen la aptitud biométrica de una fotografía de forma *stateless* (sin persistir la imagen en disco, sin registrar identidades en base de datos ni enrolar en listas de control), asegurando un umbral de calidad `DetectionScore ≥ 0.5` y la presencia de exactamente un rostro antes de su almacenamiento definitivo.

---

## 3. Marco Conceptual EARS (Easy Approach to Requirements Syntax)

| Patrón EARS | Palabra clave | Estructura base | Propósito |
| :--- | :--- | :--- | :--- |
| **Ubicuo (Ubiquitous)** | *(Sin palabra clave)* | *El `<sistema>` debe `<respuesta>`.* | Reglas, propiedades y restricciones permanentes del sistema. |
| **Disparado por evento (Event-driven)** | **CUANDO (WHEN)** | ***CUANDO*** `<evento/disparador>`, el `<sistema>` debe `<respuesta>`.* | Acciones reactivas directas ante un estímulo o solicitud entrante. |
| **Basado en estado (State-driven)** | **MIENTRAS (WHILE)** | ***MIENTRAS*** `<condición de estado>`, el `<sistema>` debe `<respuesta>`.* | Comportamiento activo únicamente durante un modo o estado específico. |
| **Comportamiento no deseado (Unwanted behavior)** | **SI ... ENTONCES (IF ... THEN)** | ***SI*** `<condición de error/excepción>`, ***ENTONCES*** el `<sistema>` debe `<respuesta>`.* | Manejo de excepciones, fallos, validaciones y casos límite. |
| **Complejo (Complex)** | **MIENTRAS / CUANDO / SI** | *Combinación de estados, eventos y excepciones.* | Escenarios multidimensionales o transaccionales condicionados. |

---

## 4. Criterios de Aceptación EARS

### A. Requisitos Ubicuos (Arquitectura y Seguridad)
* **CA-EARS-01 [Ubicuo - Naturaleza sin estado / Stateless]:**  
  El endpoint de validación previa debe procesar las imágenes de forma estrictamente sin estado (*stateless*), sin almacenar el archivo en volúmenes de disco, sin registrar entidades en la base de datos de personas, sin crear tarjetas en el motor biométrico (FindFace), sin asociarlas a listas de control ni emitir alertas de coincidencia.
* **CA-EARS-02 [Ubicuo - Autenticación requerida]:**  
  El endpoint de validación previa debe exigir autenticación válida mediante credenciales o tokens de usuario del sistema para cualquier solicitud entrante.

---

### B. Requisitos Disparados por Evento (Flujo Nominal / Happy Path)
* **CA-EARS-03 [Disparado por evento - Validación biométrica aprobada]:**  
  **CUANDO** el cliente externo envíe una petición `POST` con una imagen que contenga exactamente un (1) rostro y cuyo `DetectionScore` sea mayor o igual a 0.5 (umbral mínimo de calidad), el sistema debe responder con código HTTP 200 OK y confirmar que la fotografía es apta para reconocimiento facial (`isValid: true`).

---

### C. Requisitos de Comportamiento No Deseado (Manejo de Errores y Validaciones)
* **CA-EARS-04 [Comportamiento no deseado - Ausencia de rostro]:**  
  **SI** la imagen procesada por el motor biométrico no contiene ningún rostro detectable,  
  **ENTONCES** el endpoint debe responder indicando que la imagen no es apta (`isValid: false`), reportando el código de error `NO_FACE_DETECTED` y el mensaje explicativo: *"No se detectó ningún rostro en la imagen"*.

* **CA-EARS-05 [Comportamiento no deseado - Múltiples rostros]:**  
  **SI** el motor biométrico detecta dos (2) o más rostros en el encuadre analizado,  
  **ENTONCES** el endpoint debe rechazar la aptitud de la imagen (`isValid: false`), reportando el código de error `MULTIPLE_FACES_DETECTED` y el mensaje: *"La imagen contiene más de un rostro; debe incluir únicamente una persona"*.

* **CA-EARS-06 [Comportamiento no deseado - Puntuación de calidad insuficiente]:**  
  **SI** el motor biométrico detecta un único rostro pero su `DetectionScore` es estrictamente menor al umbral mínimo de 0.5,  
  **ENTONCES** el endpoint debe rechazar la imagen (`isValid: false`), reportando el código de error `LOW_DETECTION_SCORE` y el mensaje: *"La calidad de detección del rostro no alcanza el umbral mínimo requerido"*.

* **CA-EARS-07 [Comportamiento no deseado - Formato o archivo corrupto/inválido]:**  
  **SI** el archivo recibido no corresponde a un formato gráfico soportado (`JPEG` o `PNG`) o el archivo se encuentra corrupto,  
  **ENTONCES** el endpoint debe abortar el procesamiento sin invocar al motor biométrico, respondiendo con código HTTP 400 Bad Request y un mensaje de error indicando formato inválido.

---

## 5. Matriz de Trazabilidad (Gherkin Original vs. EARS)

| Escenario Original (Gherkin/BDD) | Criterio Adaptado (EARS) | Tipo de Patrón EARS |
| :--- | :--- | :--- |
| Validación exitosa (1 rostro, score ≥ 0.5) | **CA-EARS-03** | Disparado por evento (*Event-driven*) |
| Rechazo por ausencia de rostro | **CA-EARS-04** | Comportamiento no deseado (*Unwanted behavior*) |
| Rechazo por múltiples rostros | **CA-EARS-05** | Comportamiento no deseado (*Unwanted behavior*) |
| Rechazo por calidad insuficiente (< 0.5) | **CA-EARS-06** | Comportamiento no deseado (*Unwanted behavior*) |
| Garantía de no persistencia (Stateless) | **CA-EARS-01** | Ubicuo (*Ubiquitous*) |
| Archivo corrupto o formato no soportado | **CA-EARS-07** | Comportamiento no deseado (*Unwanted behavior*) |

---

## 6. Ventajas Técnicas y de QA
1. **Contrato de API inequívoco:** Diferencia claramente errores de validación de negocio (`NO_FACE_DETECTED`, `MULTIPLE_FACES_DETECTED`, `LOW_DETECTION_SCORE`) frente a errores de capa HTTP/transporte (`400 Bad Request`).
2. **Restricción no funcional explícita:** La naturaleza sin estado (*Stateless*) queda definida como un requisito permanente ubicuo aplicable transversalmente.
3. **Automatización directa:** La estructura *IF ... THEN* simplifica el diseño de assertions y suites automatizadas en Postman/Newman o frameworks de testing de backend.
