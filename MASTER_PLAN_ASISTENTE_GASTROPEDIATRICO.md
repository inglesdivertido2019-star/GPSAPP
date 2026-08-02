# Master Plan: Asistente de Apoyo Clínico en Gastroenterología Pediátrica (Edge AI)

## 0. Nota crítica antes de empezar

**No es posible ni responsable prometer "99% de efectividad diagnóstica"** en ningún sistema clínico, humano o de IA. Ninguna guía médica, estudio o regulador (FDA, COFEPRIS, EMA) acepta esa métrica como objetivo de diseño, y presentarla así genera:
- Riesgo legal (publicidad engañosa de dispositivo médico).
- Riesgo de seguridad del paciente (exceso de confianza / "automation bias" del médico).
- Imposibilidad técnica real: la variabilidad clínica pediátrica (neonatos vs adolescentes, presentaciones atípicas, comorbilidades) hace que ningún sistema sea correcto el 99% de las veces en diagnóstico diferencial abierto.

Por eso este plan **redefine el objetivo** a algo igual de ambicioso pero correcto y alcanzable:

> Construir un **Sistema de Apoyo a la Decisión Clínica (CDSS)** especializado en gastroenterología pediátrica, que funcione como "segundo par de ojos" con memoria perfecta de guías clínicas, literatura y protocolos — para que el médico (nunca el sistema solo) tome la mejor decisión posible, con trazabilidad de la evidencia usada y detección temprana de señales de alarma ("red flags").

El sistema **nunca emite un diagnóstico final ni una prescripción autónoma**. Siempre:
1. Presenta diagnósticos diferenciales rankeados con la evidencia/guía que los soporta.
2. Señala qué datos faltan para reducir la incertidumbre.
3. Marca explícitamente banderas rojas de urgencia/derivación.
4. Deja la decisión y la firma clínica al médico tratante.

Con esa base honesta, el resto del plan sí puede ser tan ambicioso como se necesite.

---

## 1. Objetivos del sistema

| Objetivo | Métrica realista |
|---|---|
| Cobertura de conocimiento clínico | ≥95% de guías ESPGHAN/NASPGHAN/AAP/OMS relevantes indexadas y consultables |
| Sensibilidad para banderas rojas (ej. sangrado GI, obstrucción, falla de medro severa, sospecha de enfermedad celíaca/EII, abdomen agudo) | Priorizar **alta sensibilidad** (>95%) aceptando falsos positivos, nunca falsos negativos silenciosos |
| Concordancia con diagnóstico diferencial de experto (top-5) | Validado clínicamente, objetivo inicial 80-85%, mejorable con datos reales |
| Disponibilidad | Funcional 100% offline en hardware edge; sincroniza cuando hay red |
| Tiempo de respuesta | <5s para consulta simple en Raspberry Pi 5 / Jetson |
| Trazabilidad | 100% de recomendaciones citan fuente (guía, año, nivel de evidencia) |

---

## 2. Arquitectura general

```
┌─────────────────────────────────────────────────────────────┐
│  DISPOSITIVO EDGE (Raspberry Pi 5 / Jetson Orin Nano)         │
│                                                                 │
│  ┌───────────────┐   ┌─────────────────┐   ┌────────────────┐│
│  │ Interfaz       │   │ Motor de         │   │ Base de        ││
│  │ Clínica (UI)   │◄─►│ Razonamiento     │◄─►│ Conocimiento   ││
│  │ - Anamnesis    │   │ - LLM local      │   │ Local          ││
│  │ - Percentiles  │   │   (cuantizado)   │   │ - Vector DB    ││
│  │ - Dosis pedi.  │   │ - RAG            │   │ - Guías        ││
│  │ - Red flags    │   │ - Reglas clínicas│   │ - Fármacos     ││
│  └───────────────┘   └─────────────────┘   └────────────────┘│
│           │                                        ▲            │
│           ▼                                        │            │
│  ┌───────────────────────────────────────────────┐│            │
│  │ Registro local cifrado (historia, consultas)   ││            │
│  └───────────────────────────────────────────────┘│            │
└──────────────────────────┬──────────────────────────────────┘
                            │ Sincronización oportunista (cuando hay red)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  NUBE / SERVIDOR CENTRAL (opcional, no obligatorio)            │
│  - Actualización de guías y modelo                             │
│  - Reentrenamiento con casos anonimizados (con consentimiento) │
│  - Modelo grande para consultas complejas (fallback online)    │
│  - Panel de auditoría y control de calidad                     │
└─────────────────────────────────────────────────────────────┘
```

Principio de diseño: **"offline-first"**. Todo lo esencial vive y funciona en el dispositivo. La nube es un *enhancer* (modelo más grande, actualizaciones), nunca una dependencia dura.

---

## 3. Hardware recomendado

| Nivel | Dispositivo | Uso |
|---|---|---|
| Mínimo viable | Raspberry Pi 5 (8-16GB RAM) + SSD NVMe (via hat) | Modelos LLM pequeños cuantizados (3-8B params, GGUF Q4), RAG con base vectorial local |
| Recomendado | NVIDIA Jetson Orin Nano Super (8GB) | Inferencia LLM acelerada por GPU, mejor latencia, modelos hasta 13B cuantizados |
| Óptimo clínica/hospital | Mini-PC x86 con GPU discreta (ej. RTX 4060) o servidor local | Modelos 13-30B, múltiples consultorios, mayor precisión |

Componentes adicionales:
- Pantalla táctil 7-10" (uso en consultorio).
- Batería/UPS (continuidad en cortes de energía — relevante en zonas rurales).
- Módulo 4G/WiFi opcional para sincronización.
- Cifrado de disco completo (LUKS) por ser datos de salud.

---

## 4. Stack de software

### 4.1 Motor de IA
- **Modelo base**: LLM open-weight cuantizado (familia Llama, Qwen, o Mistral médico-afinado) ejecutado con `llama.cpp` / `Ollama` para edge.
- **Fine-tuning / adaptación**: LoRA sobre corpus de gastroenterología pediátrica (no reentrenar el modelo completo — costoso e innecesario).
- **RAG (Retrieval-Augmented Generation)**: obligatorio. El modelo **nunca responde de memoria** en temas clínicos críticos — siempre recupera el fragmento de guía/fuente y lo cita.
- **Base vectorial local**: `Chroma` o `sqlite-vss` (ligero, corre bien en Raspberry Pi).
- **Motor de reglas clínicas determinístico** (no-LLM) para lo que NO debe depender de un modelo probabilístico:
  - Cálculo de dosis pediátricas por peso/superficie corporal.
  - Percentiles de crecimiento (curvas OMS/CDC).
  - Detección de banderas rojas (checklist estructurado, no generativo).
  - Interacciones medicamentosas.
  
  - **Interpretación de resultados de laboratorio** (ver 5.4): comparación contra rangos de referencia pediátricos por edad, no generación libre de "valores normales" por el LLM.

  Esto es clave: **todo lo que tenga una respuesta matemática/determinística no debe pasar por el LLM**, solo por lógica programada y verificada. El LLM se usa para razonamiento diferencial, síntesis de literatura y lenguaje natural, no para aritmética clínica.

### 4.2 Backend
- Python (FastAPI) corriendo local en el dispositivo.
- Base de datos local: SQLite (historia clínica, logs, cache).
- Contenedores ligeros (Docker/Podman) para reproducibilidad del despliegue.

### 4.3 Frontend
- Web app local (mismo patrón que ya usa este repositorio: HTML/JS servible offline vía navegador local) o app táctil dedicada (Electron/PWA instalable).
- Diseño pensado para uso rápido en consulta: formularios estructurados + chat en lenguaje natural como complemento, no como único input.

---

## 5. Base de conocimiento (el corazón del sistema)

### 5.1 Fuentes primarias a indexar
- **Guías clínicas**: ESPGHAN, NASPGHAN, AAP (Sección de Gastroenterología), OMS (nutrición y crecimiento infantil), guías nacionales del país donde se use.
- **Curvas de crecimiento**: OMS 0-5 años, CDC 2-20 años.
- **Formularios pediátricos de dosificación**: Lexicomp Pediatric, BNF for Children (si hay licencia) o fuentes abiertas equivalentes.
- **Bases de datos de interacciones y contraindicaciones**.
- **Literatura**: revisiones sistemáticas y metaanálisis relevantes (PubMed/Cochrane), priorizando evidencia nivel I-II.
- **Casos clínicos anonimizados** para calibrar el razonamiento diferencial (con supervisión de gastroenterólogos pediatras reales).
- **Protocolos de urgencia/derivación** (cuándo referir a hospital, cuándo es emergencia).

### 5.2 Áreas clínicas a cubrir (alcance clínico)
- Reflujo gastroesofágico y ERGE.
- Alergias alimentarias / APLV (alergia a proteína de leche de vaca).
- Enfermedad celíaca.
- Enfermedad inflamatoria intestinal (Crohn, colitis ulcerosa) pediátrica.
- Estreñimiento funcional y encopresis.
- Diarrea aguda/crónica, síndromes de malabsorción.
- Falla de medro / desnutrición.
- Dolor abdominal funcional / síndrome de intestino irritable pediátrico.
- Hepatología pediátrica básica (ictericia, hepatitis).
- Trastornos de motilidad, vómitos cíclicos.
- Nutrición enteral/parenteral pediátrica.
- Banderas rojas quirúrgicas (invaginación, apendicitis, obstrucción, sangrado GI alto/bajo).

### 5.3 Módulo de interpretación de laboratorios (si el paciente los aporta)

Los laboratorios son opcionales en cada consulta (no todo paciente llega con estudios), pero cuando existen deben integrarse al razonamiento, no ignorarse. Diseño:

- **Entrada**: captura estructurada (no OCR libre de imagen sin validación) — formulario con campos por analito, o carga de PDF/imagen con extracción asistida que el médico **confirma manualmente** antes de que el valor se use clínicamente.
- **Motor determinístico de interpretación**: cada analito se compara contra tablas de **rangos de referencia pediátricos por edad y sexo** (los rangos normales en niños cambian mucho entre neonato, lactante, escolar y adolescente — no son los mismos que en adulto). El motor solo clasifica: normal / anormal / crítico, no diagnostica.
- **Analitos prioritarios para gastroenterología pediátrica**:
  - Hematología: hemograma completo, VSG, PCR (marcadores inflamatorios/infecciosos).
  - Serología celiaca: anti-transglutaminasa IgA (tTG-IgA) + IgA total (para descartar déficit de IgA que invalida el resultado).
  - Calprotectina fecal (marcador clave de inflamación intestinal, distingue orgánico de funcional).
  - Panel hepático: AST, ALT, GGT, fosfatasa alcalina, bilirrubina total/directa, albúmina.
  - Electrolitos y función renal (relevante en diarrea/deshidratación).
  - Coproparasitoscópico, sangre oculta en heces, coprocultivo.
  - Antígeno de H. pylori en heces / prueba de aliento.
  - Panel nutricional: hierro/ferritina, vitamina B12, folato, vitamina D, zinc (relevante en falla de medro/malabsorción).
  - Pruebas de función pancreática (elastasa fecal) si aplica.
- **Salida hacia el motor de razonamiento**: el módulo no interpreta libremente; produce una lista estructurada de "hallazgos anormales con magnitud y contexto de edad" (ej. `calprotectina_fecal: 450 µg/g [alto; referencia <50 µg/g en >4 años]`). Esa lista estructurada es lo que se inyecta como contexto al RAG+LLM para que el diagnóstico diferencial la use y la cite, igual que citaría una guía.
- **Bandera roja automática**: valores críticos (ej. hemoglobina muy baja, PCR muy elevada, transaminasas muy alteradas) disparan alerta inmediata igual que el checklist clínico de banderas rojas — no esperan al razonamiento del LLM.
- **Nunca inventar un valor faltante**: si un lab no fue aportado, el sistema debe decir explícitamente "no disponible" y, si es relevante para el diferencial, sugerir solicitarlo — nunca asumir un valor.

### 5.4 Pipeline de ingestión de conocimiento
1. Curación por gastroenterólogo pediatra (obligatorio, no automatizable).
2. Conversión a texto estructurado + metadatos (fuente, año, nivel de evidencia, población aplicable — ej. "solo lactantes 0-6 meses").
3. Chunking semántico + generación de embeddings.
4. Indexado en vector DB local.
5. Versionado: cada actualización de guía reemplaza la anterior con changelog visible.
6. Revisión periódica (mínimo anual, o inmediata si una guía mayor se actualiza).

---

## 6. Flujo clínico del asistente (cómo se usa en consulta)

1. **Captura de datos estructurados**: edad, peso, talla, percentiles automáticos, motivo de consulta, antecedentes, síntomas guiados por checklist (no solo texto libre — reduce omisiones), **y resultados de laboratorio si el paciente los trae** (ver 5.3) — opcional, el flujo continúa igual si no hay.
2. **Triage automático de banderas rojas** (motor de reglas, no LLM): combina signos/síntomas clínicos Y valores críticos de laboratorio; si hay alguno, se muestra alerta inmediata y prioridad de derivación urgente, ANTES de cualquier razonamiento diferencial.
3. **Generación de diagnóstico diferencial**: el LLM+RAG propone una lista rankeada de posibilidades usando síntomas + antecedentes + hallazgos de laboratorio estructurados (cuando existen), con:
   - Probabilidad relativa (cualitativa: alta/media/baja, no un falso "% de certeza").
   - Evidencia/guía que sustenta cada opción.
   - Qué estudio o dato adicional (incluyendo qué laboratorio pedir) ayudaría a discriminar entre las opciones.
4. **Sugerencias de manejo**: basadas en guía, con dosis calculadas por el motor determinístico (nunca generadas por el LLM en texto libre).
5. **Registro y trazabilidad**: toda sesión queda registrada localmente (cifrada) para auditoría y aprendizaje continuo, con consentimiento informado.
6. **El médico decide y firma**: el sistema nunca cierra el caso ni prescribe sin la validación humana explícita.

---

## 7. Modo offline vs online

| Función | Offline | Online (cuando hay red) |
|---|---|---|
| Consulta de guías indexadas | ✅ Completo | ✅ |
| Diagnóstico diferencial (RAG + LLM local) | ✅ Completo | ✅ (opcionalmente modelo más grande en la nube) |
| Cálculo de dosis/percentiles | ✅ Completo (determinístico) | ✅ |
| Actualización de guías/modelo | ❌ (se acumulan cambios pendientes) | ✅ Descarga delta |
| Backup/sincronización de historia clínica | ❌ (queda local cifrado) | ✅ Sync cifrado a servidor si está autorizado |
| Consulta a modelo grande/experto remoto para casos complejos | ❌ | ✅ Opcional, con consentimiento y anonimización |

---

## 8. Seguridad, privacidad y cumplimiento normativo

Esto es innegociable por tratarse de datos de salud pediátrica:
- Cifrado en reposo (disco completo + base de datos) y en tránsito (TLS) para toda sincronización.
- Los datos de pacientes **no salen del dispositivo por defecto**.
- Si se sincroniza a nube: anonimización/seudonimización antes de salir, consentimiento explícito de tutores.
- Cumplimiento con normativa aplicable (ej. NOM-024/NOM-004 en México para expedientes clínicos electrónicos, COFEPRIS si se comercializa como dispositivo médico / software as a medical device, HIPAA si aplica en EE.UU., GDPR si aplica en UE).
- **Clasificación regulatoria**: un sistema que da recomendaciones diagnósticas a un profesional (no al paciente directamente) suele clasificar como *Clinical Decision Support Software* — investigar el marco regulatorio exacto del país objetivo ANTES de desplegar en producción clínica real. Esto puede requerir registro sanitario.
- Control de acceso por usuario (login del médico, auditoría de quién consultó qué).
- Logs de todas las recomendaciones emitidas, para trazabilidad médico-legal.

---

## 9. Validación clínica (la parte que hace o rompe el proyecto)

Un sistema como este **no se puede lanzar sin validación clínica real**. Plan sugerido:

1. **Fase de banco de pruebas**: correr el sistema contra 200-500 casos clínicos reales (anonimizados) con diagnóstico confirmado, comparar contra el gold standard.
2. **Validación por pares**: 3-5 gastroenterólogos pediatras revisan ciegos las recomendaciones del sistema vs. las suyas propias, se mide concordancia y se identifican fallos sistemáticos.
3. **Piloto controlado**: uso supervisado en 1-2 consultorios reales, con el médico siempre validando antes de actuar, midiendo tiempo ahorrado, casos donde el sistema añadió valor (detectó algo no considerado) y casos donde erró.
4. **Comité de revisión de errores**: cada vez que el sistema se equivoca o el médico lo contradice, ese caso se documenta y retroalimenta la base de conocimiento.
5. **Publicación/transparencia**: idealmente documentar la metodología de validación de forma que sea auditable (aunque no se publique formalmente).

Sin este proceso, el sistema es un prototipo interesante, no una herramienta clínica confiable.

---

## 10. Roadmap por fases

### Fase 0 — Fundación (4-6 semanas)
- Definir alcance clínico exacto y reclutar al menos 1 gastroenterólogo pediatra como asesor/validador permanente.
- Seleccionar y adquirir hardware de prueba (Raspberry Pi 5 u equivalente).
- Definir arquitectura técnica definitiva y stack.

### Fase 1 — Base de conocimiento (6-10 semanas)
- Curación y carga de guías clínicas principales.
- Construcción del pipeline RAG y vector DB local.
- Motor de reglas determinístico (dosis, percentiles, red flags).

### Fase 2 — MVP funcional (8-12 semanas)
- LLM local + RAG integrado.
- Módulo de interpretación de laboratorios (rangos de referencia pediátricos + clasificación normal/anormal/crítico).
- UI de consulta (anamnesis estructurada + chat).
- Registro local cifrado.
- Despliegue funcional en Raspberry Pi/Jetson.

### Fase 3 — Validación clínica (8-16 semanas, en paralelo con iteración)
- Pruebas con casos reales anonimizados.
- Ajuste de prompts, RAG y reglas según hallazgos.
- Medición de métricas de sensibilidad en red flags (prioridad #1).

### Fase 4 — Piloto supervisado (2-3 meses)
- Uso real en consultorio(s) con supervisión médica total.
- Recolección de feedback estructurado.
- Revisión regulatoria (determinar si requiere registro sanitario en la jurisdicción objetivo).

### Fase 5 — Escalamiento
- Empaquetado para despliegue en múltiples dispositivos.
- Mecanismo de actualización de guías/modelo (OTA cuando hay red).
- Panel de administración/auditoría central (opcional).

---

## 11. Equipo necesario

| Rol | Función |
|---|---|
| Gastroenterólogo pediatra (asesor clínico principal) | Define alcance, valida diagnósticos, cura conocimiento, participa en validación — **imprescindible, no opcional** |
| Ingeniero de ML/LLM | Arquitectura RAG, fine-tuning, cuantización, optimización edge |
| Ingeniero de software (backend/embedded) | Backend, despliegue en Raspberry Pi/Jetson, cifrado, sincronización |
| Diseñador UX clínico | Flujo de anamnesis y presentación de resultados, minimizar carga cognitiva en consulta |
| Especialista en regulación/cumplimiento | Navegar el marco legal de software médico en la jurisdicción objetivo |
| (Opcional) Segundo/tercer pediatra revisor | Para validación por pares, reduce sesgo de un solo experto |

---

## 12. Riesgos principales y mitigación

| Riesgo | Mitigación |
|---|---|
| Falso negativo en bandera roja (ej. no detectar abdomen agudo) | Motor de reglas determinístico independiente del LLM, sesgado deliberadamente hacia sobre-alertar |
| Alucinación del LLM en dosis/datos clínicos | RAG obligatorio + motor determinístico para todo lo numérico; el LLM nunca "inventa" una dosis |
| Exceso de confianza del médico en el sistema ("automation bias") | UI que siempre exige justificación/evidencia visible, nunca solo "diagnóstico: X" sin fuente |
| Desactualización de guías en dispositivos offline por meses | Indicador visible de "última actualización" y advertencia si supera X meses sin sync |
| Uso fuera de alcance (ej. neonatología crítica, cirugía) | Límites de alcance clínico explícitos y hard-coded, el sistema se abstiene y deriva |
| Responsabilidad legal | Contrato/disclaimer claro: herramienta de apoyo, la decisión y responsabilidad son del médico tratante |

---

## 13. Siguiente paso concreto

El punto de mayor apalancamiento ahora mismo no es escribir código, es **conseguir al gastroenterólogo pediatra asesor** — sin esa persona validando el alcance clínico y el conocimiento base, cualquier construcción técnica es prematura y potencialmente peligrosa.

En paralelo, técnicamente se puede empezar ya con: (1) definir el stack RAG + LLM local en Raspberry Pi/Jetson como prueba de concepto, y (2) construir el motor determinístico de percentiles/dosis pediátricas, que no depende de validación clínica externa porque usa fórmulas y tablas ya estandarizadas (OMS/CDC).
