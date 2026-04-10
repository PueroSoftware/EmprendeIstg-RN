# Bitácora de Implementación: Funcionalidad de Favoritos

Este documento registra los pasos realizados para implementar la funcionalidad de "Añadir a Favorito" y la pantalla "Mis Favoritos".

### Fase 1: Lógica para Añadir y Marcar Favoritos

El objetivo de esta fase fue permitir que un usuario marque una publicación como favorita desde la pantalla principal. Una vez marcada, el botón cambia de apariencia y se deshabilita.

**Paso 1: Preparación de Recursos de la Interfaz**

*   **1.1. Creación del Ícono de Corazón Lleno:**
    *   **Qué se hizo:** Se creó un nuevo `Vector Asset` llamado `ic_favorito_lleno.xml` con un color rojo.
    *   **Propósito:** Representar el estado "guardado" de una publicación.
    *   **Ubicación:** `app/src/main/res/drawable/ic_favorito_lleno.xml`

*   **1.2. Verificación del Ícono por Defecto:**
    *   **Qué se hizo:** Se confirmó que el `ImageButton` con ID `@+id/btnGuardar` en `item_home.xml` ya utilizaba un ícono de corazón (`ic_favorito`) como estado inicial.

**Paso 2: Implementación de la Lógica de Negocio en `HomeViewModel.kt`**

*   **2.1. Se añadió la función `addFavorite(publicacion)`:**
    *   **Qué hace:** Recibe un objeto `AgregarNegocioModel`. Obtiene el ID del usuario actual y el ID de la publicación. Se conecta a Firebase y guarda una referencia en la ruta `emprendeIstg/Favoritos/{userId}/{publicacionId}`.

*   **2.2. Se añadió la función `isFavorite(publicacionId)`:**
    *   **Qué hace:** Recibe el ID de una publicación y devuelve un `LiveData<Boolean>`.
    *   **Propósito:** Escucha en tiempo real si existe una entrada para esa publicación en los favoritos del usuario. Devuelve `true` si existe, `false` si no.

**Paso 3: Conexión de la Lógica con la Interfaz en `HomeAdapter.kt`**

*   **3.1. Se modificó el constructor del `HomeAdapter`:**
    *   **Qué se hizo:** Se añadieron los parámetros `viewModel: HomeViewModel` y `lifecycleOwner: LifecycleOwner`.
    *   **Propósito:** Darle al adaptador acceso a las funciones del ViewModel y permitir que `LiveData` funcione correctamente.

*   **3.2. Se implementó la lógica en `onBindViewHolder`:**
    *   **a. Observar el estado:** Se llama a `viewModel.isFavorite(publicacionId).observe(...)`.
    *   **b. Actualizar el ícono:** Dentro del observador, si el resultado es `true`, se establece el ícono `ic_favorito_lleno` y se deshabilita el botón. Si es `false`, se mantiene el ícono `ic_favorito` y el botón permanece habilitado.
    *   **c. Configurar la acción:** Se asigna un `setOnClickListener` al `btnGuardar` que llama a `viewModel.addFavorite(publicacion)`.

**Paso 4: Actualización Final en `HomeFragment.kt`**

*   **Qué se hizo:** Se modificó la línea donde se crea la instancia del `HomeAdapter` para pasar las instancias requeridas de `viewModel` y `viewLifecycleOwner`.
*   **Línea clave:** `adapter = HomeAdapter(emptyList(), homeViewModel, viewLifecycleOwner)`

---

### Fase 2: Creación de la Pantalla "Mis Favoritos"

El objetivo de esta fase fue construir la pantalla para que el usuario pueda ver todas las publicaciones que ha guardado.

**Paso 5: Creación de la Interfaz y Navegación**

*   **5.1. Añadir item al menú:**
    *   **Qué se hizo:** Se añadió el item "Mis Favoritos" al menú de navegación lateral.
    *   **Archivo:** `app/src/main/res/menu/activity_main_drawer.xml`

*   **5.2. Crear el Fragmento y su Layout:**
    *   **Qué se hizo:** Se crearon el archivo `FavoritosFragment.kt` y su layout `fragment_favoritos.xml`. El layout contiene un `RecyclerView` y una `ProgressBar`.
    *   **Ubicación:** `app/src/main/java/istg/edu/ec/appEmprendeISTGDev/ui/favoritos/` y `app/src/main/res/layout/`

*   **5.3. Definir la Navegación:**
    *   **Qué se hizo:** Se añadió el destino para `favoritosFragment` en el grafo de navegación para permitir la transición desde el menú.
    *   **Archivo:** `app/src/main/res/navigation/mobile_navigation.xml`

**Paso 6: Crear el `FavoritosViewModel.kt`**

*   **Qué se hizo:** Se creó un nuevo ViewModel con la lógica para leer los favoritos del usuario desde Firebase.
*   **Funcionalidad clave:**
    *   `fetchFavoritos()`: Obtiene la lista de IDs de publicaciones favoritas y luego busca los detalles completos de cada publicación (incluyendo la `imagenBase64`).
    *   `removeFavorite()`: Permite eliminar una publicación de la lista de favoritos.
    *   Expone un `LiveData` con la lista de publicaciones favoritas completas.

**Paso 7: Configurar el `FavoritosFragment` y su Adaptador**

*   **7.1. Crear el Layout del Item:**
    *   **Qué se hizo:** Se creó `item_favorito.xml`, un layout que define la apariencia de cada elemento en la grilla, mostrando la imagen, el nombre y un botón para eliminar.
    *   **Ubicación:** `app/src/main/res/layout/item_favorito.xml`

*   **7.2. Crear el Adaptador:**
    *   **Qué se hizo:** Se creó `FavoritosAdapter.kt`, que se encarga de mostrar los favoritos en una grilla, decodificar la `imagenBase64` y manejar el clic en el botón de eliminar.

*   **7.3. Configurar el `FavoritosFragment`:**
    *   **Qué se hizo:** Se conectó el `FavoritosViewModel` y el `FavoritosAdapter`. Se configuró el `RecyclerView` con un `GridLayoutManager` de 3 columnas.

**Paso 8: Solución de Problemas y Mejoras**

*   **8.1. Corrección de Navegación:**
    *   **Problema:** El clic en "Mis Favoritos" no abría la pantalla.
    *   **Solución:** Se corrigió el archivo `mobile_navigation.xml`, que no tenía la entrada para `favoritosFragment`.

*   **8.2. Actualización Automática de la Lista:**
    *   **Problema:** La lista de favoritos no se actualizaba automáticamente al añadir un nuevo elemento.
    *   **Solución:** Se modificó `FavoritosFragment.kt` para que la llamada a `viewModel.fetchFavoritos()` se realice en el método `onResume()`, asegurando que los datos se recarguen cada vez que la pantalla se vuelve visible.

*   **8.3. Limpieza de Código:**
    *   **Qué se hizo:** Se eliminaron advertencias ("X amarillas") en `HomeAdapter.kt` para mejorar la legibilidad del código.
    *   **Qué se hizo:** Se eliminó un elemento duplicado ("Compartir Aplicación") del menú de navegación.

**Paso 9: Ajustes Finales de la Interfaz de "Mis Favoritos"**

*   **9.1. Cambio de Grilla:**
    *   **Qué se hizo:** Se modificó el `GridLayoutManager` en `FavoritosFragment.kt` para que muestre una grilla de 2 columnas en lugar de 3.

*   **9.2. Rediseño del Botón de Eliminar:**
    *   **Qué se hizo:** Se reemplazó el ícono del tacho de basura por un ícono de tres puntos (`ic_overflow_menu.xml`).
    *   **Funcionalidad:** En `FavoritosAdapter.kt`, se implementó un `PopupMenu` que se muestra al hacer clic en los tres puntos, con una opción "Eliminar".

*   **9.3. Ajuste Visual del Item:**
    *   **Qué se hizo:** En `item_favorito.xml`, se añadió un margen superior a la imagen y se ajustaron las restricciones para darle una apariencia más cuadrada.

**Paso 10: Mejoras de Experiencia de Usuario (UX)**

*   **10.1. Eliminación Instantánea:**
    *   **Problema:** Al eliminar un favorito, el cambio no se reflejaba en la UI hasta que se volvía a entrar a la pantalla.
    *   **Solución:** Se modificó la función `removeFavorite` en `FavoritosViewModel.kt`. Ahora, después de que Firebase confirma la eliminación, se llama automáticamente a `fetchFavoritos()` para recargar la lista y actualizar la UI al instante.

*   **10.2. Refinamiento del Diseño del Item:**
    *   **Qué se hizo:** Se ajustó el layout `item_favorito.xml` para mejorar la composición visual.
    *   **Detalles:** Se movió el botón de menú (tres puntos) al lado del nombre del negocio (debajo de la imagen) para evitar que se superponga. Se añadió un `padding` general al contenedor para crear un margen interno que separa los elementos de los bordes de la tarjeta.

*   **10.3. Marco Decorativo:**
    *   **Qué se hizo:** Se creó un `drawable` XML (`favorito_item_border.xml`) que define un borde con esquinas redondeadas.
    *   **Resultado:** Se aplicó este `drawable` como fondo al layout de cada item, añadiendo un detalle estético que define y realza cada elemento en la grilla.
