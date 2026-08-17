# No_tengo_nombre_propio.github.io
SISTEMA SIGMA V2.5.


1. Visión General
Sistema de decodificación adaptativa que mantiene el equilibrio entre exploración (fluidez creativa) y
explotación (precisión lógica y coherencia) mediante control dinámico basado en métricas de incertidumbre
semántica, temperatura variable y mezcla de políticas. Diseñado para LLMs en producción con requisitos de
coherencia, estabilidad y latencia controlada.
Estado: Núcleo estable listo para implementación (v2.5 — señal de incertidumbre mejorada, control estabilizado y
degradación controlada).
2. Arquitectura del Sistema
El sistema se organiza en dos capas principales: un Orquestador de Memoria Persistente (OMP) opcional que
mantiene contexto de largo plazo, y el Módulo de Control (Sigma) que opera token a token regulando la fase de
generación.
Módulo de Control (Sigma)
1. Sonda de Incertidumbre → Semantic Entropy (n=3) + Degeneración n-gramas
2. Cálculo de λ(t) → Tipificación + Sigmoide + Filtro pasa-bajos + Rate limit
3. Decodificación Adaptativa → Contrastive Decoding (preferente) o Temperatura adaptativa
4. Síntesis de Logits → Mezcla/modulación según λ + normalización
5. Control Anti-Windup → Integrador saturado sobre µ_H
6. Detector de Degeneración → Protocolo de rescate cuando Deg > 0.30
3. Componentes Clave
3.1 Sonda de Incertidumbre
La sonda combina dos señales: Semantic Entropy (SE) y Degeneración (Deg).
Métrica Definición Propósito
Semantic Entropy (SE) SE = −∑c
 p
c
 · log(p
c
)
p
c
 = masa de probabilidad de cada clúster semántico de
las n=3 muestras
Mide incertidumbre semántica
real (diversidad de significado)
Degeneración (Deg) Deg(t) = (1/3) · ∑n=2..4 (repeticiones
n
 / total
n
) Detecta loops y colapso
léxico/semántico
SE se calcula muestreando n=3 continuaciones cortas y agrupándolas por equivalencia semántica (o similitud de embeddings).
Si Deg(t) > 0.30 se activa el protocolo de rescate.
3.2 Cálculo de λ(t)
λ(t) es el coeficiente de fase que determina el peso relativo entre el régimen exploratorio y el coherente.
z(t) = (SE(t) − µ
H
) / (σ
H
 + ε)
λ
raw
(t) = 1 / (1 + exp(−K · z(t)))
λ
suav
(t) = α · λ
raw
(t) + (1 − α) · λ
suav
(t−1)
λ(t) = clip(λ
suav
(t), λ(t−1) − ∆max, λ(t−1) + ∆max)
Interpretación: z ≈ +1 → λ ≈ 0.97 (exploratorio) · z ≈ −1 → λ ≈ 0.03 (coherente). El filtro pasa-bajos + rate limit evitan
oscilaciones bruscas entre tokens.
Parámetros por defecto: K = 3.5 · α = 0.30 · ∆max = 0.10 · µ
H
 y σ
H
 se estiman online (EMA).
3.3 Decodificación Adaptativa Sistema Sigma v2.5 Control de Fase Adaptativo
— 2 —
Modo preferente — Contrastive Decoding (CD): logits_final = logits_experto − α
cd · logits_amateur. La máscara
de contraste se aplica solo donde el amateur muestra baja confianza. λ modula la fuerza del contraste.
Modo fallback — Temperatura adaptativa: temperature = Tmin + λ · (T
max
− Tmin) con Tmin = 0.55 y T
max
 = 0.95.
Estrategia de ejecución: Si λ > 0.92 → solo exploratorio. Si λ < 0.08 → solo coherente. En zona intermedia se
aplica mezcla o contraste según λ.
3.4 Síntesis y Normalización de Logits
logit_final = mezcla_o_contraste(λ)
logit_centered = logit_final − mean(logit_final)
logit_normalized = logit_centered / (std(logit_final) + 1e−8)
Se utiliza estandarización completa para estabilidad numérica.
3.5 Control Anti-Windup
e(t) = 0.5 − λ(t). Actualización del integrador (solo tras varios tokens en zona de transición): µ
H
 += Ki
 · e(t), con
saturación en [µ
H_min, µ
H_max]. Ki
 = 0.01. Previene la deriva del punto de operación en regímenes estacionarios.
3.6 Detector de Degeneración y Protocolo de Rescate
Activación: Deg(t) > 0.30 (ventana de 16–24 tokens).
Acción inmediata: (1) Forzar λ(t) = 0.0 · (2) temperature = 0.50 durante 3–5 tokens · (3) Reset parcial de µ
H
 hacia
la media reciente.
La entropía semántica por sí sola no distingue siempre entre diversidad creativa útil y ruido degenerativo. Este mecanismo
actúa como última línea de defensa.
3.7 Orquestador de Memoria Persistente (OMP)
Componente opcional (fase 2). Mantiene historial en base vectorial, calcula coherencia temporal TH
 y proporciona
contexto de largo plazo. El núcleo de Sigma v2.5 funciona sin OMP. La memoria externa mejora el
comportamiento en conversaciones largas pero no es requisito de arranque.
4. Zona de Transición
Condición computable: zona_transicion(t) = (|∆λ(t)| > 0.05) ∧ (sistema no está en rescate). El sistema se considera
saludable cuando permanece una fracción significativa del tiempo en zona de transición controlada, sin
oscilaciones ni colapsos a los extremos.
5. Parámetros de Configuración
Parámetro Valor Descripción
n (muestras SE) 3 Número de continuaciones para Semantic Entropy
K 3.5 Ganancia de la sigmoide
α 0.30 Factor de suavizado de λ
∆max 0.10 Rate limit máximo por token
Deg_threshold 0.30 Umbral de activación de rescate
T_min / T_max 0.55 / 0.95 Temperaturas de los regímenes
α_cd 0.15–0.25 Fuerza base de Contrastive Decoding
K_i 0.01 Ganancia del integrador anti-windup
Ventana Deg 16–24 Tokens para cálculo de degeneración
6. Flujo de Inferencia (Token a Token) Sistema Sigma v2.5 Control de Fase Adaptativo
— 3 —
1. Obtener logits del modelo (contexto actual)
2. (Opcional) Aplicar Contrastive Decoding
3. Muestrear n=3 continuaciones y calcular Semantic Entropy (SE)
4. Calcular Deg(t)
5. Si Deg(t) > 0.30 → activar protocolo de rescate y saltar a muestreo
6. Calcular λ(t) con tipificación, sigmoide, filtro pasa-bajos y rate limit
7. Decidir régimen: λ > 0.92 → exploratorio · λ < 0.08 → coherente · resto → mezcla/contraste
8. Sintetizar y normalizar logits
9. Muestrear token
10. Actualizar µ
H
, σ
H
 y estado del anti-windup
11. (Opcional) Actualizar OMP
7. Advertencias Críticas
1. Semantic Entropy requiere muestreo adicional. Con n=3 el coste es aceptable; valores superiores deben justificarse con
mediciones de latencia.
2. Contrastive Decoding es el modo preferente cuando existe acceso a capas intermedias o a un modelo amateur. En APIs
black-box se utiliza el fallback de temperatura.
3. El rate limit (∆max = 0.10) es obligatorio para evitar oscilaciones. No desactivarlo.
4. La calibración online de µ
H
 y σ
H
 debe mantenerse activa. Valores fijos iniciales solo son válidos en los primeros tokens.
5. OMP mejora el comportamiento en diálogos largos pero no es necesario para el funcionamiento del núcleo.
8. Estado del Proyecto
Versión: 2.5 (Núcleo estable para producción)
Componentes completados: Sonda de incertidumbre basada en Semantic Entropy · Cálculo de λ con suavizado y rate limiting
· Contrastive Decoding (modo preferente) + fallback de temperatura · Ejecución condicional de regímenes · Normalización de
logits · Anti-windup con saturación · Detector de degeneración y protocolo de rescate · Definición de zona de transición.
Pendientes (no bloqueantes): Validación empírica exhaustiva y ablaciones · Calibración fina de parámetros en producción ·
Integración completa de OMP · Pruebas de carga multi-sesión.
9. Referencias Rápidas — Fórmulas Clave
Semantic Entropy: SE = −∑c
 p
c
 log(p
c
)
z(t) = (SE − µ
H
) / σ
H
λ
raw
 = 1 / (1 + exp(−K · z))
λ
suav
 = α·λ
raw
 + (1−α)·λ
prev
λ final = clip(λ
suav
, λ
prev
 ± ∆max)
Degeneración = (1/3)·∑n=2..4 (rep
n
 / total
n
)
Umbrales: Deg > 0.30 → Protocolo de rescate · λ > 0.92 → Solo exploratorio · λ < 0.08 → Solo coherente