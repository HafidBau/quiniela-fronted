<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
import Login from './components/Login.vue'

// Variables de estado general
const sesionIniciada = ref(false)
const nombreUsuario = ref('')
const partidos = ref([])
const cargando = ref(true)
const tablaPuntuaciones = ref([])
const modoEdicion = ref(true) 
const enviando = ref(false) // Controla la pantalla de "Guardando..."

// Variables para la ventana espía de predicciones
const prediccionesModal = ref([])
const mostrandoPredicciones = ref(false)
const partidoViendo = ref({ local: '', visitante: '' })

const banderas = {
  'México': '🇲🇽', 'Sudáfrica': '🇿🇦', 'República de Corea': '🇰🇷', 'República Checa': '🇨🇿',
  'Canadá': '🇨🇦', 'Bosnia y Herzegovina': '🇧🇦', 'Catar': '🇶🇦', 'Suiza': '🇨🇭',
  'Estados Unidos': '🇺🇸', 'Paraguay': '🇵🇾', 'Australia': '🇦🇺', 'Turquía': '🇹🇷',
  'Brasil': '🇧🇷', 'Marruecos': '🇲🇦', 'Haití': '🇭🇹', 'Escocia': '🏴󠁧󠁢󠁳󠁣󠁴󠁿',
  'Alemania': '🇩🇪', 'Curazao': '🇨🇼', 'Costa de Marfil': '🇨🇮', 'Ecuador': '🇪🇨',
  'Países Bajos': '🇳🇱', 'Japón': '🇯🇵', 'Suecia': '🇸🇪', 'Túnez': '🇹🇳',
  'España': '🇪🇸', 'Cabo Verde': '🇨🇻', 'Arabia Saudí': '🇸🇦', 'Uruguay': '🇺🇾',
  'Bélgica': '🇧🇪', 'Egipto': '🇪🇬', 'Irán': '🇮🇷', 'Nueva Zelanda': '🇳🇿',
  'Francia': '🇫🇷', 'Senegal': '🇸🇳', 'Irak': '🇮🇶', 'Noruega': '🇳🇴',
  'Argentina': '🇦🇷', 'Argelia': '🇩🇿', 'Austria': '🇦🇹', 'Jordania': '🇯🇴',
  'Portugal': '🇵🇹', 'RD Congo': '🇨🇩', 'Uzbekistán': '🇺🇿', 'Colombia': '🇨🇴',
  'Inglaterra': '🏴󠁧󠁢󠁥󠁮󠁧󠁿', 'Croacia': '🇭🇷', 'Ghana': '🇬🇭', 'Panamá': '🇵🇦'
}
const obtenerBandera = (pais) => banderas[pais] || '🏳️'

// Verifica si el partido ya pasó de las 0:00 hrs de su día
const partidoBloqueado = (fechaPartido) => {
  const hoy = new Date()
  const fechaLimite = new Date(fechaPartido)
  fechaLimite.setHours(0, 0, 0, 0)
  return hoy >= fechaLimite
}

const entrarAlSistema = async (usuario) => {
  nombreUsuario.value = usuario
  sesionIniciada.value = true
  localStorage.setItem('quinela_usuario', usuario) // Guardamos en memoria
  
  await obtenerPartidos() 
  await cargarMisPronosticos() 
  await cargarTabla()
}

const cerrarSesion = () => {
  sesionIniciada.value = false
  nombreUsuario.value = ''
  partidos.value = [] 
  tablaPuntuaciones.value = []
  modoEdicion.value = true 
  localStorage.removeItem('quinela_usuario') // Borramos la memoria
}

const guardarTodos = async () => {
  const pronosticosListos = partidos.value.filter(p => 
    p.goles_local !== undefined && p.goles_visitante !== undefined && 
    p.goles_local !== '' && p.goles_visitante !== '' &&
    !partidoBloqueado(p.fecha)
  ).map(p => ({
    partido_id: p.id,
    goles_local: p.goles_local,
    goles_visitante: p.goles_visitante
  }))

  if (pronosticosListos.length === 0) {
    alert("⚽ No hay pronósticos nuevos o válidos para guardar.")
    return
  }

  enviando.value = true // Encendemos animación de carga

  try {
    const respuesta = await axios.post('https://hafidbau.pythonanywhere.com/api/guardar_todos/', {
      username: nombreUsuario.value,
      pronosticos: pronosticosListos
    })

    setTimeout(() => {
      alert("✅ " + respuesta.data.mensaje)
      modoEdicion.value = false 
      enviando.value = false 
    }, 500)
    
  } catch (error) {
    console.error("Error al guardar todos:", error)
    alert("❌ Hubo un error al guardar. Intenta de nuevo.")
    enviando.value = false
  }
}

const cargarMisPronosticos = async () => {
  try {
    const respuesta = await axios.get(`https://hafidbau.pythonanywhere.com/api/mis_pronosticos/${nombreUsuario.value}/`)
    const misGolesGuardados = respuesta.data

    if (misGolesGuardados.length > 0) {
      modoEdicion.value = false
    }

    misGolesGuardados.forEach(pronostico => {
      const partidoEnPantalla = partidos.value.find(p => p.id === pronostico.partido_id)
      if (partidoEnPantalla) {
        partidoEnPantalla.goles_local = pronostico.goles_local
        partidoEnPantalla.goles_visitante = pronostico.goles_visitante
      }
    })
  } catch (error) {
    console.error("No se pudieron cargar los pronósticos anteriores:", error)
  }
}

const obtenerPartidos = async () => {
  try {
    const respuesta = await axios.get('https://hafidbau.pythonanywhere.com/api/partidos/')
    partidos.value = respuesta.data
  } catch (error) {
    console.error("Error al traer los partidos:", error)
  } finally {
    cargando.value = false
  }
}

const cargarTabla = async () => {
  try {
    const respuesta = await axios.get('https://hafidbau.pythonanywhere.com/api/tabla/')
    tablaPuntuaciones.value = respuesta.data
  } catch (error) {
    console.error("Error al cargar la tabla:", error)
  }
}

// Va a Django por las predicciones de los demás
const verPredicciones = async (partido) => {
  try {
    const respuesta = await axios.get(`https://hafidbau.pythonanywhere.com/api/predicciones/${partido.id}/`)
    prediccionesModal.value = respuesta.data
    partidoViendo.value = { local: partido.equipo_local.nombre, visitante: partido.equipo_visitante.nombre }
    mostrandoPredicciones.value = true
  } catch (error) {
    console.error("Error al cargar predicciones", error)
  }
}

const cerrarModal = () => {
  mostrandoPredicciones.value = false
  prediccionesModal.value = []
}

onMounted(() => {
  const usuarioGuardado = localStorage.getItem('quinela_usuario')
  if (usuarioGuardado) {
    entrarAlSistema(usuarioGuardado) // Login automático si hay sesión guardada
  }
})
</script>

<template>
  <main class="contenedor">
    <div class="header-oficial" style="position: relative;">
      <h1 class="titulo">FIFA WORLD CUP 26™</h1>
      <h2 class="subtitulo" v-if="sesionIniciada">QUINELA DE {{ nombreUsuario.toUpperCase() }}</h2>
      <h2 class="subtitulo" v-else>QUINELA OFICIAL</h2>
      
      <button v-if="sesionIniciada" class="btn-salir" @click="cerrarSesion">
        🚪 Salir
      </button>
    </div>

    <Login v-if="!sesionIniciada" @login-exitoso="entrarAlSistema" />

    <div v-else>
      <div v-if="cargando" class="pantalla-carga">
        <div class="spinner">⚽</div>
        <p>Cargando calendario oficial...</p>
      </div>

      <div v-else>
        
        <div class="lista-partidos">
          <div v-for="partido in partidos" :key="partido.id" class="tarjeta-partido">
            
            <div class="cinta-fecha" :style="partidoBloqueado(partido.fecha) ? 'background-color: #8b0000;' : ''">
              <span>{{ new Date(partido.fecha).toLocaleString([], {month: 'short', day: 'numeric', hour: '2-digit', minute:'2-digit'}) }}</span>
              <span v-if="partidoBloqueado(partido.fecha)"> 🔒 CERRADO</span>
            </div>
            
            <div class="marcador-container">
              <div class="equipo">
                <span class="bandera">{{ obtenerBandera(partido.equipo_local.nombre) }}</span>
                <span class="nombre-pais">{{ partido.equipo_local.nombre }}</span>
              </div>
              
              <div class="inputs-goles">
                <input type="number" min="0" placeholder="0" class="input-gol" 
                       v-model="partido.goles_local" 
                       :disabled="!modoEdicion || partidoBloqueado(partido.fecha)" 
                       :class="{'input-bloqueado': !modoEdicion || partidoBloqueado(partido.fecha)}">
                
                <span class="vs">vs</span>
                
                <input type="number" min="0" placeholder="0" class="input-gol" 
                       v-model="partido.goles_visitante" 
                       :disabled="!modoEdicion || partidoBloqueado(partido.fecha)"
                       :class="{'input-bloqueado': !modoEdicion || partidoBloqueado(partido.fecha)}">
              </div>

              <div class="equipo">
                <span class="bandera">{{ obtenerBandera(partido.equipo_visitante.nombre) }}</span>
                <span class="nombre-pais">{{ partido.equipo_visitante.nombre }}</span>
              </div>
            </div>

            <div v-if="partidoBloqueado(partido.fecha)" style="padding: 10px; border-top: 1px solid #eee; text-align: center;">
              <button class="btn-ver-predicciones" @click="verPredicciones(partido)">
                👁️ Ver predicciones de todos
              </button>
            </div>

          </div>
        </div> 
        
        <div class="contenedor-botones-inferior">
          <button v-if="modoEdicion" class="btn-guardar" @click="guardarTodos">
            💾 GUARDAR MIS PRONÓSTICOS
          </button>
          
          <div v-else class="grupo-botones-bloqueados">
            <button class="btn-guardado" disabled>
              ✅ GUARDADO
            </button>
            <button class="btn-editar" @click="modoEdicion = true">
              ✏️ EDITAR
            </button>
          </div>
        </div>

        <div v-if="tablaPuntuaciones.length > 0" class="seccion-tabla">
          <h2 class="titulo-seccion">🏆 TABLA DE POSICIONES</h2>
          <div class="tabla-contenedor">
            <div v-for="(jugador, index) in tablaPuntuaciones" :key="jugador.usuario" class="fila-jugador">
              <span class="posicion">#{{ index + 1 }}</span>
              <span class="nombre-jugador">{{ jugador.usuario.toUpperCase() }}</span>
              <span class="puntos-jugador">{{ jugador.puntos }} pts</span>
            </div>
          </div>
        </div>

      </div>
    </div>

    <div v-if="enviando" class="overlay-carga">
      <div class="caja-carga">
        <div class="spinner-pelota">⚽</div>
        <h3 class="texto-carga">Enviando Resultados...</h3>
        <p class="subtexto-carga">Asegurando tus pronósticos en la base de datos</p>
      </div>
    </div>

    <div v-if="mostrandoPredicciones" class="overlay-carga" @click.self="cerrarModal">
      <div class="caja-modal">
        <button class="btn-cerrar-modal" @click="cerrarModal">✖</button>
        <h3 class="titulo-modal">Pronósticos del Grupo</h3>
        <p class="subtitulo-modal">{{ obtenerBandera(partidoViendo.local) }} {{ partidoViendo.local }} vs {{ partidoViendo.visitante }} {{ obtenerBandera(partidoViendo.visitante) }}</p>
        
        <div class="lista-predicciones">
          <div v-for="p in prediccionesModal" :key="p.usuario" class="fila-prediccion">
            <span class="nombre-espia">{{ p.usuario.toUpperCase() }}</span>
            <span class="marcador-espia">{{ p.goles_local }} - {{ p.goles_visitante }}</span>
          </div>
          <div v-if="prediccionesModal.length === 0" style="padding: 20px; color: #666;">
            Aún no hay pronósticos para este partido.
          </div>
        </div>
      </div>
    </div>
      <a href="https://wa.me/524431205091?text=Hola%20Hafid,%20tengo%20un%20problema%20con%20la%20quiniela..." class="btn-whatsapp" target="_blank" rel="noopener noreferrer">
      💬 Soporte
    </a>
  </main>
</template>

<style>
:root {
  --verde-neon: #18ff4c;
  --verde-oscuro: #024827;
  --negro-fifa: #10013a;
  --blanco: #ffffff;
  --gris-fondo: #f4f7f6;
  --rojo-hover: #ff3b3b;
}

body { background-color: #f4f7f6; margin: 0; }
</style>

<style scoped>
/* Contenedor y Header */
.contenedor { max-width: 1200px; margin: 0 auto; font-family: 'Arial Black', Impact, sans-serif; padding: 20px; }
.header-oficial { background-color: var(--negro-fifa); color: var(--blanco); text-align: center; padding: 25px 20px; border-radius: 12px; margin-bottom: 30px; border-bottom: 8px solid var(--verde-neon); }
.titulo { margin: 0; font-size: 2.2em; letter-spacing: 2px; }
.subtitulo { margin: 5px 0 0 0; color: var(--verde-neon); font-size: 1.1em; letter-spacing: 5px; }

/* Botón Salir */
.btn-salir { position: absolute; top: 20px; right: 20px; background-color: var(--rojo-hover); color: var(--blanco); border: none; padding: 10px 15px; border-radius: 8px; font-weight: bold; font-family: Arial, sans-serif; cursor: pointer; box-shadow: 0 4px 6px rgba(0,0,0,0.3); transition: 0.3s; }
.btn-salir:hover { background-color: #d11a1a; }
@media (max-width: 600px) { .btn-salir { position: static; display: block; margin: 15px auto 0 auto; width: 150px; } }

/* Carga inicial */
.pantalla-carga { text-align: center; padding: 50px; font-family: Arial, sans-serif; color: var(--verde-oscuro); font-size: 1.2em; font-weight: bold; }
.spinner { font-size: 3em; animation: girar 1s linear infinite; margin-bottom: 15px; display: inline-block; }
@keyframes girar { 100% { transform: rotate(360deg); } }

/* Tarjetas de partidos */
.lista-partidos { display: grid; grid-template-columns: repeat(auto-fit, minmax(350px, 1fr)); gap: 25px; }
.tarjeta-partido { background: var(--blanco); border-radius: 12px; box-shadow: 0 4px 10px rgba(0,0,0,0.1); overflow: hidden; display: flex; flex-direction: column; }
.cinta-fecha { background-color: var(--verde-oscuro); color: var(--blanco); text-align: center; padding: 8px; font-family: Arial, sans-serif; font-weight: bold; font-size: 0.85em; letter-spacing: 1px; transition: 0.3s; }
.marcador-container { display: flex; justify-content: space-between; align-items: center; padding: 20px; flex-grow: 1; }
.equipo { flex: 1; display: flex; flex-direction: column; align-items: center; gap: 8px; font-size: 1.1em; color: var(--negro-fifa); text-align: center; }
.bandera { font-size: 2.2em; line-height: 1; }

/* Inputs de goles */
.inputs-goles { display: flex; align-items: center; gap: 8px; background: var(--gris-fondo); padding: 10px; border-radius: 15px; }
.input-gol { width: 45px; height: 45px; text-align: center; font-size: 1.4em; font-weight: bold; border: 2px solid #ccc; border-radius: 8px; background: var(--blanco); color: var(--negro-fifa); }
.input-gol:focus { outline: none; border-color: var(--verde-neon); box-shadow: 0 0 8px rgba(24, 255, 76, 0.3); }
.input-bloqueado { background-color: #e9ecef; color: #6c757d; border-color: #ced4da; cursor: not-allowed; }

/* Botones inferiores (Guardar/Editar) */
.contenedor-botones-inferior { text-align: center; margin-top: 40px; margin-bottom: 60px; display: flex; justify-content: center; }
.btn-guardar { width: 100%; max-width: 350px; background-color: var(--verde-neon); color: var(--negro-fifa); border: none; padding: 15px; font-size: 1.1em; font-weight: bold; cursor: pointer; transition: 0.3s ease; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,255,100,0.3); }
.btn-guardar:hover { background-color: var(--rojo-hover); color: var(--blanco); }
.grupo-botones-bloqueados { display: flex; gap: 20px; width: 100%; max-width: 450px; justify-content: center; }
.btn-guardado { flex: 1; background-color: #d1d5db; color: #4b5563; border: none; padding: 15px; font-size: 1.1em; font-weight: bold; border-radius: 12px; cursor: not-allowed; }
.btn-editar { flex: 1; background-color: #f59e0b; color: var(--negro-fifa); border: none; padding: 15px; font-size: 1.1em; font-weight: bold; border-radius: 12px; cursor: pointer; transition: 0.3s ease; box-shadow: 0 4px 15px rgba(245,158,11,0.3); }
.btn-editar:hover { background-color: #d97706; color: var(--blanco); }

/* Tabla de Posiciones */
.seccion-tabla { margin-bottom: 40px; margin-top: 20px; }
.titulo-seccion { color: var(--negro-fifa); text-align: center; font-size: 1.5em; margin-bottom: 20px; }
.tabla-contenedor { background: var(--blanco); border-radius: 12px; padding: 20px; box-shadow: 0 4px 10px rgba(0,0,0,0.1); max-width: 600px; margin: 0 auto; }
.fila-jugador { display: flex; justify-content: space-between; padding: 15px; border-bottom: 1px solid #eee; font-size: 1.2em; }
.fila-jugador:last-child { border-bottom: none; }
.posicion { font-weight: bold; color: var(--verde-oscuro); width: 40px; }
.nombre-jugador { flex-grow: 1; text-align: left; padding-left: 15px; color: var(--negro-fifa); }
.puntos-jugador { font-weight: bold; color: var(--negro-fifa); }

/* OVERLAY Y MODALES */
.overlay-carga { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background-color: rgba(16, 1, 58, 0.85); display: flex; justify-content: center; align-items: center; z-index: 9999; backdrop-filter: blur(4px); }
.caja-carga { background-color: var(--blanco); padding: 40px; border-radius: 20px; text-align: center; box-shadow: 0 15px 30px rgba(24, 255, 76, 0.2); border: 4px solid var(--verde-neon); max-width: 90%; }
.spinner-pelota { font-size: 4em; margin-bottom: 15px; display: inline-block; animation: palpitar 0.8s ease-in-out infinite alternate; }
.texto-carga { color: var(--negro-fifa); margin: 0 0 10px 0; font-size: 1.5em; font-weight: bold; }
.subtexto-carga { color: #666; margin: 0; font-family: Arial, sans-serif; font-size: 1em; }
@keyframes palpitar { 0% { transform: scale(1); } 100% { transform: scale(1.2); } }

/* Botón y Ventana Espía */
.btn-ver-predicciones { width: 100%; background: #e0e7ff; color: #3730a3; border: none; padding: 10px; border-radius: 8px; font-weight: bold; cursor: pointer; transition: 0.3s; }
.btn-ver-predicciones:hover { background: #c7d2fe; }
.caja-modal { background: var(--blanco); width: 90%; max-width: 400px; border-radius: 15px; padding: 25px; position: relative; box-shadow: 0 10px 25px rgba(0,0,0,0.5); text-align: center; font-family: Arial, sans-serif; }
.btn-cerrar-modal { position: absolute; top: 15px; right: 15px; background: none; border: none; font-size: 1.5em; cursor: pointer; color: #888; }
.btn-cerrar-modal:hover { color: var(--rojo-hover); }
.titulo-modal { margin: 0 0 5px 0; color: var(--negro-fifa); font-family: 'Arial Black', Impact, sans-serif; }
.subtitulo-modal { margin: 0 0 20px 0; color: #666; font-weight: bold; font-size: 1.1em; }
.lista-predicciones { max-height: 300px; overflow-y: auto; text-align: left; }
.fila-prediccion { display: flex; justify-content: space-between; padding: 12px; border-bottom: 1px solid #eee; background: #f8fafc; margin-bottom: 5px; border-radius: 8px; }
.nombre-espia { font-weight: bold; color: var(--negro-fifa); }
.marcador-espia { font-weight: bold; color: var(--verde-oscuro); font-size: 1.2em; background: #e5e7eb; padding: 2px 10px; border-radius: 6px; text-align: center; min-width: 60px; }

/* --- AJUSTES PARA PANTALLAS DE CELULARES --- */
@media (max-width: 450px) {
  /* Reducimos el espacio a los lados para aprovechar toda la pantalla */
  .marcador-container { 
    padding: 15px 5px; 
  }
  
  /* Hacemos la letra de los países un poco más pequeña */
  .equipo { 
    font-size: 0.85em; 
    gap: 5px; 
  }
  
  /* Achicamos un poco las banderas */
  .bandera { 
    font-size: 1.8em; 
  }
  
  /* Hacemos los cuadritos de los goles ligeramente más pequeños */
  .input-gol { 
    width: 38px; 
    height: 38px; 
    font-size: 1.2em; 
  }
  
  /* Juntamos un poco más los inputs con el "vs" */
  .inputs-goles { 
    padding: 6px; 
    gap: 5px; 
  }
}

/* --- BOTÓN DE WHATSAPP FLOTANTE --- */
.btn-whatsapp {
  position: fixed;
  bottom: 25px;
  right: 25px;
  background-color: #25D366; /* Verde oficial de WhatsApp */
  color: #ffffff;
  padding: 12px 20px;
  border-radius: 50px;
  text-decoration: none;
  font-weight: bold;
  font-family: Arial, sans-serif;
  font-size: 1.1em;
  box-shadow: 0 5px 15px rgba(37, 211, 102, 0.4);
  z-index: 10000; /* Asegura que flote sobre todo lo demás */
  display: flex;
  align-items: center;
  gap: 8px;
  transition: transform 0.3s ease, background-color 0.3s ease;
}

.btn-whatsapp:hover {
  transform: scale(1.08);
  background-color: #1ebe5d;
}

/* Ajuste para que en celular no estorbe tanto */
@media (max-width: 600px) {
  .btn-whatsapp {
    bottom: 15px;
    right: 15px;
    padding: 10px 15px;
    font-size: 1em;
  }
}
</style>