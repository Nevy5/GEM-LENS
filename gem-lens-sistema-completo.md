# GEM LENS — Sistema Completo de Scoring, Base de Datos y Marco Legal
## Documento Técnico v2.0

---

# PARTE I — ALGORITMO DE SCORING

## 1.1 Arquitectura del Sistema de Puntuación Dual

Gem Lens implementa un sistema de **puntuación dual**, análogo al que usan las empresas de grading profesionales:

### Escala Primaria: GLS (Gem Lens Score) — 0 a 10
Equivalente visual a PSA/CGC. Es el número que el usuario ve grande en el informe.

### Escala Secundaria: GLP (Gem Lens Points) — 0 a 500
Escala granular interna que alimenta el GLS. Cada categoría de análisis aporta un máximo de puntos. La suma total determina el GLS.

```
GLS = f(GLP_total)

GLP 475–500  → GLS 10  (Gem Mint)
GLP 435–474  → GLS 9   (Mint)
GLP 380–434  → GLS 8   (Near Mint-Mint)
GLP 320–379  → GLS 7   (Near Mint)
GLP 260–319  → GLS 6   (Excellent-Mint)
GLP 200–259  → GLS 5   (Excellent)
GLP 150–199  → GLS 4   (Very Good-Excellent)
GLP 100–149  → GLS 3   (Very Good)
GLP  50–99   → GLS 2   (Good)
GLP   1–49   → GLS 1   (Poor)
GLP      0   → GLS 0   (Damaged / Auténtica inválida)
```

---

## 1.2 Las 5 Categorías de Análisis + Autenticidad

Cada categoría tiene un peso máximo en GLP y sub-categorías con micro-puntuaciones.

---

### CATEGORÍA 0 — AUTENTICIDAD (Gate de entrada)
**No suma puntos: es un filtro binario. Si falla, GLS = 0 automáticamente.**

La carta debe pasar TODOS los checks de autenticidad para que el scoring continúe.

#### Sub-análisis de autenticidad:

**A) Detección tipográfica**
- La IA compara las fuentes usadas en la carta contra una base de datos de tipografías oficiales por set y año.
- Pokémon usa tipografías específicas (Gill Sans modificada en cartas base, fuentes propias en sets modernos).
- Se detectan: espaciado incorrecto, kerning diferente, peso de trazo inconsistente, tamaño de texto fuera de especificación.
- Herramientas: análisis de contornos vectoriales sobre regiones de texto extraídas de la imagen.

**B) Verificación de patrón de impresión (rosette pattern)**
- Las cartas oficiales tienen un patrón de puntos de impresión offset (rosette) visible con macro-fotografía.
- Las falsificaciones suelen tener patrones inkjet o laser (puntos más grandes, irregulares).
- La IA analiza zonas de color sólido para detectar el patrón de micro-puntos.

**C) Verificación de código de set y número**
- El número de colección y símbolo del set se cruzan contra la base de datos de sets oficiales.
- Se detectan: cartas con números inexistentes, símbolos de rareza incorrectos para el set, combinaciones imposibles.

**D) Análisis de colores de referencia**
- Los colores de fondo de las cartas (amarillo Pokémon, gradientes de tipo, bordes de tipo) tienen valores RGB/HSV específicos por set.
- Desviaciones >15% en valores de referencia son señal de falsificación.

**E) Verificación de proporciones y dimensiones**
- Una carta Pokémon estándar mide 63mm × 88mm con tolerancia de ±0.5mm.
- La IA calcula las proporciones de la carta en la imagen y verifica la relación de aspecto.
- También verifica el grosor visual del borde negro (varía por era de impresión).

**F) Análisis del reverso**
- El reverso de las cartas Pokémon tiene un diseño específico con colores y proporciones exactas.
- La IA lo compara contra la plantilla de referencia del período correspondiente.

**Resultado:**
- AUTÉNTICA (confianza alta / media / baja)
- POSIBLE FALSIFICACIÓN — razón detallada
- FALSIFICACIÓN DETECTADA — GLS = 0, no se puntúa

---

### CATEGORÍA 1 — CENTERING (Centrado)
**Peso máximo: 120 GLP**

El centering mide qué tan centrada está la imagen impresa dentro del borde de la carta. PSA lo mide como ratio (ej: 55/45 horizontal, 60/40 vertical).

#### Cálculo:
```
Centering_H = ancho_borde_izq / ancho_borde_der  (o el inverso si > 1)
Centering_V = alto_borde_sup / alto_borde_inf

Normalizado: siempre el mayor sobre el menor (ej: 60/40 no 40/60)
```

#### Tabla de puntuación — Centering horizontal (60 GLP max):
```
≤ 52/48  →  60 GLP  (perfecto)
≤ 55/45  →  55 GLP
≤ 58/42  →  45 GLP
≤ 60/40  →  35 GLP
≤ 65/35  →  20 GLP
≤ 70/30  →  10 GLP
> 70/30  →   0 GLP
```

#### Tabla de puntuación — Centering vertical (60 GLP max):
```
≤ 52/48  →  60 GLP
≤ 55/45  →  55 GLP
≤ 58/42  →  45 GLP
≤ 60/40  →  35 GLP
≤ 65/35  →  20 GLP
≤ 70/30  →  10 GLP
> 70/30  →   0 GLP
```

**Nota técnica:** La IA detecta los 4 bordes usando detección de líneas (Hough Transform). En cartas con desgaste en bordes, se usa el promedio de múltiples mediciones a lo largo del borde para reducir el error.

---

### CATEGORÍA 2 — ESQUINAS (Corners)
**Peso máximo: 120 GLP (30 por esquina × 4)**

Las esquinas son el factor más crítico en grading real. Se evalúan las 4 esquinas individualmente.

#### Sub-análisis por esquina:

**Whitening (desgaste de color en punta):**
- Se extrae una región de 20×20px alrededor de cada esquina.
- Se mide el porcentaje de píxeles que se desvían hacia blanco respecto a la referencia de la carta.
- Thresholds: <2% (perfecto), 2-5% (leve), 5-15% (moderado), >15% (severo).

**Dobleces y quiebre:**
- Se analiza la presencia de líneas diagonales en zonas de esquina (indicio de doblez).
- Se detecta discontinuidad en el gradiente de color (cambio brusco = doblez).

**Chips (fragmentos faltantes):**
- Detección de zonas donde el píxel pasa directamente del color del borde al fondo de la foto (fragmento faltante).

#### Puntuación por esquina (30 GLP):
```
Sin defecto visible          →  30 GLP
Whitening mínimo (<2%)       →  26 GLP
Whitening leve (2-5%)        →  20 GLP
Whitening moderado (5-15%)   →  12 GLP
Doblez leve detectado        →  10 GLP
Whitening severo (>15%)      →   5 GLP
Doblez severo / chip         →   2 GLP
Esquina faltante o muy rota  →   0 GLP
```

---

### CATEGORÍA 3 — BORDES (Edges)
**Peso máximo: 80 GLP (20 por borde × 4)**

Los 4 bordes (superior, inferior, izquierdo, derecho) se analizan independientemente.

#### Sub-análisis por borde:

**Roughness (aspereza / microchips):**
- Se mide la irregularidad del perfil del borde comparando con una línea recta perfecta.
- Desviación estándar del perfil del borde como métrica.

**Whitening lineal:**
- Decoloración a lo largo del borde, especialmente en bordes negros/oscuros.

**Dents (abolladuras) y cuts (cortes):**
- Detección de indentaciones en el perfil del borde.

#### Puntuación por borde (20 GLP):
```
Borde recto, sin whitening    →  20 GLP
Roughness mínimo              →  17 GLP
Whitening leve                →  13 GLP
Roughness moderado            →   9 GLP
Whitening moderado            →   6 GLP
Roughness severo / cuts       →   3 GLP
Borde muy dañado              →   0 GLP
```

---

### CATEGORÍA 4 — SUPERFICIE (Surface)
**Peso máximo: 100 GLP**

La superficie incluye el frente (arte + texto) y el reverso.

#### Sub-análisis:

**A) Scratches en la capa holo (30 GLP):**
- Las cartas holográficas tienen una capa reflectante que acumula scratches finos.
- La IA detecta líneas finas de alta frecuencia en regiones holográficas.
- Thresholds: sin scratches (30), micro-scratches sólo visibles en ángulo (24), scratches leves (16), scratches moderados (8), scratches severos (0).

**B) Print lines / defectos de impresión (20 GLP):**
- Líneas de impresión son defectos de fábrica (no cuentan negativamente como desgaste pero sí en el score final porque afectan el aspecto visual).
- Se detectan como líneas horizontales de baja intensidad paralelas al eje de impresión.

**C) Stains / manchas (20 GLP):**
- Detección de regiones con desvío de color respecto a la referencia limpia de la carta.
- Diferenciación entre: manchas de tinta, manchas de humedad, oxidación.

**D) Dents / indentaciones en superficie (15 GLP):**
- Detección de zonas con cambio de iluminación consistente con una indentación física.
- Requiere iluminación lateral en la foto para ser detectable con mayor precisión.

**E) Estado del reverso (15 GLP):**
- Análisis de superficie aplicado al reverso con los mismos sub-criterios.
- El reverso pesa menos porque es menos visible pero importa para el grading.

---

### CATEGORÍA 5 — DESGASTE DE COLOR Y FRESCURA (Color & Print Quality)
**Peso máximo: 80 GLP**

Esta categoría es exclusiva de Gem Lens respecto al grading tradicional — mide la "vitalidad" visual de la carta.

**A) Saturación de colores (30 GLP):**
- Se compara la saturación/luminosidad de zonas de color de referencia contra una carta NM del mismo set.
- Una carta decolorada por sol o tiempo pierde saturación.
- Thresholds en desviación % respecto al valor de referencia.

**B) Nitidez del texto y bordes impresos (25 GLP):**
- Se mide la nitidez de los bordes de las letras y líneas del arte.
- Cartas muy jugadas o expuestas a humedad tienen texto menos nítido.

**C) Fading / amarillamiento (25 GLP):**
- Especialmente en cartas antiguas (Base Set, Jungle, Fossil).
- Detección de corrimiento hacia tonos amarillos/marrones en las zonas blancas.

---

## 1.3 Tabla Resumen de Pesos GLP

```
CATEGORÍA                     MAX GLP    % DEL TOTAL
─────────────────────────────────────────────────────
0. Autenticidad               Gate       —
1. Centering                  120        24%
2. Esquinas (4 × 30)          120        24%
3. Bordes (4 × 20)             80        16%
4. Superficie                 100        20%
5. Color & Frescura            80        16%
─────────────────────────────────────────────────────
TOTAL                         500       100%
```

---

## 1.4 Factores de Ajuste y Penalizaciones

Además del scoring base, se aplican factores de ajuste:

**Penalización por inconsistencia fotográfica:**
- Si la calidad de la foto es insuficiente para analizar una categoría, se asigna el puntaje mínimo de esa categoría más un flag "ANÁLISIS INCOMPLETO".
- Se recomienda al usuario subir mejor foto para esa categoría específica.

**Bonus por múltiples fotos:**
- 1 foto: análisis estándar, confianza base.
- 2 fotos (frente + reverso): +5% confianza general, análisis de reverso habilitado.
- 3 fotos: +10% confianza, análisis de bordes y esquinas mejorado.
- 4+ fotos (incluyendo macro de esquinas): +15% confianza, análisis de centering más preciso.

**Penalización por condiciones de foto:**
- Foto con flash directo: reduce confianza en superficie holo.
- Foto borrosa (blur detection): reduce confianza en esquinas y texto.
- Foto en ángulo >15°: reduce confianza en centering.
- Foto con dedos en los bordes: flag de alerta, requiere resubir.

---

## 1.5 Cómo Se Nutre la IA para Mejorar

### Fase 1 — Bootstrap (0 a 1.000 análisis)
- Dataset inicial: imágenes de cartas PSA/CGC ya gradeadas (scraping de eBay listings con nota).
- Se usa transfer learning sobre modelos de visión existentes (ResNet, EfficientNet, o vision transformers).
- Ground truth: la nota PSA real es el label de entrenamiento.
- Objetivo: que el modelo aprenda la correlación entre features visuales y puntuación.

### Fase 2 — Feedback loop comunitario (1.000 a 10.000 análisis)
- Cuando un usuario luego gradea su carta con PSA/CGC, puede ingresar la nota real.
- Esa comparación (Gem Lens estimó X, PSA dio Y) se usa para recalibrar el modelo.
- Sistema de recompensa: usuarios que reportan resultados reales obtienen créditos gratis.
- Esto crea un flywheel: más usuarios → más feedback → modelo más preciso → más usuarios.

### Fase 3 — Modelo propio (10.000+ análisis)
- Con suficientes datos etiquetados, se entrena un modelo fine-tuned específico para TCG.
- Se separan modelos por juego (modelo Pokémon, modelo Yu-Gi-Oh!, modelo MTG) ya que las características visuales difieren.
- Se mantiene un modelo general como fallback.

### Base de datos de referencia por carta:
- Para cada carta del catálogo, se almacenan vectores de referencia (color, tipografía, dimensiones).
- Cuando llega una nueva foto, se busca la carta más similar en la base y se compara contra su referencia NM.
- La base se nutre de: datos oficiales de Pokémon/Konami/Wizards, imágenes de Bulbapedia, Yugipedia, MTG Wiki (uso educativo/no comercial), y cartas subidas por usuarios con alta confianza de NM.

---

# PARTE II — BASE DE DATOS: ARQUITECTURA Y NUTRICIÓN

## 2.1 Esquema de Base de Datos

### Tabla: cards_catalog (catálogo maestro)
```sql
CREATE TABLE cards_catalog (
  id              UUID PRIMARY KEY,
  game            ENUM('pokemon','yugioh','mtg','dbs','op','fab'),
  card_name       VARCHAR(200),
  set_name        VARCHAR(200),
  set_code        VARCHAR(20),
  card_number     VARCHAR(20),
  variant         VARCHAR(100),  -- holo, reverse, full_art, etc.
  rarity          VARCHAR(50),
  year            INTEGER,
  language        CHAR(2),       -- EN, JP, ES, PT
  
  -- Datos de referencia visuales
  ref_colors_json   JSONB,       -- valores HSV de referencia por zona
  ref_typography    VARCHAR(100), -- nombre de fuente oficial
  ref_dimensions    JSONB,       -- ancho, alto, ratio, tolerancias
  ref_border_width  DECIMAL,    -- grosor del borde en mm
  
  -- Datos de mercado
  tcgplayer_id    INTEGER,
  cardmarket_id   INTEGER,
  
  -- Metadatos
  created_at      TIMESTAMP,
  updated_at      TIMESTAMP,
  source          VARCHAR(50)    -- 'official', 'community', 'scrape'
);
```

### Tabla: card_analyses (análisis individuales)
```sql
CREATE TABLE card_analyses (
  id              UUID PRIMARY KEY,
  user_id         UUID,          -- NULL si anónimo
  session_id      VARCHAR(64),   -- para anónimos
  card_catalog_id UUID REFERENCES cards_catalog(id),
  
  -- Imágenes
  photos_urls     TEXT[],        -- array de URLs en Cloudinary/S3
  photos_count    INTEGER,
  
  -- Scoring
  glp_centering    INTEGER,
  glp_corners_tl   INTEGER,      -- top-left
  glp_corners_tr   INTEGER,
  glp_corners_bl   INTEGER,
  glp_corners_br   INTEGER,
  glp_edges_top    INTEGER,
  glp_edges_bottom INTEGER,
  glp_edges_left   INTEGER,
  glp_edges_right  INTEGER,
  glp_surface      INTEGER,
  glp_color        INTEGER,
  glp_total        INTEGER,
  gls_score        DECIMAL(3,1), -- 0.0 a 10.0 (puede ser 7.5)
  
  -- Autenticidad
  auth_status      ENUM('authentic','suspicious','fake','unknown'),
  auth_confidence  DECIMAL(3,2),
  auth_flags       JSONB,        -- razones de sospecha si las hay
  
  -- Análisis detallado (para mostrar en UI)
  analysis_json    JSONB,        -- output completo de la IA
  
  -- Condición textual mapeada
  condition_text   ENUM('GEM_MINT','MINT','NM_MINT','NM','EX_MINT',
                        'EX','VG_EX','VG','GOOD','POOR','DAMAGED'),
  confidence_level ENUM('high','medium','low'),
  
  -- Precios al momento del análisis
  price_usd_nm      DECIMAL(10,2),
  price_usd_lp      DECIMAL(10,2),
  price_ars_estimated DECIMAL(12,2),
  dolar_blue_rate    DECIMAL(8,2),
  price_date         TIMESTAMP,
  
  -- Grading recommendation
  grading_rec_json   JSONB,
  
  -- Visibilidad comunitaria
  is_public          BOOLEAN DEFAULT FALSE,
  public_nickname    VARCHAR(50),
  watermark_photo    BOOLEAN DEFAULT TRUE,
  
  -- Feedback real (post-grading)
  real_psa_grade     DECIMAL(3,1),
  real_cgc_grade     DECIMAL(3,1),
  real_bgs_grade     DECIMAL(4,2),  -- BGS tiene decimales (9.5)
  feedback_date      TIMESTAMP,
  
  -- Control
  ai_model_version   VARCHAR(20),
  processing_time_ms INTEGER,
  created_at         TIMESTAMP DEFAULT NOW()
);
```

### Tabla: market_prices (precios históricos)
```sql
CREATE TABLE market_prices (
  id              UUID PRIMARY KEY,
  card_catalog_id UUID REFERENCES cards_catalog(id),
  condition       VARCHAR(10),   -- NM, LP, MP, HP
  
  source          ENUM('tcgplayer','ebay','cardmarket','cardtrader','local_arg'),
  price_usd       DECIMAL(10,2),
  price_ars       DECIMAL(12,2),
  dolar_blue      DECIMAL(8,2),
  
  sample_size     INTEGER,       -- cuántas ventas/listings se tomaron
  price_min       DECIMAL(10,2),
  price_max       DECIMAL(10,2),
  
  recorded_at     TIMESTAMP DEFAULT NOW()
);

-- Index para consultas de precio más reciente
CREATE INDEX idx_market_prices_card_date 
  ON market_prices(card_catalog_id, condition, recorded_at DESC);
```

### Tabla: users (cuenta opcional)
```sql
CREATE TABLE users (
  id              UUID PRIMARY KEY,
  nickname        VARCHAR(50) UNIQUE,
  email           VARCHAR(200) UNIQUE,
  
  -- Privacidad
  email_verified  BOOLEAN DEFAULT FALSE,
  data_consent    BOOLEAN DEFAULT FALSE,  -- CONSENTIMIENTO EXPLÍCITO
  consent_date    TIMESTAMP,
  consent_version VARCHAR(10),            -- versión de T&C aceptada
  
  -- Plan
  plan            ENUM('free','pro','vendor') DEFAULT 'free',
  credits_remaining INTEGER DEFAULT 3,
  credits_reset_at  TIMESTAMP,           -- próximo reset diario
  
  -- Opcional para personalización
  location_province VARCHAR(50),
  
  created_at      TIMESTAMP DEFAULT NOW(),
  last_active_at  TIMESTAMP
);
```

---

## 2.2 Cómo Se Nutre la Base de Datos del Catálogo

### Fuentes Tier 1 — Datos Estructurados Oficiales (sin fricción legal)
- **PokéAPI** (pokeapi.co): datos de Pokémon, completamente open source, MIT License.
- **Pokémon TCG API** (pokemontcg.io): catálogo completo de cartas Pokémon con sets, números, rarezas, imágenes en alta res. Free tier con 20.000 req/día.
- **YGOPRODeck API** (ygoprodeck.com/api): catálogo Yu-Gi-Oh! completo, free.
- **Scryfall API** (scryfall.com): catálogo MTG completo, totalmente libre para uso no comercial y comercial con atribución.
- **OPTCG DB** para One Piece TCG: datos oficiales.

### Fuentes Tier 2 — Datos de Mercado
- **TCGPlayer API**: requiere registro como partner, free para developers, rate limits generosos.
- **eBay Finding API + eBay Analytics API**: gratis para developers, permite buscar "sold listings".
- **CardMarket API**: catálogo europeo, free tier disponible.
- **Cardtrader API**: plataforma internacional, API disponible.
- **bluelytics.com.ar API**: dólar blue en tiempo real, completamente gratuito, argentino.

### Fuentes Tier 3 — Enriquecimiento Visual (requiere cuidado legal)
- **Bulbapedia, Yugipedia, MTG Wiki**: contenido bajo licencias CC con restricciones de uso comercial. Las imágenes de las cartas son propiedad de sus respectivos dueños (The Pokémon Company, Konami, Wizards of the Coast). NO se pueden almacenar masivamente para uso comercial.
- **Solución:** se guardan solo los metadatos (colores de referencia, tipografías, proporciones) pero NO la imagen completa. Las imágenes se sirven en runtime desde las APIs oficiales que sí lo permiten (Pokémon TCG API sirve imágenes de cartas para apps de terceros).

### Nutrición comunitaria controlada
- Cuando un usuario sube una foto de una carta que no está en el catálogo, la IA intenta identificarla y crear el registro.
- El registro queda en estado "pendiente de verificación" hasta que 3 usuarios diferentes confirmen la misma carta.
- Esto permite escalar el catálogo sin depender solo de fuentes externas.

---

# PARTE III — MARCO LEGAL COMPLETO

## 3.1 Problemas Legales Identificados y Soluciones

---

### PROBLEMA 1: Propiedad intelectual de las cartas
**Riesgo:** Las imágenes de cartas (arte, diseño, nombre) son propiedad intelectual de The Pokémon Company International, Konami, Wizards of the Coast (Hasbro), Bandai, etc. Almacenarlas y mostrarlas sin licencia puede constituir infracción de copyright.

**Solución:**
- **No almacenar imágenes de cartas de referencia** en tu propio servidor. Usar las APIs oficiales que sí otorgan licencia para mostrar imágenes (Pokémon TCG API expresamente lo permite para apps de terceros).
- Para la imagen que SUBE EL USUARIO: esa imagen es propiedad del usuario (él tomó la foto). Se puede almacenar con su consentimiento.
- El análisis de la imagen (los datos extraídos: colores, vectores, métricas) no está sujeto a copyright —es información derivada, no la obra en sí.
- **Limitación explícita en T&C:** "Gem Lens no reproduce ni distribuye imágenes de cartas protegidas. Las imágenes mostradas en el catálogo son servidas directamente desde las APIs de sus propietarios."

---

### PROBLEMA 2: Datos personales (GDPR / Ley argentina 25.326)
**Riesgo:** Argentina tiene la Ley de Protección de Datos Personales 25.326. La DNPDP (Dirección Nacional de Protección de Datos Personales) puede multar por manejo inadecuado de datos.

**Solución:**
- Registro explícito de consentimiento informado antes de crear cuenta.
- Política de privacidad redactada en castellano simple.
- Opción de eliminar cuenta y todos los datos asociados (derecho de supresión).
- Las sesiones anónimas: datos de análisis sin identificador personal, solo session_id efímero.
- No recolectar datos innecesarios (no pedir teléfono, dirección, fecha de nacimiento si no son necesarios).
- Inscripción del banco de datos en el Registro Nacional de Bases de Datos de la DNPDP (es obligatorio para bases de datos con fines comerciales en Argentina).

---

### PROBLEMA 3: Publicar fotos de cartas de usuarios en el feed comunitario
**Riesgo:** El usuario sube la foto → Gem Lens la muestra públicamente → ¿Gem Lens tiene derecho a hacer eso?

**Solución:**
- Consentimiento explícito: checkbox separado "Acepto que esta imagen sea visible en el feed público de Gem Lens".
- La licencia que otorga el usuario es limitada: "Licencia no exclusiva, revocable, para mostrar la imagen en el feed de Gem Lens".
- El usuario puede revocarla en cualquier momento y la imagen se elimina del feed (con un plazo máximo de 48hs para propagación de cachés).
- Las imágenes del feed anónimo deben tener watermark automático con el logo de Gem Lens (esto también protege contra scraping masivo).
- Opción de blur automático en las zonas de texto (nombre, HP) si el usuario quiere máxima privacidad de su carta.

---

### PROBLEMA 4: Falsas valuaciones y responsabilidad civil
**Riesgo:** Un usuario vende una carta basándose en la valuación de Gem Lens y luego el precio es diferente → demanda por responsabilidad.

**Solución legal:**
- Disclaimer obligatorio y visible en TODOS los informes: "Esta valuación es una estimación generada por IA con fines informativos únicamente. No constituye una oferta de compra, venta o asesoramiento financiero. Los precios reales pueden variar significativamente. Gem Lens no se responsabiliza por decisiones comerciales tomadas en base a estos datos."
- Disclaimer de condición: "La condición estimada es visual y aproximada. Para grading oficial consultar PSA, CGC, BGS u otro servicio certificado."
- En los T&C: cláusula de limitación de responsabilidad explícita.
- Estos disclaimers no eliminan completamente la responsabilidad, pero la reducen drásticamente y establecen que el usuario fue informado.

---

### PROBLEMA 5: Autenticación de falsificaciones y responsabilidad
**Riesgo:** Gem Lens detecta una carta como falsa, el usuario la tenía en buena fe y la vendió como real → o al revés, Gem Lens dice "auténtica" y era falsa.

**Solución:**
- El sistema NUNCA dice "esta carta es 100% auténtica". Dice "sin indicadores de falsificación detectados (análisis visual IA)".
- El sistema NUNCA dice "esta carta es definitivamente falsa". Dice "se detectaron X indicadores inconsistentes con cartas oficiales. Recomendamos verificación presencial por un experto."
- Disclaimer específico: "La detección de autenticidad de Gem Lens es una herramienta de apoyo visual y no reemplaza la verificación física por un experto certificado. No tiene validez legal ni oficial."
- Categoría "POSIBLE FALSIFICACIÓN" en lugar de "FALSA" —lenguaje precautorio.

---

### PROBLEMA 6: Monetización y Mercado Pago — obligaciones fiscales
**Riesgo:** Si Gem Lens cobra dinero en Argentina, tiene obligaciones tributarias.

**Solución:**
- Registrarse como monotributista (o sociedad SAS) ante AFIP.
- Facturar por los ingresos de suscripciones y créditos.
- Mercado Pago retiene el IVA correspondiente en muchos casos.
- Consultar con contador especializado en plataformas digitales (existen en Argentina con experiencia en SaaS).
- Para pagos internacionales (USD 4/mes): analizar estructura legal apropiada (puede requerir cuenta Stripe o similar con entidad en el exterior).

---

### PROBLEMA 7: Uso de APIs de IA (Gemini, Claude, OpenAI)
**Riesgo:** Los T&C de estas APIs prohíben ciertos usos (desinformación, contenido de adultos, etc.) y tienen claúsulas sobre datos enviados.

**Solución:**
- Revisar y cumplir los T&C de cada proveedor (especialmente prohibición de usar los datos enviados para entrenar modelos del proveedor — configuración enterprise puede ayudar).
- No enviar datos personales identificables en los prompts (la foto no contiene PII por defecto).
- Mantener términos de servicio que informen a los usuarios que las imágenes son procesadas por servicios de IA de terceros.
- Opción: procesamiento on-premise con modelos open source (LLaMA, Gemma) para usuarios con máxima privacidad.

---

## 3.2 Documentos Legales Mínimos que Gem Lens Debe Tener

1. **Términos y Condiciones de Uso** — con cláusulas de: limitación de responsabilidad, uso permitido, propiedad intelectual, licencia del usuario sobre sus fotos, ley aplicable (Argentina).

2. **Política de Privacidad** — cumpliente con Ley 25.326, explicando: qué datos se recolectan, cómo se usan, por cuánto tiempo se conservan, con quién se comparten (APIs de IA), derechos del usuario (acceso, rectificación, supresión).

3. **Política de Cookies** — si se usan cookies para sesiones anónimas.

4. **Aviso de Descargo de Responsabilidad (Disclaimer)** — visible en cada informe generado.

5. **Política de Contenido del Feed** — qué tipo de imágenes/contenido está permitido publicar, proceso de reporte y moderación.

---

## 3.3 Checklist Legal para el Lanzamiento

- [ ] Registrar empresa (monotributo o SAS) antes de cobrar
- [ ] Inscribir base de datos en DNPDP
- [ ] Redactar T&C con abogado especializado (costo aprox: $50.000-150.000 ARS una sola vez)
- [ ] Disclaimer visible en TODOS los informes
- [ ] Consentimiento de feed explícito y separado del registro
- [ ] Política de retención de datos (¿cuánto tiempo guardás las fotos de usuarios?)
- [ ] Proceso de "derecho al olvido" implementado (botón eliminar cuenta + datos)
- [ ] Revisar T&C de Pokémon TCG API y demás APIs utilizadas
- [ ] Registrar marca "Gem Lens" en INPI Argentina (aprox. $20.000-40.000 ARS + honorarios)
- [ ] Implementar HTTPS obligatorio en todo el sitio
- [ ] Auditoría de seguridad básica antes de lanzamiento (especialmente en el storage de fotos)

---

*Gem Lens — Sistema Técnico y Legal v2.0 | Marzo 2026*
