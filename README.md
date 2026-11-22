# actividad-anexo-3
actividad universidad
from __future__ import annotations

from abc import ABC, abstractmethod
from datetime import datetime
from enum import Enum
import logging
import re
from typing import List, Optional, overload


# ==============================================================================
# 1. CONFIGURACIÓN DEL SISTEMA DE LOGS (ÚNICO USO DE ARCHIVOS)
# ==============================================================================

def configurar_logger(archivo_log: str = "software_fj.log") -> logging.Logger:
    """Configura el registro de eventos y errores hacia un archivo de texto."""
    logger = logging.getLogger("SoftwareFJ")
    logger.setLevel(logging.DEBUG)

    if not logger.handlers:
        manejador_archivo = logging.FileHandler(
            archivo_log, mode="a", encoding="utf-8"
        )
        formato = logging.Formatter(
            "%(asctime)s | %(levelname)-8s | %(message)s",
            datefmt="%Y-%m-%d %H:%M:%S",
        )
        manejador_archivo.setFormatter(formato)
        logger.addHandler(manejador_archivo)

    return logger


# ==============================================================================
# 2. JERARQUÍA DE EXCEPCIONES PERSONALIZADAS
# ==============================================================================

class SoftwareFJError(Exception):
    """Excepción base para todos los errores controlados de Software FJ."""


class DatosInvalidosError(SoftwareFJError):
    """Se lanza cuando un dato no cumple el formato o rango permitido."""


class ParametroFaltanteError(SoftwareFJError):
    """Se lanza cuando un parámetro obligatorio es nulo o está vacío."""


class OperacionNoPermitidaError(SoftwareFJError):
    """Se lanza al intentar ejecutar una acción inválida según las reglas del negocio."""


class ServicioNoDisponibleError(SoftwareFJError):
    """Se lanza cuando un servicio no tiene disponibilidad o stock suficiente."""


class CalculoInconsistenteError(SoftwareFJError):
    """Se lanza cuando los parámetros de cálculo producen valores inválidos."""


class ReservaError(SoftwareFJError):
    """Excepción de alto nivel para fallos durante el ciclo de vida de una reserva."""


# ==============================================================================
# 3. CLASE ABSTRACTA BASE PARA ENTIDADES DEL SISTEMA
# ==============================================================================

class EntidadSistema(ABC):
    """Clase abstracta que representa cualquier entidad administrada por Software FJ."""

    def __init__(self, id_entidad: str) -> None:
        if not id_entidad or not isinstance(id_entidad, str) or not id_entidad.strip():
            raise ParametroFaltanteError("El ID de la entidad es obligatorio.")
        self.__id: str = id_entidad.strip()
        self.__fecha_creacion: datetime = datetime.now()

    @property
    def id(self) -> str:
        return self.__id

    @property
    def fecha_creacion(self) -> datetime:
        return self.__fecha_creacion

    @abstractmethod
    def validar(self) -> None:
        """Valida la integridad interna de la entidad."""

    @abstractmethod
    def obtener_resumen(self) -> str:
        """Devuelve una representación resumida de la entidad."""


# ==============================================================================
# 4. CLASE CLIENTE (ENCAPSULACIÓN Y VALIDACIONES ESTRICTAS)
# ==============================================================================

class Cliente(EntidadSistema):
    """Representa un cliente de Software FJ con protección estricta de datos personales."""

    PATRON_EMAIL = re.compile(r"^[\w\.-]+@[\w\.-]+\.\w{2,}$")

    def __init__(
        self,
        id_cliente: str,
        nombre: str,
        documento: str,
        email: str,
        telefono: str,
    ) -> None:
        super().__init__(id_cliente)
        self.__nombre: str = ""
        self.__documento: str = ""
        self.__email: str = ""
        self.__telefono: str = ""

        # Uso de setters encapsulados para validar desde la construcción
        self.nombre = nombre
        self.documento = documento
        self.email = email
        self.telefono = telefono
        self.validar()

    @property
    def nombre(self) -> str:
        return self.__nombre

    @nombre.setter
    def nombre(self, valor: str) -> None:
        if not valor or not isinstance(valor, str) or not valor.strip():
            raise ParametroFaltanteError("El nombre del cliente es obligatorio.")
        if len(valor.strip()) < 3:
            raise DatosInvalidosError("El nombre debe tener al menos 3 caracteres.")
        self.__nombre = valor.strip()

    @property
    def documento(self) -> str:
        return self.__documento

    @documento.setter
    def documento(self, valor: str) -> None:
        if not valor or not isinstance(valor, str) or not valor.strip():
            raise ParametroFaltanteError("El documento de identidad es obligatorio.")
        limpio = valor.strip()
        if not limpio.isdigit() or not (6 <= len(limpio) <= 15):
            raise DatosInvalidosError(
                f"Documento '{valor}' inválido: debe contener entre 6 y 15 dígitos numéricos."
            )
        self.__documento = limpio

    @property
    def email(self) -> str:
        return self.__email

    @email.setter
    def email(self, valor: str) -> None:
        if not valor or not isinstance(valor, str) or not valor.strip():
            raise ParametroFaltanteError("El correo electrónico es obligatorio.")
        limpio = valor.strip().lower()
        if not self.PATRON_EMAIL.match(limpio):
            raise DatosInvalidosError(f"Correo electrónico inválido: '{valor}'.")
        self.__email = limpio

    @property
    def telefono(self) -> str:
        return self.__telefono

    @telefono.setter
    def telefono(self, valor: str) -> None:
        if not valor or not isinstance(valor, str) or not valor.strip():
            raise ParametroFaltanteError("El número de teléfono es obligatorio.")
        digitos = re.sub(r"[\s\-\+]", "", valor.strip())
        if not digitos.isdigit() or not (7 <= len(digitos) <= 15):
            raise DatosInvalidosError(f"Teléfono inválido: '{valor}'.")
        self.__telefono = valor.strip()

    def validar(self) -> None:
        if not all([self.__nombre, self.__documento, self.__email, self.__telefono]):
            raise ParametroFaltanteError("El cliente contiene campos obligatorios sin completar.")

    def obtener_resumen(self) -> str:
        return f"Cliente[{self.id}] {self.__nombre} | Doc: {self.__documento} | Email: {self.__email}"


# ==============================================================================
# 5. CLASE ABSTRACTA SERVICIO Y SERVICIOS ESPECIALIZADOS (POLIMORFISMO + SOBRECARGA)
# ==============================================================================

class Servicio(EntidadSistema, ABC):
    """Clase abstracta para los servicios ofrecidos por Software FJ."""

    def __init__(
        self,
        id_servicio: str,
        nombre: str,
        tarifa_base_hora: float,
        disponible: bool = True,
    ) -> None:
        super().__init__(id_servicio)
        if not nombre or not isinstance(nombre, str) or not nombre.strip():
            raise ParametroFaltanteError("El nombre del servicio es obligatorio.")
        self.__nombre: str = nombre.strip()
        self.__tarifa_base_hora: float = 0.0
        self.tarifa_base_hora = tarifa_base_hora
        self.__disponible: bool = bool(disponible)

    @property
    def nombre(self) -> str:
        return self.__nombre

    @property
    def tarifa_base_hora(self) -> float:
        return self.__tarifa_base_hora

    @tarifa_base_hora.setter
    def tarifa_base_hora(self, valor: float) -> None:
        try:
            tarifa = float(valor)
        except (TypeError, ValueError) as exc:
            raise DatosInvalidosError(f"La tarifa '{valor}' no es un número válido.") from exc
        if tarifa <= 0:
            raise DatosInvalidosError("La tarifa base por hora debe ser mayor que cero.")
        self.__tarifa_base_hora = tarifa

    @property
    def disponible(self) -> bool:
        return self.__disponible

    def establecer_disponibilidad(self, estado: bool) -> None:
        self.__disponible = bool(estado)

    def validar(self) -> None:
        if self.__tarifa_base_hora <= 0:
            raise DatosInvalidosError("Tarifa base inválida.")
        self.validar_parametros()

    @abstractmethod
    def validar_parametros(self) -> None:
        """Valida los parámetros específicos de cada tipo de servicio."""

    @abstractmethod
    def describir(self) -> str:
        """Devuelve una descripción detallada y polimórfica del servicio."""

    @abstractmethod
    def _calcular_subtotal(self, duracion_horas: float) -> float:
        """Calcula el subtotal propio de cada tipo de servicio antes de ajustes."""

    def reservar_cupo(self) -> None:
        """Bloquea el servicio al confirmar una reserva."""
        if not self.__disponible:
            raise ServicioNoDisponibleError(
                f"El servicio '{self.__nombre}' ({self.id}) no se encuentra disponible."
            )
        self.__disponible = False

    def liberar_cupo(self) -> None:
        """Libera el servicio si una reserva es cancelada."""
        self.__disponible = True

    # --- Sobrecarga de método calcular_costo (tipado estático + despacho dinámico) ---
    @overload
    def calcular_costo(self, duracion_horas: float) -> float: ...

    @overload
    def calcular_costo(self, duracion_horas: float, *, impuesto_pct: float) -> float: ...

    @overload
    def calcular_costo(
        self, duracion_horas: float, *, impuesto_pct: float, descuento_pct: float
    ) -> float: ...

    @overload
    def calcular_costo(
        self,
        duracion_horas: float,
        *,
        impuesto_pct: float = 0.0,
        descuento_pct: float = 0.0,
        recargo_fijo: float = 0.0,
    ) -> float: ...

    def calcular_costo(
        self,
        duracion_horas: float,
        *,
        impuesto_pct: float = 0.0,
        descuento_pct: float = 0.0,
        recargo_fijo: float = 0.0,
    ) -> float:
        """
        Método sobrecargado para calcular el costo final de un servicio.
        Soporta cálculo base, con impuestos, con descuentos y/o recargos fijos.
        """
        try:
            horas = float(duracion_horas)
            impuesto = float(impuesto_pct)
            descuento = float(descuento_pct)
            recargo = float(recargo_fijo)
        except (TypeError, ValueError) as exc:
            raise CalculoInconsistenteError(
                "Todos los parámetros de cálculo de costo deben ser numéricos."
            ) from exc

        if horas <= 0:
            raise CalculoInconsistenteError("La duración en horas debe ser mayor a cero.")
        if impuesto < 0 or impuesto > 100:
            raise CalculoInconsistenteError(
                f"Porcentaje de impuesto fuera de rango (0-100): {impuesto}%."
            )
        if descuento < 0 or descuento > 100:
            raise CalculoInconsistenteError(
                f"Porcentaje de descuento fuera de rango (0-100): {descuento}%."
            )
        if recargo < 0:
            raise CalculoInconsistenteError("El recargo fijo no puede ser negativo.")

        subtotal = self._calcular_subtotal(horas)
        monto_descuento = subtotal * (descuento / 100.0)
        base_gravable = subtotal - monto_descuento + recargo
        monto_impuesto = base_gravable * (impuesto / 100.0)
        total = round(base_gravable + monto_impuesto, 2)

        if total < 0:
            raise CalculoInconsistenteError("El cálculo generó un costo total negativo.")

        return total

    def obtener_resumen(self) -> str:
        estado = "Disponible" if self.disponible else "No disponible"
        return f"Servicio[{self.id}] {self.describir()} | Estado: {estado}"


class ReservaSala(Servicio):
    """Servicio especializado: Reserva de salas corporativas y de capacitación."""

    RECARGO_AUDIOVISUAL = 35000.0

    def __init__(
        self,
        id_servicio: str,
        nombre: str,
        tarifa_base_hora: float,
        capacidad_personas: int,
        incluye_audiovisual: bool = False,
    ) -> None:
        super().__init__(id_servicio, nombre, tarifa_base_hora)
        self.__capacidad_personas: int = capacidad_personas
        self.__incluye_audiovisual: bool = bool(incluye_audiovisual)
        self.validar()

    def validar_parametros(self) -> None:
        if not isinstance(self.__capacidad_personas, int) or not (1 <= self.__capacidad_personas <= 300):
            raise DatosInvalidosError(
                f"Capacidad de sala inválida ({self.__capacidad_personas}): debe estar entre 1 y 300 personas."
            )

    def _calcular_subtotal(self, duracion_horas: float) -> float:
        extra_av = self.RECARGO_AUDIOVISUAL if self.__incluye_audiovisual else 0.0
        return (self.tarifa_base_hora * duracion_horas) + extra_av

    def describir(self) -> str:
        av = "con equipo audiovisual" if self.__incluye_audiovisual else "sin equipo audiovisual"
        return (
            f"Sala '{self.nombre}' (Capacidad: {self.__capacidad_personas} personas, {av}) "
            f"- ${self.tarifa_base_hora:,.0f}/h"
        )


class AlquilerEquipo(Servicio):
    """Servicio especializado: Alquiler de hardware y equipos tecnológicos con control de stock."""

    def __init__(
        self,
        id_servicio: str,
        nombre: str,
        tarifa_base_hora: float,
        tipo_equipo: str,
        stock_disponible: int,
        unidades_por_reserva: int = 1,
    ) -> None:
        super().__init__(id_servicio, nombre, tarifa_base_hora, disponible=stock_disponible > 0)
        self.__tipo_equipo: str = tipo_equipo
        self.__stock_disponible: int = stock_disponible
        self.__unidades_por_reserva: int = unidades_por_reserva
        self.validar()

    @property
    def stock_disponible(self) -> int:
        return self.__stock_disponible

    def validar_parametros(self) -> None:
        if not self.__tipo_equipo or not isinstance(self.__tipo_equipo, str) or not self.__tipo_equipo.strip():
            raise ParametroFaltanteError("El tipo de equipo es obligatorio.")
        if not isinstance(self.__stock_disponible, int) or self.__stock_disponible < 0:
            raise DatosInvalidosError("El stock disponible no puede ser negativo.")
        if not isinstance(self.__unidades_por_reserva, int) or self.__unidades_por_reserva <= 0:
            raise DatosInvalidosError("Las unidades por reserva deben ser al menos 1.")

    def reservar_cupo(self) -> None:
        if self.__stock_disponible < self.__unidades_por_reserva:
            raise ServicioNoDisponibleError(
                f"Stock insuficiente para '{self.nombre}': disponibles {self.__stock_disponible}, "
                f"requeridas {self.__unidades_por_reserva}."
            )
        self.__stock_disponible -= self.__unidades_por_reserva
        self.establecer_disponibilidad(self.__stock_disponible >= self.__unidades_por_reserva)

    def liberar_cupo(self) -> None:
        self.__stock_disponible += self.__unidades_por_reserva
        self.establecer_disponibilidad(self.__stock_disponible >= self.__unidades_por_reserva)

    def _calcular_subtotal(self, duracion_horas: float) -> float:
        return self.tarifa_base_hora * duracion_horas * self.__unidades_por_reserva

    def describir(self) -> str:
        return (
            f"Alquiler '{self.nombre}' ({self.__tipo_equipo}, {self.__unidades_por_reserva} ud/reserva, "
            f"stock: {self.__stock_disponible}) - ${self.tarifa_base_hora:,.0f}/h por unidad"
        )


class AsesoriaEspecializada(Servicio):
    """Servicio especializado: Consultoría técnica por especialidad y nivel de complejidad."""

    MULTIPLICADORES_NIVEL = {
        "BASICO": 1.0,
        "INTERMEDIO": 1.25,
        "AVANZADO": 1.5,
    }

    def __init__(
        self,
        id_servicio: str,
        nombre: str,
        tarifa_base_hora: float,
        especialidad: str,
        nivel_complejidad: str,
        consultor_asignado: str,
    ) -> None:
        super().__init__(id_servicio, nombre, tarifa_base_hora)
        self.__especialidad: str = especialidad
        self.__nivel_complejidad: str = nivel_complejidad.upper() if isinstance(nivel_complejidad, str) else ""
        self.__consultor_asignado: str = consultor_asignado
        self.validar()

    def validar_parametros(self) -> None:
        if not self.__especialidad or not self.__especialidad.strip():
            raise ParametroFaltanteError("La especialidad de la asesoría es obligatoria.")
        if not self.__consultor_asignado or not self.__consultor_asignado.strip():
            raise ParametroFaltanteError("El nombre del consultor asignado es obligatorio.")
        if self.__nivel_complejidad not in self.MULTIPLICADORES_NIVEL:
            raise DatosInvalidosError(
                f"Nivel de complejidad inválido: '{self.__nivel_complejidad}'. "
                f"Valores válidos: {list(self.MULTIPLICADORES_NIVEL.keys())}."
            )

    def _calcular_subtotal(self, duracion_horas: float) -> float:
        factor = self.MULTIPLICADORES_NIVEL[self.__nivel_complejidad]
        return self.tarifa_base_hora * duracion_horas * factor

    def describir(self) -> str:
        return (
            f"Asesoría '{self.nombre}' ({self.__especialidad} - Nivel {self.__nivel_complejidad}, "
            f"Consultor: {self.__consultor_asignado}) - Base ${self.tarifa_base_hora:,.0f}/h"
        )


# ==============================================================================
# 6. CLASE RESERVA (CICLO DE VIDA, VALIDACIONES Y ENCADENAMIENTO DE EXCEPCIONES)
# ==============================================================================

class EstadoReserva(Enum):
    PENDIENTE = "PENDIENTE"
    CONFIRMADA = "CONFIRMADA"
    PROCESADA = "PROCESADA"
    CANCELADA = "CANCELADA"


class Reserva(EntidadSistema):
    """Integra un Cliente, un Servicio, duración y estado con control de transiciones."""

    def __init__(
        self,
        id_reserva: str,
        cliente: Cliente,
        servicio: Servicio,
        duracion_horas: float,
    ) -> None:
        super().__init__(id_reserva)
        if not isinstance(cliente, Cliente):
            raise DatosInvalidosError("La reserva requiere una instancia válida de Cliente.")
        if not isinstance(servicio, Servicio):
            raise DatosInvalidosError("La reserva requiere una instancia válida de Servicio.")

        self.__cliente: Cliente = cliente
        self.__servicio: Servicio = servicio
        self.__duracion_horas: float = duracion_horas
        self.__estado: EstadoReserva = EstadoReserva.PENDIENTE
        self.__costo_total: float = 0.0
        self.validar()

    @property
    def cliente(self) -> Cliente:
        return self.__cliente

    @property
    def servicio(self) -> Servicio:
        return self.__servicio

    @property
    def duracion_horas(self) -> float:
        return self.__duracion_horas

    @property
    def estado(self) -> EstadoReserva:
        return self.__estado

    @property
    def costo_total(self) -> float:
        return self.__costo_total

    def validar(self) -> None:
        if not isinstance(self.__duracion_horas, (int, float)) or self.__duracion_horas <= 0:
            raise DatosInvalidosError(
                f"Duración inválida ({self.__duracion_horas}): debe ser un número positivo."
            )

    def confirmar(
        self,
        impuesto_pct: float = 19.0,
        descuento_pct: float = 0.0,
        recargo_fijo: float = 0.0,
    ) -> float:
        """Confirma la reserva calculando el costo y reservando el cupo del servicio."""
        if self.__estado != EstadoReserva.PENDIENTE:
            raise OperacionNoPermitidaError(
                f"Solo se pueden confirmar reservas en estado PENDIENTE (actual: {self.__estado.value})."
            )

        try:
            # 1. Calcular el costo primero para evitar bloquear el servicio si el cálculo falla
            costo_calculado = self.__servicio.calcular_costo(
                self.__duracion_horas,
                impuesto_pct=impuesto_pct,
                descuento_pct=descuento_pct,
                recargo_fijo=recargo_fijo,
            )
            # 2. Bloquear disponibilidad/stock del servicio
            self.__servicio.reservar_cupo()
        except (CalculoInconsistenteError, ServicioNoDisponibleError) as exc:
            # Encadenamiento explícito de excepciones
            raise ReservaError(
                f"Fallo al confirmar la reserva {self.id} para '{self.__servicio.nombre}': {exc}"
            ) from exc
        else:
            # Bloque else: solo se ejecuta si NO ocurrió ninguna excepción
            self.__costo_total = costo_calculado
            self.__estado = EstadoReserva.CONFIRMADA
            return self.__costo_total

    def procesar(self) -> None:
        """Ejecuta y cierra una reserva previamente confirmada."""
        if self.__estado != EstadoReserva.CONFIRMADA:
            raise OperacionNoPermitidaError(
                f"No se puede procesar la reserva {self.id} en estado {self.__estado.value}. "
                "Debe estar CONFIRMADA."
            )
        self.__estado = EstadoReserva.PROCESADA

    def cancelar(self, motivo: str = "Cancelación solicitada por el cliente") -> None:
        """Cancela una reserva activa y libera los recursos reservados."""
        if self.__estado in (EstadoReserva.CANCELADA, EstadoReserva.PROCESADA):
            raise OperacionNoPermitidaError(
                f"No es posible cancelar la reserva {self.id} porque ya está {self.__estado.value}."
            )
        if not motivo or not motivo.strip():
            raise ParametroFaltanteError("Debe indicar un motivo de cancelación.")

        if self.__estado == EstadoReserva.CONFIRMADA:
            self.__servicio.liberar_cupo()
        self.__estado = EstadoReserva.CANCELADA

    def obtener_resumen(self) -> str:
        return (
            f"Reserva[{self.id}] Cliente: {self.__cliente.nombre} | "
            f"Servicio: {self.__servicio.nombre} | Duración: {self.__duracion_horas}h | "
            f"Estado: {self.__estado.value} | Total: ${self.__costo_total:,.2f}"
        )


# ==============================================================================
# 7. GESTOR CENTRAL DEL SISTEMA (LISTAS INTERNAS + MANEJO ROBUSTO DE ERRORES)
# ==============================================================================

class GestorSoftwareFJ:
    """Administra clientes, servicios y reservas en memoria con auditoría en archivo de logs."""

    def __init__(self, archivo_log: str = "software_fj.log") -> None:
        self.__clientes: List[Cliente] = []
        self.__servicios: List[Servicio] = []
        self.__reservas: List[Reserva] = []
        self.__logger: logging.Logger = configurar_logger(archivo_log)
        self.__logger.info("=== INICIO DE SESIÓN DEL SISTEMA SOFTWARE FJ ===")

    def _registrar_error(self, contexto: str, exc: Exception) -> None:
        causa = f" | Causa raíz: {type(exc.__cause__).__name__}: {exc.__cause__}" if exc.__cause__ else ""
        mensaje = f"{contexto} -> {type(exc).__name__}: {exc}{causa}"
        self.__logger.error(mensaje)
        print(f"  [ERROR CONTROLADO] {mensaje}")

    def registrar_cliente(
        self,
        id_cliente: str,
        nombre: str,
        documento: str,
        email: str,
        telefono: str,
    ) -> Optional[Cliente]:
        """Registra un cliente usando bloque try / except / else."""
        try:
            if any(c.id == id_cliente or c.documento == documento for c in self.__clientes):
                raise OperacionNoPermitidaError(
                    f"Ya existe un cliente registrado con ID '{id_cliente}' o documento '{documento}'."
                )
            cliente = Cliente(id_cliente, nombre, documento, email, telefono)
        except SoftwareFJError as exc:
            self._registrar_error(f"Registro de cliente ({id_cliente})", exc)
            return None
        else:
            self.__clientes.append(cliente)
            self.__logger.info(f"Cliente registrado: {cliente.obtener_resumen()}")
            print(f"  [OK] {cliente.obtener_resumen()}")
            return cliente

    def registrar_servicio(self, servicio: Servicio) -> bool:
        """Registra un servicio ya instanciado usando bloque try / except / finally."""
        exito = False
        try:
            if not isinstance(servicio, Servicio):
                raise DatosInvalidosError("El objeto proporcionado no es un Servicio válido.")
            if any(s.id == servicio.id for s in self.__servicios):
                raise OperacionNoPermitidaError(f"Ya existe un servicio con ID '{servicio.id}'.")
            servicio.validar()
            self.__servicios.append(servicio)
            exito = True
            self.__logger.info(f"Servicio registrado: {servicio.obtener_resumen()}")
            print(f"  [OK] {servicio.obtener_resumen()}")
        except SoftwareFJError as exc:
            self._registrar_error("Registro de servicio", exc)
        finally:
            self.__logger.debug(
                f"Finalizó intento de registro de servicio. Total servicios activos: {len(self.__servicios)}"
            )
        return exito

    def crear_servicio_seguro(self, clase_servicio: type[Servicio], **kwargs) -> Optional[Servicio]:
        """Instancia y registra un servicio capturando errores desde su construcción."""
        try:
            servicio = clase_servicio(**kwargs)
        except (SoftwareFJError, TypeError) as exc:
            self._registrar_error(f"Creación de {clase_servicio.__name__}", exc)
            return None
        else:
            if self.registrar_servicio(servicio):
                return servicio
            return None

    def crear_y_confirmar_reserva(
        self,
        id_reserva: str,
        cliente: Optional[Cliente],
        servicio: Optional[Servicio],
        duracion_horas: float,
        impuesto_pct: float = 19.0,
        descuento_pct: float = 0.0,
        recargo_fijo: float = 0.0,
        auto_procesar: bool = False,
    ) -> Optional[Reserva]:
        """
        Crea, confirma y opcionalmente procesa una reserva usando try / except / else / finally
        y preservando el encadenamiento de excepciones.
        """
        reserva: Optional[Reserva] = None
        try:
            if cliente is None:
                raise ParametroFaltanteError("No se proporcionó un cliente válido para la reserva.")
            if servicio is None:
                raise ParametroFaltanteError("No se proporcionó un servicio válido para la reserva.")

            reserva = Reserva(id_reserva, cliente, servicio, duracion_horas)
            total = reserva.confirmar(
                impuesto_pct=impuesto_pct,
                descuento_pct=descuento_pct,
                recargo_fijo=recargo_fijo,
            )
            if auto_procesar:
                reserva.procesar()
        except SoftwareFJError as exc:
            self._registrar_error(f"Gestión de reserva ({id_reserva})", exc)
            return None
        else:
            self.__reservas.append(reserva)
            self.__logger.info(
                f"Reserva exitosa: {reserva.obtener_resumen()} (Total facturado: ${total:,.2f})"
            )
            print(f"  [OK] {reserva.obtener_resumen()}")
            return reserva
        finally:
            self.__logger.debug(
                f"Cierre de transacción para reserva '{id_reserva}'. "
                f"Reservas registradas en sistema: {len(self.__reservas)}"
            )

    def cancelar_reserva(self, reserva: Reserva, motivo: str) -> bool:
        """Cancela una reserva existente usando bloque try / except."""
        try:
            reserva.cancelar(motivo)
            self.__logger.info(f"Reserva {reserva.id} cancelada. Motivo: {motivo}")
            print(f"  [OK] Reserva {reserva.id} cancelada correctamente ({motivo}).")
            return True
        except SoftwareFJError as exc:
            self._registrar_error(f"Cancelación de reserva ({reserva.id})", exc)
            return False

    def procesar_reserva(self, reserva: Reserva) -> bool:
        """Procesa una reserva existente usando bloque try / except."""
        try:
            reserva.procesar()
            self.__logger.info(f"Reserva {reserva.id} procesada exitosamente.")
            print(f"  [OK] Reserva {reserva.id} procesada correctamente.")
            return True
        except SoftwareFJError as exc:
            self._registrar_error(f"Procesamiento de reserva ({reserva.id})", exc)
            return False

    def mostrar_estado_general(self) -> None:
        """Imprime el estado consolidado del sistema en memoria."""
        print("\n" + "=" * 75)
        print("RESUMEN FINAL DEL SISTEMA - SOFTWARE FJ (SIN BASE DE DATOS)")
        print("=" * 75)
        print(f"Clientes registrados : {len(self.__clientes)}")
        for c in self.__clientes:
            print(f"  - {c.obtener_resumen()}")

        print(f"\nServicios registrados: {len(self.__servicios)}")
        for s in self.__servicios:
            print(f"  - {s.obtener_resumen()}")

        print(f"\nReservas registradas : {len(self.__reservas)}")
        for r in self.__reservas:
            print(f"  - {r.obtener_resumen()}")
        print("=" * 75)
        self.__logger.info("=== FIN DE SIMULACIÓN - SISTEMA ESTABLE ===")


# ==============================================================================
# 8. SIMULACIÓN DE 13 OPERACIONES COMPLETAS (VÁLIDAS Y FALLIDAS)
# ==============================================================================

def ejecutar_simulacion() -> None:
    gestor = GestorSoftwareFJ("software_fj.log")
    print("Iniciando simulación de operaciones en Software FJ...\n")

    # Operación 1: Registro válido de cliente
    print("Operación 1: Registro válido de cliente (Ana Torres)")
    cliente_1 = gestor.registrar_cliente(
        "CLI-001", "Ana María Torres", "1020304050", "ana.torres@empresa.com", "+57 310 555 1234"
    )

    # Operación 2: Registro válido de segundo cliente
    print("\nOperación 2: Registro válido de cliente (Carlos Méndez)")
    cliente_2 = gestor.registrar_cliente(
        "CLI-002", "Carlos Méndez", "80123456", "carlos.mendez@tech.co", "3204449876"
    )

    # Operación 3: Registro inválido de cliente (correo mal formado -> DatosInvalidosError)
    print("\nOperación 3: Registro inválido de cliente (email incorrecto)")
    gestor.registrar_cliente(
        "CLI-003", "Laura Gómez", "1098765432", "laura.gomez_sin_arroba.com", "3001112233"
    )

    # Operación 4: Registro inválido de cliente (nombre faltante -> ParametroFaltanteError)
    print("\nOperación 4: Registro inválido de cliente (parámetro obligatorio vacío)")
    gestor.registrar_cliente(
        "CLI-004", "   ", "1055667788", "vacio@empresa.com", "3112223344"
    )

    # Operación 5: Creación válida de servicio 1 (ReservaSala)
    print("\nOperación 5: Creación válida de servicio - Reserva de Sala")
    sala_juntas = gestor.crear_servicio_seguro(
        ReservaSala,
        id_servicio="SRV-SALA-01",
        nombre="Sala Innovación Norte",
        tarifa_base_hora=120000.0,
        capacidad_personas=20,
        incluye_audiovisual=True,
    )

    # Operación 6: Creación válida de servicio 2 (AlquilerEquipo)
    print("\nOperación 6: Creación válida de servicio - Alquiler de Equipos")
    kit_portatiles = gestor.crear_servicio_seguro(
        AlquilerEquipo,
        id_servicio="SRV-EQ-01",
        nombre="Workstations Portátiles i7",
        tarifa_base_hora=45000.0,
        tipo_equipo="Laptop Corporativa",
        stock_disponible=4,
        unidades_por_reserva=2,
    )

    # Operación 7: Creación válida de servicio 3 (AsesoriaEspecializada)
    print("\nOperación 7: Creación válida de servicio - Asesoría Especializada")
    asesoria_cloud = gestor.crear_servicio_seguro(
        AsesoriaEspecializada,
        id_servicio="SRV-ASE-01",
        nombre="Arquitectura Cloud y Seguridad",
        tarifa_base_hora=180000.0,
        especialidad="Ciberseguridad",
        nivel_complejidad="AVANZADO",
        consultor_asignado="Ing. Felipe Rojas",
    )

    # Operación 8: Creación inválida de servicio (tarifa negativa -> DatosInvalidosError)
    print("\nOperación 8: Creación inválida de servicio (tarifa base negativa)")
    gestor.crear_servicio_seguro(
        ReservaSala,
        id_servicio="SRV-SALA-ERR",
        nombre="Sala Fallida",
        tarifa_base_hora=-50000.0,
        capacidad_personas=10,
    )

    # Operación 9: Reserva exitosa con sobrecarga (IVA 19% + descuento 10%) y procesamiento
    print("\nOperación 9: Reserva exitosa de Sala (IVA 19%, descuento 10%, procesada)")
    gestor.crear_y_confirmar_reserva(
        id_reserva="RES-001",
        cliente=cliente_1,
        servicio=sala_juntas,
        duracion_horas=3,
        impuesto_pct=19.0,
        descuento_pct=10.0,
        auto_procesar=True,
    )

    # Operación 10: Reserva fallida por servicio no disponible (encadenamiento -> ReservaError)
    print("\nOperación 10: Reserva fallida (Sala ya ocupada -> ServicioNoDisponibleError encadenada)")
    gestor.crear_y_confirmar_reserva(
        id_reserva="RES-002",
        cliente=cliente_2,
        servicio=sala_juntas,
        duracion_horas=2,
        impuesto_pct=19.0,
    )

    # Operación 11: Reserva exitosa de AlquilerEquipo con sobrecarga (recargo fijo sin descuento)
    print("\nOperación 11: Reserva exitosa de Alquiler de Equipos (con recargo de transporte)")
    reserva_equipos = gestor.crear_y_confirmar_reserva(
        id_reserva="RES-003",
        cliente=cliente_2,
        servicio=kit_portatiles,
        duracion_horas=4,
        impuesto_pct=19.0,
        recargo_fijo=25000.0,
        auto_procesar=False,
    )

    # Operación 12: Reserva fallida por cálculo inconsistente (descuento 150% -> CalculoInconsistenteError)
    print("\nOperación 12: Reserva fallida (descuento fuera de rango -> CalculoInconsistenteError)")
    gestor.crear_y_confirmar_reserva(
        id_reserva="RES-004",
        cliente=cliente_1,
        servicio=asesoria_cloud,
        duracion_horas=2,
        impuesto_pct=19.0,
        descuento_pct=150.0,
    )

    # Operación 13: Cancelación válida y posterior intento inválido de procesamiento
    print("\nOperación 13: Cancelación de RES-003 y tentativa prohibida de procesarla")
    if reserva_equipos is not None:
        gestor.cancelar_reserva(reserva_equipos, "Cambio de fecha por el cliente")
        # El siguiente intento debe fallar con OperacionNoPermitidaError sin detener el programa
        gestor.procesar_reserva(reserva_equipos)

    # Estado final comprobando estabilidad completa
    gestor.mostrar_estado_general()
    print("\nLogs detallados guardados en: software_fj.log")


if __name__ == "__main__":
    ejecutar_simulacion()

