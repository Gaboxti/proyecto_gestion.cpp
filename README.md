CODIGO
/**
 * SISTEMA DE GESTIÓN EMPRESARIAL - VERSIÓN 2.0
 * Desarrollado en C++ con soporte C++11/14
 * 
 * Módulos:
 * - Gestión de Usuarios y Trabajadores
 * - Sistema de Pagos Bancarios
 * - Gestión de Cuentas
 * - Reportes y Estadísticas
 * - Backup y Restauración
 * - Licencias y Configuración
 * 
 * Autor: [Tu Nombre]
 * Fecha: 2026
 * Licencia: MIT
 */

#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <limits>
#include <cctype>
#include <fstream>
#include <sstream>
#include <unordered_map>
#include <iomanip>
#include <ctime>
#include <map>
#include <functional>

#ifdef _WIN32
#include <conio.h>
#include <windows.h>
#else
#include <unistd.h>
#endif

using namespace std;

// ==================== CONSTANTES GLOBALES ====================
const string MODULO_TRABAJADORES = "Trabajadores";
const string MODULO_PAGOS = "Pagos";
const string MODULO_CUENTAS = "Cuentas";
const string MODULO_SISTEMA = "Sistema";
const string MODULO_DATOS = "Datos";
const string MODULO_REPORTES = "Reportes";
const string MODULO_BACKUP = "Backup";

const string USUARIO_UNIVERSAL = "admin";
const string CLAVE_UNIVERSAL = "admin123";

// ==================== ESTRUCTURAS DE DATOS ====================

struct Fecha {
    int dia, mes, anio;
    Fecha(int d = 1, int m = 1, int a = 2026) : dia(d), mes(m), anio(a) {}
    
    string toString() const {
        stringstream ss;
        ss << setw(2) << setfill('0') << dia << "/"
           << setw(2) << setfill('0') << mes << "/"
           << anio;
        return ss.str();
    }
};

struct DatosBancarios {
    string banco;
    string numeroCuenta;
    string tipoCuenta; // Ahorros, Corriente
    double saldo;
    
    string toString() const {
        return banco + " - " + numeroCuenta + " (" + tipoCuenta + ")";
    }
};

struct Trabajador {
    int id;
    string nombre;
    string apellido;
    string cedula;
    string telefono;
    string correo;
    string cargo;
    string departamento;
    double salario;
    string estado; // Activo, Inactivo, Vacaciones
    Fecha fechaIngreso;
    DatosBancarios datosBancarios;
    
    string nombreCompleto() const {
        return nombre + " " + apellido;
    }
};

struct Usuario {
    int id;
    string nombre;
    string nombusuario;
    int cedula;
    string cargo;
    string gerencia;
    vector<string> modulos;
    string contrasena;
    bool esAdministradorUniversal;
    bool activo;
    bool requiereCambioContrasena;
    string fechaCreacion;
    string ultimoAcceso;
    string email;
    string telefono;
};

struct Cuenta {
    int id;
    string banco;
    string tipo;
    string titular;
    string administrador;
    int cedulaAdministrador;
    double saldo;
    string estado; // Activa, Inactiva, Bloqueada
    string fechaApertura;
};

struct Pago {
    int id;
    int idTrabajador;
    string nombreTrabajador;
    double monto;
    string fecha;
    string tipo; // Nomina, Bono, Vacaciones, Extra
    string estado; // Pendiente, Pagado, Cancelado
    string metodo; // Transferencia, Cheque, Efectivo
    string banco;
    string cuentaDestino;
    string referencia;
};

struct Producto {
    int id;
    string nombre;
    string descripcion;
    double costo;
    double precioVenta;
    int stock;
    double iva; // Porcentaje
};

struct Licencia {
    int id;
    string software;
    string version;
    string clave;
    string fechaExpiracion;
    int usuariosPermitidos;
    int usuariosActivos;
    string proveedor;
    double costoAnual;
    string estado; // Activa, Expirada, Suspendida
};

struct Configuracion {
    string nombreEmpresa;
    string rif;
    string direccion;
    string telefono;
    string email;
    string moneda;
    int diaPago;
    double iva;
    string backupAutomatico;
    int maxIntentosLogin;
};

// ==================== VARIABLES GLOBALES ====================

vector<Usuario> usuariosRegistrados;
vector<Trabajador> trabajadoresRegistrados;
vector<Cuenta> cuentasRegistradas;
vector<Pago> pagosRegistrados;
vector<Producto> productosRegistrados;
vector<Licencia> licenciasRegistradas;
Configuracion configSistema;

// Variables de sesión
string usuarioActual = "";
string nombreUsuarioActual = "";
bool esAdministradorUniversal = false;
vector<string> modulosUsuarioActual;
int idUsuarioActual = -1;

// Contadores para IDs
int nextIdTrabajador = 1;
int nextIdUsuario = 1;
int nextIdPago = 1;
int nextIdCuenta = 1;
int nextIdProducto = 1;
int nextIdLicencia = 1;

// Mapas para búsqueda rápida
unordered_map<int, Trabajador*> mapaTrabajadoresPorId;
unordered_map<string, Trabajador*> mapaTrabajadoresPorCedula;
unordered_map<int, Usuario*> mapaUsuariosPorId;
unordered_map<string, Usuario*> mapaUsuariosPorNombre;
unordered_map<int, Cuenta*> mapaCuentasPorId;
unordered_map<int, Pago*> mapaPagosPorId;

// Usuario administrador universal predefinido
const Usuario USUARIO_ADMIN_UNIVERSAL = {
    0, "Administrador Universal", "admin", 99999999,
    "Administrador del Sistema", "Gerencia General",
    {MODULO_TRABAJADORES, MODULO_PAGOS, MODULO_CUENTAS,
     MODULO_SISTEMA, MODULO_DATOS, MODULO_REPORTES, MODULO_BACKUP},
    "admin123", true, true, false,
    "2024-01-01", "", "admin@empresa.com", "555-0100"
};

// Configuración por defecto
const Configuracion CONFIG_DEFAULT = {
    "Mi Empresa C.A.", "J-12345678-9",
    "Av. Principal, Edificio Central, Piso 1",
    "0212-555-0100", "info@miempresa.com",
    "Bs", 30, 16.0, "23:00", 3
};

// ==================== FUNCIONES AUXILIARES ====================

void limpiarPantalla() {
#ifdef _WIN32
    system("cls");
#else
    system("clear");
#endif
}

int leerEntero() {
    int valor;
    while (!(cin >> valor)) {
        cin.clear();
        cin.ignore(numeric_limits<streamsize>::max(), '\n');
        cout << "Entrada inválida. Ingrese un número: ";
    }
    cin.ignore(numeric_limits<streamsize>::max(), '\n');
    return valor;
}

double leerDouble() {
    double valor;
    while (!(cin >> valor)) {
        cin.clear();
        cin.ignore(numeric_limits<streamsize>::max(), '\n');
        cout << "Entrada inválida. Ingrese un número: ";
    }
    cin.ignore(numeric_limits<streamsize>::max(), '\n');
    return valor;
}

string leerLinea() {
    string texto;
    getline(cin, texto);
    return texto;
}

void pausar() {
    cout << "\nPresione Enter para continuar...";
    cin.get();
}

void mostrarError(const string& mensaje) {
    cout << "\n❌ ERROR: " << mensaje << endl;
    pausar();
}

void mostrarExito(const string& mensaje) {
    cout << "\n✅ " << mensaje << endl;
    pausar();
}

string obtenerFechaActual() {
    time_t now = time(0);
    tm* timeinfo = localtime(&now);
    char buffer[80];
    strftime(buffer, sizeof(buffer), "%Y-%m-%d", timeinfo);
    return string(buffer);
}

string obtenerFechaHoraActual() {
    time_t now = time(0);
    tm* timeinfo = localtime(&now);
    char buffer[80];
    strftime(buffer, sizeof(buffer), "%Y-%m-%d %H:%M:%S", timeinfo);
    return string(buffer);
}

bool validarCedula(const string& cedula) {
    if (cedula.length() < 7 || cedula.length() > 10) return false;
    for (char c : cedula) {
        if (!isdigit(c)) return false;
    }
    return true;
}

bool validarCorreo(const string& correo) {
    return correo.find('@') != string::npos && 
           correo.find('.') != string::npos;
}

bool validarTelefono(const string& telefono) {
    for (char c : telefono) {
        if (!isdigit(c) && c != '+' && c != '-' && c != ' ') 
            return false;
    }
    return telefono.length() >= 10;
}

void aMayusculas(string& str) {
    transform(str.begin(), str.end(), str.begin(), ::toupper);
}

void inicializarMapas() {
    mapaTrabajadoresPorId.clear();
    mapaTrabajadoresPorCedula.clear();
    mapaUsuariosPorId.clear();
    mapaUsuariosPorNombre.clear();
    mapaCuentasPorId.clear();
    mapaPagosPorId.clear();
    
    for (auto& t : trabajadoresRegistrados) {
        mapaTrabajadoresPorId[t.id] = &t;
        mapaTrabajadoresPorCedula[t.cedula] = &t;
    }
    
    for (auto& u : usuariosRegistrados) {
        mapaUsuariosPorId[u.id] = &u;
        mapaUsuariosPorNombre[u.nombusuario] = &u;
    }
    
    for (auto& c : cuentasRegistradas) {
        mapaCuentasPorId[c.id] = &c;
    }
    
    for (auto& p : pagosRegistrados) {
        mapaPagosPorId[p.id] = &p;
    }
}

void resetearSesion() {
    usuarioActual = "";
    nombreUsuarioActual = "";
    esAdministradorUniversal = false;
    modulosUsuarioActual.clear();
    idUsuarioActual = -1;
}

// ==================== PERSISTENCIA DE DATOS ====================

void guardarDatos() {
    ofstream archivo("datos_sistema.dat", ios::binary);
    if (!archivo) {
        mostrarError("No se pudo guardar los datos");
        return;
    }
    
    // Guardar configuración
    size_t size = configSistema.nombreEmpresa.size();
    archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
    archivo.write(configSistema.nombreEmpresa.c_str(), size);
    
    size = configSistema.rif.size();
    archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
    archivo.write(configSistema.rif.c_str(), size);
    
    size = configSistema.direccion.size();
    archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
    archivo.write(configSistema.direccion.c_str(), size);
    
    size = configSistema.telefono.size();
    archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
    archivo.write(configSistema.telefono.c_str(), size);
    
    size = configSistema.email.size();
    archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
    archivo.write(configSistema.email.c_str(), size);
    
    size = configSistema.moneda.size();
    archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
    archivo.write(configSistema.moneda.c_str(), size);
    
    archivo.write(reinterpret_cast<char*>(&configSistema.diaPago), 
                  sizeof(configSistema.diaPago));
    archivo.write(reinterpret_cast<char*>(&configSistema.iva), 
                  sizeof(configSistema.iva));
    
    size = configSistema.backupAutomatico.size();
    archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
    archivo.write(configSistema.backupAutomatico.c_str(), size);
    
    archivo.write(reinterpret_cast<char*>(&configSistema.maxIntentosLogin), 
                  sizeof(configSistema.maxIntentosLogin));
    
    // Guardar trabajadores
    size_t numTrabajadores = trabajadoresRegistrados.size();
    archivo.write(reinterpret_cast<char*>(&numTrabajadores), sizeof(numTrabajadores));
    for (const auto& t : trabajadoresRegistrados) {
        archivo.write(reinterpret_cast<const char*>(&t.id), sizeof(t.id));
        
        size = t.nombre.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.nombre.c_str(), size);
        
        size = t.apellido.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.apellido.c_str(), size);
        
        size = t.cedula.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.cedula.c_str(), size);
        
        size = t.telefono.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.telefono.c_str(), size);
        
        size = t.correo.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.correo.c_str(), size);
        
        size = t.cargo.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.cargo.c_str(), size);
        
        size = t.departamento.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.departamento.c_str(), size);
        
        archivo.write(reinterpret_cast<const char*>(&t.salario), sizeof(t.salario));
        
        size = t.estado.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.estado.c_str(), size);
        
        // Guardar fecha de ingreso
        archivo.write(reinterpret_cast<const char*>(&t.fechaIngreso.dia), 
                      sizeof(t.fechaIngreso.dia));
        archivo.write(reinterpret_cast<const char*>(&t.fechaIngreso.mes), 
                      sizeof(t.fechaIngreso.mes));
        archivo.write(reinterpret_cast<const char*>(&t.fechaIngreso.anio), 
                      sizeof(t.fechaIngreso.anio));
        
        // Guardar datos bancarios
        size = t.datosBancarios.banco.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.datosBancarios.banco.c_str(), size);
        
        size = t.datosBancarios.numeroCuenta.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.datosBancarios.numeroCuenta.c_str(), size);
        
        size = t.datosBancarios.tipoCuenta.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(t.datosBancarios.tipoCuenta.c_str(), size);
        
        archivo.write(reinterpret_cast<const char*>(&t.datosBancarios.saldo), 
                      sizeof(t.datosBancarios.saldo));
    }
    
    // Guardar usuarios (excluyendo el admin universal que se recrea)
    size_t numUsuarios = usuariosRegistrados.size();
    archivo.write(reinterpret_cast<char*>(&numUsuarios), sizeof(numUsuarios));
    for (const auto& u : usuariosRegistrados) {
        archivo.write(reinterpret_cast<const char*>(&u.id), sizeof(u.id));
        
        size = u.nombre.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(u.nombre.c_str(), size);
        
        size = u.nombusuario.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(u.nombusuario.c_str(), size);
        
        archivo.write(reinterpret_cast<const char*>(&u.cedula), sizeof(u.cedula));
        
        size = u.cargo.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(u.cargo.c_str(), size);
        
        size = u.gerencia.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(u.gerencia.c_str(), size);
        
        // Guardar módulos
        size_t numModulos = u.modulos.size();
        archivo.write(reinterpret_cast<char*>(&numModulos), sizeof(numModulos));
        for (const auto& m : u.modulos) {
            size = m.size();
            archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
            archivo.write(m.c_str(), size);
        }
        
        size = u.contrasena.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(u.contrasena.c_str(), size);
        
        archivo.write(reinterpret_cast<const char*>(&u.esAdministradorUniversal), 
                      sizeof(u.esAdministradorUniversal));
        archivo.write(reinterpret_cast<const char*>(&u.activo), sizeof(u.activo));
        archivo.write(reinterpret_cast<const char*>(&u.requiereCambioContrasena), 
                      sizeof(u.requiereCambioContrasena));
        
        size = u.fechaCreacion.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(u.fechaCreacion.c_str(), size);
        
        size = u.ultimoAcceso.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(u.ultimoAcceso.c_str(), size);
        
        size = u.email.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(u.email.c_str(), size);
        
        size = u.telefono.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(u.telefono.c_str(), size);
    }
    
    // Guardar cuentas
    size_t numCuentas = cuentasRegistradas.size();
    archivo.write(reinterpret_cast<char*>(&numCuentas), sizeof(numCuentas));
    for (const auto& c : cuentasRegistradas) {
        archivo.write(reinterpret_cast<const char*>(&c.id), sizeof(c.id));
        
        size = c.banco.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(c.banco.c_str(), size);
        
        size = c.tipo.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(c.tipo.c_str(), size);
        
        size = c.titular.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(c.titular.c_str(), size);
        
        size = c.administrador.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(c.administrador.c_str(), size);
        
        archivo.write(reinterpret_cast<const char*>(&c.cedulaAdministrador), 
                      sizeof(c.cedulaAdministrador));
        archivo.write(reinterpret_cast<const char*>(&c.saldo), sizeof(c.saldo));
        
        size = c.estado.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(c.estado.c_str(), size);
        
        size = c.fechaApertura.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(c.fechaApertura.c_str(), size);
    }
    
    // Guardar pagos
    size_t numPagos = pagosRegistrados.size();
    archivo.write(reinterpret_cast<char*>(&numPagos), sizeof(numPagos));
    for (const auto& p : pagosRegistrados) {
        archivo.write(reinterpret_cast<const char*>(&p.id), sizeof(p.id));
        archivo.write(reinterpret_cast<const char*>(&p.idTrabajador), 
                      sizeof(p.idTrabajador));
        
        size = p.nombreTrabajador.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(p.nombreTrabajador.c_str(), size);
        
        archivo.write(reinterpret_cast<const char*>(&p.monto), sizeof(p.monto));
        
        size = p.fecha.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(p.fecha.c_str(), size);
        
        size = p.tipo.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(p.tipo.c_str(), size);
        
        size = p.estado.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(p.estado.c_str(), size);
        
        size = p.metodo.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(p.metodo.c_str(), size);
        
        size = p.banco.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(p.banco.c_str(), size);
        
        size = p.cuentaDestino.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(p.cuentaDestino.c_str(), size);
        
        size = p.referencia.size();
        archivo.write(reinterpret_cast<char*>(&size), sizeof(size));
        archivo.write(p.referencia.c_str(), size);
    }
    
    // Guardar IDs
    archivo.write(reinterpret_cast<char*>(&nextIdTrabajador), sizeof(nextIdTrabajador));
    archivo.write(reinterpret_cast<char*>(&nextIdUsuario), sizeof(nextIdUsuario));
    archivo.write(reinterpret_cast<char*>(&nextIdPago), sizeof(nextIdPago));
    archivo.write(reinterpret_cast<char*>(&nextIdCuenta), sizeof(nextIdCuenta));
    archivo.write(reinterpret_cast<char*>(&nextIdProducto), sizeof(nextIdProducto));
    archivo.write(reinterpret_cast<char*>(&nextIdLicencia), sizeof(nextIdLicencia));
    
    archivo.close();
    cout << "\n💾 Datos guardados correctamente." << endl;
}

void cargarDatos() {
    ifstream archivo("datos_sistema.dat", ios::binary);
    if (!archivo) {
        // Primera ejecución - usar datos por defecto
        configSistema = CONFIG_DEFAULT;
        usuariosRegistrados.push_back(USUARIO_ADMIN_UNIVERSAL);
        return;
    }
    
    // Cargar configuración
    size_t size;
    archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
    configSistema.nombreEmpresa.resize(size);
    archivo.read(&configSistema.nombreEmpresa[0], size);
    
    archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
    configSistema.rif.resize(size);
    archivo.read(&configSistema.rif[0], size);
    
    archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
    configSistema.direccion.resize(size);
    archivo.read(&configSistema.direccion[0], size);
    
    archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
    configSistema.telefono.resize(size);
    archivo.read(&configSistema.telefono[0], size);
    
    archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
    configSistema.email.resize(size);
    archivo.read(&configSistema.email[0], size);
    
    archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
    configSistema.moneda.resize(size);
    archivo.read(&configSistema.moneda[0], size);
    
    archivo.read(reinterpret_cast<char*>(&configSistema.diaPago), 
                 sizeof(configSistema.diaPago));
    archivo.read(reinterpret_cast<char*>(&configSistema.iva), 
                 sizeof(configSistema.iva));
    
    archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
    configSistema.backupAutomatico.resize(size);
    archivo.read(&configSistema.backupAutomatico[0], size);
    
    archivo.read(reinterpret_cast<char*>(&configSistema.maxIntentosLogin), 
                 sizeof(configSistema.maxIntentosLogin));
    
    // Cargar trabajadores
    size_t numTrabajadores;
    archivo.read(reinterpret_cast<char*>(&numTrabajadores), sizeof(numTrabajadores));
    trabajadoresRegistrados.clear();
    for (size_t i = 0; i < numTrabajadores; i++) {
        Trabajador t;
        archivo.read(reinterpret_cast<char*>(&t.id), sizeof(t.id));
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.nombre.resize(size);
        archivo.read(&t.nombre[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.apellido.resize(size);
        archivo.read(&t.apellido[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.cedula.resize(size);
        archivo.read(&t.cedula[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.telefono.resize(size);
        archivo.read(&t.telefono[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.correo.resize(size);
        archivo.read(&t.correo[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.cargo.resize(size);
        archivo.read(&t.cargo[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.departamento.resize(size);
        archivo.read(&t.departamento[0], size);
        
        archivo.read(reinterpret_cast<char*>(&t.salario), sizeof(t.salario));
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.estado.resize(size);
        archivo.read(&t.estado[0], size);
        
        archivo.read(reinterpret_cast<char*>(&t.fechaIngreso.dia), 
                     sizeof(t.fechaIngreso.dia));
        archivo.read(reinterpret_cast<char*>(&t.fechaIngreso.mes), 
                     sizeof(t.fechaIngreso.mes));
        archivo.read(reinterpret_cast<char*>(&t.fechaIngreso.anio), 
                     sizeof(t.fechaIngreso.anio));
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.datosBancarios.banco.resize(size);
        archivo.read(&t.datosBancarios.banco[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.datosBancarios.numeroCuenta.resize(size);
        archivo.read(&t.datosBancarios.numeroCuenta[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        t.datosBancarios.tipoCuenta.resize(size);
        archivo.read(&t.datosBancarios.tipoCuenta[0], size);
        
        archivo.read(reinterpret_cast<char*>(&t.datosBancarios.saldo), 
                     sizeof(t.datosBancarios.saldo));
        
        trabajadoresRegistrados.push_back(t);
    }
    
    // Cargar usuarios
    size_t numUsuarios;
    archivo.read(reinterpret_cast<char*>(&numUsuarios), sizeof(numUsuarios));
    usuariosRegistrados.clear();
    usuariosRegistrados.push_back(USUARIO_ADMIN_UNIVERSAL);
    
    for (size_t i = 0; i < numUsuarios; i++) {
        Usuario u;
        archivo.read(reinterpret_cast<char*>(&u.id), sizeof(u.id));
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        u.nombre.resize(size);
        archivo.read(&u.nombre[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        u.nombusuario.resize(size);
        archivo.read(&u.nombusuario[0], size);
        
        archivo.read(reinterpret_cast<char*>(&u.cedula), sizeof(u.cedula));
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        u.cargo.resize(size);
        archivo.read(&u.cargo[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        u.gerencia.resize(size);
        archivo.read(&u.gerencia[0], size);
        
        size_t numModulos;
        archivo.read(reinterpret_cast<char*>(&numModulos), sizeof(numModulos));
        u.modulos.clear();
        for (size_t j = 0; j < numModulos; j++) {
            archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
            string modulo;
            modulo.resize(size);
            archivo.read(&modulo[0], size);
            u.modulos.push_back(modulo);
        }
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        u.contrasena.resize(size);
        archivo.read(&u.contrasena[0], size);
        
        archivo.read(reinterpret_cast<char*>(&u.esAdministradorUniversal), 
                     sizeof(u.esAdministradorUniversal));
        archivo.read(reinterpret_cast<char*>(&u.activo), sizeof(u.activo));
        archivo.read(reinterpret_cast<char*>(&u.requiereCambioContrasena), 
                     sizeof(u.requiereCambioContrasena));
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        u.fechaCreacion.resize(size);
        archivo.read(&u.fechaCreacion[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        u.ultimoAcceso.resize(size);
        archivo.read(&u.ultimoAcceso[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        u.email.resize(size);
        archivo.read(&u.email[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        u.telefono.resize(size);
        archivo.read(&u.telefono[0], size);
        
        if (u.nombusuario != "admin") {
            usuariosRegistrados.push_back(u);
        }
    }
    
    // Cargar cuentas
    size_t numCuentas;
    archivo.read(reinterpret_cast<char*>(&numCuentas), sizeof(numCuentas));
    cuentasRegistradas.clear();
    for (size_t i = 0; i < numCuentas; i++) {
        Cuenta c;
        archivo.read(reinterpret_cast<char*>(&c.id), sizeof(c.id));
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        c.banco.resize(size);
        archivo.read(&c.banco[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        c.tipo.resize(size);
        archivo.read(&c.tipo[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        c.titular.resize(size);
        archivo.read(&c.titular[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        c.administrador.resize(size);
        archivo.read(&c.administrador[0], size);
        
        archivo.read(reinterpret_cast<char*>(&c.cedulaAdministrador), 
                     sizeof(c.cedulaAdministrador));
        archivo.read(reinterpret_cast<char*>(&c.saldo), sizeof(c.saldo));
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        c.estado.resize(size);
        archivo.read(&c.estado[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        c.fechaApertura.resize(size);
        archivo.read(&c.fechaApertura[0], size);
        
        cuentasRegistradas.push_back(c);
    }
    
    // Cargar pagos
    size_t numPagos;
    archivo.read(reinterpret_cast<char*>(&numPagos), sizeof(numPagos));
    pagosRegistrados.clear();
    for (size_t i = 0; i < numPagos; i++) {
        Pago p;
        archivo.read(reinterpret_cast<char*>(&p.id), sizeof(p.id));
        archivo.read(reinterpret_cast<char*>(&p.idTrabajador), 
                     sizeof(p.idTrabajador));
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        p.nombreTrabajador.resize(size);
        archivo.read(&p.nombreTrabajador[0], size);
        
        archivo.read(reinterpret_cast<char*>(&p.monto), sizeof(p.monto));
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        p.fecha.resize(size);
        archivo.read(&p.fecha[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        p.tipo.resize(size);
        archivo.read(&p.tipo[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        p.estado.resize(size);
        archivo.read(&p.estado[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        p.metodo.resize(size);
        archivo.read(&p.metodo[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        p.banco.resize(size);
        archivo.read(&p.banco[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        p.cuentaDestino.resize(size);
        archivo.read(&p.cuentaDestino[0], size);
        
        archivo.read(reinterpret_cast<char*>(&size), sizeof(size));
        p.referencia.resize(size);
        archivo.read(&p.referencia[0], size);
        
        pagosRegistrados.push_back(p);
    }
    
    // Cargar IDs
    archivo.read(reinterpret_cast<char*>(&nextIdTrabajador), sizeof(nextIdTrabajador));
    archivo.read(reinterpret_cast<char*>(&nextIdUsuario), sizeof(nextIdUsuario));
    archivo.read(reinterpret_cast<char*>(&nextIdPago), sizeof(nextIdPago));
    archivo.read(reinterpret_cast<char*>(&nextIdCuenta), sizeof(nextIdCuenta));
    archivo.read(reinterpret_cast<char*>(&nextIdProducto), sizeof(nextIdProducto));
    archivo.read(reinterpret_cast<char*>(&nextIdLicencia), sizeof(nextIdLicencia));
    
    archivo.close();
    inicializarMapas();
    cout << "\n📂 Datos cargados correctamente." << endl;
}

// ==================== AUTENTICACIÓN ====================

bool autenticarUsuario() {
    limpiarPantalla();
    string username, password;
    
    cout << "=========================================\n";
    cout << "   SISTEMA DE GESTIÓN EMPRESARIAL\n";
    cout << "=========================================\n\n";
    cout << "Usuario: ";
    cin >> username;
    cout << "Contraseña: ";
    cin >> password;
    
    // Verificar usuario universal
    if (username == USUARIO_UNIVERSAL && password == CLAVE_UNIVERSAL) {
        usuarioActual = username;
        nombreUsuarioActual = "Administrador Universal";
        esAdministradorUniversal = true;
        modulosUsuarioActual = USUARIO_ADMIN_UNIVERSAL.modulos;
        idUsuarioActual = 0;
        cout << "\n✅ Bienvenido, Administrador Universal\n";
        return true;
    }
    
    // Verificar otros usuarios
    for (auto& u : usuariosRegistrados) {
        if (u.nombusuario == username && u.contrasena == password && u.activo) {
            usuarioActual = username;
            nombreUsuarioActual = u.nombre;
            esAdministradorUniversal = u.esAdministradorUniversal;
            modulosUsuarioActual = u.modulos;
            idUsuarioActual = u.id;
            u.ultimoAcceso = obtenerFechaHoraActual();
            cout << "\n✅ Bienvenido, " << u.nombre << "\n";
            return true;
        }
    }
    
    cout << "\n❌ Usuario o contraseña incorrectos\n";
    pausar();
    return false;
}

// ==================== GESTIÓN DE TRABAJADORES ====================

void mostrarTrabajadores() {
    limpiarPantalla();
    cout << "\n=== LISTA DE TRABAJADORES ===\n";
    cout << "==============================\n\n";
    
    if (trabajadoresRegistrados.empty()) {
        cout << "No hay trabajadores registrados.\n";
        pausar();
        return;
    }
    
    cout << left << setw(5) << "ID"
         << setw(20) << "NOMBRE"
         << setw(15) << "CÉDULA"
         << setw(15) << "CARGO"
         << setw(12) << "SALARIO"
         << setw(10) << "ESTADO" << endl;
    cout << string(80, '-') << endl;
    
    for (const auto& t : trabajadoresRegistrados) {
        cout << left << setw(5) << t.id
             << setw(20) << (t.nombre + " " + t.apellido).substr(0, 18)
             << setw(15) << t.cedula
             << setw(15) << t.cargo.substr(0, 13)
             << setw(12) << fixed << setprecision(2) << t.salario
             << setw(10) << t.estado << endl;
    }
    
    cout << "\nTotal: " << trabajadoresRegistrados.size() << " trabajadores\n";
    pausar();
}

void agregarTrabajador() {
    limpiarPantalla();
    cout << "\n=== AGREGAR NUEVO TRABAJADOR ===\n";
    cout << "=================================\n\n";
    
    Trabajador t;
    t.id = nextIdTrabajador++;
    
    cout << "Nombre: ";
    t.nombre = leerLinea();
    
    cout << "Apellido: ";
    t.apellido = leerLinea();
    
    do {
        cout << "Cédula: ";
        t.cedula = leerLinea();
        if (!validarCedula(t.cedula)) {
            cout << "❌ Cédula inválida (debe tener 7-10 dígitos). Intente de nuevo.\n";
        } else if (mapaTrabajadoresPorCedula.find(t.cedula) != mapaTrabajadoresPorCedula.end()) {
            cout << "❌ Ya existe un trabajador con esa cédula. Intente de nuevo.\n";
            t.cedula = "";
        }
    } while (t.cedula.empty());
    
    do {
        cout << "Teléfono: ";
        t.telefono = leerLinea();
        if (!validarTelefono(t.telefono)) {
            cout << "❌ Teléfono inválido. Intente de nuevo.\n";
        }
    } while (t.telefono.empty());
    
    do {
        cout << "Correo: ";
        t.correo = leerLinea();
        if (!validarCorreo(t.correo)) {
            cout << "❌ Correo inválido. Intente de nuevo.\n";
        }
    } while (t.correo.empty());
    
    cout << "Cargo: ";
    t.cargo = leerLinea();
    
    cout << "Departamento: ";
    t.departamento = leerLinea();
    
    cout << "Salario mensual: ";
    t.salario = leerDouble();
    
    t.estado = "Activo";
    
    cout << "\n--- DATOS BANCARIOS ---\n";
    cout << "Banco: ";
    t.datosBancarios.banco = leerLinea();
    
    cout << "Número de cuenta: ";
    t.datosBancarios.numeroCuenta = leerLinea();
    
    cout << "Tipo de cuenta (Ahorros/Corriente): ";
    t.datosBancarios.tipoCuenta = leerLinea();
    
    t.datosBancarios.saldo = 0.0;
    
    // Fecha de ingreso (actual por defecto)
    time_t now = time(0);
    tm* timeinfo = localtime(&now);
    t.fechaIngreso = Fecha(timeinfo->tm_mday, 
                           timeinfo->tm_mon + 1, 
                           timeinfo->tm_year + 1900);
    
    trabajadoresRegistrados.push_back(t);
    mapaTrabajadoresPorId[t.id] = &trabajadoresRegistrados.back();
    mapaTrabajadoresPorCedula[t.cedula] = &trabajadoresRegistrados.back();
    
    cout << "\n✅ Trabajador agregado exitosamente con ID: " << t.id << "\n";
    pausar();
}

void buscarTrabajador() {
    limpiarPantalla();
    cout << "\n=== BUSCAR TRABAJADOR ===\n";
    cout << "1. Buscar por ID\n";
    cout << "2. Buscar por Cédula\n";
    cout << "3. Buscar por Nombre\n";
    cout << "Seleccione una opción: ";
    
    int opcion = leerEntero();
    
    if (opcion == 1) {
        cout << "Ingrese ID: ";
        int id = leerEntero();
        
        auto it = mapaTrabajadoresPorId.find(id);
        if (it != mapaTrabajadoresPorId.end()) {
            Trabajador* t = it->second;
            cout << "\n=== TRABAJADOR ENCONTRADO ===\n";
            cout << "ID: " << t->id << "\n";
            cout << "Nombre: " << t->nombreCompleto() << "\n";
            cout << "Cédula: " << t->cedula << "\n";
            cout << "Teléfono: " << t->telefono << "\n";
            cout << "Correo: " << t->correo << "\n";
            cout << "Cargo: " << t->cargo << "\n";
            cout << "Departamento: " << t->departamento << "\n";
            cout << "Salario: " << configSistema.moneda << " " 
                 << fixed << setprecision(2) << t->salario << "\n";
            cout << "Estado: " << t->estado << "\n";
            cout << "Fecha Ingreso: " << t->fechaIngreso.toString() << "\n";
            cout << "Datos Bancarios: " << t->datosBancarios.toString() << "\n";
        } else {
            cout << "❌ No se encontró trabajador con ID: " << id << "\n";
        }
    } else if (opcion == 2) {
        cout << "Ingrese cédula: ";
        string cedula = leerLinea();
        
        auto it = mapaTrabajadoresPorCedula.find(cedula);
        if (it != mapaTrabajadoresPorCedula.end()) {
            Trabajador* t = it->second;
            cout << "\n=== TRABAJADOR ENCONTRADO ===\n";
            cout << "ID: " << t->id << "\n";
            cout << "Nombre: " << t->nombreCompleto() << "\n";
            // ... mostrar demás datos
        } else {
            cout << "❌ No se encontró trabajador con cédula: " << cedula << "\n";
        }
    } else if (opcion == 3) {
        cout << "Ingrese nombre a buscar: ";
        string nombreBuscar = leerLinea();
        aMayusculas(nombreBuscar);
        
        bool encontrado = false;
        for (const auto& t : trabajadoresRegistrados) {
            string nombreCompleto = t.nombre + " " + t.apellido;
            aMayusculas(nombreCompleto);
            if (nombreCompleto.find(nombreBuscar) != string::npos) {
                if (!encontrado) {
                    cout << "\n=== RESULTADOS DE BÚSQUEDA ===\n";
                    encontrado = true;
                }
                cout << "ID: " << t.id << " | " << t.nombreCompleto() 
                     << " | " << t.cedula << " | " << t.cargo << "\n";
            }
        }
        
        if (!encontrado) {
            cout << "❌ No se encontraron trabajadores con ese nombre.\n";
        }
    }
    
    pausar();
}

void editarTrabajador() {
    limpiarPantalla();
    cout << "\n=== EDITAR TRABAJADOR ===\n";
    mostrarTrabajadores();
    
    cout << "\nIngrese ID del trabajador a editar (0 para cancelar): ";
    int id = leerEntero();
    
    if (id == 0) return;
    
    auto it = mapaTrabajadoresPorId.find(id);
    if (it != mapaTrabajadoresPorId.end()) {
        Trabajador* t = it->second;
        
        cout << "\nEditando: " << t->nombreCompleto() << "\n";
        cout << "Deje en blanco para mantener el valor actual.\n\n";
        
        string entrada;
        
        cout << "Nombre [" << t->nombre << "]: ";
        entrada = leerLinea();
        if (!entrada.empty()) t->nombre = entrada;
        
        cout << "Apellido [" << t->apellido << "]: ";
        entrada = leerLinea();
        if (!entrada.empty()) t->apellido = entrada;
        
        cout << "Teléfono [" << t->telefono << "]: ";
        entrada = leerLinea();
        if (!entrada.empty()) t->telefono = entrada;
        
        cout << "Correo [" << t->correo << "]: ";
        entrada = leerLinea();
        if (!entrada.empty()) t->correo = entrada;
        
        cout << "Cargo [" << t->cargo << "]: ";
        entrada = leerLinea();
        if (!entrada.empty()) t->cargo = entrada;
        
        cout << "Departamento [" << t->departamento << "]: ";
        entrada = leerLinea();
        if (!entrada.empty()) t->departamento = entrada;
        
        cout << "Salario [" << t->salario << "]: ";
        entrada = leerLinea();
        if (!entrada.empty()) t->salario = stod(entrada);
        
        cout << "Estado [" << t->estado << "] (Activo/Inactivo/Vacaciones): ";
        entrada = leerLinea();
        if (!entrada.empty()) t->estado = entrada;
        
        cout << "\n✅ Trabajador actualizado.\n";
    } else {
        cout << "❌ No se encontró trabajador con ID: " << id << "\n";
    }
    
    pausar();
}

void eliminarTrabajador() {
    limpiarPantalla();
    cout << "\n=== ELIMINAR TRABAJADOR ===\n";
    mostrarTrabajadores();
    
    cout << "\nIngrese ID del trabajador a eliminar (0 para cancelar): ";
    int id = leerEntero();
    
    if (id == 0) return;
    
    auto it = mapaTrabajadoresPorId.find(id);
    if (it != mapaTrabajadoresPorId.end()) {
        Trabajador* t = it->second;
        
        cout << "\n¿Está seguro de eliminar a " << t->nombreCompleto() << "? (s/n): ";
        char conf;
        cin >> conf;
        cin.ignore();
        
        if (tolower(conf) == 's') {
            // Eliminar del mapa
            mapaTrabajadoresPorCedula.erase(t->cedula);
            
            // Eliminar del vector
            for (auto itv = trabajadoresRegistrados.begin(); 
                 itv != trabajadoresRegistrados.end(); ++itv) {
                if (itv->id == id) {
                    trabajadoresRegistrados.erase(itv);
                    break;
                }
            }
            
            mapaTrabajadoresPorId.erase(id);
            cout << "✅ Trabajador eliminado.\n";
        } else {
            cout << "Operación cancelada.\n";
        }
    } else {
        cout << "❌ No se encontró trabajador con ID: " << id << "\n";
    }
    
    pausar();
}

void gestionTrabajadores() {
    int opcion;
    do {
        limpiarPantalla();
        cout << "\n=== GESTIÓN DE TRABAJADORES ===\n";
        cout << "================================\n";
        cout << "1. Ver todos los trabajadores\n";
        cout << "2. Agregar trabajador\n";
        cout << "3. Buscar trabajador\n";
        cout << "4. Editar trabajador\n";
        cout << "5. Eliminar trabajador\n";
        cout << "6. Volver al menú principal\n";
        cout << "Seleccione una opción: ";
        
        opcion = leerEntero();
        
        switch (opcion) {
            case 1: mostrarTrabajadores(); break;
            case 2: agregarTrabajador(); break;
            case 3: buscarTrabajador(); break;
            case 4: editarTrabajador(); break;
            case 5: eliminarTrabajador(); break;
            case 6: cout << "Volviendo...\n"; break;
            default: mostrarError("Opción inválida");
        }
    } while (opcion != 6);
}

// ==================== GESTIÓN DE PAGOS BANCARIOS ====================

void mostrarPagos() {
    limpiarPantalla();
    cout << "\n=== HISTORIAL DE PAGOS ===\n";
    cout << "===========================\n\n";
    
    if (pagosRegistrados.empty()) {
        cout << "No hay pagos registrados.\n";
        pausar();
        return;
    }
    
    cout << left << setw(5) << "ID"
         << setw(20) << "TRABAJADOR"
         << setw(12) << "MONTO"
         << setw(12) << "FECHA"
         << setw(10) << "BANCO"
         << setw(10) << "ESTADO" << endl;
    cout << string(70, '-') << endl;
    
    for (const auto& p : pagosRegistrados) {
        cout << left << setw(5) << p.id
             << setw(20) << p.nombreTrabajador.substr(0, 18)
             << setw(12) << fixed << setprecision(2) << p.monto
             << setw(12) << p.fecha
             << setw(10) << p.banco.substr(0, 8)
             << setw(10) << p.estado << endl;
    }
    
    // Calcular totales
    double totalPagado = 0;
    double totalPendiente = 0;
    for (const auto& p : pagosRegistrados) {
        if (p.estado == "Pagado") totalPagado += p.monto;
        else if (p.estado == "Pendiente") totalPendiente += p.monto;
    }
    
    cout << "\nTotales:\n";
    cout << "💰 Pagado: " << configSistema.moneda << " " 
         << fixed << setprecision(2) << totalPagado << "\n";
    cout << "⏳ Pendiente: " << configSistema.moneda << " " << totalPendiente << "\n";
    
    pausar();
}

void registrarPago() {
    limpiarPantalla();
    cout << "\n=== REGISTRAR NUEVO PAGO ===\n";
    cout << "=============================\n\n";
    
    if (trabajadoresRegistrados.empty()) {
        cout << "❌ No hay trabajadores registrados.\n";
        pausar();
        return;
    }
    
    // Mostrar trabajadores activos
    cout << "TRABAJADORES ACTIVOS:\n";
    for (const auto& t : trabajadoresRegistrados) {
        if (t.estado == "Activo") {
            cout << "ID: " << t.id << " | " << t.nombreCompleto() 
                 << " | Cédula: " << t.cedula << "\n";
        }
    }
    
    Pago p;
    p.id = nextIdPago++;
    p.estado = "Pendiente";
    p.fecha = obtenerFechaActual();
    
    cout << "\nID del trabajador: ";
    p.idTrabajador = leerEntero();
    
    // Buscar trabajador
    auto it = mapaTrabajadoresPorId.find(p.idTrabajador);
    if (it == mapaTrabajadoresPorId.end()) {
        cout << "❌ Trabajador no encontrado.\n";
        pausar();
        return;
    }
    
    Trabajador* t = it->second;
    p.nombreTrabajador = t->nombreCompleto();
    p.cuentaDestino = t->datosBancarios.numeroCuenta;
    p.banco = t->datosBancarios.banco;
    
    cout << "Trabajador: " << p.nombreTrabajador << "\n";
    cout << "Banco: " << p.banco << "\n";
    cout << "Cuenta: " << p.cuentaDestino << "\n\n";
    
    cout << "Tipo de pago (Nómina/Bono/Vacaciones/Extra): ";
    p.tipo = leerLinea();
    
    cout << "Monto a pagar: ";
    p.monto = leerDouble();
    
    cout << "Método de pago (Transferencia/Cheque/Efectivo): ";
    p.metodo = leerLinea();
    
    // Generar referencia única
    stringstream ss;
    ss << "PAG-" << t->cedula << "-" << time(nullptr);
    p.referencia = ss.str();
    
    char conf;
    cout << "\n¿Confirmar pago? (s/n): ";
    cin >> conf;
    cin.ignore();
    
    if (tolower(conf) == 's') {
        p.estado = "Pagado";
        pagosRegistrados.push_back(p);
        mapaPagosPorId[p.id] = &pagosRegistrados.back();
        
        cout << "\n✅ Pago registrado exitosamente.\n";
        cout << "Referencia: " << p.referencia << "\n";
        cout << "Monto: " << configSistema.moneda << " " << p.monto << "\n";
    } else {
        cout << "\nPago cancelado.\n";
    }
    
    pausar();
}

void procesarPagosPorBanco() {
    limpiarPantalla();
    cout << "\n=== PROCESAR PAGOS POR BANCO ===\n";
    cout << "=================================\n\n";
    
    // Obtener pagos pendientes
    vector<Pago> pagosPendientes;
    for (const auto& p : pagosRegistrados) {
        if (p.estado == "Pendiente") {
            pagosPendientes.push_back(p);
        }
    }
    
    if (pagosPendientes.empty()) {
        cout << "No hay pagos pendientes por procesar.\n";
        pausar();
        return;
    }
    
    // Agrupar por banco
    map<string, vector<Pago>> pagosPorBanco;
    for (const auto& p : pagosPendientes) {
        pagosPorBanco[p.banco].push_back(p);
    }
    
    cout << "Procesando pagos agrupados por banco...\n\n";
    
    // Procesar cada banco
    for (auto& [banco, lista] : pagosPorBanco) {
        cout << "🏦 BANCO: " << banco << "\n";
        cout << string(40, '-') << "\n";
        
        // Ordenar por cédula (asumiendo que el nombre incluye cédula o por ID)
        sort(lista.begin(), lista.end(), 
             [](const Pago& a, const Pago& b) {
                 return a.idTrabajador < b.idTrabajador;
             });
        
        double totalBanco = 0;
        for (auto& p : lista) {
            cout << "  - " << p.nombreTrabajador 
                 << ": " << configSistema.moneda << " " 
                 << fixed << setprecision(2) << p.monto << "\n";
            totalBanco += p.monto;
            
            // Marcar como pagado
            for (auto& pr : pagosRegistrados) {
                if (pr.id == p.id) {
                    pr.estado = "Pagado";
                    break;
                }
            }
        }
        cout << "  TOTAL BANCO: " << configSistema.moneda << " " 
             << fixed << setprecision(2) << totalBanco << "\n\n";
    }
    
    cout << "✅ Todos los pagos han sido procesados.\n";
    
    // Generar reporte
    stringstream reporte;
    reporte << "=== REPORTE DE PAGOS ===\n";
    reporte << "Fecha: " << obtenerFechaHoraActual() << "\n\n";
    
    for (const auto& [banco, lista] : pagosPorBanco) {
        reporte << "Banco: " << banco << "\n";
        double total = 0;
        for (const auto& p : lista) {
            reporte << "  " << p.nombreTrabajador << " - "
                    << configSistema.moneda << " " << p.monto << "\n";
            total += p.monto;
        }
        reporte << "  Total: " << configSistema.moneda << " " << total << "\n\n";
    }
    
    // Guardar reporte
    string nombreArchivo = "reporte_pagos_" + obtenerFechaActual() + ".txt";
    ofstream archivo(nombreArchivo);
    if (archivo) {
        archivo << reporte.str();
        archivo.close();
        cout << "📄 Reporte guardado en: " << nombreArchivo << "\n";
    }
    
    pausar();
}

void buscarPagosPorTrabajador() {
    limpiarPantalla();
    cout << "\n=== BUSCAR PAGOS POR TRABAJADOR ===\n";
    cout << "====================================\n\n";
    
    if (trabajadoresRegistrados.empty()) {
        cout << "No hay trabajadores registrados.\n";
        pausar();
        return;
    }
    
    // Mostrar lista de trabajadores
    for (const auto& t : trabajadoresRegistrados) {
        cout << "ID: " << t.id << " | " << t.nombreCompleto() 
             << " | Cédula: " << t.cedula << "\n";
    }
    
    cout << "\nIngrese ID del trabajador: ";
    int id = leerEntero();
    
    auto it = mapaTrabajadoresPorId.find(id);
    if (it == mapaTrabajadoresPorId.end()) {
        cout << "❌ Trabajador no encontrado.\n";
        pausar();
        return;
    }
    
    Trabajador* t = it->second;
    
    cout << "\n=== PAGOS DE " << t->nombreCompleto() << " ===\n";
    
    double total = 0;
    bool hayPagos = false;
    
    for (const auto& p : pagosRegistrados) {
        if (p.idTrabajador == id) {
            hayPagos = true;
            cout << "\nFecha: " << p.fecha << "\n";
            cout << "Tipo: " << p.tipo << "\n";
            cout << "Monto: " << configSistema.moneda << " " 
                 << fixed << setprecision(2) << p.monto << "\n";
            cout << "Banco: " << p.banco << "\n";
            cout << "Referencia: " << p.referencia << "\n";
            cout << "Estado: " << p.estado << "\n";
            total += p.monto;
        }
    }
    
    if (!hayPagos) {
        cout << "No hay pagos registrados para este trabajador.\n";
    } else {
        cout << "\n💰 TOTAL PAGADO: " << configSistema.moneda << " " 
             << fixed << setprecision(2) << total << "\n";
    }
    
    pausar();
}

void gestionPagos() {
    int opcion;
    do {
        limpiarPantalla();
        cout << "\n=== GESTIÓN DE PAGOS ===\n";
        cout << "=========================\n";
        cout << "1. Ver historial de pagos\n";
        cout << "2. Registrar nuevo pago\n";
        cout << "3. Procesar pagos por banco\n";
        cout << "4. Buscar pagos por trabajador\n";
        cout << "5. Volver al menú principal\n";
        cout << "Seleccione una opción: ";
        
        opcion = leerEntero();
        
        switch (opcion) {
            case 1: mostrarPagos(); break;
            case 2: registrarPago(); break;
            case 3: procesarPagosPorBanco(); break;
            case 4: buscarPagosPorTrabajador(); break;
            case 5: cout << "Volviendo...\n"; break;
            default: mostrarError("Opción inválida");
        }
    } while (opcion != 5);
}

// ==================== GESTIÓN DE CUENTAS ====================

void mostrarCuentas() {
    limpiarPantalla();
    cout << "\n=== LISTA DE CUENTAS ===\n";
    cout << "=========================\n\n";
    
    if (cuentasRegistradas.empty()) {
        cout << "No hay cuentas registradas.\n";
        pausar();
        return;
    }
    
    cout << left << setw(5) << "ID"
         << setw(15) << "BANCO"
         << setw(20) << "TITULAR"
         << setw(15) << "TIPO"
         << setw(12) << "SALDO"
         << setw(10) << "ESTADO" << endl;
    cout << string(80, '-') << endl;
    
    double totalSaldos = 0;
    for (const auto& c : cuentasRegistradas) {
        cout << left << setw(5) << c.id
             << setw(15) << c.banco.substr(0, 13)
             << setw(20) << c.titular.substr(0, 18)
             << setw(15) << c.tipo
             << setw(12) << fixed << setprecision(2) << c.saldo
             << setw(10) << c.estado << endl;
        totalSaldos += c.saldo;
    }
    
    cout << "\n💰 TOTAL SALDOS: " << configSistema.moneda << " " 
         << fixed << setprecision(2) << totalSaldos << "\n";
    
    pausar();
}

void agregarCuenta() {
    limpiarPantalla();
    cout << "\n=== AGREGAR NUEVA CUENTA ===\n";
    cout << "=============================\n\n";
    
    Cuenta c;
    c.id = nextIdCuenta++;
    c.fechaApertura = obtenerFechaActual();
    c.estado = "Activa";
    
    cout << "Banco: ";
    c.banco = leerLinea();
    
    cout << "Tipo de cuenta (Ahorros/Corriente): ";
    c.tipo = leerLinea();
    
    cout << "Titular de la cuenta: ";
    c.titular = leerLinea();
    
    cout << "Administrador de la cuenta: ";
    c.administrador = leerLinea();
    
    cout << "Cédula del administrador: ";
    c.cedulaAdministrador = leerEntero();
    
    cout << "Saldo inicial: ";
    c.saldo = leerDouble();
    
    cuentasRegistradas.push_back(c);
    mapaCuentasPorId[c.id] = &cuentasRegistradas.back();
    
    cout << "\n✅ Cuenta agregada exitosamente con ID: " << c.id << "\n";
    pausar();
}

void editarCuenta() {
    limpiarPantalla();
    cout << "\n=== EDITAR CUENTA ===\n";
    mostrarCuentas();
    
    cout << "\nIngrese ID de la cuenta a editar (0 para cancelar): ";
    int id = leerEntero();
    
    if (id == 0) return;
    
    auto it = mapaCuentasPorId.find(id);
    if (it != mapaCuentasPorId.end()) {
        Cuenta* c = it->second;
        
        cout << "\nEditando cuenta de: " << c->titular << "\n";
        cout << "Deje en blanco para mantener el valor actual.\n\n";
        
        string entrada;
        
        cout << "Banco [" << c->banco << "]: ";
        entrada = leerLinea();
        if (!entrada.empty()) c->banco = entrada;
        
        cout << "Titular [" << c->titular << "]: ";
        entrada = leerLinea();
        if (!entrada.empty()) c->titular = entrada;
        
        cout << "Tipo de cuenta [" << c->tipo << "]: ";
        entrada = leerLinea();
        if (!entrada.empty()) c->tipo = entrada;
        
        cout << "Estado [" << c->estado << "] (Activa/Inactiva/Bloqueada): ";
        entrada = leerLinea();
        if (!entrada.empty()) c->estado = entrada;
        
        cout << "\n✅ Cuenta actualizada.\n";
    } else {
        cout << "❌ No se encontró cuenta con ID: " << id << "\n";
    }
    
    pausar();
}

void gestionCuentas() {
    int opcion;
    do {
        limpiarPantalla();
        cout << "\n=== GESTIÓN DE CUENTAS ===\n";
        cout << "===========================\n";
        cout << "1. Ver todas las cuentas\n";
        cout << "2. Agregar nueva cuenta\n";
        cout << "3. Editar cuenta\n";
        cout << "4. Volver al menú principal\n";
        cout << "Seleccione una opción: ";
        
        opcion = leerEntero();
        
        switch (opcion) {
            case 1: mostrarCuentas(); break;
            case 2: agregarCuenta(); break;
            case 3: editarCuenta(); break;
            case 4: cout << "Volviendo...\n"; break;
            default: mostrarError("Opción inválida");
        }
    } while (opcion != 4);
}

// ==================== REPORTES ====================

void reporteGeneral() {
    limpiarPantalla();
    cout << "\n=== REPORTE GENERAL DE LA EMPRESA ===\n";
    cout << "=====================================\n\n";
    
    cout << "EMPRESA: " << configSistema.nombreEmpresa << "\n";
    cout << "RIF: " << configSistema.rif << "\n";
    cout << "FECHA: " << obtenerFechaHoraActual() << "\n\n";
    
    cout << "📊 ESTADÍSTICAS GENERALES:\n";
    cout << "  • Trabajadores: " << trabajadoresRegistrados.size() << "\n";
    
    int activos = 0;
    for (const auto& t : trabajadoresRegistrados) {
        if (t.estado == "Activo") activos++;
    }
    cout << "    - Activos: " << activos << "\n";
    cout << "    - Inactivos: " << trabajadoresRegistrados.size() - activos << "\n\n";
    
    cout << "  • Usuarios del sistema: " << usuariosRegistrados.size() << "\n";
    
    int usuariosActivos = 0;
    for (const auto& u : usuariosRegistrados) {
        if (u.activo) usuariosActivos++;
    }
    cout << "    - Activos: " << usuariosActivos << "\n";
    cout << "    - Inactivos: " << usuariosRegistrados.size() - usuariosActivos << "\n\n";
    
    cout << "  • Cuentas bancarias: " << cuentasRegistradas.size() << "\n";
    
    double totalSaldos = 0;
    for (const auto& c : cuentasRegistradas) {
        totalSaldos += c.saldo;
    }
    cout << "    - Saldo total: " << configSistema.moneda << " " 
         << fixed << setprecision(2) << totalSaldos << "\n\n";
    
    cout << "  • Pagos registrados: " << pagosRegistrados.size() << "\n";
    
    double totalPagado = 0;
    int pagosPendientes = 0;
    for (const auto& p : pagosRegistrados) {
        if (p.estado == "Pagado") totalPagado += p.monto;
        else if (p.estado == "Pendiente") pagosPendientes++;
    }
    cout << "    - Total pagado: " << configSistema.moneda << " " 
         << fixed << setprecision(2) << totalPagado << "\n";
    cout << "    - Pagos pendientes: " << pagosPendientes << "\n";
    
    pausar();
}

void reporteFinanciero() {
    limpiarPantalla();
    cout << "\n=== REPORTE FINANCIERO ===\n";
    cout << "===========================\n\n";
    
    // Totales por banco
    map<string, double> saldosPorBanco;
    map<string, int> cuentasPorBanco;
    
    for (const auto& c : cuentasRegistradas) {
        saldosPorBanco[c.banco] += c.saldo;
        cuentasPorBanco[c.banco]++;
    }
    
    cout << "SALDOS POR BANCO:\n";
    cout << string(40, '-') << "\n";
    
    for (const auto& [banco, saldo] : saldosPorBanco) {
        cout << left << setw(20) << banco 
             << ": " << cuentasPorBanco[banco] << " cuentas - "
             << configSistema.moneda << " " 
             << fixed << setprecision(2) << saldo << "\n";
    }
    
    // Totales de pagos
    cout << "\nPAGOS POR TIPO:\n";
    cout << string(40, '-') << "\n";
    
    map<string, double> pagosPorTipo;
    double totalPagos = 0;
    
    for (const auto& p : pagosRegistrados) {
        if (p.estado == "Pagado") {
            pagosPorTipo[p.tipo] += p.monto;
            totalPagos += p.monto;
        }
    }
    
    for (const auto& [tipo, monto] : pagosPorTipo) {
        double porcentaje = (monto / totalPagos) * 100;
        cout << left << setw(15) << tipo 
             << ": " << configSistema.moneda << " " 
             << fixed << setprecision(2) << monto
             << " (" << fixed << setprecision(1) << porcentaje << "%)\n";
    }
    
    cout << "\n💰 TOTAL PAGOS: " << configSistema.moneda << " " 
         << fixed << setprecision(2) << totalPagos << "\n";
    
    pausar();
}

void reporteRRHH() {
    limpiarPantalla();
    cout << "\n=== REPORTE DE RECURSOS HUMANOS ===\n";
    cout << "====================================\n\n";
    
    // Distribución por departamento
    map<string, int> trabajadoresPorDepto;
    map<string, double> salariosPorDepto;
    
    for (const auto& t : trabajadoresRegistrados) {
        if (t.estado == "Activo") {
            trabajadoresPorDepto[t.departamento]++;
            salariosPorDepto[t.departamento] += t.salario;
        }
    }
    
    cout << "DISTRIBUCIÓN POR DEPARTAMENTO:\n";
    cout << string(50, '-') << "\n";
    
    int totalActivos = 0;
    double totalSalarios = 0;
    
    for (const auto& [depto, count] : trabajadoresPorDepto) {
        totalActivos += count;
        totalSalarios += salariosPorDepto[depto];
        cout << left << setw(25) << depto 
             << ": " << count << " trabajadores - "
             << configSistema.moneda << " " 
             << fixed << setprecision(2) << salariosPorDepto[depto] << "\n";
    }
    
    cout << "\n📊 ESTADÍSTICAS SALARIALES:\n";
    cout << "  • Total trabajadores activos: " << totalActivos << "\n";
    cout << "  • Nómina mensual: " << configSistema.moneda << " " 
         << fixed << setprecision(2) << totalSalarios << "\n";
    
    if (totalActivos > 0) {
        cout << "  • Salario promedio: " << configSistema.moneda << " " 
             << fixed << setprecision(2) << (totalSalarios / totalActivos) << "\n";
    }
    
    pausar();
}

void gestionReportes() {
    int opcion;
    do {
        limpiarPantalla();
        cout << "\n=== REPORTES Y ESTADÍSTICAS ===\n";
        cout << "================================\n";
        cout << "1. Reporte general de la empresa\n";
        cout << "2. Reporte financiero\n";
        cout << "3. Reporte de RRHH\n";
        cout << "4. Volver al menú principal\n";
        cout << "Seleccione una opción: ";
        
        opcion = leerEntero();
        
        switch (opcion) {
            case 1: reporteGeneral(); break;
            case 2: reporteFinanciero(); break;
            case 3: reporteRRHH(); break;
            case 4: cout << "Volviendo...\n"; break;
            default: mostrarError("Opción inválida");
        }
    } while (opcion != 4);
}

// ==================== CONFIGURACIÓN DEL SISTEMA ====================

void mostrarConfiguracion() {
    limpiarPantalla();
    cout << "\n=== CONFIGURACIÓN DEL SISTEMA ===\n";
    cout << "==================================\n\n";
    
    cout << "DATOS DE LA EMPRESA:\n";
    cout << "  Nombre: " << configSistema.nombreEmpresa << "\n";
    cout << "  RIF: " << configSistema.rif << "\n";
    cout << "  Dirección: " << configSistema.direccion << "\n";
    cout << "  Teléfono: " << configSistema.telefono << "\n";
    cout << "  Email: " << configSistema.email << "\n\n";
    
    cout << "CONFIGURACIÓN GENERAL:\n";
    cout << "  Moneda: " << configSistema.moneda << "\n";
    cout << "  Día de pago: " << configSistema.diaPago << "\n";
    cout << "  IVA: " << configSistema.iva << "%\n";
    cout << "  Backup automático: " << configSistema.backupAutomatico << "\n";
    cout << "  Máx. intentos de login: " << configSistema.maxIntentosLogin << "\n";
    
    pausar();
}

void editarConfiguracion() {
    limpiarPantalla();
    cout << "\n=== EDITAR CONFIGURACIÓN ===\n";
    cout << "=============================\n\n";
    
    cout << "Deje en blanco para mantener el valor actual.\n\n";
    
    string entrada;
    
    cout << "Nombre de la empresa [" << configSistema.nombreEmpresa << "]: ";
    entrada = leerLinea();
    if (!entrada.empty()) configSistema.nombreEmpresa = entrada;
    
    cout << "RIF [" << configSistema.rif << "]: ";
    entrada = leerLinea();
    if (!entrada.empty()) configSistema.rif = entrada;
    
    cout << "Dirección [" << configSistema.direccion << "]: ";
    entrada = leerLinea();
    if (!entrada.empty()) configSistema.direccion = entrada;
    
    cout << "Teléfono [" << configSistema.telefono << "]: ";
    entrada = leerLinea();
    if (!entrada.empty()) configSistema.telefono = entrada;
    
    cout << "Email [" << configSistema.email << "]: ";
    entrada = leerLinea();
    if (!entrada.empty()) configSistema.email = entrada;
    
    cout << "Moneda [" << configSistema.moneda << "]: ";
    entrada = leerLinea();
    if (!entrada.empty()) configSistema.moneda = entrada;
    
    cout << "Día de pago [" << configSistema.diaPago << "]: ";
    entrada = leerLinea();
    if (!entrada.empty()) configSistema.diaPago = stoi(entrada);
    
    cout << "IVA (%) [" << configSistema.iva << "]: ";
    entrada = leerLinea();
    if (!entrada.empty()) configSistema.iva = stod(entrada);
    
    cout << "\n✅ Configuración actualizada.\n";
    pausar();
}

void gestionConfiguracion() {
    int opcion;
    do {
        limpiarPantalla();
        cout << "\n=== CONFIGURACIÓN DEL SISTEMA ===\n";
        cout << "==================================\n";
        cout << "1. Ver configuración actual\n";
        cout << "2. Editar configuración\n";
        cout << "3. Volver al menú principal\n";
        cout << "Seleccione una opción: ";
        
        opcion = leerEntero();
        
        switch (opcion) {
            case 1: mostrarConfiguracion(); break;
            case 2: editarConfiguracion(); break;
            case 3: cout << "Volviendo...\n"; break;
            default: mostrarError("Opción inválida");
        }
    } while (opcion != 3);
}

// ==================== BACKUP Y RESTAURACIÓN ====================

void crearBackup() {
    limpiarPantalla();
    cout << "\n=== CREAR BACKUP ===\n";
    cout << "====================\n\n";
    
    string nombreBackup = "backup_" + obtenerFechaActual() + "_" + 
                          to_string(time(nullptr)) + ".dat";
    
    // Copiar el archivo actual
    ifstream origen("datos_sistema.dat", ios::binary);
    if (!origen) {
        cout << "No hay datos para respaldar.\n";
        pausar();
        return;
    }
    
    ofstream destino(nombreBackup, ios::binary);
    if (!destino) {
        cout << "❌ No se pudo crear el backup.\n";
        pausar();
        return;
    }
    
    destino << origen.rdbuf();
    
    cout << "✅ Backup creado exitosamente.\n";
    cout << "Archivo: " << nombreBackup << "\n";
    
    pausar();
}

void restaurarBackup() {
    limpiarPantalla();
    cout << "\n=== RESTAURAR BACKUP ===\n";
    cout << "========================\n\n";
    
    cout << "Backups disponibles:\n";
    
    // Buscar archivos de backup
    system("dir backup_*.dat /b 2>nul");
    
    cout << "\nIngrese el nombre del archivo a restaurar: ";
    string nombreBackup = leerLinea();
    
    ifstream backup(nombreBackup, ios::binary);
    if (!backup) {
        cout << "❌ No se encontró el archivo.\n";
        pausar();
        return;
    }
    
    cout << "¿Está seguro de restaurar? Se perderán los datos actuales. (s/n): ";
    char conf;
    cin >> conf;
    cin.ignore();
    
    if (tolower(conf) == 's') {
        ofstream destino("datos_sistema.dat", ios::binary);
        destino << backup.rdbuf();
        
        cout << "✅ Backup restaurado. Reinicie el sistema para aplicar cambios.\n";
    } else {
        cout << "Operación cancelada.\n";
    }
    
    pausar();
}

void gestionBackup() {
    int opcion;
    do {
        limpiarPantalla();
        cout << "\n=== BACKUP Y RESTAURACIÓN ===\n";
        cout << "==============================\n";
        cout << "1. Crear backup manual\n";
        cout << "2. Restaurar desde backup\n";
        cout << "3. Volver al menú principal\n";
        cout << "Seleccione una opción: ";
        
        opcion = leerEntero();
        
        switch (opcion) {
            case 1: crearBackup(); break;
            case 2: restaurarBackup(); break;
            case 3: cout << "Volviendo...\n"; break;
            default: mostrarError("Opción inválida");
        }
    } while (opcion != 3);
}

// ==================== MENÚ PRINCIPAL ====================

void menuAdministradorUniversal() {
    int opcion;
    do {
        limpiarPantalla();
        cout << "\n=========================================\n";
        cout << "   MENÚ PRINCIPAL - ADMINISTRADOR\n";
        cout << "=========================================\n";
        cout << "Usuario: " << nombreUsuarioActual << "\n";
        cout << "=========================================\n\n";
        
        cout << "1. Gestión de Trabajadores\n";
        cout << "2. Gestión de Pagos Bancarios\n";
        cout << "3. Gestión de Cuentas\n";
        cout << "4. Reportes y Estadísticas\n";
        cout << "5. Configuración del Sistema\n";
        cout << "6. Backup y Restauración\n";
        cout << "7. Guardar Datos\n";
        cout << "8. Cerrar Sesión\n";
        cout << "\nSeleccione una opción: ";
        
        opcion = leerEntero();
        
        switch (opcion) {
            case 1: gestionTrabajadores(); break;
            case 2: gestionPagos(); break;
            case 3: gestionCuentas(); break;
            case 4: gestionReportes(); break;
            case 5: gestionConfiguracion(); break;
            case 6: gestionBackup(); break;
            case 7: guardarDatos(); pausar(); break;
            case 8: cout << "Cerrando sesión...\n"; break;
            default: mostrarError("Opción inválida");
        }
    } while (opcion != 8);
}

// ==================== PROGRAMA PRINCIPAL ====================

int main() {
    // Configurar locale para mostrar caracteres especiales
    setlocale(LC_ALL, "");
    
    cout << "=========================================\n";
    cout << "   SISTEMA DE GESTIÓN EMPRESARIAL\n";
    cout << "          Versión 2.0\n";
    cout << "=========================================\n\n";
    
    // Cargar datos
    cargarDatos();
    
    // Menú de autenticación
    while (true) {
        resetearSesion();
        
        if (autenticarUsuario()) {
            if (esAdministradorUniversal) {
                menuAdministradorUniversal();
            } else {
                // Usuario regular (puedes implementar menú específico)
                cout << "\nMódulo para usuarios regulares en desarrollo.\n";
                pausar();
            }
        } else {
            int opcion;
            cout << "\n1. Reintentar\n";
            cout << "2. Salir\n";
            cout << "Seleccione: ";
            opcion = leerEntero();
            
            if (opcion == 2) break;
        }
    }
    
    // Guardar datos al salir
    guardarDatos();
    
    cout << "\n=========================================\n";
    cout << "   ¡Gracias por usar el sistema!\n";
    cout << "=========================================\n";
    
    return 0;
}
CODIGO

# proyecto_gestion.cpp
Este código en PDF lo hice yo en c++ pero esta desactualizado lo monto para que ustedes lo usen libremente si son estudiantes o practicantes no esta completo  
# 🏦 Sistema de Gestión Bancaria y Empresarial en C++

[![C++](https://img.shields.io/badge/C++-17-blue.svg)](https://isocpp.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Estado](https://img.shields.io/badge/Estado-En%20Desarrollo-green.svg)]()

## 📋 Descripción

Un sistema completo de gestión empresarial desarrollado en C++ que incluye módulos para administración de usuarios, trabajadores, pagos bancarios, cuentas y reportes. Este proyecto nació como un "PayPal casero" y evolucionó hasta convertirse en un **ERP funcional** con más de 4000 líneas de código.

### ✨ Características Principales

- ✅ **Gestión de Usuarios** - Autenticación, roles y permisos por módulos
- ✅ **Administración de Trabajadores** - CRUD completo con búsqueda por cédula
- ✅ **Sistema de Pagos por Banco** - Pagos agrupados y ordenados por cédula
- ✅ **Módulo Bancario** - Cuentas, saldos, movimientos y bloqueos
- ✅ **Reportes Detallados** - Financieros, RRHH, productividad con timestamps
- ✅ **Backup Automático** - Respaldo y restauración de datos
- ✅ **Gestión de Licencias** - Control de software y renovaciones
- ✅ **Configuración Empresarial** - IVA, moneda, horarios y políticas

## 🚀 Cómo Empezar

### Prerrequisitos

- Compilador con soporte C++11 o superior (g++, MinGW, MSVC)
- Git (opcional, para clonar el repositorio)

### Compilación y Ejecución

```bash
# Clonar el repositorio
git clone https://github.com/tu-usuario/sistema-gestion-cpp.git

# Entrar al directorio
cd sistema-gestion-cpp

# Compilar (ajusta según tu compilador)
g++ -std=c++11 main.cpp -o sistema

# Ejecutar
./sistema
```

### Usuario por Defecto

| Usuario | Contraseña | Rol |
|---------|------------|-----|
| admin   | admin123   | Administrador Universal |

## 📁 Estructura del Proyecto

```
sistema-gestion-cpp/
├── 📄 main.cpp              # Código fuente principal
├── 📄 usuarios.dat          # Datos de usuarios (se genera al ejecutar)
├── 📄 trabajadores.dat      # Datos de trabajadores
├── 📄 cuentas.dat           # Datos de cuentas bancarias
├── 📄 pagos.dat             # Historial de pagos
├── 📄 auditoria.log         # Registro de actividades
├── 📁 backups/              # Carpeta de respaldos automáticos
├── 📄 README.md             # Este archivo
└── 📄 LICENSE               # Licencia MIT
```

## 🎯 Módulos del Sistema

### 1. Gestión de Trabajadores
- Alta, baja y modificación de empleados
- Búsqueda por cédula o nombre
- Asignación de cargos y departamentos
- Control de salarios

### 2. Sistema de Pagos
- Pagos individuales o por lote
- Agrupación por banco (Banco Venezuela, Mercantil, etc.)
- Ordenamiento automático por número de cédula
- Generación de reportes con timestamp

### 3. Módulo Bancario
- Múltiples cuentas por usuario
- Control de saldos
- Bloqueo/desbloqueo de cuentas
- Historial de movimientos

### 4. Reportes y Estadísticas
- Reporte general de la empresa
- Análisis financiero
- Estadísticas de RRHH
- Exportación a CSV

### 5. Seguridad y Backups
- Autenticación de usuarios
- Control de acceso por módulos
- Backup automático programable
- Restauración de datos

## 📊 Ejemplo de Uso

```cpp
// Procesar pagos agrupados por banco
Banco venezuela = cargarPagos("Banco Venezuela");
venezuela.ordenarPorCedula();
venezuela.procesarLote();

// Generar reporte
Reporte diario = generarReporte(venezuela);
diario.mostrar();
diario.guardar("reporte_20260311.txt");
```

## 🤝 Cómo Contribuir

¡Las contribuciones son bienvenidas! Si quieres mejorar este proyecto:

1. Haz un Fork del repositorio
2. Crea una rama (`git checkout -b feature/mejora`)
3. Haz tus cambios
4. Commit (`git commit -m 'Añadir nueva funcionalidad'`)
5. Push (`git push origin feature/mejora`)
6. Abre un Pull Request

### Ideas para Contribuir
- Agregar interfaz gráfica
- Migrar a base de datos SQL
- Implementar API REST
- Crear frontend web
- Añadir más tipos de reportes
- Mejorar la documentación

## 📝 Licencia

Este proyecto está bajo la Licencia MIT - ver el archivo [LICENSE](LICENSE) para más detalles.

## ✨ Agradecimientos

- A la comunidad de C++ por la documentación
- A Stack Overflow por salvar el código无数次
- A SIGESP por inspirar (por oposición) este proyecto
- A WhatsApp por servir de editor en momentos de necesidad

## 📬 Contacto

¿Preguntas? ¿Sugerencias? ¿Quieres contratarme?

- **Email**: [tu-email@ejemplo.com](mailto:tu-email@ejemplo.com)
- **GitHub**: [@tu-usuario](https://github.com/tu-usuario)
- **LinkedIn**: [Tu Perfil](https://linkedin.com/in/tu-perfil)

---

⭐ **Si te gusta este proyecto, ¡no olvides darle una estrella en GitHub!** ⭐

**Hecho con ❤️ y mucho ☕ por GABRIEL
