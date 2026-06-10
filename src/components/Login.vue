<script setup>
import { ref } from 'vue'
import axios from 'axios'

const usuario = ref('')
const password = ref('')
const telefono = ref('') // NUEVA VARIABLE PARA EL TELÉFONO

// Variables para controlar la pantalla y mensajes
const esRegistro = ref(false) // Si es false, muestra Login. Si es true, muestra Registro.
const mensajeExito = ref('')
const mensajeError = ref('')

const emit = defineEmits(['login-exitoso'])

const procesarFormulario = async () => {
  // Limpiamos mensajes anteriores
  mensajeExito.value = ''
  mensajeError.value = ''

  if (esRegistro.value) {
    // -----------------------------------------
    // MODO REGISTRO: Enviamos los datos a Django
    // -----------------------------------------
    
    // Validación extra: Obligamos a que pongan su teléfono
    if (!telefono.value) {
      mensajeError.value = "Por favor, ingresa tu número de WhatsApp para contactarte si es necesario."
      return
    }

    try {
      // Usamos el enlace de PythonAnywhere y enviamos el teléfono
      const respuesta = await axios.post('https://hafidbau.pythonanywhere.com/api/registro/', {
        username: usuario.value,
        password: password.value,
        telefono: telefono.value // NUEVO CAMPO ENVIADO AL BACKEND
      })
      
      // Si salió bien, mostramos el mensaje y lo regresamos a Iniciar Sesión
      mensajeExito.value = respuesta.data.mensaje
      esRegistro.value = false 
      password.value = '' // Borramos la contraseña por seguridad
      telefono.value = '' // Borramos el teléfono
      
    } catch (error) {
      // Si hubo un error (ej. el nombre ya existe), lo mostramos
      if (error.response && error.response.data) {
        mensajeError.value = error.response.data.error
      } else {
        mensajeError.value = "Error al conectar con el servidor."
      }
    }
 } else {
    // -----------------------------------------
    // MODO LOGIN: Entramos a la quinela
    // -----------------------------------------
    if (usuario.value !== '' && password.value !== '') {
      try {
        // Le preguntamos a Django si la contraseña es correcta
        const respuesta = await axios.post('https://hafidbau.pythonanywhere.com/api/login/', {
          username: usuario.value,
          password: password.value
        })
        
        // Si Django dice que todo está bien, lo dejamos pasar a los partidos
        emit('login-exitoso', respuesta.data.username)
        
      } catch (error) {
        // Si se equivoca de contraseña, mostramos el error
        if (error.response && error.response.data) {
          mensajeError.value = error.response.data.error
        } else {
          mensajeError.value = "Error al conectar con el servidor."
        }
      }
    } else {
      mensajeError.value = "Por favor ingresa usuario y contraseña"
    }
  }
}

// NUEVA FUNCIÓN: RECUPERAR CONTRASEÑA POR WHATSAPP
const recuperarContrasena = () => {
  if (usuario.value === '') {
    mensajeError.value = "Escribe tu Usuario en la casilla de arriba y luego presiona aquí para saber quién eres."
    return
  }
  
  const texto = `Hola Hafid, soy ${usuario.value}. Olvidé mi contraseña de la quiniela, ¿me ayudas a recuperarla?`
  const url = `https://wa.me/524431205091?text=${encodeURIComponent(texto)}`
  window.open(url, '_blank')
}
</script>

<template>
  <div class="login-contenedor">
    <div class="tarjeta-login">
      <h2 class="titulo-login">{{ esRegistro ? 'NUEVA CUENTA' : 'BIENVENIDO' }}</h2>
      <p class="subtitulo">{{ esRegistro ? 'Crea tu usuario para participar' : 'Ingresa para hacer tu pronóstico' }}</p>

      <div v-if="mensajeExito" class="alerta exito">{{ mensajeExito }}</div>
      <div v-if="mensajeError" class="alerta error">{{ mensajeError }}</div>

      <form @submit.prevent="procesarFormulario" class="formulario">
        <div class="grupo-input">
          <label>Usuario (Sin espacios)</label>
          <input type="text" v-model="usuario" placeholder="Ej. Gabo, Isa, Lalo..." required>
        </div>

        <div v-if="esRegistro" class="grupo-input slide-down">
          <label>Número de WhatsApp</label>
          <input type="tel" v-model="telefono" placeholder="Ej. 4431234567" required>
        </div>

        <div class="grupo-input">
          <label>Contraseña</label>
          <input type="password" v-model="password" placeholder="••••••••" required>
        </div>

        <button type="submit" class="btn-entrar">
          {{ esRegistro ? 'REGISTRARME' : 'ENTRAR A LA QUINELA' }}
        </button>
      </form>

      <p v-if="!esRegistro" class="texto-olvido" @click="recuperarContrasena">
        ¿Olvidaste tu contraseña?
      </p>

      <p class="texto-cambio">
        {{ esRegistro ? '¿Ya tienes cuenta?' : '¿No tienes cuenta?' }}
        <span class="enlace" @click="esRegistro = !esRegistro; mensajeError = ''; mensajeExito = ''">
          {{ esRegistro ? 'Inicia Sesión aquí' : 'Regístrate aquí' }}
        </span>
      </p>
    </div>
  </div>
</template>

<style scoped>
.login-contenedor {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 80vh;
  padding: 20px;
}

.tarjeta-login {
  background-color: var(--negro-fifa);
  padding: 40px;
  border-radius: 12px;
  box-shadow: 0 10px 25px rgba(0,0,0,0.5);
  width: 100%;
  max-width: 400px;
  text-align: center;
  border-top: 8px solid var(--verde-neon);
}

.titulo-login {
  color: var(--blanco);
  margin: 0;
  font-size: 2em;
  letter-spacing: 2px;
}

.subtitulo {
  color: var(--verde-neon);
  margin-top: 5px;
  margin-bottom: 25px;
  letter-spacing: 1px;
}

.grupo-input {
  display: flex;
  flex-direction: column;
  text-align: left;
  margin-bottom: 20px;
}

.grupo-input label {
  color: var(--blanco);
  font-size: 0.9em;
  margin-bottom: 8px;
  font-weight: bold;
}

.grupo-input input {
  padding: 12px;
  border-radius: 8px;
  border: 2px solid #333;
  background-color: #222;
  color: var(--blanco);
  font-size: 1.1em;
}

.grupo-input input:focus {
  outline: none;
  border-color: var(--verde-neon);
}

.btn-entrar {
  width: 100%;
  background-color: var(--verde-neon);
  color: var(--negro-fifa);
  border: none;
  padding: 15px;
  font-size: 1.1em;
  font-weight: bold;
  cursor: pointer;
  border-radius: 8px;
  margin-top: 10px;
  transition: all 0.3s ease;
}

.btn-entrar:hover {
  background-color: var(--verde-oscuro);
  color: var(--blanco);
}

/* Nuevos estilos para las alertas y el texto de cambio */
.texto-cambio {
  color: #ccc;
  margin-top: 25px;
  font-size: 0.9em;
}

.enlace {
  color: var(--verde-neon);
  cursor: pointer;
  font-weight: bold;
  text-decoration: underline;
  margin-left: 5px;
}

.alerta {
  padding: 10px;
  border-radius: 6px;
  margin-bottom: 20px;
  font-size: 0.9em;
  font-weight: bold;
}

.exito {
  background-color: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.error {
  background-color: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

/* ESTILO DEL NUEVO BOTÓN DE RECUPERACIÓN */
.texto-olvido {
  color: #888;
  margin-top: 15px;
  margin-bottom: 0;
  font-size: 0.85em;
  cursor: pointer;
  text-decoration: underline;
  transition: color 0.3s ease;
}

.texto-olvido:hover {
  color: var(--blanco);
}

/* Animación suave al abrir el registro */
.slide-down {
  animation: deslizar 0.3s ease-out;
}

@keyframes deslizar {
  from { opacity: 0; transform: translateY(-10px); }
  to { opacity: 1; transform: translateY(0); }
}
</style>