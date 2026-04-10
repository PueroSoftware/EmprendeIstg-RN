# Exposición de la Funcionalidad "Favoritos"

Este documento resume el trabajo realizado para implementar la característica de "Favoritos" en la aplicación, detallando los nuevos componentes, las modificaciones a archivos existentes y la estructura de la base de datos.

---

## 1. Resumen de la Funcionalidad

Se ha implementado un sistema completo que permite a los usuarios:
1.  **Marcar** una publicación como favorita desde la pantalla principal (`Home`).
2.  **Visualizar** todas sus publicaciones guardadas en una nueva pantalla llamada "Mis Favoritos".
3.  **Eliminar** publicaciones de su lista de favoritos.

La funcionalidad está diseñada para ser intuitiva y eficiente, con una interfaz que se actualiza en tiempo real para reflejar las acciones del usuario.

---

## 2. Archivos Nuevos Creados

Para construir esta característica, se crearon los siguientes archivos:

#### a) Clases de Kotlin (Lógica y UI):
*   `app/src/main/java/istg/edu/ec/appEmprendeISTGDev/ui/favoritos/FavoritosFragment.kt`: El fragmento que controla la pantalla "Mis Favoritos".
*   `app/src/main/java/istg/edu/ec/appEmprendeISTGDev/viewModel/FavoritosViewModel.kt`: El ViewModel que gestiona la obtención y eliminación de datos de favoritos desde Firebase.
*   `app/src/main/java/istg/edu/ec/appEmprendeISTGDev/data/adapters/FavoritosAdapter.kt`: El adaptador del RecyclerView que muestra los elementos en la grilla de favoritos.

#### b) Layouts (Diseño de la Interfaz):
*   `app/src/main/res/layout/fragment_favoritos.xml`: Layout principal de la pantalla "Mis Favoritos", que contiene el RecyclerView.
*   `app/src/main/res/layout/item_favorito.xml`: Layout que define la apariencia de cada tarjeta individual en la lista de favoritos.

#### c) Recursos Gráficos (Drawables):
*   `app/src/main/res/drawable/ic_favorito_lleno.xml`: Ícono del corazón rojo para el estado "guardado".
*   `app/src/main/res/drawable/ic_overflow_menu.xml`: Ícono de tres puntos para el menú de opciones en cada favorito.
*   `app/src/main/res/drawable/favorito_item_border.xml`: Borde decorativo para enmarcar cada tarjeta de favorito.

#### d) Menús:
*   `app/src/main/res/menu/menu_favorito_item.xml`: Define las opciones del menú emergente (ej. "Eliminar") para cada favorito.

#### e) Documentación:
*   `Bitacora_Favorito.md`: Registro detallado de cada paso y decisión tomada durante el desarrollo.
*   `Manual_Tecnico.md`: Explicación técnica del funcionamiento interno de la característica.
*   `Manual_Usuario.md`: Guía sencilla para el usuario final.

---

## 3. Archivos Modificados

Varios archivos existentes fueron modificados para integrar la nueva funcionalidad:

*   **`HomeAdapter.kt`**:
    *   **Modificación Principal:** Se inyectó el `HomeViewModel` para conectar la lógica de negocio.
    *   Se implementó la observación de `LiveData` para cambiar el ícono del botón de guardar en tiempo real.
    *   Se añadió el `onClickListener` para llamar a la función `addFavorite`.

*   **`HomeViewModel.kt`**:
    *   **Modificación Principal:** Se añadieron las funciones `addFavorite()` y `isFavorite()` para manejar la interacción con Firebase desde la pantalla principal.

*   **`HomeFragment.kt`**:
    *   **Modificación Principal:** Se actualizó la creación del `HomeAdapter` para pasarle las instancias de `viewModel` y `lifecycleOwner`.

*   **`activity_main_drawer.xml`**:
    *   **Modificación Principal:** Se añadió un nuevo `<item>` para crear el enlace a "Mis Favoritos" en el menú de navegación lateral.

*   **`mobile_navigation.xml`**:
    *   **Modificación Principal:** Se añadió un nuevo destino (`<fragment>`) para registrar `FavoritosFragment` en el grafo de navegación de la aplicación.

---

## 4. Cambios en la Base de Datos (Firebase)

Para soportar esta funcionalidad, se introdujo una nueva estructura en la Firebase Realtime Database:

*   **Nueva Ruta:** `emprendeIstg/Favoritos`

*   **Estructura:**
    ```json
    "Favoritos": {
      "USER_ID_1": {
        "PUBLICACION_ID_A": { "uid": "CREATOR_UID_A" },
        "PUBLICACION_ID_B": { "uid": "CREATOR_UID_B" }
      },
      "USER_ID_2": {
        "PUBLICACION_ID_C": { "uid": "CREATOR_UID_C" }
      }
    }
    ```

*   **Propósito:** Esta estructura actúa como un **índice**. Para cada usuario (`USER_ID`), se guarda una lista de las publicaciones que ha marcado como favoritas (`PUBLICACION_ID`). Dentro de cada favorito, se almacena el `uid` del creador de la publicación para poder localizar y cargar los datos completos de la publicación original desde el nodo `emprendeIstg/publicacion`.
