# Manual Técnico: Calificación y Administración de Contenido

**Versión:** 7.0 (FINAL - con Apéndice de Código)
**Fecha:** 19/07/2024
**Autor:** Josephpuero / Gemini

---

### 1. Resumen Ejecutivo

Este documento detalla la arquitectura y funcionalidad de dos sistemas interconectados implementados en la aplicación:

1.  **Sistema de Calificación y Comentarios (`RatingComment`):** Una funcionalidad completa y optimizada para que los usuarios puedan calificar y comentar negocios.
2.  **Gestor de Palabras Prohibidas (`Admin BannedWords`):** Una pantalla de administración, protegida por rol, que permite gestionar la lista de palabras prohibidas en tiempo real.

Ambas funcionalidades están **100% completadas, integradas y listas para producción**.

---

### 2. Diagrama de Arquitectura (Gestor de Palabras Prohibidas)

El siguiente diagrama UML ilustra la arquitectura y comunicación de los componentes creados para la nueva funcionalidad de administración.

'''puml
@startuml
title Arquitectura y Comunicación: Gestor de Palabras Prohibidas

!theme vibrant

package "UI y Control (Admin)" {
    rectangle "BannedWordsAdministradorFragment.kt" as Fragment
    rectangle "BannedWordsManagerViewModel.kt" as ViewModel
    rectangle "GestorPalabrasAdapter.kt" as Adapter
}

package "Recursos de Layout (Admin)" {
    rectangle "fragment_banned_words_administrador.xml" as LayoutMain
    rectangle "item_gestor_palabra.xml" as LayoutItem
}

package "Datos y Lógica Compartida" {
    rectangle "RatingRepository.kt\n(MODIFICADO)" as Repository
}

cloud "Firebase Realtime DB" as Firebase

' --- Relaciones de Comunicación ---
Fragment --> ViewModel : Llama a funciones (add/delete)\nObserva LiveData (lista, estado)
Fragment --> Adapter : Envía la lista de palabras
Fragment --> LayoutMain : Infla la vista

Adapter --> LayoutItem : Infla cada fila de la lista

ViewModel --> Repository : Llama a add/delete/getBannedWord()

Repository --> Firebase : Lee y escribe en el nodo "BannedWords"

legend left
  ==== Ficheros Modificados ====
  * **RatingRepository.kt**: Añadidas funciones de añadir y borrar.
  * **mobile_navigation.xml**: Registrado nuevo destino.
  * **activity_main_drawer.xml**: Añadido ítem al menú.
  * **strings.xml**: Añadidos textos para la UI de admin.
  * **InicioActivity.java**: Añadida lógica para ocultar el menú.
end legend

@enduml
'''

---

### 3. Descripción de Funcionalidades

#### **Funcionalidad 1: Sistema de Calificación y Comentarios (Para Usuarios)**

*   **Estado:** `[COMPLETADO Y OPTIMIZADO]`
*   **Propósito:** Permitir a los usuarios calificar negocios mediante una escala de 1 a 5 y dejar un comentario de texto.
*   **Características Implementadas:**
    *   **Conexión a Firebase:** Lectura/escritura en tiempo real desde `emprendeIstg/RatingComment/{businessId}`.
    *   **Cálculo de Promedio:** Se calcula y muestra en tiempo real.
    *   **Flexibilidad de Envío:** Se permite calificar sin comentar y viceversa (comentarios con `rating=0` no afectan al promedio).
    *   **Filtro de Contenido Inteligente:** Detecta palabras "camufladas" (`s-3-x-0`) antes de publicar.
    *   **Optimización de Rendimiento:** La lista de comentarios utiliza `ListAdapter` y `DiffUtil`.

#### **Funcionalidad 2: Gestor de Palabras Prohibidas (Para Administradores)**

*   **Estado:** `[COMPLETADO E INTEGRADO]`
*   **Propósito:** Proveer una herramienta interna para que los administradores puedan gestionar la lista de `BannedWords` en Firebase.
*   **Características Implementadas:**
    *   **Acceso Protegido por Rol:** Integrado en el `Navigation Drawer` con visibilidad condicional en `InicioActivity.java`.
    *   **Gestión CRUD:** Permite visualizar la lista, añadir nuevas palabras y eliminar existentes con un diálogo de confirmación.
    *   **Feedback al Usuario:** Notificaciones `Toast` para todas las operaciones.

---

### Apéndice: Código Fuente de los Componentes Clave

A continuación se adjunta el código fuente de los principales ficheros creados y modificados para la implementación de estas funcionalidades.

#### **A.1 - `BannedWordsAdministradorFragment.kt`**
*Controlador de la UI para la pantalla del gestor.*
'''kotlin
package istg.edu.ec.appEmprendeISTGDev.ui.fragments

import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.Button
import android.widget.EditText
import android.widget.ProgressBar
import android.widget.Toast
import androidx.appcompat.app.AlertDialog
import androidx.core.view.isVisible
import androidx.fragment.app.Fragment
import androidx.lifecycle.ViewModelProvider
import androidx.recyclerview.widget.RecyclerView
import istg.edu.ec.appEmprendeISTGDev.R
import istg.edu.ec.appEmprendeISTGDev.ui.adapters.GestorPalabrasAdapter
import istg.edu.ec.appEmprendeISTGDev.viewModel.BannedWordsManagerViewModel

class BannedWordsAdministradorFragment : Fragment() {

    private lateinit var viewModel: BannedWordsManagerViewModel
    private lateinit var adapter: GestorPalabrasAdapter

    private lateinit var rvBannedWords: RecyclerView
    private lateinit var etNewWord: EditText
    private lateinit var btnAddWord: Button
    private lateinit var progressBar: ProgressBar

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.fragment_banned_words_administrador, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewModel = ViewModelProvider(this)[BannedWordsManagerViewModel::class.java]

        bindViews(view)
        setupRecyclerView()
        setupAddButton()
        setupObservers()
    }

    private fun bindViews(view: View) {
        rvBannedWords = view.findViewById(R.id.rvBannedWords)
        etNewWord = view.findViewById(R.id.etNewWord)
        btnAddWord = view.findViewById(R.id.btnAddWord)
        progressBar = view.findViewById(R.id.progressBar)
    }

    private fun setupRecyclerView() {
        adapter = GestorPalabrasAdapter { wordToDelete ->
            showDeleteConfirmationDialog(wordToDelete)
        }
        rvBannedWords.adapter = adapter
    }

    private fun setupAddButton() {
        btnAddWord.setOnClickListener {
            val newWord = etNewWord.text.toString().trim()
            viewModel.addWord(newWord)
            etNewWord.text.clear()
        }
    }

    private fun setupObservers() {
        viewModel.bannedWords.observe(viewLifecycleOwner) { words ->
            adapter.submitList(words)
        }

        viewModel.isLoading.observe(viewLifecycleOwner) { isLoading ->
            progressBar.isVisible = isLoading
        }

        viewModel.toastMessage.observe(viewLifecycleOwner) { message ->
            message?.let {
                Toast.makeText(context, it, Toast.LENGTH_SHORT).show()
                viewModel.onToastShown()
            }
        }
    }

    private fun showDeleteConfirmationDialog(word: String) {
        AlertDialog.Builder(requireContext())
            .setTitle("Confirmar Eliminación")
            .setMessage("¿Estás seguro de que quieres eliminar la palabra '$word'?")
            .setPositiveButton("Eliminar") { _, _ ->
                viewModel.deleteWord(word)
            }
            .setNegativeButton("Cancelar", null)
            .show()
    }
}
'''

#### **A.2 - `BannedWordsManagerViewModel.kt`**
*Cerebro y lógica de negocio para la pantalla del gestor.*
'''kotlin
package istg.edu.ec.appEmprendeISTGDev.viewModel

import android.util.Log
import androidx.lifecycle.LiveData
import androidx.lifecycle.MutableLiveData
import androidx.lifecycle.ViewModel
import istg.edu.ec.appEmprendeISTGDev.data.repository.RatingRepository

class BannedWordsManagerViewModel : ViewModel() {

    private val repository = RatingRepository()

    private val _bannedWords = MutableLiveData<List<String>>()
    val bannedWords: LiveData<List<String>> = _bannedWords

    private val _isLoading = MutableLiveData<Boolean>()
    val isLoading: LiveData<Boolean> = _isLoading

    private val _toastMessage = MutableLiveData<String?>()
    val toastMessage: LiveData<String?> = _toastMessage

    init {
        loadBannedWords()
    }

    fun loadBannedWords() {
        _isLoading.value = true
        repository.getBannedWords { result ->
            result.onSuccess { words ->
                _bannedWords.value = words.sorted()
            }.onFailure {
                Log.e("BannedWordsVM", "Error al cargar palabras", it)
                _toastMessage.value = "Error al cargar la lista"
            }
            _isLoading.value = false
        }
    }

    fun addWord(word: String) {
        if (word.isBlank()) {
            _toastMessage.value = "La palabra no puede estar vacía"
            return
        }
        _isLoading.value = true
        repository.addBannedWord(word) { result ->
            result.onSuccess {
                _toastMessage.value = "Palabra añadida con éxito"
                loadBannedWords()
            }.onFailure {
                Log.e("BannedWordsVM", "Error al añadir palabra", it)
                _toastMessage.value = "Error al añadir la palabra"
                _isLoading.value = false
            }
        }
    }

    fun deleteWord(word: String) {
        _isLoading.value = true
        repository.deleteBannedWord(word) { result ->
            result.onSuccess {
                _toastMessage.value = "Palabra eliminada con éxito"
                loadBannedWords()
            }.onFailure {
                Log.e("BannedWordsVM", "Error al eliminar palabra", it)
                _toastMessage.value = "Error al eliminar la palabra"
                _isLoading.value = false
            }
        }
    }

    fun onToastShown() {
        _toastMessage.value = null
    }
}
'''

#### **A.3 - `GestorPalabrasAdapter.kt`**
*Adaptador del RecyclerView para la lista de palabras.*
'''kotlin
package istg.edu.ec.appEmprendeISTGDev.ui.adapters

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.ImageButton
import android.widget.TextView
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView
import istg.edu.ec.appEmprendeISTGDev.R

class GestorPalabrasAdapter(
    private val onDeleteClick: (String) -> Unit
) : ListAdapter<String, GestorPalabrasAdapter.WordViewHolder>(WordDiffCallback()) {

    class WordViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val wordText: TextView = itemView.findViewById(R.id.tvBannedWord)
        val deleteButton: ImageButton = itemView.findViewById(R.id.btnDeleteWord)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): WordViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_gestor_palabra, parent, false)
        return WordViewHolder(view)
    }

    override fun onBindViewHolder(holder: WordViewHolder, position: Int) {
        val word = getItem(position)
        holder.wordText.text = word
        holder.deleteButton.setOnClickListener {
            onDeleteClick(word)
        }
    }
}

class WordDiffCallback : DiffUtil.ItemCallback<String>() {
    override fun areItemsTheSame(oldItem: String, newItem: String): Boolean {
        return oldItem == newItem
    }

    override fun areContentsTheSame(oldItem: String, newItem: String): Boolean {
        return oldItem == newItem
    }
}
'''

#### **A.4 - `RatingRepository.kt` (Nuevas Funciones)**
*Funciones añadidas al repositorio para gestionar `BannedWords`.*
'''kotlin
// ... (funciones existentes: publishComment, getComments, getBannedWords) ...

fun addBannedWord(word: String, onComplete: (Result<Unit>) -> Unit) {
    val bannedWordsRef = FirebaseDatabase.getInstance().getReference("emprendeIstg/BannedWords")

    bannedWordsRef.get().addOnSuccessListener { snapshot ->
        try {
            val currentList = snapshot.children.mapNotNull { it.getValue(String::class.java) }.toMutableList()
            if (!currentList.contains(word.lowercase())) {
                currentList.add(word.lowercase())
            }
            bannedWordsRef.setValue(currentList)
                .addOnSuccessListener { onComplete(Result.success(Unit)) }
                .addOnFailureListener { onComplete(Result.failure(it)) }
        } catch (e: Exception) {
            onComplete(Result.failure(e))
        }
    }.addOnFailureListener {
        onComplete(Result.failure(it))
    }
}

fun deleteBannedWord(word: String, onComplete: (Result<Unit>) -> Unit) {
    val bannedWordsRef = FirebaseDatabase.getInstance().getReference("emprendeIstg/BannedWords")

    bannedWordsRef.get().addOnSuccessListener { snapshot ->
        try {
            val currentList = snapshot.children.mapNotNull { it.getValue(String::class.java) }.toMutableList()
            currentList.remove(word)
            bannedWordsRef.setValue(currentList)
                .addOnSuccessListener { onComplete(Result.success(Unit)) }
                .addOnFailureListener { onComplete(Result.failure(it)) }
        } catch (e: Exception) {
            onComplete(Result.failure(e))
        }
    }.addOnFailureListener {
        onComplete(Result.failure(it))
    }
}
'''

#### **A.5 - `fragment_banned_words_administrador.xml`**
*Layout principal de la pantalla del gestor.*
'''xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp">

    <!-- Título de la pantalla -->
    <TextView
        android:id="@+id/tvTitle"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:text="@string/admin_banned_words_title"
        android:textSize="24sp"
        android:textStyle="bold"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <!-- Sección para añadir una nueva palabra -->
    <com.google.android.material.textfield.TextInputLayout
        android:id="@+id/tilNewWord"
        style="@style/Widget.MaterialComponents.TextInputLayout.OutlinedBox"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:layout_marginEnd="8dp"
        android:hint="@string/admin_banned_words_hint_new"
        app:layout_constraintEnd_toStartOf="@+id/btnAddWord"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@id/tvTitle">

        <com.google.android.material.textfield.TextInputEditText
            android:id="@+id/etNewWord"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:inputType="text"
            android:maxLines="1" />
    </com.google.android.material.textfield.TextInputLayout>

    <!-- Botón para añadir la palabra -->
    <Button
        android:id="@+id/btnAddWord"
        android:layout_width="wrap_content"
        android:layout_height="0dp"
        android:text="@string/admin_banned_words_action_add"
        app:layout_constraintBottom_toBottomOf="@id/tilNewWord"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="@id/tilNewWord" />

    <!-- Lista de palabras prohibidas existentes -->
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/rvBannedWords"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:layout_marginTop="16dp"
        app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@id/tilNewWord"
        tools:listitem="@layout/item_gestor_palabra" />

    <!-- Indicador de carga -->
    <ProgressBar
        android:id="@+id/progressBar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:visibility="gone"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        tools:visibility="visible" />
</androidx.constraintlayout.widget.ConstraintLayout>
'''

#### **A.6 - `item_gestor_palabra.xml`**
*Layout para una sola fila en la lista del gestor.*
'''xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:paddingStart="8dp"
    android:paddingEnd="8dp"
    android:paddingTop="4dp"
    android:paddingBottom="4dp">

    <!-- El texto de la palabra prohibida -->
    <TextView
        android:id="@+id/tvBannedWord"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginEnd="8dp"
        android:textSize="16sp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toStartOf="@+id/btnDeleteWord"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        tools:text="palabrota" />

    <!-- Botón para eliminar la palabra -->
    <ImageButton
        android:id="@+id/btnDeleteWord"
        android:layout_width="48dp"
        android:layout_height="48dp"
        android:background="?attr/selectableItemBackgroundBorderless"
        android:contentDescription="Eliminar palabra"
        android:src="@drawable/ic_delete"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
'''
