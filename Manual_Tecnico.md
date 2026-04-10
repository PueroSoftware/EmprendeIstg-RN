# Manual Técnico: Funcionalidad de Favoritos

## 1. Visión General

La funcionalidad de "Favoritos" permite a los usuarios guardar publicaciones de su interés y verlas en una pantalla dedicada. La implementación sigue el patrón de diseño MVVM (Model-View-ViewModel) y utiliza Firebase Realtime Database como backend.

## 2. Componentes Clave

La funcionalidad se divide en dos partes principales:
1.  **Añadir/Marcar un favorito** desde la pantalla principal.
2.  **Visualizar y gestionar los favoritos** en la pantalla "Mis Favoritos".

### 2.1. Lógica para Añadir Favoritos (Pantalla Home)

| Componente | Rol y Responsabilidades |
| :--- | :--- |
| **`HomeFragment.kt`** | **Vista:** Muestra la lista de todas las publicaciones. Crea e inyecta las instancias de `HomeViewModel` y `LifecycleOwner` en el `HomeAdapter`. |
| **`HomeViewModel.kt`** | **ViewModel:** Contiene la lógica de negocio para interactuar con Firebase. |
| | - `addFavorite(publicacion)`: Escribe en la base de datos para guardar un nuevo favorito. |
| | - `isFavorite(publicacionId)`: Expone un `LiveData<Boolean>` que observa si una publicación específica ya está en la lista de favoritos del usuario. Esto permite que la UI reaccione en tiempo real. |
| **`HomeAdapter.kt`** | **Adaptador:** Vincula los datos de cada publicación con su representación visual (`item_home.xml`). |
| | - Observa el `LiveData` de `isFavorite` para cada ítem y actualiza el ícono del botón `btnGuardar` (`ic_favorito` o `ic_favorito_lleno`). |
| | - Asigna el `onClickListener` al `btnGuardar` para llamar a `viewModel.addFavorite()`. |
| **`item_home.xml`** | **Layout:** Define la apariencia de cada publicación en la lista principal. Contiene el `ImageButton` `btnGuardar`. |

### 2.2. Pantalla "Mis Favoritos"

| Componente | Rol y Responsabilidades |
| :--- | :--- |
| **`FavoritosFragment.kt`** | **Vista:** Contiene la lógica de la UI para la pantalla "Mis Favoritos". |
| | - Configura un `RecyclerView` con un `GridLayoutManager` de 2 columnas. |
| | - Llama a `viewModel.fetchFavoritos()` en el método `onResume()` para asegurar que la lista esté siempre actualizada al entrar a la pantalla. |
| | - Observa el `LiveData` del `FavoritosViewModel` para actualizar el adaptador con la lista de favoritos. |
| **`FavoritosViewModel.kt`** | **ViewModel:** Gestiona los datos de la pantalla "Mis Favoritos". |
| | - `fetchFavoritos()`: Obtiene los IDs de las publicaciones favoritas y luego busca los datos completos de cada una, incluyendo la `imagenBase64`. |
| | - `removeFavorite(publicacionId)`: Elimina un favorito de Firebase y, mediante un `SuccessListener`, llama a `fetchFavoritos()` para forzar una actualización inmediata de la UI. |
| **`FavoritosAdapter.kt`** | **Adaptador:** Vincula los datos de cada publicación favorita con su `item_favorito.xml`. |
| | - Muestra la `imagenBase64` usando la vista personalizada `NegocioImageView`. |
| | - Implementa un `PopupMenu` en el botón de tres puntos, que contiene la opción "Eliminar". |
| | - Al eliminar, invoca la función `onRemoveClick` que se pasa desde el fragmento, la cual a su vez llama a `viewModel.removeFavorite()`. |
| **`item_favorito.xml`** | **Layout:** Define la apariencia de cada tarjeta en la grilla de favoritos. Incluye un borde decorativo (`favorito_item_border.xml`). |
| **`menu_favorito_item.xml`** | **Menú:** Define las opciones que aparecen en el `PopupMenu` (actualmente, solo "Eliminar"). |

## 3. Flujo de Datos

### 3.1. Guardar un Favorito

1.  **Usuario** presiona `btnGuardar` en `item_home.xml`.
2.  **`HomeAdapter`** llama a `viewModel.addFavorite(publicacion)`.
3.  **`HomeViewModel`** escribe el ID de la publicación en `emprendeIstg/Favoritos/{userId}/{publicacionId}` en Firebase.
4.  El `LiveData` de `isFavorite` (que ya estaba siendo observado) detecta el cambio en la base de datos.
5.  **`HomeAdapter`** recibe el nuevo valor (`true`) y actualiza el ícono a `ic_favorito_lleno`, deshabilitando el botón.

### 3.2. Ver y Eliminar Favoritos

1.  **Usuario** navega a "Mis Favoritos".
2.  Se ejecuta `onResume()` en **`FavoritosFragment`**, que llama a `viewModel.fetchFavoritos()`.
3.  **`FavoritosViewModel`** obtiene la lista de IDs de `emprendeIstg/Favoritos` y luego, para cada ID, obtiene los datos completos de la publicación desde `emprendeIstg/publicacion`.
4.  El ViewModel actualiza su `LiveData`, que es observado por el fragmento.
5.  **`FavoritosFragment`** pasa la lista al **`FavoritosAdapter`**, que renderiza la grilla.
6.  **Usuario** presiona el botón de tres puntos y selecciona "Eliminar".
7.  **`FavoritosAdapter`** llama a la función `onRemoveClick`.
8.  **`FavoritosFragment`** llama a `viewModel.removeFavorite(publicacionId)`.
9.  **`FavoritosViewModel`** elimina el dato de Firebase y, al completarse, llama a `fetchFavoritos()` de nuevo para refrescar la lista en la UI.

## 4. Estructura en Firebase

-   **Publicaciones:** `emprendeIstg/publicacion/{creatorUid}/{publicacionId}`
    -   Contiene todos los datos de una publicación, incluyendo `imagenBase64`.
-   **Favoritos:** `emprendeIstg/Favoritos/{userId}/{publicacionId}`
    -   Actúa como un índice. Guarda una referencia al `creatorUid` de la publicación para poder encontrarla fácilmente.
