# TALLER DE AFIANZAMIENTO: ARQUITECTURA EMPRESARIAL Y TRANSICIÓN A FRAMEWORKS MODERNOS

## Integrantes:

- Deiby Fabian Prada Quintero
- Wilson Fernando Guelves Salazar
- Armando Rafael Trespalacios Lizcano
- Jair Fabian Duarte Villamizar

**Objetivo:** Consolidar conceptos de arquitectura empresarial Java EE y preparar la transición hacia PrimeFaces y Spring Boot

---

## PARTE I: REPASO Y CONSOLIDACIÓN DE CONCEPTOS (30 minutos)

### Actividad 1: Análisis Comparativo de Arquitecturas (15 minutos)

**Instrucciones:** Analice los siguientes tres escenarios y complete la tabla comparativa.



#### Escenario A: Sistema de Pedidos Online
- Múltiples usuarios hacen pedidos simultáneamente
- Cada pedido tiene múltiples ítems
- Se debe validar inventario y calcular total con impuestos
- Se registra auditoría de cada operación

#### Escenario B: Portal de Noticias
- Miles de usuarios leen noticias simultáneamente
- Pocos administradores publican contenido
- Las noticias rara vez cambian una vez publicadas
- Se necesita contar visitas por artículo

#### Escenario C: Sistema de Matrícula Universitaria
- Estudiantes se matriculan en un proceso de 5 pasos
- Pueden retroceder y cambiar selección de cursos
- Deben validar prerrequisitos y cupos disponibles
- Solo activo durante período de matrícula

**Complete la tabla:**

| Aspecto | Escenario A: Sistema de Pedidos Online | Escenario B: Portal de Noticias | Escenario C: Sistema de Matrícula Universitaria |
|---------|-----------------------------------------|--------------------------------|-----------------------------------------------|
| **Scope del Controller principal** | **@SessionScoped.** Cada cliente mantiene su propio carrito durante toda la sesión de compra, por lo que se requiere conservar el estado del pedido mientras el usuario interactúa. | **@ApplicationScoped.** La mayoría de operaciones son de solo lectura (consultar noticias). El contenido puede compartirse entre muchos usuarios sin información específica de sesión. | **@ViewScoped.** Permite mantener los datos durante los pasos de la matrícula (proceso de varios formularios). Este alcance conserva el estado mientras el usuario esté en la misma vista o flujo. *(Se aclara que este scope se investigó, ya que no se ha profundizado aún en clase, pero es el más apropiado para flujos de varios pasos.)* |
| **¿Necesita transacciones? ¿Por qué?** | **Sí.** Es necesario garantizar la integridad de los datos: si falla el cálculo del total o la validación de inventario, todo el proceso debe revertirse para mantener la consistencia de la base de datos (ACID). | **En su mayoría, no.** Las operaciones principales son lecturas de noticias (no requieren transacción). Solo las publicaciones o conteo de visitas necesitarían una transacción simple. | **Sí.** El proceso de matrícula implica múltiples pasos que deben registrarse como una única operación. Si falla la asignación de cupo o validación de prerrequisitos, debe revertirse toda la matrícula. |
| **¿Usar Facade? Justifique** | **Sí.** El *Facade* coordina la lógica de negocio del pedido: validaciones, cálculos de totales, descuentos y registro de auditoría. Simplifica la comunicación entre el Controller y el DAO. | **Opcional.** Puede emplearse un *Facade* para gestionar la publicación de noticias y el conteo de visitas, separando la lógica de negocio del acceso a datos. Sin embargo, la mayor parte son lecturas simples. | **Sí.** El *Facade* centraliza la lógica del proceso de matrícula: validación de prerrequisitos, disponibilidad de cupos y registro del estudiante, evitando lógica dispersa en los controladores. |
| **Principal desafío de concurrencia** | Controlar pedidos simultáneos sobre el mismo producto. Si varios usuarios compran al mismo tiempo, se debe evitar que el inventario quede en negativo. | Manejar lecturas concurrentes de muchos usuarios y escritura concurrente al incrementar el contador de visitas. | Evitar conflictos cuando varios estudiantes intentan matricularse simultáneamente en un mismo curso con cupos limitados. |
| **Patrón de persistencia más apropiado** | **DAO (Data Access Object).** Separa la lógica de negocio (en el Facade) del acceso a datos (en el DAO), reutilizando las conexiones gracias al *pool de conexiones*. Se investigó que también puede complementarse con JPA para control de versiones e integridad de datos. | **DAO con consultas optimizadas.** Ideal para operaciones de lectura frecuentes. Puede usarse junto a una caché de resultados para optimizar el rendimiento. | **DAO con manejo transaccional.** Centraliza las operaciones de inserción y actualización en el proceso de matrícula. Se investigó que en aplicaciones más complejas podría usarse un patrón *Repository* o JPA con control de concurrencia. |

> **Notas adicionales:**
> - Los *scopes* se seleccionaron con base en los vistos en clase (`@ApplicationScoped`, `@SessionScoped`, `@RequestScoped`, `@Dependent`, `@ViewScoped`). Solo el último fue investigado adicionalmente para ajustarse al flujo de matrícula.
> - El patrón *Facade* representa la capa de negocio que coordina la interacción entre el controlador y los DAOs.
> - El *DAO* fue el patrón de persistencia más utilizado y visto en clase; sin embargo, se investigaron complementos como JPA y Repository para comprender opciones más modernas de manejo de datos.
> - Se consideró el uso de *pool de conexiones* para mejorar el rendimiento y evitar saturación del servidor de base de datos.


### Actividad 2: Depuración de Código Problemático (15 minutos)

**Identifique y explique los problemas en el siguiente código:**

```java
// Código 1: Controller con problemas
//Con @RequestScoped crea una nueva instacia cada peticion del HTTP, esto crea el problema de que el contenido del carro, se pierde entre solicitudes.
@SessionScoped // @SessionScoped mantinene la informacion por la duracion de la sesion del usuario.
public class CarritoComprasController implements Serializable{ // Con @SessionScoped se necesita que la clase sea Serializable.
    
    @Inject
    private ProductoService productoService;
    
    private List<Producto> productosEnCarrito; // Problema?
    private double totalCompra;
    //ProductosEnCarrito no se inicializa antes de usar Add(), lo que lanza un error llamado NullPointerException.
    
    //Utilizamos un PostConstruct que es una anotacion de java que marca un método para que se ejecute automaticamente justo despues de que un bean ha sido instanciado y sus dependencias inyectadas, pero antes de que sea utilizado en la aplicacion.
    @PostConstruct
    public void init() {
        productosEnCarrito = new ArrayList<>();
        totalCompra = 0;
    }
    
    public void agregarProducto(String codigo) {
        Producto p = productoService.buscar(codigo);
        productosEnCarrito.add(p);
        totalCompra += p.getPrecio();
    }
    
    public String finalizarCompra() {
        // Procesar compra
        return "confirmacion";
    }

    // Getters y setters

}
```
### Explicación / Problemas identificados

* **Inicialización:** Originalmente `productosEnCarrito` no estaba inicializada. El `@PostConstruct` es el mecanismo correcto para inicializar colecciones en *beans* gestionados, pero es crucial **confirmar que el *bean* esté realmente gestionado** por CDI/JSF para que `@PostConstruct` se ejecute.
* **Serialización:** Al usar `@SessionScoped`, la clase debe implementar **`Serializable`** y **todos sus campos** deben ser serializables o declarados `transient` para asegurar la persistencia de la sesión (importante en despliegues en *cluster*).
* **Validaciones Ausentes:** No se valida si `productoService.buscar(codigo)` devuelve `null` ni si hay stock disponible. Llamar a `p.getPrecio()` sin chequear el objeto `p` puede causar un `NullPointerException` (NPE).
* **Fuente de Verdad del Precio:** Sumar `p.getPrecio()` directamente y de forma incremental puede ser inseguro si el precio se manipula en la interfaz de usuario. La fuente de verdad debe ser el **servicio/BD**.
* **Concurrencia de Sesión:** Aunque `@SessionScoped` implica una instancia por sesión, acciones simultáneas desde múltiples pestañas del navegador pueden provocar **condiciones de carrera** sobre la lista y el total.
* **Responsabilidad:** El *controller* está realizando manipulaciones de lista y cálculos. Parte de esta lógica debería delegarse a un `CartService` o `OrderFacade` para limpiar responsabilidades.

### Posibles mejoras

* Añadir verificación `if (p == null) { /* manejar producto no encontrado */ }` y chequear stock antes de añadir.
* Evitar actualizar `totalCompra` incrementalmente. En su lugar, usar un método **`recalcularTotal()`** que itere sobre la lista para evitar desincronización.
* Si existe riesgo de concurrencia, **proteger secciones críticas** (o sincronizar métodos) o rediseñar para evitar operaciones concurrentes en la misma sesión.
* Delegar el proceso de finalización (persistencia, validaciones, pago) a un **`OrderFacade`** que gestione transacciones y auditoría.
* Confirmar que todos los atributos sean `Serializable` para despliegue en *cluster* o considerar almacenamiento externo (DB/Redis) para el carrito si la sesión no es fiable.

###

```java
// Código 2: Service sin manejo transaccional
public class TransferenciaService {
    
    @Inject
    private CuentaDAO cuentaDAO;
    
    @Transactional //Si ocurre una excepción no controlada, el contenedor revierte todos los cambios
    public void transferir(String origen, String destino, double monto) 
            throws SQLException {
        Cuenta cuentaOrigen = cuentaDAO.buscar(origen);
        cuentaOrigen.setSaldo(cuentaOrigen.getSaldo() - monto);
        cuentaDAO.actualizar(cuentaOrigen);
        
        // ¿Qué pasa si falla aquí? 
        //Sin transacción, la operacion no es atómica ni segura devemos envolver el bloque completo en una @transactional para garantisar una consistencia ACID.
        Cuenta cuentaDestino = cuentaDAO.buscar(destino);
        cuentaDestino.setSaldo(cuentaDestino.getSaldo() + monto);
        cuentaDAO.actualizar(cuentaDestino);
    }
}
```
### Explicación / Problemas identificados

* **Ambigüedad en la Anotación Transaccional:** Es vital **confirmar la implementación** de `@Transactional` (Jakarta EE o Spring) para entender su comportamiento por defecto, especialmente ante **excepciones checked** como `SQLException`. Muchos *frameworks* no hacen *rollback* automáticamente en estos casos.
* **Rollback para Excepciones Checked:** Si el método declara `throws SQLException`, el *framework* podría no revertir la transacción. Es necesario configurar explícitamente `rollbackOn` o `rollbackFor = Exception.class` según el *stack* tecnológico para garantizar la **atomicidad**.
* **Validación de Saldo:** No se verifica si la cuenta origen tiene **saldo suficiente** antes de realizar el débito, lo que puede provocar sobregiros.
* **Condición de Carrera / Aislamiento:** Sin un mecanismo de bloqueo (p. ej., **`SELECT FOR UPDATE`**) o un nivel de aislamiento adecuado, dos operaciones simultáneas podrían causar inconsistencias (*lost update*).
* **Manejo de Excepciones:** Propagar `SQLException` a la capa superior sin control puede ser problemático. Es mejor envolverla en una **excepción de negocio** o *runtime* para una gestión de errores más limpia y para facilitar el *rollback* automático.

### Posibles mejoras 

* **Configurar Rollback:** Verificar la anotación transaccional y configurarla explícitamente para que haga *rollback* en las excepciones necesarias (ej. `@Transactional(rollbackFor = Exception.class)` en Spring).
* **Validación Previa:** **Validar condiciones previas** (saldo suficiente) antes de cualquier modificación y lanzar una excepción de negocio si no se cumple.
* **Control de Concurrencia:** Implementar un control de concurrencia en la capa DAO, ya sea con bloqueo pesimista (**`SELECT FOR UPDATE`**) o control optimista (versión y reintentos) para evitar *race conditions*.
* **Coordinación de Recursos:** Si la operación afecta múltiples recursos (BD + servicios externos), evaluar patrones como **SAGA** o **2PC/XA**.
* **Encapsulamiento de Excepciones:** Encapsular las excepciones de persistencia en excepciones de negocio o *runtime* adecuadas para garantizar el comportamiento consistente del *rollback* por parte del contenedor.

###

```java
// Código 3: JSP con lógica de negocio
<%
    //El código mezcla presentación (JSP) con lógica de negocio y acceso a datos.
    //Violación del patrón MVC: la JSP debería solo mostrar datos, no conectarse a la base.
    String cedula = request.getParameter("cedula");
    //DriverManager.getConnection() dentro del JSP crea una conexión por solicitud, lo cual es ineficiente.
    //No usa un pool de conexiones ni manejo centralizado de la base.
    Connection conn = DriverManager.getConnection("jdbc:mysql://...");
    PreparedStatement stmt = conn.prepareStatement("SELECT * FROM cliente WHERE cedula=?");
    stmt.setString(1, cedula);
    ResultSet rs = stmt.executeQuery();
    
    if(rs.next()) {
        double saldo = rs.getDouble("saldo");
        if(saldo > 1000000) {
            //out.println("Cliente VIP"); escribe directamente en la salida sin filtrar. En sistemas reales, debería mostrarse de forma controlada con etiquetas JSP o JSTL.
            out.println("Cliente VIP");
        }
        //No se capturan excepciones SQL ni se muestra un mensaje controlado. Si ocurre un error, la página se rompe.
        //Cualquier cambio en la lógica o consulta obliga a modificar la JSP. Debería usarse un servlet o clase DAO para gestionar el acceso a datos.
    }
%>
```
### Explicación / Problemas identificados

* **Mezcla de Capas (Violación MVC):** La vista (**JSP**) ejecuta directamente **acceso a la Base de Datos** (`Connection`, `DriverManager`) y contiene **lógica de negocio** (`if(saldo > 1000000)`). Esto rompe el patrón **MVC** (Modelo-Vista-Controlador), dificultando el mantenimiento y las pruebas.
* **Manejo de Recursos Deficiente:** Los objetos `Connection`, `PreparedStatement` y `ResultSet` **no se cierran**, lo que provoca **fugas de recursos** que pueden saturar el servidor de base de datos.
* **Rendimiento y Seguridad:** Se utiliza `DriverManager` en lugar de un **`DataSource` con *pool* de conexiones**, lo cual es **ineficiente y peligroso** en producción.
* **Salida Directa desde JSP:** El uso de `out.println` incrusta lógica de salida; la presentación debe limitarse a **JSTL/EL** y mostrar datos pre-procesados por el *controller*.
* **Testabilidad y Seguridad:** La lógica incrustada es difícil de probar y el manejo de excepciones no está centralizado.

### Posibles mejoras

* **Acceso a Datos:** Mover el acceso a BD a un **`ClienteDAO`** que use un **`DataSource`** (pool de conexiones) y la estructura **`try-with-resources`** para cerrar los recursos de forma segura.
* **Lógica de Negocio:** Crear un **`ClienteService`** que encapsule la regla de negocio (p. ej., determinar si el cliente es VIP).
* **Orquestación:** Usar un **`Servlet`** o **`Controller`** para recibir la cédula, **validarla**, invocar el `Service` y colocar el objeto cliente resultante como atributo de *request*.
* **Vista Pura:** La JSP solo debe usar **JSTL/EL** (Expression Language) para mostrar la información (ej. `${cliente.tipo}`), eliminando toda lógica y acceso a BD.
* **Manejo de Errores:** Asegurar el manejo de excepciones en la capa de servicio y *controller* para mostrar errores amigables en lugar de trazas al usuario.
---

**Preguntas:**

### 1. ¿Qué scope debería tener el CarritoComprasController? ¿Por qué?

* **Recomendación:** `@SessionScoped`.
* **Justificación:** El carrito debe **mantener el estado por usuario** entre múltiples peticiones durante la sesión.
    * `@RequestScoped` perdería el estado en cada petición.
    * `@ApplicationScoped` sería compartido por todos los usuarios (inadecuado).
* Se debe asegurar la serializabilidad de la clase. `@ViewScoped` es una alternativa para flujos cortos/multi-step.


### 2. ¿Qué le falta al TransferenciaService para garantizar atomicidad?

Le falta configurar y asegurar la **transaccionalidad declarativa correcta** (usar la anotación adecuada y configurar el `rollback` para excepciones *checked* si corresponde).

Además, requiere:
* **Validar condiciones previas** (saldo suficiente).
* Aplicar **control de concurrencia/aislamiento** en el DAO (como `SELECT FOR UPDATE` o control optimista).
* Encapsular/exponer las excepciones de forma controlada para que el contenedor pueda ejecutar el *rollback* correctamente.

---

### 3. ¿Qué violaciones arquitectónicas tiene el JSP? Proponga una solución.

| Aspecto | Violación | Solución Propuesta (Aplicar MVC) |
| :--- | :--- | :--- |
| **Separación de Capas** | **Mezcla** de presentación, lógica de negocio y acceso a datos en la vista. | Mover el acceso a BD a **`ClienteDAO`** y la lógica de negocio a **`ClienteService`**. |
| **Recursos y Rendimiento** | Uso de `DriverManager` en vez de `DataSource` y **recursos sin cerrar** (fugas). | Usar **`DataSource`** (pool de conexiones) y el patrón **`try-with-resources`** en el DAO. |
| **Control** | Baja testabilidad y gestión de errores descentralizada. | Usar un **`Servlet/Controller`** para orquestar y dejar la JSP únicamente para **presentación** con JSTL/EL. |
---
## PARTE II: EJERCICIO PRÁCTICO - REFACTORIZACIÓN (40 minutos)

### Ejercicio: Sistema de Reserva de Citas Médicas

**Contexto:** Tiene un sistema legacy con código mezclado. Debe refactorizarlo aplicando la arquitectura en capas correcta.

#### Código Legacy Proporcionado:

```java
// Todo en un solo servlet - MAL DISEÑO
@WebServlet("/reservarCita")
public class ReservaCitaServlet extends HttpServlet {
    
    protected void doPost(HttpServletRequest request, HttpServletResponse response) {
        try {
            // Obtener parámetros
            String cedulaPaciente = request.getParameter("cedula");
            String idMedico = request.getParameter("medico");
            String fecha = request.getParameter("fecha");
            String hora = request.getParameter("hora");
            
            // Conectar a BD directamente en el servlet
            Connection conn = DriverManager.getConnection(
                "jdbc:mysql://localhost/clinica", "root", "password");
            
            // Validar disponibilidad
            PreparedStatement ps1 = conn.prepareStatement(
                "SELECT COUNT(*) FROM citas WHERE id_medico=? AND fecha=? AND hora=?");
            ps1.setString(1, idMedico);
            ps1.setString(2, fecha);
            ps1.setString(3, hora);
            ResultSet rs = ps1.executeQuery();
            rs.next();
            
            if(rs.getInt(1) > 0) {
                response.getWriter().println("Horario no disponible");
                return;
            }
            
            // Insertar cita
            PreparedStatement ps2 = conn.prepareStatement(
                "INSERT INTO citas (cedula_paciente, id_medico, fecha, hora, estado) VALUES (?,?,?,?,?)");
            ps2.setString(1, cedulaPaciente);
            ps2.setString(2, idMedico);
            ps2.setString(3, fecha);
            ps2.setString(4, hora);
            ps2.setString(5, "RESERVADA");
            ps2.executeUpdate();
            
            // Calcular precio según especialidad
            PreparedStatement ps3 = conn.prepareStatement(
                "SELECT especialidad FROM medicos WHERE id=?");
            ps3.setString(1, idMedico);
            ResultSet rs2 = ps3.executeQuery();
            rs2.next();
            String especialidad = rs2.getString(1);
            
            double precio = 0;
            if(especialidad.equals("GENERAL")) precio = 50000;
            else if(especialidad.equals("ESPECIALISTA")) precio = 80000;
            
            response.getWriter().println("Cita reservada. Total: $" + precio);
            
        } catch(Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Tareas de Refactorización:

**1. Capa de Modelo (JavaBeans)** - Cree las clases necesarias:
   - `Cita` con todos sus atributos
   - `Medico` con especialidad
   - Incluya métodos de negocio donde corresponda

**2. Capa de Persistencia (DAO)** - Implemente:
   - `CitaDAO` con métodos: `guardar()`, `verificarDisponibilidad()`
   - `MedicoDAO` con método: `obtenerPorId()`
   - Use `@Resource` para el DataSource
   - Manejo apropiado de conexiones

**3. Capa de Servicio (Service)** - Implemente:
   - `CitaService` con método `reservarCita()`
   - Use `@Inject` para inyectar DAOs
   - Implemente manejo transaccional con `UserTransaction`
   - Método para calcular precio según especialidad

**4. Capa de Presentación (Managed Bean)** - Implemente:
   - `CitaController` con scope apropiado
   - `@Inject` del CitaService
   - Atributos del formulario con getters/setters
   - Método de acción `procesarReserva()`

**5. Patrón Facade (Opcional - Bonus)** - Implemente:
   - `ClinicaFacade` que coordine reservar cita Y enviar email de confirmación
   - Debe inyectar `CitaService` y `NotificacionService`

---

## 1. Capa de Modelo (JavaBeans)

Creamos las clases que representan las entidades principales del sistema.

```java
// Cita.java
public class Cita {
    private int id;
    private String cedulaPaciente;
    private String idMedico;
    private String fecha;
    private String hora;
    private String estado;

    public Cita() {}

    // Getters y Setters
    // ...
}
```

```java
// Medico.java
public class Medico {
    private String id;
    private String nombre;
    private String especialidad;

    public double calcularPrecio() {
        if ("GENERAL".equalsIgnoreCase(especialidad)) return 50000;
        else if ("ESPECIALISTA".equalsIgnoreCase(especialidad)) return 80000;
        return 60000;
    }

    // Getters y Setters
    // ...
}
```

---

## 2. Capa de Persistencia (DAO)

Los DAOs se encargan del acceso a la base de datos y del manejo de conexiones.

```java
// CitaDAO.java
import javax.annotation.Resource;
import javax.sql.DataSource;
import java.sql.*;

public class CitaDAO {

    @Resource(name = "jdbc/clinicaDS")
    private DataSource dataSource;

    public boolean verificarDisponibilidad(String idMedico, String fecha, String hora) throws SQLException {
        String sql = "SELECT COUNT(*) FROM citas WHERE id_medico=? AND fecha=? AND hora=?";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, idMedico);
            ps.setString(2, fecha);
            ps.setString(3, hora);
            ResultSet rs = ps.executeQuery();
            rs.next();
            return rs.getInt(1) == 0;
        }
    }

    public void guardar(Cita cita) throws SQLException {
        String sql = "INSERT INTO citas (cedula_paciente, id_medico, fecha, hora, estado) VALUES (?,?,?,?,?)";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, cita.getCedulaPaciente());
            ps.setString(2, cita.getIdMedico());
            ps.setString(3, cita.getFecha());
            ps.setString(4, cita.getHora());
            ps.setString(5, cita.getEstado());
            ps.executeUpdate();
        }
    }
}
```

```java
// MedicoDAO.java
import javax.annotation.Resource;
import javax.sql.DataSource;
import java.sql.*;

public class MedicoDAO {

    @Resource(name = "jdbc/clinicaDS")
    private DataSource dataSource;

    public Medico obtenerPorId(String id) throws SQLException {
        String sql = "SELECT * FROM medicos WHERE id=?";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, id);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                Medico m = new Medico();
                m.setId(rs.getString("id"));
                m.setNombre(rs.getString("nombre"));
                m.setEspecialidad(rs.getString("especialidad"));
                return m;
            }
            return null;
        }
    }
}
```

---

## 3. Capa de Servicio (Service)

Maneja la lógica del negocio y las transacciones.

```java
import javax.inject.Inject;
import javax.transaction.UserTransaction;

public class CitaService {

    @Inject
    private CitaDAO citaDAO;

    @Inject
    private MedicoDAO medicoDAO;

    @Inject
    private UserTransaction tx;

    public double reservarCita(String cedula, String idMedico, String fecha, String hora) throws Exception {
        tx.begin();
        try {
            if (!citaDAO.verificarDisponibilidad(idMedico, fecha, hora)) {
                throw new Exception("El horario no está disponible.");
            }

            Cita cita = new Cita();
            cita.setCedulaPaciente(cedula);
            cita.setIdMedico(idMedico);
            cita.setFecha(fecha);
            cita.setHora(hora);
            cita.setEstado("RESERVADA");

            citaDAO.guardar(cita);

            Medico medico = medicoDAO.obtenerPorId(idMedico);
            double precio = medico.calcularPrecio();

            tx.commit();
            return precio;

        } catch (Exception e) {
            tx.rollback();
            throw e;
        }
    }
}
```

---

## 4. Capa de Presentación (Managed Bean)

Recibe los datos del formulario y comunica la vista con el servicio.

```java
import javax.inject.Inject;
import javax.enterprise.context.RequestScoped;
import javax.inject.Named;

@Named
@RequestScoped
public class CitaController {

    @Inject
    private CitaService citaService;

    private String cedulaPaciente;
    private String idMedico;
    private String fecha;
    private String hora;
    private double precio;

    public void procesarReserva() {
        try {
            precio = citaService.reservarCita(cedulaPaciente, idMedico, fecha, hora);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // Getters y Setters
    // ...
}
```

---

## 5. (Bonus) Patrón Facade

Coordina la reserva y el envío de confirmaciones al paciente.

```java
import javax.inject.Inject;

public class ClinicaFacade {

    @Inject
    private CitaService citaService;

    @Inject
    private NotificacionService notificacionService;

    public void reservarYNotificar(String cedula, String idMedico, String fecha, String hora) throws Exception {
        double precio = citaService.reservarCita(cedula, idMedico, fecha, hora);
        notificacionService.enviarEmail(cedula, "Cita reservada. Total a pagar: $" + precio);
    }
}
```

---

## Conclusión

**Antes:** todo el código estaba en un único servlet, difícil de mantener.  
**Después:** se aplicó una arquitectura en capas clara:

| Capa | Responsabilidad |
|------|------------------|
| **Modelo (JavaBeans)** | Representa los datos (Cita, Medico). |
| **DAO** | Se comunica con la base de datos. |
| **Service** | Contiene la lógica del negocio y maneja transacciones. |
| **Controller (Managed Bean)** | Recibe datos del usuario y llama a los servicios. |
| **Facade (opcional)** | Coordina procesos grandes como notificaciones o auditorías. |

Esta separación mejora la **mantenibilidad, reutilización y claridad** del código.

---

## PARTE III: TRANSICIÓN A FRAMEWORKS MODERNOS (40 minutos)

### Sección A: Conceptos Puente (20 minutos)

#### Actividad 3: Tabla Comparativa de Inyección de Dependencias

Complete la siguiente tabla comparativa entre Java EE CDI, Spring y PrimeFaces:

| Concepto | Java EE / CDI | Spring / Spring Boot | PrimeFaces + CDI |
|----------|---------------|----------------------|------------------|
| Anotación para inyección | `@Inject` | | |
| Anotación para bean de sesión | `@SessionScoped` | | |
| Anotación para bean de request | `@RequestScoped` | | |
| Anotación para singleton | `@ApplicationScoped` | | |
| Componente de servicio | `@Stateless` o CDI bean | | N/A |
| Transacciones declarativas | `@Transactional` (JTA) | | |
| Configuración | `beans.xml` | | |
| Managed Bean para vista | CDI Managed Bean | | |

**Investigue y complete:**
- Spring Boot usa `@Autowired` o `@Inject` para inyección
- Spring Boot scope equivalente: `@RequestScope`, `@SessionScope`, `@ApplicationScope`
- Spring Boot usa `@Service` para servicios
- PrimeFaces trabaja con los mismos managed beans CDI de JSF
- PrimeFaces scope especial: `@ViewScoped` (de JSF)

#### Actividad 4: De Java EE a Spring Boot

**Dado este código Java EE CDI:**

```java
@Named
@SessionScoped
public class ProductoController implements Serializable {
    
    @Inject
    private ProductoService productoService;
    
    private List<Producto> productos;
    
    @PostConstruct
    public void init() {
        productos = productoService.listarTodos();
    }
    
    // getters y setters
}

@Stateless
public class ProductoService {
    
    @Inject
    private ProductoDAO productoDAO;
    
    public List<Producto> listarTodos() {
        return productoDAO.findAll();
    }
}
```

**Conviértalo a Spring Boot:**

```java
// Complete el código Spring Boot equivalente
@Controller
@SessionAttributes("productos")
public class ProductoController {
    
    // ¿Qué anotación usar para inyección?
    private ProductoService productoService;
    
    // ¿Cómo manejar la inicialización?
    
    
    // Complete los métodos
}

// ¿Qué anotación usar para el servicio?
public class ProductoService {
    
    // ¿Cómo inyectar el repository?
    private ProductoRepository productoRepository;
    
    public List<Producto> listarTodos() {
        // En Spring Boot con JPA
        return productoRepository.findAll();
    }
}
```

**Preguntas guía:**
1. ¿Qué equivale `@Inject` en Spring?
2. ¿Qué equivale `@Stateless` en Spring?
3. ¿Cómo se declara un Repository en Spring Boot?
4. ¿Qué ventajas tiene Spring Boot sobre Java EE tradicional?

### Sección B: Introducción a PrimeFaces (20 minutos)

#### Actividad 5: Comparación JSP vs PrimeFaces

**Analice estos dos códigos equivalentes:**

**JSP Tradicional:**
```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>

<form action="guardarProducto" method="post">
    <label>Nombre:</label>
    <input type="text" name="nombre" value="${producto.nombre}"/>
    
    <label>Precio:</label>
    <input type="number" name="precio" value="${producto.precio}"/>
    
    <input type="submit" value="Guardar"/>
</form>

<table>
    <tr>
        <th>Código</th>
        <th>Nombre</th>
        <th>Precio</th>
    </tr>
    <c:forEach items="${productos}" var="p">
    <tr>
        <td>${p.codigo}</td>
        <td>${p.nombre}</td>
        <td>${p.precio}</td>
    </tr>
    </c:forEach>
</table>
```

**PrimeFaces (XHTML con JSF):**
```xhtml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="http://xmlns.jcp.org/jsf/html"
      xmlns:p="http://primefaces.org/ui">
<h:head>
    <title>Productos</title>
</h:head>
<h:body>
    <h:form>
        <p:panelGrid columns="2">
            <p:outputLabel value="Nombre:"/>
            <p:inputText value="#{productoController.producto.nombre}"/>
            
            <p:outputLabel value="Precio:"/>
            <p:inputNumber value="#{productoController.producto.precio}"/>
        </p:panelGrid>
        
        <p:commandButton value="Guardar" 
                         action="#{productoController.guardar}"
                         update="tablaProductos"/>
    </h:form>
    
    <p:dataTable id="tablaProductos" 
                 value="#{productoController.productos}" 
                 var="p">
        <p:column headerText="Código">
            <h:outputText value="#{p.codigo}"/>
        </p:column>
        <p:column headerText="Nombre">
            <h:outputText value="#{p.nombre}"/>
        </p:column>
        <p:column headerText="Precio">
            <h:outputText value="#{p.precio}"/>
        </p:column>
    </p:dataTable>
</h:body>
</html>
```

**Responda:**
1. ¿Qué ventajas observa en PrimeFaces sobre JSP tradicional?
2. ¿Cómo se vinculan los componentes PrimeFaces con el Managed Bean?
3. ¿Qué es `update="tablaProductos"` y qué tecnología usa (AJAX)?
4. ¿El Managed Bean cambia al usar PrimeFaces? ¿Por qué?

#### Actividad 6: Componentes Avanzados PrimeFaces

**Investigue y describa casos de uso para estos componentes PrimeFaces:**

1. **`<p:dataTable>` con LazyDataModel**
   - ¿Cuándo usar paginación lazy?
   - ¿Qué ventaja tiene sobre cargar toda la lista?

2. **`<p:dialog>`**
   - ¿Cómo mejora la UX comparado con páginas separadas?
   - ¿Se requiere JavaScript manual?

3. **`<p:autoComplete>`**
   - ¿Cómo se implementa el método de búsqueda en el bean?
   - ¿Es necesario AJAX?

4. **`<p:fileUpload>`**
   - ¿Cómo se procesa el archivo en el Managed Bean?
   - ¿Dónde se guardaría en la arquitectura en capas?

---

## PARTE IV: CASO INTEGRADOR FINAL (10 minutos)

### Diseño de Arquitectura Completa

**Escenario:** Sistema de E-commerce con carrito de compras, gestión de inventario y procesamiento de pagos.

**Tarea:** Diseñe la arquitectura completa especificando:

**Capas y componentes:**
   - Entidades (JavaBeans)
   - DAOs
   - Services
   - Facades (si aplica)
   - Controllers
   - Vistas (JSP/PrimeFaces)

  ### Presentación (JSF / PrimeFaces)
- **Vistas**: JSF + PrimeFaces.
  - `catalogo.xhtml` — listado paginado de productos (lazy load).
  - `producto.xhtml` — detalle del producto .
  - `carrito.xhtml` — vista del carrito .
  - `checkout.xhtml` — wizard checkout multipaso .
  - `login.xhtml`, `perfil.xhtml`, `pedidos.xhtml`.

### Controladores (Managed Beans / Controllers)
- **LoginController** — autenticación, creación de sesión, logout.
- **ProductoCatalogoController** — búsqueda, filtros, paginación, lazy loading.
- **ProductoDetalleController** — ver detalles, añadir al carrito desde detalle.
- **CarritoComprasController** — añadir/quitar ítems, cambiar cantidades, calcular totales.
- **CheckoutMultipasoController** — control del wizard paso a paso, validaciones del pedido, invocar servicio de pago.
- **PedidoController** — ver historial, estatus del pedido.
- **AdminProductosController** — CRUD de productos e inventario.

### Services (lógica de negocio)
- **ProductService**
  - `findProducts`, `findById`, `updateStock` *(transaccional)*
- **CartService**
  - `getCartForUser`, `addItem`, `updateItemQuantity`
- **OrderService**
  - `createOrder`, `cancelOrder` *(transaccional)*
- **PaymentService**
  - `processPayment`, `capturePayment`
- **InventoryService**
  - `checkAvailability`, `decrementStock`, `incrementStock` *(transaccional)*

### DAOs / Repositorios
- **ProductRepository**, **UserRepository**, **OrderRepository**, **PaymentRepository**, **InventoryRepository**
- Implementación: JPA/Hibernate con `@Entity` JavaBeans.

### Entidades (JavaBeans / JPA Entities)
- **User**, **Product**, **Cart**, **Order**, **OrderItem**, **Payment**, **InventoryMovement**.

### Facades 
- **EcommerceFacade**: orquesta operaciones combinadas entre servicios.
     

2. **Scopes de cada Managed Bean:**
   - `LoginController`: ¿?
   - `ProductoCatalogoController`: ¿?
   - `CarritoComprasController`: ¿?
   - `CheckoutMultipasoController`: ¿?

| Bean | Scope | Justificación |
|------|--------|----------------|
| `LoginController` | `@SessionScoped` | Mantiene sesión del usuario. |
| `ProductoCatalogoController` | `@ViewScoped` | Conserva filtros y paginación. |
| `CarritoComprasController` | `@SessionScoped` | Carrito de usuario activo en sesión. |
| `CheckoutMultipasoController` | `@ViewScoped` o `@SessionScoped` | Según si el wizard está en una o varias vistas. |

3. **Puntos donde se requieren transacciones:**
   - Liste las operaciones que deben ser transaccionales

    1. **Crear pedido + decrementar inventario + registrar pago.**
    2. **Captura de pago y cambio de estado del pedido.**
    3. **Cancelar pedido y reembolsar.**
    4. **Reservar stock al iniciar checkout (opcional).**
    5. **Ajustes o transferencias de inventario.**
    6. **CRUD crítico como precios, promociones.**

4. **Transición a tecnologías modernas:**
   - ¿Qué cambiaría al migrar a Spring Boot?
   
        - `@ManagedBean` → `@RestController` / `@Controller`
        - `@Service` y `@Repository` reemplazan a DAOs.
        - `@Transactional` en servicios críticos.
        - `Spring Security` para autenticación.
        - Persistencia con `Spring Data JPA`.
        - Configuración externa en `application.yml`.
  
   - ¿Qué componentes PrimeFaces usaría en el catálogo?
     
        - `p:dataTable` (lazy load, filtros)
        - `p:carousel`, `p:galleria` (banners)
        - `p:dialog` (vista rápida)
        - `p:wizard` (checkout multipaso)
        - `p:growl`, `p:confirmDialog`
        - `p:card`, `p:panelGrid` (tarjetas de producto)
        - `p:ajax` (actualizaciones parciales)
  
   - ¿Cómo implementaría el carrito con AJAX de PrimeFaces?

 ```xhtml
<p:commandButton value="Añadir"
                 action="#{productoCatalogoController.addToCart(product.id)}"
                 process="@this"
                 update=":header:cartBadge :growl" />

<h:form id="header">
  <p:commandLink update=":header:cartPanel">
    <p:badge value="#{carritoComprasController.itemCount}" />
  </p:commandLink>
</h:form>

<h:form id="cartForm">
  <p:panel id="cartPanel">
    <ui:repeat value="#{carritoComprasController.items}" var="item">
      <h:outputText value="#{item.productName} - #{item.qty}" />
      <p:commandButton value="+" action="#{carritoComprasController.increment(item.productId)}" update="cartPanel cartBadge" />
      <p:commandButton value="-" action="#{carritoComprasController.decrement(item.productId)}" update="cartPanel cartBadge" />
    </ui:repeat>
  </p:panel>
</h:form>
```
---

## RECURSOS COMPLEMENTARIOS

### Para profundizar en PrimeFaces:
- PrimeFaces Showcase: https://www.primefaces.org/showcase/
- Tutoriales de DataTable con LazyDataModel
- Componentes AJAX y actualización parcial

### Para profundizar en Spring Boot:
- Spring Initializr para crear proyectos
- Comparación: `@Inject` vs `@Autowired`
- Spring Data JPA vs DAOs manuales
- `@RestController` para APIs REST

### Conceptos clave para próximas clases:
- **PrimeFaces**: LazyDataModel, AJAX, diálogos modales, FileUpload
- **Spring Boot**: Anotaciones (`@Service`, `@Repository`, `@Autowired`), Spring Data JPA, configuración con `application.properties`
- **Arquitectura moderna**: Microservicios, APIs REST, separación frontend/backend

---

## ENTREGABLE

**Documento PDF o MD con:**
1. Tablas completadas de la Parte I
2. Análisis de problemas y soluciones propuestas
3. Código refactorizado del ejercicio práctico (Parte II)
4. Tablas comparativas completadas (Parte III)
5. Diseño de arquitectura del caso integrador (Parte IV)

**Fecha de entrega:** 31 octubre 2025

---

**FIN DEL TALLER**
