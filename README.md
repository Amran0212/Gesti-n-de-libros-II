package main

import (
	"errors"
	"fmt"
	"strings"
)

// ======================= // ERRORES PERSONALIZADOS // =======================
var (
	ErrLibroNoEncontrado  = errors.New("libro no encontrado")
	ErrIDInvalido         = errors.New("ID inválido: debe ser mayor que cero")
	ErrDatosIncompletos   = errors.New("datos incompletos: título y autor son obligatorios")
	ErrRepositorioNoListo = errors.New("repositorio no inicializado correctamente")
)

// ======================= // MODELO DE DATOS (ENCAPSULADO) // =======================
// Libro representa la entidad de dominio con validación interna
type Libro struct {
	id        int // Campo PRIVADO (minúscula) para encapsulación
	titulo    string
	autor     string
	categoria string
	formato   string
}

// NewLibro crea una instancia válida de Libro con validación
// Retorna error si los datos mínimos son inválidos
func NewLibro(id int, titulo, autor, categoria, formato string) (*Libro, error) {
	titulo = strings.TrimSpace(titulo)
	autor = strings.TrimSpace(autor)

	if id <= 0 {
		return nil, ErrIDInvalido
	}
	if titulo == "" || autor == "" {
		return nil, ErrDatosIncompletos
	}

	return &Libro{
		id:        id,
		titulo:    titulo,
		autor:     autor,
		categoria: strings.TrimSpace(categoria),
		formato:   strings.TrimSpace(formato),
	}, nil
}

// GetID devuelve el identificador único (acceso controlado)
func (l *Libro) GetID() int { return l.id }

// GetTitulo devuelve el título normalizado
func (l *Libro) GetTitulo() string { return l.titulo }

// GetAutor devuelve el autor normalizado
func (l *Libro) GetAutor() string { return l.autor }

// GetCategoria devuelve la categoría
func (l *Libro) GetCategoria() string { return l.categoria }

// GetFormato devuelve el formato
func (l *Libro) GetFormato() string { return l.formato }

// String implementa fmt.Stringer para impresión formateada
func (l *Libro) String() string {
	return fmt.Sprintf("ID: %d | Título: %s | Autor: %s | Categoría: %s | Formato: %s",
		l.id, l.titulo, l.autor, l.categoria, l.formato)
}

// ======================= // INTERFAZ DE REPOSITORIO (ABSTRACCIÓN) // =======================
// RepositorioLibros define el contrato para operaciones de persistencia
// Permite cambiar implementación (memoria, BD, archivo) sin afectar la lógica de negocio
type RepositorioLibros interface {
	Agregar(libro *Libro) error
	Listar() ([]*Libro, error)
	Buscar(criterio string) ([]*Libro, error)
	Eliminar(id int) error
	SiguienteID() int
}

// ======================= // IMPLEMENTACIÓN EN MEMORIA (ENCAPSULADA) // =======================
// repositorioMemoria implementa RepositorioLibros con estado privado
type repositorioMemoria struct {
	libros    []*Libro // Slice PRIVADO: acceso solo mediante métodos
	idCounter int
}

// NewRepositorioMemoria crea una nueva instancia del repositorio en memoria
func NewRepositorioMemoria() RepositorioLibros {
	return &repositorioMemoria{
		libros:    make([]*Libro, 0),
		idCounter: 1,
	}
}

// Agregar inserta un libro en el repositorio
// Valida que el libro no sea nil y gestiona el ID automáticamente
func (r *repositorioMemoria) Agregar(libro *Libro) error {
	if libro == nil {
		return errors.New("no se puede agregar un libro nil")
	}
	// Asignar ID desde el contador interno (encapsulado)
	libroConID, err := NewLibro(r.idCounter, libro.GetTitulo(), libro.GetAutor(),
		libro.GetCategoria(), libro.GetFormato())
	if err != nil {
		return fmt.Errorf("error al crear libro con ID: %w", err)
	}
	r.libros = append(r.libros, libroConID)
	r.idCounter++
	return nil
}

// Listar devuelve una COPIA de los libros para evitar fugas de encapsulación
// Importante: Nunca exponer directamente el slice interno (r.libros)
func (r *repositorioMemoria) Listar() ([]*Libro, error) {
	if len(r.libros) == 0 {
		return []*Libro{}, nil // Retorna slice vacío, no error (diseño intencional)
	}
	// Crear copia defensiva para evitar modificaciones externas
	copia := make([]*Libro, len(r.libros))
	copy(copia, r.libros)
	return copia, nil
}

// Buscar realiza búsqueda insensible a mayúsculas/minúsculas en título y autor
// Usa strings.Contains para coincidencias parciales (ej: "garcía" encuentra "García Márquez")
func (r *repositorioMemoria) Buscar(criterio string) ([]*Libro, error) {
	criterio = strings.ToLower(strings.TrimSpace(criterio))
	if criterio == "" {
		return []*Libro{}, nil
	}

	var resultados []*Libro
	for _, libro := range r.libros {
		titulo := strings.ToLower(libro.GetTitulo())
		autor := strings.ToLower(libro.GetAutor())

		if strings.Contains(titulo, criterio) || strings.Contains(autor, criterio) {
			resultados = append(resultados, libro)
		}
	}
	return resultados, nil
}

// Eliminar remueve un libro por ID
// Retorna ErrLibroNoEncontrado si no existe (no es error catastrófico)
func (r *repositorioMemoria) Eliminar(id int) error {
	if id <= 0 {
		return ErrIDInvalido
	}

	indice := -1
	for i, libro := range r.libros {
		if libro.GetID() == id {
			indice = i
			break
		}
	}

	if indice == -1 {
		return ErrLibroNoEncontrado
	}

	// Técnica de eliminación eficiente en slices:
	// Mover el último elemento a la posición del eliminado y reducir el slice
	r.libros[indice] = r.libros[len(r.libros)-1]
	r.libros = r.libros[:len(r.libros)-1]
	return nil
}

// SiguienteID expone el próximo ID disponible SIN permitir modificar el contador
func (r *repositorioMemoria) SiguienteID() int {
	return r.idCounter
}

// ======================= // SERVICIO DE GESTIÓN (LÓGICA DE NEGOCIO) // =======================
// GestorLibros encapsula la lógica de negocio y depende de la interfaz RepositorioLibros
type GestorLibros struct {
	repo RepositorioLibros
}

// NewGestorLibros crea un nuevo gestor con inyección de dependencias
// Permite fácil testing con repositorios mock
func NewGestorLibros(repo RepositorioLibros) (*GestorLibros, error) {
	if repo == nil {
		return nil, ErrRepositorioNoListo
	}
	return &GestorLibros{repo: repo}, nil
}

// AgregarLibro gestiona la creación completa de un libro
// Separa responsabilidades: validación aquí, persistencia en el repositorio
func (g *GestorLibros) AgregarLibro(titulo, autor, categoria, formato string) error {
	// Validación de negocio antes de crear el libro
	titulo = strings.TrimSpace(titulo)
	autor = strings.TrimSpace(autor)
	if titulo == "" || autor == "" {
		return ErrDatosIncompletos
	}

	// Crear libro SIN ID (el repositorio lo asignará)
	libro, err := NewLibro(0, titulo, autor, categoria, formato)
	if err != nil {
		return fmt.Errorf("error en validación del libro: %w", err)
	}

	return g.repo.Agregar(libro)
}

// ListarLibros delega al repositorio y formatea la salida
func (g *GestorLibros) ListarLibros() ([]*Libro, error) {
	return g.repo.Listar()
}

// BuscarLibros delega al repositorio con preprocesamiento del criterio
func (g *GestorLibros) BuscarLibros(criterio string) ([]*Libro, error) {
	return g.repo.Buscar(criterio)
}

// EliminarLibro delega al repositorio con validación adicional
func (g *GestorLibros) EliminarLibro(id int) error {
	if id <= 0 {
		return ErrIDInvalido
	}
	return g.repo.Eliminar(id)
}

// ======================= // FUNCIÓN PRINCIPAL (INTERFAZ DE USUARIO) // =======================
func main() {
	// Inyección de dependencias: creamos repositorio y gestor
	repo := NewRepositorioMemoria()
	gestor, err := NewGestorLibros(repo)
	if err != nil {
		fmt.Printf("Error fatal al iniciar el sistema: %v\n", err)
		return
	}

	ejecutarMenu(gestor)
}

// ejecutarMenu gestiona la interacción con el usuario
// Separa la lógica de UI de la lógica de negocio
func ejecutarMenu(gestor *GestorLibros) {
	opcion := 0

	for opcion != 5 {
		mostrarMenu()
		fmt.Print("Seleccione una opción: ")

		// Manejo robusto de entrada: captura errores de escaneo
		_, err := fmt.Scanln(&opcion)
		if err != nil {
			fmt.Println("Error: entrada inválida. Por favor ingrese un número.")
			limpiarBuffer()
			continue
		}

		switch opcion {
		case 1:
			manejarAgregar(gestor)
		case 2:
			manejarListar(gestor)
		case 3:
			manejarBuscar(gestor)
		case 4:
			manejarEliminar(gestor)
		case 5:
			fmt.Println("Saliendo del sistema... ¡Hasta pronto!")
		default:
			fmt.Println("⚠️ Opción inválida. Seleccione un número entre 1 y 5.")
		}
	}
}

// mostrarMenu imprime la interfaz de usuario
func mostrarMenu() {
	fmt.Println("\n" + strings.Repeat("=", 50))
	fmt.Println("📚 SISTEMA DE GESTIÓN DE LIBROS ELECTRÓNICOS")
	fmt.Println(strings.Repeat("=", 50))
	fmt.Println("1. Agregar libro")
	fmt.Println("2. Listar libros")
	fmt.Println("3. Buscar libro (por título o autor)")
	fmt.Println("4. Eliminar libro")
	fmt.Println("5. Salir")
}

// manejarAgregar gestiona el flujo de creación de libros con validación
func manejarAgregar(gestor *GestorLibros) {
	fmt.Println("\n--- 📥 Agregar Nuevo Libro ---")

	var titulo, autor, categoria, formato string
	fmt.Print("Título: ")
	fmt.Scanln(&titulo)
	fmt.Print("Autor: ")
	fmt.Scanln(&autor)
	fmt.Print("Categoría: ")
	fmt.Scanln(&categoria)
	fmt.Print("Formato (PDF, EPUB, MOBI): ")
	fmt.Scanln(&formato)

	// Llamada a la lógica de negocio con manejo de errores
	if err := gestor.AgregarLibro(titulo, autor, categoria, formato); err != nil {
		fmt.Printf("❌ Error al agregar libro: %v\n", err)
		return
	}
	fmt.Println("✅ Libro agregado correctamente con ID:", gestor.repo.SiguienteID()-1)
}

// manejarListar muestra todos los libros registrados
func manejarListar(gestor *GestorLibros) {
	fmt.Println("\n--- 📖 Libros Registrados ---")
	libros, err := gestor.ListarLibros()
	if err != nil {
		fmt.Printf("❌ Error al listar libros: %v\n", err)
		return
	}

	if len(libros) == 0 {
		fmt.Println("📭 No existen libros registrados.")
		return
	}

	for _, libro := range libros {
		fmt.Println(libro.String())
	}
	fmt.Printf("📊 Total de libros: %d\n", len(libros))
}

// manejarBuscar implementa búsqueda flexible con feedback detallado
func manejarBuscar(gestor *GestorLibros) {
	fmt.Println("\n--- 🔍 Buscar Libro ---")
	var criterio string
	fmt.Print("Ingrese título o autor a buscar: ")
	fmt.Scanln(&criterio)

	resultados, err := gestor.BuscarLibros(criterio)
	if err != nil {
		fmt.Printf("❌ Error en la búsqueda: %v\n", err)
		return
	}

	if len(resultados) == 0 {
		fmt.Println("📭 No se encontraron resultados para:", criterio)
		return
	}

	fmt.Printf("✅ Encontrados %d resultado(s):\n", len(resultados))
	for _, libro := range resultados {
		fmt.Println(libro.String())
	}
}

// manejarEliminar gestiona la eliminación con confirmación implícita
func manejarEliminar(gestor *GestorLibros) {
	fmt.Println("\n--- 🗑️ Eliminar Libro ---")
	var id int
	fmt.Print("Ingrese el ID del libro a eliminar: ")

	_, err := fmt.Scanln(&id)
	if err != nil {
		fmt.Println("❌ Error: ID debe ser un número entero.")
		limpiarBuffer()
		return
	}

	if err := gestor.EliminarLibro(id); err != nil {
		if errors.Is(err, ErrLibroNoEncontrado) {
			fmt.Printf("⚠️ No existe un libro con ID %d\n", id)
		} else {
			fmt.Printf("❌ Error al eliminar: %v\n", err)
		}
		return
	}
	fmt.Printf("✅ Libro con ID %d eliminado correctamente.\n", id)
}

// limpiarBuffer descarta entrada residual después de errores de escaneo
// Soluciona el problema común de Scanln que deja '\n' en el buffer
func limpiarBuffer() {
	var dummy string
	fmt.Scanln(&dummy)
}

