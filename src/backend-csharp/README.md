# 💻 EquineLead - Backend C#

> 📍 **Navegación**: [🏠 Inicio](../../README.md) → Componentes → Backend C#

Este backend implementa una tubería de persistencia para la gestión de usuarios (leads) y productos, incluyendo el registro de interacciones y el cálculo de Lead Score mediante la integración con un motor de Data Science en Python.

> [!IMPORTANT]
> Esta documentación cubre la **Verificación en Local (Hito 1)**. La integración final en la infraestructura de **AWS** (ECS/Fargate con contenedores) es la siguiente fase del proyecto.

---

## 📊 Flujo de Gestión de Leads (Scoring)

```mermaid
sequenceDiagram
    participant U as Cliente (Curl/Swagger)
    participant API as Backend C# (ASP.NET Core)
    participant DB as Postgres (Docker)
    participant ML as Data Science API (Python)

    U->>API: 1. Crear Usuario/Producto
    API->>DB: Persistir datos básicos
    U->>API: 2. Registrar Interacción
    API->>DB: Guardar Interaction
    API->>ML: 3. Solicitar Scoring (Payload JSON)
    ML->>ML: Calcular Score & Clasificación
    ML-->>API: 4. Retornar Score + Versión Modelo
    API->>DB: 5. Persistir LeadScore (v1-rule-based)
    API-->>U: Retornar resultado con clasificación
```

### Detalle del Proceso (¿Cuándo interactúan las APIs?)

Para entender cómo se hablan **C#** y **Python**, imaginemos este flujo paso a paso:

1.  **Paso 1: Registro Inicial (Backend C#)**  
    Primero guardamos los datos básicos: quién es el usuario y qué productos tenemos. Esto se queda guardado en **Postgres**. En este punto, Python todavía no sabe nada.
    
2.  **Paso 2: El Evento Disparador (Backend C#)**  
    Cuando registras una **Interacción** (ej: Carlos vio una Gorra), el Backend C# recibe la orden. Antes de terminar, el Backend dice: *"Oye, necesito saber el puntaje de este tipo"*.
    
3.  **Paso 3: El Pedido al Experto (Interacción C# → Python) 🔄**  
    Aquí es donde ocurre la magia. El Backend C# prepara un paquete de datos (JSON) con el presupuesto del usuario, su tipo (B2B/B2C) y lo que acaba de hacer. Se lo envía por la red a la **API de Data Science (Python)** en el puerto 8090.
    
4.  **Paso 4: El Cálculo Inteligente (Data Science Python)**  
    La API de Python recibe el paquete. No guarda nada en base de datos, solo "piensa". Aplica las reglas de negocio (ej: si presupuesto > 40k, suma puntos) y genera un **Score (Número)** y una **Clasificación (Cold/Warm/Hot)**.
    
5.  **Paso 5: La Respuesta (Python → C#) 🔄**  
    Python le devuelve el resultado al Backend C#. *"Toma, Carlos tiene 20 puntos y es un lead Frío (Cold)"*.
    
6.  **Paso 6: Persistencia Final (Backend C#)**  
    El Backend C# recibe la respuesta, la traduce a su formato y la guarda permanentemente en la tabla `LeadScores` de la base de datos. Recién ahí, vos ves el resultado en tu pantalla.

---

## 🚀 Guía de Levantamiento Local (Paso a Paso)

Para que el sistema funcione correctamente, se deben levantar los tres componentes en el siguiente orden:

### 1️⃣ Pasos Previos (Solo la primera vez)
Si es la primera vez que trabajas en esta terminal, configura el acceso a las herramientas de .NET:
```bash
export DOTNET_ROOT=/snap/dotnet-sdk/current
export PATH="$PATH:$HOME/.dotnet/tools"
```

### 2️⃣ El Motor de Inteligencia (FastAPI - Python)
Levantamos al "Chef" que calculará los puntajes.
1. Abre una terminal y ve a la raíz del proyecto.
2. Ejecuta:
   ```bash
   PYTHONPATH=src/data-science ./venv/bin/python3 -m uvicorn api:app --host 0.0.0.0 --port 8090
   ```
Recuerda dejar esta terminal abierta.

### 3️⃣ La Base de Datos (PostgreSQL en Docker)
Levantamos el "Almacén" de datos.
```bash
# Limpiar si existe uno previo
sudo docker rm -f equine-postgres

# Crear y levantar el contenedor
sudo docker run -d --name equine-postgres \
  -e POSTGRES_DB=NoCountryE48DB \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres123 \
  -p 5432:5432 postgres:15
```

### 4️⃣ El Backend C# (ASP.NET Core)
Finalmente, levantamos el servidor principal.
1. Ve a la carpeta `src/backend-csharp/`.
2. Crea las tablas en la base de datos (Migraciones):
   ```bash
   dotnet ef database update --context AppDbContext
   ```
3. Inicia el servidor:
   ```bash
   dotnet run
   ```

---

## ✅ Verificación del Funcionamiento

### 🌐 A través del Navegador (Swagger)
Una vez el backend esté corriendo, accede a:
👉 [http://localhost:5286/swagger](http://localhost:5286/swagger)

Aquí puedes probar los endpoints visualmente. El flujo recomendado es:
1. `POST /api/User` -> Crear un usuario.
2. `POST /api/Product` -> Crear un producto.
3. `POST /api/Interaction` -> Registrar una interacción (esto generará el score automáticamente).

### 🖥️ A través de Terminal (SQL)
Para verificar que los datos se guardaron correctamente en la base de datos física:
```bash
# Ver el ranking de Leads actualizado (Top 5)
sudo docker exec -it equine-postgres psql -U postgres -d NoCountryE48DB -c "
SELECT u.\"UserName\", s.\"LeadScoreValue\", s.\"LeadScoreClassification\", s.\"ScoreModelVersion\"
FROM \"Users\" u
JOIN \"LeadScores\" s ON u.\"UserId\" = s.\"UserId\"
ORDER BY s.\"LeadScoreValue\" DESC;"
```

---

## 🧱 Componentes técnicos
- **Users**: Puerta de entrada para leads (viene de la **Landing Page**).
- **Products**: Catálogo de productos (puede ser poblado por el **Scrapper**).
- **LeadInteractions**: Registro de actividad (triggers de scoring).
- **LeadScores**: Resultados persistidos del motor de Data Science.

---

## 🔮 Visión y Escalabilidad (Scraping & Landing Page)

Esta arquitectura está diseñada para ser el núcleo de un sistema más grande:
1.  **Origen Landing Page**: El backend C# ya tiene los endpoints listos para recibir los datos de contacto y presupuesto que los usuarios dejen en la landing.
2.  **Origen Scraping (Rust)**: El módulo de Scrapper puede enviar periódicamente nuevos productos o actualizaciones de precios a través de la API de C#, manteniendo el catálogo sincronizado.
3.  **Flexibilidad de Data Science**: Al estar desacoplado, el motor de Python puede evolucionar para usar datos más complejos recolectados por el scrapper (como tendencias de mercado) sin necesidad de modificar drásticamente el backend de C#.
4.  **Dataset Sintético - El Laboratorio**: El proyecto cuenta con un dataset sintético que permite:
    - **Simular Escenarios**: Probar cómo reacciona el sistema ante miles de leads antes de tener tráfico real.
    - **Calibrar Reglas**: Ajustar los umbrales (40/80) basándose en distribuciones realistas de datos.
    - **Entrenamiento Futuro**: Servir como base para pasar de un modelo de "reglas fijas" a un modelo de "Aprendizaje Automático (Machine Learning)" real.

## Tecnologías usadas
- ASP.NET Core 8.0
- Entity Framework Core (PostgreSQL Provider)
- Npgsql (EnableLegacyTimestampBehavior habilitado)
- Docker (PostgreSQL 15)
- Swagger / OpenAPI

## 🧪 **Testing Relacionado**
Consulta las carpetas de pruebas para más detalle:
- [Testing de Backend](../../tests/backend-csharp/README.md)
