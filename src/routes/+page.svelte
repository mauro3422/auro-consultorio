<script lang="ts">
    import { onMount } from "svelte";

    type Turno= {
        id:number;
        paciente:string;
        hora:string;
        fecha: string;
        motivo:string;
        estado:string;
        dni: string;
        fechaNacimiento?: string;
    };
    let turnos: Turno[] = $state([
        {
            id:1,
            paciente:"Juan Perez",
            hora:"10:00",
            fecha:"2026-11-11",
            motivo:"Consulta general",
            estado:"Pendiente",
            dni:"12345678"
        },
        {
            id:2,
            paciente:"Maria Lopez",
            hora:"11:00",
            fecha:"2026-11-10",
            motivo:"Control de presión",
            estado:"Confirmado",
            dni:"87654321"
        },
        {
            id:3,
            paciente:"Carlos Sanchez",
            hora:"12:00",
            fecha:"2026-11-12",
            motivo:"Revisión de resultados",
            estado:"Cancelado",
            dni:"11223344"
        }
    ]);

    
    function cambiarEstado(id: number, nuevoEstado: string) {
        const turno = turnos.find((t) => t.id === id);

        if (turno) {
            turno.estado = nuevoEstado;
        }
    }

    /*
     * CHULETA $state (Svelte 5) — cómo conectar un campo:
     * 1. Declarar: let nombreNuevo = $state(""); // variable reactiva, empieza vacía
     * 2. Vincular en HTML: <input bind:value={nombreNuevo} required />
     *    Escribir en el input cambia nombreNuevo; cambiar nombreNuevo actualiza el input.
     * 3. Validar al enviar: if (!nombreNuevo.trim()) return;
     * 4. Guardar en turnos.push({ ... }):
     *    paciente: `${nombreNuevo.trim()} ${apellidoNuevo.trim()}`,
     * 5. Limpiar después: nombreNuevo = ""; // el input se vacía por el bind
     * Flujo: input -> bind:value -> $state -> validar -> turnos.push -> limpiar.
     * Importante: $state conserva datos en memoria mientras usás la página;
     * al recargar, los cambios se pierden (todavía no hay base de datos).
     */
    let nombreNuevo = $state("");
    let apellidoNuevo = $state("");
    let horaNueva= $state("");
    let fechaNueva = $state("");
    let motivoNuevo = $state("");
    let dniNuevo = $state("");
    let fechaNacimientoNueva = $state("");
    let mostrarFormulario = $state(false);
    let confirmarDescarte = $state(false);

    // Si se escribió cualquiera de los campos, avisamos antes de descartar.
    let hayDatosSinGuardar = $derived(Boolean(
        nombreNuevo.trim() || apellidoNuevo.trim() || dniNuevo.trim() ||
        fechaNacimientoNueva || fechaNueva || horaNueva || motivoNuevo.trim()
    ));
    
    let fechaSeleccionada = $state("");
    // El catálogo completo es independiente del día elegido en el calendario.
    let mostrarCatalogo = $state(false);

    // "Hoy" es la fecha real del consultorio; NO depende del día seleccionado.
    // Usamos zona horaria explícita para que el servidor y el navegador coincidan.
    const partesHoy = new Intl.DateTimeFormat("es-AR", {
        timeZone: "America/Argentina/Tucuman",
        year: "numeric", month: "2-digit", day: "2-digit"
    }).formatToParts(new Date());
    const valorHoy = (tipo: "year" | "month" | "day") => partesHoy.find(p => p.type === tipo)?.value ?? "";
    const fechaHoy = `${valorHoy("year")}-${valorHoy("month")}-${valorHoy("day")}`;

    let turnosFiltrados = $derived(
        turnos
            .filter(t =>
                !fechaSeleccionada ||
                t.fecha === fechaSeleccionada
            )
            .sort((a, b) =>
                (a.fecha + "T" + a.hora)
                    .localeCompare(b.fecha + "T" + b.hora)
            )
    );

    // Siguientes Turnos usa la hora actual del consultorio, no solo el día.
    // Se refresca cada minuto para que un turno que terminó deje de figurar.
    let relojActual = $state(new Date());
    onMount(() => {
        const intervalo = window.setInterval(() => {
            relojActual = new Date();
        }, 60_000);
        return () => window.clearInterval(intervalo);
    });

    let fechaHoraActual = $derived.by(() => {
        const partes = new Intl.DateTimeFormat("es-AR", {
            timeZone: "America/Argentina/Tucuman",
            year: "numeric", month: "2-digit", day: "2-digit",
            hour: "2-digit", minute: "2-digit", hourCycle: "h23"
        }).formatToParts(relojActual);
        const valor = (tipo: "year" | "month" | "day" | "hour" | "minute") =>
            partes.find(p => p.type === tipo)?.value ?? "";
        return `${valor("year")}-${valor("month")}-${valor("day")}T${valor("hour")}:${valor("minute")}`;
    });

    // No incluye cancelados ni turnos pasados; solo los cinco siguientes.
    let proximosTurnos = $derived(
        turnos
            .filter(t => t.estado !== "Cancelado" &&
                `${t.fecha}T${t.hora}` >= fechaHoraActual)
            .sort((a, b) =>
                (a.fecha + "T" + a.hora).localeCompare(b.fecha + "T" + b.hora))
            .slice(0, 5)
    );

    // Para el catálogo mostramos todos los turnos (también cancelados).
    // El resumen lateral continúa usando turnosFiltrados sin perder la fecha.
    let turnosParaTarjetas = $derived(
        mostrarCatalogo
            ? [...turnos].sort((a, b) =>
                (a.fecha + "T" + a.hora).localeCompare(b.fecha + "T" + b.hora))
            : turnosFiltrados
    );

    // Mes que estamos mirando en el calendario.
    let mesVisible = $state(new Date().getMonth());
    let anioVisible = $state(new Date().getFullYear());

    // Nombre del mes, por ejemplo: "noviembre de 2026".
    let tituloMes = $derived(
        new Intl.DateTimeFormat("es-AR", {
            month: "long",
            year: "numeric"
        }).format(new Date(anioVisible, mesVisible, 1))
    );

    // Cuántos espacios vacíos hay antes del día 1.
    let espaciosIniciales = $derived(
        (new Date(anioVisible, mesVisible, 1).getDay() + 6) % 7
    );

    // Cuántos días tiene el mes.
    let cantidadDias = $derived(
        new Date(anioVisible, mesVisible + 1, 0).getDate()
    );

    // Casilleros vacíos + días del mes.
    let diasCalendario = $derived([
        ...Array.from({ length: espaciosIniciales }, () => null),
        ...Array.from({ length: cantidadDias }, (_, i) => i + 1)
    ]);

    // Fechas con al menos un turno no cancelado.
    let fechasConTurnos = $derived(
        new Set(
            turnos
                .filter(t => t.estado !== "Cancelado")
                .map(t => t.fecha)
        )
    );

    // Mantener también las fechas con turnos cancelados: son parte del historial.
    let fechasConCancelados = $derived(
        new Set(turnos.filter(t => t.estado === "Cancelado").map(t => t.fecha))
    );

    function cambiarMes(cambio: number) {
        const fecha = new Date(anioVisible, mesVisible + cambio, 1);

        mesVisible = fecha.getMonth();
        anioVisible = fecha.getFullYear();
        fechaSeleccionada = "";
    }

    function fechaCalendario(dia: number) {
        const mes = String(mesVisible + 1).padStart(2, "0");
        const numeroDia = String(dia).padStart(2, "0");

        return `${anioVisible}-${mes}-${numeroDia}`;
    }

    // Cerrar sin perder lo escrito: si hay borrador, pedir confirmación.
    function solicitarCierreFormulario() {
        if (hayDatosSinGuardar) {
            confirmarDescarte = true;
            return;
        }
        confirmarDescarte = false;
        mostrarFormulario = false;
    }

    function limpiarBorrador() {
        nombreNuevo = "";
        apellidoNuevo = "";
        horaNueva = "";
        fechaNueva = "";
        motivoNuevo = "";
        dniNuevo = "";
        fechaNacimientoNueva = "";
    }

    function descartarBorrador() {
        limpiarBorrador();
        confirmarDescarte = false;
        mostrarFormulario = false;
    }

    // Clic fuera: el fondo sigue funcionando, pero no se pierden los campos.
    function cerrarAlPulsarFuera(event: PointerEvent) {
        if (!mostrarFormulario) return;
        const objetivo = event.target;
        if (objetivo instanceof Element &&
            objetivo.closest(".formulario-flotante, .abrir-turno")) return;
        solicitarCierreFormulario();
    }

    function cerrarConEscape(event: KeyboardEvent) {
        if (event.key === "Escape" && mostrarFormulario) solicitarCierreFormulario();
    }

    function agregarTurno(event:SubmitEvent){
        event.preventDefault();
        
       if (
            !nombreNuevo.trim() ||
            !apellidoNuevo.trim() ||
            !fechaNueva ||
            !horaNueva ||
            !motivoNuevo.trim()||
            !dniNuevo.trim()
        ) {
            return;
        }
        const nuevoId = Math.max(0, ...turnos.map(t => t.id))+1;

        turnos.push({
            id:nuevoId,
            paciente: `${nombreNuevo.trim()} ${apellidoNuevo.trim()}`,
            hora:horaNueva,
            fecha:fechaNueva,
            motivo:motivoNuevo.trim(),
            estado:"Pendiente",
            dni:dniNuevo.trim(),
            fechaNacimiento: fechaNacimientoNueva || undefined
        });
        limpiarBorrador();
        confirmarDescarte = false;
        mostrarFormulario = false; // Cerrar después de guardar, sin pedir descartar.
    }

</script>
<svelte:window onpointerdown={cerrarAlPulsarFuera} onkeydown={cerrarConEscape} />
<div class="app-layout">
    <aside class="sidebar">
        <h2>Auro Consultorio</h2>
        <p>Panel de gestion</p>
        
        <nav aria-label="Menu principal">
            <span class="menu-item activo"> Agenda</span>
            <span class="menu-item">Pacientes . Proximamente</span>
            <span class="menu-item"> Cobros . Proximamente.</span>
        </nav>    
    </aside>
    <main class="contenido">

<h1> Auro Consultorio - Agenda de turnos</h1>

<button
    type="button"
    class="abrir-turno"
    aria-expanded={mostrarFormulario}
    aria-controls="panel-nuevo-turno"
    onclick={() => mostrarFormulario = true}
>
    + Nuevo turno
</button>

{#if mostrarFormulario}
    <!-- Panel flotante NO modal: no hay fondo bloqueante ni se mueve la agenda. -->
    <section id="panel-nuevo-turno" class="formulario-flotante" aria-label="Formulario para nuevo turno">
        <button
            type="button"
            class="cerrar-flotante"
            aria-label="Cerrar formulario"
            title="Cerrar formulario"
            onclick={solicitarCierreFormulario}
        >×</button>
        <form class="formulario" onsubmit={agregarTurno}>
        <h2> Nuevo Turno</h2>
        {#if confirmarDescarte}
            <div class="aviso-borrador" role="alert">
                <strong>Tenés datos sin guardar.</strong>
                <p>¿Querés guardar el turno, seguir editando o descartar el borrador?</p>
                <div class="acciones-borrador">
                    <button type="submit">Guardar turno</button>
                    <button type="button" onclick={() => confirmarDescarte = false}>Seguir editando</button>
                    <button type="button" class="descartar-borrador" onclick={descartarBorrador}>Descartar datos</button>
                </div>
            </div>
        {/if}
        
        
            <div class="fila-nombre-apellido">
                <div>
                    <label for="nombre">Nombre <abbr class="requerido" title="Campo obligatorio">*</abbr></label>
                    <input
                        id="nombre"
                        type="text"
                        bind:value={nombreNuevo}
                        required
                    />
                </div>

                <div>
                    <label for="apellido">Apellido <abbr class="requerido" title="Campo obligatorio">*</abbr></label>
                    <input
                        id="apellido"
                        type="text"
                        bind:value={apellidoNuevo}
                        required
                    />
                </div>

                <div>
                    <label
                        for="nacimiento"
                        title="Este campo es opcional"
                    >
                        Nacimiento
                    </label>
                    <input
                        id="nacimiento"
                        type="date"
                        bind:value={fechaNacimientoNueva}
                    />
                </div>

            </div>
            
        
            <div class="fila-dni-fecha-hora">
                <!-- DNI -->
                <div>
                    <label for="dni">DNI <abbr class="requerido" title="Campo obligatorio">*</abbr></label>
                    <input
                        id="dni"
                        type="text"
                        inputmode="numeric"
                        maxlength="8"
                        pattern={'[0-9]{7,8}'}
                        bind:value={dniNuevo}
                        required
                    />
                </div>

                <!-- Fecha -->
                <div>
                    <label for="fecha">Fecha <abbr class="requerido" title="Campo obligatorio">*</abbr></label>
                    <input
                        id="fecha"
                        type="date"
                        bind:value={fechaNueva}
                        required
                    />
                </div>

                <!-- Hora -->
                <div>
                    <label for="hora">Hora <abbr class="requerido" title="Campo obligatorio">*</abbr></label>
                    <input
                        id="hora"
                        type="time"
                        bind:value={horaNueva}
                        required
                    />
                </div>
            </div>


            <label for = "motivo">Motivo <abbr class="requerido" title="Campo obligatorio">*</abbr></label>
            <textarea
                id="motivo"
                bind:value={motivoNuevo}
                rows="2"
                required
            ></textarea>
            <button type="submit">Agregar Turno</button>
    </form>
    </section>
{/if}

<div class="panel-principal">
<section class="siguientes" aria-label="Turnos por atender">
    <h2>Siguientes Turnos</h2>

    
    {#if proximosTurnos.length === 0}
        <p class="sin-proximos">No hay turnos próximos programados.</p>
    {/if}

    {#each proximosTurnos as t (t.id)}

        <div class="fila-turno">
            <strong>{t.hora}</strong> 
            <span>
                {t.paciente}
                <small class="fecha-turno">{t.fecha}</small>
            </span>
            <small class={"estado " + t.estado.toLowerCase()}>
                {t.estado}
            </small>

        </div>
    {/each}
</section>
<section class="agenda-completa" aria-label="Agenda completa">
<h2 class="seccion-titulo">Agenda completa</h2>


<div class="filtro-agenda">
    <div class="mini-calendario">

        <!-- Navegación entre meses -->
        <div class="cabecera-calendario">
            <button
                type="button"
                aria-label="Mes anterior"
                onclick={() => cambiarMes(-1)}
            >
                ‹
            </button>

            <strong>{tituloMes}</strong>

            <button
                type="button"
                aria-label="Mes siguiente"
                onclick={() => cambiarMes(1)}
            >
                ›
            </button>
        </div>

        <!-- Cuadrícula del calendario -->
        <div class="grilla-calendario">

            <!-- Encabezados de los días -->
            {#each ["L", "M", "X", "J", "V", "S", "D"] as nombre, i (i)}
                <span class="nombre-dia">{nombre}</span>
            {/each}

            <!-- Días del mes -->
            {#each diasCalendario as dia, i (i)}

                {#if dia === null}
                    <span class="celda-vacia" aria-hidden="true"></span>
                {:else}
                    {@const fechaDia = fechaCalendario(dia)}

                    <button
                        type="button"
                        class:con-turnos={fechasConTurnos.has(fechaDia)}
                        class:con-cancelados={fechasConCancelados.has(fechaDia)}
                        class:solo-cancelados={fechasConCancelados.has(fechaDia) && !fechasConTurnos.has(fechaDia)}
                        class:seleccionado={fechaSeleccionada === fechaDia}
                        class:es-hoy={fechaDia === fechaHoy}
                        aria-pressed={fechaSeleccionada === fechaDia}
                        aria-current={fechaDia === fechaHoy ? "date" : undefined}
                        title={`${fechaDia === fechaHoy ? "Hoy · " : ""}${fechasConTurnos.has(fechaDia)
                            ? (fechasConCancelados.has(fechaDia) ? "Turnos activos y cancelados" : "Turnos activos")
                            : (fechasConCancelados.has(fechaDia) ? "Solo turnos cancelados" : "Sin turnos registrados")}`}
                        onclick={() => fechaSeleccionada = fechaDia}
                    >
                        {dia}

                        {#if fechasConTurnos.has(fechaDia) || fechasConCancelados.has(fechaDia)}
                            <span class="indicadores-dia" aria-hidden="true">
                                {#if fechasConTurnos.has(fechaDia)}
                                    <span class="punto-turno"></span>
                                {/if}
                                {#if fechasConCancelados.has(fechaDia)}
                                    <span class="punto-cancelado"></span>
                                {/if}
                            </span>
                        {/if}
                    </button>
                {/if}

            {/each}
        </div>
    </div>

    <!-- Resumen del día: aparece junto al calendario solo en escritorio. -->
    <section class="resumen-dia" aria-label="Turnos del día seleccionado">
        <h3>Turnos del día</h3>
        {#if fechaSeleccionada}
            <p class="resumen-fecha">{fechaSeleccionada} · {turnosFiltrados.length} turno{turnosFiltrados.length === 1 ? "" : "s"}</p>
            {#if turnosFiltrados.length === 0}
                <p class="resumen-vacio">No hay turnos registrados para este día.</p>
            {:else}
                <div class="lista-dia">
                    {#each turnosFiltrados as turno (turno.id)}
                        <div class="mini-turno">
                            <div class="mini-turno-cabecera">
                                <strong>{turno.hora}</strong>
                                <span>{turno.paciente}</span>
                            </div>
                            <small>{turno.motivo}</small>
                            <span class={"estado " + turno.estado.toLowerCase()}>{turno.estado}</span>
                            <div class="mini-turno-acciones">
                                <button type="button" onclick={() => cambiarEstado(turno.id, "Confirmado")}>Confirmar</button>
                                <button type="button" onclick={() => cambiarEstado(turno.id, "Cancelado")}>Cancelar</button>
                            </div>
                        </div>
                    {/each}
                </div>
            {/if}
        {:else}
            <p class="resumen-vacio">Seleccioná un día del calendario para ver sus turnos acá.</p>
        {/if}
    </section>

    <div class="acciones-calendario">
        <!-- En móvil mantenemos el mensaje: allí no se muestra el resumen lateral. -->
        <p class="texto-filtro-movil">
            {fechaSeleccionada
                ? `Turnos del ${fechaSeleccionada}`
                : "Mostrando todos los turnos"}
        </p>
        <!-- En escritorio, referencia de colores en lugar de repetir la fecha. -->
        <div class="leyenda-calendario" aria-label="Referencias del calendario">
            <span><i class="muestra-hoy" aria-hidden="true"></i>Hoy</span>
            <span><i class="muestra-con-turnos" aria-hidden="true"></i>Activos</span>
            <span><i class="muestra-cancelados" aria-hidden="true"></i>Cancelados</span>
            <span><i class="muestra-seleccionado" aria-hidden="true"></i>Seleccionado</span>
        </div>

        <button
            type="button"
            aria-expanded={mostrarCatalogo}
            aria-controls="catalogo-turnos"
            onclick={() => mostrarCatalogo = !mostrarCatalogo}
        >
            {mostrarCatalogo ? "Ocultar catálogo" : "Ver todos"}
        </button>
    </div>
</div>

{#if mostrarCatalogo}
    <h3 class="titulo-catalogo">Todos los turnos · {turnos.length} registrados</h3>
{/if}
<section id="catalogo-turnos" class="turnos-grid" class:catalogo-abierto={mostrarCatalogo} aria-label={mostrarCatalogo ? "Catálogo completo de turnos" : "Turnos de la fecha seleccionada"}>

{#if turnosParaTarjetas.length === 0}
    <p>No hay turnos registrados para esta fecha.</p>
{/if}

{#each turnosParaTarjetas as turno(turno.id)}
    <article>
        <h3>{turno.paciente}</h3>
        <p>Fecha: {turno.fecha}</p>
        <p>Hora: {turno.hora}</p>
        <p>Motivo: {turno.motivo}</p>
        <p>
            Estado:
            <span class={"estado " + turno.estado.toLowerCase()}>
                {turno.estado}
            </span>
        </p>
        
        <div class="acciones">
            <button
                onclick={() => cambiarEstado(turno.id, "Confirmado")}
            >
                Confirmar
            </button>

            <button
                onclick={() => cambiarEstado(turno.id, "Cancelado")}
            >
                Cancelar
            </button>
        </div>

    </article>
{/each}
</section>
</section>
</div>
    </main>
</div>
<style>
    /* 1. Base y paleta central */
    :global(body) {
        --fondo: #D8D9D2;
        --superficie: #EAE9E2;
        --borde: #C5C9BF;
        --texto: #2D3830;
        --acento: #405848;
        --boton: #506858;
        --boton-hover: #3C5144;
        margin: 0;
        padding: 0;
        background: var(--fondo);
        color: var(--texto);
        font-family: "Segoe UI", Arial, sans-serif;
    }

    p {
        margin: 8px 0;
    }

    /* 2. Estructura y navegación */
    .app-layout {
        display: grid;
        grid-template-columns: 230px minmax(0, 1fr);
        min-height: 100vh;
    }

    .sidebar {
        background: #4D6055;
        color: #F1F0E8;
        padding: 30px 20px;
    }

    .sidebar h2 { margin: 0; }
    .sidebar p { color: #CED6CE; font-size: 14px; }

    .menu-item {
        display: block;
        padding: 12px;
        margin-top: 10px;
        border-radius: 8px;
        color: #DFE4DA;
    }

    .menu-item.activo {
        background: #E3E6DD;
        color: #35463C;
        font-weight: bold;
    }

    .contenido {
        padding: 32px;
        min-width: 0;
    }

    .contenido h1 {
        color: var(--acento);
        text-align: left;
        margin: 0 0 24px;
    }

    /* 3. Dashboard: la agenda es principal; los turnos próximos quedan a la derecha. */
    .panel-principal {
        display: grid;
        grid-template-columns: minmax(0, 1fr) minmax(280px, 300px);
        align-items: start;
        gap: 20px;
        max-width: 1180px;
        margin: 0 auto;
    }

    .agenda-completa {
        grid-column: 1;
        grid-row: 1;
        min-width: 0;
    }

    .panel-principal > .siguientes {
        grid-column: 2;
        grid-row: 1;
        position: sticky;
        top: 20px;
    }

    .agenda-completa .seccion-titulo {
        margin: 0 0 16px;
    }

    .agenda-completa .filtro-agenda {
        max-width: none;
        margin: 0 0 16px;
    }

    .titulo-catalogo {
        max-width: 940px;
        margin: 12px 0;
        font-size: 16px;
        color: var(--acento);
    }

    .agenda-completa .turnos-grid {
        max-width: none;
        grid-template-columns: repeat(auto-fit, minmax(min(100%, 220px), 1fr));
    }

    /* El resumen del día está oculto en móvil: allí siguen las tarjetas originales. */
    .resumen-dia { display: none; }

    /* Escritorio: calendario y turnos seleccionados se consultan lado a lado. */
    @media (min-width: 860px) {
        .agenda-completa .filtro-agenda {
            display: grid;
            grid-template-columns: minmax(0, 300px) minmax(0, 1fr);
            align-items: start;
            gap: 14px 20px;
        }

        .agenda-completa .mini-calendario {
            width: 100%;
            max-width: 100%;
        }

        .resumen-dia {
            display: block;
            grid-column: 2;
            grid-row: 1;
            min-width: 0;
            max-height: 322px;
            overflow-y: auto;
        }

        .resumen-dia h3 {
            margin: 0 0 8px;
            color: var(--acento);
            font-size: 16px;
        }

        .resumen-fecha {
            font-size: 12px;
            color: #637366;
            margin: 0 0 10px;
        }

        .resumen-vacio {
            margin: 0;
            color: #657268;
            font-size: 13px;
            line-height: 1.5;
        }

        .lista-dia {
            display: grid;
            gap: 10px;
        }

        .mini-turno {
            display: flex;
            flex-direction: column;
            align-items: flex-start;
            gap: 5px;
            min-width: 0;
            padding: 10px;
            background: #F4F1EA;
            border: 1px solid var(--borde);
            border-radius: 8px;
        }

        .mini-turno-cabecera {
            display: flex;
            flex-wrap: wrap;
            gap: 5px 9px;
            align-items: baseline;
            font-size: 13px;
            overflow-wrap: anywhere;
        }

        .mini-turno-cabecera strong { color: var(--acento); }
        .mini-turno > small { color: #627064; }
        .mini-turno .estado { padding: 3px 8px; font-size: 11px; }

        .mini-turno-acciones {
            display: flex;
            flex-wrap: wrap;
            gap: 6px;
        }

        .mini-turno-acciones button {
            padding: 5px 8px;
            font-size: 11px;
        }

        .agenda-completa .acciones-calendario {
            grid-column: 1 / -1;
            display: flex;
            flex-direction: row;
            flex-wrap: wrap;
            align-items: center;
            justify-content: space-between;
            align-self: auto;
            gap: 8px;
            padding-top: 10px;
            border-top: 1px solid var(--borde);
        }

        /* El catálogo se expande solo con Ver todos; no deselecciona el día. */
        .agenda-completa .turnos-grid:not(.catalogo-abierto) {
            display: none;
        }
    }

    @media (min-width: 1081px) and (max-width: 1250px) {
        .agenda-completa .filtro-agenda {
            grid-template-columns: minmax(0, 220px) minmax(0, 1fr);
            column-gap: 12px;
        }
    }

    @media (min-width: 751px) {
        /* Centrado respecto a TODA la ventana, incluyendo la barra lateral. */
        .contenido .formulario-flotante {
            right: auto;
            left: 50%;
            transform: translateX(-50%);
            width: min(670px, calc(100vw - 280px));
        }
    }

    @media (min-width: 751px) and (max-width: 990px) {
        /* El panel flotante se angosta en laptops: evitar inputs apretados. */
        .formulario-flotante .fila-nombre-apellido,
        .formulario-flotante .fila-dni-fecha-hora {
            grid-template-columns: minmax(0, 1fr);
        }
    }

    /* Botón de apertura: el panel flotante no ocupa espacio de la grilla. */
    .contenido > button {
        display: inline-flex;
        align-items: center;
        min-height: 40px;
        padding: 10px 18px;
        margin: 0 0 18px;
        font: inherit;
        font-weight: 600;
        box-shadow: 0 3px 9px #38443A14;
    }

    /* No es un modal: no hay backdrop ni bloqueo de clicks fuera del formulario. */
    .formulario-flotante {
        position: fixed;
        z-index: 30;
        top: clamp(18px, 8vh, 88px);
        right: clamp(16px, 2vw, 28px);
        width: min(740px, calc(100vw - 32px));
        max-height: calc(100dvh - 36px);
        overflow-y: auto;
        box-sizing: border-box;
        background: var(--superficie);
        border: 1px solid var(--borde);
        border-radius: 14px;
        box-shadow: 0 18px 55px #25362A46;
    }

    .formulario-flotante .formulario {
        width: 100%;
        max-width: none;
        padding: 22px;
        border: 0;
        box-shadow: none;
        background: transparent;
    }

    .formulario-flotante .formulario h2 {
        padding-right: 44px;
    }

    .cerrar-flotante {
        position: absolute;
        top: 12px;
        right: 12px;
        z-index: 1;
        display: inline-flex;
        align-items: center;
        justify-content: center;
        width: 36px;
        height: 36px;
        padding: 0;
        background: #DDE2D8;
        color: var(--acento);
        border: 1px solid var(--borde);
        font-size: 25px;
        line-height: 1;
    }

    .cerrar-flotante:hover {
        background: #CAD6CA;
    }

    .formulario {
        width: 100%;
        max-width: 800px;
        margin: 0;
        padding: 16px;
        box-sizing: border-box;
        background: var(--superficie);
        border-color: var(--borde);
        border-radius: 12px;
        box-shadow: 0 3px 10px #38443A10;
    }

    .formulario label {
        display: block;
        margin: 12px 0 6px;
    }

    .requerido {
        color: #8A3F32;
        text-decoration: none;
        cursor: help;
    }

    .formulario input {
        box-sizing: border-box;
        width: 100%;
        padding: 10px;
        border: 1px solid #CFC7B9;
        border-radius: 6px;
        background: #FFFCF6;
        color: #343831;
    }

    .formulario textarea {
        box-sizing: border-box;
        display: block;
        width: 100%;
        min-height: 48px;
        max-height: 160px;
        padding: 12px;
        border: 1px solid #CFC7B9;
        border-radius: 6px;
        background: #F4F0E8;
        color: #343831;
        font: inherit;
        line-height: 1.5;
        field-sizing: content;
        resize: vertical;
        overflow-y: auto;
    }

    .formulario button { margin-top: 15px; }

    /* Confirmación no modal: aparece dentro del panel, sin perder el borrador. */
    .aviso-borrador {
        margin: 10px 0 14px;
        padding: 12px;
        border: 1px solid #D3B29A;
        border-radius: 9px;
        background: #F7EEE3;
        color: #554334;
    }
    .aviso-borrador p { margin: 5px 0 8px; font-size: 13px; }
    .acciones-borrador { display: flex; flex-wrap: wrap; gap: 8px; }
    .acciones-borrador button { margin-top: 0; }
    .acciones-borrador .descartar-borrador {
        background: #805D51;
        color: #FFF9F2;
    }
    .acciones-borrador .descartar-borrador:hover { background: #65483E; }

    .siguientes {
        background: var(--superficie);
        border: 1px solid var(--borde);
        border-radius: 12px;
        padding: 18px;
        min-width: 0;
    }

    .siguientes h2 { margin: 0 0 16px; }

    .sin-proximos {
        margin: 0;
        padding: 8px 0;
        color: #627064;
        font-size: 13px;
        line-height: 1.5;
    }

    .fila-turno {
        display: grid;
        grid-template-columns: 44px minmax(0, 1fr) 80px;
        align-items: center;
        gap: 8px;
        padding: 8px 4px;
        border-bottom: 1px solid var(--borde);
    }

    .fila-turno strong { color: var(--acento); }
    .fila-turno small { margin-left: 0; text-align: left; }

    /* Arriba: el estado solo tiene texto de color */
    .fila-turno .estado {
        justify-self: start;
        background: transparent;
        padding: 0;
        border-radius: 0;
        font-size: 12px;
        font-weight: bold;
        white-space: nowrap;
    }

    /* 4. Agenda completa: tarjetas */
    .seccion-titulo {
        max-width: 940px;
        margin: 32px auto 16px;
        color: #35483C;
        font-size: 20px;
    }

    /* MINI GUÍA DEL CALENDARIO (lógica en <script>, estilos aquí):
     * diasCalendario = [null, null, 1, 2, ...] -> null es casilla vacía para alinear el lunes.
     * fechasConTurnos.has(fechaDia) -> true si esa fecha tiene algún turno no cancelado.
     * class:con-turnos={...} -> Svelte pone la clase si da true; CSS pinta el día.
     * class:seleccionado={...} -> Svelte pinta el botón del día que tocaste.
     * Al tocar un día cambia fechaSeleccionada; $derived actualiza las tarjetas.
     */
    .filtro-agenda {
        display: flex;
        align-items: flex-start;
        flex-wrap: wrap;
        gap: 24px;
        max-width: 940px;
        margin: 0 auto 16px;
        padding: 16px;
        box-sizing: border-box;
        background: var(--superficie);
        border: 1px solid var(--borde);
        border-radius: 12px;
    }

    .mini-calendario {
        width: 300px;
        max-width: 100%;
        min-width: 0;
    }

    .cabecera-calendario {
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 10px;
        margin-bottom: 12px;
        text-transform: capitalize;
    }

    .cabecera-calendario strong {
        color: var(--acento);
        font-size: 15px;
        text-align: center;
    }

    .cabecera-calendario button {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        flex: 0 0 34px;
        height: 34px;
        padding: 0;
        border: 1px solid var(--borde);
        background: #DDE2D8;
        color: var(--acento);
        font-size: 24px;
        line-height: 1;
    }

    .cabecera-calendario button:hover {
        background: #CED9CC;
    }

    .grilla-calendario {
        display: grid;
        grid-template-columns: repeat(7, minmax(0, 1fr));
        gap: 4px;
    }

    .nombre-dia,
    .celda-vacia {
        display: flex;
        align-items: center;
        justify-content: center;
        min-height: 24px;
        font-size: 12px;
        color: #647368;
    }

    .grilla-calendario > button {
        position: relative;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        gap: 2px;
        min-width: 0;
        min-height: 38px;
        padding: 4px;
        border: 1px solid transparent;
        border-radius: 8px;
        background: #F4F0E8;
        color: var(--texto);
        font: inherit;
        font-size: 13px;
    }

    .grilla-calendario > button:hover {
        background: #DDE5DA;
    }

    .grilla-calendario > button.con-turnos {
        background: #DEEADC;
        border-color: #9FB9A4;
        font-weight: 700;
    }

    /* Sin turnos activos, pero con historial cancelado: borde punteado terracota. */
    .grilla-calendario > button.con-cancelados.solo-cancelados:not(.seleccionado) {
        background: #F4EBE6;
        border: 1px dashed #B58470;
    }

    .grilla-calendario > button.seleccionado {
        background: var(--acento);
        border-color: var(--acento);
        color: #FAF7EE;
    }

    .grilla-calendario > button.solo-cancelados.seleccionado {
        border: 1px dashed #F5CCB8;
    }

    /* Hoy tiene un borde ocre distinto del punto verde y del fondo seleccionado. */
    .grilla-calendario > button.es-hoy {
        box-shadow: inset 0 0 0 2px #A77A35;
    }

    .grilla-calendario > button.es-hoy.seleccionado {
        box-shadow: inset 0 0 0 2px #E3BB6A;
    }

    /* Un día mixto muestra dos puntos en la misma línea, sin agrandar la celda. */
    .indicadores-dia {
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 4px;
        height: 5px;
        flex: 0 0 5px;
    }

    .punto-turno, .punto-cancelado {
        display: block;
        width: 5px;
        height: 5px;
        flex: 0 0 5px;
        border-radius: 50%;
    }
    .punto-turno { background: #63836C; }
    .punto-cancelado { background: #B57A65; }

    .grilla-calendario > button.seleccionado .punto-turno { background: #F8F5E9; }
    .grilla-calendario > button.seleccionado .punto-cancelado { background: #FFD1B7; }

    .grilla-calendario > button:focus-visible,
    .cabecera-calendario button:focus-visible {
        outline: 2px solid var(--acento);
        outline-offset: 2px;
    }

    /* Referencias del calendario; solo escritorio. */
    .leyenda-calendario { display: none; }

    .leyenda-calendario span {
        display: inline-flex;
        align-items: center;
        gap: 6px;
        color: #5B6A5E;
        font-size: 11px;
        white-space: nowrap;
    }

    .leyenda-calendario i {
        display: inline-block;
        width: 12px;
        height: 12px;
        box-sizing: border-box;
        border-radius: 4px;
    }

    .muestra-hoy { background: #F4F0E8; border: 2px solid #A77A35; }
    .muestra-con-turnos { background: #DEEADC; border: 1px solid #9FB9A4; }
    .muestra-cancelados { background: #F4EBE6; border: 1px dashed #B58470; }
    .muestra-seleccionado { background: var(--acento); }

    @media (min-width: 860px) {
        .agenda-completa .texto-filtro-movil { display: none; }
        .agenda-completa .leyenda-calendario { display: flex; flex-wrap: wrap; gap: 9px 14px; }
    }

    .acciones-calendario {
        display: flex;
        flex: 1 1 170px;
        min-width: 0;
        flex-direction: column;
        align-items: flex-start;
        justify-content: center;
        gap: 12px;
        align-self: stretch;
    }

    .acciones-calendario p {
        margin: 0;
        font-size: 14px;
        color: var(--texto);
    }

    @media (max-width: 600px) {
        .filtro-agenda {
            gap: 14px;
        }
        .mini-calendario {
            width: min(300px, 100%);
            margin: 0 auto;
        }
        .acciones-calendario {
            flex-basis: 100%;
            flex-direction: row;
            align-items: center;
            justify-content: space-between;
            flex-wrap: wrap;
        }
    }

    .turnos-grid > p {
        grid-column: 1 / -1;
        margin: 0;
        padding: 20px;
        text-align: center;
        background: var(--superficie);
        border: 1px solid var(--borde);
        border-radius: 10px;
    }

    .turnos-grid {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
        gap: 16px;
        max-width: 940px;
        margin: 0 auto;
    }

    .turnos-grid article {
        display: flex;
        flex-direction: column;
        margin: 0;
        max-width: none;
        padding: 20px;
        background: var(--superficie);
        border: 1px solid var(--borde);
        border-radius: 12px;
        box-shadow: 0 3px 10px #38443A10;
    }

    .turnos-grid article h3 {
        margin-top: 0;
        color: var(--acento);
    }

    .turnos-grid .acciones {
        display: flex;
        gap: 10px;
        margin-top: auto;
        padding-top: 16px;
    }

    /* 5. Botones y estados reutilizados */
    button {
        padding: 8px 14px;
        border: none;
        border-radius: 8px;
        background: var(--boton);
        color: #FAF7EE;
        cursor: pointer;
    }

    button:hover { background: var(--boton-hover); }

    .estado {
        display: inline-block;
        padding: 5px 10px;
        border-radius: 20px;
        font-size: 13px;
        font-weight: bold;
    }

    .pendiente {
        background-color: #fff3cd;
        color: #856404;
    }

    .confirmado {
        background-color: #d4edda;
        color: #155724;
    }

    .cancelado {
        background-color: #f8d7da;
        color: #721c24;
    }

    /* 6. Adaptación: primero la agenda, después los próximos turnos. */
    @media (max-width: 1080px) {
        .panel-principal {
            grid-template-columns: minmax(0, 1fr);
        }
        .agenda-completa {
            grid-column: 1;
            grid-row: 1;
        }
        .panel-principal > .siguientes {
            grid-column: 1;
            grid-row: 2;
            position: static;
        }
    }

    @media (max-width: 750px) {
        .app-layout { grid-template-columns: 1fr; }
        .sidebar { padding: 18px; }
        .contenido { padding: 18px; }
    }

    @media (max-width: 600px) {
        .turnos-grid { grid-template-columns: 1fr; }
        .formulario-flotante {
            top: 12px;
            right: 16px;
            width: calc(100vw - 48px);
            max-height: calc(100dvh - 24px);
        }
        .formulario-flotante .formulario { padding: 16px; }
    }
    


.fila-nombre-apellido {
    display: grid;
    grid-template-columns: repeat(3, minmax(0, 1fr));
    gap: 12px;
}

.fila-nombre-apellido > div {
    min-width: 0;
}

@media (max-width: 600px) {
    .fila-nombre-apellido {
        grid-template-columns: 1fr;
    }
}

.fila-dni-fecha-hora {
    display: grid;
    grid-template-columns: minmax(115px, 0.85fr) minmax(170px, 1.45fr) minmax(105px, 0.7fr);
    gap: 12px;
}

.fila-dni-fecha-hora > div {
    min-width: 0;
}

@media (max-width: 600px) {
    .fila-dni-fecha-hora {
        grid-template-columns: 1fr;
    }
}

.fecha-turno {
    display: block;
    font-size: 11px;
    opacity: 0.75;
    margin-top: 2px;
}
</style>